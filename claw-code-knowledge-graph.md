# claw-code 业务逻辑知识图谱

> claw-code 是 Claude Code CLI 的 Rust 开源重写。本文档以 Mermaid 图谱形式展示其完整业务逻辑架构。

---

## 1. Crate 依赖关系图

9 个 Rust crate 构成整个工作区，`rusty-claude-cli` 是二进制入口，依赖其余所有库 crate。

```mermaid
graph TD
    CLI["rusty-claude-cli<br/><i>二进制入口 / REPL</i>"]
    RT["runtime<br/><i>会话运行时 / 43 模块</i>"]
    API["api<br/><i>多提供商 HTTP 客户端</i>"]
    TOOLS["tools<br/><i>40+ 工具注册与调度</i>"]
    CMD["commands<br/><i>斜杠命令定义</i>"]
    PLG["plugins<br/><i>Hook / 外部工具</i>"]
    TEL["telemetry<br/><i>会话追踪与分析</i>"]
    COMPAT["compat-harness<br/><i>上游 TS 清单提取</i>"]
    MOCK["mock-anthropic-service<br/><i>测试用 Mock 服务</i>"]

    CLI -->|依赖| RT
    CLI -->|依赖| API
    CLI -->|依赖| TOOLS
    CLI -->|依赖| CMD
    CLI -->|依赖| TEL
    RT -->|依赖| API
    RT -->|依赖| TOOLS
    RT -->|依赖| PLG
    RT -->|依赖| CMD
    TOOLS -->|依赖| RT
    TOOLS -->|依赖| PLG
    CLI -.->|测试依赖| MOCK
    CLI -.->|构建依赖| COMPAT

    style CLI fill:#4A90D9,color:#fff
    style RT fill:#D94A4A,color:#fff
    style API fill:#D9A04A,color:#fff
    style TOOLS fill:#4AD97A,color:#fff
```

---

## 2. 核心业务流程图

用户输入到最终响应的完整生命周期。这是 claw-code 最核心的业务循环。

```mermaid
flowchart TD
    A["用户输入<br/><i>CLI 参数 / REPL 交互</i>"] --> B["解析 CliAction<br/><i>main.rs</i>"]
    B --> C{"命令类型"}
    C -->|"斜杠命令"| D["CommandHandler<br/><i>commands crate</i>"]
    C -->|"对话提示"| E["初始化运行时"]

    E --> E1["ConfigLoader::discover<br/><i>合并三层配置</i>"]
    E --> E2["发现 MCP 服务器"]
    E --> E3["加载插件"]
    E --> E4["构建 GlobalToolRegistry"]

    E1 & E2 & E3 & E4 --> F["SystemPromptBuilder::build<br/><i>组装系统提示词</i>"]

    F --> F1["注入指令文件<br/><i>CLAUDE.md 发现</i>"]
    F --> F2["注入项目上下文<br/><i>git status / date</i>"]
    F --> F3["注入工具定义<br/><i>ToolDefinition 列表</i>"]

    F1 & F2 & F3 --> G["加载/创建 Session"]
    G --> H["ConversationRuntime::run_turn"]

    H --> I["调用 ProviderClient<br/><i>stream_message</i>"]
    I --> J["SSE 流式解析<br/><i>SseParser</i>"]
    J --> K{"事件类型"}

    K -->|"TextDelta"| L["TerminalRenderer<br/><i>实时渲染 Markdown</i>"]
    K -->|"ToolUse"| M["工具执行流程"]
    K -->|"MessageStop"| N["收集 TurnSummary"]

    M --> M1["PreToolUse Hook"]
    M1 --> M2{"权限检查<br/><i>PermissionEnforcer</i>"}
    M2 -->|"允许"| M3["execute_tool<br/><i>GlobalToolRegistry 调度</i>"]
    M2 -->|"拒绝"| M4["返回拒绝消息"]
    M3 --> M5["PostToolUse Hook"]
    M5 --> M6["ToolResult → 追加到 Session"]
    M6 --> I

    N --> O{"输入 token 超阈值?<br/><i>默认 100k</i>"}
    O -->|"是"| P["自动压缩<br/><i>compact.rs</i>"]
    O -->|"否"| Q["Session 持久化<br/><i>JSONL 文件</i>"]
    P --> Q
    Q --> R["返回响应给用户"]

    style A fill:#4A90D9,color:#fff
    style H fill:#D94A4A,color:#fff
    style M3 fill:#4AD97A,color:#fff
    style I fill:#D9A04A,color:#fff
```

