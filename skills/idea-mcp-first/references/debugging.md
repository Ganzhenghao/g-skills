# IDEA MCP 调试工作流

调试会启动并控制真实进程，必须保持在用户任务允许的运行范围内。不要把能够通过静态分析回答的问题升级为调试。

## 启动前

1. 用 `get_run_configurations` 查询准确的配置名，或查询目标文件的可运行行号。
2. 需要暂停时，先用 `xdebug_set_breakpoint` 设置断点，再调用 `xdebug_start_debugger_session`。若只需采集运行值，优先使用 `suspendPolicy=NONE` 的无副作用日志点，减少对进程的干扰。
3. 只有查询结果明确声明支持动态覆盖时，才传 `programArguments`、`workingDirectory` 或 `envs`。

## 会话控制与取证

- 用 `xdebug_get_debugger_status` 获取当前会话和状态；存在多个会话时始终传准确的 `sessionId`。
- 用 `xdebug_control_session` 等待暂停、继续、单步或排空事件。不要无界轮询。
- 暂停后依次按需使用 `xdebug_get_threads`、`xdebug_get_stack`、`xdebug_get_frame_values`。
- 深入展开对象用 `xdebug_get_value_by_path`；计算只读表达式用 `xdebug_evaluate_expression`。表达式不得产生写入、I/O 或迭代器推进等副作用。
- `xdebug_run_to_line` 会改变暂停位置；之后重新获取调用栈、栈帧索引和变量路径，不复用旧值。
- `xdebug_set_variable` 会修改运行时状态，仅在用户任务明确需要状态实验时使用，并立即重新读取验证；不得把变更后的行为当作原始程序证据。

## 断点管理

- 用 `xdebug_list_breakpoints` 区分用户断点与工具创建的断点。
- `xdebug_set_breakpoint` 返回成功后，核对 `lineText`；条件或日志表达式还应通过 `xdebug_control_session` 的事件检查异步错误。
- 用 `xdebug_remove_breakpoint` 只移除本次任务创建的断点。不要删除或改写用户已有断点，除非用户明确要求。

## 结束与结论

- 调试完成后停止本次启动的会话，并清理本次创建的临时断点。
- 区分静态推断、实际观测和通过 `xdebug_set_variable` 人为构造的结果；报告使用的运行配置、关键输入和未覆盖的外部依赖。
