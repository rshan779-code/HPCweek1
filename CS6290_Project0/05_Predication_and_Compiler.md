# 📘 CS6290 Project 0 — 专题 5：谓词执行与编译器优化

> 分支的"软件解法"：用编译器手段减少或消除分支。

---

## 1. 谓词执行（Predication / If-Conversion）

### 问题：难以预测的分支

有些分支几乎是随机的（50/50），任何预测器都无能为力：

```c
if (x > 0) {
    y = a * b;      // 分支 "taken"
} else {
    y = c + d;      // 分支 "not taken"
}
// x 是运行时输入，完全随机
```

对这种分支：预测失误率 ≈ 50% → 代价极大

---

### 解决思路：两条路都执行，用条件选择结果

```asm
# 传统方法（有分支）：
CMP  x, 0
BLE  else_label      ← 预测可能错！
MUL  y, a, b
JMP  end
else_label:
ADD  y, c, d
end:

# Predication 方法（无分支）：
MUL  temp1, a, b     ← 执行 if 分支
ADD  temp2, c, d     ← 执行 else 分支
CMP  x, 0
CMOVG y, temp1       ← 若 x>0，y = temp1（条件移动）
CMOVLE y, temp2      ← 若 x≤0，y = temp2
```

**关键**：两条路都执行，用**条件移动（CMOV）**选择结果，完全**无分支**！

---

### Predication 的权衡

| 场景 | 推荐方法 | 原因 |
|------|---------|------|
| 短 if-else，难预测（50/50）| Predication | 避免 50% 失误代价 |
| 短 if-else，易预测（95%+）| 分支预测 | 省去无用执行 |
| 长 if-else（50+ 条指令）| 分支预测 | 两边都跑代价太高 |
| 循环分支 | 分支预测（动态）| 循环预测准确率很高 |

**计算盈亏平衡点**：

```
Predication 代价 = 执行 if + else 两边的指令数
                 = N_if + N_else 额外指令

分支预测代价 = 失误率 × 失误惩罚（周期）

若 N_if + N_else < 失误率 × 失误惩罚 → 用 Predication
```

**例题**：
```
if-else 各 5 条指令（N=10），失误惩罚 15 周期
Predication 额外代价：10 - 5 = 5 条多余指令 ≈ 5 周期
分支预测代价（50% 失误率）：0.5 × 15 = 7.5 周期

7.5 > 5 → Predication 更好！
```

```
if-else 各 20 条指令（N=40），失误惩罚 15 周期
Predication 额外代价：40 - 20 = 20 条多余指令 ≈ 20 周期
分支预测代价（50% 失误率）：0.5 × 15 = 7.5 周期

7.5 < 20 → 分支预测更好！
```

---

### 硬件支持

| 架构 | Predication 机制 |
|------|----------------|
| **ARM（32位）** | 每条指令有 4 位条件码，可条件执行（`ADDEQ`, `MOVNE`） |
| **x86** | `CMOVcc`（条件移动），`SETcc`（条件设置） |
| **Intel Itanium（IA-64）** | 完整 predicate 寄存器（64个），任意指令可 predicate |
| **GPU（SIMT）** | Warp 内所有线程都执行，用 mask 屏蔽不活跃线程 |

**ARM 条件执行示例**：

```asm
# C代码：if (r0 == 0) r1 = r2 + r3;
CMP   r0, #0        # 设置条件标志
ADDEQ r1, r2, r3    # 只有 Equal 时才执行

# C代码：if (r0 > 0) r1 = r2; else r1 = r3;
CMP   r0, #0
MOVGT r1, r2        # r0>0 时：r1=r2
MOVLE r1, r3        # r0≤0 时：r1=r3
```

---

## 2. 编译器 ILP 优化

### 2.1 树高度降低（Tree Height Reduction）

改变计算树的结构，减少关键路径长度（即依赖链长度）

**示例**：计算 `a + b + c + d`

```
左结合（默认）：依赖链长度 = 3
  t1 = a + b    # 周期1
  t2 = t1 + c   # 周期2（等 t1）
  t3 = t2 + d   # 周期3（等 t2）

平衡树：依赖链长度 = 2
  t1 = a + b    # 周期1
  t2 = c + d    # 周期1（可与 t1 并行！）
  t3 = t1 + t2  # 周期2（等 t1、t2）

→ 节省 1 个周期（33% 更快）！
```

**注意**：浮点加法不满足结合律，编译器需要 `-ffast-math` 或类似选项才能做这种变换。

---

### 2.2 循环展开（Loop Unrolling）

**原始循环**：

```c
for (int i = 0; i < 100; i++) {
    A[i] = B[i] + C[i];
}
```

```asm
loop:
  LOAD  r1, B[i]
  LOAD  r2, C[i]
  ADD   r3, r1, r2    ← 等 LOAD（load-use stall！）
  STORE A[i], r3
  ADD   i, i, 1
  CMP   i, 100
  BLT   loop          ← 每次迭代一个分支
```

