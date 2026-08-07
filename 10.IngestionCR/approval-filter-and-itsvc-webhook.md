# Ingestion 결재 게이트와 ITSVC Webhook 사전 조사

조사 기준:

- OpenMetadata `1.13.0-release` (`f329dd4a`)
- Atlassian 공식 문서 확인일: 2026-08-07
- OpenMetadata 소스는 변경하지 않았다. 아래 필터 코드는 구현안이 아니라 로그 검증용 제안이다.
- ITSVC의 Jira/Confluence 제품명, 버전, Cloud/Data Center 여부는 확인되지 않았다.

---

## 1. 결론

- 요청이 화이트리스트에 포함되는지 로그로 검증하는 것은 JAX-RS `ContainerRequestFilter`로 가능하다.
- 화이트리스트 키는 실제 URL 정규식보다 `HTTP method + JAX-RS path template`이 적합하다.
- 필터는 결재 절차 전체를 실행하는 곳이 아니라, 대상 요청과 승인된 내부 재실행을 구분하는 **gate**로만 두는 편이 안전하다.
- Jira/Jira Service Management는 외부 시스템으로 HTTP POST를 보내는 webhook을 지원한다.
- 결재 원장이 Jira issue/workflow라면 Jira/JSM webhook을 사용해야 한다. Confluence webhook은 페이지 등 Confluence 이벤트용이다.

---

## 2. 로그 검증용 Java Request Filter

### 2.1 현재 소스의 근거

| 목적 | 실제 파일/클래스/함수 |
|---|---|
| Request Filter 예시 | `openmetadata-service/src/main/java/org/openmetadata/service/resources/filters/ETagRequestFilter.java:28` — `ETagRequestFilter implements ContainerRequestFilter` |
| Filter 실행 함수 | 같은 파일 `:33` — `filter(ContainerRequestContext)` |
| Jersey 명시 등록 | `openmetadata-service/src/main/java/org/openmetadata/service/OpenMetadataApplication.java:375` — `environment.jersey().register(ETagRequestFilter.class)` |
| HTTP method/path 읽기 | `openmetadata-service/src/main/java/org/openmetadata/service/monitoring/MetricsRequestFilter.java:29` — `filter()` |
| JAX-RS path template 추출 예시 | 같은 파일 `:51` — `extractPathTemplate(UriInfo)` |
| 인증 필터 우선순위 | `openmetadata-service/src/main/java/org/openmetadata/service/security/DelegatingContainerRequestFilter.java:11` — `@Priority(Priorities.AUTHENTICATION)` |

`conf/openmetadata.yaml:26`의 Jersey `rootPath`는 `/api/*`다. `UriInfo`에서 비교할 resource path는 `/api`를 제외한 `/v1/...` 형식으로 정규화한다.

### 2.2 제안 위치

새 클래스 위치 예시:

```text
openmetadata-service/src/main/java/org/openmetadata/service/resources/filters/IngestionApprovalRequestFilter.java
```

향후 승인된 사용자 정보까지 확인할 때는 인증 필터 다음에 실행되도록 우선순위를 명시한다.

```java
@Priority(Priorities.AUTHORIZATION)
```

등록 위치 예시:

```java
environment.jersey().register(IngestionApprovalRequestFilter.class);
```

등록만 하면 `filter()`는 JAX-RS Resource의 `@POST`, `@PATCH`, `@DELETE` 함수보다 먼저 실행된다.

### 2.3 판정 방법

`MetricsRequestFilter.extractPathTemplate()`과 동일하게 `ExtendedUriInfo.getMatchedTemplates()`를 사용해 실제 UUID가 아닌 resource template을 얻는다.

```java
String method = requestContext.getMethod();
String pathTemplate = extractPathTemplate(requestContext.getUriInfo());
String requestKey = method + " " + normalize(pathTemplate);

boolean matched = APPROVAL_TARGETS.contains(requestKey);
LOG.info(
    "INGESTION_APPROVAL_FILTER matched={} method={} pathTemplate={}",
    matched,
    method,
    pathTemplate);
```

