# support（support）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：numpy 主分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 域职责

本域是 NumPy 的"支撑面"：测试工具、类型注解、构建期 C 代码生成、meson 构建系统。它们不直接提供数值计算，而是保障 NumPy 自身的开发、类型检查、可移植编译与测试。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 数据流图 | 职责一句话 |
|------|------|--------|----------|-----------|
| testing-utils | [testing-utils.md](testing-utils/testing-utils.md) | [架构图](testing-utils/testing-utils-architecture.html) | — | 数值断言/夹具/扩展即时编译 |
| typing-stubs | [typing-stubs.md](typing-stubs/typing-stubs.md) | [架构图](typing-stubs/typing-stubs-architecture.html) | — | ArrayLike/DTypeLike 类型别名与 Protocol |
| code-gen | [code-gen.md](code-gen/code-gen.md) | [架构图](code-gen/code-gen-architecture.html) | [数据流](code-gen/code-gen-dataflow.html) | .src 模板/ufunc/C-API 代码生成 |
| build-meson | [build-meson.md](build-meson/build-meson.md) | [架构图](build-meson/build-meson-architecture.html) | — | meson 构建与 CPU/SIMD 分派配置 |

## 3. 域级机制细节

- **构建期代码生成**：code-gen 与 build-meson 衔接——meson 调用 `_build_utils`/`code_generators` 把模板与定义表渲染为 C 源，产物不入库。
- **可选依赖**：testing（hypothesis/编译器）、build（BLAS/Fortran 编译器）缺失时优雅降级或跳过。
- **vendored 标注**：tempita、vendored-meson 为第三方副本，标注"不在本项目原创源码内"。
