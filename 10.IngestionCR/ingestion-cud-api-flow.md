# Ingestion Pipeline C/U/D 호출 위치

- 조사 기준: OpenMetadata `f329dd4a`
- 아래 줄 번호는 위 커밋 기준이다.

## 전체 순서

```text
Create/Update: UI → Java 저장 → UI → Deploy API → Java → Airflow DAG 등록
Delete:        UI → Java 삭제 → Airflow DAG 삭제
실제 실행:     Airflow DAG 실행 → Python Ingestion → Java에 결과 저장 → UI가 조회
```

## 1. Create 예시

API: `POST /api/v1/services/ingestionPipelines`

### 1) UI에서 저장 버튼 처리

- 파일: `openmetadata-ui/src/main/resources/ui/src/pages/AddIngestionPage/AddIngestionPage.component.tsx:147`
- 함수: `onAddIngestionSave()`
- 호출: `addIngestionPipeline(data)`

### 2) UI에서 Java API 호출

- 파일: `openmetadata-ui/src/main/resources/ui/src/rest/ingestionPipelineAPI.ts:33`
- 함수: `addIngestionPipeline()`
- 요청: `POST /services/ingestionPipelines`

### 3) Java API 진입

- 파일: `openmetadata-service/src/main/java/org/openmetadata/service/resources/services/ingestionpipelines/IngestionPipelineResource.java:559`
- 함수: `create(CreateIngestionPipeline create)`
- 다음 호출: 같은 파일의 `create(uriInfo, securityContext, ingestionPipeline)`

### 4) Java에서 DB 저장

- 파일: `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/EntityRepository.java:2527`
- 함수: `create()`
- 결과: 생성된 Ingestion Pipeline과 `id`를 UI에 반환

### 5) UI에서 Deploy 호출

- 파일: `openmetadata-ui/src/main/resources/ui/src/pages/AddIngestionPage/AddIngestionPage.component.tsx:120`
- 함수: `onIngestionDeploy(res.id)`
- 다음 호출: `deployIngestionPipelineById(id)`

- 파일: `openmetadata-ui/src/main/resources/ui/src/rest/ingestionPipelineAPI.ts:92`
- 함수: `deployIngestionPipelineById()`
- 요청: `POST /services/ingestionPipelines/deploy/{id}`

### 6) Java Deploy API 진입

- 파일: `openmetadata-service/src/main/java/org/openmetadata/service/resources/services/ingestionpipelines/IngestionPipelineResource.java:677`
- 함수: `deployIngestion()`
- 다음 호출: 같은 파일 `:1309`의 `deployPipelineInternal()`

### 7) Java에서 Airflow API 호출

- 파일: `openmetadata-service/src/main/java/org/openmetadata/service/clients/pipeline/airflow/AirflowRESTClient.java:327`
- 함수: `deployPipeline()`
- 요청: `POST /pluginsv2/api/v2/openmetadata/deploy` (Airflow 3.x 기준)

### 8) Airflow API 진입

- 파일: `openmetadata-airflow-apis/openmetadata_managed_apis/api/routes/deploy.py:58`
- 함수: `deploy_dag()`
- 다음 호출: `DagDeployer.deploy()`

### 9) Airflow DAG 생성

- 파일: `openmetadata-airflow-apis/openmetadata_managed_apis/operations/deploy.py:186`
- 함수: `DagDeployer.deploy()`
- 처리: 설정 JSON과 DAG Python 파일 생성 후 Airflow에 DAG 등록

## 2. Update 예시

API: `PATCH /api/v1/services/ingestionPipelines/{id}`

### 1) UI에서 저장 버튼 처리

- 파일: `openmetadata-ui/src/main/resources/ui/src/pages/EditIngestionPage/EditIngestionPage.component.tsx:188`
- 함수: `onEditIngestionSave()`
- 호출: `updateIngestionPipeline(id, jsonPatch)`

### 2) UI에서 Java API 호출

- 파일: `openmetadata-ui/src/main/resources/ui/src/rest/ingestionPipelineAPI.ts:110`
- 함수: `updateIngestionPipeline()`
- 요청: `PATCH /services/ingestionPipelines/{id}`

### 3) Java API 진입

- 파일: `openmetadata-service/src/main/java/org/openmetadata/service/resources/services/ingestionpipelines/IngestionPipelineResource.java:582`
- 함수: `updateDescription()`
- 다음 호출: `patchInternal()`

### 4) Java에서 DB 수정

- 파일: `openmetadata-service/src/main/java/org/openmetadata/service/resources/EntityResource.java:599`
- 함수: `patchInternal()`
- 다음 호출: `EntityRepository.patch()`

- 파일: `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/EntityRepository.java:3648`
- 함수: `patch()`
- Ingestion 전용 후처리: `IngestionPipelineRepository.java:804`의 `entitySpecificUpdate()`

### 5) UI에서 Deploy 호출

- 파일: `openmetadata-ui/src/main/resources/ui/src/pages/EditIngestionPage/EditIngestionPage.component.tsx:161`
- 함수: `onIngestionDeploy()`
- 요청 함수: `deployIngestionPipelineById(id)`

이후 Create의 6~9번과 동일하게 Java를 거쳐 Airflow DAG를 갱신한다.

