# HPCweek1
# CS 6290 (HPCA) 第一周内容梳理

> 覆盖 Lesson 1–3：Introduction / Metrics and Evaluation / Pipelining
> 说明：6290 的周–课时映射每学期略有调整，开课后请以 syllabus 为准。以下按课程标准顺序组织。

---

## Lesson 1 — Introduction

### 1.1 核心区分：ISA vs. Microarchitecture

| 层次 | 内容 | 变化频率 | 举例 |
|---|---|---|---|
| ISA（指令集架构） | 软件可见的契约：指令、寄存器、寻址模式、内存模型 | 极慢，向后兼容是硬约束 | x86-64、ARMv8、RISC-V |
| Microarchitecture（微架构） | ISA 的具体实现：流水线深度、缓存层次、分支预测器、乱序宽度 | 每代都变 | Zen 5、Golden Cove、Apple M 系列 |

**这门课几乎全部在讲 microarchitecture。** 反复出现的主线是：ISA 不变的前提下，如何让同一段二进制跑得更快。

### 1.2 技术趋势与"三堵墙"

Moore's Law 描述的是**晶体管密度**每约两年翻倍——它从来没有承诺性能翻倍。2005 年前后性能增长曲线拐弯，原因是三个约束同时收紧：

**Power Wall（最关键）**

动态功耗近似为：

$$P_{dynamic} \approx \frac{1}{2} \cdot C \cdot V^2 \cdot f$$

- $C$：开关电容，随晶体管数增加
- $V$：供电电压
- $f$：频率

Dennard Scaling 的原意是：晶体管变小时，$V$ 同比下降，于是单位面积功耗保持恒定。这条规律在约 65nm 后失效——$V$ 无法继续下降（阈值电压和漏电流限制），导致静态漏电功耗 $P_{static}$ 占比迅速上升。

结果：**频率停在 3–5 GHz，无法再靠提频拿性能。**

**ILP Wall**：单线程内可挖掘的指令级并行有限。加宽发射宽度的收益递减，而复杂度（重命名、调度、旁路网络）近似平方增长。

**Memory Wall**：处理器速度增长远快于 DRAM 延迟改善，访存代价（以周期计）持续放大。

### 1.3 行业应对

- **多核**：把晶体管预算花在核数上而非单核复杂度上
- **专用化**：GPU、NPU、各类加速器
- **能效优先**：性能指标从 "ops/s" 转向 "ops/joule"

> 你的嵌入式背景在这里是优势：功耗–性能权衡对你不是抽象概念。

---

## Lesson 2 — Metrics and Evaluation

**这是第一周最容易出考题的一课。** 概念简单但计算陷阱密集。

### 2.1 Iron Law of Performance（必背）

$$\text{CPU Time} = IC \times CPI \times T_{cycle} = \frac{IC \times CPI}{f}$$

| 项 | 含义 | 主要受什么影响 |
|---|---|---|
| $IC$ | 动态指令数 | ISA、编译器优化 |
| $CPI$ | 平均每指令周期数 | 微架构（流水线、缓存、分支预测） |
| $T_{cycle}$ | 时钟周期 | 工艺、流水线级数、关键路径 |

**核心陷阱：三项不独立。** 加深流水线可降低 $T_{cycle}$，但分支惩罚增大会推高 $CPI$。CISC 指令降低 $IC$，但可能推高 $CPI$ 或 $T_{cycle}$。任何只报告单项改进的结论都要追问另外两项。

### 2.2 Speedup 与 Latency/Throughput

$$\text{Speedup} = \frac{T_{old}}{T_{new}}$$

注意方向：性能是时间的倒数，速比用**旧时间除以新时间**。

**流水线的关键性质**：提高吞吐率，**不降低**单条指令的延迟——由于流水线寄存器开销，单指令延迟通常还略微上升。这是高频考点，也是直觉容易出错的地方。

### 2.3 Amdahl's Law

$$\text{Speedup}_{overall} = \frac{1}{(1 - F) + \dfrac{F}{S}}$$

- $F$：可被加速部分**在原始时间中的占比**
- $S$：该部分的局部加速比

上限：$S \to \infty$ 时，$\text{Speedup} \to \dfrac{1}{1-F}$

**worked example**

某程序 60% 时间在浮点运算。把浮点单元加速 3 倍：

$$\text{Speedup} = \frac{1}{0.4 + \frac{0.6}{3}} = \frac{1}{0.4 + 0.2} = \frac{1}{0.6} \approx 1.67$$

即便把浮点部分做到无限快：

$$\text{Speedup}_{max} = \frac{1}{0.4} = 2.5$$

**推论（Lhadma's Law）**：优化常见情况的同时，不要把不常见情况弄得过慢——否则 Amdahl 的分母会反噬你。

### 2.4 平均值的选择（高频陷阱）

| 场景 | 用哪种平均 | 理由 |
|---|---|---|
| 一组**执行时间** | 算术平均 | 时间可直接相加 |
| 一组**速率**（IPS、MFLOPS） | 调和平均 | 速率是时间的倒数 |
| 一组**归一化比率**（speedup、相对性能） | 几何平均 | 与基准选择无关 |