---

## 3. 工具系统架构图

40+ 内置工具通过 `GlobalToolRegistry` 统一管理，支持内置、插件、MCP、运行时四类工具来源。

```mermaid
flowchart TD
    REG["GlobalToolRegistry<br/><i>tools/src/lib.rs</i>"]

    subgraph 工具来源
        BUILTIN["内置工具<br/><i>mvp_tool_specs() → 40 个</i>"]
        PLUGIN["插件工具<br/><i>with_plugin_tools()</i>"]
        MCP_T["MCP 工具<br/><i>McpToolRegistry</i>"]
        RUNTIME_T["运行时工具<br/><i>with_runtime_tools()</i>"]
    end

    BUILTIN --> REG
    PLUGIN --> REG
    MCP_T --> REG
    RUNTIME_T --> REG

    REG --> DISPATCH["execute_tool(name, input)"]
    DISPATCH --> ENFORCE["PermissionEnforcer::check"]

    subgraph 文件与搜索工具
        T_READ["read_file<br/><i>文件读取</i>"]
        T_WRITE["write_file<br/><i>文件写入</i>"]
        T_EDIT["edit_file<br/><i>精确替换</i>"]
        T_GLOB["glob_search<br/><i>文件名匹配</i>"]
        T_GREP["grep_search<br/><i>内容搜索</i>"]
    end

    subgraph Shell 工具
        T_BASH["bash<br/><i>命令执行 / 沙箱</i>"]
        T_PS["PowerShell<br/><i>Windows</i>"]
    end

    subgraph 网络工具
        T_FETCH["WebFetch<br/><i>URL 内容获取</i>"]
        T_SEARCH["WebSearch<br/><i>搜索引擎</i>"]
    end

    subgraph 协调工具
        T_AGENT["Agent<br/><i>子代理创建</i>"]
        T_SKILL["Skill<br/><i>技能调用</i>"]
        T_ASK["AskUserQuestion<br/><i>交互确认</i>"]
    end

    subgraph 任务管理工具
        T_TC["TaskCreate"]
        T_TG["TaskGet"]
        T_TL["TaskList"]
        T_TU["TaskUpdate"]
        T_TS["TaskStop"]
        T_TO["TaskOutput"]
    end

    subgraph Worker 工具
        T_WC["WorkerCreate"]
        T_WS["WorkerSendPrompt"]
        T_WO["WorkerObserve"]
        T_WT["WorkerTerminate"]
    end

    subgraph MCP/LSP 工具
        T_LSP["LSP<br/><i>语言服务器协议</i>"]
        T_MCPR["ListMcpResources"]
        T_MCPREAD["ReadMcpResource"]
        T_MCPAUTH["McpAuth"]
    end

    subgraph 调度工具
        T_CRON_C["CronCreate"]
        T_CRON_D["CronDelete"]
        T_CRON_L["CronList"]
        T_REMOTE["RemoteTrigger"]
    end

    subgraph 其他工具
        T_TODO["TodoWrite"]
        T_NB["NotebookEdit"]
        T_PLAN_E["EnterPlanMode"]
        T_PLAN_X["ExitPlanMode"]
        T_CONFIG["Config"]
    end

    ENFORCE -->|"允许"| 文件与搜索工具
    ENFORCE -->|"允许"| Shell 工具
    ENFORCE -->|"允许"| 网络工具
    ENFORCE -->|"允许"| 协调工具
    ENFORCE -->|"允许"| 任务管理工具
    ENFORCE -->|"允许"| Worker 工具
    ENFORCE -->|"允许"| MCP/LSP 工具
    ENFORCE -->|"允许"| 调度工具
    ENFORCE -->|"允许"| 其他工具

    style REG fill:#4AD97A,color:#fff
    style ENFORCE fill:#D94A4A,color:#fff
```

