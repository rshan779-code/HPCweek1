# 📚 CS6290 Project 0 — 知识点总览

> 基于 CS6290 完整课程笔记提炼，聚焦 Project 0 核心考点

---

## 文件目录

| 文件 | 内容 | 重要程度 |
|------|------|---------|
| [01_Pipeline_Basics.md](./01_Pipeline_Basics.md) | 流水线基础、三种冒险、数据转发 | ⭐⭐⭐ |
| [02_Branch_Prediction.md](./02_Branch_Prediction.md) | 所有分支预测器详解（核心！）| ⭐⭐⭐⭐⭐ |
| [03_ILP_and_OOO.md](./03_ILP_and_OOO.md) | ILP、乱序执行、ROB | ⭐⭐⭐⭐ |
| [04_Performance_Metrics.md](./04_Performance_Metrics.md) | 性能公式、例题、模拟器输出解读 | ⭐⭐⭐⭐⭐ |
| [05_Predication_and_Compiler.md](./05_Predication_and_Compiler.md) | 谓词执行、编译器优化、VLIW | ⭐⭐⭐ |

---

## Project 0 核心知识图谱

```
流水线
├── 5级流水线（IF/ID/EX/MEM/WB）
├── 结构冒险 → 加硬件
├── 数据冒险 → 转发/Stall
│   └── Load-Use 必须 stall 1周期
└── 控制冒险 ←━━━━━━━━━━━━━━━━━━━┐
                                   │
分支预测（解决控制冒险）              │
├── 静态：Always Not Taken / BTFN   │
├── 动态：                           │
│   ├── 1-bit 预测器                │
│   ├── 2-bit 饱和计数器（BHT）      │
│   ├── 相关预测器（GHR + PHT）      │
│   ├── Gshare（PC XOR GHR → PHT） │
│   └── Tournament（双预测器竞争）   │
├── BTB（分支目标缓存）              │
└── RAS（函数返回地址栈）            │
                                   │
ILP 与乱序执行                      │
├── 寄存器重命名（消除假依赖）        │
├── Tomasulo 算法（保留站+CDB）      │
├── ROB（精确异常+顺序commit）       │
└── 超标量（发射宽度>1）             │
    └── 分支预测越准，IPC 越高 ━━━━━┘

性能分析
├── CPI = 1 + f_b × miss_rate × penalty
├── Speedup = CPI_old / CPI_new
├── Amdahl 定律（并行加速上限）
└── AMAT（多级Cache平均访问时间）
```

---

## ⭐ Project 0 最常考/用到的公式

```
1. CPI（含分支失误）：
   CPI = 1 + 分支频率 × 失误率 × 失误惩罚

2. Speedup：
   Speedup = CPI_旧 / CPI_新 = IPC_新 / IPC_旧

3. 分支失误率：
   miss_rate = 失误次数 / 总分支次数

4. AMAT（两级Cache）：
   AMAT = L1命中时间 + L1缺失率 × (L2命中时间 + L2缺失率 × 主存时间)
```

---

## ⭐ 分支预测器快速对比

| 预测器 | 关键特征 | 准确率 |
|--------|---------|--------|
| Always Not Taken | 无硬件 | ~50% |
| BTFN | 方向判断 | ~65% |
| 1-bit BHT | 上次结果 | ~80% |
| **2-bit BHT** | 饱和计数器，需2次错才改变 | ~85% |
| **Gshare** | PC XOR 全局历史 索引 PHT | ~93% |
| Tournament | 两个预测器+selector | ~97% |

---

## 🔑 关键概念速查

| 缩写 | 全称 | 作用 |
|------|------|------|
| BHT | Branch History Table | 存 2-bit 计数器 |
| BTB | Branch Target Buffer | 存分支目标地址 |
| GHR/BHR | Global/Branch History Register | 记录最近N次分支结果 |
| PHT | Pattern History Table | 相关预测器的计数器表 |
| RAS | Return Address Stack | 预测函数返回地址 |
| ROB | ReOrder Buffer | 保证顺序commit，支持精确异常 |
| RS | Reservation Station | 等待操作数的指令缓冲 |
| CDB | Common Data Bus | 广播执行结果 |
| RAT | Register Alias Table | 逻辑→物理寄存器映射 |
| IPC | Instructions Per Cycle | 每周期指令数（越高越好）|
| CPI | Cycles Per Instruction | IPC 的倒数 |

---

*生成日期：2026-09-03 | 基于 CS6290 完整课程笔记*
