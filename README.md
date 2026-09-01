# CS6290 - High Performance Computer Architecture 课程笔记

Georgia Tech CS6290 高性能计算机架构 中文学习笔记

---

## 📚 笔记目录

| 周次 | 文件 | 主题 |
|------|------|------|
| Week 1 | [Week1_Notes.md](Week1_Notes.md) | 课程简介 + 性能指标 + 流水线 + ILP |
| Week 2 | [Week2_Notes.md](Week2_Notes.md) | 分支预测 + 谓词执行 + ROB + 指令调度 |
| Week 3 | [Week3_Notes.md](Week3_Notes.md) | 编译器ILP + VLIW + Cache基础 + 高级Cache |
| Week 4 | [Week4_Notes.md](Week4_Notes.md) | 虚拟内存 + 内存技术 + 存储系统 + 容错 |
| Week 5 | [Week5_Notes.md](Week5_Notes.md) | 多处理器 + 缓存一致性 + 内存一致性 + 同步 + 众核 |

---

## 🔑 核心知识脉络

```
单核性能优化
├── 流水线（Pipelining）+ 冒险处理
├── ILP + 寄存器重命名 + 乱序执行
├── 分支预测（2位计数器/相关预测器/锦标赛）
├── ROB（精确异常 + 顺序提交）
└── 编译器优化（循环展开/软件流水/VLIW）

存储层次
├── Cache（局部性/3C/AMAT/多级Cache）
├── 虚拟内存（页表/TLB/缺页异常）
├── DRAM（SRAM vs DRAM/DDR/Bank交错）
└── 存储（HDD/SSD/RAID）

多核系统
├── 缓存一致性（MESI/监听/目录）
├── 内存一致性（SC/TSO/Memory Fence）
├── 同步（原子操作/自旋锁/Barrier）
└── 众核（NoC/目录协议/功耗管理）

可靠性
└── 容错（MTTF/ECC/RAID/检查点）
```

---

*笔记基于 CS6290 课程视频字幕整理 | 仅供学习参考*
