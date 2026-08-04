# Ingestion Pipeline C/U/D API 호출 흐름

## 1. 조사 목적

Ingestion Pipeline 생성·수정·삭제 작업에 결재 절차를 추가하기 전에, 현재 UI·OpenMetadata Java Server·Airflow·Python Ingestion 사이의 실제 호출 흐름과 주요 진입점을 확인한다.

- 조사 기준 OpenMetadata 커밋: `f329dd4a7e47134a2bd5a06af6181b0ee527ddd9`
- 대상 API: `/api/v1/services/ingestionPipelines`
- 이 문서에서 Java는 OpenMetadata Server를 의미한다.
- C/U/D는 Create, Update, Delete를 의미한다.

## 2. 핵심 결론

`/api/v1/services/ingestionPipelines`는 Ingestion이나 Airflow API가 아니라 **OpenMetadata Java API**다.

생성·수정 API는 우선 OpenMetadata DB의 Ingestion Pipeline 엔티티를 저장한다. 저장이 끝나면 UI가 별도의 Deploy API를 호출하고, Java가 Airflow의 OpenMetadata Plugin API를 호출하여 DAG를 생성하거나 갱신한다.

```text
생성/수정
UI → Java에 C/U 저장 → UI 응답
   → UI가 Deploy 호출 → Java → Airflow에 DAG 생성/갱신 → UI 응답

삭제
UI → Java에서 삭제 → Java → Airflow의 DAG 삭제 → UI 응답

실제 Ingestion 실행
Airflow 스케줄 또는 Trigger → Python Ingestion 실행
→ Ingestion이 Java API에 데이터와 실행 상태 저장
→ UI가 Java API를 조회하여 상태 표시
```

Deploy는 Ingestion 작업을 즉시 실행하는 API가 아니다. **Airflow에서 실행할 DAG를 등록하거나 갱신하는 API**다. 실제 Ingestion은 등록된 DAG가 스케줄되거나 Trigger됐을 때 실행된다.

## 3. Create 호출 흐름

예시 화면은 일반 Ingestion Pipeline 생성 페이지다.

### 3.1 UI에서 생성 요청

파일:

```text
openmetadata-ui/src/main/resources/ui/src/pages/
  AddIngestionPage/AddIngestionPage.component.tsx
```

함수:

```text
onAddIngestionSave()
→ addIngestionPipeline(data)
```

REST 함수:

```text
파일: openmetadata-ui/src/main/resources/ui/src/rest/ingestionPipelineAPI.ts
함수: addIngestionPipeline()
요청: POST /api/v1/services/ingestionPipelines
```

### 3.2 Java에서 엔티티 생성

```text
파일: openmetadata-service/src/main/java/org/openmetadata/service/resources/
      services/ingestionpipelines/IngestionPipelineResource.java

create(CreateIngestionPipeline create)
→ create(uriInfo, securityContext, ingestionPipeline)
→ repository.create()
```

공통 DB 저장 함수:

```text
파일: openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/
      EntityRepository.java
함수: create()
```

Java는 생성된 Ingestion Pipeline과 `id`를 UI에 반환한다. 이 시점에는 OpenMetadata DB에 엔티티가 생성됐지만 Airflow DAG는 아직 생성되지 않았다.

### 3.3 UI에서 Deploy 요청

UI는 생성 응답으로 받은 `id`를 이용하여 Deploy API를 호출한다.

```text
파일: AddIngestionPage.component.tsx
함수: onIngestionDeploy(id)

파일: openmetadata-ui/src/main/resources/ui/src/rest/ingestionPipelineAPI.ts
함수: deployIngestionPipelineById()
요청: POST /api/v1/services/ingestionPipelines/deploy/{id}
```

이후 6장의 공통 Deploy 흐름을 탄다.

## 4. Update 호출 흐름

예시 화면은 일반 Ingestion Pipeline 수정 페이지다.

### 4.1 UI에서 수정 요청

```text
파일: openmetadata-ui/src/main/resources/ui/src/pages/
      EditIngestionPage/EditIngestionPage.component.tsx
함수: onEditIngestionSave()

onEditIngestionSave()
→ compare(기존 데이터, 변경 데이터)
→ updateIngestionPipeline(id, jsonPatch)
```

