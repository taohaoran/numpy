# NumPy 源码架构分析文档

> 源码基准：git commit `3deeb20adb47f5da91e490ef8feda81114823710`
> 分析工具：archify（交互式 HTML 架构图）
> 文档语言：简体中文
> 拆分粒度：12 域 / 37 叶子，每叶子含设计文档级 MD（10 小节）+ archify 架构图

## 系统级文档

| 文档 | 说明 |
|------|------|
| [系统级架构总览](system-overview.md) | 功能总览、解决的问题、系统边界、架构图/时序图/数据流图说明 |
| [系统架构图](system-architecture.html) | 三层分层架构（Python API → Cython 绑定 → C 核心） |
| [ufunc 调用时序图](system-sequence.html) | ufunc 从 Python 调用到 SIMD 内核执行的完整生命周期 |
| [数据处理流水线](system-dataflow.html) | 输入 → 强制转换 → 数组表示 → 分派计算 → 输出 的数据流 |

## 域与叶子索引

### D1 core-ndarray（数组对象域）
[域总览](core-ndarray/core-ndarray.md)

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| ndarray-object | [ndarray-object.md](core-ndarray/ndarray-object/ndarray-object.md) | [架构图](core-ndarray/ndarray-object/ndarray-object-architecture.html) |
| dtype-system | [dtype-system.md](core-ndarray/dtype-system/dtype-system.md) | [架构图](core-ndarray/dtype-system/dtype-system-architecture.html) |
| conversion-coercion | [conversion-coercion.md](core-ndarray/conversion-coercion/conversion-coercion.md) | [架构图](core-ndarray/conversion-coercion/conversion-coercion-architecture.html) |
| iteration-indexing | [iteration-indexing.md](core-ndarray/iteration-indexing/iteration-indexing.md) | [架构图](core-ndarray/iteration-indexing/iteration-indexing-architecture.html) |
| datetime | [datetime.md](core-ndarray/datetime/datetime.md) | [架构图](core-ndarray/datetime/datetime-architecture.html) |
| stringdtype | [stringdtype.md](core-ndarray/stringdtype/stringdtype.md) | [架构图](core-ndarray/stringdtype/stringdtype-architecture.html) |

### D2 core-ufunc（通用函数域）
[域总览](core-ufunc/core-ufunc.md)

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| ufunc-object | [ufunc-object.md](core-ufunc/ufunc-object/ufunc-object.md) | [架构图](core-ufunc/ufunc-object/ufunc-object-architecture.html) · [时序图](core-ufunc/ufunc-object/ufunc-object-sequence.html) |
| ufunc-loops | [ufunc-loops.md](core-ufunc/ufunc-loops/ufunc-loops.md) | [架构图](core-ufunc/ufunc-loops/ufunc-loops-architecture.html) |
| ufunc-dispatch | [ufunc-dispatch.md](core-ufunc/ufunc-dispatch/ufunc-dispatch.md) | [架构图](core-ufunc/ufunc-dispatch/ufunc-dispatch-architecture.html) |
| ufunc-error | [ufunc-error.md](core-ufunc/ufunc-error/ufunc-error.md) | [架构图](core-ufunc/ufunc-error/ufunc-error-architecture.html) |

### D3 core-common（公共基础设施域）
[域总览](core-common/core-common.md)

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| strided-loops | [strided-loops.md](core-common/strided-loops/strided-loops.md) | [架构图](core-common/strided-loops/strided-loops-architecture.html) |
| simd | [simd.md](core-common/simd/simd.md) | [架构图](core-common/simd/simd-architecture.html) |
| npymath | [npymath.md](core-common/npymath/npymath.md) | [架构图](core-common/npymath/npymath-architecture.html) |

### D4 core-sort（排序域）
[域总览](core-sort/core-sort.md)

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| sort-algorithms | [sort-algorithms.md](core-sort/sort-algorithms/sort-algorithms.md) | [架构图](core-sort/sort-algorithms/sort-algorithms-architecture.html) · [数据流图](core-sort/sort-algorithms/sort-algorithms-dataflow.html) |

### D5 core-python-api（Python 前端域）
[域总览](core-python-api/core-python-api.md)

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| numeric-layer | [numeric-layer.md](core-python-api/numeric-layer/numeric-layer.md) | [架构图](core-python-api/numeric-layer/numeric-layer-architecture.html) |
| function-base | [function-base.md](core-python-api/function-base/function-base.md) | [架构图](core-python-api/function-base/function-base-architecture.html) |
| arrayprint-format | [arrayprint-format.md](core-python-api/arrayprint-format/arrayprint-format.md) | [架构图](core-python-api/arrayprint-format/arrayprint-format-architecture.html) |
| records-strings | [records-strings.md](core-python-api/records-strings/records-strings.md) | [架构图](core-python-api/records-strings/records-strings-architecture.html) |
| overrides-ufunc-config | [overrides-ufunc-config.md](core-python-api/overrides-ufunc-config/overrides-ufunc-config.md) | [架构图](core-python-api/overrides-ufunc-config/overrides-ufunc-config-architecture.html) |
| numerictypes-getlimits | [numerictypes-getlimits.md](core-python-api/numerictypes-getlimits/numerictypes-getlimits.md) | [架构图](core-python-api/numerictypes-getlimits/numerictypes-getlimits-architecture.html) |

