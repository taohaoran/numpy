# ufunc 内核循环层（ufunc-loops）

> 本文是 `core-ufunc` 域下的叶子子系统文档。域级总览见 `../core-ufunc.md`。
> 本文聚焦 ufunc 内层数值循环的模板代码生成、循环宏骨架、SIMD 内核特化，不展开对象层分发（见 `../ufunc-object/`）与匹配算法（见 `../ufunc-dispatch/`）。
>
> 源码基准：numpy commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 通用浮点循环 | 回调式 unary/binary 循环，按 dtype 模板实例化 | `umath/loops.c.src:43-73` |
| 浮点转型间接循环 | 半精度↔单精度↔双精度间转换调用 | `umath/loops.c.src:76-110` |
| 复数循环 | cfloat/cdouble/clongdouble 模板 | `umath/loops.c.src:117+` |
| 循环宏骨架 | UNARY_LOOP/BINARY_LOOP 等 stride 迭代宏 | `umath/fast_loop_macros.h:56-100` |
| 算术运算 SIMD | 加减乘除、取模的 SIMD 内核 | `umath/loops_arithmetic.dispatch.c.src`、`loops_modulo.dispatch.c.src` |
| 比较运算 SIMD | 大小比较的 SIMD 内核 | `umath/loops_comparison.dispatch.c.src` |
| 逻辑运算 | 逻辑与或非 | `umath/loops_logical.dispatch.cpp` |
| 三角/双曲 | sin/cos/tan/sinh/cosh/tanh | `umath/loops_trigonometric.dispatch.cpp`、`loops_hyperbolic.dispatch.cpp.src` |
| 指数对数 | exp/log/power 等 | `umath/loops_exponent_log.dispatch.c.src` |
| 复数运算 | 复数取实/虚/共轭等 | `umath/loops_unary_complex.dispatch.c.src`、`real_imag_ufuncs.cpp` |
| 半精度运算 | 16 位浮点内核 | `umath/loops_half.dispatch.c.src` |
| 极值 | minimum/maximum/fmax/fmin | `umath/loops_minmax.dispatch.c.src`、`minmax.cpp` |
| 矩阵乘 | matmul 内核 | `umath/matmul.c.src` |
| clip | 裁剪内核 | `umath/clip.cpp` |
| 浮点通用 | 快速路径浮点循环 | `umath/loops_umath_fp.dispatch.c.src`、`loops_arithm_fp.dispatch.c.src` |
| 模板工具头 | nomemoverlap、循环辅助 | `umath/loops_utils.h.src` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `PyUFuncGenericFunction` | 公共 API | 旧式循环签名 `(char** args, npy_intp* dims, npy_intp* steps, void* func)` |
| `PyArrayMethod_StridedLoop` | `array_method.h` | 新式 strided 循环：`(context, args, dimensions, strides, auxdata)` |
| `UNARY_LOOP` / `BINARY_LOOP` | `fast_loop_macros.h:64,89` | stride 迭代宏：展开 ip/op 指针与步长变量 |
| `NPY_SIMD` | `simd/simd.h` | SIMD 向量宽度宏，编译期按 ISA 定义 |
| `npyv_*` 内建函数 | `_simd/_simd.dispatch.c.src` | 跨平台 SIMD 内建抽象（load/store/add/mul 等） |
| `loops_utils.h` 内联 | `loops_utils.h.src` | `nomemoverlap` 重叠检测、循环辅助 |
| `PW_BLOCKSIZE` | `loops.c.src:32` | 两两求和分块大小 128 |

## 3. 关键调用链

### 3.1 标量回调式循环（loops.c.src）

1. `/**begin repeat**/` 模板按 dtype（e/f/d/g = half/float/double/longdouble）重复
2. 每个实例生成 `PyUFunc_@c@_@c@` 函数（:50）
3. 函数内 `UNARY_LOOP { in1 = *ip1; *op1 = f(in1); }` 逐元素调用传入的 C 数学函数
4. 步长由 `steps[]` 控制，支持非连续内存

### 3.2 SIMD dispatch 内核路径（loops_*.dispatch.c.src）

1. meson 构建时按 `meson_cpu` 配置编译 baseline 与 dispatch（AVX2/AVX512/NEON）多个版本
2. 每个 `.dispatch.c.src` 文件用 `/**begin repeat**/` 按 sfx（s8/s16/s32/s64/f32/f64）重复
3. SIMD 内核如 `simd_divide_by_scalar_contig_@sfx@`（loops_arithmetic.dispatch.c.src:56）：
   - 取 `vstep = npyv_nlanes_@sfx@` 向量长度
   - 循环 `for (; len >= vstep; ...)` 每次处理 vstep 个元素
   - 对齐/连续输入走 SIMD 快路径，否则回退标量
