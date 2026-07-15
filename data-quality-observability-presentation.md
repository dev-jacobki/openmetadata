# Data Quality & Observability

이 문서는 품질 모델, Profiler와 샘플링, 테스트 실행과 결과, Incident, Alert를 설명한다. 예시는 하나의 Table과 Column으로 통일한다.

- Table: `sample_data.ecommerce_db.shopify.dim_customer`
- Column: `customer_id`
- TestCase: `customer_id_not_null`

**목차**

1. 품질 프레임워크
2. Profiler & Data Profile
3. 테스트 실행 & 결과
4. Incident Manager
5. Alert & Notification

**이 문서에 등장하는 실행 주체**

| 주체 | 이 범위에서 하는 일 |
|---|---|
| OpenMetadata server | TestSuite·TestCase 같은 메타데이터와 Profile·Result·Incident를 저장하고, 변경 이벤트와 알림 설정을 관리 |
| Python 실행 프로세스 | Profiler 또는 TestSuite workflow를 실제로 실행. UI·스케줄은 배포된 pipeline runner, CLI는 현재 CLI 프로세스, SDK는 호출한 Python 프로세스가 이 역할을 맡음 |
| Source database | runner가 보낸 metric·test SQL에 응답. TestSuite나 TestCase를 저장하지 않음 |

## 1. 품질 프레임워크

`TestDefinition`, 데이터 자산, `TestSuite`는 모두 `TestCase`를 구성하는 정보다. 세 객체가 차례대로 실행되는 구조가 아니다. 실행 후에는 해당 `TestCase`에 시간별 `TestCaseResult`가 쌓인다.

![원본 DB의 Table과 Column, OpenMetadata에 저장되는 품질 객체의 관계](quality-model.png)

| 객체 | 실제 예시 | 저장되는 곳 | 시스템에서 맡는 역할 |
|---|---|---|---|
| `TestDefinition` | `columnValuesToBeNotNull` | OpenMetadata | 재사용할 판정 로직과 파라미터 형식 |
| Table / Column | `dim_customer.customer_id` | 원본 DB의 데이터, OpenMetadata의 자산 메타데이터 | 테스트가 읽을 데이터 대상 |
| `TestSuite` | `sample_data.ecommerce_db.shopify.dim_customer.testSuite` | OpenMetadata | TestCase의 소속과 실행 시 조회할 범위 |
| `TestCase` | `customer_id_not_null` | OpenMetadata | Definition, 대상, 파라미터를 결합한 한 건의 검사 |
| `TestCaseResult` | `Failed`, `nullCount=3` | OpenMetadata | 실행 시각별 상태와 측정값 |

### TestSuite는 무엇인가

`sample_data.ecommerce_db.shopify.dim_customer.testSuite`는 원본 DB 안의 객체가 아니다. FQN(Fully Qualified Name)은 OpenMetadata에서 엔터티를 중복 없이 식별하는 전체 이름이다. TestSuite 이름이 Table FQN과 비슷한 이유는 OpenMetadata server가 Basic TestSuite의 FQN을 `<Table FQN>.testSuite`로 만들기 때문이다.

1. 사용자가 해당 Table에 첫 TestCase를 등록한다.
2. OpenMetadata server가 이 Table의 Basic TestSuite를 조회한다.
3. 없으면 server가 Basic TestSuite를 자동 생성하고 TestCase를 연결한다.
4. 나중에 TestSuite workflow가 실행되면 Source가 이 suite에 속한 TestCase를 조회하고, Runner가 각 TestCase의 쿼리와 판정을 수행한다.

즉 TestSuite는 테스트 코드를 실행하는 프로세스가 아니라, **어떤 TestCase를 함께 조회하고 실행할지 정하는 OpenMetadata 메타데이터 엔터티**다.

| 구분 | Table 연결 | 이름 | 역할 |
|---|---|---|---|
| `Basic` | 특정 Table 1개 | `<Table FQN>.testSuite` | 해당 Table TestCase의 기본 소속 |
| `Logical` | Table에 직접 연결되지 않음 | 사용자가 지정 | 이미 존재하는 TestCase를 Table 경계와 무관하게 추가 그룹화 |

코드에 남아 있는 `Executable TestSuite`는 Basic TestSuite의 이전 명칭과 호환 API를 뜻한다. 별도의 세 번째 TestSuite 유형으로 이해하지 않는다.

