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

> `소스:`가 붙은 블록은 저장소 실제 코드에서 핵심 줄을 발췌·축약했다. 한국어 주석과 명시적인 `...` 생략 표시는 설명을 위해 추가했다. 나머지 YAML·Python·결과 블록은 설명용 예시다.

이 범위의 실행 주체는 세 가지다.

- **OpenMetadata server**: TestSuite·TestCase 메타데이터, Profile·Result·Incident, ChangeEvent와 Alert 설정을 저장한다.
- **Python 실행 프로세스**: Profiler 또는 TestSuite workflow를 실행한다. UI·스케줄은 배포된 pipeline runner가, CLI·SDK는 호출한 Python 프로세스가 시작한다.
- **Source database**: runner가 보낸 metric·test SQL에 응답한다. TestSuite나 TestCase를 저장하지 않는다.

---

## 1. 품질 프레임워크

> **담당 화면**: Test Library · Data Quality → Test Suites
>
> **경로**: `/test-library` · `/data-quality/test-suites`

`TestDefinition + 검사 대상 + parameterValues`로 `TestCase`를 만들고, `TestSuite`는 만들어진 TestCase를 포함한다. 사진의 `TestDefinition → TestSuite → TestCase`는 생성 순서가 아니다.

![TestDefinition, 검사 대상, 파라미터로 TestCase를 만들고 Table Suite와 Bundle Suite가 이를 포함하는 관계](quality-model.png)

- `TestDefinition`: 재사용할 판정 로직과 파라미터 형식. 예: `columnValuesToBeNotNull`
- `TestCase`: Definition, 대상, 기준값을 결합한 실행 가능한 검사 한 건. 예: `customer_id_not_null`
- `TestCaseResult`: TestCase를 실행할 때마다 저장되는 상태와 측정값

### Table Suite와 Bundle Suite

| UI 명칭 | 백엔드 표현 | 역할 |
|---|---|---|
| `Table Suite` | `basic=true` | 특정 Table의 Table·Column TestCase가 자동으로 속하는 기본 Suite |
| `Bundle Suite` | `basic=false` | 하나 또는 여러 Table의 기존 TestCase를 사용자가 추가로 묶는 Suite |

`sample_data.ecommerce_db.shopify.dim_customer.testSuite`는 DB 객체가 아니라 OpenMetadata 엔터티다. Table Suite의 FQN을 `<Table FQN>.testSuite`로 만들기 때문에 Table 이름처럼 보인다. TestCase는 Table Suite 하나에 기본 소속되고, 필요한 Bundle Suite에 추가로 속할 수 있다.

### 실제 코드: TestCase → Table Suite 관계 저장

소스:

- `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/TestCaseRepository.java:686-697`
- `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/TestCaseRepository.java:710-724`
- `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/TestCaseRepository.java:939-951`

```java
// [1] 요청의 parameterValues가 TestDefinition 형식과 맞는지 먼저 검증한다.
validateTestParameters(
    test.getParameterValues(),
    testDefinition.getParameterDefinition(),
    testDefinition.getTestPlatforms(),
    testDefinition);

// [2] 이 helper가 대상 Table의 Table Suite를 반환하며, 없으면 생성한다.
var testSuite = getOrCreateTestSuite(test, table);
test.setTestSuite(testSuite);

// [3] 실제 관계 방향은 TestSuite --CONTAINS--> TestCase다.
addRelationship(
    test.getTestSuite().getId(), test.getId(), TEST_SUITE, TEST_CASE, Relationship.CONTAINS);

// [4] 사용한 TestDefinition도 TestCase와 별도 관계로 저장한다.
addRelationship(
    test.getTestDefinition().getId(), test.getId(),
    TEST_DEFINITION, TEST_CASE, Relationship.CONTAINS);
```

> **주의**: 이 코드는 Table Suite 연결이다. Bundle Suite는 사용자가 기존 TestCase를 선택해 추가하는 별도 관계다. 검증이 실패하면 TestCase와 Table Suite 모두 생성되지 않는다.

### TestDefinition과 Test Library

기본 제공 TestDefinition은 25개이며 Table 9개, Column 16개로 나뉜다.