REST 함수:

```text
파일: openmetadata-ui/src/main/resources/ui/src/rest/ingestionPipelineAPI.ts
함수: updateIngestionPipeline()
요청: PATCH /api/v1/services/ingestionPipelines/{id}
본문: JSON Patch 연산 배열
```

### 4.2 Java에서 수정 처리

```text
파일: openmetadata-service/src/main/java/org/openmetadata/service/resources/
      services/ingestionpipelines/IngestionPipelineResource.java
함수: updateDescription()

updateDescription()
→ EntityResource.patchInternal()
→ EntityRepository.patch()
→ IngestionPipelineUpdater.entitySpecificUpdate()
```

관련 파일:

```text
openmetadata-service/src/main/java/org/openmetadata/service/resources/
  EntityResource.java

openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/
  EntityRepository.java
  IngestionPipelineRepository.java
```

### 4.3 수정 후 Deploy 요청

수정 API 응답을 받은 UI는 다시 Deploy API를 호출한다.

```text
파일: EditIngestionPage.component.tsx
함수: onIngestionDeploy()
요청: POST /api/v1/services/ingestionPipelines/deploy/{id}
```

### 4.4 자동 재배포 주의사항

현재 Java에는 이미 배포된 Pipeline의 일부 설정이 변경되면 PATCH 내부에서 자동 재배포하는 코드가 있다.

```text
파일: openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/
      IngestionPipelineRepository.java

IngestionPipelineUpdater.entitySpecificUpdate()
→ deployIfRequired()
→ requiresRedeployment()
→ deployPipelineBeforeUpdate()
→ pipelineServiceClient.deployPipeline()
```

자동 재배포 판단 대상에는 Schedule, Enabled, Source Config, Logger Level 변경이 포함된다. 조건을 만족하면 다음 두 배포가 모두 발생할 가능성이 있다.

```text
PATCH 내부 자동 Deploy
+
PATCH 응답 후 UI가 호출하는 명시적 Deploy
```

결재 기능을 설계할 때 수정 후 Deploy 책임을 Java 자동 재배포와 승인 실행기 중 한 곳으로 정할 필요가 있다.

## 5. Delete 호출 흐름

### 5.1 UI에서 삭제 요청

```text
파일: openmetadata-ui/src/main/resources/ui/src/components/Settings/Services/
      Ingestion/IngestionListTable/IngestionListTable.tsx
함수: deleteIngestion()

파일: openmetadata-ui/src/main/resources/ui/src/rest/ingestionPipelineAPI.ts
함수: deleteIngestionPipelineById()
요청: DELETE /api/v1/services/ingestionPipelines/{id}?hardDelete=true
```

### 5.2 Java에서 엔티티와 Airflow DAG 삭제

```text
파일: openmetadata-service/src/main/java/org/openmetadata/service/resources/
      services/ingestionpipelines/IngestionPipelineResource.java
함수: delete()

IngestionPipelineResource.delete()
→ EntityResource.delete()
→ EntityRepository.delete()
→ IngestionPipelineRepository.postDelete()
→ pipelineServiceClient.deletePipeline()
→ AirflowRESTClient.deletePipeline()
```

Airflow 삭제 호출이 시작되는 Ingestion 전용 후처리 함수:

```text
파일: openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/
      IngestionPipelineRepository.java
함수: postDelete()
```

Java Airflow Client:

```text
파일: openmetadata-service/src/main/java/org/openmetadata/service/clients/
      pipeline/airflow/AirflowRESTClient.java
함수: deletePipeline()
요청: DELETE {Airflow OM API Prefix}/delete?dag_id={pipelineName}
```

Airflow API 진입점과 실제 삭제 함수:

```text
파일: openmetadata-airflow-apis/openmetadata_managed_apis/api/routes/delete.py
함수: delete_dag()

파일: openmetadata-airflow-apis/openmetadata_managed_apis/operations/delete.py
함수: delete_dag_id()
```

`delete_dag_id()`는 다음 항목을 삭제한다.

