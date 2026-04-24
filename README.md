# Digital Twin - 数字分身平台

## 1. 项目愿景

当前 AI 的使用方式以"会话"为核心 —— 每次对话都是一次性的，知识无法积累，AI 不了解知识之间的关联。Digital Twin（数字分身）平台打破这一限制：

- **不以会话为维度，而以"分身"为维度** —— 每个分身是一个有明确职责、持续学习、能自主执行任务的智能体
- **知识持续积累** —— 对话记录、上传文件、蒸馏后的知识都沉淀在分身的专属空间
- **越用越懂你** —— 分身通过不断学习你的偏好、习惯和领域知识，逐渐成为你的专属助手
- **自动化执行** —— 教会分身工作流后，可定时自动执行（如每日股票复盘、公众号发布）

### 典型使用场景

| 分身 | 职责 | 自动化任务示例 |
|------|------|---------------|
| 股票分析师 | 行情分析、每日复盘、策略回测 | 每日收盘后自动复盘，生成分析报告 |
| 项目开发者 | 代码编写、架构设计、bug 修复 | 每日拉取 issue 并生成开发计划 |
| 育儿顾问 | 成长记录、教育方案、健康追踪 | 每周生成成长报告和教育建议 |
| 健康管家 | 运动计划、饮食记录、体检分析 | 每日提醒运动，每周生成健康周报 |
| 内容创作者 | 公众号写作、热点追踪、排版发布 | 每日追踪热点，定期生成文章草稿 |

---

## 2. 系统架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                     Digital Twin APP                         │
│                    (Tauri Desktop App)                        │
├──────────┬──────────┬───────────┬───────────┬───────────────┤
│  分身管理  │  对话交互  │  知识浏览   │  任务调度   │  全局设置     │
│  Twin Mgr │  Chat    │  Knowledge │  Scheduler │  Settings    │
└─────┬─────┴────┬─────┴─────┬─────┴─────┬─────┴───────┬──────┘
      │          │           │           │             │
┌─────▼──────────▼───────────▼───────────▼─────────────▼──────┐
│                    Twin Engine (核心引擎层)                    │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌───────────┐ │
│  │ Twin       │ │ Knowledge  │ │ Scheduler  │ │ Provider  │ │
│  │ Lifecycle  │ │ Manager    │ │ Engine     │ │ Router    │ │
│  │ 分身生命周期│ │ 知识管理器  │ │ 调度引擎   │ │ 模型路由   │ │
│  └────────────┘ └────────────┘ └────────────┘ └───────────┘ │
├─────────────────────────────────────────────────────────────┤
│                  Claw Code Runtime (底层能力层)                │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌────────┐ ┌───────┐ │
│  │ API  │ │Tools │ │ MCP  │ │Plugin│ │Commands│ │Session│ │
│  │Client│ │Exec  │ │Proto │ │System│ │Registry│ │Persist│ │
│  └──────┘ └──────┘ └──────┘ └──────┘ └────────┘ └───────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │ Anthropic│   │  OpenAI  │   │  其他     │
        │ API      │   │ Compat   │   │ Provider │
        └──────────┘   └──────────┘   └──────────┘
