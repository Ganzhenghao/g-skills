---
name: idea-mcp-first
description: 当客户端已连接提供相应能力的 IntelliJ IDEA MCP 服务时，优先使用 IDE 索引完成项目导航、符号与调用分析、编辑器上下文、诊断、重构、运行配置与调试；用户要求、仓库规范和授权边界优先。
---

# IDEA MCP 优先

当 IntelliJ IDEA MCP 服务可用时，优先利用 IDEA 的项目模型、索引和调试器完成适合 IDE 感知的工作。不要为了使用 MCP 扩大任务范围；用户要求、仓库规范、构建流程和权限规则始终优先。

## 客户端与工具适配

- 本技能要求客户端已连接提供相应能力的 IDEA MCP 服务；技能本身不会安装或配置服务。
- 下文工具名表示对应能力。客户端可能添加服务前缀，也可能仅暴露部分工具；调用前核对当前工具的真实名称和参数，不构造不存在的工具调用。
- 工具名或参数与下文不同但存在等价能力时，按当前接口使用；缺少对应能力时，遵循下述回退规则。

## 选择原则

- 已知项目目录且工具支持 `projectPath` 时，每次调用都传入该参数，避免多项目歧义；接口不支持时，使用该接口提供的项目选择机制。
- 优先使用语义层级最高且范围最窄的工具：调用关系用 `analyze_calls`，符号定位用 `search_symbol`，不要先退化为文本或正则搜索。
- 修改前先读取目标及相关调用方；只读任务不要调用写入、重构、运行或调试工具。
- 工具失败、索引结果不完整、文件不在 IDEA 项目范围内，或没有等价能力时，回退到仓库允许的 `rg`、`sed`、构建命令等，并说明会影响结论的回退。

## 工具路由

| 需求 | 优先工具 | 使用要点 |
| --- | --- | --- |
| 模块、依赖、VCS 根和目录结构 | `get_project_modules`、`get_project_dependencies`、`get_repositories`、`list_directory_tree` | 先确定模块边界，再搜索或修改 |
| 文件定位 | `search_file` | 使用 glob 匹配项目路径 |
| 精确文本或正则搜索 | `search_text`、`search_regex` | 已知字面量用文本；只有模式匹配才用正则 |
| 类、方法、字段等符号定位 | `search_symbol` | 用于获得定义位置或全限定名 |
| 符号语义和文档 | `get_symbol_info` | 需要类型、签名或 Quick Documentation 信息时使用 |
| 调用方或被调用方 | `analyze_calls` | 用 `INCOMING_CALLS` / `OUTGOING_CALLS`；歧义时先 `search_symbol` |
| 读取源码、依赖源码或反编译类 | `read_file` | 支持项目文件、JAR/JRT 源码和反编译内容；大文件按范围读取 |
| 当前编辑器上下文 | `get_all_open_file_paths` | 仅在用户提到当前打开文件或编辑器上下文时使用 |
| 在 IDEA 中定位文件 | `open_file_in_editor` | 仅在用户要求打开、展示或定位具体文件时使用 |
| 新建文件 | `create_new_file` | 仅在任务要求新增文件时使用；已有文件的精确改动使用客户端提供的补丁式编辑工具 |
| 代码符号重命名 | `rename_refactoring` | 明确要求重命名时优先，依赖 IDE 更新引用；不要用于普通文本替换 |
| 单文件或批量诊断 | `get_file_problems`、`lint_files` | 单文件用前者，多文件用后者；检查超时、未分析和不完整标记 |
| 格式化 | `reformat_file` | 只格式化任务范围内的文件，并遵守仓库对无关格式变动的限制 |
| IDEA 增量构建 | `build_project` | 用作快速编译反馈，不替代仓库规定的最终构建或测试 |
| 运行入口和配置 | `get_run_configurations` | 无 `filePath` 时列配置，有 `filePath` 时找可运行代码位置 |
| 执行运行配置 | `execute_run_configuration` | 使用查询返回的准确配置名或文件行号；仅在任务允许执行时调用 |
| IDEA 集成终端 | `execute_terminal_command` | 仅当 IDE 环境本身有价值时使用；普通 Git 和最终验收沿用仓库终端流程 |
| 动态工具调用 | `execute_tool` | 仅在直接工具无法表达调用或接口要求动态分派时使用 |

## 修改与验证边界

- 补丁式修改优先使用客户端提供的原生编辑工具，如 `apply_patch`；仓库或会话指定编辑方式时遵从其要求。不要用 IDEA MCP 包装、模拟或绕过指定的编辑工具。
- 新建文件可使用 `create_new_file`；程序符号重命名可使用 `rename_refactoring`。执行前确认它们属于用户要求的写入范围。
- 修改后先用 `get_file_problems` 或 `lint_files` 获取快速反馈；必要时用 `build_project`。最终构建、测试和验收必须执行仓库指定的 SDK、Maven、Gradle、Node 等流程。
- Git 状态、diff、暂存、提交和推送使用正常终端及授权流程，不使用 IDEA MCP 绕过仓库审计要求。
- 数据库、网络、生产环境、破坏性操作及需要审批的命令继续遵守会话权限边界。

## 调试

只有任务需要运行时证据、复现或调试时，才读取并遵循 [references/debugging.md](references/debugging.md)。普通代码阅读、评审和静态排查不要启动调试会话。
