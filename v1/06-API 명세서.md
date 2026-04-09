# 06. API 명세서

> **버전**: v1.2 (채팅방 생성 API 추가, 에러코드 통일, retryAfter 정책 명시)
> **Base URL**: `https://api.marketplace.com/api/v1`
> **인증**: `Authorization: Bearer {AccessToken}` (🔒 표시 API)

---

## 1. API 목록

### 인증 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| POST | `/auth/signup` | 회원가입 | ❌ |
| POST | `/auth/login` | 로그인 | ❌ |
| POST | `/auth/reissue` | 토큰 재발급 | ❌ |
| POST | `/auth/logout` | 로그아웃 | 🔒 |

### 회원 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| GET | `/members/me` | 내 프로필 조회 | 🔒 |
| PATCH | `/members/me` | 내 프로필 수정 | 🔒 |

### 카테고리 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| GET | `/categories` | 전체 카테고리 조회 | ❌ |

### 상품 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| GET | `/products` | 상품 목록 조회 (Cursor) | ❌ |
| GET | `/products/search` | 상품 검색 | ❌ |
| GET | `/products/{productId}` | 상품 상세 조회 | ❌ |
| POST | `/products` | 상품 등록 | 🔒 |
| PATCH | `/products/{productId}` | 상품 수정 | 🔒 |
| DELETE | `/products/{productId}` | 상품 삭제 | 🔒 |
| GET | `/members/me/products` | 내 판매 목록 | 🔒 |

### 거래 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| POST | `/products/{productId}/reserve` | 예약 요청 | 🔒 |
| PATCH | `/trades/{tradeId}/status` | 거래 상태 변경 | 🔒 |
| GET | `/trades/{tradeId}` | 거래 상세 조회 | 🔒 |

### 찜 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| POST | `/products/{productId}/likes` | 찜하기 | 🔒 |
| DELETE | `/products/{productId}/likes` | 찜 취소 | 🔒 |
| GET | `/members/me/likes` | 내 찜 목록 | 🔒 |

### 리뷰 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| POST | `/trades/{tradeId}/reviews` | 리뷰 작성 | 🔒 |
| GET | `/members/{memberId}/reviews` | 회원 리뷰 목록 | ❌ |

### 채팅 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| POST | `/chat-rooms` | 채팅방 생성 (이미 있으면 기존 반환) | 🔒 |
| GET | `/chat-rooms` | 내 채팅방 목록 | 🔒 |
| GET | `/chat-rooms/{chatRoomId}/messages` | 채팅 이력 조회 | 🔒 |

### 검색 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| GET | `/search/popular` | 인기 검색어 TOP 10 | ❌ |

---

## 2. 상세 명세

### POST /auth/signup

**Request Body**:
```json
{
  "email": "user@example.com",
  "password": "password123",
  "nickname": "민준"
}
```

**Response 201**:
```json
{
  "id": 1,
  "email": "user@example.com",
  "nickname": "민준"
}
```

**Response 409**:
```json
{ "code": "DUPLICATE_EMAIL", "message": "이미 사용 중인 이메일입니다." }
```

---

### POST /auth/login

**Request Body**:
```json
{ "email": "user@example.com", "password": "password123" }
```

**Response 200**:
```json
{
  "accessToken": "eyJ...",
  "refreshToken": "550e8400-e29b-41d4-a716-...",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```

**Response 401** (비밀번호 불일치):
```json
{ "code": "INVALID_CREDENTIALS", "message": "이메일 또는 비밀번호가 올바르지 않습니다." }
```

**Response 404** (존재하지 않는 이메일):
```json
{ "code": "MEMBER_NOT_FOUND", "message": "존재하지 않는 계정입니다." }
```

> **보안 참고**: 이메일 존재 여부를 노출하지 않으려면 401로 통일하는 방법도 있다. 사용자 편의를 위해 현재는 구분해서 반환한다.