```

### 技术栈选型

| 层级 | 技术 | 理由 |
|------|------|------|
| 桌面应用 | **Tauri 2.0** (Rust + Web) | 与 claw-code Rust 生态一致，包体小，性能好 |
| 前端 UI | **React + TypeScript** | 生态成熟，组件丰富 |
| UI 框架 | **Tailwind CSS + Shadcn/ui** | 现代化设计，快速开发 |
| 核心引擎 | **Rust** (复用 claw-code crates) | 直接复用 runtime、api、tools 等 crate |
| 本地存储 | **SQLite** (索引) + **文件系统** (数据) | 文件系统存原始数据，SQLite 做检索索引 |
| 知识向量化 | **本地 Embedding** + **SQLite-vec** | 支持语义搜索，无需外部向量数据库 |
| 调度系统 | **Tokio Cron Scheduler** | Rust 原生异步调度，复用 claw-code 的 team_cron_registry |

---

## 3. 核心数据模型

### 3.1 分身（Twin）数据结构

```
twins/                              # 所有分身的根目录
├── twin.db                         # SQLite 全局索引数据库
└── {twin-id}/                      # 单个分身目录（如: stock-analyst）
    ├── twin.json                   # 分身元数据（身份、目标、配置）
    ├── persona.md                  # 分身人设描述（系统提示词）
    ├── CLAUDE.md                   # claw-code 项目记忆文件
    │
    ├── conversations/              # 对话记录
    │   ├── index.json              # 对话索引
    │   └── {conv-id}/             
    │       ├── messages.jsonl      # 消息流（追加写入）
    │       └── summary.md          # AI 生成的对话摘要
    │
    ├── uploads/                    # 用户上传的原始文件
    │   ├── index.json              # 文件索引（文件名、类型、上传时间）
    │   └── {file-hash}/           
    │       ├── original.*          # 原始文件
    │       └── extracted.md        # 提取的文本内容
    │
    ├── knowledge/                  # 蒸馏后的知识库
    │   ├── index.json              # 知识条目索引
    │   ├── {topic}/               
    │   │   ├── content.md          # 知识内容（Markdown）
    │   │   └── metadata.json       # 来源、创建时间、关联对话
    │   └── graph.json              # 知识关联图谱
    │
    ├── skills/                     # 分身学会的技能/工作流
    │   ├── index.json              # 技能列表
    │   └── {skill-name}/          
    │       ├── skill.json          # 技能定义（触发条件、执行步骤）
    │       ├── prompt.md           # 技能执行提示词
    │       └── history.jsonl       # 技能执行历史
    │
    ├── schedules/                  # 定时任务
    │   └── schedules.json          # Cron 任务配置
    │
    └── memory/                     # claw-code 记忆系统
        ├── MEMORY.md               # 记忆索引
        └── *.md                    # 各类记忆文件
```

### 3.2 twin.json 元数据

```json
{
  "id": "stock-analyst",
  "name": "股票分析师",
  "avatar": "📈",
  "description": "专注 A 股市场分析，提供每日复盘和投资建议",
  "goal": "帮助用户建立系统化的投资分析框架，提供客观的市场分析",
  "created_at": "2026-04-23T10:00:00Z",
  "updated_at": "2026-04-23T15:30:00Z",
  "model": "claude-sonnet-4-6",
  "status": "active",
  "stats": {
    "total_conversations": 47,
    "total_messages": 326,
    "knowledge_entries": 23,
    "skills_count": 5,
    "total_tokens_used": 1250000
  },
  "tags": ["投资", "股票", "数据分析"],
  "provider_config": {
    "provider": "anthropic",
    "model": "claude-sonnet-4-6",
    "max_tokens": 8192,
    "temperature": 0.3
  }
}
```

### 3.3 Skill（技能）定义

```json
{
  "id": "daily-review",
  "name": "每日复盘",
  "description": "收盘后自动分析持仓股票的当日表现",
  "trigger": {
    "type": "schedule",
    "cron": "0 16 * * 1-5"
  },
  "steps": [
    {
      "action": "fetch_data",
      "description": "获取今日行情数据",
      "tool": "web_fetch",
      "params": { "sources": ["eastmoney", "xueqiu"] }
    },
    {
      "action": "analyze",
      "description": "分析持仓表现",
      "prompt": "skills/daily-review/prompt.md"
    },
    {
      "action": "save_knowledge",
      "description": "保存分析结果到知识库",
      "topic": "daily-reviews/{date}"
    }
  ],
  "output": {
    "type": "markdown",
    "notify": true
  }
}
```

---

## 4. 核心模块设计

### 4.1 Twin Lifecycle（分身生命周期管理）

```
┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
│ Create │───▶│ Active │───▶│ Mature │───▶│Archive │
│ 创建   │    │ 活跃    │    │ 成熟   │    │ 归档   │
└────────┘    └───┬────┘    └───┬────┘    └────────┘
                  │             │
              学习阶段       可自主执行
              需要用户教     定时任务运行
```

**职责：**
- 创建分身：初始化目录结构、生成 twin.json、persona.md
- 激活分身：加载分身上下文到 claw-code runtime，启动会话
- 切换分身：保存当前分身状态，切换到目标分身
- 归档/删除：归档分身数据或清理删除

**与 claw-code 的集成方式：**

```rust
// 复用 claw-code 的 runtime crate
use runtime::{ConversationRuntime, SessionConfig};

pub struct TwinEngine {
    twins_dir: PathBuf,
    active_twin: Option<TwinInstance>,
    db: SqlitePool,
}

pub struct TwinInstance {
    pub meta: TwinMeta,
    pub runtime: ConversationRuntime,  // claw-code 会话运行时
    pub knowledge: KnowledgeManager,
    pub scheduler: SkillScheduler,
}

