# Modem 诊断系统 Mermaid 流程图

本页与[统一设计基线](diagnostics-unified-design.md)及[C++实现架构](cpp-architecture-and-interactions.md)一致：ModemLog、CHR各自一个Socket；TCPDump在MCU本地旁路抓包，不再作为Modem第三个Socket。图示表达推荐设计，不代表驱动实现已验证。第10至12图补充C++模块和交互。

## 1. 整体数据路径：双Socket与本地快照

```mermaid
flowchart TD
    modemLog["Modem 日志服务"] --> logSocket["MCU ModemLog Socket"]
    modemChr["Modem CHR服务"] --> chrSocket["MCU CHR Socket"]
    logSocket --> reactor["Socket Reactor"]
    chrSocket --> reactor
    reactor --> logRing["ModemLog 专用块环"]
    reactor --> chrRing["CHR 专用消息块环"]
    net["正常 lwIP RX / TX"] --> tap["有界只读 Capture Tap"]
    tap --> normal["原始业务路径继续"]
    tap -.->|"尽力复制"| cap["私有 RX / TX 抓包环"]
    logRing --> storage["唯一 Storage Owner"]
    chrRing --> storage
    cap --> storage
    storage --> files["日志文件、CHR文件、PCAP文件"]
```

## 2. Socket Reactor公平接收

```mermaid
flowchart TD
    control["处理控制与停止请求"] --> sets["按空闲配额重建 fd集合"]
    sets --> wait["select 有限超时"]
    wait --> chr["CHR 就绪则有限读取"]
    chr --> log["ModemLog 就绪则有限读取"]
    log --> parse["保存分帧状态并发布块"]
    parse --> notify["通知 Storage"]
    notify --> budget["检查CPU预算和停止状态"]
    budget --> control
```

## 3. Capture Tap失败放行流程

```mermaid
flowchart TD
    packet["原包到达观察点"] --> enabled{"启用且在范围内？"}
    enabled -->|"否"| pass["原路径继续"]
    enabled -->|"是"| budget{"过滤及包率字节预算通过？"}
    budget -->|"否"| pass
    budget -->|"是"| slot{"立即取得私有槽？"}
    slot -->|"否"| drop["仅增加抓包drop"]
    slot -->|"是"| copy["限长限链段只读复制"]
    copy --> valid{"快照完整？"}
    valid -->|"否"| discard["归还未发布槽并记drop"]
    valid -->|"是"| publish["release发布槽并通知Storage"]
    drop --> pass
    discard --> pass
    publish --> pass
```

## 4. 槽位所有权状态

```mermaid
stateDiagram-v2
    [*] --> FREE
    FREE --> FILLING: 唯一生产者预约
    FILLING --> READY: release发布
    FILLING --> FREE: 取消未发布快照
    READY --> READING: Storage acquire取得
    READING --> FREE: 同步消费完成或复制至Storage私有区
    READING --> ERROR_HELD: IO失败且所有权尚未解除
    ERROR_HELD --> FREE: 确认DMA停止后显式清理
```

## 5. 公网与核间路由边界

```mermaid
flowchart TD
    diag["两个诊断 Socket"] --> ipc["IPC 地址域"]
    app["公网应用 Socket"] --> wan["WAN 默认接口"]
    ipc --> guardI["IPC 出口地址与链路校验"]
    wan --> guardW["WAN 出口拒绝 IPC 源和目的"]
    guardI --> local["Modem 本地诊断端点"]
    guardW --> modemWan["Modem 公网透传或路由"]
    modemWan --> cell["蜂窝网络"]
    wan -.->|"只读快照"| capture["MCU 本地 TCPDump"]
```

## 6. 指定业务安全停止

```mermaid
sequenceDiagram
    participant manager as 会话管理
    participant producer as Reactor或Capture入口
    participant storage as Storage Owner
    participant disk as SD驱动
    manager->>producer: 停止指定业务输入
    producer->>producer: 禁止新进入并等待在途完成
    producer-->>manager: 输入已静默
    manager->>storage: 排空该业务并关闭文件
    storage->>disk: 完成写入和同步
    disk-->>storage: 成功或有界故障
    storage-->>manager: 结果与资源清理确认
    manager->>manager: 标记STOPPED或FAULTED
```

## 7. RX与TX正常报文和抓包副本时序