4. 运行时经 `NPY_CPU_HAVE(FEATURE)` 宏检测 CPU 特性选择最优版本（见 `../../core-common/simd/`）

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `NPY_SIMD` | 编译期定义 SIMD 向量宽度，0=无 SIMD | `common/simd/simd.h` |
| `MAX_STEP_SIZE` | 2MB；步长超过此值不走 gather/scatter SIMD | `fast_loop_macros.h:41` |
| `PW_BLOCKSIZE` | 128；两两求和分块 | `loops.c.src:32` |
| `NPY_BUFSIZE` | NpyIter 缓冲区默认大小 | extobj |
| meson_cpu baseline/dispatch | 构建时指定基线 ISA 与可分派 ISA 列表 | meson.build |

## 5. 错误与重试语义

- **浮点异常**：循环结束后由 `execute_ufunc_loop` 调用 `_check_ufunc_fperr` 检查硬件 FPU 状态（见 `../ufunc-error/`）。
- **内存重叠**：`nomemoverlap`（loops_utils.h.src）在 SIMD 内核前检测输入输出重叠，重叠时回退安全路径或经 `NPY_ITER_COPY_IF_OVERLAP` 复制。
- **对齐**：`fast_loop_macros.h` 中 `AUTOVEC_OVERLAP_SIZE` 用于对齐重叠检查；非对齐输入走标量路径。
- 无自动重试；错误经 Python 异常上抛。

## 6. 并发细节

- **GIL 释放**：标量与 SIMD 内核在 `execute_ufunc_loop` 中释放 GIL 执行（`NPY_BEGIN_THREADS_THRESHOLDED`），前提是 method 不要求 Python API。
- **无内部锁**：纯计算循环，无 mutex；数据竞争靠迭代器的 `COPY_IF_OVERLAP` 保证。
- **SIMD 线程安全**：纯向量计算，无共享可变状态。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `umath/loops.c.src`：通用标量回调循环模板
- `umath/loops_*.dispatch.c.src/.cpp`：各运算类 SIMD 内核模板
- `umath/fast_loop_macros.h`：循环宏
- `umath/loops_utils.h.src`：重叠检测辅助
- `umath/matmul.c.src`、`clip.cpp`、`minmax.cpp`、`real_imag_ufuncs.cpp`

**Out-of-Scope（不在本仓库源码内）**
- CPU ISA 特性检测与分派机制（见 `../../core-common/simd/`）
- SIMD 抽象层内建实现（`common/simd/`、`_simd/`，见 `../../core-common/simd/`）
- 数学库实现（见 `../../core-common/npymath/`）
- ufunc 对象层编排（见 `../ufunc-object/`）

## 8. 与相邻子系统交互

- **上游 → ufunc-object**：对象层经 `get_strided_loop` 获取本叶子提供的内核函数指针
- **上游 → ufunc-dispatch**：分发器选择 dtype 对应的内核注册到 `_loops`
- **下游 → simd**：SIMD 内核调用 `npyv_*` 内建函数，其实现按 ISA 编译多版本
- **下游 → npymath**：标量内核调用 `npy_*` 数学函数（如 `npy_sqrt`、`npy_log`）

## 9. 语言专项适配口径

1. **.src 模板代码生成**：所有 `loops.c.src`、`loops_*.dispatch.c.src` 为手写模板，经 C 预处理器 `/**begin repeat**/` 宏在构建时展开为多 dtype 实例化 `.c` 文件。**生成文件禁止手改**。
2. **两层分发**：本叶子实现第二层——loop 内部经 SIMD 按 CPU ISA（AVX2/AVX512/SSE/NEON）切换内核；第一层（dtype 选 loop）见 `../ufunc-dispatch/`。
3. **双轨实现**：每个运算类同时有标量回调式兜底（loops.c.src）与 SIMD 优化版（loops_*.dispatch），运行时按对齐/连续/ISA 条件选择。
4. **依赖方向**：`umath/` 依赖 `common/simd/` 与 `npymath/`，单向。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 架构图 | `ufunc-loops-architecture.html` | architecture | standard |

降档说明：含代码生成层与分派层 7 组件 7 连线，showcase 标签重叠/端点方向校验未全过，降 standard 档。数据流图（.src→生成→分派管道）因节点标签自动重叠反复未过校验，改用架构图表达，不再单独产出 dataflow。JSON IR 源文件位于 `json/` 目录。
