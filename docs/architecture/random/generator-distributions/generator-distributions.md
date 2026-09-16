# 生成器与分布采样（generator-distributions）

> 本文是 `random` 域下的叶子子系统文档。域级总览见 `../random.md`。
>
> 源码基准：NumPy main 分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| Generator 类 | 包装 BitGenerator Capsule，提供约 40 种概率分布采样方法 | `numpy/random/_generator.pyx:142` |
| 均匀分布 | random（[0,1) 浮点）、integers（有界整数）、bytes、choice | `_generator.pyx:301,584,711,747` |
| 正态分布 | normal、standard_normal | `_generator.pyx:1195,1123` |
| 指数/Gamma 族 | exponential、standard_exponential、gamma、standard_gamma | `_generator.pyx:444,524,1391,1300` |
| Beta/χ²/F | beta、chisquare、noncentral_chisquare、f、noncentral_f | `_generator.pyx:363,1647,1728,1470,1572` |
| 其他分布 | vonmises、pareto、weibull、standard_t、standard_cauchy 等 | `_generator.pyx:1978,2062,2137,1871,1805` |
| 洗牌/排列 | shuffle、permutation、permutation | `_generator.pyx`（_shuffle_raw） |
| 有界整数采样 | _bounded_integers.pyx.in：按位宽特化的拒绝采样（uint8/16/32/64） | `numpy/random/_bounded_integers.pyx.in` |
| C 分布实现 | distributions.c、logfactorial.c、random_hypergeometric.c、random_mvhg_*.c | `numpy/random/src/distributions/` |

## 2. 核心类型与接口清单

| 类型/函数 | 位置 | 职责 |
|-----------|------|------|
| `Generator`（cdef class） | `_generator.pyx:142` | 分布采样主类，持有 BitGenerator Capsule |
| `bitgen_t`（C 结构体） | `numpy/random/__init__.pyx` | BitGenerator 状态指针，含 next_uint64/next_uint32/next_double 函数指针 |
| `_shuffle_raw(bitgen, n, ...)` | `_generator.pyx:75` | Fisher-Yates 洗牌，nogil |
| `random_bounded_uint64(bitgen, off, rng, mask, ...)` | `_bounded_integers.pyx.in` | 有界 64 位整数拒绝采样 |
| `spawn(n_children)` | `_generator.pyx:241` | 生成 n 个子 Generator（使用 SeedSequence.spawn） |

## 3. 关键调用链

### 3.1 rng.normal(size=100) 调用链

1. 用户调用 `rng.normal(size=100)` → `Generator.normal`（`_generator.pyx:1195`）
2. 内部调用 `standard_normal(size=size, dtype=np.float64, out=None)`（`_generator.pyx:1123`）
3. standard_normal 使用 Ziggurat 算法（method='zig'）调用 C 函数 `standard_zig(bitgen, ...)`
4. C 函数通过 `bitgen->next_uint64` 函数指针调用 BitGenerator 的 C 随机数生成
5. 填充输出 ndarray 返回

### 3.2 rng.integers(low, high, size) 调用链

1. 用户调用 `rng.integers(0, 100, size=10)` → `Generator.integers`（`_generator.pyx:584`）
2. 根据 dtype 选择对应位宽的采样函数（_rand_int32/_rand_int64 等）
3. _bounded_integers 使用拒绝采样：生成均匀 uint64 → 模运算 → 拒绝越界样本
4. nogil 释放 GIL，批量填充输出数组

## 4. 配置项

| 配置项 | 默认值/行为 | 位置 |
|--------|-------------|------|
| `method`（standard_exponential） | 'zig'（Ziggurat），也支持 'inv'（逆变换） | `_generator.pyx:524` |
| `endpoint`（integers） | False（半开区间 [low, high)） | `_generator.pyx:584` |
| `POISSON_LAM_MAX` | Poisson 分布最大 λ 值 | `_common.pyx` |

## 5. 错误与重试语义

- **拒绝采样**：有界整数采样使用拒绝采样，越界样本被丢弃并重新生成——天然重试机制
- **choice 概率校验**：p 数组求和不为 1 时抛出 ValueError
- **Poisson λ 上限**：λ 超过 POISSON_LAM_MAX 时使用正态近似
- **Kahan 求和**：Gamma 等分布的求和使用 Kahan 补偿求和减少浮点误差

## 6. 并发细节

- **GIL 释放**：所有 C 分布采样函数标注 `nogil`，在生成随机数时释放 GIL
- **线程安全**：Generator 本身不持有锁，依赖 BitGenerator 的 RLock——**多线程共享同一 Generator 不安全**
- **spawn 子生成器**：`spawn(n)` 通过 SeedSequence.spawn 生成独立的子种子序列，每个子 Generator 有独立状态，可安全并行

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `numpy/random/_generator.pyx`：Generator 类和分布采样
- `numpy/random/_bounded_integers.pyx.in`：有界整数采样模板
- `numpy/random/src/distributions/`：C 分布实现

**Out-of-Scope（不在本仓库源码内）**

- BitGenerator C 实现——见 `bit-generators` 叶子
- 无外部依赖

## 8. 与相邻子系统交互

- **上游**：用户通过 `default_rng()` 获得 Generator 实例；`np.random.Generator` 暴露为公开 API
- **下游 → BitGenerator**：Generator 持有 BitGenerator Capsule，通过 `bitgen_t` 函数指针调用随机数生成
- **下游 → C 分布**：分布算法在 `src/distributions/` C 代码中实现

## 9. 语言专项适配口径

本叶子按「Python + Cython 混合项目专项分析清单」分析：

1. **绑定边界**：Generator 是 Cython cdef class，通过 PyCapsule 接收 BitGenerator 状态。分布算法在 C 层（src/distributions/），通过 `c_distributions.pxd` 声明 C 函数接口
2. **模板代码生成**：`_bounded_integers.pyx.in` 是 Cython 模板文件，经 meson 配置按位宽生成 `_bounded_integers.pyx`——这是"代码生成产物分层"在 Cython 中的体现
3. **函数指针表**：`bitgen_t` 结构体包含 next_uint64/next_uint32/next_double 函数指针，不同 BitGenerator 注册不同实现——这是两层分发中的轻量函数指针表

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| generator-distributions 架构图 | `generator-distributions-architecture.html` | architecture | standard |

- JSON IR 源文件：`json/generator-distributions-architecture.json`
- 降档原因：standard 档通过渲染；showcase 因边布局校验未全过
- 时序图：分布采样调用链已在第 3 节文字化描述
