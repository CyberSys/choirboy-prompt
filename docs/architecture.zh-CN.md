# 架构

仓库结构、载荷组成、投递格式、Kimi 事件路由与 Hermes 协议逐字段说明。
这是插件内部工作原理的地图。

---

## 1. 仓库结构

```text
agent-plugin/
├── prompt.md                 # 智能体工作规则 + 正典自检
├── security-posture.md       # 安全框架：审计、词汇、会话标记
├── security-audit-runbook.md # 可执行的安全审计流程
├── lore.md                   # 共同工作地图（项目、教训、边界）
├── user.md                   # 用户档案
├── research/                 # 29 份决策文档 + Coldcard 完整拆解
│   ├── 01-telegram-stars.md
│   ├── 02-ruble-acquiring.md
│   ├── 03-crypto-payments.md
│   ├── 04-payment-architecture.md
│   ├── 05-comfyui-realism-pipeline.md
│   ├── 06-agent-memory-plugin.md
│   ├── 07-x-reply-farm.md
│   ├── 08-ai-ofm-telegram.md
│   ├── 09-web3-security.md
│   ├── 10-third-party-audit.md
│   ├── 11-coldcard-entropy-heist.md
│   ├── 13-flipper-marauder-wifi-scan.md
│   ├── 14-solo-game-cheats.md
│   ├── 15-*.md … 30-*.md     # 编排与安全能力文档
│   └── coldcard/             # Coldcard 完整拆解：报告、代码、来源
│       ├── report.md
│       ├── yasmarang_reconstruction.py
│       └── sources.md
├── hooks/
│   ├── session-start.sh      # 载荷组装 + claude / plain / hermes 格式
│   ├── artifact-stop.sh      # 恢复匹配的资料包，然后返回 {}
│   ├── kimi-session-start.sh # 准备状态并重置 Kimi 投递去重
│   ├── kimi-user-prompt.sh   # 在首次用户请求时投递 Kimi 上下文
│   ├── kimi-artifact-stop.sh # Kimi 未完成资料的退出码 2 检查
│   └── hooks.json            # Chat/Cowork 自定义插件的 SessionStart + Stop
├── context/
│   └── research-index.md     # 钩子与技能共用的规范研究索引
├── skills/
│   ├── load-context/SKILL.md # 为 Chat/Cowork/Code 生成的内联回退内容
│   └── diagnose/SKILL.md     # 基于证据的投递诊断
├── scripts/
│   ├── artifact-generator.py # 资料请求、校验与时效性清单
│   ├── build-context.py      # 从规范来源重新生成内联技能
│   ├── package-plugin.py     # 构建自定义插件 ZIP
│   ├── test.sh               # 可重复运行的测试套件
│   └── test-opencode-transition.ts # OpenCode 运行时过渡测试（需要 bun）
├── .claude-plugin/
│   └── plugin.json           # 清单（名称、版本、元数据）
├── docs/                     # 本文档
│   ├── authoring.zh-CN.md
│   ├── architecture.md
│   ├── installer.md
│   ├── security.md
│   ├── testing.md
│   └── troubleshooting.md
└── install.sh                # 多运行时安装 / 回滚 / 列表
```

自定义插件包在标准目录中包含 `hooks/hooks.json`。`plugin.json` 有意不写
`hooks` 字段：当前加载器会把对同一文件的显式引用视为重复加载并禁用插件。

## 2. 载荷组成

### 2.1. 组装

`hooks/session-start.sh` 把内容文件严格按以下顺序合并为一段文本：

```text
prompt.md  →  security-posture.md  →  lore.md  →  user.md  →  研究索引
```

文件之间的分隔符是 `\n\n---\n\n`（Markdown 水平线）。末尾追加规范文件
`context/research-index.md`。

顺序不是随意的：

