# Ingestion Create/Deploy 호출 흐름과 UI 유발 작업 목록

조사 기준: OpenMetadata `1.13.0-release` (`f329dd4a`)

- 현재 소스에서 확인되는 실제 호출만 기록한다.
- 1장은 `Database Service > Agents > Add Ingestion > Metadata > Add & Deploy`의 call stack이다.
- 2장은 UI에서 시작해 Java를 거쳐 Airflow 또는 Ingestion Python 작업으로 이어지는 변경성 API 목록이다.
- 줄 번호는 조사 기준 커밋의 번호다.

---

## 1. Ingestion Pipeline Create/Deploy call stack

```text
Metadata Agent 선택 → Create API → Java DB 저장 → Pipeline ID 반환
                    → Deploy API → Airflow Managed API → DAG 등록
```

Create와 Deploy는 서로 다른 요청이다. Create 성공 후 UI가 반환받은 ID로 Deploy를 호출한다.

주요 파일:

- `openmetadata-ui/src/main/resources/ui/src/components/Settings/Services/Ingestion/AddIngestionButton.component.tsx`
- `openmetadata-ui/src/main/resources/ui/src/pages/AddIngestionPage/AddIngestionPage.component.tsx`
- `openmetadata-ui/src/main/resources/ui/src/rest/ingestionPipelineAPI.ts`
- `openmetadata-service/src/main/java/org/openmetadata/service/resources/services/ingestionpipelines/IngestionPipelineResource.java`
- `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/EntityRepository.java`
- `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/IngestionPipelineRepository.java`
- `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/EntityDAO.java`
- `openmetadata-service/src/main/java/org/openmetadata/service/clients/pipeline/airflow/AirflowRESTClient.java`
- `openmetadata-airflow-apis/openmetadata_managed_apis/api/routes/deploy.py`
- `openmetadata-airflow-apis/openmetadata_managed_apis/operations/deploy.py`

| 단계 | 파일 | 함수/API |
|---|---|---|
| Metadata 선택 | `AddIngestionButton.component.tsx:46` | `handleAddIngestionClick()`; Add 화면 이동, API 없음 |
| UI Create | `AddIngestionPage.component.tsx:147` | `onAddIngestionSave()` → `addIngestionPipeline()` |
| UI REST | `ingestionPipelineAPI.ts:33` | `POST /api/v1/services/ingestionPipelines` |
| Java 진입 | `IngestionPipelineResource.java:544` | `@POST create(CreateIngestionPipeline)` (`:559`) |
| Java 생성 | 같은 파일 `:199` | `create(UriInfo, SecurityContext, IngestionPipeline)` → `repository.create()` |
| UI Deploy | `AddIngestionPage.component.tsx:120,155` | `onIngestionDeploy()` → `POST /api/v1/services/ingestionPipelines/deploy/{id}` |
| Java Deploy | `IngestionPipelineResource.java:677,1309` | `deployIngestion()` → `deployPipelineInternal()` |
| Airflow Client | `AirflowRESTClient.java:327` | `deployPipeline()` → `POST /pluginsv2/api/v2/openmetadata/deploy` |
| Airflow 진입 | `api/routes/deploy.py:58` | `deploy_dag()` |
| DAG 생성 | `operations/deploy.py:186` | `DagDeployer.deploy()` |

Java DB 저장 call stack:

```text
IngestionPipelineResource.create()                      :559
  → IngestionPipelineMapper.createToEntity()
  → IngestionPipelineResource.create()                  :199
  → EntityRepository.create()                          :2527
  → createInternal()                                   :2541
  → createNewEntity()                                  :4470
  → createNewEntityFlush()                             :4485
  → IngestionPipelineRepository.storeEntity()           :428
  → EntityRepository.store()                           :4555
  → EntityDAO.insert()                                  :139
  → INSERT INTO ingestion_pipeline_entity (fqnHash, json)
```

