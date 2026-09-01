# CS6290 Week 3 - 编译器ILP + VLIW + Cache基础 + 高级Cache

---

## 1. 编译器 ILP（Compiler ILP）

### 核心概念
- 编译器通过代码变换，帮助处理器发现和利用更多 ILP

### 编译器能做什么
1. **减少依赖链**：改变计算树结构，减少关键路径长度
2. **把独立指令挪近**：让处理器 ROB 窗口能看到并行机会

### 树高度降低（Tree Height Reduction）
- 改变计算树结构，减少关键路径长度
- 例：`((a+b)+c)+d` → `(a+b)+(c+d)` 减少依赖深度

### 指令调度（Software Instruction Scheduling）
- 编译器重排指令顺序，减少 Load-use stall
- 例：将 load 提前，让结果有时间就绪

### 循环展开（Loop Unrolling）
- 将循环体复制多次，暴露更多 ILP
- 优点：减少循环开销，提供更多指令给调度器
- 缺点：代码体积增大

### 软件流水线（Software Pipelining）
- 将不同循环迭代的指令交错执行
- 更激进的循环优化，让多次迭代同时在不同阶段执行

---

## 2. VLIW（Very Long Instruction Word）

### 核心概念
- 处理器每周期执行一条很长的指令，由编译器静态安排哪些操作并行执行

### VLIW vs 超标量对比
| 特性 | 超标量（OOO） | VLIW |
|------|------|------|
| ILP 发现 | 硬件动态发现 | 编译器静态安排 |
| 代码兼容性 | 好（ISA 不变） | 差（编译器版本绑定） |
| 硬件复杂度 | 高 | 低 |
| 功耗 | 高 | 低 |
| 适用场景 | 通用处理器 | DSP、嵌入式 |

### VLIW 的问题
- 二进制兼容性差（新处理器需要重新编译）
- 编译器难以准确预测运行时行为（缓存 miss、分支）
- 代码体积大（未使用的槽位填充 NOP）
- 典型例子：Intel Itanium（IA-64）

---

## 3. Cache 基础（Cache Review）

### 局部性原理（Locality Principle）
- **时间局部性**：最近访问的数据可能很快再次访问
- **空间局部性**：访问某地址后，附近地址也可能很快被访问

### Cache 基本参数
- **Cache 大小**：总存储容量
- **块大小（Block Size / Cache Line）**：每次从内存读取的单位（通常 64 字节）
- **相联度（Associativity）**：
  - 直接映射（1-way）：最快但冲突多
  - 全相联（Fully Associative）：冲突最少但硬件复杂
  - N 路组相联（N-way Set Associative）：折中

### Cache 地址分解
```
地址 = [Tag | Set Index | Block Offset]
```
- Block Offset：确定块内位置（log₂(块大小) 位）
- Set Index：确定放在哪个 set（log₂(组数) 位）
- Tag：验证是否命中（剩余位）

### 缺失类型（3C）
| 类型 | 描述 | 减少方法 |
|------|------|------|
| **Cold Miss（冷缺失）** | 数据首次访问 | 硬件预取 |
| **Capacity Miss（容量缺失）** | Cache 太小装不下工作集 | 增大 Cache |
| **Conflict Miss（冲突缺失）** | 多个地址竞争同一 set | 提高相联度 |

### 写策略
| 策略 | 描述 |
|------|------|
| **Write-Through** | 命中时同时写 cache 和内存，简单但带宽大 |
| **Write-Back** | 命中时只写 cache，替换时才写内存（Dirty 位） |
| **Write-Allocate** | 缺失时将块取入 cache 再写（配合 Write-Back） |
| **No-Write-Allocate** | 缺失时直接写内存（配合 Write-Through） |

### 置换策略
- **LRU（Least Recently Used）**：替换最久未使用的块（最常用）
- **FIFO / Random**：简单但效果差

---

## 4. 高级 Cache（Advanced Caches）

### AMAT（平均内存访问时间）
```
AMAT = 命中时间 + 缺失率 × 缺失惩罚
```
优化目标：降低命中时间、降低缺失率、降低缺失惩罚

### 多级 Cache
| 级别 | 典型大小 | 速度 | 目标 |
|------|------|------|------|
| L1 | 32-64 KB | 4-5 周期 | 低延迟 |
| L2 | 256 KB - 1 MB | 10-15 周期 | 低缺失率 |
| L3 | 4-32 MB | 30-50 周期 | 大容量 |
| 主存 | GB | 100-200 周期 | 最大容量 |

### 多级 AMAT
```
AMAT = L1命中时间 + L1缺失率 × (L2命中时间 + L2缺失率 × 主存访问时间)
```

### 降低缺失率的方法
- 增大块大小（利用空间局部性）
- 提高相联度
- **预取（Prefetching）**：提前取后续数据（硬件 stride 预取）
- **Victim Cache**：L1 旁放小全相联 cache 存放刚被替换的块

### 降低缺失惩罚的方法
- **关键字优先（Critical Word First）**：优先传处理器需要的那个字
- **提前重启（Early Restart）**：块还没全部到达就恢复处理器

### 非阻塞 Cache（Non-blocking Cache）
- 缺失时不 stall 处理器，继续处理后续访问
- 支持多个 outstanding miss（MSHR - Miss Status Holding Register）

### VIPT（虚索引物理标签）
- L1 Cache 用虚拟地址索引（并行 TLB 翻译）
- 用物理标签比较（TLB 翻译完成后）
- 减少延迟（index 和 tag 可并行查找）

---

## 关键术语
- **AMAT**：平均内存访问时间
- **3C**：三种 cache 缺失（Cold、Capacity、Conflict）
- **Write-Back / Write-Through**：写策略
- **LRU**：最近最少使用置换算法
- **Prefetching**：预取，减少冷缺失
- **MSHR**：缺失状态保持寄存器，支持非阻塞 cache
- **VLIW**：超长指令字，编译器驱动的并行
- **循环展开**：暴露更多 ILP 的编译器技术
