# 公共基础设施域（core-common）域总览

> 本域包含 NumPy 核心 C 层的公共基础设施：跨步循环、SIMD 分派与数学库三个叶子子系统；各叶子详情见对应文档。
> 源码基准：numpy commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 域职责

core-common 域是 NumPy 核心 C 层的公共基础设施，被 umath、multiarray 等上层模块依赖：

- **跨步循环**：提供任意 stride 的内存拷贝/转换内核与内存重叠检测（丢番图方程求解）
- **SIMD**：CPU 特性检测与跨平台 `npyv_*` 内建抽象，支撑 ufunc 内核的多 ISA 优化
- **npymath**：跨平台 C 数学库（实/复函数、IEEE-754、半精度、long double）

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序/数据流 | 职责一句话 |
|------|------|--------|-------------|------------|
| strided-loops | [strided-loops.md](strided-loops/strided-loops.md) | [架构图](strided-loops/strided-loops-architecture.html) | — | 跨步拷贝/转换、内存重叠检测 |
| simd | [simd.md](simd/simd.md) | [架构图](simd/simd-architecture.html) | — | CPU 特性检测与 SIMD 抽象分派 |
| npymath | [npymath.md](npymath/npymath.md) | [架构图](npymath/npymath-architecture.html) | — | 跨平台 C 数学库 |

## 3. 域级机制细节

### 依赖方向

core-common 是最底层公共库，被 umath（core-ufunc 域）与 multiarray 依赖：
`npymath`（纯数学）→ `simd`（向量抽象）→ `strided-loops`（内存操作）→ umath/multiarray。

### .src 模板代码生成

`lowlevel_strided_loops.c.src`、`npy_math.c.src`、`ieee754.c.src` 等 `.src` 文件用 C 预处理器 repeat 宏按 dtype 实例化，meson 构建时生成 `.c` 文件。**生成文件禁止手改**。

### 多后端 CPU 分派

simd 叶子实现"编译期多版本 + 运行期选择"：meson_cpu 配置 baseline/dispatch 目标，`npy_cpu_init` 经 CPUID/getauxval 检测，`NPY_CPU_HAVE` 宏运行时选最优内核。对应 Python+C++ 混合专项清单的两层分发机制。
