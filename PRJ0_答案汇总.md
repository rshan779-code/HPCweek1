# CS 6290 Project 0 — 答案汇总（中英对照）

> 红框里填**英文版**。中文版仅供你理解和核对，不要填进文档。

---

## 填空速查表

| 题 | 答案 |
|---|---|
| A | `54.240` 毫秒 ／ `10.650` 秒 |
| B | `94.72` ／ `2.74` ／ `97.26` ／ `94.57` |
| C | `76250392` ／ `9.44` |
| D | `9.44` ／ `5.28` ／ `0.50` ／ `9.2` |
| E | `80.49` ／ `94.74` ／ `94.72` |
| G | `4837` |
| I | `4835` |

D 的第三个空建议写成：`0.50 [9.44% × 5.28% = 0.4984%]`

---

## D 题解释

### 中文

误预测的代价不是"那一条分支指令白跑了"。分支的方向要等它在乱序核里真正执行、条件算出来才知道；在这之前，处理器已经沿着预测的那条路取指、重命名、发射了大量指令。一旦发现猜错，这些指令全部作废，流水线要清空重填。

配置文件 `cmp4-noc.conf` 给出 `issue = 2`，因而 `fetchWidth = issueWidth = retireWidth = 2`，所以每一个被作废的周期损失的是 **2 个发射槽**而不是 1 个。更关键的是 `BTACDelay = 0`，配置文件自己的注释写着 "unblock when execute"——恢复要等到分支真正执行完毕，惩罚值不是一个人为设定的常数，而是由乱序核的深度决定（整数指令窗口 `winSize = 12×2+32 = 56` 项，且 `bb4Cycle = 1` 每周期只能取一个基本块）。

观测数据与此自洽：9.2 ÷ 0.498 ≈ 18.5，即平均每次误预测浪费约 18.5 个指令发射机会，按每周期 2 个槽位折算约合 9 个周期的流水线重填。

### English

The cost of a misprediction is not the single branch instruction. A branch's direction is not known until it actually executes in the out-of-order core; before that, the processor has already fetched, renamed and issued many instructions along the predicted path. When the prediction turns out wrong, all of those are squashed and the pipeline must be refilled.

`cmp4-noc.conf` sets `issue = 2`, hence `fetchWidth = issueWidth = retireWidth = 2`, so each squashed cycle costs **2 issue slots rather than 1**. More importantly, `BTACDelay = 0`, which the configuration comments as "unblock when execute" — recovery waits until the branch actually executes, so the penalty is not a fixed constant but is set by the depth of the out-of-order core (integer window `winSize = 12×2+32 = 56` entries, and `bb4Cycle = 1` limits fetch to one basic block per cycle).

The measurements are consistent with this: 9.2 / 0.498 ≈ 18.5 instruction issue slots lost per misprediction, i.e. roughly 9 cycles of pipeline refill at 2 slots per cycle.

---

## E 题解释

### 中文

⚠️ **注意**：准确率并非单调上升。64→256 实际上微降了 0.02，答案必须处理这一点，否则与提交的报告数据自相矛盾。

准确率提升有两个机制。

**其一，预测器需要训练。** `cmp4-noc.conf` 的 `[BPredIssueX]` 一节给出 `l2size`、`localSize`、`Metasize` 各 2048 项，`historySize = 8`。这类混合预测器的每个表项都是 2 位饱和计数器，需要被同一条分支反复命中若干次才能收敛到正确方向，meta 选择器还需额外样本才能判断该信全局还是局部。n=16 时全程只执行约 9,741 次分支，平均每个表项不到 5 次更新，预测器整个生命周期都处在冷启动阶段——这正是 BPred 只有 77.89% 的直接原因。n=64 时约 77 次/表项，计数器已充分收敛，BPred 跳到 94.29%。

**其二，循环变长。** lu.C 的循环边界正比于 n（例如 `for (j=n-1; j>=0; j--)`、`for (i=0; i<j; i++)`）。循环回边分支每次迭代都 taken，只在退出时错一次，因此迭代次数越长，该分支的每一次出现被预测正确的概率就越高。

**关于 64→256 的微降**：BPred 那部分仍在提升（94.29% → 94.57%），总体从 94.74% 降到 94.72% 是因为由 RAS 处理的分支占比从 8.07% 降到 2.74%，而 RAS 是 100% 准确的那部分，其权重下降拉低了加权平均。到 n=64 时表容量已不再是瓶颈，训练带来的收益基本榨干，因此准确率饱和。

### English

Two mechanisms raise the per-branch prediction accuracy.

**First, the predictor needs training.** The `[BPredIssueX]` section of `cmp4-noc.conf` specifies `l2size`, `localSize` and `Metasize` of 2048 entries each with `historySize = 8`. Each entry is a saturating counter that must be updated repeatedly by the same branch before it converges, and the meta-selector needs further samples before it can tell whether to trust the global or the local predictor. At n=16 the entire run executes only about 9,741 branches — fewer than 5 updates per table entry on average — so the predictor never leaves its cold-start phase, which is why BPred is only 77.89%. At n=64 there are roughly 77 updates per entry, the counters have converged, and BPred rises to 94.29%.

**Second, the loops get longer.** The loop bounds in lu.C are proportional to n (e.g. `for (j=n-1; j>=0; j--)`, `for (i=0; i<j; i++)`). A backward loop branch is taken on every iteration and mispredicted only on exit, so the longer the trip count, the higher the probability that any single occurrence of that branch is predicted correctly.