| 적용 수준 | 대표 Definition |
|---|---|
| `TABLE` | `tableRowCountToBeBetween`, `tableCustomSQLQuery`, `tableDiff` |
| `COLUMN` | `columnValuesToBeNotNull`, `columnValuesToBeUnique`, `columnValuesToMatchRegex` |

**Test Library는 TestDefinition 목록 화면**이다. 여기에 규칙이 있다는 것만으로 검사가 실행되지는 않는다. Definition을 선택해 자산별 TestCase를 만들고 pipeline 실행 대상에 포함해야 한다.

---

## 2. Profiler & Data Profile

> **결과 화면**: Table → Data Observability → Table Profile / Column Profile
>
> **경로**: `/table/<Table FQN>/profiler/table-profile` · `/table/<Table FQN>/profiler/column-profile`
>
> **실행 관리**: Settings → Services → Databases → Service → Agents

`Profiler`는 metric을 계산하는 workflow이고, `Table Profile`·`Column Profile`은 저장된 결과를 조회하는 화면이다. Profile 화면을 여는 동작은 Profiler를 실행하지 않는다.

![Profiler workflow에서 Source, Processor, Sink가 담당하는 단계](profiler-sampling.png)

| 컴포넌트 | 입력 → 출력 |
|---|---|
| `OpenMetadataSource` | OpenMetadata의 대상 Table·Database 메타데이터 → DB 접근 객체가 포함된 작업 단위 |
| `ProfilerProcessor` | 작업 단위 → 원본 DB에서 계산한 `TableProfile`·`ColumnProfile` |
| `MetadataRestSink` | 계산된 Profile → OpenMetadata REST API 저장 |

### 실제 코드: Source → Processor → Sink

소스:

- `ingestion/src/metadata/workflow/profiler.py:67-75`
- `ingestion/src/metadata/workflow/ingestion.py:169-176`

```python
# [1] Source와 뒤에 실행할 Processor → Sink 순서를 구성한다.
source_class = self._get_source_class()
self.source = source_class.create(self.config.model_dump(exclude_unset=True), self.metadata)
profiler_processor = self._get_profiler_processor()
sink = self._get_sink()
self.steps = (profiler_processor, sink)

# [2] Source의 반환값을 첫 입력으로 사용한다.
for record in self.source.run():
    processed_record = record
    for step in self.steps:
        # [3] 이전 step의 반환값이 다음 step.run()의 입력으로 전달된다.
        if processed_record is not None and isinstance(step, (Processor, Stage, Sink)):
            processed_record = step.run(processed_record)
```

> **주의**: `OpenMetadataSource`는 원본 행을 계산하지 않는다. Source가 넘긴 DB 접근 정보로 `ProfilerProcessor`가 metric SQL을 실행한다. `step.run()`이 `None`을 반환하면 뒤 단계로 전달되지 않는다.

### 저장되는 Profile

주요 저장값은 다음과 같다.

- `TableProfile.rowCount`: Table 전체 행 수
- `ColumnProfile.nullCount`, `nullProportion`: NULL 개수와 비율
- `ColumnProfile.distinctCount`: 서로 다른 값의 수

### Sample Data, profileSampleConfig, failedRowsSample

| 용어 | 실제 역할 |
|---|---|
| UI `Sample Data` | Table 화면에서 값과 형태를 확인하는 예시 행. `sampleDataCount`의 기본값은 50이며 설정으로 바꿀 수 있음 |
| `profileSampleConfig` | Profiler와 Test가 계산 전에 읽을 행 범위를 제한하는 설정값 |
| `failedRowsSample` | Failed 원인 확인용 행 데이터. 지원 Validator가 `Failed`일 때 실패 행 중 최대 50개를 저장 |

`Sample Data`와 `failedRowsSample`은 행 데이터다. `profileSampleConfig`는 행 자체가 아니라 **쿼리 범위를 정하는 설정**이다.

### profileSampleConfig 예시

다음은 workflow 설정의 필요한 부분만 남긴 예시다.

```yaml
sourceConfig:
  config:
    type: Profiler
    profileSampleConfig:
      sampleConfigType: STATIC       # [1] 고정 크기 사용
      config:
        profileSample: 10            # [2] 크기 10
        profileSampleType: PERCENTAGE # [3] 전체 행의 10%로 해석
```

