[한국어](README.md) | **English**

# 🚀 linux_diagnose

**Read-only Linux OS diagnostic tool — manage on-prem/Azure/AWS/GCP servers uniformly via Azure Arc**

Diagnoses CPU/memory/disk/agent-connectivity/Syslog/OOM-Killer/patch status on a target Linux
server. Part of the same diagnostic tool suite as `pg_diagnose` / `aks_diagnose` / `adx_diagnose` /
`eh_diagnose` / `agw_diagnose` / `svcmap_diagnose` / `windows_diagnose` — following the same design
philosophy (strict read-only, JSON/HTML/table output, `health_score`/`summary`/`recommended_actions`).

## ✅ Recommended usage — Azure Arc onboarding

By default, this tool is designed to query **already-collected Azure Monitor Agent telemetry**
(Log Analytics) via read-only KQL. On-prem/AWS/GCP servers can be onboarded as
[Azure Arc-enabled servers](https://learn.microsoft.com/azure/azure-arc/servers/overview)
(`azcmagent connect`), after which they get Azure Monitor Agent installed and can send telemetry to a
Log Analytics workspace just like a native Azure VM — in other words, **Arc onboarding is the
prerequisite for this tool's officially supported path**.

```bash
python linux_diagnose.py --computer linux-app01 --workspace-id <workspace-guid> \
  --resource-id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.HybridCompute/machines/linux-app01
```

Passing either an Arc machine (`Microsoft.HybridCompute/machines`) or a native Azure VM
(`Microsoft.Compute/virtualMachines`) resource ID to `--resource-id` automatically fetches
control-plane info for both types (Arc: agent connectivity status; Azure VM: power state).

## 🔒 Hidden option — when Azure Arc can't be onboarded

For exceptional environments (firewall/policy/organizational constraints) where Azure Arc cannot be
onboarded, **a `--source direct` option that connects and collects directly via SSH is also
supported internally**. This path is not recommended for general users and therefore doesn't appear
in `--help`, but it works exactly as documented when specified explicitly:

```bash
# Not shown in --help, but works when specified explicitly
$env:LINUX_DIAGNOSE_SSH_PASSWORD = "..."
python linux_diagnose.py --source direct --host 10.0.5.20 --ssh-user diag_reader --format table
```

| | `--source azure-monitor` (default, officially supported) | `--source direct` (hidden option, for non-Arc environments) |
|---|---|---|
| What it reads | Already-collected Azure Monitor Agent telemetry (Log Analytics) | Live results of procfs reads/commands run over SSH |
| Requirements | **Azure Arc onboarding** (or Azure VM) + Azure Monitor Agent + Log Analytics workspace | SSH reachable (user/password or key) |
| Visibility | Shown in `--help` (recommended path) | Hidden from `--help` (documented here only) |
| Server impact | None (query only) | None (only read-only commands; no new agent installed via SSH) |

For the full option list (`--ssh-user`, `--ssh-password-env`, `--ssh-key-file`, `--ssh-known-hosts`,
`--ssh-insecure-auto-add-host-key`, `--skip-patch-check`, etc.), see "Data sources" below.

- 👉 Diagnoses CPU / memory / disk usage / agent (or SSH) connectivity / Syslog errors / OOM Killer
  events / patch compliance.
- 👉 Supports Azure SRE Agent and MCP Tool integration.
- 👉 When `--resource-id` is given (Azure VM or Arc machine, common to both collection modes),
  control-plane info is also fetched.

Checks:

| 📋 Category | 📋 Category |
|---|---|
| Connectivity (Heartbeat or SSH connection) | CPU usage |
| Memory usage | Disk usage (per mount) |
| Log error count (Syslog) | OOM Killer (out-of-memory process kill) events |
| Missing security updates | VM/Arc machine connectivity·power state, size, OS version (optional, ARM) |
| OS EOL lifecycle + installed EOL package detection (direct mode only) | Recommended management tools (fail2ban/auditd/Azure Arc, etc.) installed? (direct mode only) |

---

## Data sources

### `--source azure-monitor` (all read-only KQL queries / ARM lookups)

| # | Source | What it reads | Table/query |
|---|------|------------|------|
| 1 | **Heartbeat** | Agent connectivity (last heartbeat gap) | `Heartbeat` |
| 2 | **Perf** | CPU/memory/disk usage (Linux AMA counter schema) | `Perf` (`Processor`, `Memory`, `Logical Disk`) |
| 3 | **Syslog** | Error count by severity, OOM Killer detection | `Syslog` (SeverityLevel, `Out of memory`/`oom-kill` messages) |
| 4 | **Update** | Patch compliance (when Update Management is enabled) | `Update` |
| 5 | **ARM (optional)** | VM or Arc machine connectivity/power state/size/OS version (control plane) | `azure-mgmt-compute` (Azure VM) or `azure-mgmt-hybridcompute` (Arc) |

`--computer` (the Log Analytics `Computer` column value) and `--workspace-id` (Log Analytics
workspace GUID) are **required**.

```text
   --computer + --workspace-id ──▶ Heartbeat/Perf/Syslog/Update KQL (data plane)
   --resource-id (optional, Azure VM or Arc machine) ──▶ azure-mgmt-compute or
                                        azure-mgmt-hybridcompute instanceView (control plane)
                                        └─▶ merge → OS diagnostic report
```

> [!NOTE]
> Linux Perf counter object names differ from Windows: `Logical Disk` (with a space, vs. Windows'
> `LogicalDisk`), `% Used Memory`/`% Used Space` (usage-based, whereas Windows counters are
> free-space-based). Internally, the tool evaluates each in the correct direction (higher usage =
> worse) accordingly.

### `--source direct` (hidden option — direct SSH connection when Azure Arc isn't available)

| # | Source | What it reads | Command |
|---|------|------------|------|
| 1 | CPU | Usage computed from two `/proc/stat` samples (2s apart) | `cat /proc/stat; sleep 2; cat /proc/stat` |
| 2 | Memory | Usage vs. total | `cat /proc/meminfo` |
| 3 | Disk | Usage per mount (POSIX format, for portability) | `df -P` |
| 4 | OOM Killer | Detect OOM events from the kernel log | `journalctl -k` → falls back to `dmesg` on failure |
| 5 | Log errors | err/crit/emerg/alert count | `journalctl -p err..alert` → falls back to grepping `/var/log/syslog`/`messages` on failure |
| 6 | Patch | Pending security update count (auto-detects distro) | `apt list --upgradable` or `yum`/`dnf --security check-update` |

`--host` (falls back to `--computer` if omitted) and `--ssh-user` are **required**. Works even
without any Azure Monitor/Log Analytics workspace at all — this is the core of on-prem/AWS/GCP
server support.

**Same philosophy as DB/system account separation**: the SSH password is never accepted as a CLI
argument — it's read from the environment variable named by `--ssh-password-env` (default
`LINUX_DIAGNOSE_SSH_PASSWORD`), or a private key can be used via `--ssh-key-file`, or it's prompted
interactively in a terminal.