---

## 4. 配置与权限系统图

三层配置文件合并 + 三级权限模型 + Hook 生命周期管理。

```mermaid
flowchart TD
    subgraph 配置发现与合并
        UC["用户级配置<br/><i>~/.claw.json<br/>~/.config/claw/settings.json</i>"]
        PC["项目级配置<br/><i>.claw.json<br/>.claw/settings.json</i>"]
        LC["本地级配置<br/><i>.claw/settings.local.json</i>"]

        UC -->|"低优先级"| MERGE["ConfigLoader::load<br/><i>深度合并</i>"]
        PC -->|"中优先级"| MERGE
        LC -->|"高优先级"| MERGE
    end

    MERGE --> RFC["RuntimeFeatureConfig"]

    RFC --> HOOKS_CFG["HookConfig<br/><i>pre/post tool-use</i>"]
    RFC --> PERM_CFG["PermissionRuleConfig<br/><i>allow / deny / ask</i>"]
    RFC --> MCP_CFG["McpConfigCollection<br/><i>stdio/SSE/HTTP/WS/SDK</i>"]
    RFC --> PLG_CFG["PluginConfig"]
    RFC --> OAUTH_CFG["OAuthConfig"]
    RFC --> SANDBOX_CFG["SandboxConfig"]

    subgraph 权限系统
        PM["PermissionMode"]
        PM_RO["ReadOnly<br/><i>只读操作</i>"]
        PM_WS["WorkspaceWrite<br/><i>工作区写入</i>"]
        PM_DA["DangerFullAccess<br/><i>完全访问</i>"]
        PM_PR["Prompt<br/><i>逐次确认</i>"]
        PM_AL["Allow<br/><i>自动允许</i>"]

        PM --- PM_RO
        PM --- PM_WS
        PM --- PM_DA
        PM --- PM_PR
        PM --- PM_AL
    end

    PERM_CFG --> PE["PermissionEnforcer"]
    PM --> PE

    PE --> PE_BASH["check_bash<br/><i>检测破坏性命令</i>"]
    PE --> PE_FILE["check_file_write<br/><i>工作区边界检查</i>"]
    PE --> PE_RESULT{"EnforcementResult"}
    PE_RESULT -->|"Allowed"| EXEC["执行工具"]
    PE_RESULT -->|"Denied"| DENY["拒绝 + 原因"]

    subgraph Hook 生命周期
        H_PRE["PreToolUse<br/><i>工具执行前</i>"]
        H_POST["PostToolUse<br/><i>执行成功后</i>"]
        H_FAIL["PostToolUseFailure<br/><i>执行失败后</i>"]

        H_PRE -->|"JSON stdin"| HOOK_RUN["HookRunner<br/><i>子进程 / 10s 超时</i>"]
        HOOK_RUN -->|"JSON stdout"| H_RESULT["HookRunResult"]
        H_RESULT --> H_DENY["denied: 拒绝执行"]
        H_RESULT --> H_OVERRIDE["permission_override: 覆盖权限"]
        H_RESULT --> H_UPDATE["updated_input: 修改输入"]
    end

    HOOKS_CFG --> H_PRE
    HOOKS_CFG --> H_POST
    HOOKS_CFG --> H_FAIL

    style PE fill:#D94A4A,color:#fff
    style MERGE fill:#4A90D9,color:#fff
```

---

## 5. 多模型提供商架构图

`ProviderClient` 枚举统一抽象多家大模型 API，支持 Anthropic、OpenAI 兼容、xAI、DashScope。