### TestDefinition

기본 제공 `TestDefinition`은 총 25개이며, `entityType`에 따라 Table 9개와 Column 16개로 나뉜다.

| 적용 수준 | 개수 | 대표 Definition |
|---|---:|---|
| `TABLE` | 9 | `tableRowCountToBeBetween`, `tableCustomSQLQuery`, `tableDiff` |
| `COLUMN` | 16 | `columnValuesToBeNotNull`, `columnValuesToBeUnique`, `columnValuesToMatchRegex` |

예를 들어 `tableRowCountToBeBetween`은 Table의 `rowCount`를 `minValue`, `maxValue`와 비교하고, `columnValuesToBeNotNull`은 지정한 Column의 `nullCount`가 0인지 검사한다.

`TestDefinition`은 검사 방법이고, 자산별 기준은 `TestCase.parameterValues`에 둔다. 따라서 같은 Definition을 여러 Table이나 Column에 서로 다른 기준값으로 적용할 수 있다.

## 2. Profiler & Data Profile

Profiler는 Table과 Column을 읽어 수치형 Profile을 만들고 시간별로 저장한다. Profile은 현재 데이터 상태를 나타내며 그 자체로 `Success` 또는 `Failed`를 판정하지 않는다. 판정이 필요하면 Profile과 별개로 `TestCase`를 실행한다.

Profiler pipeline을 **Run Now로 실행하거나 스케줄 시각이 되면** 배포된 pipeline runner가 Python `ProfilerWorkflow`를 시작한다. 로컬에서 `metadata profile -c <config>`를 실행했다면 현재 CLI 프로세스가 같은 workflow를 시작한다. 이 workflow가 다음 세 컴포넌트를 순서대로 연결한다.

![Profiler workflow에서 Source, Processor, Sink가 담당하는 단계](profiler-sampling.png)

| 컴포넌트 | 접속 대상 | 정확한 역할 |
|---|---|---|
| `OpenMetadataSource` | OpenMetadata API와 workflow의 `source.serviceConnection` | OpenMetadata에서 Profile 대상 Table·Database 메타데이터를 조회하고, workflow에 주어진 DB 연결 설정으로 만든 접근 객체와 묶어 Processor에 전달. 원본 행을 읽거나 metric을 계산하지 않음 |
| `ProfilerProcessor` | Source database | 커넥터별 Profiler를 만들고 `profileSampleConfig`를 적용한 SQL로 `rowCount`, `nullCount`, `distinctCount` 등의 metric을 계산 |
| `MetadataRestSink` | OpenMetadata REST API | 계산이 끝난 `TableProfile`과 `ColumnProfile`을 server에 전송. metric을 다시 계산하지 않음 |

여기서 `Source`는 “원본 DB”가 아니라 **workflow의 입력 단계**라는 뜻이다. `OpenMetadataSource`가 대상 Table과 DB 접근 정보를 Processor에 넘기면, `ProfilerProcessor`가 Profile을 계산하고, `MetadataRestSink`가 그 결과를 server로 보낸다. `ProfilerWorkflow`는 이 전달 순서를 관리한다. Source database에는 Profile 결과를 쓰지 않는다.

| 저장 객체 | 주요 필드 | 의미 |
|---|---|---|
| `TableProfile` | `rowCount` | Table 전체 행 수 |
| `ColumnProfile` | `nullCount`, `nullProportion` | NULL 개수와 비율 |
| `ColumnProfile` | `distinctCount` | 서로 다른 값의 수 |

### 샘플 관련 세 용어

세 용어는 생성 시점과 사용 목적이 서로 다르다.

| 용어 | 누가·언제 만들거나 적용하는가 | 사용 위치와 역할 |
|---|---|---|
| UI `Sample Data` | 별도 Sample Data 수집 단계의 `SamplerProcessor`가 원본 DB의 예시 행을 읽음. `sampleDataCount`가 요청 행 수를 정하며 기본값과 현재 최대치는 50 | Table의 **Sample Data** 화면에서 실제 값과 형태를 확인하는 미리보기 |
| `profileSampleConfig` | Profiler 또는 TestSuite workflow가 metric·test 쿼리를 만들기 전에 적용 | Profile 계산과 Test 실행이 읽을 입력 범위를 비율 또는 행 수로 제한하는 설정 |
| `failedRowsSample` | Test가 `Failed`이고 `computePassedFailedRowCount: true`일 때 이를 지원하는 Validator가 실패 행 중 최대 50개를 추가 조회 | Test 결과 화면에서 어떤 행이 조건을 어겼는지 확인하는 진단 데이터 |

