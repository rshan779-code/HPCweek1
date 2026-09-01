# CS6290 Week 5 - 多处理器 + 缓存一致性 + 内存排序 + 内存一致性 + 同步 + 众核

---

## 1. 多处理器（Multi-Processing）

### Flynn 分类法
| 类型 | 描述 | 示例 |
|------|------|------|
| **SISD** | 单指令单数据 | 传统单核处理器 |
| **SIMD** | 单指令多数据 | GPU、向量处理器、SSE/AVX |
| **MISD** | 多指令单数据 | 罕见（容错系统） |
| **MIMD** | 多指令多数据 | 多核处理器、多处理器系统 |

### 共享内存 vs 分布式内存
- **共享内存（SMP）**：所有处理器共享同一物理内存，编程简单
- **分布式内存（集群）**：每个节点有自己的内存，通过网络通信，扩展性好

### 多线程（SMT / Hyper-Threading）
- 同一核心同时运行多个线程（共享执行单元）
- 当一个线程 stall（等内存），另一个线程可以使用 ALU
- Intel 叫 Hyper-Threading（每核 2 线程）

### Amdahl 定律对并行的限制
- 加速比受限于串行部分：`Speedup ≤ 1 / (串行比例 + 并行比例/N)`
- 即使 95% 可并行，100 核最多加速 ~17 倍

---

## 2. 缓存一致性（Cache Coherence）

### 问题描述
- 核 A 写 x=15 → A 的 cache 有 x=15，内存还是 x=0
- 核 B 读 x → B 的 cache 中是旧值 x=0 → **错误！**

### MSI 协议（三种状态）
| 状态 | 描述 |
|------|------|
| **M（Modified）** | 只有该 cache 有，已修改，内存旧 |
| **S（Shared）** | 多个 cache 可同时持有，未修改 |
| **I（Invalid）** | 无效，该 cache 没有该块 |

### MESI 协议（更常用）
- 增加 **E（Exclusive）**：只有该 cache 有，但未修改（可直接写入而不用广播）

### 监听协议（Snooping）
- 所有 cache 控制器监听共享总线
- 当有人写某地址，其他持有该地址副本的 cache 无效化

### 写策略
- **Write-Invalidate**：写时使其他 cache 中的副本无效（常用）
- **Write-Update**：写时更新所有 cache 中的副本（带宽大）

### 目录协议（Directory Coherence）
- 大规模系统（无共享总线）使用
- 目录记录哪些处理器持有哪些块的副本
- 写时通知相关处理器无效化，不需要广播

---

## 3. 内存排序（Memory Ordering）

### 为什么需要内存排序
- Store 和 Load 可能访问相同内存地址，也存在依赖
- 乱序执行时，load 可能先于 store 执行，读到旧值

### Store-Load 依赖
- `STORE [addr], val` 后的 `LOAD [addr]` 应该读到 `val`
- 乱序时 load 可能先执行，读到旧值

### Store Queue / Store-to-Load Forwarding
- 处理器维护 store queue（已发射但未 commit 的 store）
- load 执行前先查 store queue
- 若有相同地址的 store，直接用其值（**Store-to-Load Forwarding**）

### Store 提交规则
- Store 只能在 ROB 头部才能写入内存（保证按序 commit）
- Load 可以乱序执行（配合 store-to-load forwarding）

---

## 4. 内存一致性（Memory Consistency）

### Coherence vs Consistency
- **Coherence**：对**同一地址**的访问顺序（所有处理器看到相同顺序）
- **Consistency**：对**不同地址**的访问顺序（处理器可以看到什么样的顺序）

### 顺序一致性（Sequential Consistency，SC）
- 所有处理器看到的内存操作，与某一全局顺序一致
- 每个处理器的操作保持程序顺序
- **正确但性能差**（限制硬件优化）

### 宽松一致性模型
| 模型 | 允许重排序 | 使用者 |
|------|------|------|
| **TSO（Total Store Order）** | Store-Load 可重排 | x86 |
| **PSO** | Store-Store 也可重排 | SPARC |
| **Weak Ordering** | 几乎所有访问可重排 | ARM（早期） |
| **Release Consistency** | acquire/release 之间可重排 | 常见于语言内存模型 |

### 内存屏障（Memory Barrier / Fence）
- 程序员显式插入，防止编译器/处理器重排
- x86：`MFENCE`、`SFENCE`、`LFENCE`
- ARM：`DMB`、`DSB`、`ISB`

---

## 5. 同步（Synchronization）

### 为什么需要同步
- 多线程修改共享变量若不同步 → race condition（数据竞争）
- 例：两线程都加载 count=15，各自加1后都存16 → 丢失一次加法

### 原子操作（Atomic Operations）
- **Test-and-Set**：原子地读值并设为 1
- **Compare-and-Swap（CAS）**：原子比较并交换
- **Fetch-and-Add**：原子地读取并加法
- **LL/SC（Load-Linked / Store-Conditional）**：ARM 和 MIPS 使用

### 自旋锁（Spinlock）
- 基于 atomic 操作实现，等待时忙转（busy-wait）
- 适合短临界区、多核环境

### 自旋锁优化
- **Test-and-Test-and-Set**：先普通读判断是否可能可用，再用原子操作，减少总线流量
- **Exponential Backoff**：失败后等待指数增长时间再重试

### Barrier（屏障同步）
- 所有线程到达屏障点后才能继续
- 用于并行计算的阶段分隔

---

## 6. 众核（Many Cores）

### 核心挑战

**1. 一致性流量瓶颈**
- 核数增多 → 写操作增多 → 无效化（invalidation）增多 → 总线带宽成瓶颈
- 解决：片上网络（NoC）+ 目录一致性协议

**2. 片上网络（Network-on-Chip，NoC）**
- 用网格（Mesh）或环形（Ring）拓扑替代共享总线
- 允许多个核心同时通信，不再是一次一个请求
- 示例：Intel 的 Ring Bus、Mesh 拓扑

**3. 内存带宽瓶颈**
- 核数多，引脚数有限
- 解决：3D 堆叠内存（HBM）、更多内存通道

**4. 目录一致性协议**
- 记录每个 cache line 被哪些核持有
- 写时只通知相关核，不需广播到所有核
- 比总线监听更具扩展性

**5. 功耗和散热**
- 动态频率/电压调节（DVFS）
- 部分核心休眠（Power Gating）

**6. 编程模型**
- 需要新的并行编程模型（OpenMP、MPI、CUDA、OpenCL）
- 应用程序需要足够并行度才能利用众多核心

---

## 关键术语
- **MESI**：缓存一致性协议（Modified/Exclusive/Shared/Invalid）
- **Snooping**：监听总线的一致性实现
- **Directory Coherence**：目录一致性，适合大规模多核
- **TSO**：Total Store Order，x86 的内存一致性模型
- **Memory Fence**：内存屏障，防止重排序
- **CAS（Compare-and-Swap）**：原子比较并交换操作
- **Spinlock**：自旋锁，忙等待的锁
- **NoC（Network-on-Chip）**：片上网络
- **HBM（High Bandwidth Memory）**：高带宽内存，3D 堆叠
- **DVFS**：动态电压频率调节
