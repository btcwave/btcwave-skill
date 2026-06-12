# btcwave-skill

Claude Code skill for Bitcoin Wave — guides agent-assisted node setup and management.

## What it does

This skill turns a base Claude Code agent into a Bitcoin node operator. When a user says "set up my Bitcoin node" and provides a license key, the agent:

1. Installs the `btcwave` CLI
2. Detects hardware and existing nodes
3. Generates a privacy-focused configuration
4. Downloads and verifies Bitcoin Knots binaries
5. Manages the initial blockchain sync
6. Guides the human-only seed ceremony
7. Installs the full stack (Lightning, Fulcrum, BTCPay, dashboard)
8. Provides ongoing monitoring and troubleshooting

## Installation

Copy `SKILL.md` to your Claude Code skills directory:

```sh
# If using Claude Code's built-in skills
cp SKILL.md ~/.claude/skills/btcwave/SKILL.md

# Or install via the btcwave CLI
btcwave skill install
```

## How it works

The skill wraps the `btcwave` CLI — a deterministic, resumable installer written in Go. The agent doesn't make installation decisions; it operates the CLI and translates its structured output into plain language for the user. Human checkpoints (seed ceremony, channel funding, upgrade approval) are enforced by the skill's safety rules.

## Safety

- Seeds are human-only — the agent never sees, stores, or processes wallet seed words
- Fund movements require explicit user approval
- Binary installations are checksum-verified before proceeding
- Config changes are explained before being applied

## Related repos

- [btcwave-cli](https://github.com/btcwave/btcwave-cli) — the installer/manager the skill operates
- [btcwave-node](https://github.com/btcwave/btcwave-node) — configuration profile
- [btcwave-mcp](https://github.com/btcwave/btcwave-mcp) — MCP server for ongoing agent management

## License

MIT
