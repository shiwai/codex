# Codex CLI 用户使用手册

> **温馨提示**：Codex CLI 仍在积极迭代中，功能和接口可能随时调整。请在使用过程中随时关注版本更新，并在遇到 Bug 或安全问题时及时反馈。

## 1. 项目概览与许可
- Codex CLI 是一个帮助开发者在本地仓库内运行 AI 搭档的开源工具，当前仍属于实验性项目，欢迎通过 issue、PR 或讨论区参与改进。
- 本仓库采用 Apache-2.0 许可，任何基于本项目的修改和再发布都需要遵循对应条款。
- OpenAI 正在运行 Codex 开源基金，面向使用 Codex CLI 或其他 OpenAI 模型的开源项目发放最高 25,000 美元等额 API 额度资助，申请入口：[openai.com/form/codex-open-source-fund](https://openai.com/form/codex-open-source-fund/)。

## 2. 安装与系统要求
- 支持的系统：macOS 12+、Ubuntu 20.04+/Debian 10+、以及 Windows 11（通过 WSL2）。官方建议至少 4 GB 内存，8 GB 更佳。
- 发行渠道：GitHub Releases、`@openai/codex` npm 包、Homebrew cask，都在发布脚本 `codex-rs/scripts/create_github_release` 驱动下生成；可按需使用 `--publish-alpha` 或 `--publish-release` 发布并通过 Homebrew 自动 PR 分发。
- DotSlash：GitHub Release 中包含 `codex` DotSlash 文件，方便团队在多平台统一 CLI 版本。
- 从源码构建：克隆 `codex` 仓库，进入 `codex/codex-rs`，安装 Rust 工具链与 `rustfmt`/`clippy`，执行 `cargo build`、`cargo run --bin codex -- "..."`，并用 `cargo fmt`、`cargo clippy --tests`、`cargo test` 校验修改。

## 3. 登录、认证与数据政策
- 默认走 ChatGPT OAuth，运行 `codex login` 后在浏览器完成流程。若希望按调用量计费，可以把 `OPENAI_API_KEY` 通过 `codex login --with-api-key` 管道输入，或从文件读取，CLI 会避免在历史中泄露密钥。
- 旧版 `--api-key` 已弃用，请更新到 0.20.0+ 并删除旧的 `~/.codex/auth.json` 后重新登录以切换到 ChatGPT 方案。
- 远程/无头环境可在本地完成登录后复制 `auth.json`，或用 `ssh -L 1455:localhost:1455 user@host` 转发端口以在本地浏览器打开 `localhost:1455`。
- `codex exec` 也可通过设置 `CODEX_API_KEY` 覆盖登录状态（仅对非交互模式生效）。
- Codex CLI 原生支持启用 Zero Data Retention (ZDR) 的组织，可在有需要时与 OpenAI 商务团队开通该能力。

## 4. 快速上手
### 4.1 基本命令
| 命令 | 功能 | 示例 |
| --- | --- | --- |
| `codex` | 启动交互式 TUI | `codex` |
| `codex "..."` | 带首条提示启动会话 | `codex "修复 lint"` |
| `codex exec "..."` | 非交互自动化 | `codex exec "解释 utils.ts"` |
| `codex resume` | 打开历史会话选择器 | `codex resume --last` |
| `codex resume <ID>` | 按 ID 恢复会话 | `codex resume 7f9f...` |


### 4.2 提示示例与技巧
- 参考文档中的 7 个示例提示了解 Codex 可以执行的典型任务，例如重构 React 组件、编写 SQL 迁移、生成单元测试、批量重命名等。
- 使用 `@` 触发模糊文件搜索，`Esc Esc` 回溯修改上一条输入，`--cd` 切换工作根目录，`--add-dir` 授予额外可写目录，`codex completion <shell>` 生成补全脚本。
- 支持在 CLI 粘贴或通过 `-i/--image` 附加截图、示意图，帮助模型理解上下文。

### 4.3 Slash 命令与自定义提示
- 内置 `/model`、`/approvals`、`/review`、`/new`、`/init`、`/compact`、`/undo`、`/diff`、`/mention`、`/status`、`/mcp`、`/logout`、`/quit`/`/exit`、`/feedback` 等指令，可在聊天中即时调整模型、审批策略、查看 diff 或退出。
- 自定义提示放在 `$CODEX_HOME/prompts/*.md`，文件名即命令名，正文可包含 `$1…$9` 或命名占位符，支持 YAML frontmatter 描述。通过 `/prompts:<name>` 调用，可将常用操作（如 draft PR、发布流程）固化为脚本。

### 4.4 AGENTS.md 记忆
- Codex 会按照 `~/.codex` → 仓库根目录 → 当前工作目录的顺序向下查找 `AGENTS.override.md`/`AGENTS.md` 以及配置中声明的备用文件名，将内容从上到下拼接并截断到 `project_doc_max_bytes`（默认 32 KiB）。可借此为不同目录定义细化指令。

## 5. 安全与审批策略
- 审批策略决定何时提示用户，`read-only` 模式下默认 `AskForApproval::OnRequest`，可信目录可切换到 Auto 预设（`workspace-write` + `on-request`）。也可使用 `--ask-for-approval never` 实现完全自主，但风险更高。
- 常用组合：安全浏览 (`--sandbox read-only --ask-for-approval on-request`)、CI 审核 (`--sandbox read-only --ask-for-approval never`)、受控编辑 (`--sandbox workspace-write --ask-for-approval on-request`)、`--full-auto`、`--dangerously-bypass-approvals-and-sandbox` 等。
- `workspace-write` 默认禁网，可在 `config.toml` 的 `[sandbox_workspace_write]` 块中开启 `network_access = true`、添加额外可写目录或排除 `/tmp`，也可通过配置文件保存多个 profile 方便切换。
- 平台实现：macOS 用 Seatbelt、Linux 结合 Landlock + seccomp、Windows（实验性）借助受限 token 和代理封锁网络。容器环境若不支持内置沙箱，可在外层隔离后使用 `--sandbox danger-full-access`。
- 通过 `codex sandbox macos|linux [--full-auto]` 可实验命令在不同沙箱下的行为。

## 6. 非交互式自动化
- `codex exec` 适合在 CI/脚本中运行，默认只在 stderr 流式输出过程，在 stdout 打印最终回答，可用 `-o/--output-last-message` 输出到文件。
- 通过 `--full-auto` 开启可写沙箱，或 `--sandbox danger-full-access` 允许网络命令。JSON 模式 (`--json`) 会以 JSONL 流输出 turn、item、error 等事件以及 token 用量，方便与其他系统集成。
- `--output-schema` + JSON Schema 可以强制结构化返回；`-o` 只保存最终 JSON。可用 `--skip-git-repo-check` 绕过 Git 检查，`codex exec resume --last/<ID>` 继续非交互会话，保持上下文但需重新带入参数。

## 7. 配置与自定义
### 7.1 配置途径
- 优先级：CLI 专用 flag（如 `--model`）> 通用 `-c key=value` > `~/.codex/config.toml` > 内置默认。`--config` 支持嵌套键和值为任意 TOML 片段，可一次传入复杂表结构。
- `docs/example-config.md` 提供了一份涵盖全部键位的样板，可直接复制到 `config.toml` 再按需修改，包含模型、审批、沙箱、Shell 环境、历史、通知、实验特性、MCP、Profiles、OTel 等分类注释。

### 7.2 模型与特性
- 核心选项：`model`、`model_provider`、`model_reasoning_effort`、`model_reasoning_summary`、`model_verbosity`、`model_context_window`、`model_max_output_tokens`。可通过 `model_providers` 新增 OpenAI 以外的兼容 API（如 Azure、Ollama、Mistral），并设置 `env_key`、`http_headers`、`request_max_retries` 等参数。
- `[features]` 表用于集中启用实验能力，例如 `streamable_shell`、`web_search_request`、`apply_patch_freeform`、`ghost_commit` 等；旧版 `experimental_use_*` 键已弃用。

### 7.3 批准、沙箱与工具
- `approval_policy` 控制命令升级策略（`untrusted`/`on-failure`/`on-request`/`never`），`sandbox_mode` 控制沙箱层级（`read-only`/`workspace-write`/`danger-full-access`），`[sandbox_workspace_write]` 可声明可写路径、网络访问、TMPDIR 行为。
- `[tools]` 可显式开关 `web_search`、`view_image`；也可用 `approval_presets`、profiles 快速切换配置；`shell_environment_policy` 用 glob 语法控制向沙箱进程透传/剥离哪些环境变量，默认过滤包含 KEY/SECRET/TOKEN 的名称。

### 7.4 MCP 集成
- 在 `[mcp_servers]` 下配置 STDIO 或 streamable HTTP 服务器，可设定 `command`/`args`/`env`、`url`/`bearer_token_env_var`、启动和工具超时，以及白名单/黑名单。启用 `experimental_use_rmcp_client`（或 `[features].rmcp_client = true`）后可在 HTTP MCP 上使用 OAuth，并通过 `codex mcp login <server>` 登录。
- CLI 提供 `codex mcp add/list/get/remove/login` 等子命令快速管理服务器，支持输出 JSON。通知脚本可通过 `notify = ["python3", "/path/to/script"]` 接收 `agent-turn-complete` 事件。
- Codex 还可以自身作为 MCP server 运行（`codex mcp-server`），对外暴露 `codex` 和 `codex-reply` 两个工具，可供第三方代理框架调用；可使用 MCP Inspector 启动并观察事件。

### 7.5 UI、日志与历史
- `hide_agent_reasoning`/`show_raw_agent_reasoning` 控制是否展示模型推理；`[history].persistence = "none"` 可禁用历史记录；`file_opener` 支持 `vscode`/`windsurf`/`cursor` 等 URI 协议让终端引用可点击；`[tui].notifications` 利用支持的终端（iTerm2、Ghostty、WezTerm）发送桌面提醒。
- `notify` 选项可配置外部程序接收 JSON 事件，适合系统级通知或自动化；`chatgpt_base_url`、`forced_login_method`、`cli_auth_credentials_store` 则用于受控环境统一登录策略或改用系统秘钥链保存凭据。
- `project_doc_max_bytes`、`project_doc_fallback_filenames`、`history`、`profiles` 等键可精准控制文档加载、配置分层与日志保存；`file_opener`、`tui` 以及 `notify` 在 `docs/example-config.md` 中也有示范值。

## 8. 高阶功能
- `RUST_LOG` 控制 CLI/TUI 的日志等级；交互式模式默认写入 `~/.codex/log/codex-tui.log`，可用 `tail -F` 实时查看。`codex exec` 则默认 `RUST_LOG=error` 并直接输出到终端。
- `codex mcp` 子命令可以列出、添加、删除 MCP 服务器；`codex mcp-server` 则把 Codex 暴露为 MCP 工具，可通过 MCP Inspector 或多智能体框架调用，并支持 `approval-policy`、`sandbox`、`profile` 等参数。

## 9. 常见问题与解答
- “Codex” 2021 年发布的模型与本 CLI 无关；当前推荐使用 GPT-5 Codex 模型，可通过 `/model` 提升推理等级。
- 若想阻止 Codex 修改文件，可用 `--sandbox read-only` 或在会话内 `/approvals` 切换预设；要自动化可采用 `codex exec` 并结合 JSON/结构化输出。
- MCP 配置请参考本手册的配置章节；若登录遇到问题，确认 `~/.codex/auth.json`、端口转发以及安装流程是否符合指引。Windows 用户推荐 WSL2；Homebrew 老版本升级需先卸载 formula 再安装 cask。

## 10. 贡献与支持
- 在贡献新功能前请先提交 issue 获取确认，Bug/安全修复优先。开发流程包括：从 `main` 新建分支、保持变更聚焦、补齐测试与文档、运行 `cargo test && cargo clippy --tests && cargo fmt -- --config imports_granularity=Item`。PR 需填写模板、关联 issue，并在准备好后标记为 Ready for review。
- 社区遵循友好、包容、乐于分享的价值观，遇到问题可发 Discussion。安全漏洞或负责任 AI 相关问题请邮件 security@openai.com。
- 所有贡献者需通过评论 “I have read the CLA Document and I hereby sign the CLA” 签署 CLA，CLA-Bot 会自动记录；如需快速修复 DCO，可用 `git commit --amend -s --no-edit && git push -f`。CLA 具体条款参见 `docs/CLA.md`。
- 管理员可依据 `docs/release_management.md` 的流程发布新版本并同步到 npm/Homebrew。普通用户则只需关注 Releases 或使用包管理器升级即可。

## 11. 资源速查
- **示例配置**：`docs/example-config.md` – 提供覆盖所有键位的注释模板。
- **配置参考表**：`docs/config.md` 末尾的表格列举每个键的含义及默认值，便于按需查找。
- **平台沙箱细节**：`docs/platform-sandboxing.md` 已迁移到 `docs/sandbox.md#platform-sandboxing-details`，如需最新行为，请以本手册第 5 节为准。

---
本手册汇集了 `docs/` 目录内的所有官方说明，希望能帮助你在不同场景下高效、安全地驾驭 Codex CLI。如需更多帮助，欢迎查阅原文档或加入社区讨论。
