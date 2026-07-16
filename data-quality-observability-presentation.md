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

이 범위의 실행 주체는 세 가지다.

- **OpenMetadata server**: TestSuite·TestCase 메타데이터, Profile·Result·Incident, ChangeEvent와 Alert 설정을 저장한다.
- **Python 실행 프로세스**: Profiler 또는 TestSuite workflow를 실행한다. UI·스케줄은 배포된 pipeline runner가, CLI·SDK는 호출한 Python 프로세스가 시작한다.
- **검사 대상 DB(Source database)**: `dim_customer`의 실제 행이 들어 있는 MySQL·PostgreSQL·Snowflake 같은 업무 DB다. Python 실행 프로세스가 Database Service의 연결 설정으로 이 DB에 `COUNT(*)`, `IS NULL`, `COUNT(DISTINCT ...)` 등을 포함한 집계 SQL을 보내고 결과 숫자를 받는다. TestSuite·TestCase·Profile은 이 DB가 아니라 OpenMetadata server에 저장된다.

---

## 1. 품질 프레임워크

> **담당 화면**: Test Library · Data Quality → Test Cases · Data Quality → Test Suites
>
> **경로**: `/test-library` · `/data-quality/test-cases` · `/data-quality/test-suites`

`TestDefinition + 검사 대상 + parameterValues`로 `TestCase`를 만들고, `TestSuite`는 만들어진 TestCase를 포함한다. `TestDefinition → TestSuite → TestCase`는 생성 순서가 아니다. 실제로 TestCase가 TestDefinition을 참조하고, TestSuite가 TestCase를 포함한다.

![TestDefinition, 검사 대상, 파라미터로 TestCase를 만들고 Table Suite와 Bundle Suite가 이를 포함하는 관계](quality-model.png)

- `TestDefinition`: 재사용할 판정 로직과 파라미터 형식. 예: `columnValuesToBeNotNull`
- `TestCase`: Definition, 대상, 기준값을 결합한 실행 가능한 검사 한 건. 예: `customer_id_not_null`
- `TestSuite`: 관련 TestCase를 포함하는 OpenMetadata 엔터티

### TestDefinition과 Test Library

기본 제공 TestDefinition은 25개이며 Table 9개, Column 16개로 나뉜다.

| 적용 수준 | 대표 Definition |
|---|---|
| `TABLE` | `tableRowCountToBeBetween`, `tableCustomSQLQuery`, `tableDiff` |
| `COLUMN` | `columnValuesToBeNotNull`, `columnValuesToBeUnique`, `columnValuesToMatchRegex` |

**Test Library는 TestDefinition 목록 화면**이다. Definition은 검사 템플릿이며, 검사 대상과 파라미터를 지정해 TestCase를 만들어야 실제 검사 단위가 된다.

### TestCase 생성·등록

UI에서는 **Data Quality → Test Cases → Add Test Case** 또는 Table의 **Data Observability → Data Quality → Add Test**에서 TestCase를 만든다.

| 입력값 | 예시 |
|---|---|
| Test Definition | `columnValuesToBeNotNull` |
| 검사 대상 | `dim_customer.customer_id` |
| TestCase 이름 | `customer_id_not_null` |
| parameterValues | 이 Definition은 별도 기준값 없음 |

Submit하면 브라우저가 TestCase 생성 요청을 Server에 보낸다. 이 요청은 메타데이터를 등록하며 검사 대상 DB에 SQL을 실행하지 않는다.

#### UI Add Test Case → REST POST

소스:

- `openmetadata-ui/src/main/resources/ui/src/components/DataQuality/AddDataQualityTest/components/TestCaseFormV1.tsx:824-830`
- `openmetadata-ui/src/main/resources/ui/src/rest/index.ts:17-21`
- `openmetadata-ui/src/main/resources/ui/src/rest/testAPI.ts:138, 184-190`

```typescript
// [1] Form 값을 name, testDefinition, entityLink, parameterValues로 변환한다.
const testCaseObj = createTestCaseObj(values);

// [2] createTestCase()가 Server의 생성 API를 호출한다.
const createdTestCase = await createTestCase(testCaseObj);

const testCaseUrl = '/dataQuality/testCases';
const response = await APIClient.post<CreateTestCase, AxiosResponse<TestCase>>(
  testCaseUrl,
  data
);
```

`APIClient`의 base URL이 `/api/v1`이므로 실제 endpoint는 `POST /api/v1/dataQuality/testCases`다.

#### Python CRUD SDK → REST PUT

UI를 사용하지 않고 TestCase 메타데이터를 먼저 등록할 수도 있다. 다음은 SDK import와 `configure(...)`를 마친 뒤의 핵심 호출이다.

소스:

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
))  # create_or_update()가 PUT으로 등록하며 Test SQL은 실행하지 않는다.
```

UI POST와 CRUD SDK PUT 모두 Server에서 Definition·파라미터·대상을 검증하고 TestCase를 저장한다.

### Table Suite와 Bundle Suite

| UI 명칭 | 생성 방식 | TestCase 포함 방식 |
|---|---|---|
| `Table Suite` (`basic=true`) | Table의 첫 TestCase를 만들 때 Server가 자동 생성 | 해당 Table의 Table·Column TestCase가 자동으로 속함 |
| `Bundle Suite` (`basic=false`) | Data Quality → Test Suites → **Add Bundle Suite**에서 사용자가 생성 | 하나 또는 여러 Table의 기존 TestCase를 선택해 추가 |

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

// [4] 선택한 TestDefinition ID와 새 TestCase ID의 연결도 저장한다.
addRelationship(
    test.getTestDefinition().getId(), test.getId(),
    TEST_DEFINITION, TEST_CASE, Relationship.CONTAINS);
```

여기서 `TestDefinition → TestCase 관계 저장`은 UI에 새 트리를 만든다는 뜻이 아니다. OpenMetadata 내부 관계 테이블에 두 엔터티의 ID를 연결해 두는 것이다. TestCase API에 `fields=testDefinition`을 요청하면 이 관계를 다시 읽어 `testDefinition` 필드로 돌려주며, UI는 이를 **Test Type** 또는 `columnValuesToBeNotNull` 같은 Definition 이름으로 표시한다.

> **주의**: 위 코드는 TestCase를 만들 때 Table Suite에 자동 연결하는 경로다. 검증이 실패하면 TestCase와 Table Suite 모두 생성되지 않는다.

### 실제 코드: Bundle Suite 생성 → 기존 TestCase 추가

소스:

- `openmetadata-ui/src/main/resources/ui/src/components/DataQuality/BundleSuiteForm/BundleSuiteForm.tsx:280-295`
- `openmetadata-ui/src/main/resources/ui/src/rest/testAPI.ts:139, 389-395`
- `openmetadata-ui/src/main/resources/ui/src/rest/testAPI.ts:228-246`
- `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/TestCaseRepository.java:1091-1111`

```typescript
// [1] UI가 Bundle Suite를 먼저 생성한다.
const testSuite = await createTestSuites(testSuitePayload);

// [2] 이어서 화면에서 선택한 기존 TestCase들을 생성된 Suite에 추가한다.
await addTestCasesToLogicalTestSuiteBulk(testSuite.id ?? '', {
  selectAll: testCaseSelectionPayload.selectAll,
  includeIds: testCaseSelectionPayload.includeIds,
  excludeIds: testCaseSelectionPayload.excludeIds,
});

// createTestSuites() 내부: POST /api/v1/dataQuality/testSuites
const response = await APIClient.post<CreateTestSuite, AxiosResponse<TestSuite>>(
  testSuiteUrl,
  data
);
```

