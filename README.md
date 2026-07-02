## 🚀 EventHub_Diag
**eh_diagnose는 Azure Event Hubs 읽기 전용(read-only) 진단 프로그램**

Azure Event Hubs 환경을 위한 진단 도구입니다.<br/>
👉 *Control Plane → Metrics → Runtime* 관점으로 분석<br/>
👉 *Consumer Lag / Throttling / Capacity Pressure* 를 진단하며 주요 진단 항목은 다음과 같습니다.<br/>
    📋 Throughput Unit(TU) 사용률 <br/>
    📋 Throttling 발생 여부 <br/>
    📋 Incoming / Outgoing 메시지 추이 <br/>
    📋 Consumer Lag <br/>
    📋 Dead Consumer 감지 <br/>
    📋 Auto Inflate 설정 검증 <br/>
    📋 Capture 상태 <br/>
    📋 Partition 구성 <br/>
    📋 네트워크 및 인증 오류 <br/>
    📋 Storage 기반 Checkpoint 상태 <br/>
👉 Azure SRE Agent 및 MCP Tool 통합을 지원합니다.<br/>
👉 모든 동작은 엄격히 read-only 입니다. (이벤트를 send/receive하지 않고, keyslot·설정 checkpoint를 변경하지 않습니다.)

---
### 인증 도메인 (설계의 핵심)
Event Hubs 진단은 성격이 다른 3(+1)개의 접근 경로를 필요로 한다. 이들을 하나로 뭉치지 않고 명시적으로 분리하는 것이 이 도구의 핵심 설계 결정 입니다.
###	도메인	무엇을 읽나	인증 주체	플래그	필요 권한(RBAC)
✅  Control plane (ARM)	namespace SKU, Auto-Inflate, partition 수, network 규칙, TLS, retention	Entra (`DefaultAzureCredential`)	`--azure-auth`	`Reader` (namespace scope)<br/>
✅	Metrics plane	Azure Monitor 플랫폼 metric	위와 동일한 Entra 토큰, `MetricsClient` 2.x regional endpoint	(1과 공유)	`Monitoring Reader`<br/>
✅	Data / runtime plane	파티션 runtime 속성 (last enqueued sequence 등)	Event Hubs 자체 연결	`--eh-auth {entra|connstr}`	`Azure Event Hubs Data Receiver` 또는 connection string<br/>
✅	Checkpoint store (선택)	consumer checkpoint offset → lag 계산	Blob	`--checkpoint-store`	`Storage Blob Data Reader`<br/>
왜 분리하나. 도메인 1·2는 Entra 토큰 하나로 묶이지만(둘 다 ARM/Monitor 평면), 도메인 3은 Event Hubs 데이터 평면이라 인증 주체가 근본적으로 다릅니다. 이 경계를 흐리면 "metric은 보이는데 partition은 못 읽는" 부분 실패의 원인을 진단자가 오해하게 됩니다. 도구가 각 도메인의 성공/실패를 독립적으로 보고하도록 설계했습니다.

---
   --azure-auth ──▶ (1) ARM 제어 평면        (2) Azure Monitor metrics 평면 <br/>
   --eh-auth    ──▶ (3) Event Hubs 데이터 평면  (Data Receiver role | connstr)<br/>
   --checkpoint ──▶ (4) Blob checkpoint store   (Storage Blob Data Reader)<br/>

---
### ⚙️ 설치
```bash
python -m venv .venv \&\& source .venv/bin/activate     # Windows: .\\.venv\\Scripts\\Activate.ps1
pip install azure-identity azure-monitor-query azure-eventhub azure-mgmt-eventhub azure-storage-blob
```
각 라이브러리는 선택적으로 로드된다. 특정 라이브러리가 없으면 해당 평면만 `info`로 skip되고 나머지는 정상 동작한다.
> \*\*Windows note.\*\* Python stdout은 Windows에서 cp1252가 기본이라 table 렌더러가 `UnicodeEncodeError`를 낼 수 있다.<br/>
> 실행 전 `$env:PYTHONIOENCODING="utf-8"` 설정이 필요 합니다.

---
### 🧰 사용법
```bash
전체 평면 (Entra) — 가장 일반적
python eh\_diagnose.py `
  --resource-id "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.EventHub/namespaces/<ns>" `
  --azure-auth --eh-auth entra --event-hub orders --format table

개별 인자로 지정
python eh\_diagnose.py `
  --subscription <sub> --resource-group <rg> --namespace ehns-prod-krc-001 `
  --azure-auth --region koreacentral --event-hub orders

connection string으로 데이터 평면 인증 (Entra Data Receiver role이 없을 때)
python eh\_diagnose.py --namespace ehns-prod-krc-001 --azure-auth `
  --eh-auth connstr --eh-connstr "Endpoint=sb://...;SharedAccessKeyName=...;..."

