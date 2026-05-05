# QwenPaw Code Wiki

> **QwenPaw** — Qwen Personal Agent Workstation  
> 版本: v1.1.6b1 | 许可证: Apache 2.0 | Python: 3.10 ~ <3.14

---

## 目录

1. [项目概览](#1-项目概览)
2. [整体架构](#2-整体架构)
3. [目录结构](#3-目录结构)
4. [核心模块详解](#4-核心模块详解)
   - 4.1 [CLI 命令行层](#41-cli-命令行层)
   - 4.2 [App 应用层](#42-app-应用层)
   - 4.3 [Agents 智能体层](#43-agents-智能体层)
   - 4.4 [Channels 通道层](#44-channels-通道层)
   - 4.5 [Providers 模型提供者层](#45-providers-模型提供者层)
   - 4.6 [Workspace 工作空间](#46-workspace-工作空间)
   - 4.7 [Memory 记忆系统](#47-memory-记忆系统)
   - 4.8 [Context 上下文管理](#48-context-上下文管理)
   - 4.9 [Security 安全系统](#49-security-安全系统)
   - 4.10 [Crons 定时任务](#410-crons-定时任务)
   - 4.11 [MCP 客户端管理](#411-mcp-客户端管理)
   - 4.12 [ACP 智能体协作协议](#412-acp-智能体协作协议)
   - 4.13 [Plugins 插件系统](#413-plugins-插件系统)
   - 4.14 [Skills 技能系统](#414-skills-技能系统)
   - 4.15 [Backup 备份系统](#415-backup-备份系统)
   - 4.16 [Token Usage 用量追踪](#416-token-usage-用量追踪)
5. [Console 前端](#5-console-前端)
6. [关键类与函数索引](#6-关键类与函数索引)
7. [依赖关系图](#7-依赖关系图)
8. [配置与环境变量](#8-配置与环境变量)
9. [项目运行方式](#9-项目运行方式)
10. [测试体系](#10-测试体系)

---

## 1. 项目概览

QwenPaw 是一个开源的**个人 AI 助手平台**，核心特性包括：

- **本地部署**：所有数据和任务运行在用户自己的机器上，无需第三方托管
- **多通道接入**：支持钉钉、飞书、微信、Discord、Telegram、iMessage、QQ 等 15+ 通道
- **多智能体协作**：可创建多个独立 Agent，每个有自己的角色和配置，支持跨 Agent 通信
- **技能扩展**：内置调度、PDF/Office 处理、新闻摘要等技能；支持自定义技能热加载
- **多层安全**：Tool Guard（工具守卫）、文件访问控制、技能安全扫描
- **记忆进化**：Agent 从交互中学习，反思经验，主动服务

技术栈：
- **后端**：Python 3.10+ / FastAPI / AgentScope / agentscope-runtime
- **前端**：React 18 / TypeScript / Vite / Ant Design / Zustand
- **部署**：Docker / pip / 桌面应用 (pywebview)

---

## 2. 整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│                        用户交互层                                 │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐  │
│  │ Console │ │ 钉钉    │ │ 飞书    │ │ Telegram│ │ Discord  │  │
│  │ (Web UI)│ │ Channel │ │ Channel │ │ Channel │ │ Channel  │  │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬─────┘  │
│       │           │           │           │           │         │
│  ┌────┴───────────┴───────────┴───────────┴───────────┴─────┐   │
│  │              ChannelManager (统一队列管理)                 │   │
│  └──────────────────────────┬───────────────────────────────┘   │
└─────────────────────────────┼────────────────────────────────────┘
                              │
┌─────────────────────────────┼────────────────────────────────────┐
│                     App 应用层                                    │
│  ┌──────────────────────────┴───────────────────────────────┐   │
│  │            DynamicMultiAgentRunner (请求路由)              │   │
│  └──────────────┬──────────────────────────────┬────────────┘   │
│                 │                              │                 │
│  ┌──────────────┴──────────┐  ┌───────────────┴────────────┐   │
│  │   MultiAgentManager     │  │   FastAPI Routers          │   │
│  │   (多Agent生命周期)      │  │   (REST API 端点)          │   │
│  └──────────────┬──────────┘  └────────────────────────────┘   │
└─────────────────┼───────────────────────────────────────────────┘
                  │
┌─────────────────┼───────────────────────────────────────────────┐
│           Workspace 工作空间层 (每个Agent一个)                    │
│  ┌──────────────┴──────────────────────────────────────────┐   │
│  │  Workspace                                               │   │
│  │  ┌──────────┐ ┌──────────────┐ ┌──────────────────┐    │   │
│  │  │  Runner   │ │ChannelManager│ │  CronManager     │    │   │
│  │  │ (请求处理)│ │ (通道管理)   │ │ (定时任务)       │    │   │
│  │  └─────┬────┘ └──────────────┘ └──────────────────┘    │   │
│  │        │                                                 │   │
│  │  ┌─────┴─────────────────────────────────────────────┐  │   │
│  │  │            QwenPawAgent (ReActAgent)               │  │   │
│  │  │  ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │  │   │
│  │  │  │ Toolkit  │ │  Memory  │ │ Context Manager  │  │  │   │
│  │  │  │ (工具集) │ │ Manager  │ │ (上下文管理)     │  │  │   │
│  │  │  └──────────┘ └──────────┘ └──────────────────┘  │  │   │
│  │  │  ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │  │   │
│  │  │  │  Skills  │ │   MCP    │ │   Tool Guard     │  │  │   │
│  │  │  │ (技能)   │ │ Clients  │ │ (安全守卫)       │  │  │   │
│  │  │  └──────────┘ └──────────┘ └──────────────────┘  │  │   │
│  │  └──────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────┼───────────────────────────────────┐
│                    基础设施层                                     │
│  ┌──────────────┐ ┌─────────┴──┐ ┌──────────────┐ ┌─────────┐ │
│  │   Provider   │ │   Config   │ │   Security   │ │  Token  │ │
│  │   Manager    │ │ (配置管理) │ │ (安全系统)   │ │  Usage  │ │
│  │ (模型提供者) │ │            │ │              │ │ (用量)  │ │
│  └──────────────┘ └────────────┘ └──────────────┘ └─────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**核心数据流**：

1. 用户消息通过 Channel 进入系统
2. ChannelManager 通过 UnifiedQueueManager 排队和路由消息
3. DynamicMultiAgentRunner 根据 `X-Agent-Id` 头路由到对应 Workspace
4. Workspace 中的 Runner 调用 QwenPawAgent 进行 ReAct 推理
5. Agent 通过 Toolkit 调用内置工具/Skills/MCP 工具
6. Tool Guard 在工具执行前进行安全拦截
7. 结果通过 Channel 原路返回给用户

---

## 3. 目录结构

```
QwenPaw/
├── src/qwenpaw/                 # Python 后端主包
│   ├── __init__.py              # 包初始化，加载环境变量和日志
│   ├── __main__.py              # python -m qwenpaw 入口
│   ├── __version__.py           # 版本号 (1.1.6b1)
│   ├── constant.py              # 全局常量和环境变量加载
│   ├── exceptions.py            # 自定义异常体系
│   ├── cli/                     # CLI 命令行模块
│   │   ├── main.py              # Click CLI 主入口 (LazyGroup)
│   │   ├── app_cmd.py           # qwenpaw app 启动命令
│   │   ├── init_cmd.py          # qwenpaw init 初始化命令
│   │   ├── agents_cmd.py        # Agent 管理命令
│   │   ├── channels_cmd.py      # Channel 管理命令
│   │   ├── cron_cmd.py          # 定时任务命令
│   │   ├── skills_cmd.py        # 技能管理命令
│   │   ├── providers_cmd.py     # 模型提供者命令
│   │   ├── daemon_cmd.py        # 守护进程命令
│   │   ├── doctor_cmd.py        # 诊断命令
│   │   └── ...                  # 其他 CLI 子命令
│   ├── app/                     # 应用核心模块
│   │   ├── _app.py              # FastAPI 应用定义和生命周期
│   │   ├── auth.py              # Web 认证中间件
│   │   ├── multi_agent_manager.py # 多 Agent 管理器
│   │   ├── agent_context.py     # Agent 上下文 (请求级)
│   │   ├── agent_config_watcher.py # 配置热重载监听
│   │   ├── migration.py         # 数据迁移
│   │   ├── console_push_store.py # Console 推送存储
│   │   ├── channels/            # 通道实现
│   │   │   ├── base.py          # BaseChannel 抽象基类
│   │   │   ├── manager.py       # ChannelManager 队列管理
│   │   │   ├── registry.py      # 通道注册表
│   │   │   ├── renderer.py      # 消息渲染器
│   │   │   ├── schema.py        # 通道类型枚举
│   │   │   ├── command_registry.py # 控制命令注册
│   │   │   ├── unified_queue_manager.py # 统一队列管理
│   │   │   ├── console/         # Web Console 通道
│   │   │   ├── dingtalk/        # 钉钉通道
│   │   │   ├── feishu/          # 飞书通道
│   │   │   ├── wecom/           # 企业微信通道
│   │   │   ├── weixin/          # 微信通道
│   │   │   ├── telegram/        # Telegram 通道
│   │   │   ├── discord_/        # Discord 通道
│   │   │   ├── qq/              # QQ 通道
│   │   │   ├── imessage/        # iMessage 通道
│   │   │   ├── matrix/          # Matrix 通道
│   │   │   ├── mattermost/      # Mattermost 通道
│   │   │   ├── mqtt/            # MQTT 通道
│   │   │   ├── sip/             # SIP 语音通道
│   │   │   ├── voice/           # 语音通道 (Twilio)
│   │   │   ├── onebot/          # OneBot 通道
│   │   │   └── xiaoyi/          # 小翼通道
│   │   ├── routers/             # FastAPI 路由
│   │   │   ├── agent.py         # Agent 交互端点
│   │   │   ├── agent_scoped.py  # Agent 作用域路由
│   │   │   ├── agents.py        # Agent 管理端点
│   │   │   ├── auth.py          # 认证端点
│   │   │   ├── config.py        # 配置端点
│   │   │   ├── providers.py     # 模型提供者端点
│   │   │   ├── skills.py        # 技能端点
│   │   │   ├── mcp.py           # MCP 端点
│   │   │   ├── workspace.py     # 工作空间端点
│   │   │   └── ...              # 其他路由
│   │   ├── runner/              # 请求运行器
│   │   │   ├── runner.py        # AgentRunner 主逻辑
│   │   │   ├── session.py       # 会话管理
│   │   │   ├── task_tracker.py  # 任务追踪器
│   │   │   ├── command_dispatch.py # 命令分发
│   │   │   ├── mission_dispatch.py # 任务分发
│   │   │   └── control_commands/ # 控制命令处理
│   │   ├── workspace/           # 工作空间
│   │   │   ├── workspace.py     # Workspace 类定义
│   │   │   ├── service_manager.py # 服务管理器
│   │   │   └── service_factories.py # 服务工厂
│   │   ├── crons/               # 定时任务
│   │   │   ├── manager.py       # CronManager
│   │   │   ├── executor.py      # CronExecutor
│   │   │   ├── heartbeat.py     # 心跳任务
│   │   │   ├── models.py        # 任务模型
│   │   │   └── repo/            # 任务持久化
│   │   ├── mcp/                 # MCP 客户端管理
│   │   │   ├── manager.py       # MCPClientManager
│   │   │   ├── stateful_client.py # 有状态 MCP 客户端
│   │   │   └── watcher.py       # 配置热重载监听
│   │   └── approvals/           # 审批服务
│   ├── agents/                  # Agent 核心
│   │   ├── react_agent.py       # QwenPawAgent 主类
│   │   ├── model_factory.py     # 模型工厂
│   │   ├── prompt.py            # 系统提示词构建
│   │   ├── templates.py         # Agent 模板
│   │   ├── command_handler.py   # 系统命令处理
│   │   ├── tool_guard_mixin.py  # Tool Guard 混入
│   │   ├── skills_manager.py    # 技能管理
│   │   ├── skills_hub.py        # 技能中心
│   │   ├── routing_chat_model.py # 路由聊天模型
│   │   ├── schema.py            # Agent 数据结构
│   │   ├── tools/               # 内置工具
│   │   │   ├── shell.py         # Shell 命令执行
│   │   │   ├── file_io.py       # 文件读写
│   │   │   ├── file_search.py   # 文件搜索 (grep/glob)
│   │   │   ├── browser_control.py # 浏览器控制
│   │   │   ├── browser_snapshot.py # 浏览器截图
│   │   │   ├── desktop_screenshot.py # 桌面截图
│   │   │   ├── send_file.py     # 发送文件
│   │   │   ├── view_media.py    # 查看媒体
│   │   │   ├── agent_management.py # Agent 管理
│   │   │   ├── delegate_external_agent.py # 外部 Agent 委托
│   │   │   ├── get_current_time.py # 获取当前时间
│   │   │   ├── get_token_usage.py # 获取 Token 用量
│   │   │   └── utils.py         # 工具辅助函数
│   │   ├── memory/              # 记忆系统
│   │   │   ├── base_memory_manager.py # 记忆管理器基类
│   │   │   ├── reme_light_memory_manager.py # ReMe 轻量记忆
│   │   │   ├── agent_md_manager.py # Agent.md 记忆
│   │   │   └── prompts.py       # 记忆提示词
│   │   ├── context/             # 上下文管理
│   │   │   ├── base_context_manager.py # 上下文管理器基类
│   │   │   ├── light_context_manager.py # 轻量上下文管理
│   │   │   ├── agent_context.py # Agent 上下文
│   │   │   └── compactor_prompts.py # 压缩提示词
│   │   ├── mission/             # 任务系统
│   │   │   ├── mission_runner.py # 任务运行器
│   │   │   ├── handler.py       # 任务处理器
│   │   │   └── state.py         # 任务状态
│   │   ├── acp/                 # Agent 协作协议
│   │   │   ├── core.py          # ACP 核心定义
│   │   │   ├── client.py        # ACP 客户端
│   │   │   ├── server.py        # ACP 服务端
│   │   │   ├── service.py       # ACP 服务
│   │   │   ├── permissions.py   # 权限管理
│   │   │   └── tool_adapter.py  # 工具适配器
│   │   ├── hooks/               # Agent 钩子
│   │   │   └── bootstrap.py     # 引导钩子
│   │   ├── skills/              # 技能定义目录
│   │   └── utils/               # Agent 工具函数
│   │       ├── registry.py      # 注册表
│   │       ├── token_counter.py # Token 计数
│   │       ├── message_processing.py # 消息处理
│   │       └── file_handling.py # 文件处理
│   ├── providers/               # 模型提供者
│   │   ├── provider.py          # Provider 抽象基类
│   │   ├── provider_manager.py  # ProviderManager 单例
│   │   ├── openai_provider.py   # OpenAI 提供者
│   │   ├── anthropic_provider.py # Anthropic 提供者
│   │   ├── gemini_provider.py   # Gemini 提供者
│   │   ├── ollama_provider.py   # Ollama 提供者
│   │   ├── lmstudio_provider.py # LM Studio 提供者
│   │   ├── openrouter_provider.py # OpenRouter 提供者
│   │   ├── rate_limiter.py      # 速率限制器
│   │   ├── retry_chat_model.py  # 重试聊天模型
│   │   ├── multimodal_prober.py # 多模态探测
│   │   └── model_capability_cache.py # 模型能力缓存
│   ├── config/                  # 配置管理
│   │   ├── config.py            # 配置模型定义 (Pydantic)
│   │   ├── context.py           # 配置上下文
│   │   ├── utils.py             # 配置工具函数
│   │   └── timezone.py          # 时区处理
│   ├── security/                # 安全系统
│   │   ├── tool_guard/          # 工具守卫
│   │   │   ├── engine.py        # ToolGuardEngine
│   │   │   ├── models.py        # 安全模型
│   │   │   ├── approval.py      # 审批机制
│   │   │   ├── execution_level.py # 执行级别
│   │   │   └── guardians/       # 守卫者
│   │   │       ├── file_guardian.py # 文件路径守卫
│   │   │       ├── rule_guardian.py # 规则守卫
│   │   │       └── shell_evasion_guardian.py # Shell 逃逸守卫
│   │   ├── skill_scanner/       # 技能安全扫描
│   │   │   ├── scanner.py       # 技能扫描器
│   │   │   ├── scan_policy.py   # 扫描策略
│   │   │   └── analyzers/       # 分析器
│   │   └── secret_store.py      # 密钥存储
│   ├── plugins/                 # 插件系统
│   │   ├── architecture.py      # 插件架构定义
│   │   ├── loader.py            # 插件加载器
│   │   ├── registry.py          # 插件注册表
│   │   ├── runtime.py           # 运行时辅助
│   │   └── api.py               # 插件 API
│   ├── backup/                  # 备份系统
│   │   ├── orchestration.py     # 备份编排
│   │   ├── models.py            # 备份模型
│   │   ├── _ops/                # 备份操作
│   │   └── _utils/              # 备份工具
│   ├── token_usage/             # Token 用量追踪
│   │   ├── manager.py           # TokenUsageManager
│   │   ├── buffer.py            # 用量缓冲
│   │   ├── storage.py           # 用量存储
│   │   └── model_wrapper.py     # 模型包装器
│   ├── local_models/            # 本地模型管理
│   │   ├── manager.py           # LocalModelManager
│   │   ├── download_manager.py  # 下载管理
│   │   ├── llamacpp.py          # llama.cpp 后端
│   │   └── model_manager.py     # 模型管理
│   ├── envs/                    # 环境变量管理
│   │   └── store.py             # 环境变量存储
│   ├── plan/                    # 计划模式
│   │   ├── schemas.py           # 计划模型
│   │   ├── hints.py             # 计划提示
│   │   └── broadcast.py         # 广播
│   ├── tunnel/                  # 隧道 (Cloudflare)
│   ├── utils/                   # 通用工具
│   └── tokenizer/               # 分词器资源
├── console/                     # React 前端
│   ├── src/
│   │   ├── api/                 # API 客户端层
│   │   ├── components/          # UI 组件
│   │   ├── pages/               # 页面
│   │   ├── stores/              # Zustand 状态管理
│   │   ├── hooks/               # 自定义 Hooks
│   │   ├── contexts/            # React Context
│   │   ├── plugins/             # 插件加载
│   │   ├── locales/             # 国际化 (en/zh/ja/ru)
│   │   ├── layouts/             # 布局
│   │   └── utils/               # 工具函数
│   ├── package.json             # 前端依赖
│   └── vite.config.ts           # Vite 配置
├── website/                     # 文档网站
├── tests/                       # 测试
│   ├── unit/                    # 单元测试
│   ├── contract/                # 契约测试
│   ├── integration/             # 集成测试
│   └── e2e/                     # 端到端测试
├── deploy/                      # Docker 部署
├── scripts/                     # 构建和安装脚本
├── pyproject.toml               # Python 项目配置
├── docker-compose.yml           # Docker Compose
└── Makefile                     # 测试和构建命令
```

---

## 4. 核心模块详解

### 4.1 CLI 命令行层

**入口**：[main.py](file:///workspace/src/qwenpaw/cli/main.py)

CLI 基于 [Click](https://click.palletsprojects.com/) 框架，使用 `LazyGroup` 实现子命令懒加载，避免启动时导入所有模块。

**核心类**：

| 类/函数 | 文件 | 说明 |
|---------|------|------|
| `LazyGroup` | `cli/main.py` | 支持懒加载的 Click Group，按需导入子命令 |
| `cli()` | `cli/main.py` | CLI 主入口，注册所有子命令 |

**子命令列表**：

| 命令 | 模块 | 说明 |
|------|------|------|
| `qwenpaw app` | `cli/app_cmd.py` | 启动 Web 应用 |
| `qwenpaw init` | `cli/init_cmd.py` | 交互式初始化配置 |
| `qwenpaw agents` | `cli/agents_cmd.py` | Agent 增删改查 |
| `qwenpaw channels` | `cli/channels_cmd.py` | 通道管理 |
| `qwenpaw cron` | `cli/cron_cmd.py` | 定时任务管理 |
| `qwenpaw skills` | `cli/skills_cmd.py` | 技能管理 |
| `qwenpaw models` | `cli/providers_cmd.py` | 模型提供者管理 |
| `qwenpaw daemon` | `cli/daemon_cmd.py` | 守护进程管理 |
| `qwenpaw doctor` | `cli/doctor_cmd.py` | 系统诊断 |
| `qwenpaw env` | `cli/env_cmd.py` | 环境变量管理 |
| `qwenpaw auth` | `cli/auth_cmd.py` | 认证管理 |
| `qwenpaw plugin` | `cli/plugin_commands.py` | 插件管理 |
| `qwenpaw task` | `cli/task_cmd.py` | 任务管理 |
| `qwenpaw desktop` | `cli/desktop_cmd.py` | 桌面应用 |
| `qwenpaw update` | `cli/update_cmd.py` | 更新检查 |
| `qwenpaw shutdown` | `cli/shutdown_cmd.py` | 关闭服务 |
| `qwenpaw uninstall` | `cli/uninstall_cmd.py` | 卸载 |
| `qwenpaw acp` | `cli/acp_cmd.py` | ACP 管理 |

---

### 4.2 App 应用层

**核心文件**：[_app.py](file:///workspace/src/qwenpaw/app/_app.py)

应用层是整个系统的入口，基于 FastAPI 构建，负责：

1. **FastAPI 应用生命周期管理**：通过 `lifespan` 上下文管理器管理启动和关闭
2. **多 Agent 请求路由**：`DynamicMultiAgentRunner` 根据 `X-Agent-Id` 头路由请求
3. **静态文件服务**：服务 Console 前端构建产物
4. **中间件链**：认证、CORS、Agent 上下文

**启动流程**：

```
Phase 1 (快速同步，< 100ms):
  ├── 自动注册认证
  ├── 遥测数据收集
  ├── 数据迁移
  ├── 创建 MultiAgentManager
  ├── 创建 ProviderManager
  ├── 创建 LocalModelManager
  ├── 启动 TokenUsageManager
  └── 服务器就绪，开始接受请求

Phase 2 (后台异步):
  ├── 并行启动所有 Agent Workspace
  ├── 启动本地模型恢复
  ├── 加载插件系统
  ├── 注册插件控制命令
  ├── 执行插件启动钩子
  └── 设置审批服务
```

**关键类**：

| 类 | 说明 |
|----|------|
| `DynamicMultiAgentRunner` | 动态路由器，将请求分发到正确的 Workspace Runner |
| `AgentApp` | agentscope-runtime 提供的应用框架 |

**API 路由结构**：

| 前缀 | 路由模块 | 说明 |
|------|----------|------|
| `/api` | `routers/` | 通用 API |
| `/api/agent` | `AgentApp.router` | Agent 运行时 API |
| `/api/approval` | `routers/approval.py` | 审批 API |
| `/api/agents/{agentId}/...` | `routers/agent_scoped.py` | Agent 作用域 API |

---

### 4.3 Agents 智能体层

**核心文件**：[react_agent.py](file:///workspace/src/qwenpaw/agents/react_agent.py)

QwenPawAgent 是系统的核心智能体类，继承自 AgentScope 的 `ReActAgent`，并混入 `ToolGuardMixin` 提供安全拦截。

**类继承关系**：

```
QwenPawAgent → ToolGuardMixin → ReActAgent → BaseAgent
```

**QwenPawAgent 核心职责**：

| 方法 | 说明 |
|------|------|
| `__init__()` | 初始化工具集、技能、记忆、上下文、系统提示词 |
| `reply()` | 处理用户消息，支持系统命令和正常对话 |
| `_reasoning()` | 推理步骤，含媒体过滤（主动+被动）和自动续写 |
| `_acting()` | 工具执行步骤，含计划门控和 Tool Guard 拦截 |
| `_summarizing()` | 摘要步骤，含媒体过滤和 tool_use 块清理 |
| `register_mcp_clients()` | 注册 MCP 客户端到工具集 |
| `rebuild_sys_prompt()` | 重建系统提示词 |
| `interrupt()` | 中断当前回复 |

**内置工具列表**：

| 工具名 | 函数 | 说明 |
|--------|------|------|
| `execute_shell_command` | `shell.execute_shell_command` | 执行 Shell 命令 |
| `read_file` | `file_io.read_file` | 读取文件 |
| `write_file` | `file_io.write_file` | 写入文件 |
| `edit_file` | `file_io.edit_file` | 编辑文件 |
| `grep_search` | `file_search.grep_search` | Grep 搜索 |
| `glob_search` | `file_search.glob_search` | Glob 搜索 |
| `browser_use` | `browser_control.browser_use` | 浏览器控制 |
| `desktop_screenshot` | `desktop_screenshot.desktop_screenshot` | 桌面截图 |
| `view_image` | `view_media.view_image` | 查看图片 |
| `view_video` | `view_media.view_video` | 查看视频 |
| `send_file_to_user` | `send_file.send_file_to_user` | 发送文件给用户 |
| `get_current_time` | `get_current_time.get_current_time` | 获取当前时间 |
| `set_user_timezone` | `get_current_time.set_user_timezone` | 设置用户时区 |
| `get_token_usage` | `get_token_usage.get_token_usage` | 获取 Token 用量 |
| `delegate_external_agent` | `delegate_external_agent.delegate_external_agent` | 委托外部 Agent |
| `list_agents` | `agent_management.list_agents` | 列出所有 Agent |
| `chat_with_agent` | `agent_management.chat_with_agent` | 与其他 Agent 对话 |
| `submit_to_agent` | `agent_management.submit_to_agent` | 提交任务给 Agent |
| `check_agent_task` | `agent_management.check_agent_task` | 检查 Agent 任务状态 |

**系统命令**（`CommandHandler` 处理）：

| 命令 | 说明 |
|------|------|
| `/compact` | 压缩上下文 |
| `/new` | 新建会话 |
| `/clear` | 清除会话 |
| `/history` | 查看历史 |
| `/plan` | 计划模式 |
| `/proactive` | 主动服务 |
| `/dump_history` | 导出调试历史 |
| `/load_history` | 加载调试历史 |

---

### 4.4 Channels 通道层

**核心文件**：[base.py](file:///workspace/src/qwenpaw/app/channels/base.py), [manager.py](file:///workspace/src/qwenpaw/app/channels/manager.py)

通道层实现了多平台消息接入的统一抽象。

**类层次**：

```
BaseChannel (ABC)
  ├── ConsoleChannel      # Web Console (SSE)
  ├── DingTalkChannel     # 钉钉
  ├── FeishuChannel       # 飞书
  ├── WeComChannel        # 企业微信
  ├── WeixinChannel       # 微信
  ├── TelegramChannel     # Telegram
  ├── DiscordChannel      # Discord
  ├── QQChannel           # QQ
  ├── iMessageChannel     # iMessage (macOS)
  ├── MatrixChannel       # Matrix
  ├── MattermostChannel   # Mattermost
  ├── MQTTChannel         # MQTT
  ├── OneBotChannel       # OneBot
  ├── VoiceChannel        # 语音 (Twilio)
  ├── SIPChannel          # SIP 语音
  └── XiaoYiChannel       # 小翼
```

**BaseChannel 核心方法**：

| 方法 | 说明 |
|------|------|
| `consume_one()` | 消费一条消息（含时间防抖） |
| `build_agent_request_from_native()` | 将原生消息转为 AgentRequest |
| `send()` | 发送文本消息（子类实现） |
| `send_content_parts()` | 发送多类型内容 |
| `send_media()` | 发送媒体附件 |
| `start()` / `stop()` | 启动/停止通道 |
| `health_check()` | 健康检查 |
| `clone()` | 克隆通道实例（用于热重载） |

**ChannelManager 核心方法**：

| 方法 | 说明 |
|------|------|
| `from_config()` | 从配置创建所有通道 |
| `start_all()` | 启动所有通道和队列管理器 |
| `stop_all()` | 停止所有通道 |
| `enqueue()` | 线程安全地入队消息 |
| `replace_channel()` | 热替换通道（不停机重载） |
| `restart_channel()` | 重启单个通道 |
| `send_event()` | 发送事件到指定通道 |
| `send_text()` | 发送文本到指定通道 |

**消息处理流程**：

```
原生消息 → enqueue() → UnifiedQueueManager → _consume_queue()
  → BaseChannel.consume_one() → _consume_one_request()
  → _payload_to_request() → _process() (AgentRunner)
  → on_event_content() / on_event_message_completed() → send()
```

---

### 4.5 Providers 模型提供者层

**核心文件**：[provider.py](file:///workspace/src/qwenpaw/providers/provider.py), [provider_manager.py](file:///workspace/src/qwenpaw/providers/provider_manager.py)

Provider 层抽象了不同 LLM API 的接入方式，统一为 `ChatModelBase` 接口。

**类层次**：

```
Provider (ABC) ← ProviderInfo
  ├── OpenAIProvider       # OpenAI / DashScope / 兼容 API
  ├── AnthropicProvider    # Anthropic Claude
  ├── GeminiProvider       # Google Gemini
  ├── OllamaProvider       # Ollama 本地模型
  ├── LMStudioProvider     # LM Studio 本地模型
  └── OpenRouterProvider   # OpenRouter 聚合
```

**Provider 核心方法**：

| 方法 | 说明 |
|------|------|
| `check_connection()` | 检查 API 连通性 |
| `fetch_models()` | 获取可用模型列表 |
| `check_model_connection()` | 检查特定模型可用性 |
| `get_chat_model_instance()` | 获取 ChatModel 实例 |
| `probe_model_multimodal()` | 探测模型多模态能力 |
| `add_model()` / `delete_model()` | 添加/删除模型 |
| `update_config()` | 更新提供者配置 |

**ProviderManager**：全局单例，管理所有 Provider 实例，提供模型路由、速率限制、重试机制。

**关键辅助组件**：

| 组件 | 说明 |
|------|------|
| `RateLimiter` | 速率限制（信号量 + QPM 滑动窗口） |
| `RetryChatModel` | 带指数退避重试的 ChatModel 包装器 |
| `MultimodalProber` | 自动探测模型多模态能力 |
| `ModelCapabilityCache` | 缓存模型能力查询结果 |
| `OpenAIChatModelCompat` | OpenAI 兼容层消息格式化 |

---

### 4.6 Workspace 工作空间

**核心文件**：[workspace.py](file:///workspace/src/qwenpaw/app/workspace/workspace.py)

每个 Agent 拥有独立的 Workspace，封装了完整的运行时组件。

**Workspace 组成**：

| 组件 | 类型 | 说明 |
|------|------|------|
| `runner` | `AgentRunner` | 处理 Agent 请求 |
| `channel_manager` | `ChannelManager` | 管理通信通道 |
| `memory_manager` | `BaseMemoryManager` | 管理对话记忆 |
| `mcp_client_manager` | `MCPClientManager` | 管理 MCP 客户端 |
| `cron_manager` | `CronManager` | 管理定时任务 |
| `task_tracker` | `TaskTracker` | 追踪运行中任务 |
| `chat_manager` | `ChatManager` | 管理聊天会话 |

**MultiAgentManager**：管理所有 Workspace 的生命周期，支持懒加载、并行启动和热重载。

---

### 4.7 Memory 记忆系统

**核心文件**：[base_memory_manager.py](file:///workspace/src/qwenpaw/agents/memory/base_memory_manager.py)

记忆系统为 Agent 提供长期记忆能力，支持搜索、摘要和反思。

**类层次**：

```
BaseMemoryManager (ABC)
  ├── RemeLightMemoryManager   # 基于 ReMe 的轻量记忆
  └── AgentMdManager           # 基于 Agent.md 的记忆
```

**BaseMemoryManager 核心方法**：

| 方法 | 说明 |
|------|------|
| `start()` | 初始化存储后端 |
| `close()` | 关闭并释放资源 |
| `summarize()` | 摘要对话记忆 |
| `memory_search()` | 搜索记忆 |
| `list_memory_tools()` | 返回记忆工具列表（注册到 Toolkit） |

---

### 4.8 Context 上下文管理

**核心文件**：[base_context_manager.py](file:///workspace/src/qwenpaw/agents/context/base_context_manager.py)

上下文管理器负责控制对话上下文窗口，防止超出模型 Token 限制。

**类层次**：

```
BaseContextManager (ABC)
  └── LightContextManager      # 轻量上下文管理
```

**核心功能**：

| 功能 | 说明 |
|------|------|
| **Compaction** | 上下文接近 Token 上限时压缩旧消息 |
| **Tool-result pruning** | 裁剪过大的工具输出 |
| **生命周期钩子** | `pre_reasoning`, `post_acting`, `pre_reply`, `post_reply` |

---

### 4.9 Security 安全系统

**核心文件**：[engine.py](file:///workspace/src/qwenpaw/security/tool_guard/engine.py)

安全系统提供多层防护：

#### Tool Guard（工具守卫）

在工具执行前拦截和检查，防止危险操作。

```
ToolGuardEngine
  ├── RuleBasedToolGuardian     # 规则守卫 (YAML 规则)
  ├── FilePathToolGuardian      # 文件路径守卫
  └── ShellEvasionGuardian      # Shell 逃逸守卫
```

**工作流程**：
1. Agent 调用工具 → `ToolGuardMixin._acting()` 拦截
2. `ToolGuardEngine.guard()` 运行所有 Guardian
3. 根据严重级别决定：放行 / 需要审批 / 拒绝
4. 需要审批时通过 ApprovalService 请求用户确认

#### Skill Scanner（技能扫描器）

安装技能前自动扫描安全风险：

| 检测项 | 说明 |
|--------|------|
| Prompt 注入 | 检测提示词注入攻击 |
| 命令注入 | 检测 Shell 命令注入 |
| 硬编码密钥 | 检测代码中的 API Key |
| 数据泄露 | 检测数据外传行为 |

#### Secret Store（密钥存储）

使用系统 Keyring 安全存储 API Key 等敏感信息。

---

### 4.10 Crons 定时任务

**核心文件**：[manager.py](file:///workspace/src/qwenpaw/app/crons/manager.py)

基于 APScheduler 的定时任务系统。

**CronManager 核心方法**：

| 方法 | 说明 |
|------|------|
| `add_job()` | 添加定时任务 |
| `remove_job()` | 删除定时任务 |
| `list_jobs()` | 列出所有任务 |
| `start()` / `stop()` | 启动/停止调度器 |

**特殊任务**：

| 任务 | 说明 |
|------|------|
| `_heartbeat` | 心跳任务，定期检查和推送摘要 |
| `_dream` | 梦境任务，记忆反思和整理 |

**持久化**：通过 `BaseJobRepository` → `JsonJobRepository` 将任务存储到 JSON 文件。

---

### 4.11 MCP 客户端管理

**核心文件**：[manager.py](file:///workspace/src/qwenpaw/app/mcp/manager.py)

MCP (Model Context Protocol) 客户端管理器，支持热重载。

**MCPClientManager 核心方法**：

| 方法 | 说明 |
|------|------|
| `init_from_config()` | 从配置初始化客户端 |
| `get_clients()` | 获取所有客户端 |
| `reload_config()` | 热重载配置 |
| `close_all()` | 关闭所有客户端 |

**客户端类型**：

| 类型 | 类 | 说明 |
|------|-----|------|
| HTTP | `HttpStatefulClient` | HTTP/SSE 传输 |
| StdIO | `StdIOStatefulClient` | 标准输入输出传输 |

---

### 4.12 ACP 智能体协作协议

**核心文件**：[core.py](file:///workspace/src/qwenpaw/agents/acp/core.py)

ACP (Agent Client Protocol) 实现了 Agent 间的协作通信协议。

**核心组件**：

| 组件 | 说明 |
|------|------|
| `ACPClient` | ACP 客户端，连接外部 Agent |
| `ACPServer` | ACP 服务端，暴露 Agent 能力 |
| `ACPService` | ACP 服务管理 |
| `Permissions` | 权限管理 |
| `ToolAdapter` | 工具适配器，将 ACP 工具转为 AgentScope 工具 |

**预配置的 ACP Agent**：

| Agent | 命令 | 说明 |
|-------|------|------|
| `opencode` | `opencode acp` | OpenCode |
| `qwen_code` | `qwen --acp` | Qwen Code |
| `claude_code` | `npx @zed-industries/claude-agent-acp` | Claude Code |
| `codex` | `npx @zed-industries/codex-acp` | Codex |

---

### 4.13 Plugins 插件系统

**核心文件**：[architecture.py](file:///workspace/src/qwenpaw/plugins/architecture.py)

插件系统允许第三方扩展 QwenPaw 的功能。

**插件架构**：

```
PluginManifest          # 插件清单 (id, name, version, entry)
PluginRecord            # 加载记录 (manifest, source_path, enabled, instance)
PluginLoader            # 插件加载器 (发现、加载、验证)
PluginRegistry          # 插件注册表 (管理 Provider、Command、Hook)
RuntimeHelpers          # 运行时辅助 (ProviderManager 访问)
```

**插件扩展点**：

| 扩展点 | 说明 |
|--------|------|
| `Provider` | 注册新的模型提供者 |
| `ControlCommand` | 注册新的控制命令 |
| `StartupHook` | 应用启动时执行 |
| `ShutdownHook` | 应用关闭时执行 |
| `Frontend Module` | 前端模块扩展 |

---

### 4.14 Skills 技能系统

**核心文件**：`agents/skills_manager.py`, `agents/skills_hub.py`

技能是 Agent 能力的扩展单元，以目录形式组织。

**技能管理流程**：

1. `ensure_skills_initialized()` — 初始化技能目录
2. `resolve_effective_skills()` — 解析当前通道的有效技能
3. `apply_skill_config_env_overrides()` — 应用技能配置环境覆盖
4. `Toolkit.register_agent_skill()` — 注册技能到工具集

---

### 4.15 Backup 备份系统

**核心文件**：[orchestration.py](file:///workspace/src/qwenpaw/backup/orchestration.py)

备份系统支持 Agent 配置、记忆、技能等数据的打包和恢复。

**核心功能**：

| 功能 | 说明 |
|------|------|
| `create()` | 创建备份（支持选择范围） |
| `restore()` | 恢复备份（支持冲突解决） |
| `import_backup()` | 导入外部备份文件 |

---

### 4.16 Token Usage 用量追踪

**核心文件**：`token_usage/manager.py`

追踪每个 Agent 的 Token 使用量。

**核心组件**：

| 组件 | 说明 |
|------|------|
| `TokenUsageManager` | 管理用量记录、定期刷新 |
| `TokenUsageBuffer` | 缓冲写入，减少 I/O |
| `TokenUsageStorage` | JSON 文件持久化 |
| `TokenCountingModelWrapper` | 包装 ChatModel 计算 Token |

---

## 5. Console 前端

**技术栈**：React 18 + TypeScript + Vite + Ant Design + Zustand

**核心页面**：

| 页面 | 路径 | 说明 |
|------|------|------|
| Chat | `/` | 主聊天界面 |
| Login | `/login` | 登录页 |
| Settings - Agents | `/settings/agents` | Agent 管理 |
| Settings - Models | `/settings/models` | 模型配置 |
| Settings - Skills | `/settings/skills` | 技能池 |
| Settings - Backups | `/settings/backups` | 备份管理 |
| Settings - Token Usage | `/settings/token-usage` | 用量统计 |
| Settings - Voice | `/settings/voice` | 语音配置 |
| Settings - Agent Stats | `/settings/agent-stats` | Agent 统计 |

**关键前端组件**：

| 组件 | 说明 |
|------|------|
| `AgentSelector` | Agent 选择器 |
| `ApprovalCard` | 审批卡片 |
| `PlanPanel` | 计划面板 |
| `SkillVisual` | 技能可视化 |
| `MarkdownCopy` | Markdown 复制 |
| `LanguageSwitcher` | 语言切换 |
| `ThemeToggleButton` | 主题切换 |
| `ConsolePollService` | Console 轮询服务 |

**API 客户端层** (`console/src/api/`)：

按模块组织，每个模块包含 `modules/` (API 调用) 和 `types/` (TypeScript 类型)。

---

## 6. 关键类与函数索引

### 后端核心类

| 类 | 模块路径 | 说明 |
|----|----------|------|
| `QwenPawAgent` | `qwenpaw.agents.react_agent` | 主 Agent 类，ReAct 推理 + 工具守卫 |
| `DynamicMultiAgentRunner` | `qwenpaw.app._app` | 多 Agent 请求路由器 |
| `MultiAgentManager` | `qwenpaw.app.multi_agent_manager` | 多 Agent 生命周期管理 |
| `Workspace` | `qwenpaw.app.workspace.workspace` | 单 Agent 工作空间 |
| `BaseChannel` | `qwenpaw.app.channels.base` | 通道抽象基类 |
| `ChannelManager` | `qwenpaw.app.channels.manager` | 通道队列管理 |
| `Provider` | `qwenpaw.providers.provider` | 模型提供者抽象基类 |
| `ProviderManager` | `qwenpaw.providers.provider_manager` | 提供者管理器 (单例) |
| `BaseMemoryManager` | `qwenpaw.agents.memory.base_memory_manager` | 记忆管理器基类 |
| `BaseContextManager` | `qwenpaw.agents.context.base_context_manager` | 上下文管理器基类 |
| `ToolGuardEngine` | `qwenpaw.security.tool_guard.engine` | 工具守卫引擎 |
| `CronManager` | `qwenpaw.app.crons.manager` | 定时任务管理器 |
| `MCPClientManager` | `qwenpaw.app.mcp.manager` | MCP 客户端管理器 |
| `AgentRunner` | `qwenpaw.app.runner.runner` | Agent 请求运行器 |
| `TaskTracker` | `qwenpaw.app.runner.task_tracker` | 任务追踪器 |
| `PluginLoader` | `qwenpaw.plugins.loader` | 插件加载器 |
| `PluginRegistry` | `qwenpaw.plugins.registry` | 插件注册表 |
| `LocalModelManager` | `qwenpaw.local_models.manager` | 本地模型管理器 |
| `TokenUsageManager` | `qwenpaw.token_usage.manager` | Token 用量管理器 |
| `CommandHandler` | `qwenpaw.agents.command_handler` | 系统命令处理器 |
| `LazyGroup` | `qwenpaw.cli.main` | CLI 懒加载命令组 |

### 后端核心函数

| 函数 | 模块路径 | 说明 |
|------|----------|------|
| `cli()` | `qwenpaw.cli.main` | CLI 主入口 |
| `lifespan()` | `qwenpaw.app._app` | FastAPI 应用生命周期 |
| `create_model_and_formatter()` | `qwenpaw.agents.model_factory` | 创建模型和格式化器 |
| `build_system_prompt_from_working_dir()` | `qwenpaw.agents.prompt` | 构建系统提示词 |
| `load_config()` | `qwenpaw.config` | 加载全局配置 |
| `load_agent_config()` | `qwenpaw.config.config` | 加载 Agent 配置 |
| `convert_model_exception()` | `qwenpaw.exceptions` | 转换模型异常为统一格式 |
| `get_available_channels()` | `qwenpaw.config` | 获取可用通道列表 |

### 前端核心组件

| 组件 | 路径 | 说明 |
|------|------|------|
| `App` | `console/src/App.tsx` | 应用根组件 |
| `MainLayout` | `console/src/layouts/MainLayout/` | 主布局 |
| `ChatPage` | `console/src/pages/Chat/` | 聊天页面 |
| `agentStore` | `console/src/stores/agentStore.ts` | Agent 状态管理 |

---

## 7. 依赖关系图

### 后端核心依赖

```
qwenpaw
├── agentscope==1.0.19.post1          # Agent 框架核心
├── agentscope-runtime==1.1.4         # Agent 运行时引擎
├── httpx>=0.27.0                     # HTTP 客户端
├── fastapi (via agentscope-runtime)  # Web 框架
├── uvicorn>=0.40.0                   # ASGI 服务器
├── pydantic (via agentscope)         # 数据验证
├── click (via agentscope)            # CLI 框架
├── apscheduler>=3.11.2,<4            # 定时任务调度
├── playwright>=1.49.0                # 浏览器自动化
├── reme-ai==0.3.1.8                  # 记忆系统
├── google-genai>=1.67.0              # Gemini API
├── python-telegram-bot>=20.0         # Telegram Bot
├── discord-py>=2.3                   # Discord Bot
├── dingtalk-stream>=0.24.3           # 钉钉流式 API
├── lark-oapi>=1.5.3                  # 飞书 API
├── matrix-nio>=0.24.0                # Matrix 协议
├── paho-mqtt>=2.0.0                  # MQTT 协议
├── twilio>=9.10.2                    # Twilio 语音
├── wecom-aibot-python-sdk==1.0.2     # 企业微信
├── pywebview>=4.0                    # 桌面应用
├── cryptography>=43.0.0              # 加密
├── keyring>=25.0.0                   # 密钥存储
├── pyyaml>=6.0                       # YAML 解析
├── shortuuid>=1.0.0                  # 短 UUID
├── json-repair>=0.30.0               # JSON 修复
├── pillow>=10.0.0                    # 图像处理
├── modelscope>=1.35.0                # ModelScope 集成
├── huggingface_hub>=0.20.0           # HuggingFace 集成
├── agent-client-protocol>=0.9.0      # ACP 协议
└── transformers>=4.30.0              # 本地模型 Tokenizer
```

### 前端核心依赖

```
qwenpaw-console
├── react@18                          # UI 框架
├── antd@5                            # UI 组件库
├── @agentscope-ai/chat              # AgentScope 聊天组件
├── @agentscope-ai/design            # AgentScope 设计系统
├── zustand@5                         # 状态管理
├── react-router-dom@7                # 路由
├── i18next + react-i18next           # 国际化
├── @ant-design/x-markdown            # Markdown 渲染
├── @ant-design/plots                 # 图表
├── @dnd-kit                          # 拖拽排序
├── react-window                      # 虚拟滚动
├── ahooks                            # React Hooks 库
└── dayjs                             # 日期处理
```

### 模块间依赖关系

```
CLI → App → MultiAgentManager → Workspace
                              ├── Runner → QwenPawAgent → Toolkit + Memory + Context
                              ├── ChannelManager → BaseChannel[] → UnifiedQueueManager
                              ├── CronManager → CronExecutor
                              └── MCPClientManager → MCPClient[]

App → ProviderManager → Provider[] → ChatModelBase
App → LocalModelManager → LlamaCpp / Ollama
App → PluginLoader → PluginRegistry → Provider / Command / Hook
App → TokenUsageManager → Buffer → Storage

QwenPawAgent → ToolGuardMixin → ToolGuardEngine → Guardian[]
QwenPawAgent → SkillsManager → Skill[] → Toolkit
QwenPawAgent → CommandHandler → System Commands
QwenPawAgent → ACPClient → External Agents
```

---

## 8. 配置与环境变量

### 工作目录

| 路径 | 环境变量 | 默认值 | 说明 |
|------|----------|--------|------|
| 工作目录 | `QWENPAW_WORKING_DIR` | `~/.qwenpaw` | 主数据目录 |
| 密钥目录 | `QWENPAW_SECRET_DIR` | `{WORKING_DIR}.secret` | 敏感数据目录 |
| 备份目录 | `QWENPAW_BACKUP_DIR` | `{WORKING_DIR}.backups` | 备份目录 |

### 核心环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `QWENPAW_LOG_LEVEL` | `info` | 日志级别 |
| `QWENPAW_AUTH_ENABLED` | `false` | 启用 Web 认证 |
| `QWENPAW_RUNNING_IN_CONTAINER` | `false` | 容器内运行标识 |
| `QWENPAW_OPENAPI_DOCS` | `false` | 启用 API 文档 |
| `QWENPAW_CORS_ORIGINS` | `""` | CORS 允许的源 |
| `QWENPAW_TOOL_GUARD_ENABLED` | `true` | 启用工具守卫 |
| `QWENPAW_TOOL_GUARD_APPROVAL_TIMEOUT_SECONDS` | `300` | 审批超时 |
| `DASHSCOPE_API_KEY` | - | DashScope API Key |
| `DASHSCOPE_BASE_URL` | `https://dashscope.aliyuncs.com/compatible-mode/v1` | DashScope 基础 URL |

### LLM 并发控制

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `QWENPAW_LLM_MAX_CONCURRENT` | `10` | 最大并发 LLM 调用 |
| `QWENPAW_LLM_MAX_QPM` | `600` | 每分钟最大查询数 |
| `QWENPAW_LLM_MAX_RETRIES` | `3` | 最大重试次数 |
| `QWENPAW_LLM_BACKOFF_BASE` | `1.0` | 退避基数 |
| `QWENPAW_LLM_BACKOFF_CAP` | `10.0` | 退避上限 |
| `QWENPAW_LLM_RATE_LIMIT_PAUSE` | `5.0` | 429 暂停时间 |
| `QWENPAW_LLM_ACQUIRE_TIMEOUT` | `300.0` | 信号量获取超时 |

### 记忆压缩

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `QWENPAW_MEMORY_COMPACT_KEEP_RECENT` | `3` | 压缩时保留的最近消息数 |
| `QWENPAW_MEMORY_COMPACT_RATIO` | `0.7` | 压缩触发比例 |

> **注意**：所有 `QWENPAW_` 前缀的环境变量都支持 `COPAW_` 前缀的旧版兼容回退。

---

## 9. 项目运行方式

### 方式一：pip 安装

```bash
pip install qwenpaw
qwenpaw init --defaults
qwenpaw app
# 访问 http://127.0.0.1:8088/
```

### 方式二：从源码安装

```bash
git clone https://github.com/agentscope-ai/QwenPaw.git
cd QwenPaw

# 构建前端
cd console && npm ci && npm run build
cd ..

# 复制前端构建产物
mkdir -p src/qwenpaw/console
cp -R console/dist/. src/qwenpaw/console/

# 安装 Python 包
pip install -e .

# 开发模式（含测试和格式化工具）
pip install -e ".[dev,full]"

# 初始化并启动
qwenpaw init --defaults
qwenpaw app
```

### 方式三：Docker

```bash
docker pull agentscope/qwenpaw:latest
docker run -p 127.0.0.1:8088:8088 \
  -v qwenpaw-data:/app/working \
  -v qwenpaw-secrets:/app/working.secret \
  -v qwenpaw-backups:/app/working.backups \
  agentscope/qwenpaw:latest
```

或使用 Docker Compose：

```bash
docker-compose up -d
```

### 方式四：脚本安装

```bash
# macOS / Linux
curl -fsSL https://qwenpaw.agentscope.io/install.sh | bash

# Windows (PowerShell)
irm https://qwenpaw.agentscope.io/install.ps1 | iex
```

### 方式五：桌面应用

从 [GitHub Releases](https://github.com/agentscope-ai/QwenPaw/releases) 下载安装包。

### 运行测试

```bash
# 运行所有测试
make test

# 单元测试
make test-unit

# 契约测试
make test-contract

# 集成测试
make test-integration

# 快速检查
make quick

# 覆盖率报告
make coverage-full

# 通道测试
make test-channel
```

---

## 10. 测试体系

### 测试目录结构

```
tests/
├── unit/                       # 单元测试
│   ├── agents/                 # Agent 相关测试
│   │   ├── context/            # 上下文管理器测试
│   │   ├── hooks/              # 钩子测试
│   │   ├── memory/             # 记忆管理器测试
│   │   ├── tools/              # 工具测试
│   │   └── utils/              # Agent 工具函数测试
│   ├── app/                    # 应用层测试
│   ├── channels/               # 通道测试
│   ├── cli/                    # CLI 测试
│   ├── local_models/           # 本地模型测试
│   ├── providers/              # 提供者测试
│   ├── routers/                # 路由测试
│   ├── security/               # 安全测试
│   ├── token_usage/            # 用量测试
│   ├── utils/                  # 工具测试
│   └── workspace/              # 工作空间测试
├── contract/                   # 契约测试
│   ├── channels/               # 通道接口契约
│   └── providers/              # 提供者接口契约
├── integration/                # 集成测试
└── e2e/                        # 端到端测试
```

### 测试标记

| 标记 | 说明 |
|------|------|
| `@pytest.mark.slow` | 慢速测试 |
| `@pytest.mark.unit` | 单元测试 |
| `@pytest.mark.contract` | 契约测试 |
| `@pytest.mark.integration` | 集成测试 |

### 测试配置

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "function"

[tool.coverage.report]
fail_under = 30
show_missing = true
```