```java
// [3] Server는 기존 TestCase와 Bundle Suite의 CONTAINS 관계를 추가한다.
bulkAddToRelationship(
    testSuite.getId(), testCaseIds,
    TEST_SUITE, TEST_CASE, Relationship.CONTAINS);
```

`addTestCasesToLogicalTestSuiteBulk()`는 `PUT /api/v1/dataQuality/testCases/logicalTestCases/bulk`를 호출한다. UI의 Bundle Suite를 백엔드 API에서는 `logical test suite`라고 부르므로 경로에 `logicalTestCases`가 사용된다. Bundle Suite에 추가해도 TestCase의 기본 Table Suite는 그대로 유지되며, 같은 TestCase가 여러 Bundle Suite에 들어갈 수 있다. Bundle에서 제거할 때도 TestCase 자체를 지우지 않고 이 `CONTAINS` 관계만 삭제한다.

---

## 2. Profiler & Data Profile

> **결과 화면**: Table → Data Observability → Table Profile / Column Profile
>
> **경로**: `/table/<Table FQN>/profiler/table-profile` · `/table/<Table FQN>/profiler/column-profile`
>
> **실행 관리**: Settings → Services → Databases → Service → Agents

Database Service 기준으로 `Profiler`는 검사 대상 DB에 SQL을 보내 Table과 Column의 숫자 요약값을 계산하고 그 결과를 OpenMetadata server에 저장하는 Python 작업이다. 여기서 `metric`은 `rowCount`, `nullCount`, `distinctCount`처럼 데이터 상태를 나타내는 숫자 한 항목을 뜻한다.

`workflow`라는 말은 이 작업을 `Source → Processor → Sink` 세 단계로 나누어 순서대로 실행한다는 뜻이다. `Table Profile`·`Column Profile` 화면은 저장된 결과를 조회할 뿐이며, 화면을 여는 동작 자체가 Profiler를 실행하지는 않는다.

`OpenMetadataSource`는 Python workflow의 컴포넌트이고, **검사 대상 DB(Source database)**는 실제 행이 저장된 외부 DB다. 이름에 `Source`가 함께 들어가지만 서로 다른 대상이다.

![Profiler workflow에서 Source, Processor, Sink가 담당하는 단계](profiler-sampling.png)

| 단계 | 실제로 하는 일 | 출력 또는 저장 결과 |
|---|---|---|
| `OpenMetadataSource` | OpenMetadata API에서 실행 대상 Table·Column·Profiler 설정을 조회하고, workflow에 준비된 Database Service 연결 설정으로 `ProfilerSource`를 구성 | `ProfilerSourceAndEntity`: 대상 Table과 검사 대상 DB에 접속할 객체 |
| `ProfilerProcessor` | Profiler runner를 만들고 `process()`를 호출한다. 내부 `ProfilerInterface`가 검사 대상 DB에 집계 SQL을 실행해 Profile 요청 객체를 만듦 | `ProfilerResponse`: Table 정보와 `CreateTableProfileRequest` |
| `MetadataRestSink` | Processor가 만든 Profile을 OpenMetadata REST API에 PUT | Profile 저장 완료. 갱신된 `Table`을 반환하며 workflow 종료 |

`Sink`는 workflow의 **마지막 저장 단계**를 뜻한다. `MetadataRestSink`는 metric을 계산하지 않고, 앞 단계가 만든 결과를 받아 `PUT /api/v1/tables/{tableId}/tableProfile`로 저장한다. 이름에 `Metadata`가 붙은 이유는 저장 대상이 검사 대상 DB가 아니라 OpenMetadata server이기 때문이다.

### metric을 계산할 때 DB에서 일어나는 일

아래 SQL은 동작을 단순화한 개념형이다. `rowCount`는 전체 Table을 대상으로 한다. Column metric의 `<profile 대상 범위>`는 샘플링 미설정 시 전체 Table, 설정 시 Profiler 설정으로 선택한 샘플 범위다. 실제 쿼리는 DB 종류에 따라 합쳐지거나 달라질 수 있다.

| metric | 개념 SQL | 반환값 예시 |
|---|---|---|
| `rowCount` | `SELECT COUNT(*) FROM dim_customer` | `12430` |
| `nullCount` | `SELECT COUNT(*) FROM <profile 대상 범위> WHERE customer_id IS NULL` | `3` |
| `distinctCount` | `SELECT COUNT(DISTINCT customer_id) FROM <profile 대상 범위>` | `12427` |

### Custom Metric: 기본 목록에 없는 metric 추가

`rowCount`, `nullCount`, `distinctCount`는 동작을 설명하기 위해 고른 대표 예시다. 기본 Profiler에는 `mean`, `min`, `max`, `nullProportion`, `uniqueCount` 등 다른 metric도 있다. 관계형 DB에서 기본 목록에 없는 업무 지표는 Table 또는 Column Profile의 **Add → Custom Metric**에서 등록한다.

> **화면**: Table → Data Observability → Table Profile 또는 Column Profile → Add → Custom Metric

필수 입력은 `name`과 `expression`이고, Column Profile에 연결할 때 `columnName`을 추가한다.

- `name`: Profile 결과에서 사용할 metric 이름
- `columnName`: Column Profile에 연결할 때만 지정하는 Column 이름
- `expression`: 관계형 검사 대상 DB에서 실행할 **SQL 전체문**. 숫자 한 개를 반환해야 함

예를 들어 활성 고객 수를 Table metric으로 추가하는 요청은 다음과 같다.

소스:

- `openmetadata-spec/src/main/resources/json/schema/api/tests/createCustomMetric.json:12-22, 38`
- `openmetadata-ui/src/main/resources/ui/src/rest/customMetricAPI.ts:37-44`

```http
PUT /api/v1/tables/{tableId}/customMetric
Content-Type: application/json

{
  "name": "active_customer_count",
  "expression": "SELECT COUNT(*) FROM shopify.dim_customer WHERE status = 'ACTIVE'"
}
```

이 요청은 정의만 OpenMetadata server에 저장하며 SQL을 즉시 실행하지 않는다. 다음 Profiler 실행에서 Source가 Table의 `customMetrics`를 함께 읽고, Processor가 이를 `MetricTypes.Custom` 작업으로 추가한 뒤 아래 코드를 통해 SQL을 검사 대상 DB에서 실행한다.

여기서 구분할 점은 `Table.customMetrics`와 `TableProfile.customMetrics`가 서로 다른다는 것이다.

- `Table.customMetrics`: Custom Metric **정의**. 이름·SQL expression을 저장하며, native Profiler가 다음 실행에서 사용할 대상이다.
- `TableProfile.customMetrics`: 실행된 Profile **결과**. timestamp별 `name`·`value`를 저장한다.

소스:

- `ingestion/src/metadata/profiler/source/metadata.py:43, 92-98`
- `ingestion/src/metadata/profiler/processor/core.py:224-249, 329-353, 436-447`
- `ingestion/src/metadata/profiler/interface/sqlalchemy/profiler_interface.py:374-409`

