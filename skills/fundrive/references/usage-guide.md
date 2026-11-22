# FunDrive 使用参考

本文档面向需要在业务代码或脚本中集成、调用 `fundrive` 及独立驱动包 `fundrives.*` 的场景。接口规范权威定义见 [002-interface-contract.md](https://github.com/farfarfun-drive/fundrive-dev/blob/master/docs/development/driver-contract/002-interface-contract.md)。

## 1. 安装方式

### 核心库与按需 extras

`fundrive` 要求 `Python >= 3.12`，驱动依赖通过 PEP 621 optional extras 按需安装：

```bash
# 仅核心库（内置 local/os, github, gitee, onedrive, tsinghua, pcloud, mediafire, zenodo, openxlab, tianchi 等）
uv add fundrive

# 安装指定网盘驱动依赖
uv add "fundrive[alipan]"     # aligo + fundrive-alipan
uv add "fundrive[baidu]"      # fundrive-baidu
uv add "fundrive[lanzou]"     # fundrive-lanzou + funfile
uv add "fundrive[oss]"        # oss2
uv add "fundrive[ossutil]"    # funinstall
uv add "fundrive[amazon]"     # boto3 + botocore（S3 别名 "s3" / "amazon"）
uv add "fundrive[google]"     # google-api-python-client + google-auth*
uv add "fundrive[dropbox]"    # dropbox
uv add "fundrive[webdav]"     # webdav4
uv add "fundrive[pan115]"     # p115client + funfile（别名 "pan115" / "115"）
uv add "fundrive[wenshushu]"  # base58 + pycryptodomex

# 一次性安装全部可选驱动
uv add "fundrive[all]"
```

### 独立使用驱动包

4 个独立驱动包均可脱离核心库单独安装（`baidu` / `lanzou` / `quark` 支持 `Python >= 3.10`，`alipan` 要求 `>= 3.12`）：

```bash
uv add fundrive-alipan   # import fundrives.aliopen
uv add fundrive-baidu    # import fundrives.baidu
uv add fundrive-lanzou   # import fundrives.lanzou
uv add fundrive-quark    # import fundrives.quark（独立包，核心库无 quark extra）
```

## 2. 获取与发现驱动

核心库采用**按需懒加载**，`import fundrive` 不会导入任何第三方 SDK：

```python
from fundrive import (
    DRIVE_SPECS,
    create_drive,
    get_drive,
    get_supported_drives,
    list_available_drives,
    list_missing_drives,
)

# 1. 廉价查询全部支持驱动 key（不触发 import）
keys = get_supported_drives()  # 等价于 list(DRIVE_SPECS)

# 2. 按驱动名创建实例（key 大小写不敏感；依赖缺失时抛出带 pip install 提示的 ImportError）
drive = get_drive("s3", access_key_id="...", secret_access_key="...", bucket_name="...")
drive2 = create_drive("dropbox", access_token="...")  # get_drive 的便捷封装

# 3. 也可以按类名懒加载（只导入该驱动模块）
from fundrive.drives import OSSDrive
oss = OSSDrive()

# 4. 仅在需要诊断环境时探测（会逐个尝试导入）
installed = list_available_drives()
missing = list_missing_drives()  # {"alipan": "fundrive[alipan]", ...}
```

## 3. 核心 API 速查（`BaseDrive`）

### 必须实现的基础方法

| 方法 | 规范签名 | 返回值 / 说明 |
| --- | --- | --- |
| `login` | `(*args, **kwargs)` | `bool`，失败返回 `False`（部分强校验驱动如 `cmcloud` 依赖外部票据） |
| `exist` | `(fid: str, *args, **kwargs)` | `bool` |
| `mkdir` | `(fid: str, name: str, return_if_exist: bool = True, *args, **kwargs)` | `str`（新建或已存在目录的 `fid`） |
| `delete` | `(fid: str, *args, **kwargs)` | `bool` |
| `get_file_list` | `(fid: str, *args, **kwargs)` | `list[DriveFile]` |
| `get_dir_list` | `(fid: str, *args, **kwargs)` | `list[DriveFile]` |
| `get_file_info` | `(fid: str, *args, **kwargs)` | `DriveFile \| None`（必须判空） |
| `get_dir_info` | `(fid: str, *args, **kwargs)` | `DriveFile \| None`（必须判空） |
| `upload_file` | `(filepath: str, fid: str, *args, **kwargs)` | `bool`（`filepath` 为本地路径，`fid` 为目标目录 ID） |
| `download_file` | `(fid: str, save_dir=None, filename=None, filepath=None, overwrite=False, *args, **kwargs)` | `bool`（`filepath` 优先，否则 `save_dir` + `filename`） |

### 内置目录递归方法（基类已提供默认实现）

- `download_dir(fid, save_dir, recursion=True, overwrite=False, ignore_filter=None, **kwargs) -> bool`
- `upload_dir(filedir, fid, recursion=True, overwrite=False, **kwargs) -> bool`

### 可选高级方法（并非所有网盘均支持）

- `search(keyword, fid=None, file_type=None, *args, **kwargs) -> list[DriveFile]`
- `move(source_fid, target_fid, *args, **kwargs) -> bool`
- `copy(source_fid, target_fid, *args, **kwargs) -> bool`
- `rename(fid, new_name, *args, **kwargs) -> bool`
- `share(*fids, password, expire_days=0, description="") -> Any`
- `save_shared(shared_url, fid, password=None) -> bool`
- `get_quota(*args, **kwargs) -> dict`
- `get_recycle_list()`, `restore(fid)`, `clear_recycle()`
- `get_download_url(fid)`, `get_upload_url(fid, filename)`

### `DriveFile` 对象

`DriveFile` 继承自 `dict`，同时支持下标与属性访问：

```python
for f in drive.get_file_list(drive.root_fid or "/"):
    print(f.fid, f.name, f.size, f.filename)  # 等价于 f["fid"], f["name"], f.get("size")
    extra = f.ext  # 排除 fid/name/size 后的驱动扩展字段
```

## 4. 凭据与配置管理

推荐顺序（高优先级覆盖低优先级）：
1. 构造函数或 `login()` 显式传入关键字参数。
2. `funsecret` 配置库：键名格式 `fundrive.<drive_name>.<param>`，namespace 统一为 `"fundrive"`。
3. 环境变量：建议命名 `FUNDRIVE_<DRIVE>_<KEY>`。

> 注意：不要将 token / cookie / access_secret 硬编码进代码或输出到日志中。

## 5. 跨网盘搬运与工具函数

```python
from fundrive import copy_data, format_size, parse_size, sanitize_filename, get_file_hash

# 跨网盘整目录复制（使用临时目录中转并在结束后自动清理）
# drive1、drive2 均需先 login()
copy_data(drive1=src_drive, drive2=dst_drive, from_fid="src_dir_fid", to_fid="dst_parent_fid")
```

统一异常体系位于 `fundrive.core.exceptions`（也从顶层 `fundrive` 导出）：
`FunDriveError`、`AuthenticationError`、`AuthorizationError`、`NetworkError`、`RateLimitError`、`FileNotFoundError`、`FileExistsError`、`InsufficientStorageError`、`InvalidParameterError`、`OperationNotSupportedError`、`UploadError`、`DownloadError`。

## 6. 2.x 常见迁移与避坑

- **根目录 `fid` 差异**：不同网盘根标识不同（如 `""`、`"."`、`"/"`、`"root"`、`"0"`），优先使用 `drive.root_fid` 或查阅对应驱动的 `README.md`。
- **1.x → 2.x 关键字变更**：
  - `local_dir` → `save_dir`（`download_file` / `download_dir`）
  - `local_path` → `filepath`（`upload_file`）或 `filedir`（`upload_dir`）
- **不要直接传 `filepath` 裸文件名却忽略父目录**：在写自定义下载调用时，如只传 `filename` 不传 `save_dir` 会落到当前目录。