각 흐름을 분리해서 보면 다음과 같다.

```text
Profile/Test 계산:  원본 행 -- profileSampleConfig --> 실행 대상 행 -- metric·판정 --> Profile 또는 TestCaseResult
UI 값 미리보기:    원본 행 -- sampleDataCount ------> Sample Data -------> Table 화면
실패 원인 확인:    Failed TestCaseResult ---------> failedRowsSample ---> Test 결과 화면
```

`Sample Data`와 `failedRowsSample`은 행 데이터이고, `profileSampleConfig`는 행 데이터가 아니라 **쿼리 범위를 정하는 설정값**이다. `failedRowsSample`은 전체 실패 건수나 테스트 입력 범위를 뜻하지 않는다. Data Quality workflow에 `profileSampleConfig`가 있으면 실패 행 샘플도 그 실행 범위 안에서 추출된다. 다음 실행이 `Success`이면 server는 이전 `failedRowsSample`을 제거한다.

### profileSampleConfig 설정

다음은 Profiler workflow의 `sourceConfig.config`에 넣는 설명용 설정 예시다. 저장소 코드의 발췌가 아니다.

```yaml
sourceConfig:
  config:
    type: Profiler
    profileSampleConfig:
      sampleConfigType: STATIC
      config:
        profileSample: 10
        profileSampleType: PERCENTAGE
```

| 설정 | 허용 값 | 실행 범위 |
|---|---|---|
| `sampleConfigType` | `STATIC` | 고정 비율 또는 고정 행 수 |
| `sampleConfigType` | `DYNAMIC` | `config.smartSampling: true`이면 전체 `rowCount`에 따른 내장 구간을 사용. `false`이면 `config.thresholds[].rowCountThreshold`에 맞는 범위를 사용 |
| `profileSampleType` | `PERCENTAGE` | `profileSample: 10`을 10%로 해석 |
| `profileSampleType` | `ROWS` | `profileSample`을 행 수로 해석 |

- 적용 우선순위는 실행 설정의 `tableConfig` → `schemaConfig` → `databaseConfig`, 그다음 자산에 저장된 Table의 `tableProfilerConfig` → DatabaseSchema의 `databaseSchemaProfilerConfig` → Database의 `databaseProfilerConfig` → workflow 기본값 순서다. 어디에도 `profileSampleConfig`가 없으면 별도 샘플 제한 없이 전체 데이터를 읽는다.
- 샘플링을 사용해도 `TableProfile.rowCount`는 전체 Table에서 계산한다. `nullCount`, `nullProportion`, `distinctCount`처럼 행을 집계하는 metric과 Test 결과는 실제 샘플 범위를 기준으로 해석한다.

## 3. 테스트 실행 & 결과

TestCase를 **등록하는 단계**와 **실행하는 단계**는 시스템 안에서 구분된다. UI에서는 Add Test로 먼저 등록하고 나중에 실행한다. YAML과 SDK는 한 번의 workflow 안에서 누락된 TestCase를 등록한 직후 이어서 실행할 수 있다.

| 시점 | 실행 주체 | 동작 |
|---|---|---|
| UI 등록 | 사용자 | **Data Quality → Add Test**에서 TestCase를 server에 미리 저장 |
| YAML·SDK 등록 | `TestCaseRunner` | workflow 시작 시 설정에 있는 TestCase가 없으면 server에 생성 |
| UI Run Now·스케줄 | 배포된 pipeline runner | Python `TestSuiteWorkflow`를 시작 |
| CLI·SDK 실행 | 현재 CLI 프로세스 또는 SDK 호출 프로세스 | `metadata test -c ...` 또는 SDK `run()`이 같은 workflow를 직접 시작 |
| 판정 | `TestCaseRunner` | Source database에 metric·test SQL을 실행하고 Validator로 상태를 결정 |
| 결과 저장 | `MetadataRestSink`와 OpenMetadata server | Sink가 REST API로 결과를 보내고 server가 시계열 `TestCaseResult`로 저장 |