```python
for metric in metrics:
    # [1] 변경·관리 명령 등 금지된 SQL token이 있는지 먼저 검사한다.
    if not is_safe_sql_query(metric.expression):
        raise RuntimeError(f"SQL expression is not safe\n\n{metric.expression}")

    # [2] 등록한 SQL 전체문을 Database Service 계정으로 대상 DB에서 실행한다.
    crs = session.execute(text(metric.expression))
    row = crs.scalar()  # [3] 쿼리가 반환한 첫 scalar 값을 읽는다.

    # [4] 일반 Profile과 함께 저장할 Custom Metric 결과를 만든다.
    custom_metrics.append(
        CustomMetricProfile(name=metric.name.root, value=row)
    )
```

Profiler가 만든 값은 `TableProfile.customMetrics` 또는 해당 `ColumnProfile.customMetrics`에 포함되고, 기존 Profile과 같은 REST Sink를 통해 저장된다.

**제약**

- `columnName`은 결과를 연결할 Column만 정한다. SQL에 Table이나 Column을 자동으로 넣지는 않는다.
- 관계형 DB의 Custom Metric SQL에는 `profileSampleConfig`가 자동 적용되지 않는다. 샘플 범위가 필요하면 `expression`에 직접 작성한다. workflow YAML의 `metrics`도 기본 metric 선택용이지 새 metric 정의용이 아니다.
- SQL로 표현할 수 없는 임의 Python metric을 UI·YAML로 추가하는 공개 플러그인 경로는 없다. 이 경우 ingestion 코드를 확장하거나 외부에서 값을 사전 계산해야 한다.

### SQL 없이 별도 프로세스로 Profile metric을 계산하는 방법

기본 Profiler workflow에는 임의의 외부 프로세스를 callback으로 등록하는 설정이 없다. 대신 다음 두 경로 중 하나를 선택한다.

#### 1) 독립 프로세스가 Profile API에 기록

Airflow·Cron·Kubernetes Job 같은 별도 작업이 업무 지표를 계산하고, OpenMetadata server의 Profile API에 결과를 PUT한다. 이 방식은 ingestion 소스를 수정하지 않아도 된다.

```http
PUT /api/v1/tables/{tableId}/tableProfile
Content-Type: application/json

{
  "tableProfile": {
    "timestamp": 1763078400000,
    "rowCount": 12430,
    "customMetrics": [
      {"name": "fraud_score_avg", "value": 0.91}
    ]
  }
}
```

외부 프로세스가 호출하는 것은 Custom Metric 정의 API가 아니라 Profile 결과 API다. 따라서 SQL 없이도 `TableProfile.customMetrics`에 값을 기록할 수 있다. 다만 이 호출만으로 `Table.customMetrics` 정의가 생성되거나 native Profiler가 해당 metric을 실행하게 되지는 않는다.

`tableProfile`은 새 시계열 Profile로 저장된다. `customMetrics`만 보내면 그 시점의 최신 Profile에 `rowCount` 같은 기본값이 빠질 수 있으므로, 기존 최신 Profile을 읽어 필요한 값을 합친 뒤 보내거나 기본 metric까지 함께 계산하는 편이 안전하다. 이 API에는 `EDIT_DATA_PROFILE` 권한이 필요하다.

> **UI에서 보이는 범위**: Profile 결과만 저장해도 Custom Metric 그래프가 자동으로 생기는 것은 아니다. UI는 `Table.customMetrics` 정의와 Profile 안의 같은 `name` 값을 함께 사용한다. SQL 없는 외부 metric을 native Profiler와 UI 그래프에서 정식으로 사용하려면 ingestion Profiler 또는 별도 UI/API 확장이 필요하다.

소스:

- `openmetadata-service/src/main/java/org/openmetadata/service/resources/databases/TableResource.java:1398-1424`
- `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/TableRepository.java:1032-1121`
- `openmetadata-spec/src/main/resources/json/schema/api/data/createTableProfile.json:8-29`
- `openmetadata-ui/src/main/resources/ui/src/components/Database/Profiler/TableProfiler/TableProfilerChart/TableProfilerChart.tsx:91-130`
- `openmetadata-ui/src/main/resources/ui/src/utils/TableProfilerUtils.ts:112-145`

#### 2) ingestion Profiler 구현을 확장

같은 Profiler workflow 안에서 실행하려면 connector용 `ProfilerInterface`를 확장하고 ServiceSpec에 `profiler_class`를 지정한다. 계산 함수가 외부 서비스나 사내 라이브러리를 호출한 뒤 `result["table"]["customMetrics"]`에 Profile 결과를 넣도록 만들 수 있다.

아래는 실제 구현의 핵심 구조만 남긴 발췌다. 생성자와 연결 설정은 생략했지만, **`get_all_metrics()`가 반환하는 `result["table"]`에 `customMetrics`를 넣는 지점**이 중요하다.

```python
class MyProfilerInterface(SQAProfilerInterface):
    def get_all_metrics(self, metric_funcs):
        result = super().get_all_metrics(metric_funcs)
        result["table"].setdefault("customMetrics", []).append(
            CustomMetricProfile(
                name="fraud_score_avg",
                value=run_company_calculator(self.table_entity),
            )
        )
        return result


ServiceSpec = DefaultDatabaseSpec(
    metadata_source_class=MySource,
    profiler_class=MyProfilerInterface,
    sampler_class=MySampler,
    connection_class=MyConnection,
)
```

Profiler Source가 ServiceSpec의 클래스를 선택하고, `ProfilerProcessor`가 `runner.process()`를 호출하므로 Sink와 Profile 저장 경로는 그대로 재사용된다. SQLAlchemy가 아닌 Source라면 `SQAProfilerInterface` 대신 `ProfilerInterface`를 직접 구현하고 해당 Source용 sampler·계산 로직도 제공해야 한다. YAML의 `metrics: [fraud_score_avg]`처럼 기본 metric 목록에서 선택하려면 `Metric` 클래스, `Metrics` registry, `profilerConfiguration.json`의 metric enum과 생성 모델까지 ingestion 소스에 맞춰 수정해야 한다. 현재 구현에는 이 Python metric을 UI에서 동적으로 등록하는 공개 plugin API가 없다.

소스:

- `ingestion/src/metadata/utils/service_spec/service_spec.py:38-67, 138-146`
- `ingestion/src/metadata/profiler/source/database/base/profiler_source.py:180-205, 247-285`
- `ingestion/src/metadata/profiler/processor/processor.py:75-105`
- `ingestion/src/metadata/profiler/processor/core.py:478-563`
- `ingestion/src/metadata/profiler/metrics/registry.py:62-116`
- `openmetadata-spec/src/main/resources/json/schema/configuration/profilerConfiguration.json:9-50`
- `openmetadata-spec/src/main/resources/json/schema/metadataIngestion/databaseServiceProfilerPipeline.json:109-113`

### 실제 코드: Source → Processor → Sink

소스:

- `ingestion/src/metadata/workflow/profiler.py:67-75`
- `ingestion/src/metadata/profiler/source/fetcher/fetcher_strategy.py:320-339`
- `ingestion/src/metadata/profiler/processor/processor.py:75-103`
- `ingestion/src/metadata/workflow/ingestion.py:169-176`
- `ingestion/src/metadata/ingestion/sink/metadata_rest.py:990-1002`
- `ingestion/src/metadata/ingestion/ometa/mixins/table_mixin.py:376-389`