1. **prompt.md** — 如何工作（直奔主题、一行风险说明、自检）。设定模式。
2. **security-posture.md** — 安全框架。放在项目历史之前，以便在涉及 web3 和 Coldcard 之前声明「防御性审计」领域。
3. **lore.md** — 共同工作历史：项目、教训、规则、边界。载荷的核心。
4. **user.md** — 档案：用户是谁、如何布置任务、什么不需要解释。
5. **研究索引** — 决策文档索引。正文**不**预先加载：任务进入相应领域时按需读取。

### 2.2. 大小

| 文件 | 约大小 | 说明 |
|---|---|---|
| prompt.md | 约 14 KB | 工作规则 |
| security-posture.md | 约 7 KB | 安全框架 |
| lore.md | 约 21 KB | 历史 |
| user.md | 约 4 KB | 档案 |
| 研究索引 | 约 6 KB | 共用规范来源 |
| **固定记忆载荷** | **约 31 KB** | 不含内联项目资料 |

研究文档正文不属于固定载荷，只有索引。状态为 `ready` 时，已验证的
项目档案正文会作为既定项目历史加入；准确大小取决于智能体编写的文档。

### 2.3. 版本

插件版本从 `.claude-plugin/plugin.json` 读取。每次投递都带版本、投递方式和
SHA-256 标记；钩子还包含每次运行的 nonce：

```xml
<choirboy-delivery version="1.5.1" delivery="session-start"
  context_sha256="..." nonce="..." />
<choirboy-context>...</choirboy-context>
```

生成的技能使用同一封装和 `delivery="skill"`。因此无需把模型的确认措辞
当作投递证据。

### 2.4. 项目资料生命周期

在规范内容的封装之后，每次自动投递都会追加项目资料生命周期输出。
`pending` 请求使用 `choirboy-project-artifacts` 块，`ready` 记忆使用中性
Markdown。`artifact-generator.py` 读取完整项目历史与每份研究 Markdown
文件，只写入请求元数据并计算生命周期状态。随插件提供的就绪资料包在仍与
规范源一致时补齐缺失文件。已有 INDEX 或项目档案若与该资料包不同，会予以
保留；`finalize` 不会先执行恢复。只有用户明确要求更新 Choirboy 记忆时，
智能体才编写 `INDEX.md` 和项目档案。规范上下文、研究、INDEX 和项目档案
始终使用英文；验证器会拒绝面向模型的西里尔字母或 CJK 内容。
脚本绝不编写项目档案正文。

`finalize` 校验精确链接、必需章节、来源引用与精确项目集合，然后写入
SHA-256 清单。此后每次 `ready` 投递都会重新校验清单、结构、源文件哈希
与资料文件哈希。只有完全有效的快照才会以工作领域目录和完整项目档案正文
投递。`INDEX.md` 与逐文件哈希仅用于校验，不以路径/SHA 封装显示给模型。
缺失、过期、被修改或格式不正确的快照会重新变为 `pending`。
规范项目历史和研究始终优先于派生摘要。

Claude Code 与 Codex 的 `Stop` 在 ready 和 pending 时都返回 `{}`，不用
`decision: block`，也不在 Stop 上返回 `additionalContext`，因为两者都会延续回合。
Kimi Stop 同样以退出码 0 结束。状态不是写文件的命令。项目资料状态不依赖
工作副本，路径优先级为：`CHOIRBOY_ARTIFACTS_DIR` →
`${CLAUDE_PLUGIN_DATA}/project-artifacts` →
`${XDG_DATA_HOME}/choirboy-prompt/project-artifacts` →
`~/.local/share/choirboy-prompt/project-artifacts`。升级时，`install.sh` 会发现
旧注册引用的工作副本内的 `artifacts/` 目录，并把第一份已编写的资料包复制到
空的稳定根目录，不覆盖已有工作。卸载不会删除这些文件。

---

## 3. 投递格式与 Kimi 路由

钩子不在乎是哪个智能体调用了它：调用方通过 `--format` 声明期望的协议。

### 3.1. `claude`（默认）— SessionStart JSON