로그 검증 단계에서는 다음 동작을 하지 않는다.

- `requestContext.abortWith(...)` 호출
- ITSVC API 호출
- 요청 body 읽기 또는 저장
- Authorization header, 비밀번호, 요청 payload 로깅

### 2.4 직접 Ingestion API 화이트리스트

```text
POST   /v1/services/ingestionPipelines
PATCH  /v1/services/ingestionPipelines/{id}
POST   /v1/services/ingestionPipelines/deploy/{id}
POST   /v1/services/ingestionPipelines/trigger/{id}
POST   /v1/services/ingestionPipelines/toggleIngestion/{id}
POST   /v1/services/ingestionPipelines/kill/{id}
DELETE /v1/services/ingestionPipelines/{id}
```

- query string은 path template에 포함되지 않는다.
- Delete의 `hardDelete`, `recursive`를 구분해야 하면 `requestContext.getUriInfo().getQueryParameters()`를 별도로 검사한다.
- 현재 UI의 Bulk Re-deploy는 Java의 `/bulk/deploy` API를 사용하지 않고 선택한 Pipeline별 Deploy API를 반복 호출한다.

### 2.5 간접 유발 API 화이트리스트

이전 조사 문서 `ingestion-cud-api-flow.md` 2장의 UI 유발 API 중 아래 요청도 조건에 따라 Airflow/Ingestion 변경 작업으로 이어진다.

| 구분 | method + path template |
|---|---|
| AutoPilot | `POST /v1/apps/trigger/AutoPilotApplication` |
| Test Connection | `POST /v1/automations/workflows`, `POST /v1/automations/workflows/trigger/{id}` |
| External App | `POST /v1/apps`, `PATCH /v1/apps/{id}`, `POST /v1/apps/deploy/{name}`, `POST /v1/apps/trigger/{name}`, `POST /v1/apps/stop/{name}`, `DELETE /v1/apps/name/{name}` |
| Data Contract | `POST /v1/dataContracts`, `PATCH /v1/dataContracts/{id}`, `POST/PUT /v1/dataContracts/odcs/yaml`, `POST /v1/dataContracts/{id}/validate`, `POST /v1/dataContracts/entity/validate`, `DELETE /v1/dataContracts/{id}` |
| 상위 엔티티 재귀 삭제 | `DELETE /v1/services/{serviceCategory}/async/{id}`, `DELETE /v1/databases/async/{id}`, `DELETE /v1/databaseSchemas/async/{id}`, `DELETE /v1/tables/async/{id}`, `DELETE /v1/dataQuality/testSuites/{id}` |

간접 API는 요청마다 반드시 Airflow를 호출하는 것은 아니다. 엔티티 종류와 연결된 Pipeline 존재 여부에 따라 호출되므로, 필터 로그에서는 **Airflow 호출 가능 요청**으로 표시해야 한다.

`{serviceCategory}`는 설명용 표기이며 실제 JAX-RS path parameter가 아니다. 화이트리스트에는 `databaseServices`, `dashboardServices` 등 UI에서 삭제 가능한 실제 Service Resource 경로를 각각 넣거나, 이 Service 경로 묶음에만 제한된 pattern rule을 둬야 한다.

### 2.6 확인 로그 예시

```text
INGESTION_APPROVAL_FILTER matched=true method=POST pathTemplate=/v1/services/ingestionPipelines/deploy/{id}
INGESTION_APPROVAL_FILTER matched=false method=GET pathTemplate=/v1/services/ingestionPipelines/{id}
```

최소 확인 항목:

1. Add & Deploy에서 Create와 Deploy가 각각 `matched=true`인지 확인
2. Edit에서 PATCH와 Deploy가 각각 `matched=true`인지 확인
3. Run/Pause/Kill/Delete가 각각 `matched=true`인지 확인
4. 목록/상세/상태/로그 GET 요청은 `matched=false`인지 확인
5. Data Quality, AutoPilot 등 간접 API가 해당 규칙으로 판정되는지 확인

---

## 3. 실제 결재 연결 흐름

사용자는 결재 요청 등록 응답을 받은 뒤 UI를 닫아도 된다. 승인 후 Java가 저장된 요청으로 기존 Java API를 다시 호출한다.

아래 API 이름은 현재 소스에 존재하는 것이 아니라 흐름 설명을 위한 예시다.

```text
UI
  → 결재 요청 API
  → 요청자, method, path, query, body, 대상 entity를 Java DB에 PENDING으로 저장
  → ITSVC CR 생성
  ← 202 Accepted + approvalRequestId

ITSVC 승인
  → Java webhook 수신 API
  → DB 상태를 APPROVED로 변경
  → Java Executor가 저장된 요청 조회
  → 승인 실행 ID를 포함해 기존 Java C/U/D API 호출
  → Request Filter가 승인 실행 ID와 원 요청을 검증
  → 기존 JAX-RS Resource 실행
  → DB 상태를 SUCCEEDED 또는 FAILED로 변경
```

### 3.1 Filter 역할

1. 현재 요청이 결재 대상인지 판정
2. 일반 사용자의 직접 C/U/D 요청은 승인 흐름 없이 실행되지 않도록 차단
3. Java Executor의 재호출은 승인 실행 ID를 DB에서 검증한 뒤 기존 API로 통과

승인 실행 ID는 단순 내부용 header 값만 확인하면 안 된다. DB의 승인 상태와 요청자, method, path, entity가 저장된 원 요청과 일치해야 한다.

### 3.2 결재 요청 저장은 별도 API 권장

최초 C/U/D 요청을 Request Filter가 그대로 저장하고 종료할 수도 있지만, Filter 시점에는 기존 Resource 함수의 권한 검사와 request body 역직렬화가 아직 실행되지 않았다.

따라서 UI가 별도 결재 요청 API를 호출하고, 그 API에서 다음 작업을 수행하는 편이 명확하다.

1. 요청자 권한 확인
2. 실행 파라미터 검증 및 DB 저장
3. ITSVC CR 생성
4. `202 Accepted` 반환

Filter는 기존 C/U/D API를 우회 호출하지 못하게 보호하고, 승인 후 Executor 재호출만 통과시킨다.

### 3.3 최소 안전장치

- 상태 변경은 `PENDING → APPROVED/REJECTED → EXECUTING → SUCCEEDED/FAILED`로 제한한다.
- `APPROVED → EXECUTING` 변경은 원자적으로 한 번만 성공하게 해 중복 webhook과 중복 실행을 막는다.
- Java Executor의 내부 재호출에도 같은 Filter가 적용되므로, 검증된 일회성 승인 실행 ID로 재결재 루프를 방지한다.
- Authorization header나 사용자 token은 DB에 저장하지 않는다. Executor는 별도 Service/Bot 자격증명을 사용하고 원 요청자는 감사 정보로 남긴다.
- Ingestion/Test Connection body에는 비밀값이 포함될 수 있으므로 필요한 값만 저장하고 민감값은 암호화해야 한다.
- 저장 시점의 entity version 또는 `If-Match`를 보존해 승인 대기 중 대상이 변경되면 기존 API가 충돌로 실패하도록 한다.

Webhook은 승인 완료를 알리는 trigger로 사용한다. ITSVC 상태 조회 API가 제공된다면 실행 직전에 승인 상태를 다시 확인할 수 있다.

---

## 4. Jira/JSM/Confluence Webhook 지원 여부

### 4.1 Jira Data Center

지원한다.