```mermaid
flowchart TD
    MODEL["模型名称<br/><i>--model 参数</i>"]
    ENV["环境变量<br/><i>API_KEY / BASE_URL</i>"]

    MODEL --> DETECT["detect_provider_kind<br/><i>providers/mod.rs</i>"]
    ENV --> DETECT

    DETECT --> ROUTE{"路由规则"}

    ROUTE -->|"claude*"| ANTH["AnthropicClient<br/><i>Messages API</i>"]
    ROUTE -->|"gpt-* / openai/*"| OAI["OpenAiCompatClient<br/><i>Chat Completions</i>"]
    ROUTE -->|"grok*"| XAI["OpenAiCompatClient<br/><i>xAI 端点</i>"]
    ROUTE -->|"qwen* / kimi*"| DASH["OpenAiCompatClient<br/><i>DashScope 端点</i>"]
    ROUTE -->|"OPENAI_BASE_URL"| CUSTOM["OpenAiCompatClient<br/><i>自定义端点</i>"]

    subgraph ProviderClient 枚举
        PC_ENUM["ProviderClient"]
        PC_ENUM --- ANTH
        PC_ENUM --- OAI
        PC_ENUM --- XAI
    end

    subgraph Provider Trait
        TRAIT["trait Provider"]
        TRAIT_SEND["send_message → MessageResponse"]
        TRAIT_STREAM["stream_message → Stream"]
        TRAIT --- TRAIT_SEND
        TRAIT --- TRAIT_STREAM
    end

    ANTH & OAI & XAI & DASH --> TRAIT

    subgraph 统一消息格式
        REQ["MessageRequest<br/><i>model, messages, tools,<br/>temperature, max_tokens</i>"]
        RESP["MessageResponse<br/><i>content, usage, stop_reason</i>"]
        STREAM_EVT["StreamEvent<br/><i>TextDelta, ToolUse, Usage</i>"]
    end

    TRAIT_SEND --> RESP
    TRAIT_STREAM --> SSE["SseParser<br/><i>sse.rs</i>"]
    SSE --> STREAM_EVT

    subgraph 模型特殊适配
        COMPAT_KIMI["Kimi: 排除 is_error 字段"]
        COMPAT_REASON["推理模型: 剥离调优参数<br/><i>o1/o3/o4/qwen-qwq/grok-3-mini</i>"]
        COMPAT_GPT5["GPT-5: max_completion_tokens"]
    end

    OAI --> COMPAT_KIMI
    OAI --> COMPAT_REASON
    OAI --> COMPAT_GPT5

    subgraph 端点配置
        EP_ANTH["api.anthropic.com"]
        EP_OAI["api.openai.com/v1"]
        EP_XAI["api.x.ai/v1"]
        EP_DASH["dashscope.aliyuncs.com<br/>/compatible-mode/v1"]
        EP_CUSTOM["OPENAI_BASE_URL<br/><i>Ollama / OpenRouter / vLLM</i>"]
    end

    ANTH -.-> EP_ANTH
    OAI -.-> EP_OAI
    XAI -.-> EP_XAI
    DASH -.-> EP_DASH
    CUSTOM -.-> EP_CUSTOM

    subgraph 错误处理与重试
        ERR["ApiError"]
        ERR_CTX["ContextWindowExceeded"]
        ERR_AUTH["ExpiredOAuthToken / Auth"]
        ERR_RETRY["RetriesExhausted<br/><i>指数退避</i>"]
        ERR_BODY["RequestBodySizeExceeded"]
        ERR --- ERR_CTX
        ERR --- ERR_AUTH
        ERR --- ERR_RETRY
        ERR --- ERR_BODY
    end

    style DETECT fill:#4A90D9,color:#fff
    style PC_ENUM fill:#D9A04A,color:#fff
    style SSE fill:#4AD97A,color:#fff
```

---

## 6. 会话与消息模型图

Session 管理完整对话历史，支持 JSONL 持久化、日志轮转、自动压缩。