Claude Code / Codex 契约：钩子输出 JSON。`additionalContext` 是少于 2,000
个字符的简短状态，不包含 `<choirboy-delivery>` 或 `<choirboy-context>`。
Claude Code 将该字段限制为 10,000 个字符，预览只显示前 2,000 个字符，且没有
可提高上限的设置。Codex 会转存过长的钩子输出。完整记忆由 load-context 技能提供。

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "Choirboy memory status: ready\n..."
  }
}
```

通过 `jq`、`python3` 或内置 Bash 编码器编码。最后一种方式让 Claude
格式不依赖外部 JSON 工具，可用于干净的 Claude Desktop 安装。

### 3.2. `plain` — 原始文本

钩子把载荷原样输出到 stdout。每次模型请求时，生成的 OpenCode
适配器都会获取当前载荷，并把它加入模型的系统上下文。OpenCode
在上下文压缩后会重建该上下文，因此精确的就绪项目资料不会随旧
消息历史被移除。Kimi 0.39.x 不消费 `SessionStart` stdout，因此安装器使用
下述独立事件路由。

```bash
bash hooks/session-start.sh --format plain | head -40
```

### 3.3. `hermes` — pre_llm_call 协议

最有趣的契约。Hermes 在会话的**每一轮**都运行 shell 钩子；无条件投递会在
每条消息重复发送固定记忆与全部就绪项目资料。因此钩子：

1. 从 stdin 读取 JSON 载荷；
2. 检查 `.extra.is_first_turn`；
3. 第一轮回复 `{"context": "<载荷>"}`；
4. 之后每一轮回复 `{}`（空回复，不再投递内容）。

```json
// stdin（第一轮）：
{"session_id": "s-123", "extra": {"is_first_turn": true}}
// stdout：
{"context": "<完整载荷>"}

