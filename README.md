# g-skills

个人跨工具 Agent Skills 源码仓库。每项技能独立封装，维护一份源码，按需安装到 Codex、Claude Code 等支持 `SKILL.md` 的客户端。

## 目录结构

```text
g-skills/
├── README.md
├── AGENTS.md
└── skills/
    └── idea-mcp-first/
        ├── SKILL.md
        ├── agents/
        │   └── openai.yaml
        └── references/
            └── debugging.md
```

- `skills/<skill-name>/SKILL.md`：技能入口，说明适用场景、工作流程和约束。
- 技能内的 `references/`：按需读取的详细规范与操作说明。
- 技能内的 `scripts/`：该技能专属的可执行辅助程序，按需增加。
- 技能内的 `assets/`：用于生成输出的模板、图片等素材，按需增加。
- 技能内的 `agents/`：客户端专属配置；`openai.yaml` 是 OpenAI 扩展，通用流程不依赖它。

有实际需求时再增加根目录 `scripts/`（安装、校验等仓库管理工具）和 `tests/`（脚本测试、行为验证用例），不预建空目录。

## 技能索引

| 技能 | 用途 | 依赖 |
| --- | --- | --- |
| [idea-mcp-first](skills/idea-mcp-first/SKILL.md) | 优先使用 IntelliJ IDEA MCP 完成项目导航、语义分析、诊断与调试 | 客户端已连接提供相应能力的 IDEA MCP 服务；工具不可用时按技能说明回退 |

## 安装与使用

本仓库的 `skills/` 是源码目录。将选定的整个技能目录复制或软链接到客户端配置的技能搜索目录，再按照该客户端的机制加载或刷新。加载目录及软链接支持以所用客户端版本为准。

例如，在仓库根目录执行以下命令，将技能复制到自己选定的技能搜索目录；先将占位路径替换为实际路径：

```sh
SKILL_TARGET_DIR='/替换为客户端的技能搜索目录'
mkdir -p "$SKILL_TARGET_DIR"
if [ -e "$SKILL_TARGET_DIR/idea-mcp-first" ] || [ -L "$SKILL_TARGET_DIR/idea-mcp-first" ]; then
  printf '%s\n' '目标已存在，请先检查现有技能。'
else
  cp -R skills/idea-mcp-first "$SKILL_TARGET_DIR/"
fi
```

使用复制方式安装后，仓库更新不会自动同步到安装目录；重新安装前先检查目标中的本地修改。技能中的 MCP 工具、脚本运行环境和权限仍需由客户端提供，安装技能本身不会安装这些依赖。

## 编写约定

- 技能目录名与 `SKILL.md` 中的 `name` 一致，使用小写英文、数字和连字符。
- `SKILL.md` 使用包含 `name`、`description` 的 YAML 头部；描述明确说明何时使用。
- 每项技能可单独复制使用，资源使用技能目录内的相对路径，避免引用其他技能或个人机器的绝对路径。
- 通用流程不绑定客户端专属工具前缀；具体工具名称、参数和可用性以当前客户端暴露的接口为准。
- 详细说明放在 `references/` 并从入口链接；无需为每项技能重复创建 README。
- 不收录真实密钥、生产地址、个人缓存或生成的临时产物。

格式参考：[Agent Skills 规范](https://agentskills.io/specification)。
