# 遗留随机状态（legacy-mtrand）

> 本文是 `random` 域下的叶子子系统文档。域级总览见 `../random.md`。
>
> 源码基准：NumPy main 分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| RandomState 类 | 遗留随机数 API 主类，np.random 单例即此类实例 | `numpy/random/mtrand.pyx:121` |
| 遗留分布算法 | legacy_gauss/normal/pareto/weibull/standard_gamma/standard_t/exponential/power/gamma 等 | `mtrand.pyx:71-80`（extern 声明） |
| 遗留分布 C 实现 | legacy-distributions.c：旧版分布算法实现，保证向后兼容 | `numpy/random/src/legacy/legacy-distributions.c` |
| 增强 BitGenerator 状态 | aug_bitgen_t：在 bitgen_t 基础上增加 has_gauss/gauss 缓存字段 | `mtrand.pyx:64-69` |
| Binomial 状态结构 | s_binomial_t：Binomial 分布的缓存状态 | `mtrand.pyx:31-50` |

## 2. 核心类型与接口清单

| 类型/函数 | 位置 | 职责 |
|-----------|------|------|
| `RandomState`（cdef class） | `mtrand.pyx:121` | 遗留随机数 API，包装 MT19937 BitGenerator |
| `aug_bitgen_t`（C 结构体） | `mtrand.pyx:64` | 增强 BitGenerator：bitgen_t + has_gauss/gauss 缓存 |
| `legacy_gauss(aug_state)` | `legacy-distributions.c` | 旧版 Box-Muller 正态采样（成对，缓存第二个值） |
| `s_binomial_t`（C 结构体） | `mtrand.pyx:31` | Binomial 分布缓存状态 |

## 3. 关键调用链

### 3.1 np.random.randn() 调用链

1. 用户调用 `np.random.randn()` → 全局 RandomState 实例的 randn 方法
2. RandomState 内部持有 MT19937 BitGenerator 的 Capsule
3. 调用 `legacy_normal(aug_bitgen, loc, scale)`（C 函数）
4. legacy_normal 使用 Box-Muller 变换，通过 aug_bitgen->bit_generator 调用 MT19937 生成均匀随机数
5. 若 has_gauss=1，直接返回缓存的 gauss 值；否则计算一对并缓存第二个

## 4. 配置项

| 配置项 | 默认值/行为 | 位置 |
|--------|-------------|------|
| 内部位生成器 | MT19937（固定） | `mtrand.pyx:19` |
| aug_bitgen 缓存 | has_gauss/gauss 成对正态采样缓存 | `mtrand.pyx:64-68` |

## 5. 错误与重试语义

- 与 Generator 相同的拒绝采样机制
- 旧版分布算法保持与 NumPy 1.x 完全一致的输出（种子相同时）

## 6. 并发细节

- RandomState 内部使用 MT19937 BitGenerator 的 RLock
- aug_bitgen_t 的 gauss 缓存使旧版 normal 不是线程安全的（缓存状态竞争）
- 推荐使用 Generator + default_rng 替代

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `numpy/random/mtrand.pyx`：RandomState 类
- `numpy/random/src/legacy/legacy-distributions.c`：旧版分布 C 实现

**Out-of-Scope（不在本仓库源码内）**

- MT19937 位生成器——见 `bit-generators` 叶子
- 新分布算法——见 `generator-distributions` 叶子

## 8. 与相邻子系统交互

- **上游**：`np.random` 全局单例是 RandomState 实例
- **下游 → BitGenerator**：RandomState 包装 MT19937 Capsule
- **下游 → legacy C**：旧分布算法在 src/legacy/ 中

## 9. 语言专项适配口径

本叶子按「Python + Cython 混合项目专项分析清单」分析：

1. **遗留兼容层**：RandomState 是 NumPy 1.x 的遗留 API，内部复用新 BitGenerator 基础设施（MT19937），但分布算法保留旧版实现以保证向后兼容
2. **增强结构体**：aug_bitgen_t 在 bitgen_t 基础上增加 gauss 缓存——这是新旧架构混合的典型模式

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| legacy-mtrand 架构图 | `legacy-mtrand-architecture.html` | architecture | showcase |

- JSON IR 源文件：`json/legacy-mtrand-architecture.json`
- 降档原因：无