> 참고: 이미 배포된 Pipeline의 Schedule, Enabled, Source Config, Logger Level을 수정하면 `IngestionPipelineRepository.java:827`의 `deployIfRequired()`에서도 자동 Deploy할 수 있다.

## 3. Delete 예시

API: `DELETE /api/v1/services/ingestionPipelines/{id}?hardDelete=true`

### 1) UI에서 삭제 버튼 처리

- 파일: `openmetadata-ui/src/main/resources/ui/src/components/Settings/Services/Ingestion/IngestionListTable/IngestionListTable.tsx:138`
- 함수: `deleteIngestion()`
- 호출: `deleteIngestionPipelineById(id)`

### 2) UI에서 Java API 호출

- 파일: `openmetadata-ui/src/main/resources/ui/src/rest/ingestionPipelineAPI.ts:104`
- 함수: `deleteIngestionPipelineById()`
- 요청: `DELETE /services/ingestionPipelines/{id}?hardDelete=true`

### 3) Java API 진입

- 파일: `openmetadata-service/src/main/java/org/openmetadata/service/resources/services/ingestionpipelines/IngestionPipelineResource.java:871`
- 함수: `delete()`
- 다음 호출: `EntityResource.delete()`

### 4) Java에서 DB 삭제

- 파일: `openmetadata-service/src/main/java/org/openmetadata/service/resources/EntityResource.java:682`
- 함수: `delete()`
- 다음 호출: `EntityRepository.delete()`

- 파일: `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/EntityRepository.java:3963`
- 함수: `delete()`
- 다음 호출: `IngestionPipelineRepository.postDelete()`

### 5) Java에서 Airflow 삭제 API 호출

- 파일: `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/IngestionPipelineRepository.java:495`
- 함수: `postDelete()`
- 다음 호출: `pipelineServiceClient.deletePipeline()`

- 파일: `openmetadata-service/src/main/java/org/openmetadata/service/clients/pipeline/airflow/AirflowRESTClient.java:355`
- 함수: `deletePipeline()`
- 요청: `DELETE /pluginsv2/api/v2/openmetadata/delete?dag_id={pipelineName}` (Airflow 3.x 기준)

### 6) Airflow API 진입 및 DAG 삭제

- 파일: `openmetadata-airflow-apis/openmetadata_managed_apis/api/routes/delete.py:54`
- 함수: `delete_dag()`
- 다음 호출: `delete_dag_id()`

- 파일: `openmetadata-airflow-apis/openmetadata_managed_apis/operations/delete.py:27`
- 함수: `delete_dag_id()`
- 처리: DAG 파일, 설정 JSON, Airflow DB 정보 삭제

Delete에는 후속 Deploy 호출이 없다.

## 4. Airflow가 실제 Ingestion을 실행하는 위치

Deploy는 DAG 등록까지만 수행한다. 등록된 Metadata Ingestion DAG가 스케줄 또는 Trigger로 실행될 때 다음 함수가 호출된다.

### 1) Airflow PythonOperator 실행 함수

- 파일: `openmetadata-airflow-apis/openmetadata_managed_apis/workflows/ingestion/common.py:208`
- 함수: `metadata_ingestion_workflow()`
- 다음 호출: `MetadataWorkflow.create()` → `workflow.execute()`

### 2) Ingestion 실행 함수

- 파일: `ingestion/src/metadata/workflow/base.py:269`
- 함수: `execute()`
- 처리: Source 데이터를 읽어 Sink를 통해 Java API에 저장

## 5. Ingestion이 Java API를 호출하는 위치

실행 상태 저장을 예시로 잡는다.

### 1) Ingestion 상태 저장 시작

- 파일: `ingestion/src/metadata/workflow/workflow_status_mixin.py:103`
- 함수: `set_ingestion_pipeline_status()`
- 다음 호출: `create_or_update_pipeline_status()`

### 2) Java API 경로 생성

- 파일: `ingestion/src/metadata/ingestion/ometa/mixins/ingestion_pipeline_mixin.py:40`
- 함수: `create_or_update_pipeline_status()`
- 요청: `PUT /api/v1/services/ingestionPipelines/{fqn}/pipelineStatus`

### 3) Python 공통 HTTP 호출

- 파일: `ingestion/src/metadata/ingestion/ometa/client.py:177`
- 함수: `REST._request()`

### 4) Java API 진입

- 파일: `openmetadata-service/src/main/java/org/openmetadata/service/resources/services/ingestionpipelines/IngestionPipelineResource.java:1153`
- 함수: `addPipelineStatus()`

## 6. UI가 Ingestion 결과를 조회하는 위치

Ingestion이 UI를 직접 호출하지 않는다. UI가 Java에 저장된 상태를 조회한다.

### 1) UI 조회 함수

- 파일: `openmetadata-ui/src/main/resources/ui/src/components/Settings/Services/Ingestion/IngestionRecentRun/IngestionRecentRuns.component.tsx:49`
- 함수: `fetchPipelineStatus()`

### 2) Java API 호출 함수

- 파일: `openmetadata-ui/src/main/resources/ui/src/rest/ingestionPipelineAPI.ts:155`
- 함수: `getRunHistoryForPipeline()`
- 요청: `GET /api/v1/services/ingestionPipelines/{fqn}/pipelineStatus`
