# 随机数域（random）域总览

> 本域包含 NumPy 随机数系统的 3 个叶子子系统；各叶子详情见对应文档。
> 源码基准：NumPy main 分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 域职责

本域实现 NumPy 的随机数系统，采用「BitGenerator + Generator」双层架构：BitGenerator 提供原始随机位源，Generator 在其上提供约 40 种概率分布采样。按「Python + Cython 混合项目专项分析清单」分析：Cython 绑定边界、位生成器抽象、分布采样算法、GIL 释放与线程安全。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 职责一句话 |
|------|------|--------|------------|
| bit-generators | [bit-generators.md](bit-generators/bit-generators.md) | [架构图](bit-generators/bit-generators-architecture.html) | 位生成器抽象层（BitGenerator 基类 + MT19937/PCG64/Philox/SFC64） |
| generator-distributions | [generator-distributions.md](generator-distributions/generator-distributions.md) | [架构图](generator-distributions/generator-distributions-architecture.html) | Generator 分布采样层（约 40 种概率分布） |
| legacy-mtrand | [legacy-mtrand.md](legacy-mtrand/legacy-mtrand.md) | [架构图](legacy-mtrand/legacy-mtrand-architecture.html) | 遗留 RandomState API（np.random 单例，向后兼容） |

## 3. 域级机制细节

- **双层架构**：BitGenerator（随机位源）→ Generator（分布采样），通过 PyCapsule 传递 C 状态指针
- **函数指针表**：`bitgen_t` 结构体包含 next_uint64/next_uint32/next_double 函数指针，不同位生成器注册不同实现
- **GIL 释放**：所有 C 随机数生成和分布采样函数标注 `nogil`，释放 GIL 允许多线程并行
- **种子序列**：SeedSequence 来自 O'Neill 的 C++11 seed_seq，将用户种子扩展为初始化向量池
- **模板代码生成**：`_bounded_integers.pyx.in` 经 meson 配置按位宽生成 `.pyx` 文件

## 4. 域级图

本域各叶子独立出图，无域级架构图。
