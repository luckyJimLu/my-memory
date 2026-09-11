# ModemLog、TCPDump、CHR 独立通道与弹性工作池设计

> **历史设计，非当前实施基线。** 请先阅读[统一方案](diagnostics-unified-design.md)与[修订流程图](modem-complete-mermaid-flows.md)。
>
> 本页动态线程池不是当前推荐。worker_release_slot在vTaskDelete(NULL)之前调用存在栈复用竞态，不得照抄。当前为一个双Socket Reactor加一个Storage Owner，TCPDump走本地私有快照通道。


## 1. 结论

三种业务分别走独立底层通道：

| 业务 | 通道 | 持久缓冲 | 典型特征 |
|---|---|---|---|
| ModemLog | Channel 0 | ModemLog RX Ring | 持续、高吞吐 |
| TCPDump | Channel 1 | Packet Descriptor Ring | 持续抓包、允许统计丢包 |
| CHR | Channel 2 | CHR Message Ring | 事件/分段诊断数据 |

线程不能成为业务连续性的载体。

> 通道、Socket、协议状态、序号、文件偏移和缓冲区属于持久业务上下文；动态线程只是临时执行者。

推荐结构：

```mermaid
flowchart TD
    D["核间驱动 / ISR / DMA"] --> C1["ModemLog通道"]
    D --> C2["TCPDump通道"]
    D --> C3["CHR通道"]
    C1 --> R1["ModemLog RX Ring"]
    C2 --> R2["TCPDump RX Ring"]
    C3 --> R3["CHR RX Ring"]
    R1 --> P["共享弹性工作池"]
    R2 --> P
    R3 --> P
    P --> S["唯一Storage Owner"]
    S --> F["三个独立文件"]
```

三个通道和业务上下文长期存在；0～2个处理Worker按需创建并在空闲后退出；写盘由一个串行Storage Owner完成。

## 2. 为什么不能让业务线程直接拥有通道

如果每种业务线程同时拥有Socket、通道、协议状态和文件，当线程空闲退出时会出现：

- 底层数据到达但没有接收者；
- Socket接收窗口停止推进；
- 通道回调引用失效的任务句柄；
- 半包解析状态丢失；
- 文件偏移、PCAP头或CHR事件状态丢失；
- 未完成DMA仍引用已回收栈；
- 新线程无法判断从哪个序号继续；
- 强制删除任务时互斥锁和资源无法安全释放。

因此必须把“业务生命周期”和“线程生命周期”分开。

## 3. 四层架构

### 3.1 持久通道接入层

底层驱动、ISR、DMA回调或IPC回调长期存在，只负责：

1. 根据通道ID找到业务；
2. 将数据写入该业务的有界环形缓冲；
3. 更新接收序号和统计；
4. 将业务标记为ready；
5. 立即返回。

回调中禁止创建/删除任务、调用FatFs、等待Worker或执行复杂解析。

~~~c
typedef enum {
    DIAG_CHANNEL_MODEMLOG = 0,
    DIAG_CHANNEL_TCPDUMP  = 1,
    DIAG_CHANNEL_CHR      = 2,
    DIAG_CHANNEL_COUNT
} diag_channel_id_t;
~~~

### 3.2 持久业务上下文层

每种业务使用一个静态上下文：

~~~c
typedef enum {
    SERVICE_STOPPED,
    SERVICE_STARTING,
    SERVICE_ACTIVE,
    SERVICE_QUIESCING,
    SERVICE_DRAINING,
    SERVICE_ERROR
} service_state_t;

typedef struct {
    diag_channel_id_t channel_id;

    _Atomic service_state_t state;
    _Atomic uint32_t generation;
    _Atomic uint32_t scheduled;
    _Atomic uint32_t processing;

    uint32_t rx_sequence;
    uint32_t committed_sequence;
    uint64_t file_offset;

    spsc_ring_t rx_ring;
    protocol_parser_t parser;
    service_stats_t stats;

    int socket_fd;
    FIL *file;
} diag_service_ctx_t;

static diag_service_ctx_t g_services[DIAG_CHANNEL_COUNT];
~~~

工作线程退出后，上下文仍保存：

- 通道和Socket状态；
- 半包解析状态；
- 环形缓冲区读写位置；
- 接收、处理和提交序号；
- 文件句柄与偏移；
- 流控水位；
- generation与错误统计。

### 3.3 弹性处理层

Worker只处理有界数据，不长期持有Socket、文件、DMA、协议状态或锁。

处理过程：

1. 从全局ready队列取得 `channel_id + generation`；
2. 获得该业务的串行执行权；
3. 处理一个字节预算或时间预算；
4. 把新状态提交到持久上下文；
5. 释放执行权；
6. 返回线程池等待其他业务。

### 3.4 串行存储层

三种业务如果写同一块SD卡/FatFs，只保留一个Storage Owner：

```mermaid
flowchart LR
    M["ModemLog写请求"] --> Q["Storage调度器"]
    T["TCPDump写请求"] --> Q
    C["CHR写请求"] --> Q
    Q --> W["唯一Storage Owner"]
    W --> SD["FatFs / SDMMC"]
```