```python
# [1] workflow는 Processor 다음에 Sink가 실행되도록 순서를 정한다.
self.steps = (profiler_processor, sink)

# [2] Source 내부 EntityFetcher가 Table마다 작업 단위를 만든다.
yield Either(
    right=ProfilerSourceAndEntity(
        profiler_source=profiler_source,  # 검사 대상 DB용 Profiler를 만들 객체
        entity=table,                     # 이번에 처리할 Table
    ),
)

# [3] Processor가 그 record를 받아 검사 대상 DB에서 metric을 계산한다.
profiler_runner = record.profiler_source.get_profiler_runner(
    record.entity, self.profiler_config)
profile = profiler_runner.process()
return Either(right=profile)  # ProfilerResponse를 반환

# [4] 공통 loop가 Source 결과를 받아 Processor → Sink 순서로 전달한다.
processed_record = record
for step in self.steps:
    # 앞 단계가 결과를 만들지 못했다면 다음 단계는 호출하지 않는다.
    if processed_record is not None and isinstance(step, (Processor, Stage, Sink)):
        processed_record = step.run(processed_record)

# [5] Sink가 ProfilerResponse의 Table과 Profile을 OpenMetadata API에 저장한다.
table = self.metadata.ingest_profile_data(
    table=record.table,
    profile_request=record.profile,
)

# [6] ingest_profile_data()가 실제 Profile endpoint를 호출한다.
resp = self.client.put(
    f"{self.get_suffix(Table)}/{table.id.root}/tableProfile",
    data=profile_request.model_dump_json(),
)
return Table(**resp)
```

> **코드 읽는 법**: `Either(right=...)`의 `right`는 정상 결과를 담아 공통 workflow에 넘기는 값이다. 실제 `dim_customer` 행은 검사 대상 DB에 그대로 있고, Processor 내부 Profiler runner가 그 DB에 집계 SQL을 보낸다. 정상 경로는 `ProfilerSourceAndEntity → ProfilerResponse → REST 저장`이다. 해당 Table을 처리하다 오류가 발생해 앞 단계가 정상 결과를 만들지 못하면 `processed_record`가 `None`이므로 다음 `MetadataRestSink`를 호출하지 않는다.

### 저장되는 Profile

주요 저장값은 다음과 같다.

- `TableProfile.rowCount`: Table 전체 행 수
- `ColumnProfile.nullCount`, `nullProportion`: NULL 개수와 비율
- `ColumnProfile.distinctCount`: 서로 다른 값의 수

### 프로파일링 범위를 정하는 `profileSampleConfig`

`profileSampleConfig`는 Profiler가 Column metric을 계산할 때 읽을 행 범위를 정하는 설정이다. 행 데이터를 저장하는 필드가 아니라 Processor가 실행할 쿼리 범위에 적용된다.

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
- `DYNAMIC`: Table 행 수에 따라 샘플 크기를 자동으로 선택하며, 필요하면 `rowCountThreshold`별 크기를 지정한다.
- 적용 순서: Table 설정을 먼저 사용하고, 없으면 Schema → Database → workflow 기본 설정 순서로 확인한다.

> **주의**: 샘플링해도 `TableProfile.rowCount`는 전체 Table에서 계산한다. `ColumnProfile.nullCount`, `distinctCount`는 선택된 샘플 범위를 기준으로 해석한다.

---

## 3. 테스트 실행 & 결과

> **담당 화면**: Table → Data Observability → Data Quality · Data Quality → Test Cases
>
> **경로**: `/table/<Table FQN>/profiler/data-quality` · `/data-quality/test-cases`

`TestSuiteWorkflow`는 실행할 TestCase를 조회하고, 검사 대상 DB에서 SQL을 실행한 뒤, `TestCaseResult`를 OpenMetadata server에 저장하는 Python workflow다.

![저장된 pipeline 또는 CLI와 SDK가 TestSuiteWorkflow를 시작하고 TestCaseResult를 저장하는 흐름](test-execution-pipeline.png)

TestSuiteWorkflow는 두 경로로 시작할 수 있다.

| 시작 방법 | 누가 시작하는가 | 설정은 어디에 있는가 |
|---|---|---|
| `Pipeline runner` | UI의 **Run Now** 또는 저장된 cron schedule | OpenMetadata server에 등록된 TestSuite pipeline |
| `CLI YAML` / `TestRunner.run()` | 터미널 명령을 실행한 사용자 또는 Python 애플리케이션 | YAML 파일 또는 Python 코드 |

두 시작 경로는 아래의 같은 Source → Processor → Sink 단계를 사용한다.

| 컴포넌트 | 실제 역할 |
|---|---|
| `TestSuiteSource` | Server의 Suite에서 실행할 TestCase와 대상 Table을 조회하고 Database Service 연결 설정을 준비 |
| `TestCaseRunner` | 검사 대상 DB에 Test SQL을 실행하고 Validator로 상태를 판정 |
| `MetadataRestSink` | 판정이 끝난 `TestCaseResult`를 OpenMetadata REST API에 POST하여 저장 |

### Run Now와 예약 실행

일정은 TestCase나 TestSuite 엔터티가 아니라 **TestSuite 타입 Ingestion Pipeline**에 저장된다. UI의 **Run Now**는 이 pipeline을 즉시 시작하고, cron schedule은 `airflowConfig.scheduleInterval` 시각에 시작한다.

`TestSuiteSource`는 기본적으로 Suite의 TestCase 전체를 조회한다. `source.sourceConfig.config.testCases`는 그중 이번 pipeline에서 실행할 TestCase 이름만 고르는 선택 필터다.

| Pipeline의 TestCase 선택 | 실행 범위 |
|---|---|
| `testCases` 미지정 | 실행 시점에 Suite에 속한 모든 TestCase |
| `testCases: [customer_id_not_null]` | 이름 목록에 적힌 TestCase만 실행 |

```yaml
source:
  sourceConfig:
    config:
      type: TestSuite
      entityFullyQualifiedName: sample_data.ecommerce_db.shopify.dim_customer
      testCases: [customer_id_not_null] # 선택 필터. 생략하면 Suite 전체
```

UI에서 TestCase 하나에 설정한 일정도 내부적으로는 이 이름 필터를 가진 TestSuite pipeline이다.

### CLI YAML로 workflow 시작

CLI YAML은 Server에 저장된 TestCase를 실행할 수 있고, `processor.config.testCases`에 실행할 TestCase 정의를 직접 작성할 수도 있다. Processor는 Source가 조회한 목록에 같은 이름이 없으면 이 정의를 Server에 create-or-update로 반영하고 같은 workflow의 실행 목록에 포함한다.

아래는 설정에 직접 작성한 TestCase 한 건을 실행하는 데 필요한 핵심 필드다.

```yaml
source:
  sourceConfig:
    config:
      type: TestSuite
      entityFullyQualifiedName: sample_data.ecommerce_db.shopify.dim_customer

processor:
  type: orm-test-runner
  config:
    forceUpdate: false
    testCases:
      # [1] workflow 설정에 직접 작성한 TestCase 정의
      - name: customer_id_not_null
        testDefinitionName: columnValuesToBeNotNull
        columnName: customer_id
        computePassedFailedRowCount: true
```

생략한 `source.type`, `source.serviceName`, `sink.config`, 서버 주소와 인증까지 포함한 완전한 파일은 `metadata test -c test-suite.yaml`로 실행한다.

### 실제 코드: 설정의 TestCase 정의 → Server 반영·실행 목록 추가

소스: `ingestion/src/metadata/data_quality/processor/test_case_runner.py:155-202`의 핵심 발췌