**Security (host key verification)**: unknown host keys are **rejected** by default
(`RejectPolicy`). It's recommended to point `--ssh-known-hosts <path>` at a known_hosts file. This
can be bypassed with `--ssh-insecure-auto-add-host-key` (test environments only, MITM risk — using
it surfaces a security warning finding in the report).

```text
   --host + --ssh-user (+ password/key) ──▶ SSH connection
                                          └─▶ read-only commands (procfs/df/journalctl, etc.) → OS diagnostic report
   --resource-id (optional, Azure VM or Arc machine) ──▶ azure-mgmt-compute/azure-mgmt-hybridcompute (control plane)
```

---

## ⚡ Server load from `--source direct`

Since direct mode runs commands on the target server, understanding the load impact matters.
Bottom line: **each run executes 6 short query commands sequentially (mostly ms to hundreds of ms)
— it is not a continuously polling monitoring agent.**

| Command | Load level | Notes |
|---|---|---|
| `cat /proc/stat` (twice, `sleep 2` apart) | Very low | `sleep` just waits without using CPU (simple kernel counter reads) |
| `cat /proc/meminfo` | Very low | Kernel counter read, returns instantly |
| `df -P` | Low | Only queries mounted filesystem metadata (can be delayed if a network mount is unresponsive) |
| `journalctl -k` / `dmesg` (OOM detection) | Low–moderate | Can take longer if the journal is very large (`--since` limits the range) |
| `journalctl -p err..alert` (log errors) | Low–moderate | Same as above |
| `apt list --upgradable` / `yum`·`dnf --security check-update` | **Moderate–high** | apt is relatively light (local cache), but **yum/dnf fetch fresh repo metadata over the network**, making it noticeably slower than other commands and adding load to the repo server |

