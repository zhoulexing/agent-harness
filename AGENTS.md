# Agent Harness 工程约定

本仓库是独立的 Agent Harness 实验工程。先读 README.md；各包职责见 packages/README.md，验证范围见 tests/README.md。

- 修改前核对分支和已有改动。模型协议留在 packages/llm，轮次、工具和权限留在 packages/core，会话持久化留在 packages/session；apps/cli 负责装配。
- 安装依赖：pnpm install --frozen-lockfile；离线验证：pnpm test。macOS 沙箱测试在其他系统跳过，不算跨平台验证通过。
- pnpm test:live 会调用真实模型，须有本次任务授权和显式模型配置。离线测试通过不代表真实模型或终端交互通过。
- 不放宽权限或在拒绝后换工具绕过。取消及中断后，不自动重试结果未知的工具操作。
- 凭据通过环境变量或被忽略的本地 .env 提供；不提交 .env、会话日志、生成结果或依赖目录。
- 工程不依赖个人 Home Repo 的目录布局。CLAUDE.md 链接本文件，共用规则。