![TestCase 등록과 TestSuiteWorkflow 실행이 분리되고 Runner만 원본 DB를 조회하는 구조](test-execution-pipeline.png)

`TestSuiteWorkflow` 내부의 역할은 다음과 같다.

| 단계 | 실제 클래스 | 입력과 출력 |
|---|---|---|
| Source | `TestSuiteSource` | TestSuite에 속한 Table·TestCase와 DatabaseService 연결 정보를 조회 |
| Processor | `TestCaseRunner` | YAML·SDK 설정의 누락 TestCase를 생성하고, 원본 DB 쿼리와 Validator 실행 결과를 수집 |
| Sink | `MetadataRestSink` | `TestCaseResult`와 선택적인 `failedRowsSample`을 OpenMetadata REST API로 전송 |

UI에서는 Table 상세 화면의 **Data Quality → Add Test**에서 Definition, Column, 파라미터를 선택한다. 이 동작은 TestCase만 등록하며, 실제 검사는 이후 Run Now 또는 스케줄이 시작한다. 반면 아래 YAML과 SDK 예시는 workflow가 시작될 때 TestCase를 생성·조회한 뒤 같은 실행에서 바로 검사한다.

### YAML로 등록하고 실행하기

다음 YAML은 중요한 필드만 남긴 설명용 구성 예시다. 저장소 파일의 발췌가 아니다. 실제 실행에서는 쿼리 가능한 DatabaseService의 Table FQN을 사용해야 한다. 서버 주소와 인증을 담는 `workflowConfig.openMetadataServerConfig`는 환경별 값이므로 생략했다.

```yaml
source:
  type: TestSuite
  serviceName: sample_data
  sourceConfig:
    config:
      type: TestSuite
      entityFullyQualifiedName: sample_data.ecommerce_db.shopify.dim_customer
      profileSampleConfig:
        sampleConfigType: STATIC
        config:
          profileSample: 10
          profileSampleType: PERCENTAGE

processor:
  type: orm-test-runner
  config:
    testCases:
      - name: customer_id_not_null
        testDefinitionName: columnValuesToBeNotNull
        columnName: customer_id
        computePassedFailedRowCount: true

sink:
  type: metadata-rest
  config: {}
```

바깥의 `source.type: TestSuite`는 TestSuite Source를 선택하고, 안쪽의 `sourceConfig.config.type: TestSuite`는 TestSuite pipeline 설정을 선택한다. Source는 `entityFullyQualifiedName`으로 OpenMetadata의 Table을 찾고, 그 Table의 DatabaseService 연결 정보를 가져온다. Runner는 이 연결로 원본 DB에 접속한다. `columnName`은 Table FQN과 결합되어 `<#E::table::sample_data.ecommerce_db.shopify.dim_customer::columns::customer_id>`라는 `entityLink`가 된다. `entityLink`는 TestCase가 어느 Table 또는 Column을 검사하는지 가리키는 OpenMetadata 내부 참조 문자열이다. 서버 설정까지 포함한 파일은 `metadata test -c test-suite.yaml`로 실행한다.

### Python SDK로 등록하고 실행하기

다음 코드는 같은 TestCase를 Python SDK로 구성하는 예시다. `run()`은 SDK 내부에서 `TestSuiteWorkflow`를 생성하고 실행한 뒤 결과 목록을 반환한다.

참고 위치: `ingestion/src/metadata/sdk/examples/dq_as_code_example.py:19-60`의 API 사용 예시를 문서 대상에 맞게 축약. `run()` 구현은 `ingestion/src/metadata/sdk/data_quality/runner.py:240-258`.

```python
from metadata.sdk.data_quality import ColumnValuesToBeNotNull, TestRunner

runner = TestRunner.for_table(
    "sample_data.ecommerce_db.shopify.dim_customer"
)
test = (
    ColumnValuesToBeNotNull(column="customer_id")
    .with_name("customer_id_not_null")
    .with_compute_row_count(True)
)
runner.add_test(test)
results = runner.run()
```

### 판정 코드

YAML과 SDK의 `columnValuesToBeNotNull`은 Runner 내부에서 같은 Validator로 연결된다. 아래는 Validator가 상태와 행 수를 계산하는 실제 코드의 핵심 발췌다.