这样避免多线程同时阻塞SD卡、FatFs重入锁、DMA所有权混乱和文件偏移竞争。

Storage Owner在任一业务ACTIVE、写队列非空或DMA未完成时不得退出。

## 4. 独立通道接收

~~~c
void diag_channel_rx(diag_channel_id_t channel,
                     const void *data,
                     uint32_t length)
{
    diag_service_ctx_t *ctx = &g_services[channel];

    if (atomic_load_explicit(&ctx->state,
                             memory_order_acquire)
        != SERVICE_ACTIVE) {
        ctx->stats.rx_while_inactive++;
        return;
    }

    uint32_t generation =
        atomic_load_explicit(&ctx->generation,
                             memory_order_relaxed);

    uint32_t accepted =
        ring_write(&ctx->rx_ring, data, length);

    ctx->stats.rx_bytes += accepted;

    if (accepted != length) {
        ctx->stats.dropped_bytes += length - accepted;
        apply_channel_backpressure(ctx);
    }

    service_schedule(ctx, generation);
}
~~~

每个业务必须使用独立RX Ring，不能共用一个数据队列。否则ModemLog可能占满所有空间，让CHR和TCPDump失去接收能力。

## 5. 同一业务必须串行执行

共享线程池可以有多个Worker，但同一种业务同一时刻最多只能有一个Worker处理。它相当于Java的SerialExecutor或strand。

~~~c
void service_schedule(diag_service_ctx_t *ctx,
                      uint32_t generation)
{
    uint32_t expected = 0;

    if (atomic_compare_exchange_strong(
            &ctx->scheduled,
            &expected,
            1U)) {

        ready_item_t item = {
            .channel_id = ctx->channel_id,
            .generation = generation,
        };

        ready_queue_push(&g_ready_queue, &item);
        worker_pool_ensure_capacity();
    }
}
~~~

Worker示例：

~~~c
void elastic_worker(void *argument)
{
    for (;;) {
        ready_item_t item;

        if (!ready_queue_pop(&g_ready_queue,
                             &item,
                             WORKER_IDLE_TIMEOUT)) {
            if (worker_pool_can_shrink() &&
                worker_has_no_resource()) {
                worker_release_slot();
                vTaskDelete(NULL);
            }
            continue;
        }

        diag_service_ctx_t *ctx =
            &g_services[item.channel_id];

        if (item.generation !=
            atomic_load_explicit(&ctx->generation,
                                 memory_order_acquire)) {
            continue;
        }

        if (atomic_exchange(&ctx->processing, 1U) != 0U) {
            continue;
        }

        process_service_budget(ctx,
                               MAX_BYTES_PER_RUN,
                               MAX_TIME_PER_RUN_MS);

        atomic_store_explicit(&ctx->processing,
                              0U,
                              memory_order_release);

        service_finish_or_reschedule(ctx,
                                     item.generation);
    }
}
~~~

## 6. 防止丢唤醒

处理结束时不能只执行 `scheduled = 0`，因为新数据可能恰好到达。应清零后再次检查：

~~~c
void service_finish_or_reschedule(
    diag_service_ctx_t *ctx,
    uint32_t generation)
{
    if (!ring_empty(&ctx->rx_ring)) {
        ready_queue_push_service(ctx, generation);
        return;
    }

    atomic_store_explicit(&ctx->scheduled,
                          0U,
                          memory_order_release);

    if (!ring_empty(&ctx->rx_ring)) {
        service_schedule(ctx, generation);
    }
}
~~~

这样即使处理线程退出或更换，也不会出现“环中有数据但没有调度项”。

## 7. generation保证新旧会话隔离

每次启动业务时递增generation：

~~~c
uint32_t service_start(diag_service_ctx_t *ctx)
{
    uint32_t generation =
        atomic_fetch_add(&ctx->generation, 1U) + 1U;

    reset_parser(&ctx->parser);
    reset_ring(&ctx->rx_ring);
    ctx->rx_sequence = 0;
    ctx->committed_sequence = 0;
    ctx->file_offset = 0;

    open_service_file(ctx);
    open_channel(ctx->channel_id);

    atomic_store(&ctx->state, SERVICE_ACTIVE);
    return generation;
}
~~~

ready项、写盘请求和异步回调都携带generation。业务停止后快速重启时，旧generation的数据不能写入新文件。

## 8. 三通道调度与流控

### 8.1 独立水位

每个通道独立设置：

| 水位 | 行为 |
|---:|---|
| 小于70% | 正常处理 |
| 70%～85% | 提升该业务调度优先级 |
| 85%～95% | 请求该通道减速 |
| 95%以上 | 执行业务专属丢弃策略 |

### 8.2 不同业务采用不同策略

- **ModemLog**：优先PAUSE/RESUME、TCP背压或降低日志等级；
- **TCPDump**：允许丢包，但必须写入抓包drop统计；
- **CHR**：优先保留完整事件，必要时限制日志业务；
- 控制消息必须有保留的描述符和缓冲资源。