**为什么比率必须用几何平均**：算术平均处理归一化比率时，结果会随参考机器的选择而改变——同一组数据换个 baseline 就能得出相反结论。几何平均没有这个问题。

### 2.5 有问题的指标

- **MIPS**：不同 ISA 的指令做的事不同，跨架构无意义
- **MFLOPS**：只覆盖浮点，且不同浮点操作代价差异大
- **纯频率**：忽略 $CPI$ 和 $IC$

**SPEC benchmark** 存在的意义就是用真实程序集合规避上述问题。

---

## Lesson 3 — Pipelining

### 3.1 经典五级流水线

`IF → ID → EX → MEM → WB`

| 级 | 动作 |
|---|---|
| IF | 取指，PC 更新 |
| ID | 译码，读寄存器堆 |
| EX | ALU 运算 / 地址计算 / 分支判定 |
| MEM | 数据存储器访问 |
| WB | 写回寄存器堆 |

理想情况 $CPI = 1$，理想加速比 ≈ 级数。实际达不到，原因就是下面的 hazards。

$$CPI_{actual} = 1 + \text{stalls per instruction}$$

### 3.2 三类 Hazard

**Structural Hazard**——资源冲突

同一周期两条指令争用同一硬件单元。经典例子：单一存储端口时 IF 与 MEM 冲突。解法是加资源（分离 I-cache / D-cache）。

**Data Hazard**——数据依赖

| 类型 | 全称 | 性质 | 顺序流水线中是否出现 |
|---|---|---|---|
| RAW | Read After Write | 真依赖 | **会** |
| WAR | Write After Read | 假依赖（名字冲突） | 否 |
| WAW | Write After Write | 假依赖 | 否 |

**关键理解**：WAR 和 WAW 只在乱序执行中才成为问题，靠寄存器重命名消除。顺序五级流水线只需处理 RAW。这个区分在后续 Tomasulo 章节是基础。

**Forwarding（旁路）**：把 EX 或 MEM 阶段的结果直接送回 EX 的输入，绕过寄存器堆。能消除大部分 RAW 停顿。

**Load-Use Hazard**：唯一 forwarding 也救不了的情况——load 的数据到 MEM 结束才可得，而下一条指令 EX 阶段就要用。必须停 1 周期。编译器可通过指令调度把无关指令插入该槽位来填补。

**Control Hazard**——分支

分支结果在 EX 阶段才确定，此前已取入 2 条错误路径指令。处理手段递进：

1. 固定预测 not-taken，错了就 flush
2. Delayed branch（把分支后若干槽位交给编译器填充，MIPS 采用）
3. 动态分支预测（后续课程重点）
4. 提前分支判定电路（把比较逻辑挪到 ID 级，减少惩罚）

$$\text{Branch Penalty Cost} = \text{branch frequency} \times \text{misprediction rate} \times \text{penalty cycles}$$

### 3.3 深流水线的权衡

加深流水线 → $T_{cycle}$ 下降 → 频率上升，但：

- 分支预测错误惩罚随级数线性增长
- 流水线寄存器的建立/保持时间开销占比上升
- 冒险检测与旁路网络复杂度上升

存在最优深度，超过后总性能反而下降。Pentium 4 的 31 级流水线是教科书级的反面案例。

---

## 常见考试陷阱清单

1. **Speedup 方向搞反**——是 $T_{old}/T_{new}$，不是反过来
2. **Amdahl 中 $F$ 用错基准**——必须是**原始**总时间中的占比
3. **对 speedup 取算术平均**——必须用几何平均
4. **认为流水线降低单指令延迟**——它只提高吞吐率
5. **在顺序流水线里讨论 WAR/WAW**——那里不存在
6. **忽略 Iron Law 三项的耦合**——只报一项改进的方案通常有隐藏代价
7. **对速率取算术平均**——速率用调和平均

---

## 第一周实操建议

**优先级排序**（假设你要补一周进度）：

1. 把 Iron Law 和 Amdahl's Law 练到能盲写并做变式题——后续几乎每章的量化分析都建立在这两条上
2. 把 forwarding 和 load-use hazard 用时序图手画一遍，标出每条指令在每周期处于哪一级
3. Lesson 1 的技术趋势部分快速过，理解结论即可，不需要记数字

**与你已有背景的衔接**：
- CSE 6220 的并行加速比分析和 Amdahl 是同一套思路，你可以直接迁移
- 嵌入式经验对功耗墙、流水线深度权衡的理解有直接帮助
- CS 6291 的 VLIW 调度本质上是"用编译器静态填补 hazard 槽位"，和这里的 delayed branch / load-use 调度是同一个问题的两种解法——两门课在这里会互相强化

**参考书**：Hennessy & Patterson, *Computer Architecture: A Quantitative Approach*，第 1 章（趋势与量化原理）和附录 C（流水线）正好对应本周内容。