소스 위치: `ingestion/src/metadata/data_quality/validations/column/base/columnValuesToBeNotNull.py:129-141`.

```python
null_count = metric_values[Metrics.nullCount.name]
total_rows = metric_values.get(Metrics.rowCount.name)

matched = null_count == 0
failed_count = null_count
passed_count = total_rows - null_count if total_rows else 0

return {
    "matched": matched,
    "passed_rows": passed_count,
    "failed_rows": failed_count,
    "total_rows": total_rows,
}
```

`matched=True`이면 `Success`, `False`이면 `Failed`다. 즉 쿼리가 정상 실행된 뒤 `nullCount=3`을 얻었다면 실행 오류가 아니라 데이터가 규칙을 통과하지 못한 것이다.

### TestCaseResult 읽기

| 상태 | 의미 |
|---|---|
| `Success` | 실행을 완료했고 판정 조건이 참 |
| `Failed` | 실행을 완료했지만 판정 조건이 거짓 |
| `Aborted` | TestCase의 쿼리 또는 metric 계산 오류로 판정을 완료하지 못함 |
| `Queued` | 실행 대기 중 |

10% 샘플로 선택된 1,243행 중 NULL이 3개인 경우의 설명용 결과 예시는 다음과 같다. 저장소 코드의 발췌가 아니다.

```yaml
timestamp: 1763078400000
testCaseStatus: Failed
result: "Found nullCount=3. It should be 0"
testResultValue:
  - name: nullCount
    value: "3"
passedRows: 1240
failedRows: 3
passedRowsPercentage: 99.76
failedRowsPercentage: 0.24
```

| 결과 필드 | 이 예시에서 읽는 값 |
|---|---|
| `timestamp` | 이 결과가 실행된 시각의 epoch milliseconds |
| `testCaseStatus` | 쿼리는 완료됐지만 조건이 거짓이므로 `Failed` |
| `result` | 사람이 읽는 판정 설명 |
| `testResultValue` | 판정에 사용한 metric 이름 `nullCount`와 실제 값 `3` |
| `passedRows`, `failedRows` | 실행 대상 1,243행 중 통과 1,240행, 실패 3행 |
| `passedRowsPercentage`, `failedRowsPercentage` | 실행 대상 행을 분모로 계산한 99.76%, 0.24% |

`passedRows`와 `failedRows`는 `computePassedFailedRowCount: true`이고 해당 Definition이 행 수 계산을 지원할 때 채워진다. 샘플링했다면 행 수와 비율은 전체 Table이 아니라 실행된 샘플 범위에 대한 값이다.

Validator가 만든 결과에는 선택적으로 `failedRowsSample`이 붙을 수 있다. Sink는 먼저 `TestCaseResult`를 저장한 뒤 이 샘플을 별도 API로 저장한다. OpenMetadata server가 실패 결과를 받으면 저장 과정에서 `incidentId`도 결과에 연결한다.

## 4. Incident Manager

Incident는 `Failed TestCaseResult`를 해결할 작업으로 관리한다. Python Runner가 Incident를 직접 만들지 않는다. Incident 생성 여부는 **각 TestCaseResult를 받는 OpenMetadata server의 REST 요청 처리 중에 동기적으로 결정**되며, 별도 주기 작업이 아니다.

![Failed TestCaseResult로 생성된 Incident의 New, Ack, Assigned, Resolved 상태 전이](incident-workflow.png)

실패 한 건이 저장될 때의 순서는 다음과 같다.

1. `TestCaseRunner`가 Validator 결과를 `MetadataRestSink`에 전달한다.
2. Sink가 해당 TestCase의 결과를 OpenMetadata REST API로 POST한다.
3. Server의 `addTestCaseResult()`가 같은 TestCase의 미해결 Incident를 조회한다.
4. 상태가 `Failed`이면 기존 Incident의 ID를 재사용하거나 새 Incident를 만들고, 그 값을 결과의 `incidentId`에 넣는다.
5. Server가 `TestCaseResult`를 시계열 레코드로 저장한 뒤 TestCase의 최신 상태도 갱신한다.
6. Sink는 결과 저장이 끝난 뒤 선택적인 `failedRowsSample`을 별도 요청으로 전송한다.

3~5번의 호출 순서는 `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/TestCaseResultRepository.java:86-112`, 아래 분기 코드는 같은 파일의 `:220-228`이다.

