# Troubleshooting

Diagnose registration, execution, delivery, and model behavior separately. A
Claude acknowledgement is not evidence that `SessionStart` ran.

## Surface matrix

| Surface | Install | Primary delivery | Fallback / limit |
|---|---|---|---|
| Claude Code CLI | `install.sh --target claude` or `install.ps1 --target claude` | automatic `SessionStart` status | ask the agent to read local `skills/load-context/SKILL.md` |
| Desktop Code local | direct hooks in `settings.json` | automatic `SessionStart` status | ask the agent to read local `skills/load-context/SKILL.md` |
| Claude Chat | custom plugin | load-context skill | Chat does not execute hooks |
| Claude Cowork | custom plugin | hook where available | load-context skill; hook runtime may drop output |
| Desktop Code SSH / Cloud / WSL | register hooks in the execution environment | `SessionStart` status when hooks are supported | the checkout and hook runtime must be available there |

## Four-stage diagnosis

1. **Registration:** `./install.sh --list` reports Claude as `installed`, and the
   relevant `settings.json` points to the current checkout. For Chat/Cowork,
   check that the custom plugin is enabled.
2. **Execution:** the session receives the short “Choirboy memory status”.
   When the host sets `CLAUDE_PLUGIN_DATA`, the Bash hook also records version,
   hash, and nonce in `${CLAUDE_PLUGIN_DATA}/latest-delivery.log`.
3. **Delivery:** ask the Claude Code agent to read local
   `skills/load-context/SKILL.md`, then `skills/diagnose/SKILL.md` to inspect
   the `choirboy-delivery` marker next to `choirboy-context`. In Chat/Cowork,
   select the **load-context** and **diagnose** skills.
4. **Behavior:** only after the first three stages pass should model acceptance,
   conflicting instructions, or memory be investigated.

Marker delivery values:

- `session-start` — a full plain payload (Kimi, OpenCode, Hermes) reached the model. Claude Code and Codex SessionStart do not emit this marker; they emit a short status;
- `skill` — the load-context fallback supplied the lore. A short status is not this marker.

## Common failures

### Claude Code receives no status

Register the hooks from the repository checkout:

```bash
./install.sh --target claude
```

On Windows:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\install.ps1 --target claude
```

Start a new session. If the checkout moved, run the installer again from its new
location. Use `--settings` or `--project` when the session reads a different
settings file; see [Installer](installer.en.md).

### Plugin is installed in Chat but nothing changes

Chat does not execute `SessionStart`. Select the **load-context** skill or ask
Claude to load Choirboy context.

### Cowork shows the plugin but no marker

Invoke **load-context**. Cowork hook execution is not deterministic across
current builds; the skill is the supported fallback.

### Windows hook error

Manual installation uses PowerShell hooks and needs `pwsh`. The Chat/Cowork
custom plugin uses Bash hooks: `${CLAUDE_PLUGIN_ROOT}` is a single argument,
but automatic delivery needs `bash` on `PATH`. Install Git for Windows or use
the skill fallback. The skill itself has no Bash, jq, Python, or Node dependency.

### `pwsh` is not recognized on Windows

The old command `pwsh -NoProfile -File .\install.ps1` needs PowerShell 7
preinstalled. Run `powershell -NoProfile -ExecutionPolicy Bypass -File .\install.ps1`
instead: it works in Windows PowerShell 5.1 and installs PowerShell 7
automatically. If hooks report that `pwsh` is missing after the install,
restart the terminal and the agent app so they pick up the new `PATH`.

### Cloud, SSH, or WSL

Keep the checkout in the environment where Claude Code actually runs, and
register its hooks there. Local host settings and paths do not configure a
separate remote runtime. In WSL, use `./install.sh --target claude` from the
Linux checkout. If the runtime cannot execute hooks, ask the agent to read
`skills/load-context/SKILL.md` from that checkout.

## Maintainer checks

```bash
python3 scripts/build-context.py --check
bash scripts/test.sh
python3 scripts/package-plugin.py
```

The suite validates manifests, skill freshness, hook JSON, the dependency-free
Bash encoder, installer idempotency/rollback, diagnostics, and ZIP contents.
