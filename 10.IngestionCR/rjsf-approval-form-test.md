# RJSF 결재 폼 임시 페이지 테스트

## 질문

> UI에 임시 페이지를 만들고 RJSF로 결재 폼을 테스트하려면 어떻게 해야 하는가?

## 답변

OpenMetadata `1.13.0-release`에는 RJSF가 이미 포함되어 있다. 새 패키지나 Java API 없이 아래 3개 UI 파일만 추가·수정하면 폼 렌더링, 필수값 검증, submit 결과 확인까지 테스트할 수 있다.

이 문서는 구현 절차만 제안한다. 현재 OpenMetadata 소스는 수정하지 않았으며, 아래 `ApprovalFormTestPage`와 `/approval-form-test`는 현재 코드에 없는 테스트용 이름이다.

### 현재 소스 근거

| 확인 항목 | 실제 파일/함수 |
|---|---|
| RJSF 의존성 | `openmetadata-ui/src/main/resources/ui/package.json:85` — `@rjsf/core`, `@rjsf/utils`, `@rjsf/validator-ajv8` `5.24.13` |
| OM 공통 RJSF wrapper | `openmetadata-ui/src/main/resources/ui/src/components/common/FormBuilder/FormBuilder.tsx:49` — `FormBuilder` |
| RJSF 렌더링/submit | 같은 파일 `:140` — `<Form>`, `:164` — `onSubmit` |
| wrapper 사용 예시 | `openmetadata-ui/src/main/resources/ui/src/components/Settings/Applications/ApplicationConfiguration/ApplicationConfiguration.tsx:62` — `<FormBuilder>` |
| URL 상수 | `openmetadata-ui/src/main/resources/ui/src/constants/constants.ts:130` — `ROUTES` |
| 인증 후 화면 라우팅 | `openmetadata-ui/src/main/resources/ui/src/components/AppRouter/AuthenticatedAppRouter.tsx` — `AuthenticatedAppRouter` |

`FormBuilder`를 재사용하면 OM의 field/error template과 submit/cancel 버튼을 그대로 사용할 수 있다.

## 1. 테스트 페이지 추가

제안 파일:

```text
openmetadata-ui/src/main/resources/ui/src/pages/ApprovalFormTestPage/ApprovalFormTestPage.tsx
```

```tsx
import { IChangeEvent } from '@rjsf/core';
import { RJSFSchema, UiSchema } from '@rjsf/utils';
import validator from '@rjsf/validator-ajv8';
import { Card, Col, Row, Typography } from 'antd';
import { useState } from 'react';
import FormBuilder from '../../components/common/FormBuilder/FormBuilder';
import { ServiceCategory } from '../../enums/service.enum';
import { withPageLayout } from '../../hoc/withPageLayout';

interface ApprovalFormData {
  requestTitle?: string;
  operation?: string;
  approver?: string;
  reason?: string;
}

const approvalSchema: RJSFSchema = {
  type: 'object',
  additionalProperties: false,
  required: ['requestTitle', 'operation', 'approver', 'reason'],
  properties: {
    requestTitle: { type: 'string', title: 'CR 제목', minLength: 1 },
    operation: {
      type: 'string',
      title: '작업',
      enum: ['CREATE', 'UPDATE', 'DEPLOY', 'DELETE'],
      default: 'CREATE',
    },
    approver: { type: 'string', title: '결재 승인자', minLength: 1 },
    reason: { type: 'string', title: '요청 사유', minLength: 1 },
  },
};

const approvalUiSchema: UiSchema = {
  reason: {
    'ui:widget': 'textarea',
    'ui:options': { rows: 4 },
  },
};

const ApprovalFormTestPage = () => {
  const [submittedData, setSubmittedData] = useState<ApprovalFormData>();

  const handleSubmit = ({ formData }: IChangeEvent) => {
    setSubmittedData(formData as ApprovalFormData);
  };

  return (
    <Row className="p-lg" gutter={[16, 16]}>
      <Col span={12}>
        <Card title="RJSF 결재 폼 테스트">
          <FormBuilder
            cancelText="초기화"
            okText="요청 내용 확인"
            schema={approvalSchema}
            serviceCategory={ServiceCategory.DATABASE_SERVICES}
            uiSchema={approvalUiSchema}
            validator={validator}
            onCancel={() => setSubmittedData(undefined)}
            onSubmit={handleSubmit}
          />
        </Card>
      </Col>

      <Col span={12}>
        <Card title="Submit 결과">
          {submittedData ? (
            <pre data-testid="submitted-data">
              {JSON.stringify(submittedData, null, 2)}
            </pre>
          ) : (
            <Typography.Text type="secondary">
              폼을 제출하면 데이터가 여기에 표시됩니다.
            </Typography.Text>
          )}
        </Card>
      </Col>
    </Row>
  );
};

export default withPageLayout(ApprovalFormTestPage);
```

- `approvalSchema`: 입력 필드, 필수값, enum을 정의한다.
- `approvalUiSchema`: `reason`만 textarea로 표시한다.
- `handleSubmit()`: 제출 데이터를 오른쪽 Card에 표시할 뿐 API를 호출하지 않는다.
- `serviceCategory`: `FormBuilder.Props`의 필수값을 충족하기 위한 값이다. 이 테스트에서 Database Service API를 호출한다는 의미는 아니다.

## 2. 임시 URL 상수 추가

파일:

```text
openmetadata-ui/src/main/resources/ui/src/constants/constants.ts
```

`ROUTES`에 다음 항목을 추가한다.

```ts
APPROVAL_FORM_TEST: '/approval-form-test',
```

## 3. 인증 라우터에 페이지 등록

파일:

```text
openmetadata-ui/src/main/resources/ui/src/components/AppRouter/AuthenticatedAppRouter.tsx
```

다른 page lazy loading 선언과 같은 위치에 추가한다.

```tsx
const ApprovalFormTestPage = withSuspenseFallback(
  React.lazy(
    () => import('../../pages/ApprovalFormTestPage/ApprovalFormTestPage')
  )
);
```

`AuthenticatedAppRouter`의 `<Routes>` 안에서 마지막 wildcard route보다 앞에 추가한다.

```tsx
<Route
  element={<ApprovalFormTestPage />}
  path={ROUTES.APPROVAL_FORM_TEST}
/>
```

이 경로는 로그인 후 라우터에만 등록된다. 현재 단계에는 Java API가 없으므로 별도 Ingestion 권한 검사는 넣지 않는다. 실제 결재 요청 기능으로 전환할 때는 UI route 보호와 별개로 Java API에서 권한을 다시 검사해야 한다.

## 4. 확인 방법

소스의 UI 실행 안내는 다음 명령을 사용한다.

```shell
make yarn_start_dev_ui
```

접속 URL:

```text
http://localhost:3000/approval-form-test
```

확인 항목:

1. 빈 상태로 submit하면 네 필드에 필수값 오류가 표시되는가
2. `operation`이 네 항목의 select로 표시되는가
3. 값을 채우고 submit하면 오른쪽에 같은 JSON이 표시되는가
4. 초기화 후 입력값과 submit 결과가 사라지는가
5. Network 탭에서 submit으로 발생한 Java/ITSVC API 요청이 없는가

## 테스트 범위 이후

기본 RJSF 동작이 확인된 뒤에만 승인자 검색 widget과 결재 요청 API를 연결한다. 현재 테스트의 `approver`는 단순 문자열이며, 실제 ITSVC 사용자 조회 방식은 확인되지 않았으므로 제안하지 않는다.
