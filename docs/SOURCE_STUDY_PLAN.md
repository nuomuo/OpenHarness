# OpenHarness（`oh`）源码学习计划

> 基于 `README.zh-CN.md` 与实际代码结构整理（版本 v0.1.9，Python ≥ 3.10）
> 目标：系统掌握一个开源 Agent Harness 的完整实现——Agent Loop、Tools/Skills/Plugins、Memory、Permissions/Hooks、Multi-Agent、Provider 工作流、TUI 与 `ohmo` personal agent。

---

## 一、项目全景

### 1.1 技术栈

| 层 | 技术 |
|----|------|
| CLI | Typer + Rich + questionary |
| TUI | 双前端：Textual（Python）+ React/Ink（`frontend/terminal`，Node） |
| 模型接入 | anthropic / openai SDK，统一 `api` 抽象层 |
| 协议 | MCP（Model Context Protocol）、WebSocket |
| IM 渠道 | Telegram / Slack / Discord / Feishu / Matrix / QQ / Email 等 |
| 包管理 | uv + hatchling |

### 1.2 仓库布局

```
OpenHarness/
├── src/openharness/     # 核心库（~30 个子模块）
├── ohmo/                # personal-agent app（独立 CLI）
├── frontend/terminal/   # React/Ink TUI（TypeScript）
├── autopilot-dashboard/ # autopilot 仪表盘
├── scripts/             # 安装/测试脚本
├── tests/               # 测试
└── docs/                # 文档
```

### 1.3 核心调用链（先记住这条主线）

```
cli.py (Typer 入口, oh/ohmo 命令)
  └─ main() 参数解析 → dry-run 预览 / 交互 / -p 打印模式
      └─ ui/runtime.py 或 ui/backend_host.py (ReactBackendHost)
          └─ engine/query_engine.py (QueryEngine: 持有会话历史)
              └─ engine/query.py :: run_query()   ← Agent Loop 核心
                  ├─ api/*_client.stream_message()  (模型流式调用)
                  ├─ tools/* (43+ 工具注册表)
                  ├─ permissions/checker.py (权限校验)
                  ├─ hooks/executor.py (Hook 执行)
                  └─ services/compact/ (auto-compact 上下文压缩)
```

---

## 二、模块地图（按重要度排序）

| 模块 | 职责 | 关键文件（大小） | 优先级 |
|------|------|------------------|--------|
| `engine/` | Agent Loop、流式事件、成本跟踪 | `query.py`(39K)、`query_engine.py`(13K)、`messages.py` | ★★★★★ |
| `api/` | Provider 抽象：anthropic/openai/codex/copilot 客户端 + registry | `registry.py`(14K)、`openai_client.py`(19K)、`codex_client.py`(17K) | ★★★★★ |
| `tools/` | 43+ 内置工具（bash、文件、grep、任务、cron、worktree…） | `base.py`（ToolRegistry 抽象）、`__init__.py`（注册入口） | ★★★★★ |
| `permissions/` | 权限模式、路径规则、命令拦截 | `checker.py`(7K) | ★★★★ |
| `hooks/` | Hook 事件、加载、执行、热重载 | `executor.py`(9K) | ★★★★ |
| `skills/` | Markdown skills 按需加载（兼容 anthropics/skills） | `loader.py`(8K)、`bundled/content/*.md` | ★★★★ |
| `plugins/` | Claude 风格插件加载/安装 | `loader.py`(27K) | ★★★ |
| `memory/` | MEMORY.md 持久记忆、相关性、搜索 | `manager.py`、`relevance.py`、`schema.py`(15K) | ★★★★ |
| `services/` | compact（65K 大模块）、cron、session 存储、LSP、autodream | `compact/__init__.py`、`session_storage.py` | ★★★★ |
| `swarm/` | multi-agent：registry、mailbox、worktree、team 生命周期 | `team_lifecycle.py`(29K)、`in_process.py`(25K)、`permission_sync.py`(38K) | ★★★ |
| `mcp/` | MCP client 配置与工具桥接 | `client.py`(11K) | ★★★ |
| `config/` | settings(43K)、schema、paths | `settings.py` | ★★★ |
| `auth/` | 凭据管理、外部登录（claude/codex/copilot） | `manager.py`(20K)、`external.py`(21K) | ★★★ |
| `ui/` | Textual App + React backend host + runtime(32K) | `backend_host.py`(39K)、`runtime.py`(32K) | ★★★ |
| `channels/` | IM 渠道适配（12 种） | `impl/feishu.py`(53K)、`impl/telegram.py`(21K) | ★★ |
| `ohmo/` | personal agent + gateway（常驻 IM 服务） | `gateway/runtime.py`(51K)、`cli.py` | ★★ |
| `autopilot/` | 自动巡航服务（92K，独立子系统） | `service.py` | ★ |
| `coordinator/` | coordinator 模式、agent 定义 | `agent_definitions.py`(45K) | ★ |
| 其余 | sandbox、bridge、prompts、state、tasks、themes、vim、voice 等 | 较小，穿插阅读 | ★ |

