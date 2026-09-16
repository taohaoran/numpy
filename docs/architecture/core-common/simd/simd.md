# SIMD 抽象层与 CPU 分派（simd）

> 本文是 `core-common` 域下的叶子子系统文档。域级总览见 `../core-common.md`。
> 本文聚焦 CPU 特性检测、SIMD 内建抽象层与多 ISA 版本编译分派，是 ufunc 内层 SIMD 特化的底层支撑。
>
> 源码基准：numpy commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| CPU 特性初始化 | `npy_cpu_init` 调用平台检测函数填充特性表 | `common/npy_cpu_features.c:50` |
| x86 检测 | `npy__cpu_init_features` 经 CPUID 检测 AVX2/AVX512/SSE | `common/npy_cpu_features.c:464` |
| ARM 检测 | `npy__cpu_init_features_arm8` 经 getauxval 检测 NEON/LSX | `common/npy_cpu_features.c:784,816` |
| 环境变量禁用 | `npy__cpu_check_env` 解析 `NPY_DISABLE_CPU_FEATURES` | `common/npy_cpu_features.c:265` |
| 基线校验 | `npy__cpu_validate_baseline` 验证基线 ISA 满足 | `common/npy_cpu_features.c:236` |
| 查询特性 | `npy_cpu_have(feature_id)` 运行时查特性 | `common/npy_cpu_features.c:42` |
| Python 接口 | `_simd` 模块：get_floatstatus/clear_floatstatus | `_simd/_simd.c:5-21` |
| 多版本模块加载 | `_simd_exec` 按 NPY_CPU_HAVE 加载各 ISA 子模块 | `_simd/_simd.c:59` |
| SIMD 内建抽象 | `npyv_*` 跨平台向量内建（load/store/add/mul/compare） | `common/simd/simd.h`、`_simd/_simd.dispatch.c.src` |
| ISA 后端目录 | avx2/avx512/sse/neon/lsx 各自内建实现 | `common/simd/{avx2,avx512,sse,neon,lsx}/` |
| 分派追踪 | `npy_cpu_dispatch_trace` 记录当前分派目标 | `common/npy_cpu_dispatch.c:35` |
| 分派注册表 | `__cpu_targets_info__` Python 可见分派信息 | `common/npy_cpu_dispatch.c:10` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `npy_cpu_have(feature_id)` | `npy_cpu_features.c:42` | 运行时查询 CPU 是否支持某特性 |
| `NPY_CPU_HAVE(FEATURE)` | `npy_cpu_features.h:164` | 宏：`npy_cpu_have(NPY_CPU_FEATURE_##FEATURE)` |
| `npyv_*` | `common/simd/simd.h` | 跨平台 SIMD 内建函数族 |
| `NPY_SIMD` | `simd.h` | 编译期 SIMD 向量宽度 |
| `NPY_MTARGETS_CONF_*` | meson 生成宏 | 遍历配置的 dispatch 目标 |
| `_simd_exec` | `_simd.c:59` | 模块初始化：加载 baseline 与各 dispatch 子模块 |

## 3. 关键调用链

### 3.1 CPU 检测与分派

1. 模块加载时 `_simd_exec`（_simd.c:59）调用 `npy_cpu_init()`（:68）
2. `npy_cpu_init`（npy_cpu_features.c:50）调用平台 `npy__cpu_init_features`：
   - x86：CPUID 指令检测 AVX2/AVX512/SSE
   - ARM：getauxval/elf_aux_info 检测 NEON/LSX
3. `npy__cpu_check_env`（:265）读 `NPY_DISABLE_CPU_FEATURES` 环境变量禁用指定特性
4. `npy__cpu_validate_baseline`（:236）验证基线 ISA 满足
5. 运行时内核经 `NPY_CPU_HAVE(FEATURE)` 宏查询特性，选择最优编译版本

### 3.2 多版本编译

1. meson 构建配置 baseline（如 SSE2/NEON）与 dispatch（AVX2/AVX512）目标
2. 每个 `loops_*.dispatch.c.src` 编译为多个版本（每 ISA 一份）
3. `_simd_exec` 经 `NPY_MTARGETS_CONF_DISPATCH` 宏逐目标调用 `simd_create_module_<TARGET>` 创建子模块
4. 不支持的目标设为 None；运行时按 CPU 特性选用

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `NPY_DISABLE_CPU_FEATURES` | 逗号分隔特性名，禁用指定 CPU 优化 | `npy_cpu_features.c:265` |
| meson baseline | 编译基线 ISA 列表 | meson.build |
| meson dispatch | 可分派 ISA 列表 | meson.build |
| `NPY_SIMD` | 0=无 SIMD，否则向量宽度 | `simd.h` |

## 5. 错误与重试语义

- **基线不满足**：`npy__cpu_validate_baseline` 失败时报错退出
- **特性不支持**：`npy_cpu_have` 返回 0，回退 baseline 版本
- **模块重复加载**：`_simd_exec` 检测 `module_loaded` 标志，禁止重复
- 无自动重试

## 6. 并发细节

- **CPU 检测**：进程初始化时单次执行，无后续竞争
- **分派**：纯只读查询，线程安全
- **GIL**：`_simd` 模块在 free-threaded Python 中标注 `Py_MOD_GIL_NOT_USED`（_simd.c:129）
- **单实例**：`module_loaded` 标志确保 `_simd` 模块每进程仅加载一次

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `common/npy_cpu_features.c/h`：CPU 特性检测
- `common/npy_cpu_dispatch.c/h`：分派追踪
- `common/simd/`：SIMD 内建抽象与各 ISA 后端
- `_simd/_simd.c`：Python 模块封装
- `_simd/_simd.dispatch.c.src`：npyv_* 内建的 Python 绑定模板

**Out-of-Scope（不在本仓库源码内）**
- 实际算子内核（见 `../../core-ufunc/ufunc-loops/`）
- 编译器自动向量化（由编译器完成）
- MLAS/oneDNN 等外部 BLAS 加速库（不在本仓库）

## 8. 与相邻子系统交互

- **上游 → ufunc-loops**：SIMD 内核调用 `npyv_*` 内建函数
- **上游 → ufunc-object**：执行时按 CPU 特性选最优内核版本
- **下游 → 平台**：调用 CPUID/getauxval 等平台 API
- **下游 → meson 构建**：依赖 meson_cpu 配置生成多版本编译

## 9. 语言专项适配口径

1. **两层分派并存**：本叶子是 NumPy 的"DispatchStub 式"内层特化——编译期多 ISA 版本 + 运行期 `npy_cpu_have` 选择，对应混合专项清单第 10 条。
2. **代码生成**：`_simd/_simd.dispatch.c.src` 用 repeat 宏按数据类型（u8/s8/.../f64）实例化内建绑定。
3. **多后端矩阵**：x86（SSE/AVX2/AVX512）与 ARM（NEON/LSX）双平台，各自 ISA 后端目录。
4. **依赖方向**：`common/simd/` 被 `umath/` 使用，单向。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 架构图 | `simd-architecture.html` | architecture | standard |

降档说明：含 CPU 检测与多版本编译两层 6 组件 6 连线，showcase 端点方向校验未全过，降 standard 档。JSON IR 源文件位于 `json/` 目录。