```mermaid
flowchart TD
    subgraph Session 结构
        S["Session<br/><i>session.rs</i>"]
        S_MSG["messages: Vec&lt;ConversationMessage&gt;"]
        S_COMP["compaction_metadata"]
        S_FORK["fork_provenance<br/><i>分叉历史</i>"]
        S_WS["workspace_root"]
        S_USAGE["token_usage"]

        S --- S_MSG
        S --- S_COMP
        S --- S_FORK
        S --- S_WS
        S --- S_USAGE
    end

    subgraph ConversationMessage
        CM["ConversationMessage"]
        CM_ROLE["role: MessageRole"]
        CM_BLOCKS["blocks: Vec&lt;ContentBlock&gt;"]
        CM_USAGE["usage: TokenUsage"]

        CM --- CM_ROLE
        CM --- CM_BLOCKS
        CM --- CM_USAGE
    end

    subgraph MessageRole
        ROLE_S["System"]
        ROLE_U["User"]
        ROLE_A["Assistant"]
        ROLE_T["Tool"]
    end

    CM_ROLE --- ROLE_S & ROLE_U & ROLE_A & ROLE_T

    subgraph ContentBlock
        CB["ContentBlock 枚举"]
        CB_TEXT["Text { text }"]
        CB_TU["ToolUse { id, name, input }"]
        CB_TR["ToolResult { tool_use_id,<br/>tool_name, output, is_error }"]

        CB --- CB_TEXT
        CB --- CB_TU
        CB --- CB_TR
    end

    CM_BLOCKS --> CB

    subgraph 持久化
        SS["SessionStore"]
        SS_JSONL["JSONL 格式<br/><i>一条消息一行</i>"]
        SS_ROT["日志轮转<br/><i>256KB 触发, 最多 3 份</i>"]
        SS_PATH["~/.local/share/<br/>opencode/sessions/"]

        SS --> SS_JSONL
        SS --> SS_ROT
        SS --> SS_PATH
    end

    S_MSG --> SS

    subgraph 会话恢复
        RESUME["--resume 参数"]
        RESUME_ID["按 session-id"]
        RESUME_LATEST["latest"]
        RESUME_PATH["按路径"]

        RESUME --- RESUME_ID
        RESUME --- RESUME_LATEST
        RESUME --- RESUME_PATH
    end

    RESUME --> SS

    subgraph 自动压缩
        COMPACT["CompactionConfig<br/><i>compact.rs</i>"]
        COMPACT_THR["阈值: 100k input tokens<br/><i>CLAUDE_CODE_AUTO_COMPACT_<br/>INPUT_TOKENS</i>"]
        COMPACT_KEEP["保留最近 4 条消息"]
        COMPACT_SAFE["工具调用对安全<br/><i>不拆散 ToolUse/ToolResult</i>"]
        COMPACT_SUM["旧消息 → 摘要文本"]

        COMPACT --- COMPACT_THR
        COMPACT --- COMPACT_KEEP
        COMPACT --- COMPACT_SAFE
        COMPACT --- COMPACT_SUM
    end

    S_MSG -->|"token 超阈值"| COMPACT
    COMPACT -->|"压缩后"| S_MSG

    subgraph TurnSummary
        TS["TurnSummary"]
        TS_MSG["assistant_messages"]
        TS_TOOL["tool_results"]
        TS_USAGE["usage / iterations"]
        TS_CACHE["PromptCacheEvent"]
        TS_COMP_E["AutoCompactionEvent"]

        TS --- TS_MSG
        TS --- TS_TOOL
        TS --- TS_USAGE
        TS --- TS_CACHE
        TS --- TS_COMP_E
    end

    style S fill:#4A90D9,color:#fff
    style CM fill:#D94A4A,color:#fff
    style CB fill:#D9A04A,color:#fff
    style COMPACT fill:#4AD97A,color:#fff
```

---

## 7. 高级子系统关系图

Worker 编排、Lane 事件、分支健康检测、故障恢复、策略引擎共同构成 claw-code 的自主编排能力。

