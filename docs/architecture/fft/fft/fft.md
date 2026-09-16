# 傅里叶变换（fft）

> 本文是 `fft` 域下的叶子子系统文档。域级总览见 `../fft.md`。
>
> 源码基准：NumPy main 分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 标准 FFT | fft、ifft（一维复 FFT）；fftn、ifftn（N 维）；fft2、ifft2（二维） | `numpy/fft/_pocketfft.py` |
| 实数 FFT | rfft、irfft（一维实数）；rfftn、irfftn（N 维）；rfft2、irfft2（二维） | `_pocketfft.py` |
| Hermitian FFT | hfft、ihfft（Hermitian 对称变换） | `_pocketfft.py` |
| 核心内部函数 | _raw_fft：所有 FFT 变体的统一入口，处理 norm 归一化和实数/复数分支 | `_pocketfft.py:58` |
| 频域辅助 | fftshift、ifftshift（零频居中）；fftfreq、rfftfreq（采样频率） | `numpy/fft/_helper.py` |
| C++ 包装 | _pocketfft_umath.cpp：C++ Cython 包装层，连接 Python API 与 pocketfft | `numpy/fft/_pocketfft_umath.cpp` |

## 2. 核心类型与接口清单

| 类型/函数 | 位置 | 职责 |
|-----------|------|------|
| `_raw_fft(a, n, axis, is_real, is_forward, norm, out)` | `_pocketfft.py:58` | 核心内部函数：计算归一化因子，分派到 C++ ufunc |
| `fft(a, n, axis, norm)` | `_pocketfft.py` | 正向一维复 FFT |
| `rfft(a, n, axis, norm)` | `_pocketfft.py` | 正向一维实数 FFT（输出 n/2+1 复数） |
| `fftshift(x, axes)` | `_helper.py:20` | 将零频分量移到频谱中心 |
| `fftfreq(n, d)` | `_helper.py` | 返回采样频率数组 |

## 3. 关键调用链

### 3.1 np.fft.fft(x) 调用链

1. 用户调用 `np.fft.fft(x)` → `_pocketfft.py` 的 `fft()` 函数（经 NEP-18 包装）
2. `fft()` 调用 `_raw_fft(a, n, axis, is_real=False, is_forward=True, norm)`
3. `_raw_fft` 计算归一化因子 fct（backward=1, ortho=1/sqrt(n), forward=1/n）
4. 调用 `_pocketfft_umath.fftn(a, norm, ...)`（C++ 扩展）
5. C++ 层调用 vendored pocketfft 库执行实际 FFT
6. 返回结果（乘以 fct 归一化因子）

## 4. 配置项

| 配置项 | 默认值/行为 | 位置 |
|--------|-------------|------|
| `norm` | "backward"（正向不归一化，逆向 1/n）；"ortho"（双向 1/sqrt(n)）；"forward"（正向 1/n，逆向不缩放） | `_pocketfft.py:68-76` |
| `n`（变换长度） | None（使用输入长度）；指定时零填充或截断 | `_pocketfft.py` |
| dtype 提升 | float32→float64、complex64→complex128（自动提升） | `__init__.py:127` |

## 5. 错误与重试语义

- **无效 n**：n < 1 时抛出 ValueError（`_pocketfft.py:59`）
- **无效 norm**：非 backward/ortho/forward 时抛出 ValueError（`_pocketfft.py:75`）
- **非连续输入**：自动拷贝为连续数组

## 6. 并发细节

- Python 层不管理线程；C++ pocketfft 释放 GIL
- 多线程调用不同 FFT 函数安全（无共享状态）

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `numpy/fft/_pocketfft.py`：Python FFT API
- `numpy/fft/_helper.py`：频域辅助函数
- `numpy/fft/_pocketfft_umath.cpp`：C++ Cython 包装
- `numpy/fft/__init__.py`：再导出

**Out-of-Scope（不在本项目原创源码内）**

- `numpy/fft/pocketfft/`：vendored 第三方 pocketfft C++ 库（Reiner Bauer），不在本项目原创源码内

## 8. 与相邻子系统交互

- **上游**：`numpy/__init__.py` 从 `numpy.fft` 再导出
- **下游 → C++ 扩展**：`_pocketfft.py` → `_pocketfft_umath`（C++ Cython）→ pocketfft C++ 库
- **下游 → overrides**：所有公开函数经 `@array_function_dispatch` 包装

## 9. 语言专项适配口径

本叶子按「Python + C++ 混合项目专项分析清单」分析：

1. **绑定边界**：Python API（_pocketfft.py）→ C++ Cython 包装（_pocketfft_umath.cpp）→ vendored pocketfft C++ 库。C++ 核心是第三方 vendored，不深入分析其内部实现
2. **统一入口**：_raw_fft 是所有 FFT 变体的统一入口，通过 is_real/is_forward 参数区分 6 种变体（fft/ifft/rfft/irfft/hfft/ihfft），这是配置驱动静态分派的典型案例

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| fft 架构图 | `fft-architecture.html` | architecture | showcase |

- JSON IR 源文件：`json/fft-architecture.json`
- 降档原因：无
- 数据流图：FFT 调用链为线性单层转发，不适用额外数据流图
