**한국어** | [English](README.en.md)

# 🚀 linux_diagnose

**Linux 서버 읽기 전용(read-only) OS 진단 프로그램 — 온프레미스/Azure/AWS/GCP 등 어디서든 Azure Arc로 통합 관리**

대상 Linux 서버의 CPU/메모리/디스크/에이전트 연결/Syslog/OOM Killer/패치 상태를 진단하는 도구입니다. `pg_diagnose` / `aks_diagnose` / `adx_diagnose` / `eh_diagnose` / `agw_diagnose` / `svcmap_diagnose` / `windows_diagnose`와 동일한 진단 도구 제품군의 일원이며, 동일한 설계 철학(엄격한 read-only, JSON/HTML/table 출력, `health_score`/`summary`/`recommended_actions`)을 그대로 따릅니다.

## ✅ 권장 사용 방식 — Azure Arc 연결

이 도구는 기본적으로 **Azure Monitor Agent 원격 측정(Log Analytics)**을 읽기 전용 KQL로 조회하도록 설계되었습니다. 온프레미스/AWS/GCP 서버는 [Azure Arc-enabled servers](https://learn.microsoft.com/azure/azure-arc/servers/overview)로 온보딩하면(`azcmagent connect`), Azure VM과 동일하게 Azure Monitor Agent를 설치하고 Log Analytics workspace로 원격 측정을 보낼 수 있습니다 — 즉 **Arc 연결이 이 도구의 "정식 지원 경로"의 전제 조건**입니다.

```bash
python linux_diagnose.py --computer linux-app01 --workspace-id <workspace-guid> \
  --resource-id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.HybridCompute/machines/linux-app01
```

`--resource-id`에 Arc 머신(`Microsoft.HybridCompute/machines`) 또는 Azure VM(`Microsoft.Compute/virtualMachines`) 리소스 ID를 지정하면, 두 유형 모두에 대해 제어 평면 정보(Arc는 에이전트 연결 상태, Azure VM은 전원 상태)를 자동으로 조회합니다.

## 🔒 숨은 옵션 — Azure Arc를 연결할 수 없는 경우

방화벽·정책·조직 사정 등으로 Azure Arc를 연결할 수 없는 예외적인 환경을 위해, **SSH로 직접 접속해 수집하는 `--source direct` 옵션도 내부적으로 지원**합니다. 이 옵션은 일반 사용자에게 권장하는 경로가 아니므로 `--help`에는 표시되지 않지만, 아래처럼 명시적으로 지정하면 그대로 동작합니다.

```bash
# --help에는 나오지 않지만 실제로 동작하는 숨은 옵션
$env:LINUX_DIAGNOSE_SSH_PASSWORD = "..."
python linux_diagnose.py --source direct --host 10.0.5.20 --ssh-user diag_reader --format table
```

| | `--source azure-monitor` (기본, 정식 지원) | `--source direct` (숨은 옵션, Arc 미연결 시) |
|---|---|---|
| 무엇을 읽나 | 이미 수집된 Azure Monitor Agent 원격 측정(Log Analytics) | SSH로 접속해 실시간 procfs/명령 실행 결과 |
| 필요 조건 | **Azure Arc 온보딩**(또는 Azure VM) + Azure Monitor Agent + Log Analytics workspace | SSH 접근 가능(사용자/비밀번호 또는 키) |
| 노출 여부 | `--help`에 표시됨(권장 경로) | `--help`에 숨김(문서에서만 안내) |
| 서버 변경 여부 | 없음(질의만) | 없음(읽기 전용 명령만 실행, SSH로 새 에이전트 설치 없음) |

전체 옵션(`--ssh-user`, `--ssh-password-env`, `--ssh-key-file`, `--ssh-known-hosts`, `--ssh-insecure-auto-add-host-key`, `--skip-patch-check` 등)은 이 문서 하단 "숨은 옵션 상세"를 참고하세요.

- 👉 CPU / 메모리 / 디스크 사용률 / 에이전트(또는 SSH) 연결 상태 / Syslog 오류 / OOM Killer 발생 / 패치 준수 상태를 진단합니다.
- 👉 Azure SRE Agent 및 MCP Tool 통합을 지원합니다.
- 👉 `--resource-id` 지정 시(Azure VM 또는 Arc 머신, 두 수집 방식 공통) 제어 평면 정보도 함께 조회합니다.

주요 진단 항목:

| 📋 항목 | 📋 항목 |
|---|---|
| 연결 상태(Heartbeat 또는 SSH 연결) | CPU 사용률 |
| 메모리 사용률 | 디스크 사용률(마운트별) |
| 로그 오류 건수(Syslog) | OOM Killer(메모리 부족 프로세스 강제 종료) 발생 |
| 보안 업데이트 누락 | VM/Arc 머신 연결·전원 상태, 크기, OS 버전(선택, ARM) |

---

## 데이터 소스

### `--source azure-monitor` (모두 읽기 전용 KQL 질의 / ARM 조회)

| # | 소스 | 무엇을 읽나 | 테이블/질의 |
|---|------|------------|------|
| 1 | **Heartbeat** | 에이전트 연결 상태(마지막 하트비트 간격) | `Heartbeat` |
| 2 | **Perf** | CPU/메모리/디스크 사용률(Linux AMA 카운터 스키마) | `Perf` (`Processor`, `Memory`, `Logical Disk`) |
| 3 | **Syslog** | 심각도별 오류 건수, OOM Killer 탐지 | `Syslog` (SeverityLevel, `Out of memory`/`oom-kill` 메시지) |
| 4 | **Update** | 패치 준수 상태(Update Management 활성화 시) | `Update` |
| 5 | **ARM(선택)** | VM 또는 Arc 머신의 연결·전원 상태/크기/OS 버전(제어 평면) | `azure-mgmt-compute`(Azure VM) 또는 `azure-mgmt-hybridcompute`(Arc) |

`--computer`(Log Analytics `Computer` 컬럼 값)와 `--workspace-id`(Log Analytics workspace GUID)가 **필수**입니다.

```text
   --computer + --workspace-id ──▶ Heartbeat/Perf/Syslog/Update KQL (데이터 평면)
   --resource-id (선택, Azure VM 또는 Arc 머신) ──▶ azure-mgmt-compute 또는
                                        azure-mgmt-hybridcompute instanceView (제어 평면)
                                        └─▶ 병합 → OS 진단 리포트
```

> [!NOTE]
> Linux Perf 카운터는 Windows와 오브젝트 이름 표기가 다릅니다: `Logical Disk`(공백 있음, Windows는 `LogicalDisk`), `% Used Memory`/`% Used Space`(사용률 기준, Windows는 반대로 여유공간 기준). 도구 내부에서 이 차이를 각각 올바른 방향(사용률 높을수록 나쁨)으로 평가합니다.

### `--source direct` (숨은 옵션 — Azure Arc 연결 불가 시, SSH로 직접 접속)

| # | 소스 | 무엇을 읽나 | 명령 |
|---|------|------------|------|
| 1 | CPU | `/proc/stat` 2회 샘플링(2초 간격)으로 사용률 계산 | `cat /proc/stat; sleep 2; cat /proc/stat` |
| 2 | 메모리 | 전체 대비 사용률 | `cat /proc/meminfo` |
| 3 | 디스크 | 마운트별 사용률(POSIX 형식, 이식성 우선) | `df -P` |
| 4 | OOM Killer | 커널 로그에서 OOM 이벤트 탐지 | `journalctl -k` → 실패 시 `dmesg` 폴백 |
| 5 | 로그 오류 | err/crit/emerg/alert 건수 | `journalctl -p err..alert` → 실패 시 `/var/log/syslog`·`messages` grep 폴백 |
| 6 | 패치 | 보안 업데이트 대기 건수(배포판 자동 감지) | `apt list --upgradable` 또는 `yum`/`dnf --security check-update` |

`--host`(미지정 시 `--computer` 값 사용)와 `--ssh-user`가 **필수**입니다. Azure Monitor/Log Analytics workspace가 전혀 없어도 동작합니다 — 이것이 온프레미스·AWS·GCP 서버 지원의 핵심입니다.

**DB 계정/시스템 계정 분리와 동일한 철학**: SSH 비밀번호는 CLI 인자로 절대 받지 않습니다 — `--ssh-password-env`로 지정한 환경변수(기본 `LINUX_DIAGNOSE_SSH_PASSWORD`)에서 읽거나, `--ssh-key-file`(개인키 파일)을 사용하거나, 대화형 터미널이면 프롬프트로 입력받습니다.

**보안(호스트 키 검증)**: 기본적으로 알 수 없는 호스트 키는 **거부**합니다(`RejectPolicy`). `--ssh-known-hosts <경로>`로 known_hosts 파일을 지정하는 것을 권장합니다. `--ssh-insecure-auto-add-host-key`(테스트 환경 전용, MITM 위험 — 사용 시 리포트에 보안 경고가 표시됩니다)로 우회할 수 있습니다.

```text
   --host + --ssh-user (+ 비밀번호/키) ──▶ SSH 접속
                                          └─▶ procfs/df/journalctl 등 읽기 전용 명령 → OS 진단 리포트
   --resource-id (선택, Azure VM 또는 Arc 머신) ──▶ azure-mgmt-compute/azure-mgmt-hybridcompute (제어 평면)
```

---

## ⚡ `--source direct`의 대상 서버 부하

direct 모드는 대상 서버에서 명령을 실행하므로 "부하가 얼마나 되는지"가 중요합니다. 결론: **한 번 실행할 때마다 짧은 조회 명령 6개를 순차 실행하는 수준(대부분 수 ms~수백 ms)이며, 지속적으로 폴링하는 모니터링 에이전트가 아닙니다.**

| 명령 | 부하 수준 | 비고 |
|---|---|---|
| `cat /proc/stat` (2회, `sleep 2` 간격) | 매우 낮음 | `sleep`은 CPU를 쓰지 않고 대기만 함(단순 커널 카운터 읽기) |
| `cat /proc/meminfo` | 매우 낮음 | 커널 카운터 읽기, 즉시 반환 |
| `df -P` | 낮음 | 마운트된 파일시스템 메타데이터만 조회(네트워크 마운트가 응답 없으면 지연 가능) |
| `journalctl -k` / `dmesg` (OOM 탐지) | 낮음~보통 | 저널 크기가 매우 크면 다소 걸릴 수 있음(`--since`로 범위 제한) |
| `journalctl -p err..alert` (로그 오류) | 낮음~보통 | 위와 동일 |
| `apt list --upgradable` / `yum`·`dnf --security check-update` | **보통~높음** | apt는 로컬 캐시라 비교적 가벼우나, **yum/dnf는 저장소 메타데이터를 네트워크로 새로 받아와** 다른 명령보다 눈에 띄게 느리고 저장소 서버에도 부하를 줍니다 |

가장 부하가 큰 항목은 **패치 점검(yum/dnf)**입니다. 운영 환경에서 반복 진단 시 이 부담을 피하고 싶다면 `--skip-patch-check`로 건너뛸 수 있습니다:

```bash
python linux_diagnose.py --source direct --host 10.0.5.20 --ssh-user diag_reader --skip-patch-check
```

추가로, SSH 명령 각각에는 20초 타임아웃이 걸려 있어(개별 `_ssh_run` 호출), 특정 명령이 예상외로 오래 걸려도 전체 진단이 무한정 멈추지 않고 해당 항목만 "조회 실패"로 표시된 채 나머지가 계속 진행됩니다.

---

## ⚙️ 설치

```bash
python -m venv .venv && source .venv/bin/activate     # Windows: .\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

`--source direct`는 `paramiko`(순수 파이썬 SSH 클라이언트)를 사용합니다 — 별도 SSH 클라이언트 설치가 필요 없습니다.

> [!TIP]
> **Windows note.** 실행 전 `$env:PYTHONIOENCODING="utf-8"` 를 설정하세요(테이블 렌더러의 `UnicodeEncodeError` 방지).

---

## 🧰 사용법

```bash
# [azure-monitor] 기본 진단 (table)
python linux_diagnose.py --computer linux-app01 --workspace-id <workspace-guid>

# [azure-monitor] VM 제어 평면(전원 상태/크기)까지 포함
python linux_diagnose.py --computer linux-app01 --workspace-id <workspace-guid> \
  --resource-id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Compute/virtualMachines/linux-app01

# [direct] 온프레미스/AWS/GCP 등 Azure Monitor 없이 SSH로 직접 진단
$env:LINUX_DIAGNOSE_SSH_PASSWORD = "..."   # PowerShell (또는 export, bash)
python linux_diagnose.py --source direct --host 10.0.5.20 --ssh-user diag_reader --format table

# [direct] SSH 키 인증 + known_hosts 검증
python linux_diagnose.py --source direct --host onprem-db01.corp.local --ssh-user diag_reader \
  --ssh-key-file ~/.ssh/diag_reader_id_ed25519 --ssh-known-hosts ~/.ssh/known_hosts

# 임계값 조정 (두 방식 공통)
python linux_diagnose.py --computer linux-app01 --workspace-id <guid> \
  --cpu-warn 75 --cpu-crit 90 --disk-used-warn 80 --disk-used-crit 90

# JSON 출력 + CI용 exit code
python linux_diagnose.py --computer linux-app01 --workspace-id <guid> --format json --exit-code

# HTML 리포트 파일로 저장
python linux_diagnose.py --computer linux-app01 --workspace-id <guid> --format html -o linux_report.html

# 데모 데이터로 미리보기 (Azure 호출 없음)
python linux_diagnose.py --demo --format table
```

---

## 🧰 조회 기간 선택 & 이력 저장

**"1주일치", "1일치" 같은 기간을 직접 고를 수 있습니다** — azure-monitor 모드에서 두 가지 방식을 지원합니다.

| 방식 | 예시 | 비고 |
|---|---|---|
| 상대 기간 `--hours` (기본) | `--hours 24`(1일), `--hours 168`(1주일), `--hours 720`(30일) | 항상 "지금부터 N시간 전"까지 |
| 절대 기간 `--start-time`/`--end-time` | `--start-time 2026-07-28T00:00:00Z --end-time 2026-07-31T00:00:00Z` | 특정 날짜 구간(예: 지난주 월~수)을 정확히 지정 |

```bash
# 지난주 특정 3일 구간만 조회
python linux_diagnose.py --computer linux-app01 --workspace-id <guid> \
  --start-time 2026-07-28T00:00:00Z --end-time 2026-07-31T00:00:00Z --format html -o last_week.html
```

**direct 모드(SSH)는 대상 서버의 "지금 이 순간" 상태만 조회할 수 있어 과거 날짜 조회가 불가능합니다** (`--start-time`/`--end-time`과 함께 쓰면 오류 반환). 대신 **`--save-snapshot <경로>`**로 실행할 때마다 결과를 JSON Lines 파일에 이어붙일 수 있습니다 — 이를 cron으로 주기 실행(예: 매시간)하면, 직접 시계열 이력을 축적해 나중에 비교·추세 분석을 할 수 있습니다.

```bash
# 매시간 실행되도록 cron에 등록 → 스스로 이력을 쌓음
python linux_diagnose.py --source direct --host 10.0.5.20 --ssh-user diag_reader \
  --format json --save-snapshot /var/log/diag-history/linux-app01.jsonl
```

---

## 🧰 진단 규칙 (Diagnostic Rules)

| Category | Signal | Warning | Critical | 계산식 / 비고 |
|----------|--------|---------|----------|---------------|
| `connectivity` | Heartbeat 간격 | ≥ `--heartbeat-warn-min`(기본 15분) | ≥ `--heartbeat-crit-min`(기본 60분) | `now - last_heartbeat` |
| `cpu` | 평균 % Processor Time | ≥ `--cpu-warn`(기본 80%) | ≥ `--cpu-crit`(기본 95%) | `Perf` 창 평균 |
| `memory` | 평균 % Used Memory | ≥ `--mem-warn`(기본 85%) | ≥ `--mem-crit`(기본 95%) | `Perf` 창 평균 |
| `disk` | 최대 % Used Space(마운트별) | ≥ `--disk-used-warn`(기본 85%) | ≥ `--disk-used-crit`(기본 95%) | 가장 사용률이 높은 마운트 |
| `syslog` | err/crit/emerg/alert 건수 | ≥ `--log-err-warn`(기본 10건) | ≥ `--log-err-crit`(기본 50건) | 창 내 합계 |
| `stability` | OOM Killer 발생 | — | 1건 이상 = critical | `Out of memory`/`oom-kill` 메시지 탐지 |
| `patch` | 보안 업데이트 누락 건수 | 1건 이상 | 10건 이상 | `Update` 테이블(Update Management) |
| `control_plane` | VM 전원 상태 | — | Deallocated/Stopped = critical | ARM instanceView(선택) |

모든 항목은 데이터가 없으면 숨기지 않고 **"미평가"**로 표시됩니다(다른 도구와 동일한 리포트 완결성 철학).

---

## 🧰 출력 스키마 (Output Schema)

다른 도구와 동일한 `category` + `severity` 스키마를 사용하며, 최상위에 `summary`/`health_score`/`severity_counts`/`recommended_actions`/`needs_input`이 포함됩니다.

| 모드 | 내용 |
|---|---|
| `--format table` (기본) | 사람이 읽기 쉬운 텍스트 테이블 |
| `--format json` | `checks[]` 스키마 JSON (SRE Agent / MCP 연동용) |
| `--format html` | 요약 카드 + 진단표 (`-o`로 파일 저장) |

- `--exit-code` 지정 시 CI용 종료코드 반환(critical=2, warning=1, 그 외 0)

### 샘플 출력

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
      "title": "OOM Killer 발생 감지",
      "detail": "3건 발생. 예: Out of memory: Killed process 12345 (java) ...",
      "recommendation": "메모리 부족으로 프로세스가 강제 종료되었습니다. 메모리 누수/과다 사용 프로세스를 확인하고 필요 시 메모리를 증설하세요."
    }
  ],
  "health_score": 55,
  "severity_counts": {"critical": 1, "warning": 2, "info": 0, "ok": 4},
  "summary": "건강 점수 55/100 (위험). linux-app01, last 24h (위험 1, 주의 2, 정보 0).",
  "worst_severity": "critical"
}
```

---

## 🧰 자율 발견 → 재호출 루프 (needs_input)

`--source azure-monitor`에서 `--workspace-id`가 없으면 필수값으로 안내되고, `--resource-id`가 없으면(어느 모드든) 선택값으로 안내됩니다 — 결과 JSON의 최상위 `needs_input`에 필요한 값과 `discovery_hint`(Resource Graph KQL 예시), `reinvoke_example`(재호출 명령 예시)가 채워집니다. `--source direct`는 온프레미스/AWS/GCP일 수있어 resource_id가 실재하지 않을 수 있으므로 **강요하지 않습니다**(사용자가 알고 있으면 직접 `--resource-id`로 전달 가능). Azure SRE Agent는 이를 참고해 값을 확정한 뒤 도구를 재호출할 수 있습니다.

---

## 🧰 MCP / Azure SRE Agent 통합

`mcp_server.py`에 `diagnose_linux_os` 도구로 등록되어 있습니다(pg/aks/adx/eh/agw/svcmap/windows와 동일 패턴, `_run()` 공통 래퍼로 타임아웃/실패를 구조화 JSON으로 반환). `source` 파라미터로 `azure-monitor`/`direct`를 선택하며, `direct`일 때는 `host`/`ssh_user`를 전달합니다(비밀번호는 MCP 인자로 전달되지 않으며, MCP 컨테이너에 `LINUX_DIAGNOSE_SSH_PASSWORD` 또는 마운트된 SSH 키가 미리 준비돼 있어야 함).

Managed Identity에 다음 RBAC를 부여하면 됩니다(azure-monitor 모드 및/또는 제어 평면 사용 시).

- **Log Analytics workspace**: `Log Analytics Reader` (Perf/Syslog/Heartbeat/Update 조회)
- **VM(선택, `--resource-id` 사용 시)**: `Reader` (instanceView 조회)

`direct` 모드는 Azure 권한과 무관하며, 대상 서버에 SSH로 접속할 수 있는 계정(읽기 전용 권한 권장)만 있으면 됩니다.

---

## 한계 / 정확성 주의

- Arc-enabled server(Microsoft.HybridCompute)는 현재 버전에서 제어 평면 조회를 지원하지 않습니다(Azure VM만 지원). 데이터 평면(Perf/Syslog/Heartbeat 또는 SSH 직접 수집)은 Arc 서버도 동일하게 동작합니다.
- `--source azure-monitor`: `Syslog`/`Update` 수집은 데이터 수집 규칙(DCR)에 해당 시설(facility)/심각도가 포함돼 있어야 채워집니다. 배포판에 따라 syslog 데몬 설정(rsyslog/journald 포워딩)이 필요할 수 있습니다.
- `--source direct`: 패치 점검은 apt/yum/dnf만 자동 감지합니다(다른 패키지 관리자는 미지원). `journalctl`이 없는 구버전 배포판은 `dmesg`/`/var/log/syslog` 폴백을 사용하므로 로그 보존 기간이 짧으면 일부 이벤트를 놓칠 수 있습니다.
- 원격 명령 실행(Run Command 등)은 사용하지 않습니다 — azure-monitor 모드는 이미 수집된 원격 측정만 읽고, direct 모드도 읽기 전용 명령(`cat`/`df`/`journalctl` 등)만 실행합니다.
