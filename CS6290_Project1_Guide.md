# CS 6290 Project 1 — 讲解 & 例题指南

---

## Part 1：读懂配置文件与源码

### 核心概念：Tournament（锦标赛）预测器

Tournament 预测器由三部分组成：

```
        PC地址
          │
    ┌─────┴──────┐
    │            │
    ▼            ▼
 简单预测器   历史预测器
 (Bimodal)   (Global/Local)
    │            │
    └─────┬──────┘
          │
     元预测器 (Meta)
     决定听谁的
          │
          ▼
       最终预测
```

---

### 如何读 BPred.cpp 中的 BPHybrid

打开文件后找 `BPHybrid` 的构造函数，大概长这样：

```cpp
BPHybrid::BPHybrid(int32_t i, const char *section, const char *sname)
{
  // 元预测器：metaSize 个条目，每个是 metaBits 位计数器
  metaTable   = new BPHistory(metaSize, metaBits);

  // 简单预测器（Bimodal）：bimodalSize 个条目，每个是 bimodalBits 位计数器
  bimodalTable = new BPBimodal(bimodalSize, bimodalBits);

  // 全局历史预测器：history 位历史，ghistSize 个 PHT 条目
  ghistTable  = new BPGlobal(history, ghistSize, ghistBits);
}
```

对应配置文件 `[BPredIssueX]` 中大概是：

```ini
[BPredIssueX]
type        = Hybrid
metaSize    = 1024      ← 元预测器条目数
metaBits    = 2         ← 元预测器每条目位数
bimodalSize = 2048      ← 简单预测器条目数（参数标签示例）
bimodalBits = 2         ← 简单预测器位数
history     = 11        ← 历史位数
ghistSize   = 2048      ← 历史预测器 PHT 条目数（参数标签示例）
ghistBits   = 2         ← 历史预测器每条目位数
```

> **做题方法：** 打开 BPHybrid 构造函数，看它从配置文件读哪些参数名（`SescConf->getInt(section, "参数名")`），参数名就是你要填的标签。

---

### 填空示例（假设值，实际以你读到的源码为准）

> **元预测器**：有 **1024** 个条目，每条目是 **2**-bit 计数器。

> 根据 PC 地址决定使用：简单预测器 还是 **全局（global）** 历史预测器。

> **简单预测器**：使用 **2**-bit 计数器，共 **2048** 个，参数标签为 **`bimodalSize`**。

> **历史预测器**：使用 **11** 位历史，PHT 共 **2048** 个条目，参数标签为 **`ghistSize`**，每条目 **2**-bit 计数器。

---

## Part 2：跑模拟器 & 填表计算

### Step 1：备份配置文件

```bash
cp ~/sesc/confs/cmp4-noc.conf ~/sesc/confs/cmp4-noc.conf.bak
```

### Step 2：编译 raytrace

```bash
cd ~/sesc/apps/Splash2/raytrace
make
```

### Step 3：依次运行三种预测器

```bash
# Hybrid（默认，不改配置）
cd ~/sesc/apps/Splash2/raytrace
~/sesc/sesc.opt -f HyA -c ~/sesc/confs/cmp4-noc.conf -ort.out -ert.err raytrace.mipseb -m128 Input/reduced.env

# Oracle（改 type="Oracle"）
~/sesc/sesc.opt -f OrA -c ~/sesc/confs/cmp4-noc.conf -ort.out -ert.err raytrace.mipseb -m128 Input/reduced.env

# NotTaken（改 type="NotTaken"）
~/sesc/sesc.opt -f NTA -c ~/sesc/confs/cmp4-noc.conf -ort.out -ert.err raytrace.mipseb -m128 Input/reduced.env
```

### Step 4：用 report.pl 读取结果

```bash
~/sesc/scripts/report.pl sesc_raytrace.mipseb.HyA
```

关注输出中的：
- `BPred` 行 → 整体预测准确率
- `nCycles` 行 → 总周期数

---

### 加速比计算示例

假设模拟后得到以下周期数（**仅为示例，非真实结果**）：

