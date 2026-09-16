# build-meson（build-meson）

> 本文是 `support` 域下的叶子子系统文档。域级总览见 `../support.md`。
> 本文只展开 NumPy 的 meson 构建系统与 CPU 分派配置，不重复展开其他支撑子系统。
>
> 源码基准：numpy 主分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 顶层构建脚本 | meson 构建入口，编排全部子目录 | `meson.build` |
| 构建选项声明 | BLAS/LAPACK、线程、SIMD、CPU 分派选项 | `meson.options` |
| CPU 特性检测 | 解析 cpu-baseline/cpu-dispatch，生成 main_config.h | `meson_cpu/meson.build` |
| 架构特定配置 | x86/arm/ppc64/riscv64/s390x/loongarch64 各自特性集 | `meson_cpu/x86/`、`arm/`、`ppc64/` 等 |
| 分派宏模板 | main_config.h 模板，含运行期分派宏 | `meson_cpu/main_config.h.in` |
| Python 构建辅助 | 代码生成调用、版本探测 | `numpy/_build_utils/` |
| vendored meson | 构建系统副本 | `vendored-meson/`（第三方 vendored） |

## 2. 核心类型与接口清单

| 项 | 位置 | 职责 |
|------|------|------|
| `option(...)` 集合 | `meson.options` | 声明全部可调构建选项及默认值 |
| `CPU_BASELINE / CPU_DISPATCH_NAMES` | `meson_cpu/meson.build` | 暴露给后续构建的已启用特性集 |
| `CPU_CONF_PREFIX = 'NPY_'` | `meson_cpu/meson.build` | 分派宏统一前缀 |
| `main_config.h.in` | `meson_cpu/main_config.h.in` | 配置头模板，由 meson 变量填充 |
| `gitversion.py` | `numpy/_build_utils/gitversion.py` | 从 git 元数据推导版本 |

## 3. 关键调用链

1. 用户 `pip install .` / `meson setup build` → 读 `meson.build` 顶层。
2. meson 读 `meson.options`，取 `blas/lapack`、`cpu-baseline=min`、`cpu-dispatch=max` 等。
3. `meson_cpu/meson.build` 解析选项 → 按目标架构（x86/arm/...）确定 `CPU_BASELINE` 与 `CPU_DISPATCH`。
4. 用 `main_config.h.in` 渲染出含 `NPY_*` 分派宏的 `main_config.h`。
5. `_build_utils` 代码生成器（见 code-gen 叶子）产出 C 源，连同 `_core` 源交 C/Cython 编译器多目标编译。

## 4. 配置项（meson.options 重点）

| 选项 | 默认 | 行为 |
|------|------|------|
| `blas` / `lapack` | `auto` | 按 `blas-order`/`lapack-order` 自动探测 |
| `blas-order` / `lapack-order` | `['auto']` | 优先搜索顺序（mkl/openblas/blis...） |
| `allow-noblas` | true | 无 BLAS 时用慢速内置回退 |
| `use-ilp64` | false | 用 ILP64（64 位整型）接口 |
| `mkl-threading` | auto | MKL 线程后端（seq/iomp/gomp/tbb） |
| `disable-threading` | false | 关闭线程支持 |
| `enable-openmp` | false | 启用 OpenMP |
| `cpu-baseline` | `min` | 最低必需 CPU 特性集 |
| `cpu-dispatch` | `max` | 运行期额外分派的特性集 |
| `disable-optimization/svml/highway` | false | 关闭各类 CPU 优化 |
| `test-simd` | 各架构特性列表 | SIMD 测试模块覆盖的特性 |

## 5. 错误与重试语义

- **找不到 BLAS**：`allow-noblas=true` 时降级内置回退并告警；否则构建失败。
- **CPU 特性不支持**：meson_cpu 对不识别的特性集报错。
- 无运行期重试；构建期错误即终止。

## 6. 并发细节

- meson 配置/脚本为声明式构建描述，无运行期线程模型。
- 实际 C 编译由 meson 按 `--parallel` 并行调度（外部）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `meson.build`、`meson.options`、`meson_cpu/`、`numpy/_build_utils/`、`tools/`。

**Out-of-Scope（不在本仓库源码内）**
- `vendored-meson/`——第三方 vendored 构建系统副本，不在本项目原创源码内。
- C/Cython/Fortran 编译器、BLAS/LAPACK 库——外部工具。

## 8. 与相邻子系统交互

- **上游**：开发者/CI 通过 pip/meson 触发。
- **本叶子 → 下游**：调用 `code-gen` 叶子的代码生成器、`f2py` 生成 Fortran 接口、外部编译器。
- 与 `code-gen`（生成 C 源）、`f2py`（生成 Fortran 包装）共同构成构建链。

## 9. 语言专项适配口径（纯 Python / 构建系统）

- **配置驱动分派**：`cpu-baseline`/`cpu-dispatch` 选项驱动"同一 C 源按多特性集编译多目标、运行期分派"，是配置驱动静态多态在构建层的体现。
- **多后端矩阵**：`meson_cpu/<arch>` 按 x86/arm/ppc64/riscv/s390x/loongarch 分别声明特性集，是 CPU 后端矩阵。
- **代码生成衔接**：meson 调用 `_build_utils` 与 `code_generators`，是构建期代码生成的编排层。
- **vendored 标注**：vendored-meson 为第三方副本，标注外部。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| 架构图 | `build-meson-architecture.html` | architecture | standard |

- 降档原因：cpu→arch 竖边穿 buildutil、竖边标签重叠，去边/`labelDy` 后落 standard；render 退出码 0、HTML 约 805KB。
- 未补时序/数据流图：构建系统为声明式配置，流程图价值有限，用第 3 节步骤化文字表达。
- JSON IR 源文件：`json/build-meson-architecture.json`。
