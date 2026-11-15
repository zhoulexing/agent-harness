# Packages

当前有三个包：`llm` 定义统一模型契约，并由 `adapters/pi-ai/` 对接真实协议；`core` 驱动轮次和工具；`session` 保存事件、投影消息并恢复中断。依赖方向为 `core → llm/session`、`session → llm/contract`，装配代码位于 `apps/cli/`。扩展边界见 [项目 README](../README.md#扩展边界)。
