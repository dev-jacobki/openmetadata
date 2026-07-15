# Data Quality & Observability

이 문서는 품질 모델, Profiler와 샘플링, 테스트 실행과 결과, Incident, Alert를 설명한다. 예시는 하나의 테이블과 컬럼으로 통일한다.

- Table: `sample_data.ecommerce_db.shopify.dim_customer`
- Column: `customer_id`
- TestCase: `customer_id_not_null`

**목차**

1. 품질 프레임워크
2. Profiler & Data Profile
3. 테스트 실행 & 결과
4. Incident Manager
5. Alert & Notification

## 1. 품질 프레임워크

`TestDefinition`, 데이터 자산, `TestSuite`는 모두 `TestCase`를 구성하는 정보다. 세 객체가 차례대로 실행되는 구조가 아니다. 실행 후에는 해당 `TestCase`에 시간별 `TestCaseResult`가 쌓인다.

![TestDefinition, 데이터 자산, TestSuite가 TestCase를 구성하고 TestCaseResult가 생성되는 관계](quality-model.png)

| 객체 | 실제 예시 | 시스템에서 맡는 역할 |
|---|---|---|
| `TestDefinition` | `columnValuesToBeNotNull` | 재사용할 판정 로직과 파라미터 형식 |
| Table / Column | `dim_customer.customer_id` | 테스트가 읽을 실제 데이터 자산 |
| `TestSuite` | `dim_customer.testSuite` | 관련 TestCase의 실행 단위와 소속 |
| `TestCase` | `customer_id_not_null` | 정의, 대상, 파라미터를 결합한 한 건의 검사 |
| `TestCaseResult` | `Failed`, `nullCount=3` | 실행 시각별 상태와 측정값 |

기본 제공 `TestDefinition`은 총 25개이며, `entityType`에 따라 Table 9개와 Column 16개로 나뉜다.

| 적용 수준 | 개수 | 대표 Definition |
|---|---:|---|
| `TABLE` | 9 | `tableRowCountToBeBetween`, `tableCustomSQLQuery`, `tableDiff` |
| `COLUMN` | 16 | `columnValuesToBeNotNull`, `columnValuesToBeUnique`, `columnValuesToMatchRegex` |

예를 들어 `tableRowCountToBeBetween`은 Table의 `rowCount`를 `minValue`, `maxValue`와 비교하고, `columnValuesToBeNotNull`은 지정한 Column의 `nullCount`가 0인지 검사한다.

`TestDefinition`은 검사 방법이고, 자산별 기준은 `TestCase.parameterValues`에 둔다. 따라서 같은 Definition을 여러 Table이나 Column에 서로 다른 기준값으로 적용할 수 있다.

## 2. Profiler & Data Profile

Profiler는 Table과 Column을 읽어 수치형 Profile을 만들고 시간별로 저장한다. Profile은 현재 데이터 상태를 나타내며 그 자체로 `Success` 또는 `Failed`를 판정하지 않는다. 판정이 필요하면 Profile과 별개로 `TestCase`를 실행한다.

`ProfilerWorkflow`는 `OpenMetadataSource → ProfilerProcessor → MetadataRestSink` 순서로 동작한다. Source가 대상 Table과 연결 정보를 준비하고, Processor가 샘플링과 metric 계산을 수행하며, Sink가 생성된 Profile을 서버에 저장한다.

![Table에 샘플링 설정을 적용하고 TableProfile과 ColumnProfile을 만드는 과정](profiler-sampling.png)

| 저장 위치 | 주요 필드 | 의미 |
|---|---|---|
| `TableProfile` | `rowCount` | Table 전체 행 수 |
| `ColumnProfile` | `nullCount`, `nullProportion` | NULL 개수와 비율 |
| `ColumnProfile` | `distinctCount` | 서로 다른 값의 수 |

### 샘플링 범위