---

## 三、学习计划（6 个阶段）

### 阶段 1：跑起来 + 主线打通（第 1 周）

**目标**：理解一条用户消息从 CLI 到模型再回来的完整路径。

1. 环境与入口
   - `git clone && uv sync --extra dev && uv run oh`
   - 读 [pyproject.toml](file:///e:/工作文档/OpenHarness/pyproject.toml)：入口 `openharness.cli:app`（`oh`/`openh`/`openharness`）与 `ohmo.cli:app`
2. CLI 骨架：[cli.py](file:///e:/工作文档/OpenHarness/src/openharness/cli.py)（94K，重点读函数清单）
   - `main()`（L2181 起）：`-p/--print`、`--output-format`、`--dry-run`、`--resume` 等参数
   - `_build_dry_run_preview()`（L396）：dry-run 如何解析配置/skills/commands/MCP 并给出 `ready/warning/blocked`
   - `setup_cmd()`（L1752）→ `provider use/add` → `auth status`：workflow+profile 模型
3. Agent Loop 核心：[engine/query.py](file:///e:/工作文档/OpenHarness/src/openharness/engine/query.py)
   - `run_query()`（L633）：**逐行精读**——auto-compact 检查 → 图片预处理 → `stream_message` 流式循环 → 工具调用分发
   - `QueryContext`（L138）：循环的全部依赖注入
   - `_execute_tool_call()`（L887）：工具执行 + 权限 + 结果回填
   - `_remember_*` 系列函数：工具元数据如何回流到上下文
4. 封装层：[engine/query_engine.py](file:///e:/工作文档/OpenHarness/src/openharness/engine/query_engine.py)
   - `QueryEngine` 如何组合 api_client / tool_registry / permission_checker / hooks
5. 事件模型：`engine/stream_events.py`、`engine/messages.py`（`ConversationMessage` / block 类型）

**练手**：`uv run oh -p "hello" --output-format stream-json`，对照 `run_query` 的 yield 事件逐个理解。

### 阶段 2：API 抽象与 Provider 工作流（第 2 周前半）

**目标**：理解"workflow + profile"如何统一 5 种后端。

1. [api/provider.py](file:///e:/工作文档/OpenHarness/src/openharness/api/provider.py)：`SupportsStreamingMessages` 协议与 `ApiMessageRequest` 等数据结构
2. [api/registry.py](file:///e:/工作文档/OpenHarness/src/openharness/api/registry.py)：preset profile 注册、default model 解析
3. 四个客户端对比读：`client.py`（Anthropic 风格基线）→ `openai_client.py` → `codex_client.py`（复用 `~/.codex/auth.json`）→ `copilot_client.py`（OAuth 流程）
4. 认证体系：[auth/manager.py](file:///e:/工作文档/OpenHarness/src/openharness/auth/manager.py) + `auth/external.py`（Claude 订阅复用 `~/.claude/.credentials.json`）
5. 配置：[config/settings.py](file:///e:/工作文档/OpenHarness/src/openharness/config/settings.py)（43K，可先只读 schema 与加载路径部分）+ `config/paths.py`

**练手**：手动执行 `oh setup` 走一遍流程，然后 `oh provider add` 添加一个自定义 endpoint，观察配置文件变化（`~/.openharness/` 下）。

### 阶段 3：工具 / Skills / Plugins / MCP（第 2 周后半）

**目标**：掌握可扩展性的三大支柱。

1. 工具体系
   - [tools/base.py](file:///e:/工作文档/OpenHarness/src/openharness/tools/base.py)：`BaseTool`、`ToolExecutionContext`、`ToolRegistry`、`ToolResult`（全文精读，文件小）
   - [tools/__init__.py](file:///e:/工作文档/OpenHarness/src/openharness/tools/__init__.py)：`create_default_tool_registry()` 注册顺序
   - 精读 3 个代表工具：`bash_tool.py`（涉及权限/沙箱）、`file_edit_tool.py`（字符串替换语义）、`agent_tool.py`（spawn subagent）
   - 浏览其余工具按名字理解用途即可（task_*、cron_*、worktree、plan mode、todo）
2. Skills：[skills/loader.py](file:///e:/工作文档/OpenHarness/src/openharness/skills/loader.py) + `skills/_frontmatter.py` + `bundled/content/*.md`（7 个内置 skill 是最好的样例）
3. Plugins：[plugins/loader.py](file:///e:/工作文档/OpenHarness/src/openharness/plugins/loader.py)（27K）——Claude 风格 plugin 的发现/加载/合并
4. MCP：[mcp/client.py](file:///e:/工作文档/OpenHarness/src/openharness/mcp/client.py) + `mcp/config.py` + `tools/mcp_tool.py`（MCP server → ToolRegistry 桥接）+ `mcp_auth_tool.py`

**练手**：写一个最小自定义工具（如 `time_tool.py`），注册进 registry 并在 `-p` 模式验证；再用 markdown 写一个 skill 放入 skills 目录验证按需加载。

### 阶段 4：治理——Permissions / Hooks / Sandbox（第 3 周前半）

1. [permissions/checker.py](file:///e:/工作文档/OpenHarness/src/openharness/permissions/checker.py)：permission mode（default/acceptEdits/plan/bypassPermissions…）、path allow/deny rules、denied commands
2. [hooks/](file:///e:/工作文档/OpenHarness/src/openharness/hooks)：`events.py` → `schemas.py` → `loader.py` → `executor.py`（执行时机与 `run_query` 中的调用点对应）→ `hot_reload.py`
3. Sandbox：`sandbox/adapter.py`、`docker_backend.py`、`path_validator.py`——工具执行的容器隔离
4. utils：`network_guard.py`（网络防护）、`file_lock.py`、`shell.py`

**练手**：配置一条 hook（如 PreToolUse 拦截 bash），配置一条 path deny rule，验证行为。

### 阶段 5：记忆 / 会话 / 上下文管理（第 3 周后半）

1. Memory：[memory/manager.py](file:///e:/工作文档/OpenHarness/src/openharness/memory/manager.py)、`relevance.py`（相关性打分）、`search.py`、`schema.py`
2. 会话持久化：[services/session_storage.py](file:///e:/工作文档/OpenHarness/src/openharness/services/session_storage.py) + `session_backend.py` + `session_memory/`
3. **Auto-compact（大重点）**：[services/compact/__init__.py](file:///e:/工作文档/OpenHarness/src/openharness/services/compact/__init__.py)（65K）——microcompact（清旧工具结果）与 LLM 全量摘要的两级策略，与 `run_query` 开头的调用呼应
4. System prompt 组装：[prompts/](file:///e:/工作文档/OpenHarness/src/openharness/prompts)——`system_prompt.py`、`claudemd.py`（CLAUDE.md 发现）、`context.py`、`environment.py`
5. 记忆提取：`services/memory_extract/`、`personalization/`

**练手**：构造长对话触发 auto-compact，观察事件流；检查 `~/.openharness/` 下 session 与 MEMORY.md 的落盘格式。

### 阶段 6：UI / Multi-Agent / ohmo / Channels（第 4 周）

1. TUI 双栈
   - Textual 版：[ui/textual_app.py](file:///e:/工作文档/OpenHarness/src/openharness/ui/textual_app.py)
   - React/Ink 版：[ui/backend_host.py](file:///e:/工作文档/OpenHarness/src/openharness/ui/backend_host.py)（`ReactBackendHost`，L73）+ [ui/runtime.py](file:///e:/工作文档/OpenHarness/src/openharness/ui/runtime.py) + `ui/protocol.py`（前后端协议）
   - 前端：[frontend/terminal/src/App.tsx](file:///e:/工作文档/OpenHarness/frontend/terminal/src/App.tsx) 及 components/（CommandPicker、ToolCallDisplay、SwarmPanel 等）
   - `oh --dry-run` 理解 UI 启动前置检查
2. Multi-Agent（swarm）
   - `swarm/types.py` → `registry.py`（team registry）→ `mailbox.py`（agent 间消息）→ `in_process.py` / `subprocess_backend.py`（两种执行后端）→ `worktree.py`（git worktree 隔离）→ `team_lifecycle.py`
   - 入口工具：`tools/agent_tool.py`、`team_create_tool.py`、`send_message_tool.py`
   - `coordinator/`：coordinator 模式与 agent 定义（45K，选读）
3. `ohmo` personal agent
   - [ohmo/cli.py](file:///e:/工作文档/OpenHarness/ohmo/cli.py)（init/config 命令）→ `workspace.py`（`~/.ohmo` 结构：soul.md/identity.md/BOOTSTRAP.md）→ [ohmo/gateway/runtime.py](file:///e:/工作文档/OpenHarness/ohmo/gateway/runtime.py)（51K 常驻服务）
   - [ohmo/gateway/bridge.py](file:///e:/工作文档/OpenHarness/ohmo/gateway/bridge.py)：channels 与 harness 的桥接
4. Channels（选读，按兴趣挑 1-2 个）：`channels/bus/`（事件总线）→ `impl/base.py` → `impl/telegram.py`（结构清晰适合入门）→ feishu.py（最复杂）
5. 周边选读：`autopilot/`（92K 自动巡航）、`services/cron*`、`tasks/`、`bridge/`、`voice/`

**练手**：`ohmo init && ohmo config` 走完 onboarding；用 agent_tool spawn 一个 subagent 完成小任务，观察 swarm registry 状态。

---

## 四、每阶段通用学习方法

1. **先跑后读**：每阶段先用对应 CLI 命令产生真实行为，再带着现象读代码。
2. **主线优先**：大文件（cli.py 94K、autopilot/service.py 92K、compact 65K）不要通读，用函数清单定位入口再深入。
3. **测试即文档**：`tests/` 目录与 `scripts/test_harness_features.py`、`scripts/test_real_skills_plugins.py` 是行为规格说明。
4. **事件流调试**：`--output-format stream-json` + `--dry-run` 是理解内部状态的两个窗口。
5. **改动验证**：每学完一个模块，做一个小改动（加工具/加 hook/加 skill）验证理解。

## 五、里程碑自检清单

- [ ] 能画出一条消息从 `oh -p "..."` 到返回的完整时序
- [ ] 能解释 workflow / profile / auth-source / base-url 四者的关系
- [ ] 能独立新增一个内置工具和 markdown skill
- [ ] 能说清 permission mode 与 hook 的执行时机（对应 `run_query` 中的代码位置）
- [ ] 能解释 auto-compact 两级策略的触发条件
- [ ] 能描述 React TUI 前后端协议与事件流
- [ ] 能解释 swarm 中 in-process 与 subprocess 两种 subagent 后端的差异
- [ ] 能说清 `ohmo` gateway 如何把 IM 消息路由到 harness

## 六、风险与提示

- `commands/registry.py`（124K）、`autopilot/service.py`（92K）、`ohmo/gateway/runtime.py`（51K）是三个超大文件，建议最后阶段按需查阅，不影响主线理解。
- README 提到的部分能力（如 43+ tools）以 `tools/__init__.py` 的注册表为准逐一核对。
- 中文文档与代码可能有版本差（当前 v0.1.9），以 `CHANGELOG.md` 和代码为准。
