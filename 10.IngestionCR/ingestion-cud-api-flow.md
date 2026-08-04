# Ingestion Pipeline C/U/D 호출 위치

조사 기준: OpenMetadata `f329dd4a` (줄 번호도 해당 커밋 기준)

```text
Create/Update: UI → Java 저장 → UI가 Deploy 호출 → Java → Airflow DAG 등록
Delete:        UI → Java 삭제 → Java → Airflow DAG 삭제
실제 실행:     Airflow DAG 실행 → Python Ingestion → Java 결과 저장 → UI 조회
```

## 1. Create 예시

대상 API: `POST /api/v1/services/ingestionPipelines`

1. UI 저장 버튼
   - `openmetadata-ui/src/main/resources/ui/src/pages/AddIngestionPage/AddIngestionPage.component.tsx:147`
   - `onAddIngestionSave()` → `addIngestionPipeline(data)`

2. UI에서 Java API 호출
   - `openmetadata-ui/src/main/resources/ui/src/rest/ingestionPipelineAPI.ts:33`
   - `addIngestionPipeline()` → `POST /services/ingestionPipelines`

3. Java API 진입
   - `openmetadata-service/src/main/java/org/openmetadata/service/resources/services/ingestionpipelines/IngestionPipelineResource.java:559`
   - `create(CreateIngestionPipeline)` → 같은 파일 `:199`의 `create(...)`

4. Java DB 저장
   - `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/EntityRepository.java:2527`
   - `create()` → 생성된 Pipeline `id`를 UI에 반환

5. UI가 Deploy API 호출
   - `AddIngestionPage.component.tsx:120` — `onIngestionDeploy(id)`
   - `ingestionPipelineAPI.ts:92` — `deployIngestionPipelineById()`
   - `POST /services/ingestionPipelines/deploy/{id}`

6. Java Deploy API 진입
   - `IngestionPipelineResource.java:677` — `deployIngestion()`
   - `IngestionPipelineResource.java:1309` — `deployPipelineInternal()`

7. Java가 Airflow API 호출
   - `openmetadata-service/src/main/java/org/openmetadata/service/clients/pipeline/airflow/AirflowRESTClient.java:327`
   - `deployPipeline()` → `POST /pluginsv2/api/v2/openmetadata/deploy` (Airflow 3.x)

8. Airflow API 진입
   - `openmetadata-airflow-apis/openmetadata_managed_apis/api/routes/deploy.py:58`
   - `deploy_dag()` → `DagDeployer.deploy()`

9. Airflow DAG 생성
   - `openmetadata-airflow-apis/openmetadata_managed_apis/operations/deploy.py:186`
   - `DagDeployer.deploy()` → 설정 JSON과 DAG 파일 생성 및 등록

## 2. Update 예시

대상 API: `PATCH /api/v1/services/ingestionPipelines/{id}`

1. UI 저장 버튼
   - `openmetadata-ui/src/main/resources/ui/src/pages/EditIngestionPage/EditIngestionPage.component.tsx:188`
   - `onEditIngestionSave()` → `updateIngestionPipeline(id, jsonPatch)`

2. UI에서 Java API 호출
   - `openmetadata-ui/src/main/resources/ui/src/rest/ingestionPipelineAPI.ts:110`
   - `updateIngestionPipeline()` → `PATCH /services/ingestionPipelines/{id}`

3. Java API 진입
   - `IngestionPipelineResource.java:582` — `updateDescription()`
   - `EntityResource.java:599` — `patchInternal()`
   - `EntityRepository.java:3648` — `patch()`

4. Ingestion Pipeline 전용 수정 처리
   - `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/IngestionPipelineRepository.java:804`
   - `IngestionPipelineUpdater.entitySpecificUpdate()`

5. UI가 Deploy API 호출
   - `EditIngestionPage.component.tsx:161` — `onIngestionDeploy()`
   - 이후 Create의 6~9번과 동일