The heaviest item is the **patch check (yum/dnf)**. If you want to avoid this overhead on repeated
diagnostics in production, skip it with `--skip-patch-check`:

```bash
python linux_diagnose.py --source direct --host 10.0.5.20 --ssh-user diag_reader --skip-patch-check
```

Additionally, each SSH command has a 20-second timeout (per `_ssh_run` call), so even if a
particular command unexpectedly takes long, the overall diagnosis doesn't hang indefinitely — that
item is simply marked "query failed" while the rest continues.

---

## ⚙️ Prerequisites

**When using MCP server / Azure SRE Agent**
- No separate install is needed — this tool is already installed as a pip package in the MCP server container, and the required credentials (managed identity or `LINUX_DIAGNOSE_SSH_PASSWORD`) are already configured there. Just call the `diagnose_linux_os` tool from the SRE Agent.

**When running standalone**
- Python 3.10+
- For direct mode (SSH), you need SSH access to the target Linux server. azure-monitor mode needs `az login` (locally) or a managed identity.

---

## ⚙️ Installation & Execution

**When using MCP server / Azure SRE Agent**, no installation is needed — calling the tool from the SRE Agent portal runs it inside the MCP server.

**When running standalone**:

```bash
python -m venv .venv && source .venv/bin/activate     # Windows: .\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

`--source direct` uses `paramiko` (a pure-Python SSH client) — no separate SSH client installation
is needed.

> [!TIP]
> **Windows note.** Set `$env:PYTHONIOENCODING="utf-8"` before running (prevents `UnicodeEncodeError`
> in the table renderer).

---

## 🧰 Usage

```bash
# [azure-monitor] basic diagnosis (table)
python linux_diagnose.py --computer linux-app01 --workspace-id <workspace-guid>

# [azure-monitor] including VM control plane (power state/size)
python linux_diagnose.py --computer linux-app01 --workspace-id <workspace-guid> \
  --resource-id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Compute/virtualMachines/linux-app01

# [direct] diagnose directly over SSH, no Azure Monitor needed (on-prem/AWS/GCP, etc.)
$env:LINUX_DIAGNOSE_SSH_PASSWORD = "..."   # PowerShell (or export, in bash)
python linux_diagnose.py --source direct --host 10.0.5.20 --ssh-user diag_reader --format table

# [direct] SSH key auth + known_hosts verification
python linux_diagnose.py --source direct --host onprem-db01.corp.local --ssh-user diag_reader \
  --ssh-key-file ~/.ssh/diag_reader_id_ed25519 --ssh-known-hosts ~/.ssh/known_hosts

# [direct] password auth + known_hosts verification (Windows, verified working)
# 1) connect once manually via ssh to confirm the fingerprint and register it in known_hosts with 'yes'
#    ssh admin@192.168.1.5
# 2) afterwards, just point --ssh-known-hosts at the Windows OpenSSH known_hosts path
python linux_diagnose.py --source direct --host 192.168.1.5 --ssh-user admin `
  --ssh-known-hosts $env:USERPROFILE\.ssh\known_hosts --format html -o linux.html

# adjust thresholds (both modes)
python linux_diagnose.py --computer linux-app01 --workspace-id <guid> \
  --cpu-warn 75 --cpu-crit 90 --disk-used-warn 80 --disk-used-crit 90

# JSON output + exit code for CI
python linux_diagnose.py --computer linux-app01 --workspace-id <guid> --format json --exit-code

# save an HTML report to a file
python linux_diagnose.py --computer linux-app01 --workspace-id <guid> --format html -o linux_report.html

# preview with demo data (no Azure calls)
python linux_diagnose.py --demo --format table
```

---

## 🧰 Diagnosing Multiple Servers at Once (Multi-host)

**There's no built-in batch option** — `--host`/`--computer` only accepts a single target per run (the same design principle across every tool in this suite). To diagnose multiple servers at once, run an external loop.

