# 位生成器（bit-generators）

> 本文是 `random` 域下的叶子子系统文档。域级总览见 `../random.md`。
>
> 源码基准：NumPy main 分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| BitGenerator 基类 | 定义随机位源抽象接口：random_raw、seed、state 读写、jump、advance | `numpy/random/bit_generator.pyx:506` |
| SeedSequence 种子序列 | 将用户种子（int/str/list）扩展为 32 位初始化向量池，来自 O'Neill 的 C++11 seed_seq | `bit_generator.pyx:261` |
| SeedlessSeedSequence | 无种子序列（自动随机播种） | `bit_generator.pyx:236` |
| MT19937 | 梅森旋转算法（2003 版），624 状态字 | `numpy/random/_mt19937.pyx`、`src/mt19937/mt19937.c` |
| PCG64 | PCG XSL-RR 128 位生成器，支持 jump/advance 子流分割 | `numpy/random/_pcg64.pyx`、`src/pcg64/pcg64.c` |
| Philox | 计数器型生成器（DaBiT 家族），可并行 | `numpy/random/_philox.pyx`、`src/philox/philox.c` |
| SFC64 | Small Fast Chaotic 生成器（Chris Doty-Perkins） | `numpy/random/_sfc64.pyx`、`src/sfc64/sfc64.c` |
| splitmix64 | PCG64 的辅助混合函数（用于种子初始化） | `src/splitmix64/splitmix64.c` |

## 2. 核心类型与接口清单

| 类型/函数 | 位置 | 职责 |
|-----------|------|------|
| `BitGenerator`（cdef class） | `bit_generator.pyx:506` | 抽象基类：持有 RLock、random_raw 接口、Capsule 包装 |
| `SeedSequence`（cdef class） | `bit_generator.pyx:261` | 种子序列：entropy 处理、spawn 子序列、generate 填充池 |
| `MT19937`（cdef class） | `_mt19937.pyx` | 梅森旋转，extern C 函数 mt19937_next64/next32/next_double |
| `PCG64`（cdef class） | `_pcg64.pyx` | PCG 生成器，extern pcg64_next64/set_seed/jump/advance |
| `Philox`（cdef class） | `_philox.pyx` | 计数器生成器 |
| `SFC64`（cdef class） | `_sfc64.pyx` | SFC 生成器 |
| `random_raw(size, output)` | `bit_generator.pyx:649` | 生成原始随机字节的核心接口 |

## 3. 关键调用链

### 3.1 BitGenerator 构造与播种

1. 用户 `np.random.default_rng(12345)` → 创建 PCG64 BitGenerator
2. 构造函数创建 SeedSequence(12345)
3. SeedSequence.entropy 经 `_coerce_to_uint32_array` 转换为 uint32 数组
4. SeedSequence.generate() 使用混合函数（INIT_A/MULT_A 等常量）扩展为 pool
5. 将 pool 传给 C 层的 `pcg64_set_seed(state, seed, inc)` 完成初始化

### 3.2 random_raw 生成

1. 用户或 Generator 调用 `bitgen.random_raw(size=1000)`
2. BitGenerator 获取 RLock
3. 调用 Cython 绑定的 C 函数（如 `pcg64_next64(state)`），nogil 释放 GIL
4. 填充输出数组后释放 RLock

## 4. 配置项

| 配置项 | 默认值/行为 | 位置 |
|--------|-------------|------|
| `pool_size`（SeedSequence） | 4（uint32 字数） | `bit_generator.pyx:56` |
| `spawn_key` | 子序列生成键 | `bit_generator.pyx:261` |
| RLock | 每个 BitGenerator 实例持有 | `bit_generator.pyx:41` |

## 5. 错误与重试语义

- **负种子**：`_int_to_uint32_array` 对负整数抛出 ValueError（`bit_generator.pyx:69`）
- **种子字符串解析**：十六进制（0x 前缀）和十进制数字串分别处理；无法解析时抛出 ValueError
- **jump/advance**：仅支持 jump 的生成器（MT19937/PCG64）在不支持时抛出 AttributeError

## 6. 并发细节

- **GIL 释放**：所有 C 随机数生成函数标注 `nogil`，在生成随机数时释放 GIL，允许多线程并行
- **RLock 保护**：BitGenerator 持有 `threading.RLock`，random_raw 在调用期间持有锁——**BitGenerator 本身是线程安全的**
- **Generator 线程安全**：Generator 包装 BitGenerator 的 Capsule，不直接持有锁；多线程使用同一 Generator 不安全
- **Capsule 所有权**：BitGenerator 通过 PyCapsule_New 将 C 状态指针暴露给 Generator，生命周期由 BitGenerator 管理

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `numpy/random/bit_generator.pyx`：BitGenerator 基类和 SeedSequence
- `numpy/random/_mt19937.pyx`、`_pcg64.pyx`、`_philox.pyx`、`_sfc64.pyx`：Cython 绑定
- `numpy/random/src/{mt19937,pcg64,philox,sfc64,splitmix64}/`：C 实现（含 LICENSE）

**Out-of-Scope（不在本仓库源码内）**

- 无外部依赖；所有 C 实现在仓库内 vendored

## 8. 与相邻子系统交互

- **上游**：`default_rng()` 工厂函数选择默认位生成器（PCG64）；用户直接实例化 MT19937/PCG64 等
- **下游 → Generator**：BitGenerator 的 Capsule 被传入 `Generator`，供分布采样使用（见 `generator-distributions` 叶子）
- **下游 → distributions**：C 分布采样函数通过 BitGenerator 的函数指针表调用 next64/next32/next_double

## 9. 语言专项适配口径

本叶子按「Python + Cython 混合项目专项分析清单」分析：

1. **绑定边界**：Cython `.pyx` 文件通过 `cdef extern from "src/xxx.h"` 声明 C 函数，`nogil` 释放 GIL 进行实际随机数生成。C 实现完全在 `src/` 下的 `.c`/`.h` 文件中
2. **位生成器抽象**：BitGenerator 是 cdef class 基类，定义统一接口（random_raw/seed/state/jump），子类提供不同算法。这是"策略模式"在随机数领域的应用——同一接口，不同算法（MT19937/PCG64/Philox/SFC64）
3. **函数指针表**：每个子类将 C 函数包装为 `void* → uint64_t` 等签名的函数指针（如 `mt19937_uint64`），注册到 BitGenerator 基类供分布采样通用调用

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| bit-generators 架构图 | `bit-generators-architecture.html` | architecture | showcase |

- JSON IR 源文件：`json/bit-generators-architecture.json`
- 降档原因：无
- 时序图：BitGenerator 构造与播种流程已在第 3 节文字化描述