impl TwinEngine {
    /// 激活一个分身 —— 将分身目录作为 claw-code 的工作目录
    pub async fn activate(&mut self, twin_id: &str) -> Result<()> {
        let twin_dir = self.twins_dir.join(twin_id);
        
        // 加载分身配置
        let meta = TwinMeta::load(&twin_dir)?;
        
        // 以分身目录为工作目录，初始化 claw-code runtime
        // 这样 CLAUDE.md、memory/ 等都自然归属于该分身
        let config = SessionConfig {
            work_dir: twin_dir.clone(),
            model: meta.provider_config.model.clone(),
            system_prompt_extra: fs::read_to_string(twin_dir.join("persona.md"))?,
            ..Default::default()
        };
        
        let runtime = ConversationRuntime::new(config).await?;
        
        self.active_twin = Some(TwinInstance {
            meta,
            runtime,
            knowledge: KnowledgeManager::new(&twin_dir),
            scheduler: SkillScheduler::new(&twin_dir),
        });
        
        Ok(())
    }
}
```

### 4.2 Knowledge Manager（知识管理器）

知识管理是数字分身的核心能力，采用三层架构：

```
         用户输入
            │
            ▼
┌──────────────────────┐
│   Raw Layer (原始层)   │  对话记录、上传文件
│   conversations/      │  完整保留，不做处理
│   uploads/            │
└──────────┬───────────┘
           │ 蒸馏
           ▼
┌──────────────────────┐
│ Knowledge Layer(知识层)│  结构化知识条目
│   knowledge/          │  从对话和文件中提取的要点
│                       │  带有来源追溯和关联关系
└──────────┬───────────┘
           │ 索引
           ▼
┌──────────────────────┐
│  Index Layer (索引层)  │  SQLite 全文检索
│   twin.db             │  向量语义搜索
│   (FTS5 + vec)        │  知识图谱关联查询
└──────────────────────┘
```

**知识蒸馏流程：**

```
对话结束
  │
  ├──▶ 自动摘要：生成对话摘要 → conversations/{id}/summary.md
  │
  ├──▶ 知识提取：从对话中提取关键知识点
  │       │
  │       ├── 新知识 → 创建 knowledge/{topic}/content.md
  │       └── 已有知识 → 更新/补充现有条目
  │
  └──▶ 关系构建：更新 knowledge/graph.json
          - 知识点之间的关联
          - 知识点与对话的追溯链接
          - 知识点与技能的关联
```

**知识检索 —— 用于构建分身上下文：**

每次对话时，系统会根据用户输入自动检索相关知识，注入到 AI 的上下文中：

```rust
pub struct KnowledgeManager {
    twin_dir: PathBuf,
    db: SqlitePool,
}

impl KnowledgeManager {
    /// 对话前：检索相关知识，构建上下文
    pub async fn build_context(&self, user_input: &str) -> Vec<KnowledgeEntry> {
        let mut results = Vec::new();
        
        // 1. 全文检索 (FTS5)
        results.extend(self.fts_search(user_input).await?);
        
        // 2. 向量语义搜索
        results.extend(self.vector_search(user_input).await?);
        
        // 3. 去重 + 按相关度排序
        deduplicate_and_rank(&mut results);
        
        // 4. 裁剪到 token 预算内
        truncate_to_budget(results, MAX_CONTEXT_TOKENS)
    }
    
    /// 对话后：蒸馏知识
    pub async fn distill(&self, conversation: &Conversation) -> Result<()> {
        // 调用 AI 从对话中提取知识点
        let entries = self.extract_knowledge(conversation).await?;
        
        for entry in entries {
            if let Some(existing) = self.find_similar(&entry).await? {
                // 合并到已有知识
                self.merge_knowledge(existing, entry).await?;
            } else {
                // 创建新知识条目
                self.create_knowledge(entry).await?;
            }
        }
        
        // 更新知识图谱
        self.update_graph().await?;
        
        Ok(())
    }
}
```

### 4.3 Skill Scheduler（技能调度引擎）

```rust
pub struct SkillScheduler {
    twin_dir: PathBuf,
    jobs: Vec<ScheduledJob>,
}

pub struct ScheduledJob {
    pub skill_id: String,
    pub cron: String,           // "0 16 * * 1-5" 
    pub next_run: DateTime<Utc>,
    pub last_run: Option<DateTime<Utc>>,
    pub enabled: bool,
}

