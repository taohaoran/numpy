# 傅里叶变换域（fft）域总览

> 本域包含 NumPy 傅里叶变换系统的 1 个叶子子系统；叶子详情见对应文档。
> 源码基准：NumPy main 分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 域职责

本域实现 NumPy 的离散傅里叶变换（DFT）功能，基于 vendored pocketfft C++ 库。Python 层提供 fft/ifft/rfft/irfft/hfft/ihfft 及多维变体，以及 fftshift/fftfreq 等辅助函数。按「Python + C++ 混合项目专项分析清单」分析：Python 包装 + vendored C++ 核心的边界。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 职责一句话 |
|------|------|--------|------------|
| fft | [fft.md](fft/fft.md) | [架构图](fft/fft-architecture.html) | 傅里叶变换 Python API + pocketfft C++ 包装 |

## 3. 域级机制细节

- **统一入口**：`_raw_fft` 是所有 FFT 变体的统一内部入口，通过 is_real/is_forward 参数区分 6 种变体
- **归一化**：norm 参数控制 backward（默认）/ortho/forward 三种归一化模式
- **vendored C++**：pocketfft 是第三方 C++ 库（Reiner Bauer），在 `numpy/fft/pocketfft/` 中 vendored，不在本项目原创源码内

## 4. 域级图

本域仅 1 个叶子，无单独域级图。
