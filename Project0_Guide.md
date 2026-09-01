# CS6290 Project 0 讲解指南

> 基于 CS6290 课程知识点的 Project 0 详细解析

---

## 📌 整体目标

Project 0 是入门实验，主要做两件事：
1. **Part 1（60分）**：用 SESC 模拟器跑 LU 矩阵分解，分析**分支预测**准确率
2. **Part 2（40分）**：编译自己的 C 程序，分析为什么执行了那么多指令

---

## Part 1：分支预测实验

### 🔧 环境搭建

1. 安装 Oracle VirtualBox
2. 导入 `CS6290 Project VM.ova`
3. 在 VM 里编译模拟器：
```bash
cd ~/sesc
make
```

### 🚀 运行模拟命令

```bash
~/sesc/sesc.opt -fn256.rpt -c ~/sesc/confs/cmp4-noc.conf -olu.out -elu.err lu.mipseb -n 256 -p1
```

**参数说明：**

| 参数 | 含义 | 对应课程知识 |
|------|------|------|
| `-c cmp4-noc.conf` | 模拟器配置文件 | 定义分支预测器类型、Cache大小、ROB大小等 |
| `-n 256` | 矩阵大小 256×256 | 影响循环次数，从而影响分支预测热身 |
| `-p1` | 单核运行 | 多核在 Project 3 才用 |
| `-fn256.rpt` | 报告文件名后缀 | 保存模拟统计结果 |

> ⚠️ **重要**：每次运行前先删除旧报告文件，否则模拟器会追加内容导致数据错误

### 三次模拟命令

```bash
# 矩阵大小 16
~/sesc/sesc.opt -fn16.rpt -c ~/sesc/confs/cmp4-noc.conf -olu.out -elu.err lu.mipseb -n 16 -p1

# 矩阵大小 64
~/sesc/sesc.opt -fn64.rpt -c ~/sesc/confs/cmp4-noc.conf -olu.out -elu.err lu.mipseb -n 64 -p1

# 矩阵大小 256
~/sesc/sesc.opt -fn256.rpt -c ~/sesc/confs/cmp4-noc.conf -olu.out -elu.err lu.mipseb -n 256 -p1
```

> ⚠️ 每次模拟后检查 `lu.out` 和 `lu.err`，确认程序正常完成

### 📊 报告脚本解读

```bash
~/sesc/scripts/report.pl sesc_lu.mipseb.n256.rpt
```

**报告关键指标说明：**

| 指标 | 含义 | 对应课程知识 |
|------|------|------|
| **Total** | 整体分支预测命中率 | Week 2 分支预测 |
| **RAS 准确率** | Return Address Stack 准确率 | 函数返回地址预测 |
| **BPRed 准确率** | 方向预测器（非RAS）的准确率 | 2位计数器/锦标赛预测器 |
| **IPC** | 每周期指令数 | Week 1 ILP 概念 |
| **MisBr** | 因分支预测错误浪费的周期% | 分支惩罚代价 |
| **nInst** | 完成的动态指令总数 | — |

---

## ❓ 核心问题解析

### 问题 1：为什么 MisBr（预测错误浪费的周期%）远大于错误预测分支占所有指令的比例？

**知识点：分支惩罚（Branch Penalty）**

现代深流水线处理器（这个模拟器模拟的是类似 Pentium 4 的处理器，约 20 级流水线）：

- 一次预测错误 = 冲刷约 15-20 条错误指令 = 损失 **15-20 个周期**
- 因此：1% 的指令是错误预测分支 → 可能浪费高达 **15-20% 的周期**！

**计算举例：**
```
设：分支占所有指令的 15%
    分支预测准确率 98%（即 2% 预测错误）
    分支惩罚 = 15 周期

错误预测分支 = 15% × 2% = 0.3% 的指令
浪费周期 = 0.3% × 15 = 4.5% 的总周期
```

这就是为什么**分支预测准确率对 IPC 影响极大**，即使 1% 的预测错误也会造成明显性能损失。

---

### 问题 2：为什么矩阵越大（n=16→64→256），分支预测准确率越高？

**知识点：动态分支预测器的"热身"（Warmup）**

LU 分解是嵌套循环结构，其分支模式**非常规律**：