---

### GET /products?cursor=&size=20

**Query Parameters**:

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| cursor | String | ❌ | 마지막 항목의 커서값 (없으면 첫 페이지) |
| size | Integer | ❌ | 기본값 20, 최대 50 |
| status | String | ❌ | SALE(기본), RESERVED, SOLD, ALL |

**Response 200**:
```json
{
  "content": [
    {
      "id": 42,
      "title": "침대 프레임 퀸사이즈",
      "price": 150000,
      "status": "SALE",
      "thumbnailUrl": "https://...",
      "likeCount": 7,
      "createdAt": "2025-06-01T10:00:00"
    }
  ],
  "nextCursor": "eyJjcmVhdGVkQXQiOiIyMDI1...",
  "hasNext": true
}
```

---

### GET /products/search

**Query Parameters**:

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| keyword | String | ❌ | 제목 검색어 |
| categoryId | Long | ❌ | 소분류 카테고리 ID |
| status | String | ❌ | 기본값 SALE |
| cursor | String | ❌ | 커서 기반 페이지네이션 |
| size | Integer | ❌ | 기본값 20 |

---

### POST /products (상품 등록)

**Request Body**:
```json
{
  "title": "침대 프레임 퀸사이즈",
  "description": "2년 사용, 상태 양호...",
  "price": 150000,
  "categoryId": 12,
  "imageUrls": ["https://...", "https://..."]
}
```

**Response 201**:
```json
{ "id": 42, "title": "침대 프레임 퀸사이즈", "status": "SALE" }
```

**Response 400** (유효성 오류):
```json
{
  "code": "INVALID_REQUEST",
  "message": "입력값이 올바르지 않습니다.",
  "errors": [
    { "field": "title", "message": "제목은 2자 이상 40자 이하여야 합니다." },
    { "field": "price", "message": "가격은 100원 이상이어야 합니다." }
  ]
}
```

**Response 401** (미인증):
```json
{ "code": "UNAUTHORIZED", "message": "로그인이 필요합니다." }
```

**이미지 수정 정책** (`PATCH /products/{productId}` 연동):
- `imageUrls` 배열을 전달하면 **전체 교체** 방식으로 처리 (기존 PRODUCT_IMAGE 전체 삭제 후 재등록)
- `imageUrls` 필드 미전달 시 이미지 변경 없음
- 이미지 개별 삭제·순서 변경 API는 Out of Scope (향후 확장)

---

### POST /products/{productId}/reserve (예약 요청 ⭐)

**Request Body**: 없음 (productId만 사용)

**Response 200** (예약 성공):
```json
{
  "tradeId": 99,
  "productId": 42,
  "status": "RESERVED",
  "chatRoomId": 55
}
```

**Response 400** (본인 상품 예약 시도):
```json
{ "code": "SELF_RESERVATION", "message": "본인의 상품은 예약할 수 없습니다." }
```

**Response 401** (미인증):
```json
{ "code": "UNAUTHORIZED", "message": "로그인이 필요합니다." }
```

**Response 404** (상품 없음):
```json
{ "code": "PRODUCT_NOT_FOUND", "message": "존재하지 않는 상품입니다." }
```

**Response 409** (이미 예약됨):
```json
{
  "code": "ALREADY_RESERVED",
  "message": "이미 예약된 상품입니다. 잠시 후 다시 확인해보세요.",
  "retryAfter": 1
}
```

---

### PATCH /trades/{tradeId}/status (거래 상태 변경)

**Request Body**:
```json
{ "status": "TRADING" }
```

**허용 전이** (호출자 기준):
- 판매자: RESERVED → TRADING, TRADING → SOLD, RESERVED|TRADING → SALE(취소)
- 구매자: RESERVED → SALE(예약 취소)

**Response 200**:
```json
{ "tradeId": 99, "status": "TRADING" }
```