| 预测器 | 周期数 |
|--------|--------|
| NotTaken | 120,000,000 |
| Hybrid | 100,000,000 |
| Oracle | 85,000,000 |

**加速比 = 基准周期数 ÷ 当前周期数**（以 Hybrid 为基准）

```
NotTaken 加速比 = 100,000,000 ÷ 120,000,000 = 0.8333 X  （比 Hybrid 慢）
Hybrid   加速比 = 100,000,000 ÷ 100,000,000 = 1.0000 X  （基准）
Oracle   加速比 = 100,000,000 ÷  85,000,000 = 1.1765 X  （比 Hybrid 快）
```

> ⚠️ 加速比必须用**周期数**计算，精确到 **4 位小数**。

---

### Part C：修改 renameDelay

```bash
# 找到配置文件中 [issueX] 节
# 将 renameDelay = 1 改为 renameDelay = 7
nano ~/sesc/confs/cmp4-noc.conf
```

改完后再跑三次（`-f HyC`、`-f OrC`、`-f NTC`），记录新的周期数。

---

### Part D：为什么流水线越深，分支预测越重要？

**核心原因：**

```
流水线深度 = 分支预测错误惩罚（周期数）

renameDelay=1  → 预测错了，冲掉少数流水线阶段，损失小
renameDelay=7  → 预测错了，需要冲掉更多阶段，损失大
```

**示例答案：**
> 当流水线加深时（renameDelay 从 1 增加到 7），分支预测错误需要清空更多流水线阶段，每次错误惩罚的周期数增加。因此，较差的预测器（如 NotTaken）在深流水线中损失更多周期，而 Oracle 的优势也更加明显。这说明更好的分支预测在深流水线中**更加**重要。

---

### 估算分支预测惩罚周期数

**方法一：用 Oracle 和 Hybrid 的周期差 ÷ 错误次数**

```
惩罚 ≈ (Hybrid周期数 - Oracle周期数) ÷ Hybrid预测错误总次数
```

report.pl 输出中找 `BPred Mispred` 或类似字段获取错误次数。

**方法二：用 NotTaken 和 Oracle 的差异推算**

```
惩罚 ≈ (NotTaken周期数 - Oracle周期数) ÷ (NotTaken错误次数 - Oracle错误次数)
```

两种方法结果应接近，选择你有数据的那种解释即可。

---

## Part 3：修改源码统计分支预测

### 目标

对每条**静态**分支指令（由 PC 唯一标识），统计：
- `correct[PC]`：方向预测正确次数
- `wrong[PC]`：方向预测错误次数

### 修改 BPred.h — 添加数据结构

```cpp
// 在 BPHybrid 类的 private 部分添加：
#include <map>
#include <cstdint>

std::map<uint64_t, int64_t> correct_count;  // PC -> 正确次数
std::map<uint64_t, int64_t> wrong_count;    // PC -> 错误次数
```

### 修改 BPred.cpp — 在预测时记录

在 `BPHybrid::predict()` 函数中，找到判断预测是否正确的地方，添加统计：

```cpp
// 伪代码，具体变量名以源码为准
bool prediction = /* 预测结果 */;
bool actual     = /* 实际跳转结果 */;
uint64_t pc     = /* 当前指令PC */;

if (prediction == actual) {
    correct_count[pc]++;
} else {
    wrong_count[pc]++;
}
```

### 修改 BPred.cpp — 在仿真结束时输出

在析构函数或 `~BPHybrid()` 中添加输出：

