---
name: idea-mcp-first
description: 当客户端已连接 IntelliJ IDEA MCP 时，按任务使用 IDE 索引完成代码导航、符号与调用分析、依赖源码阅读、检查、重构和运行测试；与原生终端和文件编辑工具协同。
---

# IDEA MCP 优先

按任务选择范围最窄的 IDEA 语义或索引能力，减少重复搜索和读取。用户要求、仓库规范、构建环境及授权边界优先；只读分析不调用写入或运行工具。

## 适配与调用原则

- 依赖已连接的 IDEA MCP 服务，以及原生终端、Git 和文件编辑工具；技能本身不安装服务或修改客户端配置。
- 下文省略客户端工具前缀；调用前核对当前注册的真实名称和参数，不猜测工具或添加不存在的参数。
- 已知项目目录且接口支持时，每次传入 `projectPath`；接口不支持时使用其项目选择机制。
- 已知路径或完整符号名时直接读取或分析，跳过无关搜索。不要固定查询全部模块、依赖、仓库及编辑器信息。
- 项目元数据和已读片段在同一任务内复用，相关状态变化后再刷新。
- 工具缺失、失败、不支持目标或结果不完整时，按能力回退到原生工具；不循环重试，并说明影响结论的缺口。不使用调试工具。

## 工具路由

| 任务意图 | 工具与策略 |
| --- | --- |
| 已知文件，阅读实现 | `read_file` 局部读取；普通本地文件也可用原生读取工具 |
| 文件路径未知 | `search_file`，使用路径 glob |
| 类、方法、字段定位 | `search_symbol`；需要类型、签名或声明信息时再用 `get_symbol_info` |
| 谁调用谁 | 已知完整符号名时直接 `analyze_calls`，否则先定位符号 |
| 配置键、SQL、字符串 | `search_text`；需要模式匹配才用 `search_regex`，需要匹配行上下文可用原生 `rg` |
| 第三方实现 | 项目内未找到合适符号时，`search_symbol(include_external=true)`，再读取返回的依赖路径 |
| 新建或修改文件 | 使用原生创建或局部编辑工具；已有文件不通过整文件覆盖模拟补丁 |
| 符号重命名 | `rename_refactoring`，先预览再执行，默认 `applyAutomaticRenamers=false` |
| 检查改动 | 对涉及文件批量 `lint_files`，单文件也使用它；按改动性质选择构建和相关测试 |
| 格式化或整理导入 | 按需 `reformat_file` 或 `optimize_imports`，不作为每次修改后的固定步骤 |
| 定向测试或运行 | `get_run_configurations` 发现准确配置或运行点，再用 `execute_run_configuration` |
| 用户提到当前编辑器 | 才用 `get_all_open_file_paths` |
| 陌生项目结构 | 按需 `list_directory_tree`、`get_project_modules`、`get_project_dependencies` 或 `get_repositories` |
| Git 与变更归属 | Git 使用原生终端；提交或分析分组归属时才读 `get_changelists`，不自动管理分组 |

涉及外部符号、调用树续查、重命名、检查、构建或运行时，读取 [references/workflows.md](references/workflows.md) 的对应小节，核对结果与失败边界。

## 初始调用预算

以下是显式传入的起始值，需要更多证据时再扩大；不是工具默认值或硬性上限。

| 工具 | 初始参数 |
| --- | --- |
| 四类 `search_*` | `limit=20`，已知目录或模块时同时传 `paths` |
| `read_file` | `offset` 指向目标附近（从 1 开始），`limit=120` |
| `analyze_calls` | `depth=2`、`maxChildren=10`、`maxNodes=60` |
| `list_directory_tree` | `maxDepth=2`，优先目标目录 |
| `get_changelists` | `includeFiles=false`；检查文件归属时再设为 `true`，并按需限制 `maxFilesPerChangelist` |

- 搜索的 `more=true` 时先缩小路径或查询，确需更多结果再提高 `limit`；搜索接口没有分页 `offset`。结果只声明路径和坐标，不能假定包含源码上下文或 FQN。
- 检查 `more`、`partialResultReason`、`timedOut`、`listingTimedOut` 等适用标记；未完成的空结果不能作为不存在问题的证据。
- `get_project_modules` 只声明模块名称和类型；`get_project_dependencies` 没有过滤或分页参数，避免为普通定位拉取全部依赖。

## 工具注册方案

只在调整工具暴露范围时读取 [references/tool-selection.md](references/tool-selection.md)：保留 20 项工具，changelist 只读，排除清单按当前注册结果核对。注册精简需要实际过滤层支持；技能文本不会移除工具定义或保证 token 收益，也不授权更新已安装技能或客户端配置。
