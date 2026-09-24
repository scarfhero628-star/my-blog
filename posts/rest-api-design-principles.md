---
title: REST API 설계 원칙 — 개발자가 꼭 알아야 할 7가지 베스트 프랙티스
date: 2026-09-24
tags: [REST API, 웹 개발, 백엔드, API 설계]
description: 실무에서 바로 적용할 수 있는 REST API 설계 베스트 프랙티스 7가지를 예제 코드와 함께 정리했습니다.
---

백엔드 개발을 하다 보면 API 설계에서 적지 않은 시간을 쓰게 됩니다. 잘 설계된 API는 팀 생산성을 높이고, 프론트엔드 개발자와의 협업을 원활하게 만들며, 유지보수 비용을 크게 줄여줍니다. 반대로 일관성 없이 만들어진 API는 혼란을 낳고, 시간이 지날수록 고치기 어려운 기술 부채가 됩니다.

이 글에서는 REST API를 설계할 때 반드시 지켜야 할 7가지 원칙을 실전 예제와 함께 정리합니다.

---

## 1. 리소스 중심으로 URL을 설계하라

REST API에서 URL은 **리소스(자원)** 를 나타내야 합니다. 동작(action)이 아닌 명사로 표현하는 것이 핵심입니다.

```
# 나쁜 예 — 동사 사용
GET /getUser/1
POST /createPost
DELETE /deleteComment/5

# 좋은 예 — 명사(리소스) 사용
GET /users/1
POST /posts
DELETE /comments/5
```

컬렉션은 복수형으로, 단일 리소스는 ID를 붙여 표현합니다.

```
GET  /articles         # 전체 목록 조회
GET  /articles/42      # 특정 글 조회
POST /articles         # 새 글 생성
PUT  /articles/42      # 특정 글 전체 수정
PATCH /articles/42     # 특정 글 일부 수정
DELETE /articles/42    # 특정 글 삭제
```

중첩 관계는 부모 리소스 아래에 표현하되, 깊이는 2단계 이하로 유지하는 것이 좋습니다.

```
GET /users/1/posts          # 사용자 1의 게시글 목록
GET /users/1/posts/5        # 사용자 1의 게시글 5번
```

---

## 2. HTTP 메서드를 목적에 맞게 사용하라

HTTP 메서드는 각각의 의미가 있습니다. 모든 요청을 GET이나 POST로 처리하는 것은 REST의 기본을 무시하는 행위입니다.

| 메서드 | 용도 | 멱등성 |
|--------|------|--------|
| `GET` | 조회 | O |
| `POST` | 생성 | X |
| `PUT` | 전체 수정 | O |
| `PATCH` | 일부 수정 | X |
| `DELETE` | 삭제 | O |

**멱등성(idempotency)** 이란 같은 요청을 여러 번 보내도 결과가 동일한 성질입니다. GET, PUT, DELETE는 멱등하지만, POST와 PATCH는 그렇지 않을 수 있습니다.

---

## 3. 적절한 HTTP 상태 코드를 반환하라

응답 상태 코드는 클라이언트에게 요청 처리 결과를 명확하게 전달하는 수단입니다. 모든 상황에 `200 OK`를 반환하거나, 에러를 200으로 감싸서 본문에 담는 방식은 피해야 합니다.

```
# 자주 쓰는 상태 코드
200 OK          # 조회/수정 성공
201 Created     # 생성 성공
204 No Content  # 삭제 성공 (본문 없음)

400 Bad Request      # 잘못된 요청 (입력값 오류)
401 Unauthorized     # 인증 필요
403 Forbidden        # 접근 권한 없음
404 Not Found        # 리소스 없음
409 Conflict         # 충돌 (중복 데이터 등)
422 Unprocessable    # 유효성 검사 실패

500 Internal Server Error  # 서버 오류
```

실전에서 자주 헷갈리는 구분:
- `401`은 "누구인지 모른다(인증 없음)", `403`은 "알고 있지만 허가되지 않았다(권한 없음)"
- `404`는 리소스 자체가 없을 때, `410 Gone`은 과거에 있었지만 영구 삭제됐을 때 사용

---

## 4. 일관된 에러 응답 형식을 정의하라