- `STATIC`: `PERCENTAGE` 또는 `ROWS`로 고정 범위를 사용한다.
- `DYNAMIC`: smart sampling 내장 구간 또는 `rowCountThreshold`별 설정을 사용한다.
- 우선순위: 실행 설정의 Table → Schema → Database, 자산에 저장된 Table → Schema → Database 설정, workflow 기본값 순서다.

> **주의**: 샘플링해도 `TableProfile.rowCount`는 전체 Table에서 계산한다. `nullCount`, `distinctCount`, Test 결과는 실제 샘플 범위를 기준으로 해석한다.

---

## 3. 테스트 실행 & 결과

> **담당 화면**: Table → Data Observability → Data Quality · Data Quality → Test Cases
>
> **경로**: `/table/<Table FQN>/profiler/data-quality` · `/data-quality/test-cases`

TestCase **등록**과 **실행**은 별도 동작이다.

![UI와 REST 또는 Python CRUD SDK의 사전 등록, pipeline과 YAML 또는 TestRunner의 실행 경로](test-execution-pipeline.png)

| 방법 | Server에 등록되는 시점 | 같은 호출에서 검사 실행 |
|---|---|---|
| UI **Add Test** / REST `POST` | 신규 TestCase 생성 요청 시 | 안 함 |
| REST `PUT` / `TestCases.create()` | create-or-update 요청 시 | 안 함 |
| YAML `metadata test -c` | workflow 시작 시 누락 Case 생성 | 실행함 |
| DQ SDK `TestRunner.run()` | 누락 Case 생성, 기존 Case 일부 필드 갱신이 기본값 | 실행함 |

### UI로 사전 등록: Add Test → REST POST

소스:

- `openmetadata-ui/src/main/resources/ui/src/components/DataQuality/AddDataQualityTest/components/TestCaseFormV1.tsx:824-830`
- `openmetadata-ui/src/main/resources/ui/src/rest/index.ts:17-21`
- `openmetadata-ui/src/main/resources/ui/src/rest/testAPI.ts:138, 184-190`

```typescript
// [1] Form 값을 name, testDefinition, entityLink, parameterValues로 변환한다.
const testCaseObj = createTestCaseObj(values);

// [2] createTestCase()에 객체를 넘긴다. 이 await는 등록 응답을 기다린다.
const createdTestCase = await createTestCase(testCaseObj);

const testCaseUrl = '/dataQuality/testCases';

// [3] 브라우저가 Server에 신규 TestCase를 POST한다.
const response = await APIClient.post<CreateTestCase, AxiosResponse<TestCase>>(
  testCaseUrl,
  data
);
```

`APIClient`의 base URL이 `/api/v1`이므로 실제 HTTP endpoint는 `POST /api/v1/dataQuality/testCases`다.

> **주의**: Submit 뒤 조건에 따라 실행 pipeline을 추가로 만들 수는 있지만, 위 TestCase POST 자체는 원본 DB에 test SQL을 실행하지 않는다.

### Python CRUD SDK로 사전 등록: REST PUT

다음은 SDK import와 `configure(...)`를 마친 뒤의 핵심 호출이다.

구현 위치:

- `ingestion/src/metadata/sdk/entities/testcases.py:6-16`
- `ingestion/src/metadata/sdk/entities/base.py:144-148`
- `ingestion/src/metadata/ingestion/ometa/ometa_api.py:521-538`

```python
TestCases.create(CreateTestCaseRequest(
    name="customer_id_not_null",
    testDefinition=FullyQualifiedEntityName("columnValuesToBeNotNull"),
    entityLink=EntityLink(
        "<#E::table::sample_data.ecommerce_db.shopify.dim_customer::columns::customer_id>"
    ),
))  # [1] 내부 create_or_update()가 PUT으로 등록만 한다. Test는 실행하지 않는다.
```

Server는 이 요청에서 Definition·파라미터·대상을 검증하고 TestCase와 Table Suite 관계를 저장한다.

### 등록 후 실행: YAML workflow

아래는 workflow 설정의 핵심 필드 발췌다.