```python
# [1] Source가 조회한 실행 목록에 같은 이름이 없는 설정 정의를 고른다.
test_cases_to_create = [
    cli_test_case_definition for cli_test_case_definition in cli_test_cases_definitions
    if cli_test_case_definition.name not in test_case_names
]

for test_case_to_create in test_cases_to_create:
    # [2] Definition·대상·파라미터를 PUT create-or-update로 Server에 반영한다.
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

Source가 조회한 실행 목록에 같은 이름의 Case가 있으면 `forceUpdate: false`에서 기존 설정을 유지한다. `true`일 때도 갱신 범위는 `entityLink`, 비어 있지 않은 `parameterValues`, `computePassedFailedRowCount`이며 이름·Definition·설명은 바꾸지 않는다.

### DQ SDK로 같은 workflow 시작

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
results = runner.run()  # [1] TestCase 정의 반영 → SQL 실행 → 결과 반환
```

`TestRunner.add_test()`가 Python 코드의 TestCase 정의를 workflow 설정에 넣고, `run()`이 TestSuiteWorkflow를 시작한다. 기본값에서는 같은 이름의 기존 Case도 제한된 필드를 갱신하며, 이를 끄려면 `add_test()` 전에 `runner.setup(force_test_update=False)`를 호출한다.

시작 경로와 관계없이 `TestCaseRunner`는 각 TestDefinition에 연결된 통과·실패 판정 코드인 Validator를 실행한다.

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

> **주의**: TestCase 정의는 OpenMetadata server에 있지만, 실제 SQL은 Python Runner가 `dim_customer` 행이 있는 검사 대상 DB에 보낸다. `MetadataRestSink`는 SQL을 실행하지 않고 판정이 끝난 Result만 Server에 저장한다.

### 판정 코드

`columnValuesToBeNotNull` TestDefinition은 등록·실행 경로와 관계없이 같은 Validator로 판정된다.

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

`TestCaseResult`는 TestCase 한 번의 실행 상태와 판정에 사용한 값을 저장한다. 다음은 이번 실행에서 처리한 1,243행 중 NULL이 3개인 예시다.

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

### 실패 행을 확인하는 `failedRowsSample`

지원하는 Validator가 `Failed`를 반환하면 원인 확인용 실패 행을 최대 50개까지 OpenMetadata server에 저장할 수 있다. `failedRowsSample`은 실패 원인을 확인하기 위한 데이터이며, `Success / Failed` 판정 기준으로 사용되지는 않는다.

### SQL이 없는 Source의 Profile과 TestCase

지금까지 2절의 Profile과 3절의 TestCase는 관계형 Database Service의 SQL 경로를 기준으로 설명했다. SQL을 사용할 수 없는 Source에서 OpenMetadata가 SQL을 흉내 내는 것은 아니다. **자산이 Table·Column인지**, 그리고 **connector가 어떤 실행 엔진을 제공하는지**에 따라 처리 범위가 달라진다.

- **Datalake Database Service**: S3·GCS·Azure·local 파일을 OpenMetadata의 Table로 수집한 뒤 DataFrame으로 읽는다. Profile과 TestCase 모두 Pandas 구현을 사용한다.
- **MongoDB·DynamoDB**: 공통적으로 adaptor의 `item_count()`로 `rowCount`를 계산하고, MongoDB adaptor는 일부 Column aggregate도 제공한다. `scan()`은 Profile metric이 아니라 Sample Data 행 조회에 사용한다. SQL Profiler의 모든 metric을 지원하는 것은 아니며, NoSQL 전용 TestCase 실행 엔진은 없다.
- **Kafka Topic, Dashboard·Chart, Storage Container**: Table·Column이 아니므로 이 문서의 Profiler·TestCase workflow 대상이 아니다. 같은 파일도 Storage Service의 Container로 연결한 경우와 Datalake Database Service의 Table로 연결한 경우의 지원 범위가 다르다.

TestDefinition의 대상 형식이 `TABLE`과 `COLUMN`뿐이고, Server도 TestCase의 `entityLink`에서 연결된 Table을 읽어 검증하기 때문이다.

소스:

- `openmetadata-spec/src/main/resources/json/schema/tests/testDefinition.json:44-51`
- `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/TestCaseRepository.java:664-692`

#### Datalake: DataFrame에서 Profile·Test 실행

소스:

- `ingestion/src/metadata/ingestion/source/database/datalake/service_spec.py:12-17`
- `ingestion/src/metadata/data_quality/interface/pandas/pandas_test_suite_interface.py:68-90`
- `ingestion/src/metadata/data_quality/builders/validator_builder.py:50-84`

```python
# [1] Datalake connector가 SQLAlchemy 대신 Pandas 구현을 지정한다.
ServiceSpec = DefaultDatabaseSpec(
    metadata_source_class=DatalakeSource,
    profiler_class=PandasProfilerInterface,
    test_suite_class=PandasTestSuiteInterface,
    sampler_class=DatalakeSampler,
    connection_class=DatalakeConnection,
)

# [2] Test 실행 시 파일을 읽은 DataFrame으로 Runner를 만든다.
self._runner = PandasRunner(
    dataset=self.sampler.get_dataset(),
    raw_dataset=self.sampler.raw_dataset,
)

# [3] 같은 TestDefinition 이름에서 pandas용 Validator를 선택한다.
return self.validator_builder_class(
    runner=self._runner,
    test_case=test_case,
    test_definition=test_definition,
    entity_type=entity_type,
    source_type=SourceType.PANDAS,
)
```

예를 들어 `columnValuesToBeNotNull`은 SQL의 `IS NULL` 대신 DataFrame의 NULL 개수를 계산한다. 판정 결과부터 `MetadataRestSink → TestCaseResult POST`까지는 관계형 DB 경로와 같다.

Datalake의 Custom Metric도 SQL을 실행하지 않는다. `expression`에 `status == 'ACTIVE'` 같은 `DataFrame.query()` boolean 조건을 넣고, 조건에 맞는 행 수를 Profile 값으로 저장한다. 이 경로는 sampler가 만든 DataFrame을 사용하므로 샘플링 설정의 영향을 받는다.

소스: `ingestion/src/metadata/profiler/interface/pandas/profiler_interface.py:277-309`의 핵심 발췌

```python
row = sum(
    len(df.query(metric.expression).index)  # 조건에 맞는 행 수
    for df in runner()                     # sampler가 만든 DataFrame
    if len(df.query(metric.expression).index)
)
```

#### MongoDB·DynamoDB: native API로 Profile 계산

소스:

- `ingestion/src/metadata/profiler/adaptors/factory.py:71-75`
- `ingestion/src/metadata/profiler/interface/nosql/profiler_interface.py:47-83, 93-123`
- `ingestion/src/metadata/profiler/adaptors/dynamodb.py:31-40`
- `ingestion/src/metadata/profiler/adaptors/mongodb.py:124-163`
- `ingestion/src/metadata/sampler/nosql/sampler.py:70-87`

```python
# [1] Table metric은 SQL 대신 connector adaptor의 NoSQL 함수를 실행한다.
fn = metric().nosql_fn(runner)
result[metric.name()] = fn(self.table)

# [2] MongoDB의 Column metric은 adaptor의 aggregate API를 사용한다.
aggs = [metric(column).nosql_fn(runner)(self.table) for metric in metrics]
row = runner.get_aggregates(self.table, column, aggs)
```