impl SkillScheduler {
    /// 执行一个技能
    pub async fn execute_skill(&self, skill_id: &str) -> Result<SkillOutput> {
        let skill = self.load_skill(skill_id)?;
        let mut context = SkillContext::new(&self.twin_dir);
        
        for step in &skill.steps {
            match step.action.as_str() {
                "fetch_data" => {
                    // 使用 claw-code 的 tools crate 执行 web_fetch
                    let data = tools::web_fetch(&step.params).await?;
                    context.set("fetched_data", data);
                }
                "analyze" => {
                    // 加载提示词，结合知识库上下文，调用 AI
                    let prompt = fs::read_to_string(&step.prompt)?;
                    let knowledge = self.knowledge.build_context(&prompt).await?;
                    let response = self.runtime.send_message(
                        &format!("{}\n\n相关知识:\n{}", prompt, knowledge)
                    ).await?;
                    context.set("analysis", response);
                }
                "save_knowledge" => {
                    // 保存结果到知识库
                    self.knowledge.save(&step.topic, context.get("analysis")).await?;
                }
                _ => {}
            }
        }
        
        Ok(context.into_output())
    }
}
```

### 4.4 Provider Router（模型路由）

复用 claw-code 的 api crate，支持多模型切换：

```rust
// 直接复用 claw-code 的 api crate
use api::{client::ApiClient, providers::{anthropic, openai_compat}};

pub struct ProviderRouter {
    providers: HashMap<String, Box<dyn Provider>>,
}

impl ProviderRouter {
    /// 每个分身可以配置不同的模型
    /// - 日常对话用 Sonnet（快且便宜）
    /// - 复杂分析用 Opus（最强推理）
    /// - 简单任务用 Haiku（最快最省）
    pub async fn route(&self, twin: &TwinMeta, task_type: TaskType) -> &dyn Provider {
        match task_type {
            TaskType::Chat => &self.providers[&twin.provider_config.model],
            TaskType::Analysis => &self.providers["claude-opus-4-6"],
            TaskType::QuickTask => &self.providers["claude-haiku-4-5"],
        }
    }
}
```

---

## 5. 前端界面设计

### 5.1 整体布局

```
┌─────────────────────────────────────────────────────────┐
│  Digital Twin                              [设置] [通知]  │
├──────────┬──────────────────────────────────────────────┤
│          │                                              │
│  📈 股票  │  ┌─────────────────────────────────────┐    │
│  分析师   │  │         股票分析师                      │    │
│  ────── │  │  目标: 系统化投资分析框架                 │    │
│  💻 项目  │  │  知识: 23条  技能: 5个  对话: 47次      │    │
│  开发者   │  │                                      │    │
│  ────── │  │  [对话] [知识库] [技能] [定时任务]       │    │
│  👶 育儿  │  ├─────────────────────────────────────┤    │
│  顾问    │  │                                      │    │
│  ────── │  │   对话区域 / 知识浏览 / 技能编辑         │    │
│  🏃 健康  │  │                                      │    │
│  管家    │  │                                      │    │
│  ────── │  │                                      │    │
│          │  │                                      │    │
│  ── ── ─ │  │                                      │    │
│  [+新分身]│  │                                      │    │
│          │  └─────────────────────────────────────┘    │
│          │  ┌─────────────────────────────────────┐    │
│          │  │ 💬 输入消息...              [发送]    │    │
│          │  └─────────────────────────────────────┘    │
└──────────┴──────────────────────────────────────────────┘
```

### 5.2 页面结构

| 页面 | 功能 |
|------|------|
| **分身列表**（左侧栏） | 展示所有分身，状态指示（活跃/运行中/休眠），快速切换 |
| **分身主页** | 分身概览：统计、最近对话、知识增长趋势、运行中的任务 |
| **对话页面** | 与分身对话，支持文件上传，实时显示知识提取状态 |
| **知识库页面** | 浏览分身知识：搜索、分类、知识图谱可视化、手动编辑 |
| **技能页面** | 查看/编辑分身技能，测试运行，查看执行历史 |
| **定时任务页面** | 配置自动化任务，查看执行日志，启停控制 |
| **创建分身** | 引导式创建：选择模板或自定义，设置人设和目标 |

### 5.3 关键交互流程

**创建分身流程：**
```
选择创建方式
  ├── 从模板创建（股票分析、项目开发、内容创作...）
  │     └── 预设人设 + 初始技能 + 推荐知识结构
  └── 自定义创建
        ├── 1. 输入名称、描述
        ├── 2. 定义目标和职责
        ├── 3. 选择模型和参数
        └── 4. 创建完成 → 初始化目录 → 开始首次对话