에러가 발생했을 때 어떤 정보를 어떤 형식으로 내려줄지 미리 정해두어야 합니다. 클라이언트가 에러를 파싱하고 처리하기 쉬운 형태여야 합니다.

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "입력값이 올바르지 않습니다.",
    "details": [
      {
        "field": "email",
        "message": "유효한 이메일 주소를 입력해주세요."
      },
      {
        "field": "password",
        "message": "비밀번호는 8자 이상이어야 합니다."
      }
    ]
  }
}
```

`code`는 기계가 읽는 에러 코드, `message`는 사람이 읽는 설명입니다. `details`로 필드별 상세 오류를 전달하면 프론트엔드에서 폼 유효성 검사에 바로 활용할 수 있습니다.

---

## 5. API 버전 관리를 처음부터 도입하라

API는 한 번 릴리즈하면 쉽게 바꾸기 어렵습니다. 클라이언트가 이미 사용 중인 API를 갑자기 변경하면 서비스 장애로 이어집니다. 처음부터 버전 관리를 도입하면 하위 호환성을 유지하면서 새 기능을 추가할 수 있습니다.

버전 관리 방식은 크게 세 가지입니다.

```
# URL 경로에 버전 포함 (가장 널리 쓰임)
GET /v1/users
GET /v2/users

# 쿼리 파라미터
GET /users?version=1

# 요청 헤더
GET /users
Accept: application/vnd.myapp.v1+json
```

URL 경로 방식이 가장 직관적이고, 브라우저에서도 바로 테스트할 수 있어 실무에서 많이 쓰입니다.

---

## 6. 페이지네이션과 필터링을 표준화하라

데이터가 많아지면 목록 조회 API에 반드시 페이지네이션이 필요합니다. 일관된 형식을 정해두지 않으면 API마다 제각각이 돼버립니다.

```
# 페이지 기반 페이지네이션
GET /articles?page=2&pageSize=20

# 커서 기반 페이지네이션 (대규모 데이터에 적합)
GET /articles?cursor=eyJpZCI6MTAwfQ==&limit=20

# 필터링 및 정렬
GET /articles?status=published&author=1&sort=createdAt&order=desc
```

응답에도 메타데이터를 포함하면 클라이언트가 다음 페이지가 있는지 알 수 있습니다.

```json
{
  "data": [...],
  "pagination": {
    "total": 245,
    "page": 2,
    "pageSize": 20,
    "totalPages": 13
  }
}
```

---

## 7. API 문서화를 코드와 함께 관리하라

아무리 잘 설계된 API도 문서가 없으면 쓰기 어렵습니다. 더 큰 문제는 코드와 문서가 분리되면 시간이 지날수록 내용이 어긋나게 된다는 것입니다.

**OpenAPI(Swagger)** 스펙을 코드에 직접 작성하거나, NestJS의 `@ApiProperty()`, Spring의 Springdoc처럼 코드에서 문서를 자동 생성하는 방식을 사용하면 항상 최신 상태를 유지할 수 있습니다.

```yaml
# OpenAPI 3.0 예시
paths:
  /articles/{id}:
    get:
      summary: 특정 글 조회
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: 조회 성공
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Article'
        '404':
          description: 글을 찾을 수 없음
```

---

## 마치며

REST API 설계는 한 번 정해지면 바꾸기 어렵기 때문에, 처음에 신중하게 설계하는 것이 중요합니다. 오늘 살펴본 7가지 원칙을 요약하면 다음과 같습니다.

1. **리소스 중심 URL** — 명사로 표현하고 계층 구조 활용
2. **HTTP 메서드 의미 준수** — GET/POST/PUT/PATCH/DELETE 목적에 맞게
3. **정확한 상태 코드** — 2xx/4xx/5xx 의미를 정확히 사용
4. **일관된 에러 형식** — code, message, details 구조화
5. **버전 관리** — URL 경로에 버전 포함으로 하위 호환성 확보
6. **페이지네이션 표준화** — 필터·정렬·페이지 파라미터 통일
7. **코드와 함께하는 문서** — OpenAPI로 자동화

이 원칙들은 완벽한 정답이 아니라 수많은 실무 경험에서 나온 권장 사항입니다. 팀의 상황과 서비스 특성에 맞게 규칙을 정하고, 무엇보다 **팀 내에서 일관성 있게 적용하는 것** 이 가장 중요합니다.
