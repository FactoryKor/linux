**한국어** | [English](README.en.md)

# 🚀 linux_diagnose

**Linux 서버(Azure VM / Arc-enabled server) 읽기 전용(read-only) OS 진단 프로그램**

대상 Linux 서버의 CPU/메모리/디스크/에이전트 연결/Syslog/OOM Killer/패치 상태를 진단하는 도구입니다. `pg_diagnose` / `aks_diagnose` / `adx_diagnose` / `eh_diagnose` / `agw_diagnose` / `svcmap_diagnose` / `windows_diagnose`와 동일한 진단 도구 제품군의 일원이며, 동일한 설계 철학(엄격한 read-only, JSON/HTML/table 출력, `health_score`/`summary`/`recommended_actions`)을 그대로 따릅니다.

- 👉 이미 수집된 **Azure Monitor Agent 원격 측정(Perf/Syslog/Heartbeat)**을 읽기 전용 KQL로 조회합니다. 서버에 새 명령을 내리거나 원격 접속(SSH 등)을 하지 않습니다.
- 👉 CPU / 메모리 / 디스크 사용률 / 에이전트 연결 상태 / Syslog 오류 / OOM Killer 발생 / 패치 준수 상태를 진단합니다.
- 👉 Azure SRE Agent 및 MCP Tool 통합을 지원합니다.
- 👉 `--resource-id` 지정 시 VM 전원 상태/크기/OS 버전(제어 평면)도 함께 조회합니다.

주요 진단 항목:

| 📋 항목 | 📋 항목 |
|---|---|
| 에이전트 연결(Heartbeat) 상태 | CPU 사용률(Processor/% Processor Time) |
| 메모리 사용률(Memory/% Used Memory) | 디스크 사용률(Logical Disk/% Used Space) |
| Syslog err/crit/emerg/alert 건수 | OOM Killer(메모리 부족 프로세스 강제 종료) 발생 |
| 보안 업데이트 누락(Update Management) | VM 전원 상태/크기/OS 버전(선택, ARM) |

---

## 데이터 소스 (모두 읽기 전용 KQL 질의 / ARM 조회)

| # | 소스 | 무엇을 읽나 | 테이블/질의 |
|---|------|------------|------|
| 1 | **Heartbeat** | 에이전트 연결 상태(마지막 하트비트 간격) | `Heartbeat` |
| 2 | **Perf** | CPU/메모리/디스크 사용률(Linux AMA 카운터 스키마) | `Perf` (`Processor`, `Memory`, `Logical Disk`) |
| 3 | **Syslog** | 심각도별 오류 건수, OOM Killer 탐지 | `Syslog` (SeverityLevel, `Out of memory`/`oom-kill` 메시지) |
| 4 | **Update** | 패치 준수 상태(Update Management 활성화 시) | `Update` |
| 5 | **ARM(선택)** | VM 전원 상태/크기/OS 버전(제어 평면) | `azure-mgmt-compute` instanceView |

`--computer`(Log Analytics `Computer` 컬럼 값)와 `--workspace-id`(Log Analytics workspace GUID)가 **필수**입니다. `--resource-id`는 선택이며, VM 제어 평면 정보(전원 상태 등)를 추가로 조회할 때 사용합니다.

```text
   --computer + --workspace-id ──▶ Heartbeat/Perf/Syslog/Update KQL (데이터 평면)
   --resource-id (선택)         ──▶ azure-mgmt-compute instanceView (제어 평면)
                                    └─▶ 병합 → OS 진단 리포트
```

> [!NOTE]
> Linux Perf 카운터는 Windows와 오브젝트 이름 표기가 다릅니다: `Logical Disk`(공백 있음, Windows는 `LogicalDisk`), `% Used Memory`/`% Used Space`(사용률 기준, Windows는 반대로 여유공간 기준). 도구 내부에서 이 차이를 각각 올바른 방향(사용률 높을수록 나쁨)으로 평가합니다.

---

## ⚙️ 설치

```bash
python -m venv .venv && source .venv/bin/activate     # Windows: .\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

> [!TIP]
> **Windows note.** 실행 전 `$env:PYTHONIOENCODING="utf-8"` 를 설정하세요(테이블 렌더러의 `UnicodeEncodeError` 방지).

---

## 🧰 사용법

```bash
# 기본 진단 (table)
python linux_diagnose.py --computer linux-app01 --workspace-id <workspace-guid>

# VM 제어 평면(전원 상태/크기)까지 포함
python linux_diagnose.py --computer linux-app01 --workspace-id <workspace-guid> \
  --resource-id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Compute/virtualMachines/linux-app01

# 임계값 조정
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

`--workspace-id`나 `--resource-id`가 없으면 결과 JSON의 최상위 `needs_input`에 필요한 값과 `discovery_hint`(Resource Graph KQL 예시), `reinvoke_example`(재호출 명령 예시)가 채워집니다. Azure SRE Agent는 이를 참고해 값을 확정한 뒤 도구를 재호출할 수 있습니다.

---

## 🧰 MCP / Azure SRE Agent 통합

`mcp_server.py`에 `diagnose_linux_os` 도구로 등록되어 있습니다(pg/aks/adx/eh/agw/svcmap/windows와 동일 패턴, `_run()` 공통 래퍼로 타임아웃/실패를 구조화 JSON으로 반환).

Managed Identity에 다음 RBAC를 부여하면 됩니다.

- **Log Analytics workspace**: `Log Analytics Reader` (Perf/Syslog/Heartbeat/Update 조회)
- **VM(선택, `--resource-id` 사용 시)**: `Reader` (instanceView 조회)

---

## 한계 / 정확성 주의

- Arc-enabled server(Microsoft.HybridCompute)는 현재 버전에서 제어 평면 조회를 지원하지 않습니다(Azure VM만 지원). 데이터 평면(Perf/Syslog/Heartbeat)은 Arc 서버도 동일하게 동작합니다.
- `Syslog`/`Update` 수집은 데이터 수집 규칙(DCR)에 해당 시설(facility)/심각도가 포함돼 있어야 채워집니다. 배포판에 따라 syslog 데몬 설정(rsyslog/journald 포워딩)이 필요할 수 있습니다.
- 원격 명령 실행(SSH/Run Command 등)은 사용하지 않습니다 — 이미 수집된 원격 측정만 읽으므로 최소 권한(Reader 계열)으로 동작합니다.