```

**对话 + 知识沉淀流程：**
```
用户发送消息
  │
  ├──▶ 检索分身知识库 → 注入相关知识到上下文
  ├──▶ 加载分身人设 (persona.md)
  ├──▶ 加载分身记忆 (memory/)
  │
  ▼
AI 生成回复（流式输出）
  │
  ├──▶ 实时显示给用户
  ├──▶ 追加到 messages.jsonl
  │
  ▼
对话结束/手动触发
  │
  ├──▶ 生成对话摘要
  ├──▶ 蒸馏知识点 → 用户确认/修改 → 保存
  └──▶ 更新分身记忆（claw-code memory 系统）
```

---

## 6. 项目目录结构

```
digital-twin/
├── README.md                    # 本文档
├── claw-code/                   # claw-code CLI 源码（作为底层引擎）
│   └── rust/                    # Rust 工作空间
│       └── crates/              
│           ├── api/             # API 客户端（直接复用）
│           ├── runtime/         # 会话运行时（直接复用）
│           ├── tools/           # 工具执行（直接复用）
│           ├── plugins/         # 插件系统（直接复用）
│           ├── commands/        # 命令注册（直接复用）
│           └── telemetry/       # 遥测（直接复用）
│
├── crates/                      # Digital Twin 新增 crate
│   ├── twin-core/               # 分身核心逻辑
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── twin.rs          # Twin 生命周期管理
│   │       ├── knowledge.rs     # 知识管理器
│   │       ├── skill.rs         # 技能定义与执行
│   │       ├── scheduler.rs     # 定时调度
│   │       ├── distiller.rs     # 知识蒸馏引擎
│   │       └── db.rs            # SQLite 操作
│   │
│   ├── twin-tauri/              # Tauri 应用集成层
│   │   └── src/
│   │       ├── main.rs          # Tauri 入口
│   │       ├── commands.rs      # Tauri 命令（前后端桥接）
│   │       ├── state.rs         # 应用状态管理
│   │       └── events.rs        # 事件系统（流式推送）
│   │
│   └── twin-templates/          # 分身模板库
│       └── src/
│           ├── lib.rs
│           └── templates/       # 内置模板定义
│
├── ui/                          # 前端（React + TypeScript）
│   ├── package.json
│   ├── src/
│   │   ├── App.tsx
│   │   ├── components/
│   │   │   ├── TwinList.tsx     # 分身列表
│   │   │   ├── TwinChat.tsx     # 对话界面
│   │   │   ├── KnowledgeView.tsx# 知识浏览
│   │   │   ├── SkillEditor.tsx  # 技能编辑
│   │   │   └── SchedulePanel.tsx# 定时任务面板
│   │   ├── hooks/               # React hooks
│   │   ├── stores/              # 状态管理（Zustand）
│   │   └── lib/
│   │       └── tauri.ts         # Tauri API 封装
│   └── tailwind.config.ts
│
├── twins/                       # 用户数据目录（运行时生成）
│   ├── twin.db                  # 全局索引数据库
│   └── {twin-id}/              # 各分身数据（见 3.1 节）
│
└── Cargo.toml                   # 工作空间配置
```

---

## 7. Claw-code 集成策略

### 7.1 复用关系

```
claw-code crate          复用方式              用于
─────────────────────────────────────────────────────
runtime                  依赖引用              会话管理、消息处理、记忆系统
api                      依赖引用              多模型 API 调用、流式输出
tools                    依赖引用              工具执行（文件、搜索、网络）
plugins                  依赖引用              插件扩展机制
commands                 部分复用              复用命令解析，扩展分身专属命令
telemetry                依赖引用              使用量追踪
```

### 7.2 Cargo.toml 工作空间配置

```toml
[workspace]
members = ["crates/*"]
resolver = "2"

[workspace.dependencies]
# 复用 claw-code 的 crate（通过 path 依赖）
claw-runtime = { path = "claw-code/rust/crates/runtime", package = "runtime" }
claw-api = { path = "claw-code/rust/crates/api", package = "api" }
claw-tools = { path = "claw-code/rust/crates/tools", package = "tools" }
claw-plugins = { path = "claw-code/rust/crates/plugins", package = "plugins" }
claw-commands = { path = "claw-code/rust/crates/commands", package = "commands" }
claw-telemetry = { path = "claw-code/rust/crates/telemetry", package = "telemetry" }