```yaml
source:
  sourceConfig:
    config:
      type: TestSuite
      entityFullyQualifiedName: sample_data.ecommerce_db.shopify.dim_customer
      # testCases: [customer_id_not_null]
      # [1] Server의 Table Suite에서 조회한 기존 Case를 이 이름들로 제한한다.

processor:
  type: orm-test-runner
  config:
    forceUpdate: false
    testCases:
      # [2] 필터 뒤에 병합할 inline 정의다. 누락 Case는 Server에 등록한다.
      - name: customer_id_not_null
        testDefinitionName: columnValuesToBeNotNull
        columnName: customer_id
        computePassedFailedRowCount: true
```

생략한 `source.type`, `source.serviceName`, `sink.config`, 서버 주소와 인증까지 포함한 완전한 파일은 `metadata test -c test-suite.yaml`로 실행한다.

### 실제 코드: YAML 정의 → TestCase 등록·병합

소스: `ingestion/src/metadata/data_quality/processor/test_case_runner.py:155-202`의 핵심 발췌

```python
# [1] Source가 넘긴 실행 목록에 같은 이름이 없는 inline 정의를 고른다.
test_cases_to_create = [
    cli_test_case_definition for cli_test_case_definition in cli_test_cases_definitions
    if cli_test_case_definition.name not in test_case_names
]

for test_case_to_create in test_cases_to_create:
    # [2] Definition·대상·파라미터를 PUT으로 전달해 Server에 존재하게 만든다.
    test_case = self.metadata.create_or_update(
        CreateTestCaseRequest(
            name=test_case_to_create.name,
            testDefinition=FullyQualifiedEntityName(test_case_to_create.testDefinitionName),
            entityLink=EntityLink(entity_link.get_entity_link(...)),
            parameterValues=(
                list(test_case_to_create.parameterValues)
                if test_case_to_create.parameterValues else None
            ),
            computePassedFailedRowCount=test_case_to_create.computePassedFailedRowCount,
            # ... description, displayName, owners 생략
        )
    )
    test_cases.append(test_case)  # [3] 반환된 Case를 이번 실행 목록에 병합한다.
```

`sourceConfig.config.testCases`는 **기존 Case 조회 범위**, `processor.config.testCases`는 그 뒤에 **추가할 inline 정의**다. 둘을 함께 쓰면 inline 정의는 source 필터에 없어도 실행 목록에 병합될 수 있다.

> **주의**: YAML은 등록 전용 manifest가 아니다. 기본 `forceUpdate: false`는 기존 Case 설정을 덮어쓰지 않는다. `true`여도 갱신 범위는 `entityLink`, 비어 있지 않은 `parameterValues`, `computePassedFailedRowCount`이며 이름·Definition·설명은 바꾸지 않는다.

### DQ SDK: `run()`에서 등록하고 실행

참고 위치:

- `ingestion/src/metadata/sdk/examples/dq_as_code_example.py:19-60`
- `ingestion/src/metadata/sdk/data_quality/runner.py:240-258`
- `ingestion/src/metadata/sdk/data_quality/workflow_config_builder.py:83-90, 191-194`

```python
from metadata.sdk.data_quality import ColumnValuesToBeNotNull, TestRunner

runner = TestRunner.for_table("sample_data.ecommerce_db.shopify.dim_customer")
runner.add_test(
    ColumnValuesToBeNotNull(column="customer_id")
    .with_name("customer_id_not_null")
    .with_compute_row_count(True)
)
results = runner.run()  # [1] Case 등록·일부 갱신 → SQL 실행 → 결과 반환
```

`TestRunner`는 기본적으로 같은 이름의 기존 Case도 일부 필드를 갱신한다. 갱신을 끄려면 `for_table()` 직후, `add_test()` 전에 `runner.setup(force_test_update=False)`를 호출한다.

### TestCase 일정과 TestSuite 일정

일정은 TestCase나 TestSuite가 아니라 **TestSuite 타입 Ingestion Pipeline**에 저장된다.

| `sourceConfig.config.testCases` | 실행 범위 |
|---|---|
| 값 없음 | 실행 시점에 Suite에 속한 모든 TestCase. 나중에 추가한 Case도 포함 |
| TestCase 이름 목록 | 목록에 적힌 Case만 실행 |

cron은 `airflowConfig.scheduleInterval`에 저장된다. UI에서 보이는 “TestCase 일정”도 실제로는 선택한 Case 이름을 가진 Suite pipeline이다.