```mermaid
flowchart TD
    subgraph Worker 编排系统
        WR["WorkerRegistry<br/><i>worker_boot.rs</i>"]
        WS_STATE["WorkerStatus 状态机"]
        WS_SPAWN["Spawning"]
        WS_TRUST["TrustRequired"]
        WS_READY["ReadyForPrompt"]
        WS_RUN["Running"]
        WS_DONE["Finished / Failed"]

        WS_STATE --> WS_SPAWN --> WS_TRUST --> WS_READY --> WS_RUN --> WS_DONE

        SEB["StartupEvidenceBundle<br/><i>超时诊断证据</i>"]
        SFC["StartupFailureClassification"]
        SFC_TRUST["TrustRequired"]
        SFC_PROMPT["PromptMisdelivery"]
        SFC_TIMEOUT["PromptAcceptanceTimeout"]
        SFC_DEAD["TransportDead"]
        SFC_CRASH["WorkerCrashed"]

        SFC --- SFC_TRUST & SFC_PROMPT & SFC_TIMEOUT & SFC_DEAD & SFC_CRASH
        WS_DONE -->|"失败时收集"| SEB
        SEB --> SFC
    end

    subgraph Lane 事件系统
        LE["LaneEventSchema<br/><i>lane_events.rs</i>"]
        LE_START["lane.started"]
        LE_READY["lane.ready"]
        LE_BLOCKED["lane.blocked"]
        LE_RED["lane.red"]
        LE_GREEN["lane.green"]
        LE_COMMIT["lane.commit.created"]
        LE_PR["lane.pr.opened"]
        LE_MERGE["lane.merge.ready"]

        LE --- LE_START & LE_READY & LE_BLOCKED
        LE --- LE_RED & LE_GREEN
        LE --- LE_COMMIT & LE_PR & LE_MERGE
    end

    subgraph Lane 状态
        LS["LaneEventStatus"]
        LS_RUN["Running"]
        LS_READY2["Ready"]
        LS_BLOCKED2["Blocked"]
        LS_RED2["Red"]
        LS_GREEN2["Green"]
        LS_COMP["Completed"]
        LS_FAIL["Failed"]
        LS_MERGED["Merged"]

        LS --- LS_RUN & LS_READY2 & LS_BLOCKED2
        LS --- LS_RED2 & LS_GREEN2
        LS --- LS_COMP & LS_FAIL & LS_MERGED
    end

    subgraph 故障分类
        LFC["LaneFailureClass"]
        LFC_PROMPT2["PromptDelivery"]
        LFC_TRUST2["TrustGate"]
        LFC_BRANCH["BranchDivergence"]
        LFC_COMPILE["Compile"]
        LFC_TEST["Test"]
        LFC_PLUGIN["PluginStartup"]
        LFC_MCP["McpStartup / Handshake"]
        LFC_INFRA["Infra"]

        LFC --- LFC_PROMPT2 & LFC_TRUST2 & LFC_BRANCH
        LFC --- LFC_COMPILE & LFC_TEST
        LFC --- LFC_PLUGIN & LFC_MCP & LFC_INFRA
    end

    subgraph 分支健康检测
        SB["stale_branch.rs"]
        BF["BranchFreshness"]
        BF_FRESH["Fresh"]
        BF_STALE["Stale"]
        BF_DIV["Diverged"]

        BF --- BF_FRESH & BF_STALE & BF_DIV

        SBP["StaleBranchPolicy"]
        SBP_REBASE["AutoRebase"]
        SBP_MERGE["AutoMergeForward"]
        SBP_WARN["WarnOnly"]
        SBP_BLOCK["Block"]

        SBP --- SBP_REBASE & SBP_MERGE & SBP_WARN & SBP_BLOCK
    end

    subgraph 故障恢复
        RR["recovery_recipes.rs"]
        FS["FailureScenario<br/><i>7 种场景</i>"]
        RS["RecoveryStep<br/><i>可执行步骤</i>"]
        RL["RecoveryLedger<br/><i>恢复尝试记录</i>"]

        FS --> RS
        RS --> RL
    end

    subgraph 策略引擎
        PE_ENG["policy_engine.rs"]
        PR["PolicyRule<br/><i>条件 + 动作 + 优先级</i>"]
        PC_COND["PolicyCondition"]
        PC_AND["And"]
        PC_OR["Or"]
        PC_GREEN3["GreenAt"]
        PC_STALE["StaleBranch"]

        PA["PolicyAction"]
        PA_MERGE["MergeToDev"]
        PA_RECOVER["RecoverOnce"]
        PA_ESCALATE["Escalate"]

        PE_ENG --> PR
        PR --> PC_COND
        PC_COND --- PC_AND & PC_OR & PC_GREEN3 & PC_STALE
        PR --> PA
        PA --- PA_MERGE & PA_RECOVER & PA_ESCALATE
    end

    %% 子系统间关系
    WR -->|"状态变更触发"| LE
    LE -->|"失败事件"| LFC
    LFC -->|"匹配场景"| FS
    BF_STALE & BF_DIV -->|"触发策略"| PE_ENG
    LFC -->|"输入条件"| PC_COND
    PA_RECOVER -->|"执行"| RS

    style WR fill:#4A90D9,color:#fff
    style LE fill:#D94A4A,color:#fff
    style PE_ENG fill:#D9A04A,color:#fff
    style RR fill:#4AD97A,color:#fff
```