- Airflow DAG Python 파일
- Ingestion 설정 JSON 파일
- Airflow DB의 `DagModel`, `DagRun`

삭제 후에는 별도의 Deploy나 Ingestion 실행이 없다.

## 6. 공통 Deploy 흐름

### 6.1 Java Deploy API

```text
파일: openmetadata-service/src/main/java/org/openmetadata/service/resources/
      services/ingestionpipelines/IngestionPipelineResource.java

deployIngestion()
→ deployPipelineInternal()
→ pipelineServiceClient.deployPipeline()
```

실제 Airflow HTTP 호출:

```text
파일: openmetadata-service/src/main/java/org/openmetadata/service/clients/
      pipeline/airflow/AirflowRESTClient.java
함수: deployPipeline()
요청: POST {Airflow OM API Prefix}/deploy
```

Java의 `pipelineServiceClient`는 설정으로 구현체를 생성한다. Airflow를 사용하는 환경에서는 최종적으로 `AirflowRESTClient`가 HTTP 요청을 전송한다. 중간의 `MeteredPipelineServiceClient`는 호출 지표를 기록한 뒤 실제 Client에 위임한다.

### 6.2 Airflow Deploy API

Airflow Plugin의 공통 Blueprint 등록:

```text
파일: openmetadata-airflow-apis/openmetadata_managed_apis/api/app.py
함수: get_blueprint()
```

Deploy API 진입점:

```text
파일: openmetadata-airflow-apis/openmetadata_managed_apis/api/routes/deploy.py
함수: get_fn() 내부 deploy_dag()
```

실제 DAG 생성:

```text
파일: openmetadata-airflow-apis/openmetadata_managed_apis/operations/deploy.py
함수: DagDeployer.deploy()

deploy_dag()
→ Ingestion Pipeline 요청 파싱
→ DagDeployer.deploy()
→ Ingestion 설정 JSON 저장
→ DAG Python 파일 생성
→ Airflow에 DAG 등록
→ Java에 처리 결과 반환
```

Airflow API Prefix는 감지된 버전에 따라 다음 중 하나를 사용한다.

```text
Airflow 3.x: /pluginsv2/api/v2/openmetadata
기타:         /api/v2/openmetadata 또는 /api/v1/openmetadata
```

## 7. Airflow에서 실제 Ingestion을 실행하는 흐름

Deploy가 완료되면 Airflow에 DAG가 등록된다. 이후 스케줄 시간이 되거나 Trigger API가 호출되면 DAG의 PythonOperator가 Python Ingestion 함수를 실행한다.

Metadata Pipeline을 예로 들면 다음 순서다.

```text
Airflow가 생성된 DAG 로드
→ WorkflowFactory.create()/generate_dag()
→ WorkflowBuilder.build()
→ build_metadata_dag()
→ build_dag()
→ CustomPythonOperator(python_callable=metadata_ingestion_workflow)

스케줄 또는 Trigger
→ metadata_ingestion_workflow()
→ MetadataWorkflow.create()
→ workflow.execute()
→ Source → Sink 처리
```

주요 파일과 함수:

```text
openmetadata-airflow-apis/openmetadata_managed_apis/resources/dag_runner.j2

openmetadata-airflow-apis/openmetadata_managed_apis/workflows/
  workflow_factory.py              WorkflowFactory
  workflow_builder.py              WorkflowBuilder.build()
  ingestion/metadata.py            build_metadata_dag()
  ingestion/common.py              build_dag(), metadata_ingestion_workflow()

ingestion/src/metadata/workflow/
  ingestion.py                     IngestionWorkflow.create()
  base.py                          BaseWorkflow.execute()
  metadata.py                      MetadataWorkflow
```

Pipeline Type이 Profiler, Usage, TestSuite 등이라면 `WorkflowBuilder`가 각 타입에 등록된 다른 DAG Builder를 선택하지만, Airflow PythonOperator가 Python Ingestion Workflow를 호출하는 기본 구조는 같다.

## 8. Java가 호출하는 Airflow API의 앞단

Java의 업무별 HTTP 호출 시작점은 `AirflowRESTClient`다.