### D6 random（随机数域）
[域总览](random/random.md)

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| bit-generators | [bit-generators.md](random/bit-generators/bit-generators.md) | [架构图](random/bit-generators/bit-generators-architecture.html) |
| generator-distributions | [generator-distributions.md](random/generator-distributions/generator-distributions.md) | [架构图](random/generator-distributions/generator-distributions-architecture.html) |
| legacy-mtrand | [legacy-mtrand.md](random/legacy-mtrand/legacy-mtrand.md) | [架构图](random/legacy-mtrand/legacy-mtrand-architecture.html) |

### D7 fft（傅里叶变换域）
[域总览](fft/fft.md)

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| fft | [fft.md](fft/fft/fft.md) | [架构图](fft/fft/fft-architecture.html) |

### D8 linalg（线性代数域）
[域总览](linalg/linalg.md)

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| linalg-core | [linalg-core.md](linalg/linalg-core/linalg-core.md) | [架构图](linalg/linalg-core/linalg-core-architecture.html) |
| linalg-python | [linalg-python.md](linalg/linalg-python/linalg-python.md) | [架构图](linalg/linalg-python/linalg-python-architecture.html) |

### D9 lib（工具库域）
[域总览](lib/lib.md)

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| npyio | [npyio.md](lib/npyio/npyio.md) | [架构图](lib/npyio/npyio-architecture.html) · [数据流图](lib/npyio/npyio-dataflow.html) |
| array-utils | [array-utils.md](lib/array-utils/array-utils.md) | [架构图](lib/array-utils/array-utils-architecture.html) |
| stride-tricks | [stride-tricks.md](lib/stride-tricks/stride-tricks.md) | [架构图](lib/stride-tricks/stride-tricks-architecture.html) |

### D10 ma-polynomial-matrix（掩码/多项式/矩阵域）
[域总览](ma-polynomial-matrix/ma-polynomial-matrix.md)

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| masked-array | [masked-array.md](ma-polynomial-matrix/masked-array/masked-array.md) | [架构图](ma-polynomial-matrix/masked-array/masked-array-architecture.html) |
| polynomial | [polynomial.md](ma-polynomial-matrix/polynomial/polynomial.md) | [架构图](ma-polynomial-matrix/polynomial/polynomial-architecture.html) |
| matrix | [matrix.md](ma-polynomial-matrix/matrix/matrix.md) | [架构图](ma-polynomial-matrix/matrix/matrix-architecture.html) |

### D11 f2py（Fortran 接口域）
[域总览](f2py/f2py.md)

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| f2py-core | [f2py-core.md](f2py/f2py-core/f2py-core.md) | [架构图](f2py/f2py-core/f2py-core-architecture.html) · [数据流图](f2py/f2py-core/f2py-core-dataflow.html) |

### D12 support（支撑域）
[域总览](support/support.md)

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| testing-utils | [testing-utils.md](support/testing-utils/testing-utils.md) | [架构图](support/testing-utils/testing-utils-architecture.html) |
| typing-stubs | [typing-stubs.md](support/typing-stubs/typing-stubs.md) | [架构图](support/typing-stubs/typing-stubs-architecture.html) |
| code-gen | [code-gen.md](support/code-gen/code-gen.md) | [架构图](support/code-gen/code-gen-architecture.html) · [数据流图](support/code-gen/code-gen-dataflow.html) |
| build-meson | [build-meson.md](support/build-meson/build-meson.md) | [架构图](support/build-meson/build-meson-architecture.html) |

## 文档结构说明

```
docs/architecture/
├── README.md                  # 本文件（三层索引）
├── system-overview.md         # 系统级总览
├── system-architecture.html   # 系统架构图
├── system-sequence.html       # ufunc 调用时序图
├── system-dataflow.html       # 数据处理流水线
├── json/                      # 系统级 JSON IR
├── core-ndarray/              # D1 域（6 叶）
├── core-ufunc/                # D2 域（4 叶）
├── core-common/               # D3 域（3 叶）
├── core-sort/                 # D4 域（1 叶）
├── core-python-api/           # D5 域（6 叶）
├── random/                    # D6 域（3 叶）
├── fft/                       # D7 域（1 叶）
├── linalg/                    # D8 域（2 叶）
├── lib/                       # D9 域（3 叶）
├── ma-polynomial-matrix/      # D10 域（3 叶）
├── f2py/                      # D11 域（1 叶）
└── support/                   # D12 域（4 叶）
```

每个叶子目录含：`<leaf>.md`（设计文档级分析，固定 10 小节）、`<leaf>-architecture.html`（至少 1 张架构图）、按需补充的时序图/数据流图、`json/`（archify JSON IR）。