NoSQL Profiler 구현에서 query·window·system·custom metric은 계산하지 않으므로 SQL 경로와 지원 범위가 같지 않다. 또한 TestCase 실행 엔진을 고르는 `SourceType`에는 `SQL`과 `PANDAS`만 있으며 NoSQL 전용 Validator 경로는 없다.

Table·Column으로 등록돼 있지만 OpenMetadata의 native Test 실행 엔진이 없는 Source는 외부 품질 도구가 **기존 Table·Column TestCase를 검사한 뒤 그 TestCase FQN으로** `POST /api/v1/dataQuality/testCases/testCaseResults/{testCaseFQN}`에 결과를 저장할 수 있다. 이 경우 OpenMetadata는 결과 조회·Incident·Alert를 담당하지만 검사는 외부 도구가 담당한다. Topic·Dashboard·Container 자체에는 이 TestCase를 직접 연결할 수 없다.

#### 외부 품질 도구를 연결하는 위치

외부 도구는 OpenMetadata server 안에 넣는 것이 아니라, 도구의 실행 환경에 **adapter/action**을 둔다. 이 adapter가 도구의 결과를 OpenMetadata의 TestDefinition·TestCase·TestCaseResult 형식으로 변환한다.

구성 위치는 두 가지다.

- 회사의 CI·Airflow·Kubernetes Job에서 이미 도구를 실행한다면, 그 저장소에 adapter를 함께 둔다. OpenMetadata server 코드를 수정할 필요가 없다.
- OpenMetadata ingestion workflow로 패키징하려면 `ingestion/src/metadata/<tool>/` 아래에 action/connector를 만들고 ingestion 이미지에 포함한다. 이 경우에도 실제 품질 계산은 외부 도구가 하고, action은 결과 변환·전송을 담당한다.

1. 대상 Table·Column을 OpenMetadata에 먼저 등록한다.
2. 도구의 검사 하나를 TestDefinition과 TestCase로 연결한다. TestDefinition의 `testPlatforms`에는 `GreatExpectations`, `dbt`, `Soda`, `Other` 같은 외부 플랫폼을 넣을 수 있다.
3. CI·Airflow·Kubernetes Job에서 Great Expectations, dbt, Soda 등의 검사를 실행한다.
4. 결과의 통과 여부를 `Success / Failed / Aborted`로 바꾸고, metric 값은 `testResultValue`에 넣어 TestCaseResult API로 전송한다.

OpenMetadata에 이미 포함된 Great Expectations adapter도 같은 구조다.

소스: `ingestion/src/metadata/great_expectations/action.py:563-615`

```python
# 아래 코드는 외부 결과 객체에서 필요한 값을 가져오는 핵심 부분만 남긴 것이다.
# [1] 외부 expectation 종류를 TestDefinition으로 등록/조회
test_definition = ometa_conn.get_or_create_test_definition(
    test_definition_fqn=result["expectation_config"]["expectation_type"],
    test_definition_description="...",
    entity_type=EntityType.COLUMN if "column" in result["expectation_config"]["kwargs"]
    else EntityType.TABLE,
    test_platforms=[TestPlatform.GreatExpectations],
)

# [2] 검사 대상 Table·Column에 TestCase를 연결
# test_case_fqn은 이 결과와 Table FQN을 조합해 만든 대상 식별자다.
test_case = ometa_conn.get_or_create_test_case(
    test_case_fqn,
    entity_link=get_entity_link(
        Table,
        fqn=table_entity.fullyQualifiedName.root,
        column_name=fqn.split_test_case_fqn(test_case_fqn).column,
    ),
    test_definition_fqn=test_definition.fullyQualifiedName.root,
)

# [3] 외부 실행 결과만 OpenMetadata에 전송
# metric_values는 외부 도구의 observed_value 등에서 만든 testResultValue다.
ometa_conn.add_test_case_results(
    TestCaseResult(
        timestamp=Timestamp(now_ms),
        testCaseStatus=TestCaseStatus.Success if result["success"]
        else TestCaseStatus.Failed,
        testResultValue=metric_values,
    ),
    test_case_fqn=test_case.fullyQualifiedName.root,
)
```

OpenMetadata의 native TestSuite runner는 `testPlatforms`에 `OpenMetadata`가 없는 TestCase를 실행하지 않으므로, 외부 도구가 실행 주체가 된다. 결과 POST가 들어오면 그 뒤의 Result 저장·Incident 연결·Observability Alert 처리는 OpenMetadata server가 기존 경로대로 수행한다.

소스:

- `ingestion/src/metadata/data_quality/processor/test_case_runner.py:256-280`
- `ingestion/src/metadata/ingestion/ometa/mixins/tests_mixin.py:64-81`
- `openmetadata-service/src/main/java/org/openmetadata/service/resources/dqtests/TestCaseResultResource.java:90-135`

---

## 4. Incident Manager

> **담당 화면**: Incident Manager · Table → Data Observability → Incidents
>
> **경로**: `/incident-manager` · `/table/<Table FQN>/profiler/incidents`

Incident는 실패 상태와 해결 이력을 관리한다. 최초 `New`에서는 담당 Task가 없고, 사용자가 `Ack`하거나 담당자를 `Assigned`할 때 해결 Task가 생성된다. Python Runner가 Incident를 만드는 것이 아니라, OpenMetadata server가 **결과 POST 요청을 저장하는 중에 동기적으로** 연결한다.

![Failed TestCaseResult로 생성된 Incident의 New, Ack, Assigned, Resolved 상태 전이](incident-workflow.png)

### 실제 코드 1: MetadataRestSink → Result POST

Incident 생성은 `MetadataRestSink`가 `TestCaseResult`를 Server에 POST하는 요청에서 시작된다.

소스:

- `ingestion/src/metadata/ingestion/sink/metadata_rest.py:1019-1027`
- `ingestion/src/metadata/ingestion/ometa/mixins/tests_mixin.py:64-81`

```python
# [1] Sink가 Runner 결과에서 body와 URL에 넣을 TestCase FQN을 꺼낸다.
for result in record.test_results or []:
    self.metadata.add_test_case_results(
        test_results=result.testCaseResult,                    # POST body
        test_case_fqn=result.testCase.fullyQualifiedName.root, # URL의 {fqn}
    )

# [2] OMetaTestsMixin.add_test_case_results()가 실제 Server endpoint로 POST한다.
self.client.post(
    f"{self.get_suffix(TestCaseResult)}/{quote(test_case_fqn)}",
    test_results.model_dump_json(),
)
```

실제 요청은 `POST /api/v1/dataQuality/testCases/testCaseResults/{testCaseFQN}`이다. Server의 `TestCaseResultResource.addTestCaseResult()`가 요청을 받은 뒤 아래 `TestCaseResultRepository.addTestCaseResult()`에 위임하고, Repository가 같은 HTTP 요청 안에서 Incident 연결과 Result 저장을 처리한다.

### 실제 코드 2: Result POST → Incident → 시계열 저장

소스:

- `openmetadata-service/src/main/java/org/openmetadata/service/resources/dqtests/TestCaseResultResource.java:90-135`
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
| `Ack` | `EditTests` 또는 `EditAll` 권한 사용자 | 누른 사용자를 해결 Task 담당자로 연결 |
| `Assigned` | `EditTests` 또는 `EditAll` 권한 사용자 | 선택한 User에게 해결 Task 할당 |
| `Resolved` | `EditTests` 또는 `EditAll` 권한 사용자 | Incident 완료 |