- 관계 저장: `EntityRepository.storeRelationshipsInternal()` → `IngestionPipelineRepository.storeRelationships()` → Service와 `CONTAINS` 관계 저장
- Create 단계는 Airflow를 호출하지 않는다.
- `DagDeployer.deploy()`는 `{dag_id}.json`과 DAG Python 파일을 만들고 Airflow에 등록한다.
- 실제 Ingestion은 스케줄 도래 또는 `POST .../trigger/{id}` 이후 실행된다.
- 실행 상태는 Python이 `PUT /api/v1/services/ingestionPipelines/{fqn}/pipelineStatus`로 Java에 저장한다.

---

## 2. UI에서 유발되어 Airflow/Ingestion으로 이어지는 작업

### 2.1 Ingestion Pipeline 직접 작업

공통 UI 파일:

- `openmetadata-ui/src/main/resources/ui/src/pages/AddIngestionPage/AddIngestionPage.component.tsx`
- `openmetadata-ui/src/main/resources/ui/src/pages/EditIngestionPage/EditIngestionPage.component.tsx`
- `openmetadata-ui/src/main/resources/ui/src/components/Settings/Services/Ingestion/IngestionListTable/PipelineActions/PipelineActions.tsx`
- `openmetadata-ui/src/main/resources/ui/src/components/Settings/Services/Ingestion/IngestionListTable/PipelineActions/PipelineActionsDropdown.tsx`
- `openmetadata-ui/src/main/resources/ui/src/components/Settings/Services/Ingestion/IngestionListTable/IngestionListTable.tsx`

| UI 위치/작업 | Java API URL | 실제 의미 |
|---|---|---|
| Service Details > Agents > Add Agent > Add & Deploy | `POST /api/v1/services/ingestionPipelines` → `POST /api/v1/services/ingestionPipelines/deploy/{id}` | Java DB 생성 후 Airflow DAG 등록 |
| Agents > Edit | `PATCH /api/v1/services/ingestionPipelines/{id}` → `POST /api/v1/services/ingestionPipelines/deploy/{id}` | 설정 저장 후 DAG 재배포 |
| Agents > Run | `POST /api/v1/services/ingestionPipelines/trigger/{id}` | 즉시 실행 |
| Agents > Deploy / Re-deploy | `POST /api/v1/services/ingestionPipelines/deploy/{id}` | DAG 등록/갱신 |
| Agents > Pause / Resume | `POST /api/v1/services/ingestionPipelines/toggleIngestion/{id}` | Airflow DAG 활성/비활성 변경 |
| Agents > Kill | `POST /api/v1/services/ingestionPipelines/kill/{id}` | 실행 중인 Workflow 중지 |
| Agents > Delete | `DELETE /api/v1/services/ingestionPipelines/{id}?hardDelete=true` | Java 엔티티와 Airflow DAG 삭제 |
| Settings > Services > Pipelines > Bulk Re-deploy | 선택한 Pipeline마다 `POST /api/v1/services/ingestionPipelines/deploy/{id}` | 선택 DAG 재배포 |

스케줄은 별도 API가 아니다. Add/Edit 요청의 `airflowConfig.scheduleInterval`에 포함된다. 예약 시각에 실행되는 것은 Airflow Scheduler이므로 그 시점에는 UI 요청이 없다.

### 2.2 Data Quality Pipeline

UI 파일:

- `openmetadata-ui/src/main/resources/ui/src/components/DataQuality/AddDataQualityTest/TestSuiteIngestion.tsx`
- `openmetadata-ui/src/main/resources/ui/src/components/DataQuality/TestSuite/TestSuitePipelineTab/TestSuitePipelineTab.component.tsx`
- `openmetadata-ui/src/main/resources/ui/src/components/DataQuality/BundleSuiteForm/BundleSuiteForm.tsx`
- `openmetadata-ui/src/main/resources/ui/src/components/DataQuality/AddDataQualityTest/components/TestCaseFormV1.tsx`