- 外层循环大多数迭代：**跳转**（继续循环）
- 最后一次迭代：**不跳转**（退出循环）

| 矩阵大小 | 循环次数 | 预测器状态 | 准确率 |
|------|------|------|------|
| n=16 | 较少 | 预测器还在"学习"，频繁遇到循环结束的错误预测 | 较低 |
| n=64 | 中等 | 预测器逐渐稳定 | 中等 |
| n=256 | 很多 | 2位饱和计数器达到稳定状态，只有极少数迭代末尾出错 | 最高 |

**关键理解**：
- 2位饱和计数器需要**2次相同结果**才改变预测
- 循环体执行越多次，末尾那1次错误的"占比"越小
- 这就是动态预测器依赖历史信息的体现

---

## Part 2：编译和指令分析

### 修改 hello.c

```c
#include <stdio.h>
int main(int argc, char *argv[]){
    printf("Hi! I am [你的名字]\n");
    return 0;
}
```

### 本地测试

```bash
gcc -o hello hello.c
./hello
```

### 交叉编译到 MIPS

```bash
/mipsroot/cross-tools/bin/mips-unknown-linux-gnu-gcc \
  -O0 -g -static -mabi=32 \
  -fno-delayed-branch \
  -fno-optimize-sibling-calls \
  -msplit-addresses -march=mips4 \
  -o hello.mipseb hello.c
```

**编译选项说明：**

| 选项 | 含义 |
|------|------|
| `-O0` | 不优化，保持代码可读性 |
| `-static` | 静态链接，不依赖动态库 |
| `-fno-delayed-branch` | 禁用 MIPS 分支延迟槽 |
| `-march=mips4` | 目标架构 MIPS IV |

### 运行模拟

```bash
~/sesc/sesc.opt -fhello0.rpt -c ~/sesc/confs/cmp4-noc.conf -ohello.out hello.mipseb
```

---

## ❓ Part 2 核心问题解析

### 问题 3：为什么 hello.c 这么简单，却执行了那么多指令？

**知识点：静态链接 + C 运行时启动代码**

程序启动不是直接跳到 `main()`，而是先执行 `__glibc_start_main`：

```
程序入口(_start) → __glibc_start_main → 初始化堆/栈/环境变量/信号处理 → main()
```

- 使用了 `-static`，包含完整 C 标准库（几百KB代码）
- 即使 main 只有一行，**启动代码**就需要执行数十万条指令
- 可以用 `objdump -d hello.mipseb` 查看汇编验证

### 问题 4：加感叹号到某个数量后，指令数反而减少了，为什么？

**知识点：字符串长度与内存对齐**

- `printf` 对于简单字符串实际上会被优化调用 `puts` 或直接调用 `write`
- 字符串存在内存中，底层 `puts/fwrite` 函数按字（4字节）或双字（8字节）复制数据
- 当字符串长度跨越对齐边界时，需要额外处理非对齐部分（更多指令）
- 当长度**恰好对齐**时，可以用高效的字对齐批量复制，指令数反而减少

**验证方法：**
```bash
# 查看汇编代码
mips-unknown-linux-gnu-objdump -d hello.mipseb | grep -A 50 "<main>"
```

---

## 📋 提交文件清单

**Part 1：**
- `sesc_lu.mipseb.n16.rpt`
- `sesc_lu.mipseb.n64.rpt`
- `sesc_lu.mipseb.n256.rpt`
- `PRJ0.docx`（填写答案后）

**Part 2：**
- `hello.c`（最终版本，带使指令数下降的感叹号数量）
- `sesc_hello.mipseb.hello0.rpt`（原始版本）
- `sesc_hello.mipseb.helloN.rpt`（指令数下降的那个版本）

---

## 🔑 相关课程知识点索引

| 问题 | 对应 Week | 核心概念 |
|------|------|------|
| 分支惩罚计算 | Week 2 | 流水线冲刷、分支延迟 |
| 预测器准确率提升 | Week 2 | 2位饱和计数器、预测器热身 |
| 指令数多的原因 | Week 1 | 动态指令数、静态vs动态代码 |
| 字符串长度与指令数 | Week 3 | Cache 对齐、内存访问效率 |

---

*基于 CS6290 课程内容整理 | 仅供学习参考，请勿直接抄袭作业答案*
