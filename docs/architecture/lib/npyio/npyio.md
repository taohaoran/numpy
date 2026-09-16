# npyio（npyio）

> 本文是 `lib` 域下的叶子子系统文档。域级总览见 `../lib.md`。
> 本文只展开 NumPy 磁盘 IO 能力（.npy/.npz 二进制格式与文本读写），不重复展开 `array-utils`（数组辅助函数）与 `stride-tricks`（视图技巧）的职责。
>
> 源码基准：numpy 主分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

NumPy 的磁盘 IO 分两层：薄包装层（`numpy/lib/npyio.py`、`format.py`，仅做再导出）与实现层（`_npyio_impl.py`、`_format_impl.py`、`_iotools.py`）。

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| .npy 单数组二进制读写 | 自描述二进制格式：魔数 + 头（dict 字面量）+ 原始字节 | `numpy/lib/_format_impl.py` |
| .npz 多数组归档 | zip 归档打包多个 .npy，按 key 懒加载 | `numpy/lib/_npyio_impl.py:116`（`NpzFile`） |
| 二进制 API 编排 | `save/savez/savez_compressed/load` | `numpy/lib/_npyio_impl.py:312/505/581/682/756` |
| 文本写入 | `savetxt` 按 fmt 格式化逐行写出 | `numpy/lib/_npyio_impl.py:1399` |
| 文本读取 | `loadtxt/genfromtxt/fromregex` 解析文本表 | `numpy/lib/_npyio_impl.py:1120/1735/1632` |
| 文本转换工具 | 行切分、列名校验、字符串→dtype 转换 | `numpy/lib/_iotools.py` |
| 内存映射 | `open_memmap` 把 .npy 映射为数组视图 | `numpy/lib/_format_impl.py:899` |
| 数据源抽象 | `DataSource` 统一本地/远程文件获取 | `numpy/lib/_npyio_impl.py`（`_datasource.py`） |
| 薄包装再导出 | `npyio.py`/`format.py` 只 `from ._xxx_impl import ...` | `numpy/lib/npyio.py:1`、`format.py:1` |

关键对外暴露点：`numpy.save/load/savez/loadtxt/savetxt/genfromtxt`（经 `numpy/__init__.py` 再导出）。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `_header_size_info` | `_format_impl.py:187` | 格式版本 → 头长度结构与编码映射（1.0 用 `<H` 两字节，2.0/3.0 用 `<I` 四字节） |
| `read_array_header_1_0/2_0` | `_format_impl.py:511/552` | 按版本读头字节并用 `ast.literal_eval` 还原头字典 |
| `dtype_to_descr / descr_to_dtype` | `_format_impl.py:251/311` | dtype 与可序列化 descr 之间的往返（record array 空名字字段的特殊处理） |
| `read_array / write_array` | `_format_impl.py:787/709` | 头之后的数组字节读/写（支持 mmap 与 allow_pickle） |
| `NpzFile(Mapping)` | `_npyio_impl.py:116` | 字典式、懒加载的 npz 视图；`__getitem__` 时才解压单个 .npy |
| `BagObj` | `_npyio_impl.py:50` | 让 `npz.f.key` 属性式访问等价于 `npz[key]` |
| `zipfile_factory` | `_npyio_impl.py:100` | 决定用 `zipfile` 还是 `zipfile3`（兼容层）打开归档 |
| `LineSplitter` | `_iotools.py:133` | 把一行文本按分隔符/引号切分为列 |
| `NameValidator` | `_iotools.py:229` | 校验/生成结构化数组字段名 |
| `StringConverter` | `_iotools.py:452` | 单列字符串 → 目标 dtype 的转换器，含类型推断与失败缓存 |
| `easy_dtype` | `_iotools.py:824` | 由用户给定的 dtype 描述 + names 构造最终 dtype |
| `ConverterError / ConversionWarning` | `_iotools.py:423/452` | 列转换失败的异常族 |

## 3. 关键调用链

**链路一：`np.load` 读取 .npy**（与 `npyio-dataflow.html` 对应）
1. `load(file, mmap_mode=...)`（`_npyio_impl.py:312`）先判断是否 .npz（zip 魔数），否则走单 .npy 路径。
2. `read_magic(fp)`（`_format_impl.py:230`）读取 `MAGIC_LEN` 字节，校验 `\x93NUMPY` 前缀并返回 `(major, minor)`。
3. 按版本分派 `read_array_header_1_0/2_0`（`_format_impl.py:511/552`），读头长度字段后用 `_read_array_header` 内的 `ast.literal_eval` 还原 `{descr, fortran_order, shape}` 字典。
4. `descr_to_dtype(descr)`（`_format_impl.py:311`）把描述符转回 `dtype`。
5. `read_array(fp, ...)`（`_format_impl.py:787`）按 dtype/shape/order 用 `frombuffer` 或 `memmap` 把后续字节重建为 ndarray。

**链路二：`np.loadtxt` 文本解析**
1. `loadtxt`（`_npyio_impl.py:1120`）打开文件/迭代器，逐行交给 `_read`（`_npyio_impl.py:861`）。
2. `_read` 用 `LineSplitter` 切列，按列收集字符串。
3. 每列经 `StringConverter`（`_iotools.py:452`）推断/转换为目标 dtype，堆叠为结构化或普通数组。

**链路三：`np.savez` 写出**
1. `savez`（`_npyio_impl.py:581`）→ `_savez`（`_npyio_impl.py:756`）遍历 `*args, **kwds`。
2. 每个数组经 `_format_impl.write_array` 写成内存缓冲，再写入 zip 归档（`compress` 决定是否压缩）。

## 4. 配置项