- `jira:issue_updated`를 외부 URL로 HTTP POST할 수 있다.
- JQL로 대상 Project/Issue 범위를 제한할 수 있다.
- `issue_updated` payload의 `changelog.items`로 변경된 필드를 확인할 수 있다.
- 특정 승인/반려 workflow transition에 webhook post function을 연결할 수도 있다.
- 관리 UI 또는 REST API로 webhook을 등록할 수 있다.

따라서 결재가 Jira issue status 또는 workflow transition으로 관리된다면, OM 수신 API에서 `issue key`, 변경 전·후 status, CR과 OM 작업의 식별자를 확인할 수 있다.

공식 문서: [Jira Data Center Webhooks](https://developer.atlassian.com/server/jira/platform/webhooks/)

### 4.2 Jira Service Management

지원한다.

- Automation rule의 WHEN/IF 조건 뒤에 webhook THEN action을 둘 수 있다.
- Jira issue payload 또는 custom JSON payload를 외부 시스템으로 전송할 수 있다.
- 승인/반려 조건을 Automation rule에서 구분하면 필요한 상태 변경만 보낼 수 있다.

Data Center 공식 문서에는 webhook 실패 시 재시도하지 않는다고 명시되어 있다. 수신 API는 빠르게 2xx를 반환하고, 중복 수신에 안전하게 처리해야 한다.

공식 문서: [Jira Service Management Webhooks](https://developer.atlassian.com/server/jira/platform/jira-service-desk-webhooks/)

Jira Service Management Cloud는 승인 상태 조회 API도 제공한다.

```text
GET /rest/servicedeskapi/request/{issueIdOrKey}/approval
GET /rest/servicedeskapi/request/{issueIdOrKey}/approval/{approvalId}
```

공식 문서: [JSM Cloud Request/Approval REST API](https://developer.atlassian.com/cloud/jira/service-desk/rest/api-group-request/)

### 4.3 Jira Cloud

지원한다.

- `jira:issue_updated` webhook과 JQL 필터를 지원한다.
- Cloud app의 dynamic webhook 등록 방식은 Data Center 관리 API와 다르다.

공식 문서: [Jira Cloud Webhooks](https://developer.atlassian.com/cloud/jira/platform/webhooks/)

### 4.4 Confluence

Confluence Cloud와 Data Center도 webhook을 지원하지만, 기본 대상은 page 생성·수정·삭제 등 Confluence 이벤트다.

```text
POST <CONFLUENCE_URL>/rest/api/webhooks   # Data Center 등록 API
```

결재 상태가 Jira issue/workflow에 저장된다면 Confluence webhook을 사용할 이유는 확인되지 않는다. 결재 원장이 Confluence page나 별도 plugin이라면 그 plugin이 발생시키는 이벤트를 추가로 확인해야 한다.

공식 문서:

- [Confluence Data Center Webhooks](https://developer.atlassian.com/server/confluence/webhooks/)
- [Confluence Cloud Webhooks](https://developer.atlassian.com/cloud/confluence/using-webhooks/)

---

## 5. ITSVC 담당자에게 확인할 항목

공식 Jira/JSM에는 필요한 기능이 있지만, 사내 ITSVC에서 동일하게 개방됐는지는 확인되지 않았다.

1. 실제 제품과 버전: Jira/JSM Cloud 또는 Data Center
2. 결재 완료/반려가 저장되는 issue field와 workflow status
3. `jira:issue_updated` 또는 Automation webhook 사용 권한
4. Project/issue를 제한할 JQL
5. 폐쇄망에서 ITSVC → OM webhook URL로 접근 가능한지
6. webhook 인증 방식과 서명 검증 방식
7. 실패 재시도 정책과 이벤트 고유 ID 제공 여부
8. webhook 후 승인 상태를 REST API로 재검증할 수 있는지

ITSVC 자체의 webhook 제공 여부는 사내 제품 설정을 확인하기 전에는 확정할 수 없다.