스케줄 시각이나 **Run Now**로 workflow가 시작되면, `TestCaseRunner`가 각 TestCase의 Definition에 연결된 Validator를 실행한다.

### 실제 코드: Runner → REST Sink → Result POST

소스:

- `ingestion/src/metadata/workflow/data_quality.py:42-48`
- `ingestion/src/metadata/data_quality/processor/test_case_runner.py:101-107`
- `ingestion/src/metadata/ingestion/sink/metadata_rest.py:1019-1027`
- `ingestion/src/metadata/ingestion/ometa/mixins/tests_mixin.py:78-81`

```python
# [1] Workflow가 Processor 다음에 Sink가 오도록 순서를 정한다.
self.steps = (test_runner_processor, sink)

# [2] Processor가 각 Case의 SQL·판정을 실행해 Result 목록을 반환한다.
test_results = [
    test_case_result for test_case in openmetadata_test_cases
    if (test_case_result := self._run_test_case(test_case, test_suite_runner))
]
return Either(right=TestCaseResults(test_results=test_results))

# [3] 다음 단계인 MetadataRestSink가 각 Result와 TestCase FQN을 받는다.
for result in record.test_results or []:
    self.metadata.add_test_case_results(
        test_results=result.testCaseResult,
        test_case_fqn=result.testCase.fullyQualifiedName.root,
    )

# [4] 마지막으로 Server의 해당 TestCase 결과 endpoint에 POST한다.
self.client.post(
    f"{self.get_suffix(TestCaseResult)}/{quote(test_case_fqn)}",
    test_results.model_dump_json(),
)
```

> **주의**: TestCase는 OpenMetadata server에 있지만 SQL은 Python Runner가 Source database에 보낸다. Sink는 판정이 끝난 Result만 Server에 저장한다.

### 판정 코드

YAML과 SDK의 `columnValuesToBeNotNull`은 같은 Validator로 연결된다.

소스: `ingestion/src/metadata/data_quality/validations/column/base/columnValuesToBeNotNull.py:129-141`

```python
null_count = metric_values[Metrics.nullCount.name]     # [1] SQL로 계산된 NULL 수
total_rows = metric_values.get(Metrics.rowCount.name) # [2] 실제 실행 범위의 행 수

matched = null_count == 0                             # [3] Definition의 통과 조건
failed_count = null_count
passed_count = total_rows - null_count if total_rows else 0

return {
    "matched": matched,
    "passed_rows": passed_count,
    "failed_rows": failed_count,
    "total_rows": total_rows,
}
```

`matched=True`이면 `Success`, `False`이면 `Failed`다. 쿼리는 정상 실행됐지만 조건이 거짓인 상태가 `Failed`이고, 쿼리나 metric 계산을 끝내지 못한 상태가 `Aborted`다. `Queued`는 실행 대기 상태다.

### TestCaseResult 읽기

10% 샘플에서 선택된 1,243행 중 NULL이 3개인 설명용 예시다.

```yaml
timestamp: 1763078400000       # [1] 실행 시각, epoch milliseconds
testCaseStatus: Failed         # [2] 실행 완료, 판정 조건은 거짓
result: "Found nullCount=3. It should be 0"
testResultValue:
  - name: nullCount            # [3] 판정에 사용한 metric
    value: "3"
passedRows: 1240               # [4] 실행 범위에서 통과한 행
failedRows: 3                  # [5] 실행 범위에서 실패한 행
passedRowsPercentage: 99.76
failedRowsPercentage: 0.24
```

`passedRows`와 `failedRows`는 `computePassedFailedRowCount: true`이고 해당 Definition이 지원할 때만 채워진다. 샘플링했다면 행 수와 비율의 분모는 전체 Table이 아니라 실행한 샘플 범위다.

---

## 4. Incident Manager

> **담당 화면**: Incident Manager · Table → Data Observability → Incidents
>
> **경로**: `/incident-manager` · `/table/<Table FQN>/profiler/incidents`

Incident는 `Failed TestCaseResult`를 해결할 작업으로 관리한다. Python Runner가 Incident를 만드는 것이 아니라, OpenMetadata server가 **결과 POST 요청을 저장하는 중에 동기적으로** 연결한다.

