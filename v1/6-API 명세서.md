# 06. API 명세서

> **버전**: v1.0
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

**Response 409** (이미 예약됨):
```json
{ "code": "ALREADY_RESERVED", "message": "이미 예약된 상품입니다." }
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
| 401 | UNAUTHORIZED | 인증 실패 / 토큰 만료 |
| 403 | FORBIDDEN | 권한 없음 (타인 리소스) |
| 404 | NOT_FOUND | 리소스 없음 |
| 409 | DUPLICATE_EMAIL | 이메일 중복 |
| 409 | ALREADY_RESERVED | 이미 예약된 상품 |
| 500 | INTERNAL_ERROR | 서버 오류 |

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