**Response 400** (허용되지 않는 상태 전이):
```json
{ "code": "INVALID_STATUS_TRANSITION", "message": "현재 상태에서는 해당 전이가 불가능합니다." }
```

**Response 403** (권한 없음 — 타인 거래 변경 시도):
```json
{ "code": "FORBIDDEN", "message": "해당 거래에 대한 권한이 없습니다." }
```

**Response 404** (거래 없음):
```json
{ "code": "TRADE_NOT_FOUND", "message": "존재하지 않는 거래입니다." }
```

---

### POST /chat-rooms (채팅방 생성)

**Request Body**:
```json
{ "productId": 42 }
```

**처리 로직**:
1. 동일 `(buyer_id, product_id)` 채팅방 존재 여부 조회
2. 존재하면 → 기존 `chatRoomId` 반환 (멱등)
3. 없으면 → CHAT_ROOM 생성 후 반환

**Response 200** (기존 채팅방 존재):
```json
{ "chatRoomId": 55, "created": false }
```

**Response 201** (신규 채팅방 생성):
```json
{ "chatRoomId": 56, "created": true }
```

**Response 400** (본인 상품에 채팅 시도):
```json
{ "code": "SELF_CHAT", "message": "본인의 판매글에는 채팅을 시작할 수 없습니다." }
```

**Response 404** (상품 없음):
```json
{ "code": "PRODUCT_NOT_FOUND", "message": "존재하지 않는 상품입니다." }
```

---

### GET /search/popular

**Response 200** (Caffeine 캐시, TTL 10분):
```json
{
  "keywords": ["침대 프레임", "아이패드", "자전거", "원피스", "책상"],
  "cachedAt": "2025-06-01T10:00:00"
}
```

---

## 3. 공통 에러 응답

```json
{
  "code": "ERROR_CODE",
  "message": "에러 설명",
  "timestamp": "2025-06-01T10:00:00"
}
```

| HTTP Status | Code | 설명 |
|-------------|------|------|
| 400 | INVALID_REQUEST | 요청 값 유효성 오류 |
| 400 | INVALID_STATUS_TRANSITION | 허용되지 않는 거래 상태 전이 |
| 400 | SELF_RESERVATION | 본인 상품 예약 시도 |
| 400 | SELF_CHAT | 본인 상품에 채팅 시도 |
| 401 | UNAUTHORIZED | 인증 실패 / 토큰 만료 |
| 401 | INVALID_CREDENTIALS | 이메일 또는 비밀번호 불일치 |
| 403 | FORBIDDEN | 권한 없음 (타인 리소스 접근) |
| 404 | NOT_FOUND | 리소스 없음 |
| 404 | MEMBER_NOT_FOUND | 존재하지 않는 회원 |
| 404 | PRODUCT_NOT_FOUND | 존재하지 않는 상품 |
| 404 | TRADE_NOT_FOUND | 존재하지 않는 거래 |
| 409 | DUPLICATE_EMAIL | 이메일 중복 |
| 409 | ALREADY_RESERVED | 이미 예약된 상품 (동시 요청 포함, 중복 클릭 포함) |
| 409 | LOCK_TIMEOUT | DB 락 대기 타임아웃 — 잠시 후 재시도 안내 |
| 500 | INTERNAL_ERROR | 서버 오류 |

> **에러코드 통일 이유**: `DUPLICATE_RESERVATION`과 `ALREADY_RESERVED`는 동일한 상황(이미 예약된 상품)을 가리키므로 `ALREADY_RESERVED`로 단일화한다. 중복 클릭, 동시 요청, 타인 선점 모두 이 코드를 사용한다.

### retryAfter 정책

`ALREADY_RESERVED` 응답에는 `retryAfter` 힌트를 포함한다.

```json
{
  "code": "ALREADY_RESERVED",
  "message": "이미 예약된 상품입니다. 잠시 후 상태를 다시 확인해보세요.",
  "retryAfter": 1
}
```