3절의 `MetadataRestSink → add_test_case_results() → client.post()` 호출이 아래 Server 메서드로 들어온다.

![Failed TestCaseResult로 생성된 Incident의 New, Ack, Assigned, Resolved 상태 전이](incident-workflow.png)

### 실제 코드: Result POST → Incident → 시계열 저장

소스:

- `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/TestCaseResultRepository.java:86-112`
- `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/TestCaseResultRepository.java:220-228`
- `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/TestCaseResolutionStatusRepository.java:507-542`

```java
TestCase testCase = Entity.getEntityByName(TEST_CASE, fqn, "", Include.ALL);

// [1] Result를 insert하기 전에 Incident ID를 동기적으로 결정한다.
setTestCaseResultIncidentId(testCaseResult, testCase, updatedBy);

// ... dimension result 처리 생략

// [2] helper가 넣은 incidentId를 TestCaseResult와 함께 시계열 저장한다.
((CollectionDAO.TestCaseResultTimeSeriesDAO) timeSeriesDao)
    .insert(
        testCase.getFullyQualifiedName(), TESTCASE_RESULT_EXTENSION,
        TEST_CASE_RESULT_FIELD, JsonUtils.pojoToJson(testCaseResult),
        testCaseResult.getIncidentId());
```

호출된 `setTestCaseResultIncidentId()` 내부는 다음과 같다.

```java
// helper 내부: Failed면 미해결 Incident를 재사용하고, 없으면 New를 만든다.
if (TestCaseStatus.Failed.equals(testCaseResult.getTestCaseStatus())) {
  UUID incidentStateId =
      TestCaseResolutionStatusRepository.getOrCreateIncident(testCase, updatedBy);
  testCaseResult.setIncidentId(incidentStateId);
} else {
  testCaseResult.setIncidentId(null);
}
```

`getOrCreateIncident()`는 같은 TestCase의 미해결 Incident ID가 있으면 반환하고, 없거나 이미 `Resolved`면 `New` 상태를 만든다. `Success`, `Aborted`, `Queued` 결과에는 새 Incident ID를 넣지 않는다.

### Incident 상태와 담당자

| 상태 | 변경 주체 | 동작 |
|---|---|---|
| `New` | Server | 담당자 없이 Incident 시작 |
| `Ack` | 편집 권한 사용자 | 누른 사용자를 해결 Task 담당자로 연결 |
| `Assigned` | 편집 권한 사용자 | 선택한 User에게 해결 Task 할당 |
| `Resolved` | 편집 권한 사용자 | Incident 완료 |

담당자 지정은 admin 전용이 아니다. Server API는 대상 Table 또는 TestCase에 대한 `EditTests`나 `EditAll` 권한 중 하나를 요구한다.

> **주의**: 같은 TestCase의 미해결 Incident가 있으면 재실패해도 같은 ID를 사용한다. 다음 실행이 `Success`여도 기존 Incident는 자동으로 `Resolved`되지 않는다.

---

## 5. Alert & Notification

> **담당 화면**: Observability → Alerts
>
> **경로**: `/observability/alerts`

Alert는 실행 때마다 생성되는 메시지가 아니라 사용자가 미리 만드는 `EventSubscription` 설정이다. Incident 생성·해결이 Alert를 자동 생성하는 구조는 아니다.

![Failed TestCaseResult의 Incident 동기 처리와 Event Subscription 비동기 알림 경로](alert-notification-flow.png)

### 실제 코드 1: TestCaseResult → TestCase ChangeEvent

소스: `openmetadata-service/src/main/java/org/openmetadata/service/formatter/util/FormatterUtil.java:317-358`의 핵심 발췌