---

## 附录：系统全局关系总览

```mermaid
flowchart LR
    USER["用户"] --> CLI_M["CLI 入口<br/><i>rusty-claude-cli</i>"]
    CLI_M --> CONFIG["配置系统<br/><i>runtime/config</i>"]
    CLI_M --> PROMPT["提示词构建<br/><i>runtime/prompt</i>"]
    CLI_M --> SESSION_M["会话管理<br/><i>runtime/session</i>"]

    CONFIG --> PERM["权限系统<br/><i>permission_enforcer</i>"]
    CONFIG --> HOOK_M["Hook 系统<br/><i>hooks</i>"]
    CONFIG --> MCP_M["MCP 集成<br/><i>McpToolRegistry</i>"]

    PROMPT --> CONV["会话运行时<br/><i>ConversationRuntime</i>"]
    SESSION_M --> CONV

    CONV --> PROVIDER["模型提供商<br/><i>ProviderClient</i>"]
    CONV --> TOOL_M["工具系统<br/><i>GlobalToolRegistry</i>"]
    CONV --> COMPACT_M["自动压缩<br/><i>compact</i>"]

    TOOL_M --> PERM
    TOOL_M --> HOOK_M
    TOOL_M --> MCP_M

    PROVIDER --> SSE_M["流式解析<br/><i>SseParser</i>"]
    SSE_M --> RENDER["终端渲染<br/><i>TerminalRenderer</i>"]

    CONV --> WORKER_M["Worker 编排<br/><i>WorkerRegistry</i>"]
    WORKER_M --> LANE_M["Lane 事件<br/><i>LaneEventSchema</i>"]
    LANE_M --> POLICY_M["策略引擎<br/><i>PolicyEngine</i>"]
    POLICY_M --> RECOVER_M["故障恢复<br/><i>RecoveryRecipes</i>"]

    CONV --> TEL_M["遥测<br/><i>telemetry</i>"]

    style USER fill:#4A90D9,color:#fff
    style CONV fill:#D94A4A,color:#fff
    style PROVIDER fill:#D9A04A,color:#fff
    style TOOL_M fill:#4AD97A,color:#fff
```

---

## 关键数据指标

| 指标 | 数值 |
|------|------|
| Rust crate 数量 | 9 |
| Rust 代码行数 | ~48,600 LOC |
| 测试代码行数 | ~2,568 LOC |
| 内置工具数量 | 40+ |
| 支持的模型提供商 | 4（Anthropic / OpenAI / xAI / DashScope） |
| 斜杠命令数量 | 40+ |
| Hook 事件类型 | 3（PreToolUse / PostToolUse / PostToolUseFailure） |
| 权限级别 | 5（ReadOnly / WorkspaceWrite / DangerFullAccess / Prompt / Allow） |
| Lane 事件类型 | 20+ |
| 故障恢复场景 | 7 |
| 会话日志轮转阈值 | 256 KB |
| 自动压缩默认阈值 | 100k tokens |
| 单指令文件字符上限 | 4,000 |
| 指令文件总字符上限 | 12,000 |