코드의 `incidentStateId` 또는 `stateId`는 Incident 상태 이력을 묶는 UUID이고, `TestCaseResult`에는 같은 UUID가 `incidentId` 필드로 저장된다.

```java
if (TestCaseStatus.Failed.equals(testCaseResult.getTestCaseStatus())) {
  UUID incidentStateId =
      TestCaseResolutionStatusRepository.getOrCreateIncident(testCase, updatedBy);
  testCaseResult.setIncidentId(incidentStateId);
} else {
  testCaseResult.setIncidentId(null);
}
```

| 상태 | 누가 바꾸는가 | 시스템 동작 |
|---|---|---|
| `New` | Server가 미해결 Incident가 없는 `Failed`를 저장할 때 | 새 Incident를 시작 |
| `Ack` | 사용자 | 문제를 확인하고 자신에게 작업을 연결 |
| `Assigned` | 사용자 | 지정한 담당자에게 해결 작업을 할당 |
| `Resolved` | 사용자 | 해결 작업을 종료하고 Incident를 완료 |

- 같은 TestCase가 다시 실패해도 기존 Incident가 미해결이면 동일한 `incidentId`를 이어서 사용한다.
- 이후 상태 기록도 같은 Incident ID 아래에 쌓이므로 하나의 문제에 대한 처리 이력을 볼 수 있다.
- `Success`와 `Aborted` 결과에는 새 Incident를 연결하지 않는다. 기존 Incident가 `Resolved`된 뒤 다시 `Failed`가 발생하면 새 `incidentId`로 시작한다.
- 다음 실행이 `Success`여도 기존 Incident가 자동으로 `Resolved`되지는 않는다. 담당자가 해결 상태를 명시적으로 변경한다.

## 5. Alert & Notification

Incident가 Alert를 호출하는 구조는 아니다. 다만 같은 결과 저장 요청 안에서 server는 Incident를 연결하고 결과를 저장한 뒤 TestCase의 최신 `testCaseResult`를 갱신한다. 이어서 HTTP 응답을 만드는 과정에서 server 내부의 `ChangeEventHandler`가 이 결과를 `testCase / entityUpdated` ChangeEvent로 저장한다. 여기까지는 동기 처리다. 그 이후 server 내부 백그라운드 컴포넌트인 Event Subscription scheduler가 설정된 `pollInterval`마다 아직 처리하지 않은 이벤트를 읽고 필터를 평가한다. 통과한 이벤트만 server 내부 Destination publisher가 Slack, Webhook 또는 Email로 보낸다.

```text
TestCaseResult POST
  → Incident 조회·생성 및 incidentId 연결
  → Result 저장·TestCase 최신 결과 갱신
  → ChangeEvent(testCase / entityUpdated) 저장     [여기까지 동기]
  ⇢ Event Subscription scheduler 조회·필터        [여기부터 비동기]
  → Destination 전송
```

![Failed TestCaseResult의 Incident 동기 처리와 Event Subscription 비동기 알림 경로](alert-notification-flow.png)

실패한 TestCase만 알리는 Event Subscription의 핵심 값은 다음과 같다.

| 구분 | 실제 값 | 역할 |
|---|---|---|
| Entity | `testCase` | TestCase 변경 이벤트만 선택 |
| Event Type | `entityUpdated` | 결과 저장으로 TestCase가 갱신된 이벤트 선택 |
| 변경 필드 | `testCaseResult` | 새 실행 결과가 들어온 변경 |
| Action | `GetTestCaseStatusUpdates` | TestCase 상태 변경 필터 사용 |
| Condition | `matchTestResult({'Failed'})` | 결과 상태가 Failed인 이벤트만 통과 |
| Destination | `Slack`, `Webhook`, `Email` | 통과한 이벤트의 전달 채널 |

하나의 Subscription에 여러 Destination을 둘 수 있지만, 필터를 통과하지 않은 이벤트는 어느 채널에도 전달되지 않는다. **이 문서의 실패 TestCase Subscription에서는** Incident 상태가 아니라 TestCase ChangeEvent가 알림 입력이다. 따라서 Incident 연결은 결과 저장 요청 안에서 즉시 끝나지만, 알림 전달은 scheduler가 ChangeEvent를 읽은 뒤 비동기로 일어난다.
