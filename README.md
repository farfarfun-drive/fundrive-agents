# fundrive-skills

面向 AI Agent（Antigravity / Codex / Claude Code 等）的 [fundrive](https://github.com/farfarfun-drive/fundrive) 生态 Skill 集合，按 `skills/<skill-name>/` 目录组织，可在一个仓库内持续扩展多个 Skill。

## Skills

| Skill | 适用场景 | 入口 |
| --- | --- | --- |
| `fundrive` | `fundrive` 核心库与 `fundrives.*` 驱动包的接入使用、驱动开发、契约测试、Ruff 基线及 `fundrive-dev` 子模块编排 | [skills/fundrive/SKILL.md](skills/fundrive/SKILL.md) |

## 目录结构

```text
fundrive-skills/
├── .github/workflows/validate.yml   # 自动校验所有 skills/*/SKILL.md、openai.yaml 与相对链接
├── skills/
│   └── <skill-name>/
│       ├── SKILL.md                 # Skill 主入口（YAML frontmatter + 核心步骤）
│       ├── agents/openai.yaml       # Agent UI 元数据
│       └── references/              # 按需加载的详细参考文档
├── LICENSE
└── README.md
```

## 安装

### 方式一：使用 `skills` CLI（推荐）

```bash
npx skills add farfarfun-drive/fundrive-skills
```

### 方式二：克隆并安装到本地 Agent 目录

```bash
git clone https://github.com/farfarfun-drive/fundrive-skills.git
cd fundrive-skills

# Antigravity（项目级）
mkdir -p /path/to/your-project/.agents/skills
cp -R skills/* /path/to/your-project/.agents/skills/

# Codex（全局软链，拉取仓库后自动生效）
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
for skill in skills/*; do
  ln -s "$(pwd)/$skill" "${CODEX_HOME:-$HOME/.codex}/skills/$(basename "$skill")"
done

# Claude Code（全局）
mkdir -p ~/.claude/skills
cp -R skills/* ~/.claude/skills/
```

## 新增 Skill

在 `skills/<new-skill-name>/` 下创建 `SKILL.md`（frontmatter 的 `name` 必须与目录名一致）和 `agents/openai.yaml`，按需在 `references/` 放置长文档，并在本 `README.md` 的 Skills 表格中登记。CI 的 [validate.yml](.github/workflows/validate.yml) 会自动扫描 `skills/*` 校验元数据与 Markdown 相对链接。

---

## 关于 farfarfun-drive

[farfarfun](https://github.com/farfarfun) 的网盘 / 云存储产品线：以
[fundrive](https://github.com/farfarfun-drive/fundrive) 的统一 `BaseDrive` 接口为核心，
配套各网盘的独立驱动包与蓝奏云 SDK / 桌面客户端。

- 🏠 组织主页：<https://github.com/farfarfun-drive>
- 📦 PyPI：<https://pypi.org/user/niuliangtao/>
- 📧 联系：farfarfun@qq.com

本项目基于 [MIT](LICENSE) 协议开源。