### 8.3 公平调度

建议按“业务优先级 + 单轮预算”调度，而不是让某业务一直处理到完全为空：

| 业务 | 示例权重 | 单次预算 |
|---|---:|---:|
| CHR | 4 | 4～16 KB或2 ms |
| ModemLog | 3 | 16～32 KB或3 ms |
| TCPDump | 2 | 16～32 KB或3 ms |

实际权重应根据CHR重要性、日志速率和抓包允许丢包率调整。

## 9. 弹性线程池参数

STM32通常是单核，多个CPU Worker不会获得Java服务器式的并行收益。线程池主要用于共享栈资源和隔离阻塞。

推荐：

~~~text
最小处理Worker：0或1
最大处理Worker：2
空闲超时：2～5秒
每次处理预算：8～32 KB或1～5 ms
静态栈槽：2个
Storage Owner：任一业务ACTIVE时保持
通道接收器：会话期间始终存在
~~~

如果三种业务主要是接收后直接写盘，最省资源的方案实际上是：

~~~text
底层回调/现有tcpip上下文：三个通道快速分流
独立RX Ring：3个
Storage Owner：1个
弹性解析Worker：0～1个
~~~

只有存在压缩、过滤、PCAP封装或复杂CHR解析时，才需要第二个处理Worker。

## 10. 使用固定栈槽，不频繁使用系统堆

频繁 `xTaskCreate()` / `vTaskDelete()` 可能造成堆碎片和峰值内存不可预测。

推荐预留两个静态执行槽：

~~~c
#define WORKER_SLOT_COUNT  2
#define WORKER_STACK_WORDS 512

typedef struct {
    StaticTask_t tcb;
    StackType_t stack[WORKER_STACK_WORDS];
    _Atomic uint32_t in_use;
} worker_slot_t;

static worker_slot_t g_worker_slots[WORKER_SLOT_COUNT];
~~~

使用 `xTaskCreateStatic()` 创建Worker。线程退出后把槽归还池中，下一种业务复用。

这不会把栈归还通用堆，但能把三种业务各自一份栈，缩减为整个系统共享的两份最大栈，并避免碎片。

## 11. Worker安全退出条件

只有同时满足以下条件才允许空闲退出：

~~~text
ready队列为空
当前Worker无执行中的业务
不持有业务锁
不拥有Socket或底层通道
不拥有文件句柄
没有未完成DMA
没有未提交缓冲区
当前Worker数量高于最小值
~~~

不要由管理线程异步强制删除Worker。让Worker在安全点自行退出。

FreeRTOS中，删除任务的内核动态内存由Idle任务回收；任务代码自己申请的资源不会自动释放。因此动态创建任务时必须保证Idle任务能运行，并在自删除前释放应用资源。

## 12. 业务生命周期

```mermaid
stateDiagram-v2
    [*] --> STOPPED
    STOPPED --> STARTING: start
    STARTING --> ACTIVE: 通道和文件就绪
    ACTIVE --> QUIESCING: stop
    QUIESCING --> DRAINING: 停止新输入
    DRAINING --> STOPPED: 排空并同步
    ACTIVE --> ERROR: 通道或存储错误
    ERROR --> DRAINING: 受控停止
```

正确启动顺序：

1. 初始化持久上下文；
2. 递增generation；
3. 打开输出文件；
4. 初始化RX Ring；
5. 注册并打开该业务的独立通道；
6. 设置ACTIVE；
7. 收到数据后按需创建Worker。

正确停止顺序：

1. 设置QUIESCING；
2. 通知底层停止该通道新数据；
3. 等待通道确认；
4. 排空RX Ring和写队列；
5. 等待DMA完成；
6. `f_sync()`并关闭文件；
7. 关闭通道；
8. 设置STOPPED；
9. 允许Worker和Storage Owner在空闲超时后退出。

不能先删除Worker再关闭通道。

## 13. 连续性不变量

实现中应保证：

1. 一个通道只属于一种业务；
2. 一种业务同时最多一个处理Worker；
3. 工作线程不拥有长期资源；
4. 文件写成功后才推进提交序号；
5. 通道停止后才能关闭业务上下文；
6. 数据排空后才能关闭文件；
7. 所有异步任务携带generation；
8. 线程切换不改变业务处理顺序；
9. 某通道满不能耗尽其他通道的保留资源；
10. Storage Owner退出前不存在未完成DMA。

## 14. 推荐最终配置

> **三个独立底层通道 + 三个静态业务上下文和RX Ring + 一个串行Storage Owner + 0～2个共享弹性处理Worker。**

线程可以空闲退出，但仅限无状态Worker。只要任一业务ACTIVE，其独立通道接收入口、持久上下文和存储所有者就必须继续存在。

## 15. 参考

- [FreeRTOS Kernel task.h：xTaskCreateStatic与vTaskDelete](https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/main/include/task.h)
- [FreeRTOS Kernel任务实现](https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/main/tasks.c)