**展开 4 次**：

```c
for (int i = 0; i < 100; i += 4) {
    A[i]   = B[i]   + C[i];
    A[i+1] = B[i+1] + C[i+1];
    A[i+2] = B[i+2] + C[i+2];
    A[i+3] = B[i+3] + C[i+3];
}
```

**好处**：
1. 减少循环控制开销（4次迭代只有1个分支/计数）
2. 暴露更多 ILP（4组独立操作可并行）
3. 编译器可以重排指令，隐藏 load 延迟：

```asm
# 展开后重排，隐藏 load 延迟：
LOAD  r1, B[i]        # 提前 load i
LOAD  r2, B[i+1]      # 提前 load i+1
LOAD  r3, B[i+2]      # 提前 load i+2
LOAD  r4, B[i+3]      # 提前 load i+3
LOAD  r5, C[i]
LOAD  r6, C[i+1]
ADD   r7, r1, r5      # 此时 r1, r5 已就绪，无 stall！
LOAD  r8, C[i+2]
ADD   r9, r2, r6
LOAD  r10, C[i+3]
ADD   r11, r3, r8
STORE A[i], r7
ADD   r12, r4, r10
STORE A[i+1], r9
STORE A[i+2], r11
STORE A[i+3], r12
```

**代价**：代码体积增大，寄存器压力增大

---

### 2.3 软件流水线（Software Pipelining）

将**相邻迭代**的指令交错，让每个阶段都有工作做：

```
普通：
  迭代1: Load → (等待) → Add → Store
  迭代2: Load → (等待) → Add → Store
  
软件流水线：
  周期1: 迭代1 Load
  周期2: 迭代2 Load | 迭代1（等待）
  周期3: 迭代3 Load | 迭代2（等待）| 迭代1 Add
  周期4: 迭代4 Load | 迭代3（等待）| 迭代2 Add | 迭代1 Store
  ...
  → 稳态时每周期完成1次迭代，无 stall！
```

---

### 2.4 指令调度（Software Instruction Scheduling）

编译器重排指令顺序，减少流水线 stall：

**原始**（有 load-use stall）：

```asm
LOAD  r1, 0(r2)     # 周期1
ADD   r3, r1, r4    # 周期2（等 r1！stall 1 周期）
LOAD  r5, 4(r2)     # 周期4
ADD   r6, r5, r7    # 周期5（等 r5！stall 1 周期）
```

**调度后**（消除 stall）：

```asm
LOAD  r1, 0(r2)     # 周期1：提前 load
LOAD  r5, 4(r2)     # 周期2：提前第二个 load（与第一个无依赖！）
ADD   r3, r1, r4    # 周期3：r1 已就绪（load在周期2结束，无stall）
ADD   r6, r5, r7    # 周期4：r5 已就绪
```

→ 从 6 个周期变成 4 个周期，快 33%！

---

## 3. VLIW（Very Long Instruction Word）

### 原理

编译器静态决定哪些操作并行，打包成一条长指令：

```
一条 VLIW 指令（每个槽位对应一个执行单元）：

[ALU1 op][ALU2 op][FP op][Load op][Store op][Branch op]
   加法      减法    乘法    取数      存数       分支
```

处理器每周期执行这整条"宽指令"，完全按编译器安排并行。

### VLIW vs 超标量对比

| 特性 | 超标量（OOO） | VLIW |
|------|-------------|------|
| ILP 发现 | 硬件动态（乱序执行）| 编译器静态 |
| 运行时适应 | 能（处理缓存缺失等）| 不能 |
| 硬件复杂度 | 高（ROB、RS、重命名）| 低 |
| 功耗 | 高 | 低 |
| 二进制兼容 | 好 | 差（改硬件要重编译）|
| 代码体积 | 小 | 大（NOP 填充）|
| 典型用途 | 通用CPU | DSP、嵌入式、GPU |

**VLIW 的致命弱点**：

```
if (cache miss) {
    // 编译器无法预测 cache miss 时间
    // 静态调度假设的延迟不准确
    // 导致：要么加大量 NOP（安全但慢），要么出错
}
```

---

## 4. 总结：分支处理策略对比

| 策略 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **动态分支预测** | 大多数情况 | 自适应，准确率高 | 错误时代价高 |
| **Predication** | 短小、难预测的 if-else | 完全无分支失误 | 多执行无用指令 |
| **循环展开** | 规则循环 | 减少分支，暴露ILP | 代码膨胀 |
| **软件流水线** | 有延迟的规则循环 | 最大化利用执行单元 | 编译器复杂 |
| **VLIW** | 固定负载的嵌入式 | 硬件简单 | 不灵活，兼容性差 |

---
*CS6290 Project 0 专题笔记 | 生成日期：2026-09-03*
