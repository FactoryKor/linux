[한국어](README.md) | **English**

# 🚀 linux_diagnose

**Read-only Linux OS diagnostic tool — manage on-prem/Azure/AWS/GCP servers uniformly via Azure Arc.**

Part of the same diagnostic tool suite as `pg_diagnose` / `aks_diagnose` / `adx_diagnose` / `eh_diagnose` / `agw_diagnose` / `svcmap_diagnose` / `windows_diagnose` — strict read-only, JSON/HTML/table output, `health_score`/`summary`/`recommended_actions`.

## Recommended path — Azure Arc

This tool is designed by default to query **already-collected Azure Monitor Agent telemetry** (Log Analytics) via read-only KQL. On-prem/AWS/GCP servers can be onboarded as [Azure Arc-enabled servers](https://learn.microsoft.com/azure/azure-arc/servers/overview) (`azcmagent connect`), after which Azure Monitor Agent installs and reports telemetry just like a native Azure VM — **Arc onboarding is the prerequisite for this tool's officially supported path**.

```bash
python linux_diagnose.py --computer linux-app01 --workspace-id <workspace-guid> \
  --resource-id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.HybridCompute/machines/linux-app01
```

`--resource-id` accepts either an Arc machine (`Microsoft.HybridCompute/machines`) or a native Azure VM (`Microsoft.Compute/virtualMachines`) — control-plane info (Arc: agent connectivity status; Azure VM: power state) is fetched automatically for either type.

## Hidden option — when Azure Arc can't be connected

For exceptional environments where Azure Arc cannot be onboarded (firewall/policy constraints), a **direct SSH collection mode also exists internally**. It is not advertised via `--help` (not the recommended path), but works exactly as documented when specified explicitly:

```bash
# Not shown in --help, but works when specified explicitly
$env:LINUX_DIAGNOSE_SSH_PASSWORD = "..."
python linux_diagnose.py --source direct --host 10.0.5.20 --ssh-user diag_reader --format table
```

| | `azure-monitor` (default, officially supported) | `direct` (hidden, for non-Arc environments) |
|---|---|---|
| Reads | Already-collected Azure Monitor Agent telemetry via KQL | Live procfs/command output over SSH |
| Requires | **Azure Arc onboarding** (or Azure VM) + AMA + Log Analytics workspace | SSH reachable |
| Visibility | Shown in `--help` | Hidden from `--help` (documented here only) |

## Checks

| Category | Signal |
|---|---|
| `connectivity` | Heartbeat gap (azure-monitor) or SSH connection success (direct) |
| `cpu` | Processor / % Processor Time (or `/proc/stat` delta in direct mode) |
| `memory` | Memory / % Used Memory (or `/proc/meminfo` in direct mode) |
| `disk` | Logical Disk / % Used Space (worst mount, or `df -P` in direct mode) |
| `syslog` | err/crit/emerg/alert log count (or `journalctl`/`dmesg` in direct mode) |
| `stability` | OOM Killer detection (`Out of memory` / `oom-kill`) |
| `patch` | Missing security updates (Update Management, or apt/yum/dnf in direct mode) |
| `control_plane` | VM power state/size/OS (optional, via `--resource-id`, either mode) |

## Install

```bash
pip install -r requirements.txt
```

`--source direct` uses `paramiko` (pure Python SSH client, no external SSH binary needed).

## Usage

```bash
# azure-monitor mode
python linux_diagnose.py --computer linux-app01 --workspace-id <workspace-guid> --format table

# direct mode (no Azure Monitor needed)
$env:LINUX_DIAGNOSE_SSH_PASSWORD = "..."
python linux_diagnose.py --source direct --host 10.0.5.20 --ssh-user diag_reader --format table

python linux_diagnose.py --demo --format html -o linux_demo.html
```

## Required inputs

## Usage

```bash
# azure-monitor mode
python linux_diagnose.py --computer linux-app01 --workspace-id <workspace-guid> --format table

# hidden direct mode (no Azure Arc/Azure Monitor needed)
$env:LINUX_DIAGNOSE_SSH_PASSWORD = "..."
python linux_diagnose.py --source direct --host 10.0.5.20 --ssh-user diag_reader --format table

python linux_diagnose.py --demo --format html -o linux_demo.html
```

## Time window & history

- `--hours N` (default): relative window, e.g. `--hours 24` (1 day), `--hours 168` (1 week).
- `--start-time`/`--end-time` (ISO 8601): absolute date range, azure-monitor mode only.
- `--save-snapshot <path>`: append this run's JSON result to a JSON Lines file — combine with a scheduled task/cron to build your own trend history over time (works in either mode; direct mode cannot use `--start-time`/`--end-time` since SSH only reflects the target's current state).

## Required inputs

- azure-monitor mode: `--computer` (Log Analytics `Computer` column) + `--workspace-id`.
- direct mode (hidden): `--host` + `--ssh-user` (password via env var, or `--ssh-key-file`).
- `--resource-id` (optional, either mode): Azure VM or Arc machine ARM resource id, for control-plane info.

If required values are omitted, the JSON output's top-level `needs_input` explains what's missing and how to discover it (Resource Graph hint + a ready-to-run reinvoke example) — the same "autonomous discovery → reinvoke" pattern used by the other tools in this suite.

## RBAC

- Log Analytics workspace: `Log Analytics Reader`
- VM/Arc machine (optional): `Reader`

## Limitations

- `Syslog`/`Update` collection requires the data collection rule (DCR) to include the relevant facility/severity; some distros need rsyslog/journald forwarding configured.
- Direct mode's patch check (yum/dnf repo metadata refresh) can be slow and adds repo/network load — skip it with `--skip-patch-check` when running against production servers.
