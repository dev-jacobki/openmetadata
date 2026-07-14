# Data Quality & Observability 발표 자료 기준

## 발표 범위

이 자료는 OpenMetadata의 **Data Quality & Observability**만 다룬다.

- Profile과 Data Profile
- TestDefinition · TestSuite · TestCase의 연결
- UI · YAML · Python SDK로 정의한 테스트의 실행과 결과 저장
- 실패 결과의 Incident 관리
- Event Subscription 기반 Alert와 Slack · Webhook · Email 알림

따라 해 보기, 프로젝트 데모, 토론, 발표 준비 안내, 별도 읽기 자료는 발표 문서에 넣지 않는다.

## 문서가 따라갈 흐름

전체 관계를 짧게 보여 준 뒤 다음 순서로 자연스럽게 설명한다.

1. **Profile: 데이터 상태 측정** — 메트릭이 무엇을 남기는지와 Sample Data의 위치를 설명한다.
2. **TestDefinition · TestSuite · TestCase: 검사 연결** — 재사용 규칙, 검사 묶음, 자산별 실제 검사가 어떻게 연결되는지 설명한다.
3. **Test 실행: 설정에서 결과까지** — 설정이 Source, Runner, Result, Sink를 거쳐 서버에 저장되는 경로와 전달되는 데이터를 설명한다.
4. **Incident Manager: 실패 이슈 처리** — 반복 실패가 미해결 Incident와 이어지고 상태에 따라 작업이 처리되는 흐름을 설명한다.
5. **Alert & Notification: 이벤트 전달** — 필터를 통과한 이벤트만 정한 채널로 전달된다는 점을 짧고 분명하게 설명한다.

## 작성 원칙

- 표나 사전식 `용어 / 이게 무엇인가 / 흐름에서 하는 일` 구조로 쓰지 않는다. 처음 나오는 용어는 문장 안에서 자연스럽게 뜻을 밝힌다.
- 기능명과 목적이 드러나는 제목을 사용한다. 추상적인 구호나 당연한 문장은 넣지 않는다.
- 처음 보는 사람은 용어를 이해할 수 있어야 하고, 개발자는 실제 소스 코드로 흐름을 확인할 수 있어야 한다.
- 코드는 반드시 이 저장소의 실제 코드를 사용한다. 코드 앞에는 `파일: 경로 (행 범위)`를 표시하고, 코드 블록 안에는 설명 문장을 덧붙이지 않는다.
- 구현 세부사항을 과도하게 파고들기보다, 어떤 코드가 어떤 값 또는 객체를 다음 단계로 넘기는지에 집중한다.
- 각 큰 흐름이 끝나는 곳에는 핵심을 찌르는 질문·답변을 한두 개만 둔다. 기초적인 정의 반복 질문은 피한다.

## 이미지 원칙

- 이미지는 화면 캡처가 아니라 설명을 보조하는 간결한 도식으로 만든다.
- 이미지 안에는 관계를 읽는 데 필요한 단어만 넣는다. 긴 문장, 작은 글씨, 많은 빈 공간, 칸을 넘는 글자는 허용하지 않는다.
- 상세 설명은 본문과 코드 블록에 두고, 이미지는 관계와 순서만 보여 준다.
- 모든 이미지는 이 문서와 같은 폴더에 둔다. 외부 URL과 `assets` 하위 폴더를 사용하지 않는다.

## 현재 파일 구성

- `data-quality-observability-presentation.md` — 발표 본문
- `quality-concept-map.png` — Profile과 테스트 객체 관계
- `test-execution-pipeline.png` — 설정부터 결과 저장까지의 실행 경로
- `failure-notification-flow.png` — 실패, Incident, 구독, 알림 채널의 연결

## 확인할 실제 소스

- `ingestion/src/metadata/workflow/data_quality.py` — Source · Runner · Sink 구성
- `ingestion/src/metadata/workflow/ingestion.py` — 레코드를 단계별로 전달하는 공통 실행 루프
- `ingestion/src/metadata/data_quality/processor/test_case_runner.py` — TestCase별 결과 생성
- `ingestion/src/metadata/ingestion/sink/metadata_rest.py` — TestCaseResult 저장과 실패 행 표본 처리
- `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/TestCaseResolutionStatusRepository.java` — Incident 상태 처리
- `openmetadata-spec/src/main/resources/json/schema/events/eventSubscription.json` — Alert 필터와 목적지 구조

수정 후에는 Markdown 이미지 참조가 모두 실제 파일을 가리키는지, 코드 블록이 열고 닫혔는지 확인한다.