```mermaid
sequenceDiagram
    participant driver as 正常驱动
    participant tap as Capture Tap
    participant pool as 私有抓包槽
    participant network as lwIP正常路径
    participant storage as Storage Owner
    driver->>tap: 原包仍由正常路径持有
    tap->>pool: 尝试预约并限长复制
    alt 私有槽可用且预算通过
        pool-->>tap: 快照已发布
        tap-->>driver: 返回
        driver->>network: 继续原始包处理
        storage->>pool: 异步读取副本
        storage->>storage: 序列化和写盘
        storage->>pool: 消费结束归还槽
    else 抓包无法完成
        tap-->>driver: 只记drop并返回
        driver->>network: 继续原始包处理
    end
```

## 8. 存储过载时的业务退让

```mermaid
flowchart TD
    slow["Storage变慢或故障"] --> cap["减少或停止抓包admission"]
    cap --> log["ModemLog独立高水位流控"]
    log --> chr["CHR按可靠性协议处理"]
    chr --> failed{"存储超过故障截止时间？"}
    failed -->|"否"| recover["按低水位及冷却条件恢复"]
    failed -->|"是"| fault["诊断文件FAULTED并安全清理"]
    fault --> keep["正常WAN继续运行"]
    recover --> keep
```

## 9. 会话状态而非线程生命周期

```mermaid
stateDiagram-v2
    [*] --> STOPPED
    STOPPED --> PREPARING: Storage准备资源和文件
    PREPARING --> ACTIVE: 输入admission开启
    PREPARING --> FAULTED: 初始化失败
    ACTIVE --> QUIESCING: 指定业务停止
    QUIESCING --> DRAINING: 输入静默且inflight归零
    DRAINING --> STOPPED: 排空和关闭成功
    ACTIVE --> FAULTED: IO或通道失败
    QUIESCING --> FAULTED: 停止超时
    DRAINING --> FAULTED: 同步失败
    FAULTED --> STOPPED: 安全隔离与清理完成
```

## 关键阅读约束

- 抓包失败只丢副本，原始报文继续；这不是零CPU开销保证。
- 两个应用任务为Socket Reactor和Storage Owner，不包含现有lwIP/驱动/RTOS任务。
- Capture RX/TX可能并发；使用独立私有槽与非自旋try-guard，不长期持有正常网络pbuf。
- Storage单一消费并管理FIL，原缓冲何时可复用和何时持久化是不同事件。
- RX/TX接口层、offload、PCAP LinkType和实际调用上下文以[主方案](diagnostics-unified-design.md)为准。

## 10. C++模块与执行上下文

```mermaid
flowchart TD
    api["DiagnosticApi"] -->|"固定请求槽"| coordinator["SessionCoordinator"]
    subgraph reactorTask["SocketReactor 任务"]
        coordinator --> reactor["SocketReactor"]
        reactor --> log["LogSession"]
        reactor --> chr["ChrSession"]
    end
    log --> rings["Log 与 CHR 专用环"]
    chr --> rings
    hooks["已有 lwIP RX / TX 上下文"] --> tap["CaptureTap 与 CaptureGate"]
    tap --> cap["独立 RX / TX 快照环"]
    coordinator -->|"存储命令槽"| storage["StorageOwner 任务"]
    rings --> storage
    cap --> storage
    storage --> files["FatFsPort 与三个文件会话"]
    storage -.->|"保留至确认的结果槽"| coordinator
```

## 11. C++异步启动交互

```mermaid
sequenceDiagram
    participant app as 应用
    participant ctl as Coordinator及Reactor
    participant disk as StorageOwner
    participant source as Socket会话或CaptureGate
    app->>ctl: 提交 Start 请求
    ctl->>disk: PrepareFile 与 generation
    disk-->>ctl: 保留完成结果
    alt 文件准备成功
        ctl->>source: 非阻塞连接或启用快照
        source-->>ctl: READY 或连接完成
        ctl-->>app: 请求完成为 RUNNING
    else 文件准备失败
        ctl-->>app: 请求完成为 FAULTED
    end
```

## 12. CHR持久化确认交互

```mermaid
sequenceDiagram
    participant modem as Modem CHR
    participant rx as ChrSession及Reactor
    participant ring as CHR专用环
    participant disk as StorageOwner
    modem->>rx: 有序消息字节流
    rx->>ring: 完整记录或有界分段发布
    ring->>disk: 按序消费
    disk->>disk: 写入并按策略同步
    disk-->>rx: 同步检查点与 generation
    rx-->>modem: 协议允许时发送持久化 ACK
```