참고: 이미 배포된 Pipeline의 일부 설정을 수정하면 `IngestionPipelineRepository.java:827`의 `deployIfRequired()`에서도 자동 Deploy할 수 있다.

## 3. Delete 예시

대상 API: `DELETE /api/v1/services/ingestionPipelines/{id}?hardDelete=true`

1. UI 삭제 버튼
   - `openmetadata-ui/src/main/resources/ui/src/components/Settings/Services/Ingestion/IngestionListTable/IngestionListTable.tsx:138`
   - `deleteIngestion()` → `deleteIngestionPipelineById(id)`

2. UI에서 Java API 호출
   - `openmetadata-ui/src/main/resources/ui/src/rest/ingestionPipelineAPI.ts:104`
   - `deleteIngestionPipelineById()` → `DELETE /services/ingestionPipelines/{id}?hardDelete=true`

3. Java API 진입 및 DB 삭제
   - `IngestionPipelineResource.java:871` — `delete()`
   - `EntityResource.java:682` — `delete()`
   - `EntityRepository.java:3963` — `delete()`

4. Ingestion Pipeline 삭제 후처리
   - `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/IngestionPipelineRepository.java:495`
   - `postDelete()` → `pipelineServiceClient.deletePipeline()`

5. Java가 Airflow API 호출
   - `AirflowRESTClient.java:355`
   - `deletePipeline()` → `DELETE /pluginsv2/api/v2/openmetadata/delete?dag_id={pipelineName}` (Airflow 3.x)

6. Airflow API 진입 및 DAG 삭제
   - `openmetadata-airflow-apis/openmetadata_managed_apis/api/routes/delete.py:54` — `delete_dag()`
   - `openmetadata-airflow-apis/openmetadata_managed_apis/operations/delete.py:27` — `delete_dag_id()`

Delete에는 후속 Deploy가 없다.

## 4. Airflow가 Ingestion을 실행하는 위치

Deploy는 DAG 등록까지만 한다. DAG가 스케줄 또는 Trigger됐을 때 아래 함수가 실행된다.

1. Airflow PythonOperator
   - `openmetadata-airflow-apis/openmetadata_managed_apis/workflows/ingestion/common.py:208`
   - `metadata_ingestion_workflow()` → `MetadataWorkflow.create()` → `workflow.execute()`

2. Python Ingestion
   - `ingestion/src/metadata/workflow/base.py:269`
   - `execute()` → Source 데이터를 읽어 Sink를 통해 Java API에 저장

## 5. Ingestion → Java 상태 저장 예시

1. 상태 저장 시작
   - `ingestion/src/metadata/workflow/workflow_status_mixin.py:103`
   - `set_ingestion_pipeline_status()` → `create_or_update_pipeline_status()`

2. Java API 호출 경로 생성
   - `ingestion/src/metadata/ingestion/ometa/mixins/ingestion_pipeline_mixin.py:40`
   - `create_or_update_pipeline_status()` → `PUT /api/v1/services/ingestionPipelines/{fqn}/pipelineStatus`

3. Python 공통 HTTP 호출
   - `ingestion/src/metadata/ingestion/ometa/client.py:177`
   - `REST._request()`

4. Java API 진입
   - `IngestionPipelineResource.java:1153`
   - `addPipelineStatus()`

## 6. Java → Airflow / Ingestion → Java 앞단

- Java에서 Airflow로 나가는 앞단: `AirflowRESTClient.deployPipeline()`, `deletePipeline()`, `runPipeline()`
- Airflow에서 받는 앞단: `api/routes/deploy.py:deploy_dag()`, `delete.py:delete_dag()`, `trigger.py:trigger_dag()`
- Ingestion에서 Java로 나가는 공통 앞단: `ingestion/ometa/client.py:REST._request()`
- Java에서 상태를 받는 앞단: `IngestionPipelineResource.addPipelineStatus()`

UI는 Ingestion이 직접 호출하지 않는다. `ingestionPipelineAPI.ts:155`의 `getRunHistoryForPipeline()`으로 Java에 저장된 상태를 조회한다.