| 配置 / 常量 | 默认 / 行为 | 位置 |
|------|------|------|
| `MAGIC_PREFIX` | `b'\x93NUMPY'`，npy 魔数 | `_format_impl.py:178` |
| `ARRAY_ALIGN` | 64，头字节对齐到的边界（2 的幂 16~4096） | `_format_impl.py:180` |
| `BUFFER_SIZE` | `2**18`，读 npz 的缓冲字节数 | `_format_impl.py:181` |
| `_MAX_HEADER_SIZE` | 10000，`literal_eval` 头解析安全上限 | `_format_impl.py:196` |
| `allow_pickle` | `load` 默认 False、`save` 默认 True；为 True 时跳过头大小限制 | `_npyio_impl.py:312/505` |
| `max_header_size` | 可覆盖 `_MAX_HEADER_SIZE` | `_npyio_impl.py:194` |
| `comments/delimiter` | 文本 IO 默认 `'#'` / `None` | `_npyio_impl.py:1120` |

## 5. 错误与重试语义

- **魔数不符**：`read_magic` 直接 `raise ValueError`（`_format_impl.py:243`），不重试。
- **头大小超限**：`_read_array_header` 检查 `max_header_size`，超限报错以避免恶意/损坏头造成 `literal_eval` 开销或崩溃。
- **版本不支持**：`_check_version` 仅允许 (1,0)/(2,0)/(3,0)，其余 `ValueError`（`_format_impl.py:199`）。
- **列转换失败**：`StringConverter` 抛 `ConverterError`；`genfromtxt` 中按 `filling_values`/`usemask` 策略降级为缺失值（NaN 或掩码），而非整体失败。
- **pickle 安全**：`allow_pickle=False`（load 默认）时拒绝对象数组，防止反序列化任意代码执行。
- 本叶子无重试/退避机制（一次性同步 IO，失败即抛）。

## 6. 并发细节

- 纯 Python 同步 IO，受 GIL 约束，单线程执行；无内部线程或锁。
- `NpzFile` 懒加载：解压发生在用户访问某个 key 时，`zipfile` 对象持有底层文件句柄；`own_fid` 决定是否在关闭时负责关闭文件。
- `open_memmap` 借操作系统 mmap 把数组映射进地址空间，可跨进程共享（外部 OS 能力，不在本仓库源码内实现）。
- 读文件期间持有文件对象引用，无临界区竞争。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `numpy/lib/_format_impl.py`：npy 二进制格式读写、头解析、dtype 往返、memmap。
- `numpy/lib/_npyio_impl.py`：save/load/savez 编排、loadtxt/savetxt/genfromtxt/fromregex 文本解析、NpzFile。
- `numpy/lib/_iotools.py`：行切分、列名校验、字符串转换工具。
- 薄包装 `npyio.py`/`format.py`/`_datasource.py`。

**Out-of-Scope（不在本仓库源码内）**
- `zipfile`/`gzip`/`pickle` 标准库与文件系统调用——外部依赖。
- 实际字节落盘与 mmap 由 OS 内核完成——外部。
- 数组内存分配与底层 ufunc 计算由 `_core`（其他域）负责，本叶子只搬运字节。
- 掩码数组的文本读入由 `numpy/ma/extras.py`（`ma-polynomial-matrix` 域）的 `ma.genfromtxt` 包装，不在本叶子。

## 8. 与相邻子系统交互

- **上游（用户 API 层）**：`numpy/__init__.py` 把 `save/load/loadtxt/savetxt` 等符号再导出给用户；dispatcher（`_save_dispatcher` 等）为 `__array_function__` 协议服务。
- **本叶子 → 下游**：调用 `numpy.dtype`/`ndarray` 构造（`_core` 域）、`zipfile`、`pickle`、`ast.literal_eval`。
- **下游 → 本叶子**：`numpy.lib._datasource` 把远程/压缩资源归一为文件对象后交给本叶子。
- 方向：用户 → `numpy.lib.npyio` → 二进制/文本解析 → `ndarray` → 文件系统。

## 9. 语言专项适配口径（纯 Python 项目专项）

- **能力缝归组**：本叶子是"格式协议 + 解析管道"能力缝——`_format_impl` 定义 npy 格式契约（头 dict schema：`EXPECTED_KEYS = {'descr','fortran_order','shape'}`），`_npyio_impl` 是编排层，`_iotools` 是可复用的文本转换工具缝（`StringConverter` 是独立可实例化的转换器对象）。
- **懒加载工程**：`NpzFile` 实现 `Mapping` 协议做键级懒加载，`zipfile` 延迟到构造时才 `import`（`_npyio_impl.py:196` 注释：zipfile 依赖 gzip 可选组件）。
- **代码生成/DRY**：无；薄包装层 `npyio.py`/`format.py` 是"实现移到 `_impl` 模块后保留原路径"的重导出接缝（`__all__` 与 `@set_module` 控制对外模块名）。
- **配置驱动分派**：格式版本三元组 `(major,minor)` 驱动 `_header_size_info` 选结构与编码，是运行期按版本分派的静态表。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| 架构图 | `npyio-architecture.html` | architecture | standard |
| 读取管道数据流图 | `npyio-dataflow.html` | dataflow | standard |

- 降档原因：architecture 中间列节点较多，format→dtype 与文件系统的连接在 showcase 严格布局校验下触发"边穿无关节点"与"沿区域边框走线"约束，调整后仍需 standard 档；dataflow 因同列双节点竖边标签与节点重叠，`labelDy` 补偿后落 standard。两图 render 退出码均为 0、HTML 非空（约 800KB）。
- JSON IR 源文件：`json/npyio-architecture.json`、`json/npyio-dataflow.json`。