정상 UI 흐름에서 `New` Incident는 실패 결과 저장 시 자동 생성되며 사용자가 별도로 만드는 단계는 없다. 이후 상태 변경과 담당자 지정은 대상 Table 또는 TestCase에 대해 `EditTests`나 `EditAll` 중 하나의 권한이 있는 사용자가 수행할 수 있다.

> **주의**: 같은 TestCase의 미해결 Incident가 있으면 재실패해도 같은 ID를 사용한다. 다음 실행이 `Success`여도 기존 Incident는 자동으로 `Resolved`되지 않는다.

---

## 5. Alert & Notification

> **담당 화면**: Observability → Alerts · Settings → Notifications
>
> **경로**: `/observability/alerts` · `/settings/notifications/alerts`

TestCase 상태 알림을 설정하는 화면은 **Observability → Alerts**다. 여기서 만든 규칙은 `alertType=Observability`이며, Settings의 `alertType=Notification` 규칙과 구분해야 한다.

| 용어 | 의미 |
|---|---|
| `Alert` / `EventSubscription` | ChangeEvent를 어떤 조건으로 검사하고 어디로 보낼지 저장한 공통 규칙 모델 |
| `Observability Alert` | `/observability/alerts`에서 만드는 품질·pipeline 상태 규칙. TestCase의 `Failed` 조건은 이 타입 |
| `Notification Alert` | `/settings/notifications/alerts`에서 만드는 별도 규칙. 고유 값은 `alertType=Notification` |
| `ActivityFeedAlert` | `alertType=ActivityFeed`인 시스템 규칙. OpenMetadata 화면의 Activity Feed를 처리 |
| `ChangeEvent` | TestCaseResult 저장이나 Incident Task 변경처럼 “무슨 일이 발생했는가”를 기록한 이벤트 |

일반 문장에서 `notification`은 알림 전달을 포괄할 수 있지만, 제품의 `Notification Alert`는 위의 특정 `alertType`을 뜻한다. 따라서 두 의미를 섞지 않는 것이 안전하다.

소스:

- `openmetadata-ui/src/main/resources/ui/src/pages/AddObservabilityPage/AddObservabilityPage.tsx:391-395`
- `openmetadata-ui/src/main/resources/ui/src/pages/AddNotificationPage/AddNotificationPage.tsx:414-418`
- `openmetadata-service/src/main/resources/json/data/EntityObservabilityFilterDescriptor.json:323-405`

```typescript
// Observability → Alerts에서 저장하는 값
<Form.Item hidden initialValue={AlertType.Observability} name="alertType" />

// Settings → Notifications에서 저장하는 값
<Form.Item hidden initialValue={AlertType.Notification} name="alertType" />
```

테스트 결과 알림은 다음 순서로 동작한다.

1. 사용자가 `Source=Test Case`, `Result=Failed`, Destination을 가진 Observability Alert를 미리 등록한다.
2. `Failed TestCaseResult`가 저장되면 Server가 `testCase / entityUpdated / testCaseResult` ChangeEvent를 만든다.
3. EventSubscription job이 이 이벤트를 읽어 기존 Alert의 Source·Result 조건과 비교한다.
4. 조건이 맞을 때만 Destination의 수신자를 찾고 설정된 채널로 메시지를 전송한다.

따라서 Incident가 생성되거나 해결될 때 새 Alert 규칙이 자동으로 생기는 것은 아니다. 실패 결과와 Incident 생성은 같은 Result POST 요청에서 일어나지만, **TestCase Observability Alert는 Failed Result의 ChangeEvent를 감시**한다.

`Resolved`는 Incident 처리 상태 변경일 뿐 `Success TestCaseResult`가 아니다. 따라서 해결 동작만으로 TestCase 결과 Alert의 `Success` 조건이 실행되지는 않는다.

### Internal Destination과 인앱 알림

Observability Alert와 Notification Alert의 Destination에는 서로 독립적인 두 값이 있다.

- `category`: **수신자를 어디에서 찾는가**. `Users`, `Teams`, `Owners`, `Followers`, `Admins` 등은 OpenMetadata에 저장된 사용자·관계에서 수신자를 찾고, `External`은 Alert 설정에 주소나 endpoint를 직접 넣는다.
- `type`: **어떤 채널로 전달하는가**. 사용자 Alert UI에서는 `Email`, `Slack`, `MsTeams`, `GChat`, `Webhook`을 선택한다.

`Users`, `Followers`, `Admins`를 선택하면 UI가 `Email`만 허용하고, Server가 OpenMetadata User의 `email`을 읽어 보낸다. `Teams`와 `Owners`는 Email 외에 대상 User·Team Profile에 저장된 Slack·MS Teams·GChat·Generic Webhook 설정을 사용할 수 있다. 대상 Profile에 해당 webhook이 없으면 그 수신자는 만들어지지 않는다. `External`은 특정 OpenMetadata User를 찾는 방식이 아니라 Alert에 이메일 주소나 webhook endpoint를 직접 저장하는 방식이다.

소스:

- `openmetadata-ui/src/main/resources/ui/src/utils/Alerts/AlertsUtil.tsx:317-331`
- `openmetadata-service/src/main/java/org/openmetadata/service/notifications/recipients/context/Recipient.java:21-58`
- `openmetadata-service/src/main/java/org/openmetadata/service/notifications/recipients/context/EmailRecipient.java:30-55`
- `openmetadata-service/src/main/java/org/openmetadata/service/notifications/recipients/context/WebhookRecipient.java:63-86, 180-207`
- `openmetadata-service/src/main/java/org/openmetadata/service/notifications/recipients/strategy/impl/ExternalRecipientResolver.java:31-87`

```java
// Internal category: User/Team을 찾은 뒤 type에 맞는 연락처로 변환한다.
if (notificationType == SubscriptionType.EMAIL) {
  return EmailRecipient.fromUser(user);       // user.email
}
return WebhookRecipient.fromUser(user, notificationType); // profile.subscription.*

// External category: Alert에 직접 넣은 최종 주소/endpoint를 사용한다.
if (notificationType == SubscriptionType.EMAIL) {
  return action.getReceivers().stream()
      .map(EmailRecipient::new)                // email 주소마다 Recipient 생성
      .collect(Collectors.toUnmodifiableSet());
}
Webhook webhook = JsonUtils.convertValue(destination.getConfig(), Webhook.class);
return Set.of(new WebhookRecipient(webhook)); // Slack·MS Teams·GChat·Generic Webhook
```

따라서 UI의 **Internal Destination**은 “OpenMetadata 화면 안에 띄운다”는 뜻이 아니다. OpenMetadata의 User·Team·Owner 관계에서 대상을 찾는다는 뜻이며, 찾은 대상에게 실제로 보내는 방법은 `type`이 정한다. 예를 들어 `category=Owners`, `type=Email`이면 이벤트 대상의 Owner를 조회한 뒤 이메일을 보낸다.

#### “각 사용자마다 외부 채널을 등록하나?”

Alert를 만든 사람이 수신자 category와 type을 한 번 정한다. `Users`, `Followers`, `Admins`는 현재 UI에서 Email만 선택할 수 있으므로, 선택된 User의 `email`로 보낸다. User Profile에 Slack URL을 넣어 두었다고 해서 이 category가 개인별 Slack 발송으로 바뀌지는 않는다. `Teams`와 `Owners`에서 Slack·MS Teams·GChat·Webhook을 선택한 경우에만, 관계로 찾은 User·Team Profile의 `profile.subscription.*` endpoint가 사용된다. `External`은 사용자 Profile과 무관하게 Alert에 직접 넣은 이메일 주소 또는 webhook endpoint를 사용한다.