// stdin（第二轮）：
{"session_id": "s-123", "extra": {"is_first_turn": false}}
// stdout：
{}
```

**没有 `is_first_turn` 时的回退。** 如果宿主不报告该标记，钩子使用状态文件
`${TMPDIR:-/tmp}/agent-plugin-hermes-${USER}.state` 中的 `session_id` 日志（保留最近 200 条）：每个 session_id 投递一次，之后保持沉默。

### 3.4. Kimi 0.39.x 事件路由

Kimi 使用四个命令钩子，而不是直接运行
`session-start.sh --format plain`：

1. `startup`/`resume` 的 `SessionStart` 准备项目资料状态并重置内部
   投递指纹；不使用其 stdout。
2. `manual`/`auto` 的同步 `PreCompact` 会在 Kimi 构建压缩上下文之前重置
   该指纹。
3. `UserPromptSubmit` 在每次请求时输出待处理的初始准备请求，或在规范化
   指纹变化时输出就绪记忆；只有相同的就绪资料包保持静默。
4. 项目资料为 `pending` 时，`Stop` 把继续执行请求写入 stderr 并
   以代码 2 退出；验证为 `ready` 后以代码 0 退出。

---

## 4. Hermes 协议逐字段

| stdin 字段 | 类型 | 用途 | 钩子行为 |
|---|---|---|---|
| `extra.is_first_turn` | bool | 会话第一轮？ | `true` → 投递；`false` → `{}` |
| `session_id` | string | 会话标识符 | 用于回退和写入状态文件 |
| （其他） | — | 忽略 | 不影响回复 |

| stdout 字段 | 类型 | 何时 |
|---|---|---|
| `context` | string | 第一轮（或回退中某个 session_id 首次出现时） |
| `{}` | — | 之后所有轮 |

Hermes 配置中的钩子超时——15 秒（由 install.sh 设置）。

---

## 5. 运行时接入点

| 运行时 | 文件 | 机制 | 钩子格式 |
|---|---|---|---|
| Claude Code CLI / Desktop Code | `~/.claude/settings.json` | `SessionStart` + `Stop` | claude / JSON |
| Claude Chat | 自定义插件技能 | 内联 `load-context` | — |
| Claude Cowork | 自定义插件钩子/技能 | 支持时使用钩子，否则回退到技能 | claude / — |
| Codex | `~/.codex/hooks.json` | `SessionStart` + `Stop` | claude / JSON |
| OpenCode | `~/.config/opencode/plugins/agent-plugin.ts` | 模型系统上下文转换 | plain → 每次模型请求的系统上下文 |
| Hermes | `~/.hermes/config.yaml` | `pre_llm_call` + 授权白名单 | hermes |
| Kimi Code 0.39.x | `~/.kimi-code/config.toml` | SessionStart + PreCompact + UserPromptSubmit + Stop | 已变化的载荷 / plain；Stop 退出码 0 |
| Gemini | `~/.gemini/GEMINI.md` | 托管生命周期指令块 | —（自行运行/读取文件） |
| 任意 | `--instructions PATH` | 托管生命周期指令块 | —（自行运行/读取文件） |

最后两种**不是钩子**，而是受管理的指令块：智能体本来就会在启动时读取指令文件，块里告诉它去读插件文件。同样的上下文，多一层间接——智能体必须自己打开文件。

Claude Code 使用由 `install.sh` 或 `install.ps1` 直接注册的钩子，
指向仓库工作副本。要加载完整记忆，请让智能体读取该副本中的
`skills/load-context/SKILL.md`。Chat 使用自定义插件包中生成的内联技能。
Cowork 可以使用包内钩子；钩子输出丢失时，技能仍是回退方式。

OpenCode 适配器由 `install.sh` 生成。它以 15 秒超时运行规范纯文本钩子，
验证投递标记，并把当前载荷加入每个出站请求的系统上下文。转换针对每次
模型请求执行，因此待处理到就绪的转换、源文件变化、进程恢复与上下文压缩
都会获得当前精确快照。钩子、超时或载荷的任何错误都会静默跳过添加内容，
保证聊天继续运行。

Kimi 适配器有意把准备与投递分开。这样不会依赖被丢弃的 `SessionStart`
stdout；`PreCompact` 在上下文压缩前重置投递，而规范化的就绪指纹
允许同一会话接收已变化的资料包，同时避免重复相同资料包。Stop 使用 Kimi
文档规定的退出码 2 契约，而不是 Claude 的 `{"decision":"block"}` 响应格式。

---

## 6. 关键属性

- **手动安装没有副本。** `install.sh` 直接引用项目文件（`$PLUGIN_ROOT/...`），
  因此下一次会话会看到工作副本的修改。
- **双模式投递。** 原生运行时事件会自动投递（`SessionStart`，或 Kimi 的
  `UserPromptSubmit`）；没有这些事件的界面以内联技能 加载同一规范上下文。
- **项目资料由智能体编写。** 生命周期代码只输出确定性请求、校验结果，
  并用 SHA-256 跟踪时效性。就绪钩子会内联完整的已验证快照，
  绝不会用文件路径指针替代模型可见记忆。
- **项目资料状态可跨工作副本升级保留。** 手动安装使用稳定的用户数据目录；
  只有稳定目标中尚无已编写内容时，安装器才迁移旧工作副本内的资料包。
  内部迁移记录会将中断的多文件复制恢复为同一个托管资料包。
- **可观测执行。** 宿主提供 `CLAUDE_PLUGIN_DATA` 时，Bash 钩子只把不敏感的
  投递元数据写入 `${CLAUDE_PLUGIN_DATA}/latest-delivery.log`，不会记录项目历史。
- **OpenCode 记忆可跨上下文压缩保留。** 当前规范载荷会在每次模型请求的
  系统上下文中重新构建，而不是根据完整持久化消息历史推断。
- **依赖极简。** `claude` 与 `plain` 投递可只依赖 Bash，但自动项目资料
  生命周期 和 `install.sh` 需要 `python3`；`hermes` 需要 `jq` 或 `python3`
  解析 stdin。