```text
파일: openmetadata-service/src/main/java/org/openmetadata/service/clients/
      pipeline/airflow/AirflowRESTClient.java

Deploy  → deployPipeline()
Delete  → deletePipeline()
Trigger → runPipeline()
```

Airflow의 공통 API 등록 앞단:

```text
파일: openmetadata-airflow-apis/openmetadata_managed_apis/api/app.py
함수: get_blueprint()
```

업무별 HTTP 수신 진입점:

```text
Deploy  → api/routes/deploy.py  → deploy_dag()
Delete  → api/routes/delete.py  → delete_dag()
Trigger → api/routes/trigger.py → trigger_dag()
```

## 9. Airflow/Ingestion이 호출하는 Java API의 앞단

Airflow의 관리 REST API가 작업 완료 후 Java를 직접 호출하는 구조는 아니다. Airflow가 실행한 **Python Ingestion 코드가 OpenMetadata REST Client를 사용하여 Java API를 호출한다.**

### 9.1 Ingestion Pipeline 실행 상태 저장 예시

```text
WorkflowStatusMixin.set_ingestion_pipeline_status()
→ OMetaIngestionPipelineMixin.create_or_update_pipeline_status()
→ REST.put()
→ REST._request()
→ PUT /api/v1/services/ingestionPipelines/{fqn}/pipelineStatus
→ IngestionPipelineResource.addPipelineStatus()
```

Python 상태 처리:

```text
파일: ingestion/src/metadata/workflow/workflow_status_mixin.py
함수: set_ingestion_pipeline_status()
```

Pipeline Status API 구성:

```text
파일: ingestion/src/metadata/ingestion/ometa/mixins/
      ingestion_pipeline_mixin.py
함수: create_or_update_pipeline_status()
```

Python 공통 HTTP 호출 최하단:

```text
파일: ingestion/src/metadata/ingestion/ometa/client.py
함수: REST._request()
```

Java 수신 함수:

```text
파일: openmetadata-service/src/main/java/org/openmetadata/service/resources/
      services/ingestionpipelines/IngestionPipelineResource.java
함수: addPipelineStatus()
```

실제 수집된 Table, Topic 등의 데이터 저장 API는 엔티티 종류에 따라 Java Resource가 달라진다. 모든 Ingestion 결과가 하나의 공통 Java Controller로 들어가는 구조는 아니다.

### 9.2 UI의 실행 상태 조회

Ingestion이 UI를 직접 호출하지 않는다. UI가 Java에 저장된 상태를 조회한다.

```text
GET /api/v1/services/ingestionPipelines/{fqn}/pipelineStatus
```

```text
파일: openmetadata-ui/src/main/resources/ui/src/rest/ingestionPipelineAPI.ts
함수: getRunHistoryForPipeline()

파일: openmetadata-ui/src/main/resources/ui/src/components/Settings/Services/
      Ingestion/IngestionRecentRun/IngestionRecentRuns.component.tsx
함수: fetchPipelineStatus()
```

최종 관계는 다음과 같다.

```text
Ingestion → Java에 실행 상태 저장
UI → Java에서 실행 상태 조회
```

## 10. 결재 시스템 설계 시 확인할 사항

현재 코드 조사 결과를 기준으로 다음 사항은 기획 단계에서 결정할 필요가 있다.

1. Create와 Update 승인 실행 후 Deploy까지 자동으로 수행할지 결정한다.
2. Update의 Java 자동 재배포와 승인 실행기의 Deploy가 중복되지 않도록 배포 책임을 한 곳으로 정한다.
3. Delete는 Java 삭제 후처리에서 Airflow DAG도 삭제하므로 별도의 Deploy 후속 작업이 필요 없다.
4. 결재 Filter 대상은 HTTP Method만으로 판단하지 않고 정확한 C/U/D 경로로 제한한다.
5. `/deploy/{id}`는 결재 Filter에 다시 잡혀 승인 요청이 반복되지 않도록 제외하거나 내부 승인 실행 Context를 사용한다.
6. UI는 Ingestion 완료 통지를 직접 받지 않으므로, 결재와 실행 상태 표시도 Java에 저장된 상태를 조회하는 방식을 고려한다.