**On the slight drop from 64 to 256:** BPred itself still improves (94.29% → 94.57%). The overall figure falls from 94.74% to 94.72% because the share of branches handled by the RAS drops from 8.07% to 2.74%; since the RAS is the 100%-accurate component, reducing its weight pulls the weighted average down. By n=64 table capacity is no longer the bottleneck and the training benefit is essentially exhausted, so accuracy saturates.

---

## H 题解释

### 中文

程序是静态链接的，执行并不从 `main` 开始。反汇编显示 `main`（0x401310–0x401368）只有 23 条指令，紧随其后就是 `__libc_start_main`（0x401370）。glibc 的启动流程在 `main` 之前完成进程初始化——重定位、TLS 设置、locale/ctype 表、stdio 初始化（包括为 stdout 分配缓冲区）——并在退出时 flush 缓冲区、发出 write 系统调用。

**证据**：用完全相同的编译选项和配置，仿真一个 `main` 为空的程序执行了 **3952** 条指令，占 hello 那 4837 条的 **81.7%**。也就是说只有约 885 条指令能归因于 printf 本身，而 `main` 自身仅 23 条。

### English

The program is statically linked and execution does not begin at `main`. The disassembly shows that `main` (0x401310–0x401368) is only 23 instructions and is immediately followed by `__libc_start_main` (0x401370). glibc's startup performs process initialization before `main` runs — relocation, TLS setup, locale/ctype tables, and stdio initialization including allocating the stdout buffer — and on exit it flushes the buffer and issues the write syscall.

**Evidence:** an otherwise identical program with an empty `main`, compiled with the same options and simulated with the same configuration, executes **3952** instructions — **81.7%** of the 4837 executed by hello. Only about 885 instructions are attributable to the printf itself, and just 23 to `main`.

---

## I 题解释

### 中文

**实验数据**

| 感叹号 | 字符串长度 | nInst | 变化 |
|---|---|---|---|
| 0 | 18 | 4837 | — |
| 1 | 19 | 4844 | +7 |
| 2 | 20 | 4851 | +7 |
| 3 | 21 | 4858 | +7 |
| 4 | 22 | **4835** | **−23** |

**解释**

由于格式串不含任何转换符且以 `\n` 结尾，编译器把 `printf` 替换成了 `_IO_puts`——反汇编中 `main` 的 0x401344 处是 `bal 402190 <_IO_puts>`，根本没有调用 printf。`_IO_puts` 做的第一件事是在 0x4021c4 调用 `strlen`。

MIPS 的 `strlen`（0x40d620）不是一个逐字节循环：

- 开头 `andi v0,a0,0x3` 检查指针是否 4 字节对齐，不对齐则先逐字节推进（0x40d648–0x40d65c）直到对齐
- 主循环在 0x40d674，每轮 `lw` 加载 4 个字节，用 `+0xfefefeff` / `nor` / `and 0x80808080` 这个惯用位运算检测该 word 内是否含零字节
- 一旦检测到零字节，0x40d694–0x40d6b8 处的四条 `lb` 逐字节定位它的确切位置，零字节落在第几个字节就执行几条

因此总指令数不随字符串长度单调增长，而是随长度模 4 呈锯齿状。前三次每加一个字符稳定增加 7 条指令，说明字符串仍停留在同一个对齐类别内；第四个感叹号把长度带到 22，改变了最后一个 word 内零字节的位置（从而改变收尾 `lb` 的执行条数），指令数因此降到 4835，比上一次少 23 条——与此前每字符 +7 的趋势方向相反。

另一个佐证：周期数反而从 82494 升到 85447，指令变少而周期变多，说明执行路径确实发生了切换，分支误预测的分布随之改变。

### English

The compiler replaces `printf` with `_IO_puts`, since the format string contains no conversion specifiers and ends in `\n` — the disassembly of `main` shows `bal 402190 <_IO_puts>` at 0x401344, with no call to printf at all. The first thing `_IO_puts` does is call `strlen` at 0x4021c4.

MIPS `strlen` (0x40d620) is not a byte-at-a-time loop:

- it first checks pointer alignment with `andi v0,a0,0x3` and, if unaligned, byte-steps (0x40d648–0x40d65c) until the pointer is word-aligned;
- the main loop at 0x40d674 loads 4 bytes per iteration with `lw` and detects a zero byte within the word using the `+0xfefefeff` / `nor` / `and 0x80808080` idiom;
- once a zero byte is detected, the four `lb` instructions at 0x40d694–0x40d6b8 locate it within the word, executing one to four of them depending on its position.

The total instruction count therefore does not grow monotonically with string length; it varies with length modulo 4. The first three additions each cost a constant 7 instructions (4837 → 4844 → 4851 → 4858), showing the string stayed within the same alignment class. The fourth exclamation mark brings the length to 22, changing the position of the terminating zero byte within the final word and hence the number of trailing `lb` instructions executed. The run drops to 4835 — 23 fewer than the previous run, in the opposite direction from the +7-per-character trend.

A corroborating detail: the cycle count rises from 82494 to 85447 even though the instruction count falls, confirming that the execution path changed and with it the distribution of branch mispredictions.

---

## 提交清单

| 文件 | 状态 |
|---|---|
| `PRJ0.docx` | 待填写 |
| `sesc_lu.mipseb.n16.rpt` | ✅ |
| `sesc_lu.mipseb.n64.rpt` | ✅ |
| `sesc_lu.mipseb.n256.rpt` | ✅ |
| `hello.c`（4 个感叹号那版） | ✅ |
| `sesc_hello.mipseb.hello0.rpt` | ✅ |
| `sesc_hello.mipseb.hello4.rpt` | ✅ |

不打包、单独上传、文件名一字不差。中间轮次的 hello1/2/3 报告不要交。