# 新增依赖
tauri = "2"
sqlx = { version = "0.8", features = ["sqlite", "runtime-tokio"] }
tokio-cron-scheduler = "0.11"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
```

### 7.3 关键集成点

**1. 会话运行时复用**

每个分身激活时，以分身目录作为 claw-code 的工作目录初始化 `ConversationRuntime`。这样：
- `CLAUDE.md` → 分身的项目记忆
- `memory/` → 分身的持久记忆
- 会话文件自动存储在分身目录下

**2. 知识注入**

在 claw-code 的 `prompt.rs`（系统提示词组装）环节注入分身知识：
- 加载 `persona.md` 作为额外系统提示
- 每次对话前用 KnowledgeManager 检索相关知识
- 将检索结果作为上下文注入消息流

**3. 工具执行复用**

技能执行直接调用 claw-code 的 tools crate：
- `web_fetch` → 抓取网页数据
- `bash` → 执行脚本
- `file_ops` → 文件读写
- `grep/glob` → 搜索

---

## 8. 知识蒸馏机制详解

### 8.1 蒸馏时机

| 触发时机 | 动作 |
|----------|------|
| 对话结束 | 自动生成摘要 + 提取知识点（后台异步） |
| 用户手动触发 | 对选中内容执行蒸馏 |
| 文件上传 | 提取文本内容 + 生成知识条目 |
| 技能执行后 | 保存执行结果为知识 |
| 定期整理 | 合并相似知识、清理过时条目 |

### 8.2 蒸馏提示词模板

```markdown
你是知识蒸馏引擎。分析以下对话内容，提取有价值的知识点。

## 分身背景
{persona.md 内容}

## 已有知识概要
{现有知识条目标题列表}

## 待蒸馏对话
{对话内容}

## 要求
1. 提取可复用的知识点（事实、规则、偏好、决策依据）
2. 标注每个知识点的分类和关联主题
3. 如果与已有知识重叠，标注需要合并/更新的条目
4. 忽略临时性、一次性的信息
5. 以 JSON 格式输出
```

### 8.3 知识图谱

`knowledge/graph.json` 存储知识之间的关联：

```json
{
  "nodes": [
    { "id": "macd-strategy", "label": "MACD 交易策略", "category": "策略" },
    { "id": "risk-control", "label": "风险控制原则", "category": "风控" }
  ],
  "edges": [
    {
      "source": "macd-strategy",
      "target": "risk-control",
      "relation": "depends_on",
      "label": "策略执行需遵循风控规则"
    }
  ]
}
```

---

## 9. 开发路线图

### Phase 1：MVP（核心可用）
- [ ] 项目脚手架搭建（Tauri + React + Rust workspace）
- [ ] 分身 CRUD（创建、编辑、删除、列表）
- [ ] 基础对话功能（复用 claw-code runtime）
- [ ] 对话记录持久化
- [ ] 文件上传与存储
- [ ] 基础知识浏览

### Phase 2：知识引擎
- [ ] 对话自动摘要
- [ ] 知识蒸馏（对话 → 知识条目）
- [ ] 知识检索（全文搜索 + 向量搜索）
- [ ] 对话时自动注入相关知识
- [ ] 知识手动编辑和管理

### Phase 3：技能与自动化
- [ ] 技能定义与编辑
- [ ] 技能手动执行
- [ ] 定时任务调度
- [ ] 执行历史和日志查看
- [ ] 分身模板系统

### Phase 4：高级特性
- [ ] 知识图谱可视化
- [ ] 跨分身知识关联
- [ ] 分身间协作（如：内容创作者调用股票分析师的数据）
- [ ] 导入/导出分身
- [ ] 移动端适配（Tauri Mobile）
- [ ] 分身市场（分享模板）

---

## 10. 核心设计原则

1. **文件优先** —— 所有数据以文件形式存储，可读可编辑，不被锁在数据库里
2. **渐进增强** —— SQLite 只做索引加速，即使数据库损坏也不丢失原始数据
3. **引擎复用** —— 最大化复用 claw-code 的能力，不重复造轮子
4. **离线可用** —— 除 AI API 调用外，所有功能本地运行
5. **用户可控** —— 知识蒸馏结果需用户确认，不自动覆盖用户认知
6. **隐私安全** —— 所有数据本地存储，不上传到任何第三方服务
