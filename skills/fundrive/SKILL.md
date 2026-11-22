---
name: fundrive
description: Guide for using and developing the fundrive unified cloud storage ecosystem (core library fundrive and standalone driver packages fundrive-alipan, fundrive-baidu, fundrive-lanzou, fundrive-pan115, fundrive-quark) inside the fundrive-dev workspace. Use this skill when writing code that integrates fundrive drives, installing or configuring cloud storage extras, implementing or fixing a BaseDrive driver, maintaining fundrives.* namespace packages, or performing cross-repo testing, linting, submodule sync, and releases.
---

# FunDrive 开发与使用指南

本 Skill 覆盖 `fundrive` 生态的两类任务：**接入使用**与**驱动/工作区开发**。按当前任务读取对应参考文档：

- **使用 `fundrive` / `fundrives.*`（安装、选驱动、认证配置、文件操作、跨盘搬运）**：读 [usage-guide.md](./references/usage-guide.md)
- **开发驱动与跨仓库维护（新增/修改驱动、契约测试、Ruff 基线、子模块与发布纪律）**：读 [driver-development.md](./references/driver-development.md)

## 1. 工作区结构速查

编排仓库 [fundrive-dev](https://github.com/farfarfun-drive/fundrive-dev) 本身不含应用源码，通过 Git 子模块管理 6 个纯库包（`package` 类，共用 `scripts/funbuild.toml` 共享版本号）：

| 别名 | 路径 | PyPI 包 | 命名空间 | 与核心库关系 |
| --- | --- | --- | --- | --- |
| `core` | `apps/fundrive` | `fundrive` | `fundrive` | 核心框架：`BaseDrive` + 20+ 内置驱动 |
| `alipan` | `apps/fundrive-alipan` | `fundrive-alipan` | `fundrives.aliopen` | `fundrive[alipan]` 可选依赖 |
| `baidu` | `apps/fundrive-baidu` | `fundrive-baidu` | `fundrives.baidu` | `fundrive[baidu]` 可选依赖 |
| `lanzou` | `apps/fundrive-lanzou` | `fundrive-lanzou` | `fundrives.lanzou` | `fundrive[lanzou]` 可选依赖 |
| `pan115` | `apps/fundrive-pan115` | `fundrive-pan115` | `fundrives.pan115` | `fundrive[pan115]` 可选依赖 |
| `quark` | `apps/fundrive-quark` | `fundrive-quark` | `fundrives.quark` | **独立包**，无 `fundrive[quark]` |

权威契约与治理入口：
- 快速防踩坑清单：[CLAUDE.md](https://github.com/farfarfun-drive/fundrive-dev/blob/master/CLAUDE.md)
- 项目治理总览：[001-overview.md](https://github.com/farfarfun-drive/fundrive-dev/blob/master/docs/project/project-governance/001-overview.md)
- 统一接口契约：[002-interface-contract.md](https://github.com/farfarfun-drive/fundrive-dev/blob/master/docs/development/driver-contract/002-interface-contract.md)

## 2. 使用速查

```python
from fundrive import get_drive, get_supported_drives, list_missing_drives

# 零开销查看全部已注册驱动名（不会导入第三方 SDK）
print(get_supported_drives())

# 按需懒加载驱动实例（不区分大小写）
drive = get_drive("local")  # 或 "oss" / "google" / "baidu" / "alipan" / "s3" 等
if not drive.login():
    raise RuntimeError("登录失败")

files = drive.get_file_list(drive.root_fid or "/")
drive.upload_file(filepath="/local/a.txt", fid=drive.root_fid or "/")
drive.download_file(fid="remote_fid", save_dir="./downloads", overwrite=False)
```

使用要点：
- **优先关键字传参**（2.x 统一参数名：`filepath`、`filedir`、`save_dir`、`fid`），`get_file_info()` / `get_dir_info()` 可能返回 `None`，调用方必须判空。
- **不要无故访问 `AVAILABLE_DRIVES`**：首次访问会触发全量探测导入 20+ 第三方 SDK；查询支持列表请用 `get_supported_drives()` 或 `DRIVE_SPECS.keys()`。
- **凭据配置优先级**：构造函数显式参数 > `funsecret.read_secret("fundrive.<drive>.<key>", namespace="fundrive")` > 环境变量 `FUNDRIVE_<DRIVE>_<KEY>`。
- **夸克网盘**：直接 `uv add fundrive-quark` 并 `import fundrives.quark`，不要用 `fundrive[quark]`。

## 3. 开发硬约束（修改代码前必守）

1. **接口与注册表契约（`apps/fundrive`）**：
   - `BaseDrive` 方法签名必须与 `src/fundrive/core/base.py` 兼容，保留位置参数顺序并接受 `*args, **kwargs`（由 `tests/test_drive_contract.py` 做 AST 静态检查）。
   - 新增驱动必须同时登记到 `src/fundrive/drives/__init__.py` 的 `DRIVE_SPECS` **与** `__all__`（由 `tests/test_registry.py` 校验）。
   - 所有 HTTP 请求必须显式设置 `timeout`，日志和异常消息禁止输出密钥/token/cookie（由 `test_http_timeouts.py` 与 `test_credential_logging.py` 把关）。
2. **PEP 420 命名空间禁令（五个驱动包）**：
   - **严禁**在任何驱动包的 `src/fundrives/` 下创建 `__init__.py`，否则会遮蔽另外四个驱动包。
3. **统一 Ruff 基线**：
   - 6 个 app 的 `[tool.ruff.lint]` `select` 与 `ignore` 必须逐字一致，修改需同步全部仓库并跑 `bash scripts/check-lint-baseline.sh`。
4. **子模块提交顺序与发布纪律**：
   - 改动 `apps/<name>` 必须**先在子模块内 commit + push**，再回根仓库 `git add apps/<name>` 提交指针，顺序不可颠倒。
   - 不要手改 `scripts/funbuild.toml` 的版本号；发布统一走 `bash scripts/build.sh`（`funbuild build`）。
   - `APP_REGISTRY`（`scripts/setup.sh`）是 app 清单唯一来源，修改后同步文档表格并跑 `bash scripts/check-apps.sh`。

## 4. 校验步骤

改完代码后按范围运行对应验证命令，全部通过才算完成：

```bash
# 根仓库治理与脚本检查
for f in scripts/*.sh; do bash -n "$f"; done
shellcheck scripts/*.sh
bash scripts/check-apps.sh
bash scripts/check-links.sh
bash scripts/check-lint-baseline.sh

# 单个 app 的 lint、格式与单测（以 apps/fundrive 为例）
cd apps/fundrive
uv sync --group dev
uv run ruff check src tests
uv run ruff format --check src tests
uv run pytest -v

# 跨 app 联调安装与命名空间共存检查（仓库根目录）
uv venv
bash scripts/setup.sh install-dev all
.venv/bin/python -c "import fundrive, fundrives.aliopen, fundrives.baidu, fundrives.lanzou, fundrives.pan115, fundrives.quark; assert len(list(fundrives.__path__)) == 5"
```