```powershell
# PowerShell, sequential
$env:LINUX_DIAGNOSE_SSH_PASSWORD = "..."
$hosts = "10.0.5.20", "10.0.5.21", "192.168.1.5"
foreach ($h in $hosts) {
  python linux_diagnose.py --source direct --host $h --ssh-user diag_reader `
    --ssh-known-hosts $env:USERPROFILE\.ssh\known_hosts --format html -o "$h.html"
}
```

```powershell
# PowerShell, parallel (ForEach-Object -Parallel, PowerShell 7+)
$hosts | ForEach-Object -Parallel {
  python linux_diagnose.py --source direct --host $_ --ssh-user diag_reader --format json -o "$_.json"
} -ThrottleLimit 5
```

```bash
# bash, parallel (xargs)
printf '%s\n' 10.0.5.20 10.0.5.21 192.168.1.5 | \
  xargs -P 5 -I{} python linux_diagnose.py --source direct --host {} --ssh-user diag_reader --format json -o {}.json
```

Running many hosts in parallel increases concurrent connection load proportionally to the number of
targets (each diagnosis itself is a lightweight read-only operation), so limit `-ThrottleLimit`/`-P`
appropriately in production. To see results at a glance, aggregate/tabulate the per-host JSON output
yourself, or have MCP/SRE Agent iterate over a host list and call the tool for each one.

---

## 🧰 Time-window selection & history

**You can choose a period like "the last week" or "yesterday"** — the azure-monitor mode supports
two approaches.

| Approach | Example | Notes |
|---|---|---|
| Relative `--hours` (default) | `--hours 24` (1 day), `--hours 168` (1 week), `--hours 720` (30 days) | Always "N hours ago until now" |
| Absolute `--start-time`/`--end-time` | `--start-time 2026-07-28T00:00:00Z --end-time 2026-07-30T00:00:00Z` | Pin down an exact date range (e.g., Mon–Wed of last week) |

```bash
# query a specific 3-day window from last week
python linux_diagnose.py --computer linux-app01 --workspace-id <guid> \
  --start-time 2026-07-28T00:00:00Z --end-time 2026-07-31T00:00:00Z --format html -o last_week.html
```

**Direct mode (SSH) can only query the target server's "right now" state, so past-date queries
aren't possible** (using it together with `--start-time`/`--end-time` returns an error). Instead,
you can use **`--save-snapshot <path>`** to append each run's results to a JSON Lines file — running
this periodically via cron (e.g., hourly) lets you build your own time-series history for later
comparison/trend analysis.

```bash
# register in cron to run hourly → builds its own history
python linux_diagnose.py --source direct --host 10.0.5.20 --ssh-user diag_reader \
  --format json --save-snapshot /var/log/diag-history/linux-app01.jsonl