`ActivityFeed`도 backend의 `SubscriptionType`에는 있지만 사용자 Alert UI 선택지에서는 제외된다. 이 type은 아래의 시스템 Activity Feed 경로에서 사용한다.

### ActivityFeedAlert의 위치

`ActivityFeedAlert`는 `EventSubscription`을 이용하지만, 사용자가 만드는 Observability Alert나 Notification Alert가 아니다. 시스템이 미리 만든 `alertType=ActivityFeed`, `provider=system` 규칙이며, `ActivityFeedPublisher`가 Feed Thread를 저장하고 WebSocket으로 화면에 전달한다. Settings → Notifications 화면에 보일 수 있는 이유는 UI가 목록을 조회할 때 `ActivityFeedAlert`를 이름으로 별도 조회해 함께 붙이기 때문이다. 즉, **같은 ChangeEvent/EventSubscription 인프라를 공유하지만 Alert type·필터·전달 publisher가 분리**되어 있다.

따라서 `Alert`라는 상위 개념 전체가 외부 채널 전용인 것은 아니다. 사용자가 만드는 TestCase Observability Alert와 Notification Alert는 Email·Slack·MS Teams·GChat·Webhook 전달을 사용하고, 시스템 ActivityFeedAlert는 OpenMetadata 내부 Feed로 전달한다.

소스:

- `openmetadata-service/src/main/resources/json/data/eventsubscription/ActivityFeedEvents.json:2-6, 36-41, 116-118`
- `openmetadata-ui/src/main/resources/ui/src/pages/NotificationListPage/NotificationListPage.tsx:153-165`
- `openmetadata-service/src/main/java/org/openmetadata/service/apps/bundles/changeEvent/feed/ActivityFeedPublisher.java:57-71`

소스:

- `openmetadata-spec/src/main/resources/json/schema/events/eventSubscription.json:78-109, 120-176`
- `openmetadata-ui/src/main/resources/ui/src/constants/Alerts.constants.tsx:41-47`
- `openmetadata-service/src/main/java/org/openmetadata/service/notifications/recipients/RecipientResolver.java:127-155`
- `openmetadata-service/src/main/java/org/openmetadata/service/apps/bundles/changeEvent/AlertFactory.java:14-25`

```java
// [1] category로 수신자 해석 방법을 선택한다.
SubscriptionCategory category = destination.getCategory();
RecipientResolutionStrategy strategy = STRATEGIES.get(category);
recipients.addAll(strategy.resolve(event, action, destination));

// [2] type으로 실제 전송 publisher를 선택한다.
return switch (config.getType()) {
  case EMAIL -> new EmailPublisher(subscription, config);
  case SLACK -> new SlackEventPublisher(subscription, config);
  case MS_TEAMS -> new MSTeamsPublisher(subscription, config);
  case WEBHOOK -> new GenericPublisher(subscription, config);
  case ACTIVITY_FEED -> new ActivityFeedPublisher(subscription, config);
  // GChat, Governance Workflow 생략
};
```

OpenMetadata 화면 안 알림은 다음 두 경로가 별도로 처리한다.

1. **Activity Feed**: 시스템 `ActivityFeedAlert`가 ChangeEvent를 Feed Thread로 저장하고 `activityFeed` WebSocket 채널로 방송한다. 기본 필터는 `testCase`·`testSuite`를 제외하고 `testCaseResult` 변경 필드도 포함하지 않으므로, Failed TestCase Observability Alert가 Activity Feed까지 자동 생성하지는 않는다.
2. **Incident 담당 Task**: 사용자가 Incident를 `Ack`하거나 `Assigned`할 때 Task Thread를 만들고 담당자에게 `taskChannel` WebSocket으로 보낸다. 이 경로는 TestCase Observability Alert의 Destination 발송과 별개다.

소스:

- `openmetadata-service/src/main/resources/json/data/eventsubscription/ActivityFeedEvents.json:2-40, 116-118`
- `openmetadata-service/src/main/java/org/openmetadata/service/apps/bundles/changeEvent/feed/ActivityFeedPublisher.java:57-71`
- `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/TestCaseResolutionStatusRepository.java:391-442`
- `openmetadata-service/src/main/java/org/openmetadata/service/util/WebsocketNotificationHandler.java:156-179`

```java
// Activity Feed: Feed를 저장하고 전체 feed 채널에 방송한다.
feedRepository.create(thread, changeEvent);
WebSocketManager.getInstance().broadCastMessageToAll(
    WebSocketManager.FEED_BROADCAST_CHANNEL, JsonUtils.pojoToJson(thread));

// Incident Task: Task 담당자에게만 별도 WebSocket 메시지를 보낸다.
WebSocketManager.getInstance().sendToManyWithUUID(
    receiversList, WebSocketManager.TASK_BROADCAST_CHANNEL, jsonThread);
```

즉, Failed TestCase **Observability Alert 자체는** 사용자 UI에서 선택한 Email·Slack·MS Teams·GChat·Webhook 같은 외부 전달 채널로만 보낸다. 이때 `Internal Destination`은 내부 화면 알림이 아니라 수신자를 OpenMetadata에서 찾는 방식이다. `Notification Alert`는 외부 메시지의 동의어가 아니라 Settings의 별도 `alertType`이며, Activity Feed와 Incident Task 인앱 알림은 다시 별도 경로다. Incident Task를 외부 채널로도 보내려면 **Settings → Notifications**에서 Task 이벤트를 대상으로 한 EventSubscription을 따로 설정해야 한다.

![Failed TestCaseResult의 Incident 동기 처리와 Event Subscription 비동기 알림 경로](alert-notification-flow.png)

그림에서 `Incident 연결`은 같은 POST 안의 동기 처리이고, 실제 알림 경로는 `Result 저장 → ChangeEvent 저장 → EventSubscription scheduler`다.

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

### Failed TestCase Observability Alert 설정값

| 구분 | 실제 값 | 결정 위치 |
|---|---|---|
| Source | `Test Case` | Alert UI |
| Action | `Get Test Case Status Updates` | Alert UI |
| Result | `Failed` | Alert UI. `Success`, `Aborted`, `Queued`도 선택 가능 |
| 수신자 `category` | `Users`, `Teams`, `Owners`, `Followers`, `Admins` 또는 `External` | Alert UI |
| 전달 `type` | `Email`, `Slack`, `MsTeams`, `GChat`, `Webhook` 등 | Alert UI |
| ChangeEvent | `entityType=testCase`, `eventType=entityUpdated` | Server 내부 |
| 변경 필드 | `testCaseResult` | Server 내부 |

Test 결과 상태 `Success / Failed / Aborted / Queued`와 Incident 처리 상태 `New / Ack / Assigned / Resolved`는 서로 다르다. 정상 완료 알림은 `Success`, 판정 실패 알림은 `Failed`, 실행 중단 알림은 `Aborted`를 선택한다.

> **주의**: Alert 설정이 없거나 조건이 맞지 않으면 테스트가 실행돼도 알림은 전송되지 않는다. Incident의 `Ack`·`Assigned` 때 Task 담당자에게 가는 인앱 알림은 EventSubscription Alert와 별개다.
