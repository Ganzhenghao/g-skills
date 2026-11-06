# IDEA MCP 按需工作流

仅阅读当前任务涉及的小节；参数和能力以当前注册接口为准。

## 符号与依赖源码

- `search_symbol` 的结构化结果是路径和坐标，没有声明 FQN。缺少完整符号名时，用返回位置调用 `get_symbol_info`，必要时局部读取包名、所属类型和声明，补齐后再调用 `analyze_calls`。
- `get_symbol_info` 的信息取决于语言，不能假定一定返回完整声明或 FQN；检查 `partialResultReason`。
- 外部符号在项目内无合适结果时再启用 `include_external=true`。对返回的 JAR/JRT、源码或反编译路径直接使用 `read_file(file_path=...)`，不要自行拼接归档路径；按需分段读取。

## 调用关系与续查

- `analyze_calls` 用 `INCOMING_CALLS` 查调用方，`OUTGOING_CALLS` 查被调用方。`symbolFqn` 使用完整名称，重载歧义时使用工具返回的精确签名。
- 展开时原样复制返回的 `treePath`，用 `childOffset` 继续读取该节点的直接子项，保持范围限制；不要每次重新请求整棵树。
- 输出被深度、子节点数或总节点数限制时，只对相关分支继续展开；有截断时不声称已列出全部调用方。
- 调用层级不等于全部引用。字段访问、配置引用等按语义需求补充检查；静态调用关系不能证明所有动态运行路径。

## 语义重命名

- 修改前读取目标。优先传唯一匹配且包含 `symbolName` 的 `contextSnippet`；否则使用已核实的 1-based `line` 和 `column`。无定位信息时，文件必须只有一个对应声明。
- 先设 `preview=true`、`applyAutomaticRenamers=false`，检查影响与冲突；在任务授权范围内再设 `preview=false` 执行，不为常规步骤额外请求许可。
- 多候选时重复原参数并增加返回的 `targetIndex`；冲突未解决前不执行。额外自动重命名仅在任务需要且已核对影响时启用。
- 执行以 `applied=true` 为成功信号，不能只看 `ok`。`error.kind=write_failed` 可能已部分写入，按提示核对实际差异后再决定下一步，不盲目重试或全局文本替换。
- 执行后检查实际差异，只验证本次修改范围；刷新受影响的符号位置和已读内容。

## 检查、构建与格式化

- `lint_files` 一次传入涉及的文件；`min_severity` 仅使用 `warning`、`strong_warning` 或 `error`，按所需检查范围选择。
- 核对请求文件均有返回条目；空 `items` 或遗漏文件不能证明这些文件已检查。逐项检查 `timedOut`、`notAnalyzedReason`，并检查顶层 `more`。空 `problems` 只在该文件确已完整分析时才表示该次检查没有问题；不完整时缩小批次或按能力回退。
- 需要编译反馈时用 `build_project`，可用 `filesToRebuild` 缩小范围；除非任务需要，不做全量重建。结合 `isSuccess`、`problems` 和 `timedOut` 判断结果，不能将超时或未知状态视为成功。
- 构建不能代替测试，IDE 检查也不能代替仓库要求的 SDK、构建和验收流程；文档等不需要编译的改动不触发构建。Java 命令遵守仓库的 SDKMAN 与环境检查规则。
- 格式化和导入整理只传入任务涉及且确需处理的文件。`optimize_imports` 只删除、排序已有导入，不补充缺失导入；`reformat_file` 按整个文件格式化。
- 同时需要格式化及整理导入时，可用 `reformat_file(optimizeImports=true)` 减少重复调用。两者的差异输出都可能在 200 行后截断，需要完整核对时使用原生 diff。

## 普通运行与定向测试

- 优先发现目标文件的运行点：`get_run_configurations(filePath=...)`；需要已有配置时才省略 `filePath`。复用已获得且仍有效的配置或入口信息。
- `execute_run_configuration` 只选择一种定位方式：准确的 `configurationName`，或返回的 `filePath` 与 1-based `line`；两种方式不能混用。
- 默认 `waitForExit=true`，按任务设有界 `timeout`。启动命令、环境和输入必须属于任务允许的执行范围；不要把启动本身当作验证完成。
- 默认不传 `programArguments`、`workingDirectory`、`envs`。仅在所选配置明确返回 `supportsDynamicLaunchOverrides=true` 且任务确需覆盖时传入；文件运行点没有该能力证据时不传。
- 覆盖只对本次运行生效：字符串覆盖参数缺失、null 或空串保留原值，空白字符串才会清空；`envs` 与已有环境合并。不要用空串模拟清空或误改启动值。
- 等待超时后进程可能仍在后台运行；没有 `exitCode` 时只报告已启动或尚未观察到退出，不立即重复启动。当前注册工具没有普通运行会话的查询或停止接口，不猜测调用。
- 需要更多输出时可用原生读取工具分段读取返回的 `fullOutputPath`，它可能继续增长；日志或退出码仍不足以证明测试覆盖，核对具体测试结果。
- 保留用户原有运行配置和进程，不调用调试工具获取状态或停止进程；涉及生产、破坏性操作等继续遵守仓库授权规则。