```

---

## 🧰 Diagnostic Rules

| Category | Signal | Warning | Critical | Formula / Notes |
|----------|--------|---------|----------|---------------|
| `connectivity` | Heartbeat gap | ≥ `--heartbeat-warn-min` (default 15 min) | ≥ `--heartbeat-crit-min` (default 60 min) | `now - last_heartbeat` |
| `cpu` | Average % Processor Time | ≥ `--cpu-warn` (default 80%) | ≥ `--cpu-crit` (default 95%) | Average over the `Perf` window |
| `memory` | Average % Used Memory | ≥ `--mem-warn` (default 85%) | ≥ `--mem-crit` (default 95%) | Average over the `Perf` window |
| `disk` | Maximum % Used Space (per mount) | ≥ `--disk-used-warn` (default 85%) | ≥ `--disk-used-crit` (default 95%) | Mount with the highest usage |
| `syslog` | err/crit/emerg/alert count | ≥ `--log-err-warn` (default 10) | ≥ `--log-err-crit` (default 50) | Total within the window |
| `stability` | OOM Killer event | — | 1+ = critical | `Out of memory`/`oom-kill` message detection |
| `patch` | Missing security update count | 1+ | 10+ | `Update` table (Update Management) |
| `os_lifecycle` | Distro EOL lifecycle (direct mode only) | ≤ `--eol-warn-days` (default 180) until EOL | Already past EOL | Embedded lifecycle table lookup |
| `software_eol` | Known EOL packages installed (direct mode only) | 1+ found | — | Python2/PHP/MySQL/OpenSSL, etc. pattern match |
| `recommended_software` | Recommended management tool missing (direct mode only) | — | — | Always info (not treated as a problem) |
| `control_plane` | VM power state | — | Deallocated/Stopped = critical | ARM instanceView (optional) |

Every item is shown as **"not evaluated"** rather than hidden when data is unavailable (same
report-completeness philosophy as the other tools).

---

## 🧰 Output Schema

Uses the same `category` + `severity` schema as the other tools, with top-level
`summary`/`health_score`/`severity_counts`/`recommended_actions`/`needs_input`.

| Mode | Content |
|---|---|
| `--format table` (default) | Human-readable text table |
| `--format json` | `checks[]`-schema JSON (for SRE Agent / MCP integration) |
| `--format html` | Summary cards + diagnostic table (`-o` to save to a file) |

- `--exit-code` returns a CI-friendly exit code (critical=2, warning=1, otherwise 0)

### Sample output

```json
{
  "tool": "linux_diagnose",
  "version": "1.0.0",
  "computer": "linux-app01",
  "window": "last 24h",
  "checks": [
    {
      "category": "stability",
      "severity": "critical",
      "title": "OOM Killer event detected",
      "detail": "3 occurrences. e.g., Out of memory: Killed process 12345 (java) ...",
      "recommendation": "A process was killed due to low memory. Check for memory leaks/high-usage processes and add memory if needed."
    }
  ],
  "health_score": 55,
  "severity_counts": {"critical": 1, "warning": 2, "info": 0, "ok": 4},
  "summary": "Health score 55/100 (critical). linux-app01, last 24h (critical 1, warning 2, info 0).",
  "worst_severity": "critical"
}
```

---

## 🧰 Autonomous discovery → reinvoke loop (needs_input)

In `--source azure-monitor`, a missing `--workspace-id` is flagged as required, and a missing
`--resource-id` (in either mode) is flagged as optional — the top-level `needs_input` in the result
JSON is populated with the needed values, a `discovery_hint` (an example Resource Graph KQL query),
and a `reinvoke_example` (an example reinvocation command). `--source direct` may target on-prem/
AWS/GCP hosts where a resource_id might not even exist, so it is **never forced** (you can still pass
`--resource-id` directly if you know it). Azure SRE Agent can use this to resolve the value and
reinvoke the tool.

---

## 🧰 MCP / Azure SRE Agent integration

Registered in `mcp_server.py` as the `diagnose_linux_os` tool (same pattern as pg/aks/adx/eh/agw/
svcmap/windows — a common `_run()` wrapper turns timeouts/failures into structured JSON). The
`source` parameter selects `azure-monitor`/`direct`; when `direct`, `host`/`ssh_user` are passed (the
password is never passed as an MCP argument — the MCP container must have
`LINUX_DIAGNOSE_SSH_PASSWORD` set, or a mounted SSH key, in advance).

Grant the Managed Identity the following RBAC (when using azure-monitor mode and/or the control plane):

- **Log Analytics workspace**: `Log Analytics Reader` (Perf/Syslog/Heartbeat/Update queries)
- **VM (optional, when using `--resource-id`)**: `Reader` (instanceView query)

`direct` mode is independent of Azure permissions — it only needs an account that can connect to the
target server over SSH (read-only permissions recommended).

---

## Limitations / Accuracy notes

- Arc-enabled servers (Microsoft.HybridCompute) don't support control-plane queries in the current
  version (only Azure VM is supported). The data plane (Perf/Syslog/Heartbeat, or direct SSH
  collection) works the same for Arc servers.
- `--source azure-monitor`: `Syslog`/`Update` collection requires the data collection rule (DCR) to
  include the relevant facility/severity; some distros need rsyslog/journald forwarding configured.
- `--source direct`: the patch check only auto-detects apt/yum/dnf (other package managers aren't
  supported). Older distros without `journalctl` fall back to `dmesg`/`/var/log/syslog`, so some
  events may be missed if log retention is short.
- No remote command execution (Run Command, etc.) is used — azure-monitor mode only reads
  already-collected telemetry, and direct mode only runs read-only commands (`cat`/`df`/`journalctl`,
  etc.).


---

## License

This project is licensed under the [MIT License](LICENSE).
