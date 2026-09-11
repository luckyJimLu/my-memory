# Modem 诊断系统设计

## 当前设计入口

- [统一设计基线：ModemLog、CHR双Socket与MCU本地抓包](diagnostics-unified-design.md)
- [C++整体架构与交互契约](cpp-architecture-and-interactions.md)
- [完整Mermaid流程图](modem-complete-mermaid-flows.md)

当前约束：ModemLog独立Socket、CHR独立Socket；TCPDump在MCU侧lwIP收发路径只读镜像，不占第三个核间Socket。抓包采用私有静态快照池，不能长期占有正常网络pbuf；过载只丢抓包副本。建议一个Socket Reactor加一个Storage Owner，复用现有lwIP和驱动上下文。

方案含官方依据、RAM/吞吐计算、PCAP格式和实机验收计划。文档与伪代码尚未完成目标板编译、时序或性能验证，不承诺抓包零影响。

## 历史设计与修正

以下保留演进记录，不作为独立实施依据；发生冲突以统一设计基线为准。

| 文档 | 状态与主要修正 |
|---|---|
| [最初日志写盘方案](stm32-lwip-modem-log-storage-design.md) | 单Socket草案；原子操作、写入与持久化边界已修订 |
| [双网络域方案](modem-mcu-dual-netif-routing-design.md) | 地址域仍有价值；NAT只是可选拓扑，路由Hook不是完整隔离证明 |
| [弹性线程池草案](multi-channel-elastic-worker-pool-design.md) | 不采用自删除前归还静态栈；TCPDump不再视为Modem独立输入通道 |
| [早期架构审查](multi-channel-elastic-worker-pool-design-review.md) | 原pbuf_ref异步落盘建议已撤销；改成私有快照和双Socket Reactor |

## 设计演进

本次统一更新明确了本地抓包位置，重整两路Socket与Storage职责，重写流程图并标记历史方案；补充C++静态组合、类与任务映射、跨线程控制、缓冲所有权和显式停机协议。源码依据来自上游lwIP、FreeRTOS、FatFs、libpcap、PCAP格式资料与C++语言文档；详见各文档脚注和源码指纹。

后续实施先确认STM32型号、lwIP版本、总RAM、WAN峰值pps、SD最坏停顿和CHR可靠性要求。
