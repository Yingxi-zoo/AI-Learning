# Claude Code 操作命令手册

> **来源**：B 站视频《Claude Code 从 0 到 1 全攻略：MCP / SubAgent / Agent Skill / Hook / 图片 / 上下文处理 / 后台任务》
> 视频地址：https://www.bilibili.com/video/BV14rzQB9EJj/
> **整理说明**：该视频官方接口未提供字幕轨道，本手册依据观众转写的视频文字版（微信公众号）提取全部操作，命令以官方标准用法为准。

---

## 一、环境搭建与基础交互

1. `curl -fsSL https://claude.ai/install.sh | bash` — 官方一键安装 Claude Code 到本机终端环境。
2. `brew install --cask claude-code` —（macOS）用 Homebrew 安装，比 `curl | bash` 更规范的替代方案。
3. `claude` — 在终端启动 Claude Code，进入项目目录后运行即可让其在本项目上下文中工作。
4. `mkdir my-todo && cd my-todo` — 新建并进入项目目录，再启动 `claude`，让 AI 在正确的工程上下文中干活。
5. `/login` — 首次启动未自动提示登录时，手动触发登录授权流程；支持订阅账号（Pro/Max）或 API Key 两种接入。
6. `Shift + Tab` — 在三种模式间循环切换：
   - **默认模式**：每次文件创建/修改都需你确认，安全可控，适合第一次进陌生项目；
   - **自动模式**：会话内文件编辑自动通过，可连续改代码，但**终端命令仍会确认**（文件权限 ≠ 命令权限）；
   - **规划模式（Plan Mode）**：先出方案、不直接改文件，适合重构 / 迁移 / 大模块新增。
   - 补充：也可用启动参数 `--permission-mode default|acceptEdits|bypassPermissions|plan` 固定模式。

## 二、复杂任务处理与终端控制

7. `!` 前缀（如 `!ls`、`!npm install`、`!npm run dev`、`!open index.html`）— 在 Claude Code 内进入 Bash 模式执行终端命令，让它真正"帮你做"而不是只"说怎么做"。
8. `claude --dangerously-skip-permissions` — 启动时跳过所有权限检测，进入"权限绕过"状态，执行命令不再频繁确认；是效率开关也是风险开关，仅限本地可控沙盒 / 实验目录使用。
9. `npm run dev &`（或交互中把长命令放后台）— 把持续运行的开发服务挂到后台，避免占用前台终端阻塞 Claude Code；用 `/tasks` 查看后台任务列表并在面板中停止。
10. `/rewind` — Rewind 版本回滚：回到指定请求节点的变更点，可选"同时回滚代码和会话 / 只回滚代码 / 只回滚会话 / 放弃回滚"；**注意它不是 Git**，无法恢复 `mkdir`、`npm install` 等终端副作用产生的残留。

## 三、多模态与上下文管理

11. 拖拽图片 / `Ctrl + V` 粘贴图片（macOS 也是 **Ctrl+V**，不是 Cmd+V）— 把设计稿或截图作为图像输入，直接下指令"按这张图修改页面"；适合快速近似还原、调整视觉方向。
12. `claude mcp add --transport http figma https://mcp.figma.com/mcp` — 安装 Figma 的 MCP Server，让 Claude Code 读取结构化设计信息（组件 / 字体 / 间距 / 样式），从"凭截图猜"升级为"依据设计上下文还原"。
13. `claude -c` — 启动时直接 continue（继续）最近一次会话，免去重新讲项目背景和需求。
14. `/resume` — 在会话内恢复历史会话，从历史列表中选择并回到之前的对话；装完 MCP 重启后用它回到原会话。
15. `/mcp` — 查看当前已连接的 MCP 工具，确认 Figma 相关工具并完成授权。
16. `/compact` — 压缩当前上下文：保留核心需求、提炼关键信息、压掉冗长日志与中间过程，适合同一项目继续推进；`Ctrl + o` 可查看压缩后的上下文内容。
17. `/clear` — 彻底清空上下文，适合切换到完全无关的任务 / 目录，避免旧上下文干扰判断。
18. `CLAUDE.md`（项目级 `./CLAUDE.md`、用户级 `~/.claude/CLAUDE.md`）— 项目记忆文件，把技术栈 / 目录约束 / 输出语言 / 风格偏好等反复约定固化成长期记忆；可用 `/init` 让 Claude Code 分析项目后生成初版，再自行补充。