Profiler와 Data Quality Test는 `profileSampleConfig`로 읽을 범위를 제한할 수 있다. 다음 설정은 그림의 `STATIC · 10%`에 해당하며, Profiler 워크플로의 `sourceConfig.config` 안에 들어가는 핵심 부분이다.

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
| `sampleConfigType` | `DYNAMIC` | 전체 행 수와 threshold에 따라 실행 시 범위 결정 |
| `profileSampleType` | `PERCENTAGE` | `profileSample: 10`을 10%로 해석 |
| `profileSampleType` | `ROWS` | `profileSample`을 행 수로 해석 |

- Table, Schema, Database, 워크플로 설정 어디에도 적용할 `profileSampleConfig`가 없으면 별도의 샘플 제한 없이 전체 데이터를 읽는다. 자산에 저장된 더 구체적인 설정이 워크플로 설정보다 먼저 적용될 수 있다.
- 샘플링을 사용해도 `TableProfile.rowCount`는 전체 Table에서 계산한다. `nullCount`, `nullProportion`, `distinctCount`처럼 행을 집계하는 metric과 Test 결과는 실제 샘플 범위를 기준으로 해석한다.
- UI의 `Sample Data`는 값 확인용 예시 행이다. `profileSampleConfig`나 실패 결과의 `failedRowsSample`과 용도가 다르다.

## 3. 테스트 실행 & 결과

UI, YAML, Python SDK는 TestCase를 정의·등록하는 세 가지 입구다. 이후 스케줄이나 실행 요청이 시작되면 `TestSuiteWorkflow`가 Source, Runner, Sink를 순서대로 수행한다.

![UI, YAML, Python SDK에서 TestSuiteWorkflow를 거쳐 TestCaseResult가 저장되는 과정](test-execution-pipeline.png)

| 단계 | 실제 클래스 | 입력과 출력 |
|---|---|---|
| Source | `TestSuiteSource` | Table, TestCase, DatabaseService 연결 정보를 준비 |
| Processor | `TestCaseRunner` | TestCase를 생성·조회하고 Validator 실행 결과를 수집 |
| Sink | `MetadataRestSink` | `TestCaseResult`를 OpenMetadata REST API에 저장 |

UI에서는 Table 상세 화면의 **Data Quality → Add Test**에서 Definition, Column, 파라미터를 선택한다.

다음 YAML은 그림의 YAML 입구에서 중요한 부분만 발췌한 구조 예시다. 실제 실행에서는 쿼리 가능한 DatabaseService의 Table FQN을 사용해야 한다. 서버 주소와 인증을 담는 `workflowConfig.openMetadataServerConfig`는 환경별 값이므로 생략했다.

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

바깥의 `source.type: TestSuite`는 TestSuite Source를 선택하고, 안쪽의 `sourceConfig.config.type: TestSuite`는 TestSuite 파이프라인 설정을 선택한다. 실제 DB 커넥터와 연결 정보는 `entityFullyQualifiedName`으로 찾은 Table의 DatabaseService에서 가져온다. `columnName`은 Table FQN과 결합되어 내부 `entityLink`가 된다. 서버 설정까지 포함한 파일은 `metadata test -c test-suite.yaml`로 실행한다.

다음 코드는 서버와 대상 DatabaseService 연결을 사용할 수 있는 환경에서 같은 TestCase를 Python SDK로 구성하는 핵심 부분이다. `run()`은 별도 엔진이 아니라 동일한 `TestSuiteWorkflow`를 실행해 결과 목록을 반환한다.

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

YAML과 SDK의 `columnValuesToBeNotNull`은 Runner 내부에서 같은 Validator로 연결된다. 아래는 Validator의 `_evaluate_test_condition()`이 상태와 행 수를 계산하는 핵심 발췌다.

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

`results`의 각 항목에는 `testCase`, `testCaseResult`, 선택적인 `failedRowsSample`이 들어 있다. 10% 샘플로 선택된 1,243행 중 NULL이 3개인 경우, 비율을 소수 둘째 자리로 반올림한 `testCaseResult`의 핵심 필드는 다음과 같다.

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