```java
if (entityTimeSeries instanceof TestCaseResult) {
  // [1] POST로 저장한 Result를 testCase의 entityUpdated 이벤트로 표현한다.
  eventType = EventType.ENTITY_UPDATED;
  TestCaseResult testCaseResult =
      JsonUtils.readOrConvertValue(entityTimeSeries, TestCaseResult.class);
  TestCase testCase =
      Entity.getEntityByName(TEST_CASE, testCaseResult.getTestCaseFQN(), "*", Include.ALL);

  // ... 알림 템플릿용 inherited fields와 failedRowsSample 로딩 생략

  ChangeEvent changeEvent = getChangeEvent(
      updateBy, eventType, testCase.getEntityReference().getType(),
      testCase.withUpdatedAt(testCaseResult.getTimestamp()));

  // [2] Alert의 Result 조건이 읽을 변경 필드 이름은 testCaseResult다.
  return changeEvent.withChangeDescription(
      new ChangeDescription().withFieldsUpdated(
          List.of(new FieldChange().withName(TEST_CASE_RESULT)
              .withNewValue(testCase.getTestCaseResult()))))
      .withEntity(testCase)
      .withEntityFullyQualifiedName(testCase.getFullyQualifiedName());
}
```

### 실제 코드 2: ChangeEvent 저장 → 필터 → Destination

소스:

- `openmetadata-service/src/main/java/org/openmetadata/service/events/ChangeEventHandler.java:75-79`
- `openmetadata-service/src/main/java/org/openmetadata/service/apps/bundles/changeEvent/AbstractEventConsumer.java:231-264, 403-409`
- `openmetadata-service/src/main/java/org/openmetadata/service/events/scheduled/EventSubscriptionScheduler.java:249-255`

```java
// [1] ChangeEvent를 server의 change_event 테이블에 넣는다.
if (!changeEvent.getEventType().equals(EventType.ENTITY_NO_CHANGE)) {
  Entity.getCollectionDAO().changeEventDAO().insert(JsonUtils.pojoToJson(changeEvent));
}

// [2] AbstractEventConsumer.execute(): 자신의 offset 이후 이벤트를 조회한다.
ResultList<ChangeEvent> batch = pollEvents(offset, eventSubscription.getBatchSize());
eventsWithReceivers.putAll(createEventsWithReceivers(batch.getData()));
// ... batch 크기와 alert metrics 기록 생략
if (!eventsWithReceivers.isEmpty()) {
  publishEvents(eventsWithReceivers);
}

// [3] publishEvents(): 미리 저장된 Source·Result 조건에 맞는 이벤트만 남긴다.
Map<ChangeEvent, Set<UUID>> filteredEvents =
    getFilteredEvents(eventSubscription, events);

// ... 통과한 이벤트의 수신자 계산 생략

// [4] 설정된 Destination publisher가 실제 전송한다.
publisher.sendMessage(event, recipients);
```

결과 저장과 알림 발송은 한 HTTP 호출 안에서 끝나지 않는다. EventSubscription job이 설정된 `pollInterval`마다 ChangeEvent를 읽고 규칙을 평가한 뒤 비동기로 전송한다.

### Failed TestCase Alert 설정값

| 구분 | 실제 값 | 결정 위치 |
|---|---|---|
| Source | `Test Case` | Alert UI |
| Action | `Get Test Case Status Updates` | Alert UI |
| Result | `Failed` | Alert UI. `Success`, `Aborted`, `Queued`도 선택 가능 |
| Destination | Users, Teams, Owners, Followers, Admins 또는 외부 채널 | Alert UI |
| ChangeEvent | `entityType=testCase`, `eventType=entityUpdated` | Server 내부 |
| 변경 필드 | `testCaseResult` | Server 내부 |

Test 결과 상태 `Success / Failed / Aborted / Queued`와 Incident 처리 상태 `New / Ack / Assigned / Resolved`는 서로 다르다. 정상 완료 알림은 `Success`, 판정 실패 알림은 `Failed`, 실행 중단 알림은 `Aborted`를 선택한다.

### 누구에게 알리는가

| Destination | 수신 대상 |
|---|---|
| `Users`, `Teams` | Alert에 지정한 사용자 또는 팀 |
| `Owners`, `Followers` | 이벤트 대상 TestCase 또는 자산의 소유자·팔로워 |
| `Admins` | OpenMetadata admin 사용자 |
| External | 지정한 Email 주소, Slack·MS Teams·GChat 채널 또는 Webhook endpoint |

> **주의**: Alert 설정이 없거나 조건이 맞지 않으면 테스트가 실행돼도 알림은 전송되지 않는다. Incident의 `Ack`·`Assigned` 때 Task 담당자에게 가는 인앱 알림은 EventSubscription Alert와 별개다.
