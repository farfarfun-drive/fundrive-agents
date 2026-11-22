# FunDrive 驱动开发与工作区维护参考

本文档总结在 `fundrive-dev` 工作区开发新驱动、修改既有驱动以及跨仓库协作的完整流程与硬性约束。详细文档见：
- [002-local-development.md](https://github.com/farfarfun-drive/fundrive-dev/blob/master/docs/project/project-governance/002-local-development.md)
- [003-release-and-entrypoint.md](https://github.com/farfarfun-drive/fundrive-dev/blob/master/docs/project/project-governance/003-release-and-entrypoint.md)
- [004-known-deviations.md](https://github.com/farfarfun-drive/fundrive-dev/blob/master/docs/project/project-governance/004-known-deviations.md)
- [002-interface-contract.md](https://github.com/farfarfun-drive/fundrive-dev/blob/master/docs/development/driver-contract/002-interface-contract.md)
- [003-driver-development.md](https://github.com/farfarfun-drive/fundrive-dev/blob/master/docs/development/driver-contract/003-driver-development.md)

## 1. 本地开发环境准备

### 初始化与更新子模块

```bash
# 首次 plain clone 后初始化全部子模块
bash scripts/init.sh

# 将 5 个子模块统一快进到各仓库 master 最新（手工开发推荐）
git submodule foreach 'git checkout master && git pull --ff-only'
```

> 警告：拉取子模块新 commit 后、尚未在根仓库 `git add apps/` 提交指针前，**不要运行 `bash scripts/init.sh`**，它会把子模块退回旧指针。

### 两种环境路径（不要混用）

1. **跨 app 联调（仓库根目录 `.venv`）**：
   ```bash
   uv venv
   bash scripts/setup.sh install-dev all     # 5 个 app 可编辑安装到根 .venv
   bash scripts/setup.sh install-dev baidu   # 仅装单个 app
   ```
   `scripts/setup.sh` 要求明确目标环境（激活的 venv、根 `.venv` 或 `FUNDRIVE_INSTALL_SYSTEM=1`），不会悄悄装进系统 Python。
2. **单 app 独立测试（`apps/<name>/.venv`）**：
   ```bash
   cd apps/fundrive
   uv sync --group dev                       # 裸环境（不装可选 SDK，最贴近 CI）
   uv run pytest -v
   ```

> 依赖不锁死：5 个 app 的 `uv.lock` 均刻意在 `.gitignore` 中，不在版本控制内。

## 2. 在 `apps/fundrive` 中新增或修改内置驱动

### 目录结构

```text
apps/fundrive/src/fundrive/drives/<name>/
├── __init__.py    # 导出驱动类，如 from .drive import FooDrive; __all__ = ["FooDrive"]
├── drive.py       # 继承 BaseDrive 的主实现
├── README.md      # 说明根 fid 约定、认证方式、**kwargs 扩展与能力边界
└── example.py     # 可选，标准示例与 BaseDriveTest 测试入口
```

### 实现检查清单

1. **继承与构造**：
   - 继承 `fundrive.core.BaseDrive`，调用 `super().__init__(*args, **kwargs)`。
   - 初始化 `self._root_fid`、`self._is_logged_in`。
2. **签名向上兼容（`tests/test_drive_contract.py` 静态校验）**：
   - 重写 `BaseDrive` 公开方法时，位置参数顺序与名称必须与 `src/fundrive/core/base.py` 一致，且接受 `*args, **kwargs`。
   - 禁止发明平行方法名替代契约方法（如用 `copy_object` / `move_object` / `create_share_link` / `move_file` 代替 `copy` / `move` / `share`）。
3. **路径、超时与安全（对应契约测试均在 CI 中运行）**：
   - 下载路径解析统一使用 `fundrive.core.base.get_filepath` 与 `ensure_parent_dir`（避免裸文件名 `os.makedirs("")` 报错）。
   - 所有 HTTP 请求必须显式设置 `timeout`（`tests/test_http_timeouts.py`）。
   - 禁止在日志或异常消息中打印 token / secret / cookie / password（`tests/test_credential_logging.py`）。
   - 遵守登录契约（`tests/test_login_contract.py`）。
4. **驱动注册表（`tests/test_registry.py` 静态校验）**：
   - 在 `apps/fundrive/src/fundrive/drives/__init__.py` 中：
     - 将驱动加入 `DRIVE_SPECS`（`extra=None` 表示仅依赖核心依赖；若依赖第三方 SDK，填 extra 名）。
     - **同步将驱动类名加入 `__all__`**。
   - 如新增了 `extra`：在 `apps/fundrive/pyproject.toml` 的 `[project.optional-dependencies]` 中声明，并同步加入 `all` extra 与 `[dependency-groups] all`。
   - 确保 `import fundrive` 不会在导入期拉入该第三方 SDK。
5. **单元测试与文档**：
   - 在 `apps/fundrive/tests/test_<name>_drive.py` 补充单元测试，用 mock 覆盖核心读写路径与错误边界，确保在裸环境（`uv sync --group dev`）下可直接运行通过。
   - 同步更新 `apps/fundrive/README.md` 的驱动状态总览表。

## 3. 开发五个独立驱动包（`fundrives.*`）

- **PEP 420 命名空间约束**：`apps/fundrive-{alipan,baidu,lanzou,pan115,quark}/src/fundrives/` 下**绝对不能**放 `__init__.py`；所有包内代码放在 `src/fundrives/<portion>/` 中。
- **兼容性**：
  - `fundrive-alipan` / `fundrive-pan115`：`requires-python = ">=3.12"`
  - `fundrive-baidu` / `fundrive-lanzou` / `fundrive-quark`：`requires-python = ">=3.10"`，CI 会在最低 Python 版本上跑 `test-floor` 任务，禁止使用高于下界的语法或标准库 API。
- **核心库联动**：若修改了 `fundrives.{aliopen,baidu,lanzou,pan115}` 的公共 API，需同步核对 `apps/fundrive/src/fundrive/drives/{alipan,baidu,lanzou,pan115}/` 中的适配代码，并在根目录 `.venv` 中 `install-dev all` 联调验证。

## 4. 统一 Ruff 规则基线

五个 app 的 `[tool.ruff.lint]` 必须保持逐字相同的基线（详见 [004-known-deviations.md](https://github.com/farfarfun-drive/fundrive-dev/blob/master/docs/project/project-governance/004-known-deviations.md) 第 6 条）：

```toml
select = ["E4", "E7", "E9", "F", "I", "UP", "B", "SIM", "RET", "C4", "PLE", "T100", "S102"]
ignore = ["E501", "PLE1205"]
```

- 允许的差异仅限：`target-version`（对齐各自 `requires-python`）、特定 `per-file-ignores`、以及自愿加严的 `extend-select`（如 `fundrive-quark` 保留 `BLE`/`DTZ`）。
- 禁止在单个仓库私自加 `ignore` 绕过检查，CI 的 [check-lint-baseline.sh](https://github.com/farfarfun-drive/fundrive-dev/blob/master/scripts/check-lint-baseline.sh) 会拦截漂移。
- `BLE001` 与 `DTZ` 目前为有意识暂缓项，除非专项重构，否则不要擅自变更基线。

## 5. 跨仓库提交与发布纪律

### 提交顺序（不可颠倒）

```bash
# 1. 先在被修改的子模块内 commit 并 push 到该 app 自己的远端
git -C apps/fundrive add ...
git -C apps/fundrive commit -m "feat: 新增/修复 ..."
git -C apps/fundrive push origin master

# 2. 再回到根仓库更新指针（多个 app 协同改动合并成一个根 commit）
git add apps/fundrive
git commit -m "chore: 更新 fundrive 子模块指针"
git push origin master
```

### 新增 / 删除 app 时的清单维护

- 唯一来源：[setup.sh](https://github.com/farfarfun-drive/fundrive-dev/blob/master/scripts/setup.sh) 的 `APP_REGISTRY`。
- 手写副本需同步更新：`README.md`、`docs/project/project-governance/001-overview.md`、`docs/project/project-governance/003-release-and-entrypoint.md`。
- 运行 `bash scripts/check-apps.sh` 校验双向一致。

### 发布（共享版本号）

- 构建发布入口是 [build.sh](https://github.com/farfarfun-drive/fundrive-dev/blob/master/scripts/build.sh)（即 `funbuild build`），由它自动递增 `scripts/funbuild.toml` 的共享版本号并向 5 个子模块 fan out 构建、发布和统一 push。
- **禁止**手动调低或手工编辑 `scripts/funbuild.toml` 的版本号；版本号只能单调前进。
- 不要在单个 app 仓库中单独触发非工作区发版。