| UI 위치/작업 | Java API URL | 조건 |
|---|---|---|
| Test Suite > Pipelines > Add | `POST /api/v1/services/ingestionPipelines` → `POST /api/v1/services/ingestionPipelines/deploy/{id}` | TestSuite Pipeline 생성/배포 |
| Test Suite > Pipelines > Edit | `PATCH /api/v1/services/ingestionPipelines/{id}` → `POST /api/v1/services/ingestionPipelines/deploy/{id}` | 설정·스케줄 수정 |
| Test Suite 생성 시 Pipeline 함께 생성 | 위 Add API와 동일 | Airflow 사용 가능 시 Deploy |
| Test Case 생성 시 Pipeline 자동 생성 | 위 Add API와 동일 | 기존 Pipeline이 없고 생성 조건을 만족할 때 |
| Test Suite Pipeline Run/Pause/Kill/Delete | 2.1의 동일 API | 공유 `IngestionListTable` 사용 |

### 2.3 AutoPilot을 통한 Agent 생성·실행

| UI 위치/작업 | UI 파일/함수 | Java API URL |
|---|---|---|
| Add Service 완료 | `openmetadata-ui/src/main/resources/ui/src/pages/AddServicePage/AddServicePage.component.tsx` — `triggerTheAutoPilotApplication()` | `POST /api/v1/services/{serviceCategory}` → `POST /api/v1/apps/trigger/AutoPilotApplication` |
| Service Details > Trigger AutoPilot | `openmetadata-ui/src/main/resources/ui/src/components/DataAssets/DataAssetsHeader/DataAssetsHeader.component.tsx` — `triggerTheAutoPilotApplication()` | `POST /api/v1/apps/trigger/AutoPilotApplication` |

`AutoPilotWorkflow.json`에는 Metadata, Lineage, Usage Pipeline의 생성·Deploy·Run 작업이 있다. AutoPilot이 비활성 상태이거나 제외된 Service Type이면 실행하지 않는다.

### 2.4 Test Connection

- UI 파일: `openmetadata-ui/src/main/resources/ui/src/components/common/TestConnection/TestConnection.tsx`
- UI 위치: Add Service의 Connection 설정, Service Details의 Edit Connection

```text
POST /api/v1/automations/workflows
  → POST /api/v1/automations/workflows/trigger/{workflowId}
```

첫 번째 API는 임시 Workflow 엔티티를 생성하고, 두 번째 API가 Airflow의 Test Connection Workflow를 실행한다.

### 2.5 External Application

UI 파일:

- `openmetadata-ui/src/main/resources/ui/src/pages/AppInstall/AppInstall.component.tsx`
- `openmetadata-ui/src/main/resources/ui/src/components/Settings/Applications/AppDetails/AppDetails.component.tsx`
- `openmetadata-ui/src/main/resources/ui/src/components/Modals/StopScheduleRun/StopScheduleRunModal.tsx`

아래 항목은 `appType=External`인 Application에만 해당한다.

| UI 작업 | Java API URL | 실제 의미 |
|---|---|---|
| App 설치 및 스케줄 정의 | `POST /api/v1/apps` | 예약형 External App의 IngestionPipeline 정의 생성 |
| App Deploy | `POST /api/v1/apps/deploy/{appName}` | Airflow DAG 등록 |
| App Run | `POST /api/v1/apps/trigger/{appName}` | 즉시 실행 |
| App 실행 중지 | `POST /api/v1/apps/stop/{appName}` 또는 `?runId={runId}` | 전체 또는 특정 실행 중지 |
| App 설정 수정 | `PATCH /api/v1/apps/{id}` | 배포된 External Pipeline의 source 설정 변경 시 자동 재배포 가능 |
| App Disable/Uninstall | `DELETE /api/v1/apps/name/{appName}?hardDelete={true\|false}` | External App의 Airflow DAG 삭제 |

현재 `AbstractNativeApplication.scheduleExternal()`은 기존 Pipeline을 갱신할 때 App 설정만 `sourceConfig`에 반영한다. UI에서 변경한 cron을 기존 Pipeline의 `airflowConfig.scheduleInterval`에 복사하는 코드는 확인되지 않는다. 따라서 기존 External App의 스케줄 변경이 Airflow 스케줄 변경으로 이어진다고 볼 수 없다.