```cpp
BPHybrid::~BPHybrid() {
    // 合并所有 PC 的统计
    std::map<uint64_t, std::pair<int64_t,int64_t>> stats;
    for (auto& kv : correct_count)
        stats[kv.first].first  = kv.second;
    for (auto& kv : wrong_count)
        stats[kv.first].second = kv.second;

    // 按完成次数分组统计
    int64_t g1=0, g2=0, g3=0, g4=0;          // 各组静态指令数
    int64_t c1=0, c2=0, c3=0, c4=0;          // 各组正确次数
    int64_t t1=0, t2=0, t3=0, t4=0;          // 各组总次数

    for (auto& kv : stats) {
        int64_t total   = kv.second.first + kv.second.second;
        int64_t correct = kv.second.first;
        if      (total < 20)   { g1++; c1+=correct; t1+=total; }
        else if (total < 200)  { g2++; c2+=correct; t2+=total; }
        else if (total < 2000) { g3++; c3+=correct; t3+=total; }
        else                   { g4++; c4+=correct; t4+=total; }
    }

    // 按题目顺序输出（不影响 report.pl）
    fprintf(stderr, "=== Branch Prediction Stats ===\n");
    fprintf(stderr, "Group 1-19:     count=%lld, accuracy=%.4f%%\n",
            g1, t1>0 ? 100.0*c1/t1 : 0.0);
    fprintf(stderr, "Group 20-199:   count=%lld, accuracy=%.4f%%\n",
            g2, t2>0 ? 100.0*c2/t2 : 0.0);
    fprintf(stderr, "Group 200-1999: count=%lld, accuracy=%.4f%%\n",
            g3, t3>0 ? 100.0*c3/t3 : 0.0);
    fprintf(stderr, "Group 2000+:    count=%lld, accuracy=%.4f%%\n",
            g4, t4>0 ? 100.0*c4/t4 : 0.0);
}
```

> ⚠️ 使用 `fprintf(stderr, ...)` 输出到标准错误，不影响 `report.pl` 读取的标准输出文件（sesc_raytrace.mipseb.*）。

### 运行并保存输出

```bash
# Hybrid 预测器（用原始配置）
~/sesc/sesc.opt -f HyA -c ~/sesc/confs/cmp4-noc.conf -ort.out -ert.err raytrace.mipseb -m128 Input/reduced.env 2> rt.out.Hybrid

# NotTaken 预测器
~/sesc/sesc.opt -f NTA -c ~/sesc/confs/cmp4-noc.conf -ort.out -ert.err raytrace.mipseb -m128 Input/reduced.env 2> rt.out.NT
```

输出示例（**假设值**）：

```
=== Branch Prediction Stats ===
Group 1-19:     count=342,  accuracy=61.23%
Group 20-199:   count=87,   accuracy=74.56%
Group 200-1999: count=43,   accuracy=88.91%
Group 2000+:    count=12,   accuracy=95.34%
```

---

### Part 3 分析题参考思路

**如果使用更大输入，准确率会怎么变？**

- **Hybrid**：准确率会**提升**。因为大输入下，大多数分支被执行更多次，预测器有更多机会"学习"规律，2-bit 饱和计数器趋于稳定，高频分支（2000+组）占主导。
- **NT（总预测不跳转）**：准确率**基本不变或略微变化**。NT 不学习，准确率只取决于程序中"不跳转"分支的比例，与输入规模关系不大。

---

## 常见错误提醒

| 错误 | 正确做法 |
|------|---------|
| 用 IPC 算加速比 | 必须用**周期数**（nCycles） |
| 加速比截断 3.1415 | 四舍五入为 3.1416 |
| 改了 stdout 导致 report.pl 读不到 | 输出统计用 `stderr` |
| 修改配置后忘记还原 | Part 3 前用备份文件恢复 |
| 提交时压缩了文件 | 每个文件单独上传，不打包 |

---

## 快速做题流程图

```
开始
 │
 ├─ Part 1：读源码 BPHybrid → 填空（无需跑模拟器）
 │
 ├─ Part 2：
 │   ├─ 备份 cmp4-noc.conf
 │   ├─ 跑 HyA / OrA / NTA → 填表A
 │   ├─ 改 renameDelay=7
 │   ├─ 跑 HyC / OrC / NTC → 填表C
 │   └─ 分析 D、E 题（用周期数推导）
 │
 └─ Part 3：
     ├─ 还原配置文件
     ├─ 修改 BPred.h + BPred.cpp
     ├─ 重新编译模拟器
     ├─ 跑 Hybrid + NT，输出到 rt.out.*
     └─ 填写分组统计 + 分析题
```