consumer lag까지 (checkpoint store 경로)
python eh\_diagnose.py --resource-id <rid> --azure-auth --event-hub orders `
  --checkpoint-store "https://<st>.blob.core.windows.net/eh-checkpoints" `
  --consumer-group "$Default" --lag-warn 5000 --lag-crit 50000

metric만 빠르게 (데이터 평면 생략)
python eh\_diagnose.py --resource-id <rid> --azure-auth --region koreacentral --eh-auth none

JSON 출력 + CI용 exit code (crit=2, warn=1)
python eh\_diagnose.py --resource-id <rid> --azure-auth --format json --exit-code

'--region`을 생략하면 `--azure-auth`로 조회한 namespace `location`에서 자동 유도한다.
```
---
### 🧰 Diagnostic Rules
eh_diagnose는 adx_diagnose와 동일하게 단일 임계값 테이블 기반(rule-driven) 으로 동작합니다.<br/>
모든 진단 결과는 다음 두 규칙 세트에서 생성됩니다.<br/>
  * METRIC_RULES<br/>
  * DERIVED_DEFAULTS<br/>
조직 표준에 맞춰 임계값만 조정하면 진단 정책을 쉽게 변경할 수 있습니다.
#### Metric-Based Rules (METRIC_RULES)

| Category          | Metric               | Aggregation | Warning | Critical | Meaning / Recommendation |
|-------------------|----------------------|-------------|---------|----------|--------------------------|
| throttling        | ThrottledRequests    | Total       | > 1     | > 100    | Capacity pressure. Enable Auto-Inflate or increase TU/PU capacity. |
| server_errors     | ServerErrors         | Total       | > 1     | > 50     | Service-side failures. Correlate with throttling events. |
| user_errors       | UserErrors           | Total       | > 100   | > 1000   | Client-side issues such as authentication, entity names, or request format. |
| quota_errors      | QuotaExceededErrors  | Total       | > 1     | > 50     | Quota exhaustion. Compare Size and ActiveConnections against tier limits. |
| capture_backlog   | CaptureBacklog       | Average     | > 1     | > 1000   | Capture processing backlog. Verify target Storage permissions and throughput. |
> [!TIP] 위 규칙은 Azure Monitor Metrics 기반으로 평가됩니다.

#### Derived Rules (DERIVED_DEFAULTS)
런타임 데이터 및 메트릭을 조합하여 계산되는 규칙입니다.<br/>

| Category         | Signal                    | Warning | Critical | Formula / Notes |
|------------------|----------------------------|---------|----------|-----------------|
| backlog          | Egress / Ingress Ratio     | < 0.90  | < 0.50   | OutgoingMessages / IncomingMessages (window total) |
| partition_skew   | Retained Event Imbalance   | ≥ 2.0x  | ≥ 5.0x   | max / mean retained events per partition |
| connections      | Connection Saturation      | ≥ 80%   | ≥ 95%    | ActiveConnections(max) / MaxConnections |
| consumer_lag     | Worst Partition Lag        | ≥ lag-warn | ≥ lag-crit | last_enqueued_seq - checkpoint_seq |
> [!TIP] consumer_lag는 Azure Monitor Metric이 아닌 Runtime + Checkpoint Store 정보를 활용하여 계산됩니다.

#### Configuration Audit Rules
Control Plane 설정을 검사하여 운영 모범 사례(Best Practice) 위반 여부를 확인합니다.

| Condition | Severity | Rationale |
|-----------|----------|-----------|
| Auto-Inflate Disabled (Standard Tier) | Warning | Traffic spikes may cause throttling. |
| publicNetworkAccess = Enabled | Warning | Private Endpoint is recommended for production environments. |
| minimumTlsVersion < 1.2 | Critical | Weak TLS version permitted. |
| disableLocalAuth = false (SAS enabled) | Info | Entra ID-only authentication is recommended. |
| Partition Count Reported | Info | Basic/Standard tiers cannot change partition count after creation. |

---
### 🧰 Output Schema
eh_diagnose는 PostgreSQL 진단 도구(pg_diagnose)와 동일한 스키마를 사용합니다.<br/>
## Severity Levels

| Severity | Meaning |
|-----------|-----------|
| critical | Immediate action required |
| warning | Investigation and optimization recommended |
| info | Informational finding |
| ok | Healthy / No issues detected |
Sample Output
```bash
{
  "tool": "eh_diagnose",
  "version": "1.0.0",
  "namespace": "ehns-prod-krc-001",
  "event_hub": "orders",
  "generated_at": "2026-07-01T00:36:51+00:00",
  "window": "last 60m",
  "checks": [
    {
      "category": "throttling",
      "severity": "critical",
      "title": "Requests throttled (capacity pressure)",
      "detail": "ThrottledRequests (total)=1500 over window. Enable Auto-Inflate or increase throughput capacity.",
      "evidence": {
        "metric": "ThrottledRequests",
        "value": 1500,
        "warn": 1,
        "crit": 100
      }
    }
  ],
  "worst_severity": "critical"
}
```
### 🧰 Diagnostic Categories
| Category | Checks |
|-----------|-----------|
| Capacity | • Throughput Unit utilization<br>• Auto-Inflate configuration<br>• Ingress/Egress utilization |
| Throttling | • Throttled Requests<br>• Server Busy exceptions<br>• Quota exceeded conditions |
| Consumer Lag | • Partition-level lag<br>• Consumer Group health<br>• Stale checkpoints |
| Availability | • Receiver connectivity status<br>• Sender connectivity status<br>• Error rate trends |
| Capture | • Capture enabled status<br>• Storage write failures |
| Partition | • Hot partitions<br>• Partition imbalance |
| Checkpoint | • Checkpoint update delays<br>• Dead consumer detection |
### 🧰 Consumer Lag Detection
Consumer Lag은 Event Hubs 환경에서 가장 중요한 운영 지표 중 하나입니다.<br/>
eh_diagnose는 Azure Monitor Metrics에 직접 의존하지 않고 Runtime + Checkpoint 기반 방식(B)을 우선적으로 사용합니다.<br/>
### 🧰 Approach A — Azure Monitor Consumer Lag Metric
제한사항
AMQP consumer는 해당 consumer group에 활성 receiver가 있을 때만 lag이 emit되고,
Kafka consumer group은 마지막 offset commit이 retention보다 오래되면 더 이상
emit되지 않습니다. 즉 "lag metric 부재"가 곧 "lag 없음"이 아니라 "receiver가
죽었거나 commit이 오래됨"일 수 있습니다. metric 이름·가용성이 tier/시점별로 다를 수
있으므로 이 도구는 A를 하드코딩하지 않습니다.
### 🧰 Approach B — Runtime + Checkpoint Calculation (Recommended)
eh_diagnose 기본 방식입니다. Checkpoint Store가 제공되면 실제 데이터를 기반으로 Consumer Lag을 계산합니다.
**계산식**
```bash
lag = last_enqueued_sequence_number (데이터 평면)
− checkpoint blob의 sequencenumber (Blob)
```
(파티션별로 계산)
**Checkpoint blob 경로**
`{fqdn}/{eventhub}/{consumer_group}/checkpoint/{partition_id}`  *(SDK가 소문자화)*
> [!TIP]
> metric이 사라지는 **dead/idle consumer** 상황도 잡아냅니다.
핵심 개선 포인트:
* 공식을 코드블록으로 분리 — 인라인 문장보다 좌변=우변 구조가 즉시 이해됨<br/>
* 경로를 별도 항목으로 — 문장 끝에 붙이면 묻힘<br/>
* dead/idle 장점을 콜아웃(> [!TIP])으로 — Approach B 대비 이점이라 강조 가치가 있음<br/>
* 원문의 백슬래시 이스케이프(last\_enqueued...)는 코드/백틱 안에서는 불필요하니 제거<br/>
---
###
MCP / Azure SRE Agent 통합
기존 `mcp\_server.py`에 thin wrapper 한 줄로 추가된다 (pg/aks/adx와 동일 패턴, `\_\_file\_\_`-anchored path + `sys.executable`).
```python
# mcp\_server.py (발췌)
@mcp.tool()
def eh\_diagnose(resource\_id: str, event\_hub: str = "",
                window\_minutes: int = 60) -> str:
    args = \[sys.executable, os.path.join(HERE, "eh\_diagnose.py"),
            "--resource-id", resource\_id, "--azure-auth",
            "--eh-auth", "entra", "--format", "json",
            "--window-minutes", str(window\_minutes)]
    if event\_hub:
        args += \["--event-hub", event\_hub]
    return subprocess.run(args, capture\_output=True, text=True,
                          encoding="utf-8").stdout
```
Azure SRE Agent에서 "이 namespace가 throttle 나는데 왜?" 류 질의에 그대로 응답하는
진단 surface가 된다. Managed Identity를 SRE Agent에 부여하고 위 4개 RBAC role을
namespace/storage scope로 할당하면 된다.
Event Hubs Namespace
* Azure Event Hubs Data Owner 또는
* Azure Event Hubs Data Receiver
* Azure Event Hubs Data Sender
Azure Monitor
* Monitoring Reader
Checkpoint Storage
* Storage Blob Data Reader
> [!TIP] 정확성 주의
> 메트릭 REST 이름은 Microsoft Learn "Monitoring data reference for Azure Event Hubs" 기준.<br/>
> 조직 namespace에서 `az monitor metrics list-definitions --resource <rid>`로 실제 노출 metric을 한 번 대조 권장.
Azure Monitor metric은 90일 보관, 차트는 한 번에 30일까지만 시각화된다.
partition 수 변경 가능 여부는 tier·시점별로 다르므로(Standard는 불변), 고객
회신 전 Learn에서 재확인.
