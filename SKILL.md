# Bitcoin Wave — Node Setup & Management Skill

You are assisting a user in setting up and managing a Bitcoin Wave node. Bitcoin Wave is a self-custodial Bitcoin banking stack that uses stock Bitcoin Knots with a curated configuration profile. You drive the `btcwave` CLI — the deterministic installer — and guide the user through decisions that require human judgment.

## Prerequisites

The `btcwave` CLI must be installed. If not found, install it:

```sh
# macOS / Linux (downloads the latest release binary)
curl -fsSL https://github.com/btcwave/btcwave-cli/releases/latest/download/install.sh | sh

# Or build from source
git clone https://github.com/btcwave/btcwave-cli.git && cd btcwave-cli && go build -o /usr/local/bin/btcwave ./cmd/btcwave/
```

## Getting a License Key

Purchase at [btcwave.app](https://btcwave.app) — $49 one-time. After checkout you receive a key in the format `WAVE-FULL-XXXX-XXXX`. The key redeems once at install; after that, your node never contacts btcwave.app again.

## Setup Flow

When the user says "set up my node" or provides a license key, follow this sequence. **Do not skip steps. Do not proceed past human checkpoints without explicit confirmation.**

### 1. Validate the key
The user provides a license key like `WAVE-FULL-7K3M-XXXX`. Confirm you have it before proceeding.

### 2. Determine the target
Ask: "Where should I install the node?" Options:
- **This machine** — local install (development/testing)
- **A device on your network** — SSH to a Raspberry Pi, mini-PC, or server on the LAN. You'll need the hostname/IP and SSH credentials.
- **A cloud VPS** — BYO server with SSH access. Minimum: 2 vCPU, 4GB RAM, 1TB+ NVMe SSD, Debian/Ubuntu. See "VPS Deployment" below.

For LAN devices (the most common path), you'll need:
- IP address or hostname
- SSH username and password (or key-based auth)
- The device should be running a Debian-based Linux (Raspberry Pi OS, Ubuntu, Debian)

### 3. Run setup
```sh
btcwave setup --key WAVE-FULL-7K3M-XXXX --target <host> --json
```

Parse the JSON output at each phase and report progress to the user in plain language:
- **Detect phase**: Report hardware specs. Flag if disk is too small (<700GB for full node).
- **Existing node**: If a Bitcoin Core or Knots node is found, explain the migration path — their chain data is preserved, no resync needed.
- **Config phase**: Explain what the generated config does (Tor privacy, spam filtering, ZMQ for Lightning).
- **Install phase**: Report download progress and checksum verification. This is the trust moment — emphasise that binaries are verified against official checksums.

### 4. Initial Block Download (IBD)
After install, the node begins syncing the blockchain. This takes **4–20+ hours** depending on hardware and network. You will NOT be present for the entire sync.

Tell the user:
> "Your node is syncing. This takes several hours — you don't need to do anything. Run `btcwave status` anytime to check progress, or I'll check for you when you ask."

### 5. Seed Ceremony (HUMAN CHECKPOINT)
**CRITICAL: This step is for the human only. You must NOT see, store, process, or remember the seed words.**

When the stack reaches the wallet creation phase, say exactly:
> "It's time to create your wallet seed. This is the ONE step I can't help with — and shouldn't. Bitcoin Wave will display 24 words on your screen. Write them on paper. Not in a file, not in a note, not in a screenshot. Paper only, stored somewhere safe. I'll look away now — tell me when you're done."

Then wait. Do not parse, read, or request the seed output. When the user confirms they've recorded it, proceed.

### 6. Stack Completion
After the seed ceremony, the remaining stack installs automatically:
- Lightning (LND) — payment channels
- Fulcrum — block explorer / wallet index
- BTCPay — payment processing
- Dashboard — web UI at `http://<target>:8380`
- MCP server — your ongoing management interface

Report each component as it completes.

## VPS Deployment

For users who want a cloud-hosted node instead of local hardware:

### Recommended providers
Any provider with NVMe SSD and unmetered/high bandwidth. Vultr, Hetzner, and OVH are common choices.

### Minimum specs
| Resource | Minimum | Recommended |
|---|---|---|
| CPU | 2 vCPU | 4 vCPU |
| RAM | 4 GB | 8 GB |
| Storage | 1 TB NVMe SSD | 2 TB NVMe SSD |
| Bandwidth | 2 TB/month | Unmetered |
| OS | Debian 12 or Ubuntu 24.04 | Same |

### VPS setup differs from local in two ways:
1. **No LAN discovery** — the user provides SSH credentials directly.
2. **Firewall** — the installer opens only the ports needed: 8333 (Bitcoin P2P), 9735 (Lightning), and 8380 (dashboard, optionally behind a reverse proxy with auth).

The CLI handles the rest identically — same config, same install, same verification.

## MCP Server Integration

After setup, the MCP server provides agent-native access to the node. It runs as a stdio JSON-RPC server on the node machine.

### Starting the MCP server
```sh
btcwave-mcp --rpcuser <user> --rpcpassword <pass>
# Or with cookie auth (auto-detected):
btcwave-mcp --cookie /home/bitcoin/.bitcoin/.cookie
```

### Available tools

| Tool | Purpose | Returns |
|---|---|---|
| `btcwave_node_status` | Full node status | height, sync %, peers, mempool, hashrate, chain, version |
| `btcwave_get_block` | Get block by height or hash | block header, tx count, size, timestamp |
| `btcwave_get_transaction` | Get transaction details | inputs, outputs, confirmations, fee |
| `btcwave_estimate_fee` | Fee estimate for confirmation target | sat/vB for target block count (default: 6) |
| `btcwave_get_peers` | Connected peer list | address, version, latency, direction for each peer |
| `btcwave_get_mempool` | Mempool summary | tx count, size in MB, min fee rate |
| `btcwave_doctor` | Run 5-point health check | sync, peers, mempool, version, disk — pass/warn/fail each |

### MCP config for Claude Code
Add to your MCP config (`.mcp.json` or Claude Code settings):
```json
{
  "mcpServers": {
    "btcwave": {
      "command": "ssh",
      "args": ["user@node-ip", "/usr/local/bin/btcwave-mcp", "--rpcuser", "btcwave", "--rpcpassword", "YOUR_RPC_PASSWORD"]
    }
  }
}
```

This lets your agent call `btcwave_node_status`, `btcwave_doctor`, etc. directly as MCP tools.

## Autonomy Ladder

Bitcoin Wave implements graduated agent permissions via the vault system (`btcwave-vault`). The user chooses their comfort level:

| Level | Name | What the agent can do | Credential scope |
|---|---|---|---|
| L0 | Read | Check status, read blockchain data | ReadOnly macaroon |
| L1 | Receive | Generate invoices, receive payments | Invoice macaroon |
| L2 | Propose & Approve | Draft transactions, user approves each one | PayOnly macaroon with spending limits |
| L3 | Autonomous | Send within daily/per-tx limits set by user | ChannelAdmin macaroon with caps |

### Spending limits (L2 and L3)
```json
{
  "per_tx_limit_sats": 100000,
  "daily_limit_sats": 500000
}
```

The agent MUST check `vault.CheckSpend(amount)` before any outgoing payment. The vault enforces limits regardless of what the agent requests.

**Default to L0.** Only escalate when the user explicitly asks. When proposing an escalation, explain what new capabilities it grants and what risks it introduces.

## Ongoing Management

After setup, use these commands for day-to-day management:

### Check node health
```sh
btcwave status --host <target-ip> --rpcuser <user> --rpcpassword <pass> --json
```
Report: sync status, peer count, mempool size, hashrate, disk usage.

### Run diagnostics
```sh
btcwave doctor --host <target-ip> --rpcuser <user> --rpcpassword <pass> --json
```
Report each check result. If any fail, explain what's wrong and suggest fixes.

### Troubleshooting Guide

| Symptom | Likely cause | Fix |
|---|---|---|
| 0 peers | Tor not running or firewall blocking | Check `systemctl status tor`, verify port 9050 |
| Sync stuck | Low disk space or network issues | Run `btcwave doctor`, check `df -h` |
| RPC timeout | Node stopped or wrong credentials | Check `systemctl status bitcoind`, verify cookie/rpcauth |
| High memory | Too many connections | Reduce `maxconnections` in bitcoin.conf |
| Dashboard blank | Dashboard can't reach RPC | Restart dashboard, check cookie auth |
| LND won't start | Seed not created or wrong permissions | Check `/home/bitcoin/.lnd/`, run seed ceremony if missing |
| Fulcrum slow | Initial index build in progress | Normal — first index takes 6-12 hours on Pi 5 |
| BTCPay unreachable | Docker not running | `sudo systemctl restart docker`, then `docker compose up -d` |

## Hardware Buying Guide

If the user needs hardware recommendations:

| Budget | Recommendation | Notes |
|---|---|---|
| $100–150 | Raspberry Pi 5 (8GB) + 1TB NVMe | The Bitcoin Wave reference platform |
| $150–250 | Raspberry Pi 5 (16GB) + 2TB NVMe | Room for growth, better for Lightning |
| $300–500 | Intel N100 mini-PC + 2TB NVMe | More headroom, faster IBD |
| $500+ | Used Dell/HP micro desktop + 2TB SSD | Best performance per dollar |

All options need: USB-C power, Ethernet (preferred over WiFi), and a reliable SSD (not an SD card for chain storage).

## Safety Rules

1. **Never touch seeds.** Never read, store, log, or process wallet seed words.
2. **Never send funds without explicit approval.** Even checking balances is fine; moving sats requires the user to confirm amount and destination.
3. **Always verify before installing.** Report checksum verification results before proceeding with binary installation.
4. **Explain before changing.** Before modifying config, explain what changes and why. For migration from Core, explain what's being backed up.
5. **Prefer --json mode.** Always use `--json` for CLI calls so you get structured output. Present results in plain language to the user.
6. **Respect the autonomy level.** Never attempt operations beyond the user's configured autonomy level. If a task requires escalation, ask first.
7. **One node per key.** Each license key redeems once. Do not attempt to reuse a redeemed key on a different machine.

## Resources

- [GitHub — all repos](https://github.com/btcwave)
- [Buy a license](https://btcwave.app)
- [Dashboard API](http://<node-ip>:8380/api/status) — JSON endpoint for node metrics
