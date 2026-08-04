[한국어](README.md) | **English**

# 🚀 linux_diagnose

**Read-only Linux OS diagnostic tool for Azure VM / Arc-enabled servers.**

Part of the same diagnostic tool suite as `pg_diagnose` / `aks_diagnose` / `adx_diagnose` / `eh_diagnose` / `agw_diagnose` / `svcmap_diagnose` / `windows_diagnose` — strict read-only, JSON/HTML/table output, `health_score`/`summary`/`recommended_actions`.

It reads **already-collected Azure Monitor Agent telemetry** (Heartbeat / Perf / Syslog) via read-only KQL against a Log Analytics workspace — it never issues remote commands (no SSH/Run Command) to the target server.

## Checks

| Category | Signal |
|---|---|
| `connectivity` | Heartbeat gap (agent connectivity) |
| `cpu` | Processor / % Processor Time |
| `memory` | Memory / % Used Memory |
| `disk` | Logical Disk / % Used Space (worst mount) |
| `syslog` | err/crit/emerg/alert log count |
| `stability` | OOM Killer detection (`Out of memory` / `oom-kill`) |
| `patch` | Missing security updates (Update Management) |
| `control_plane` | VM power state/size/OS (optional, via `--resource-id`) |

## Install

```bash
pip install -r requirements.txt
```

## Usage

```bash
python linux_diagnose.py --computer linux-app01 --workspace-id <workspace-guid> --format table
python linux_diagnose.py --computer linux-app01 --workspace-id <guid> --format json --exit-code
python linux_diagnose.py --demo --format html -o linux_demo.html
```

## Required inputs

- `--computer`: value of the Log Analytics `Computer` column.
- `--workspace-id`: Log Analytics workspace GUID.
- `--resource-id` (optional): VM ARM resource id, for power state/size/OS control-plane info.

If `--workspace-id`/`--resource-id` are omitted, the JSON output's top-level `needs_input` explains what's missing and how to discover it (Resource Graph hint + a ready-to-run reinvoke example) — the same "autonomous discovery → reinvoke" pattern used by the other tools in this suite.

## RBAC

- Log Analytics workspace: `Log Analytics Reader`
- VM (optional): `Reader`

## Limitations

- Arc-enabled servers (Microsoft.HybridCompute) are not supported for control-plane lookups yet (Azure VM only). Data-plane checks (Perf/Syslog/Heartbeat) work the same for Arc servers.
- `Syslog`/`Update` collection requires the data collection rule (DCR) to include the relevant facility/severity; some distros need rsyslog/journald forwarding configured.