Internal Application의 예약 실행은 Java Quartz 경로이므로 이 목록에서 제외한다.

### 2.6 Data Contract 품질 검증

UI 파일:

- `openmetadata-ui/src/main/resources/ui/src/components/DataContract/AddDataContract/AddDataContract.tsx`
- `openmetadata-ui/src/main/resources/ui/src/components/DataContract/ContractDetailTab/ContractDetail.tsx`
- `openmetadata-ui/src/main/resources/ui/src/components/DataContract/ContractTab/ContractTab.tsx`
- `openmetadata-ui/src/main/resources/ui/src/components/DataContract/ODCSImportModal/ODCSImportModal.component.tsx`

아래 경로는 Contract에 `qualityExpectations`가 있을 때 TestSuite Ingestion Pipeline으로 이어진다.

| UI 작업 | Java API URL | 실제 의미 |
|---|---|---|
| Contract 생성 | `POST /api/v1/dataContracts` | 필요한 TestSuite Pipeline이 없으면 생성·Deploy |
| Contract 수정 | `PATCH /api/v1/dataContracts/{id}` | 품질 조건 추가 시 필요한 Pipeline 생성·Deploy |
| ODCS YAML 신규 Import | `POST /api/v1/dataContracts/odcs/yaml?...` | Contract 생성과 동일 |
| ODCS YAML Merge/Replace | `PUT /api/v1/dataContracts/odcs/yaml?...` | Contract 생성/수정과 동일 |
| Run Now | `POST /api/v1/dataContracts/{id}/validate` | TestSuite Pipeline 즉시 실행 |
| 상속 Contract Run Now | `POST /api/v1/dataContracts/entity/validate?entityId={id}&entityType={type}` | 유효 Contract의 TestSuite Pipeline 즉시 실행 |
| Contract 삭제 | `DELETE /api/v1/dataContracts/{id}?hardDelete=true&recursive=true` | 연결된 TestSuite Pipeline과 Airflow DAG 삭제 |

### 2.7 상위 엔티티 삭제에 따른 간접 Pipeline 삭제

공통 UI 파일:

- `openmetadata-ui/src/main/resources/ui/src/components/DataAssets/DataAssetsHeader/DataAssetsHeader.component.tsx`
- `openmetadata-ui/src/main/resources/ui/src/context/AsyncDeleteProvider/AsyncDeleteProvider.tsx`

| UI 삭제 위치 | Java API URL | Airflow까지 이어지는 조건 |
|---|---|---|
| Service Details > Delete | `DELETE /api/v1/services/{serviceCategory}/async/{id}?recursive=true&hardDelete={boolean}` | Service에 Ingestion Pipeline이 있을 때 |
| Database Details > Delete | `DELETE /api/v1/databases/async/{id}?recursive=true&hardDelete={boolean}` | 하위 TestSuite에 Pipeline이 있을 때 |
| Database Schema Details > Delete | `DELETE /api/v1/databaseSchemas/async/{id}?recursive=true&hardDelete={boolean}` | 하위 TestSuite에 Pipeline이 있을 때 |
| Table Details > Delete | `DELETE /api/v1/tables/async/{id}?recursive=true&hardDelete={boolean}` | Table의 TestSuite에 Pipeline이 있을 때 |
| Test Suite Details > Delete | `DELETE /api/v1/dataQuality/testSuites/{id}?recursive=true&hardDelete=true` | TestSuite에 Pipeline이 있을 때 |

재귀 삭제로 `IngestionPipelineRepository.postDelete()`에 도달하면 `pipelineServiceClient.deletePipeline()`이 Airflow DAG를 삭제한다.

### 2.8 제외 범위

- Airflow 상태, 로그, 실행 이력, Health 조회 API
- UI 호출 없이 백엔드·CLI에서만 시작하는 경로
- Internal Application의 Java Quartz 실행
- Airflow Scheduler가 예약 시각에 자체적으로 시작하는 실행: UI는 Add/Edit 시 스케줄만 저장한다.