## 四、高级功能扩展与定制

19. Hook（事件钩子）— 在特定事件自动触发预定义逻辑（格式化、lint、补文件头、跑测试、发通知等）。配置方式：会话内 `/hooks` 命令，或在 `.claude/settings.json` / `~/.claude/settings.json` 的 `hooks` 字段配置；事件含 `PreToolUse`、`PostToolUse`、`Notification`、`Stop`、`SubagentStop`、`UserPromptSubmit`。
20. Agent Skill（技能）— 把高频固定套路的输出沉淀成"能力说明书"（含元信息 + 具体要求两部分）。存放于 `项目级 .claude/skills/<name>/SKILL.md` 或 `用户级 ~/.claude/skills/<name>/SKILL.md`；通过对话自动触发或 `/skill` 主动调用，适合日报 / 周报 / 会议纪要等格式化高频任务。
21. SubAgent（子代理 / 分身）— 派一个拥有独立上下文与工具权限的"专项顾问"去干重分析任务。存放于 `项目级 .claude/agents/<name>.md` 或 `用户级 ~/.claude/agents/<name>.md`；用 `/agents` 查看 / 创建，内置 `Explore`、`Plan`、`general-purpose`，适合代码审核、大目录分析、安全 / 风险检查。
22. Skill 与 SubAgent 的区别 — **Skill 共享主会话上下文**（套路，适合轻量、格式化、高频任务：日报 / 周报 / 总结）；**SubAgent 有独立上下文**（分身，适合重分析、大量内容处理：大项目审核 / 风险检查 / 文档扫描）。口诀：高频格式任务用 Skill，深度分析任务用 SubAgent。
23. Plugin（插件 / 能力包）— 把 Skill、SubAgent、Hook 等组件及配置打包成一键安装包，适合团队复用；通过 `/plugin` 命令从 marketplace 安装 / 管理。

---

## 附：命令速查表（随时查阅）

| 命令 | 作用 |
|------|------|
| `curl -fsSL https://claude.ai/install.sh \| bash` | 一键安装 |
| `brew install --cask claude-code` | Homebrew 安装（macOS） |
| `claude` | 启动 Claude Code |
| `/login` | 手动登录授权 |
| `Shift + Tab` | 切换 默认 / 自动 / 规划 模式 |
| `!command` | 在对话内执行终端命令 |
| `claude --dangerously-skip-permissions` | 跳过所有权限检测 |
| `npm run dev &` + `/tasks` | 后台运行服务并管理后台任务 |
| `/rewind` | 回滚到指定对话节点 |
| 拖拽 / `Ctrl + V` | 粘贴图片（macOS 也是 Ctrl+V） |
| `claude mcp add --transport http <name> <url>` | 安装 MCP Server |
| `claude -c` | 启动时继续上次会话 |
| `/resume` | 会话内恢复历史会话 |
| `/mcp` | 查看已连接 MCP 工具 |
| `/compact`（配合 `Ctrl + o`） | 压缩上下文并查看 |
| `/clear` | 清空上下文 |
| `CLAUDE.md` / `/init` | 项目记忆文件及初始化 |
| `/hooks` | 配置事件钩子 |
| `.claude/skills/<name>/SKILL.md` + `/skill` | Agent Skill 定义与调用 |
| `.claude/agents/<name>.md` + `/agents` | SubAgent 定义与查看 |
| `/plugin` | 安装 / 管理插件包 |

> **使用心法**（来自视频结语）：小任务直接做，大任务先 Plan Mode；`dangerously-skip-permissions` 只当实验环境加速器；尽早写好 `CLAUDE.md`；重复输出用 Skill，重分析用 SubAgent；设计还原尽量接 MCP；Rewind 只是应急，Git 仍是正式版本控制核心。