- `result`는 사람이 읽는 설명이고, `testResultValue`는 판정에 사용한 metric 이름과 실제 값이다.
- `passedRows`와 `failedRows`는 `computePassedFailedRowCount: true`이고 해당 Definition이 행 수 계산을 지원할 때 채워진다.
- 샘플링했다면 행 수와 비율은 전체 Table이 아니라 실행된 샘플 범위에 대한 값이다.
- `failedRowsSample`이 저장되더라도 실패 원인 확인용 일부 행일 뿐, 검사 범위를 뜻하지 않는다.
- 서버는 `Failed` 결과를 저장할 때 `incidentId`를 연결한다.

## 4. Incident Manager

Incident는 `Failed TestCaseResult`를 해결할 작업으로 관리한다. Python Runner가 Incident를 직접 만드는 것이 아니라, 서버가 실패 결과를 저장하는 시점에 같은 TestCase의 미해결 Incident를 찾거나 새로 만든다.

![Failed TestCaseResult가 New에서 Ack 또는 Assigned를 거쳐 Resolved로 변경되는 과정](incident-workflow.png)

| 상태 | 시스템 동작 |
|---|---|
| `New` | 새 미해결 Incident를 시작 |
| `Ack` | 문제를 확인한 사용자를 기준으로 작업을 열고 할당 |
| `Assigned` | 지정한 담당자에게 해결 작업을 할당 |
| `Resolved` | 연결된 해결 작업을 종료하고 Incident를 완료 |

실패 결과와 Incident가 연결되는 서버 저장 코드의 핵심 발췌는 다음과 같다.

```java
if (TestCaseStatus.Failed.equals(testCaseResult.getTestCaseStatus())) {
  UUID incidentStateId =
      TestCaseResolutionStatusRepository.getOrCreateIncident(testCase, updatedBy);
  testCaseResult.setIncidentId(incidentStateId);
} else {
  testCaseResult.setIncidentId(null);
}
```

- 같은 TestCase가 다시 실패해도 기존 Incident가 미해결이면 동일한 `stateId`를 이어서 사용한다.
- 이후 상태 기록도 같은 `stateId` 아래에 쌓이므로 하나의 문제에 대한 처리 이력을 볼 수 있다.
- 다음 실행이 `Success`여도 기존 Incident가 자동으로 `Resolved`되지는 않는다. 담당자가 해결 상태를 명시적으로 변경한다.

## 5. Alert & Notification

Incident와 Alert는 직렬 연결이 아니다. `Failed TestCaseResult`가 저장되면 한 경로에서는 Incident를 관리하고, 다른 경로에서는 TestCase의 변경을 `ChangeEvent`로 만들어 `EventSubscription`이 필터링한다.

![Failed TestCaseResult에서 Incident 경로와 ChangeEvent 알림 경로가 분리되는 구조](alert-notification-flow.png)

실패한 TestCase만 알리는 Event Subscription의 핵심 값은 다음과 같다.

| 구분 | 실제 값 | 역할 |
|---|---|---|
| Entity | `testCase` | TestCase 변경 이벤트만 선택 |
| Event Type | `entityUpdated` | 결과 저장으로 TestCase가 갱신된 이벤트 선택 |
| 변경 필드 | `testCaseResult` | 새 실행 결과가 들어온 변경 |
| Action | `GetTestCaseStatusUpdates` | TestCase 상태 변경 필터 사용 |
| Condition | `matchTestResult({'Failed'})` | 결과 상태가 Failed인 이벤트만 통과 |
| Destination | `Slack`, `Webhook`, `Email` | 통과한 이벤트의 전달 채널 |

하나의 Subscription에 여러 Destination을 둘 수 있지만, 필터를 통과하지 않은 이벤트는 어느 채널에도 전달되지 않는다.
