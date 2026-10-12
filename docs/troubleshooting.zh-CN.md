# 故障排除

请分别诊断注册、执行、投递和模型行为。Claude 的“我已阅读上下文”并不能证明
`SessionStart` 已运行。

## 界面矩阵

| 界面 | 安装 | 主要投递 | 回退 / 限制 |
|---|---|---|---|
| Claude Code CLI | `install.sh --target claude` 或 `install.ps1 --target claude` | 自动 `SessionStart` 状态 | 要求智能体读取本地 `skills/load-context/SKILL.md` |
| Desktop Code local | 在 `settings.json` 中直接注册钩子 | 自动 `SessionStart` 状态 | 要求智能体读取本地 `skills/load-context/SKILL.md` |
| Claude Chat | 自定义插件 | load-context 技能 | Chat 不执行钩子 |
| Claude Cowork | 自定义插件 | 支持时使用钩子 | load-context 技能；钩子运行环境可能丢弃输出 |
| Desktop Code SSH / Cloud / WSL | 在执行环境中注册钩子 | 支持钩子时输出 `SessionStart` 状态 | 该环境必须能访问工作副本和钩子运行时 |

## 四阶段诊断

1. **注册：**`./install.sh --list` 显示 Claude 的状态为 `installed`，
   对应的 `settings.json` 指向当前工作副本。对于 Chat/Cowork，
   确认自定义插件已启用。
2. **执行：**会话收到简短的“Choirboy memory status”。
   如果宿主设置了 `CLAUDE_PLUGIN_DATA`，Bash 钩子还会将版本、哈希和 nonce
   记录到 `${CLAUDE_PLUGIN_DATA}/latest-delivery.log`。
3. **投递：**要求 Claude Code 智能体读取本地
   `skills/load-context/SKILL.md`，然后读取 `skills/diagnose/SKILL.md`，
   检查 `choirboy-context` 旁的 `choirboy-delivery` 标记。
   在 Chat/Cowork 中选择 **load-context** 和 **diagnose** 技能。
4. **行为：**只有前三步通过后，才调查模型接受、指令冲突或记忆。

`delivery` 值：

- `session-start` — 完整的纯文本载荷（Kimi、OpenCode、Hermes）已到达模型。Claude Code 与 Codex 的 SessionStart 不输出此标记，只输出简短状态；
- `skill` — load-context 回退技能提供了项目记忆。简短状态不是此标记。

## 常见故障

### Claude Code 没有收到状态

从仓库工作副本注册钩子：

```bash
./install.sh --target claude
```

在 Windows 上：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\install.ps1 --target claude
```

开始新会话。如果移动了工作副本，请从新位置重新运行安装器。
如果会话读取其他设置文件，请使用 `--settings` 或 `--project`；
参见[安装器](installer.zh-CN.md)。

### Chat 已安装插件，但行为没有变化

Chat 不执行 `SessionStart`。请选择 **load-context** 技能，或要求 Claude
加载 Choirboy 上下文。

### Cowork 显示插件，但没有标记

调用 **load-context**。当前 Cowork 版本的钩子执行并不确定，技能是受支持的回退方式。

### Windows 钩子错误

手动安装使用 PowerShell 钩子，需要 `pwsh`。Chat/Cowork 自定义插件使用
Bash 钩子：`${CLAUDE_PLUGIN_ROOT}` 作为单独参数传递，但自动投递需要
`PATH` 中存在 `bash`。安装 Git for Windows，或使用技能。
技能本身不依赖 Bash、jq、Python 或 Node。

### Windows 提示无法识别 `pwsh`

旧命令 `pwsh -NoProfile -File .\install.ps1` 需要预先安装 PowerShell 7。
请改用 `powershell -NoProfile -ExecutionPolicy Bypass -File .\install.ps1`：
它可在 Windows PowerShell 5.1 中运行，并会自动安装 PowerShell 7。如果安装后
钩子提示找不到 `pwsh`，请重启终端和智能体应用，让它们读取新的 `PATH`。

### Cloud、SSH 或 WSL

将工作副本放在 Claude Code 实际运行的环境中，并在该环境注册钩子。
本地主机的设置和路径不会配置独立的远程运行环境。在 WSL 中，从 Linux
工作副本运行 `./install.sh --target claude`。如果运行环境不能执行钩子，
请要求智能体读取该工作副本中的 `skills/load-context/SKILL.md`。

## 维护者检查

```bash
python3 scripts/build-context.py --check
bash scripts/test.sh
python3 scripts/package-plugin.py
```

测试套件验证清单、技能时效性、钩子 JSON、无外部依赖的 Bash 编码器、
安装器幂等性与回滚、诊断和 ZIP 内容。
