# npymath 数学库（npymath）

> 本文是 `core-common` 域下的叶子子系统文档。域级总览见 `../core-common.md`。
> 本文聚焦 NumPy 跨平台 C 数学库：实/复数学函数、IEEE-754 低层操作、半精度与 longdouble 支持，是 ufunc 内核的计算基础。
>
> 源码基准：numpy commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 实数学函数 | sqrt/log/exp/sin/cos/tan/pow 等 C99 函数的跨平台封装 | `npymath/npy_math.c.src`、`npy_math_internal.h.src` |
| 复数数学函数 | csqrt/cexp/clog/cpow/cabs 等，源自 FreeBSD msun 库 | `npymath/npy_math_complex.c.src` |
| IEEE-754 低层 | nextafter/nextafterf/frexp/ldexp 等位级操作 | `npymath/ieee754.c.src` |
| 半精度转换 | npy_half↔float/double 转换与分类 | `npymath/halffloat.cpp` |
| longdouble | long double 数学函数封装 | `common/npy_longdouble.c` |
| 导出符号 | `npy_math.c` 关闭内联，生成外部可见符号 | `npymath/npy_math.c` |
| 通用头 | `npy_math_common.h`、`npy_math_private.h` | `npymath/` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `npy_half` | `numpy/halffloat.h` | 16 位半精度浮点类型（uint16 位模式） |
| `npy_cfloat`/`npy_cdouble`/`npy_clongdouble` | ndarraytypes.h | 复数类型 |
| `npy_half_to_float` 等 | `halffloat.cpp` | 半精度转换函数 |
| `npy_c*` 函数族 | `npy_math_complex.c.src` | 复数数学函数 |
| `npy_*` 函数族 | `npy_math_internal.h.src` | 实数学函数宏/内联封装 |
| `NPY_INLINE_MATH` | `npy_math.c` | 0=生成外部符号，1=仅头文件内联 |

## 3. 关键调用链

### 3.1 数学函数调用

1. ufunc 内核（loops_*.dispatch）调用 `npy_sqrt(x)`、`npy_log(x)` 等
2. `npy_math_internal.h.src` 中宏映射到系统 libm 或本地实现
3. 复数操作经 `npy_math_complex.c.src` 中的 BSD 移植实现
4. 半精度输入先经 `npy_half_to_float` 转换为 float 再计算

### 3.2 半精度转换

1. `npy_half_to_float(h)`（halffloat.cpp:20）：`Half::FromBits(h)` → `static_cast<float>`
2. `npy_float_to_half(f)`（:33）：`Half(f).Bits()` 转回 16 位模式
3. 转换时按需触发 underflow/overflow 异常（`NPY_HALF_GENERATE_OVERFLOW`）

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `NPY_INLINE_MATH` | 0=npy_math.c 生成外部符号；1=仅头文件内联 | `npy_math.c:3` |
| `NPY_HALF_GENERATE_OVERFLOW` | 1=半精度转换触发 overflow 异常 | `halffloat.cpp:6` |
| `NPY_HALF_GENERATE_INVALID` | 1=半精度转换触发 invalid 异常 | `halffloat.cpp:7` |
| long double 平台差异 | 80 位 x86 / 64 位 double / 128 位 quadruple | `npy_longdouble.c` |

## 5. 错误与重试语义

- **数学域错误**：sqrt(负)/log(零) 等按 C99 语义返回 NaN/Inf，不抛异常；由 ufunc 层 fperr 检查捕获
- **半精度溢出**：转换时设置硬件 overflow 标志
- **long double 差异**：平台相关精度，不做补偿
- 无自动重试

## 6. 并发细节

- **纯函数**：数学函数无状态，线程安全
- **GIL**：被 ufunc 内层循环调用时 GIL 已释放
- **无内部锁**

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `npymath/npy_math.c.src`：实数学函数模板
- `npymath/npy_math_complex.c.src`：复数数学函数（BSD 移植）
- `npymath/ieee754.c.src`：IEEE-754 位级操作
- `npymath/halffloat.cpp`：半精度转换
- `common/npy_longdouble.c`：long double 封装

**Out-of-Scope（不在本仓库源码内）**
- 系统 libm（glibc/msvc runtime）：大部分实数学函数转发到系统库
- BLAS/LAPACK（见 linalg 子系统）
- SIMD 向量化（见 `../simd/`）

## 8. 与相邻子系统交互

- **上游 → ufunc-loops**：内核调用 npy_* 数学函数
- **下游 → 系统 libm**：大部分实函数转发到平台 C 库
- **下游 → npymath 库**：编译为独立 `npymath` 静态库供扩展使用

## 9. 语言专项适配口径

1. **.src 模板**：`npy_math.c.src`、`ieee754.c.src` 用 repeat 宏按浮点类型实例化。
2. **跨平台移植**：复数函数源自 FreeBSD msun 库（BSD 许可证），用于补全 C99 缺失。
3. **半精度**：C++ 实现（halffloat.cpp），使用 `np::Half` 类。
4. **依赖方向**：npymath 是最底层库，被 umath 与 multiarray 依赖，不反向依赖。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 架构图 | `npymath-architecture.html` | architecture | standard |

降档说明：6 组件 5 连线，showcase 标签重叠校验未全过，降 standard 档。数学库为纯函数集合，无复杂状态机或管道，不单独产出 sequence/dataflow 图。JSON IR 源文件位于 `json/` 目录。