- `retryAfter: 1` = 1초 후 상품 상세 재조회 권장
- 클라이언트는 이 값을 보고 1초 후 `GET /products/{id}`를 호출해 현재 status를 확인
- 예약 취소로 인해 다시 SALE이 됐다면 예약 버튼을 재활성화
- 이 방식은 서버 Push 없이 단순 폴링으로 상태 동기화를 달성하는 최소 구현

---

## 4. WebSocket/STOMP 명세

### 연결

```
ws://api.marketplace.com/ws-chat
STOMP CONNECT headers:
  Authorization: Bearer {AccessToken}
```

### 구독

| Destination | 설명 |
|-------------|------|
| `/topic/room.{chatRoomId}` | 채팅방 메시지 수신 |
| `/user/queue/errors` | 에러 메시지 수신 |

### 메시지 전송

```
STOMP SEND /app/chat.sendMessage
Body:
{
  "chatRoomId": 55,
  "content": "아직 판매 중인가요?"
}
```

### 수신 메시지 형식

```json
{
  "messageId": 1001,
  "chatRoomId": 55,
  "senderId": 7,
  "senderNickname": "이서연",
  "content": "아직 판매 중인가요?",
  "createdAt": "2025-06-01T10:05:00"
}
```

---

## 5. OpenAPI YAML (핵심 엔드포인트)

```yaml
openapi: 3.0.3
info:
  title: 중고거래 플랫폼 API
  version: 1.0.0
  description: 당근마켓 스타일 중고거래 백엔드

servers:
  - url: https://api.marketplace.com/api/v1

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
  schemas:
    ErrorResponse:
      type: object
      properties:
        code:
          type: string
        message:
          type: string
        timestamp:
          type: string
          format: date-time

paths:
  /auth/login:
    post:
      tags: [Auth]
      summary: 로그인
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [email, password]
              properties:
                email:
                  type: string
                  format: email
                password:
                  type: string
                  minLength: 8
      responses:
        '200':
          description: 로그인 성공
          content:
            application/json:
              schema:
                type: object
                properties:
                  accessToken:
                    type: string
                  refreshToken:
                    type: string
                  tokenType:
                    type: string
                  expiresIn:
                    type: integer
        '401':
          description: 인증 실패

  /products:
    get:
      tags: [Product]
      summary: 상품 목록 조회 (Cursor 페이지네이션)
      parameters:
        - name: cursor
          in: query
          schema:
            type: string
        - name: size
          in: query
          schema:
            type: integer
            default: 20
        - name: status
          in: query
          schema:
            type: string
            enum: [SALE, RESERVED, SOLD, ALL]
            default: SALE
      responses:
        '200':
          description: 성공
    post:
      tags: [Product]
      summary: 상품 등록
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [title, price, categoryId]
              properties:
                title:
                  type: string
                  minLength: 2
                  maxLength: 40
                description:
                  type: string
                  maxLength: 1000
                price:
                  type: integer
                  minimum: 100
                  maximum: 99999999
                categoryId:
                  type: integer
                imageUrls:
                  type: array
                  items:
                    type: string
                  maxItems: 10
      responses:
        '201':
          description: 등록 성공
        '401':
          description: 인증 필요

  /products/{productId}/reserve:
    post:
      tags: [Trade]
      summary: 상품 예약 요청 (동시성 제어 적용)
      security:
        - bearerAuth: []
      parameters:
        - name: productId
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: 예약 성공
        '409':
          description: 이미 예약된 상품
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

  /search/popular:
    get:
      tags: [Search]
      summary: 인기 검색어 TOP 10 (Caffeine 캐시)
      responses:
        '200':
          description: 성공
          content:
            application/json:
              schema:
                type: object
                properties:
                  keywords:
                    type: array
                    items:
                      type: string
                  cachedAt:
                    type: string
                    format: date-time
```
