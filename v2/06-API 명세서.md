# 06. API 명세서

> **버전**: v3.0 (위치 API 추가 — 동네 인증·근처 상품 조회, 에러코드 전체 정합성 수정)
> **인증**: `Authorization: Bearer {AccessToken}` (🔒 표시 API)

---

## 1. 공통 응답 형식

### 성공 응답

```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": { ... }
}
```

> `data`가 null인 경우 JSON에서 제외됨 (`@JsonInclude(NON_NULL)`)

### 에러 응답

```json
{
  "code": "M001",
  "message": "존재하지 않는 회원입니다.",
  "timestamp": "2025-06-01T10:00:00"
}
```

> `retryAfter` 필드: `ALREADY_RESERVED`, `LOCK_TIMEOUT` 에러 시에만 선택적으로 포함됨.

### 유효성 오류 응답

```json
{
  "code": "INVALID_REQUEST",
  "message": "입력값이 올바르지 않습니다.",
  "errors": [
    { "field": "title", "message": "제목은 2자 이상 40자 이하여야 합니다." }
  ]
}
```

---

## 2. API 목록

### 인증 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| POST | `/auth/signup` | 회원가입 | ❌ |
| POST | `/auth/login` | 로그인 | ❌ |
| POST | `/auth/refresh` | 토큰 재발급 (RT Rotation) | ❌ |
| POST | `/auth/logout` | 로그아웃 | 🔒 |

### 회원 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| GET | `/members/me` | 내 프로필 조회 | 🔒 |
| PATCH | `/members/me` | 내 프로필 수정 | 🔒 |
| PATCH | `/members/me/password` | 비밀번호 변경 | 🔒 |
| DELETE | `/members/me` | 회원 탈퇴 | 🔒 |
| GET | `/members/{memberId}` | 상대방 프로필 조회 | ❌ |

### 카테고리 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| GET | `/categories` | 전체 카테고리 조회 | ❌ |

### 상품 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| POST | `/products` | 상품 등록 | 🔒 |
| GET | `/products` | 상품 목록/검색 (통합, Cursor) | ❌ |
| GET | `/products/{productId}` | 상품 상세 조회 | ❌ |
| PATCH | `/products/{productId}` | 상품 수정 | 🔒 |
| DELETE | `/products/{productId}` | 상품 삭제 | 🔒 |
| GET | `/products/me` | 내 판매 목록 | 🔒 |

### 거래 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| POST | `/products/{productId}/reserve` | 예약 요청 | 🔒 |
| PATCH | `/trades/{tradeId}/status` | 거래 상태 변경 | 🔒 |
| GET | `/trades/{tradeId}` | 거래 상세 조회 | 🔒 |
| GET | `/members/me/trades` | 내 거래 리스트 | 🔒 |

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
| GET | `/members/{memberId}/reviews` | 받은 리뷰 목록 | ❌ |
| GET | `/members/{memberId}/reviews/written` | 작성한 리뷰 목록 | ❌ |

### 채팅 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| POST | `/chat-rooms` | 채팅방 생성 (멱등) | 🔒 |
| GET | `/chat-rooms` | 내 채팅방 목록 | 🔒 |
| GET | `/chat-rooms/{chatRoomId}/messages` | 채팅 이력 조회 | 🔒 |
| DELETE | `/chat-rooms/{chatRoomId}/leave` | 채팅방 나가기 | 🔒 |
| PATCH | `/chat-rooms/{chatRoomId}/read` | 읽음 처리 | 🔒 |

### 위치 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| POST | `/api/location/verify/gps` | GPS 동네 인증 | 🔒 |
| POST | `/api/location/verify/address` | 주소 텍스트 동네 인증 | 🔒 |
| GET | `/api/products/nearby` | 근처 상품 조회 | 🔒 |

### 검색 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| GET | `/search/popular` | 인기 검색어 TOP 10 (Redis ZSET) | ❌ |

---

## 3. 상세 명세

### POST /auth/signup

**Request Body**:
```json
{
  "email": "user@example.com",
  "password": "password123!",
  "nickname": "민준",
  "profileImageUrl": "https://example.com/profile.jpg"
}
```

| 필드 | 타입 | 필수 | 검증 |
|------|------|------|------|
| email | String | ✅ | 이메일 형식 |
| password | String | ✅ | 8자 이상, 영문·숫자·특수문자 각 1자 이상 |
| nickname | String | ✅ | 2~20자 |
| profileImageUrl | String | ❌ | URL 또는 data URI 형식 |

**Response 201**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {
    "id": 1,
    "email": "user@example.com",
    "nickname": "민준",
    "createdAt": "2025-06-01T10:00:00"
  }
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
{ "email": "user@example.com", "password": "password123!" }
```

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {
    "accessToken": "eyJ...",
    "refreshToken": "eyJ...",
    "tokenType": "Bearer",
    "expiresIn": 900
  }
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

---

### POST /auth/refresh

**Request Body**:
```json
{ "refreshToken": "eyJ..." }
```

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {
    "accessToken": "eyJ...(새 AT)",
    "refreshToken": "eyJ...(새 RT)",
    "tokenType": "Bearer",
    "expiresIn": 900
  }
}
```

**Response 401**:
```json
{ "code": "TOKEN_EXPIRED", "message": "토큰이 만료되었습니다. 다시 로그인해주세요." }
```

> AT가 만료된 상태에서 호출하므로 Authorization 헤더 대신 Body로 RT를 전달.
> 성공 시 AT + RT 모두 새로 발급 (RT Rotation).

---

### POST /auth/logout 🔒

**Request Body**: 없음

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "로그아웃 되었습니다."
}
```

> AT를 Redis 블랙리스트에 등록, RT 삭제.

---

### GET /members/me 🔒

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {
    "id": 1,
    "email": "user@example.com",
    "nickname": "민준",
    "profileImageUrl": "https://...",
    "averageRating": 4.5,
    "reviewCount": 10,
    "createdAt": "2025-06-01T10:00:00"
  }
}
```

---

### PATCH /members/me 🔒

**Request Body**:
```json
{
  "nickname": "새닉네임",
  "profileImageUrl": "https://new-image.com/img.jpg"
}
```

> `profileImageUrl` 처리: `null` = 변경 없음, `""` = 이미지 삭제, `URL/data URI` = 업데이트

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {
    "id": 1,
    "email": "user@example.com",
    "nickname": "새닉네임",
    "profileImageUrl": "https://new-image.com/img.jpg",
    "averageRating": 4.5,
    "reviewCount": 10,
    "createdAt": "2025-06-01T10:00:00"
  }
}
```

---

### PATCH /members/me/password 🔒

**Request Body**:
```json
{
  "oldPassword": "password123!",
  "newPassword": "newPassword456!"
}
```

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "비밀번호가 변경되었습니다."
}
```

---

### DELETE /members/me 🔒

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "회원탈퇴가 완료되었습니다."
}
```

---

### GET /members/{memberId}

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {
    "id": 5,
    "nickname": "이서연",
    "profileImageUrl": "https://...",
    "averageRating": 4.8,
    "reviewCount": 23,
    "createdAt": "2025-06-01T10:00:00"
  }
}
```

> 비로그인도 조회 가능. email은 제외 (MemberProfileResponse).

---

### GET /categories

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": [
    { "id": 1, "name": "전자기기" },
    { "id": 2, "name": "가구/인테리어" }
  ]
}
```

---

### POST /products 🔒

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

| 필드 | 타입 | 필수 | 검증 |
|------|------|------|------|
| title | String | ✅ | 2~40자 |
| description | String | ✅ | - |
| price | Integer | ✅ | 100 이상 |
| categoryId | Long | ✅ | - |
| imageUrls | List\<String\> | ❌ | - |

**Response 201**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {
    "id": 42,
    "sellerId": 1,
    "title": "침대 프레임 퀸사이즈",
    "description": "2년 사용, 상태 양호...",
    "price": 150000,
    "status": "SALE",
    "categoryName": "가구/인테리어",
    "sellerNickname": "민준",
    "imageUrls": ["https://...", "https://..."],
    "createdAt": "2025-06-01T10:00:00"
  }
}
```

**이미지 수정 정책** (`PATCH /products/{productId}` 연동):
- `imageUrls` 배열을 전달하면 **전체 교체** 방식으로 처리 (기존 PRODUCT_IMAGE 전체 삭제 후 재등록)
- `imageUrls` 필드 미전달 시 이미지 변경 없음
- 이미지 개별 삭제·순서 변경 API는 Out of Scope (향후 확장)

---

### GET /products

상품 목록 조회와 검색을 하나의 API로 통합.

**Query Parameters**:

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| keyword | String | ❌ | 제목 검색어 (검색 시 Redis 점수 반영) |
| categoryId | Long | ❌ | 카테고리 ID (Caffeine 캐시 활용) |
| status | String | ❌ | SALE / RESERVED / SOLD / TRADING |
| cursor | Long | ❌ | 마지막 항목의 ID (없으면 첫 페이지) |
| size | Integer | ❌ | 기본값 20 |

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {
    "content": [
      {
        "id": 42,
        "title": "침대 프레임 퀸사이즈",
        "price": 150000,
        "status": "SALE",
        "thumbnailUrl": "https://...",
        "createdAt": "2025-06-01T10:00:00"
      }
    ],
    "nextCursor": "eyJjcmVhdGVkQXQiOiIyMDI1...",
    "hasNext": true
  }
}
```

---

### GET /products/{productId}

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {
    "id": 42,
    "sellerId": 1,
    "title": "침대 프레임 퀸사이즈",
    "description": "2년 사용, 상태 양호...",
    "price": 150000,
    "status": "SALE",
    "categoryName": "가구/인테리어",
    "sellerNickname": "민준",
    "imageUrls": ["https://...", "https://..."],
    "createdAt": "2025-06-01T10:00:00"
  }
}
```

---

### GET /products/me 🔒

**Query Parameters**:

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| status | String | ❌ | SALE / RESERVED / SOLD_OUT |
| cursor | String | ❌ | 커서 |
| size | Integer | ❌ | 기본값 10 |

**Response 200**: `CursorPageResponse<ProductListResponse>` (GET /products와 동일 구조)

---

### POST /products/{productId}/reserve 🔒

**Request Body**: 없음 (productId만 사용)

**Response 200** (예약 성공):
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {
    "tradeId": 99,
    "productTitle": "침대 프레임 퀸사이즈",
    "sellerNickname": "민준",
    "buyerNickname": "이서연"
  }
}
```

**Response 400** (본인 상품 예약 시도):
```json
{ "code": "SELF_RESERVATION", "message": "본인의 상품은 예약할 수 없습니다." }
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

### PATCH /trades/{tradeId}/status 🔒

**Request Body**:
```json
{ "status": "TRADING" }
```

**허용 전이** (호출자 기준):
- 판매자: RESERVED → TRADING
- 구매자: TRADING → SOLD
- 양쪽 모두: RESERVED|TRADING → SALE(취소)

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다."
}
```

**Response 400** (허용되지 않는 상태 전이):
```json
{ "code": "INVALID_STATUS_TRANSITION", "message": "현재 상태에서는 해당 전이가 불가능합니다." }
```

**Response 403** (권한 없음):
```json
{ "code": "TRADE_FORBIDDEN", "message": "해당 거래에 대한 권한이 없습니다." }
```

---

### GET /trades/{tradeId} 🔒

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {
    "productTitle": "침대 프레임 퀸사이즈",
    "status": "RESERVED",
    "price": 150000,
    "productCreatedAt": "2025-06-01T10:00:00",
    "soldAt": null,
    "sellerNickname": "민준",
    "buyerNickname": "이서연"
  }
}
```

---

### GET /members/me/trades 🔒

**Query Parameters**:

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| role | String | ✅ | SELLER / BUYER |

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": [
    {
      "tradeId": 99,
      "productTitle": "침대 프레임 퀸사이즈",
      "productId": 42,
      "price": 150000,
      "status": "RESERVED",
      "updatedAt": "2025-06-01T10:00:00"
    }
  ]
}
```

---

### POST /products/{productId}/likes 🔒

**Request Body**: 없음

**Response 201**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다."
}
```

---

### DELETE /products/{productId}/likes 🔒

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다."
}
```

---

### GET /members/me/likes 🔒

**Query Parameters**:

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| page | Integer | ❌ | 기본값 0 (Offset 기반) |
| size | Integer | ❌ | 기본값 10 |
| sort | String | ❌ | 기본값 createdAt,DESC |

**Response 200** (Offset 기반 Page):
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {
    "content": [
      {
        "id": 42,
        "title": "침대 프레임 퀸사이즈",
        "price": 150000,
        "status": "SALE",
        "thumbnailUrl": "https://...",
        "createdAt": "2025-06-01T10:00:00"
      }
    ],
    "totalElements": 23,
    "totalPages": 3,
    "size": 10,
    "number": 0
  }
}
```

---

### POST /trades/{tradeId}/reviews 🔒

**Request Body**:
```json
{
  "rating": 4.5,
  "content": "친절하게 거래해주셨어요."
}
```

| 필드 | 타입 | 필수 | 검증 |
|------|------|------|------|
| rating | BigDecimal | ✅ | 0.5~5.0, 0.5 단위 |
| content | String | ✅ | 500자 이내 |

**Response 201**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다."
}
```

---

### GET /members/{memberId}/reviews

**Query Parameters**:

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| cursor | String | ❌ | 커서 |
| size | Integer | ❌ | 기본값 10, 최대 50 |

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {
    "content": [
      {
        "id": 1,
        "reviewerId": 7,
        "rating": 4.5,
        "content": "친절하게 거래해주셨어요.",
        "createdAt": "2025-06-01T14:00:00",
        "reviewerNickname": "이서연",
        "reviewerProfileImageUrl": "https://...",
        "productTitle": "침대 프레임 퀸사이즈"
      }
    ],
    "nextCursor": "eyJ...",
    "hasNext": true
  }
}
```

---

### GET /members/{memberId}/reviews/written

작성한 리뷰 목록 조회. 응답 구조는 `GET /members/{memberId}/reviews`와 동일.

---

### POST /chat-rooms 🔒

**Request Body**:
```json
{ "productId": 42 }
```

**처리 로직**:
1. productId로 Product 조회 → sellerId 자동 추출
2. 동일 `(buyer_id, product_id)` 채팅방 존재 여부 조회
3. 존재하면 → 기존 `chatRoomId` 반환 (멱등)
4. 없으면 → CHAT_ROOM 생성 후 반환

**Response 200** (기존 채팅방 존재):
```json
{
  "code": "SUCCESS",
  "message": "기존 채팅방을 반환합니다.",
  "data": { "chatRoomId": 55, "created": false }
}
```

**Response 201** (신규 채팅방 생성):
```json
{
  "code": "SUCCESS",
  "message": "채팅방이 생성되었습니다.",
  "data": { "chatRoomId": 56, "created": true }
}
```

**Response 400** (본인 상품에 채팅 시도):
```json
{ "code": "SELF_CHAT", "message": "본인의 판매글에는 채팅을 시작할 수 없습니다." }
```

---

### GET /chat-rooms 🔒

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": [
    {
      "chatRoomId": 55,
      "opponentId": 5,
      "opponentNickname": "민준",
      "productId": 42,
      "opponentProfileImageUrl": "https://...",
      "lastMessage": "네, 아직 있습니다.",
      "unreadCount": 3
    }
  ]
}
```

---

### GET /chat-rooms/{chatRoomId}/messages 🔒

**Query Parameters**:

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| cursor | Long | ❌ | 마지막으로 내려준 `messageId` (없으면 최신부터) |
| size | Integer | ❌ | 기본값 30 |

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {
    "content": [
      {
        "messageId": 1003,
        "chatRoomId": 55,
        "senderId": 7,
        "senderNickname": "이서연",
        "content": "네, 아직 있습니다.",
        "isRead": true,
        "createdAt": "2025-06-01T10:06:00"
      }
    ],
    "nextCursor": "1002",
    "hasNext": true
  }
}
```

---

### DELETE /chat-rooms/{chatRoomId}/leave 🔒

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "채팅방을 성공적으로 나갔습니다."
}
```

---

### PATCH /chat-rooms/{chatRoomId}/read 🔒

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "메시지를 읽음 처리했습니다."
}
```

---

### POST /api/location/verify/gps 🔒

**Request Body**:
```json
{
  "lat": 37.549,
  "lng": 126.913
}
```

| 필드 | 타입 | 필수 | 검증 |
|------|------|------|------|
| lat | Double | ✅ | -90.0 ~ 90.0 |
| lng | Double | ✅ | -180.0 ~ 180.0 |

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "동네 인증이 완료되었습니다.",
  "data": {
    "locationName": "마포구 합정동",
    "locationRadius": 3
  }
}
```

**Response 400**:
```json
{ "code": "KAKAO_REGION_NOT_FOUND", "message": "해당 좌표의 행정구역 정보를 찾을 수 없습니다." }
```

**Response 502**:
```json
{ "code": "KAKAO_API_ERROR", "message": "카카오 API 호출에 실패했습니다." }
```

---

### POST /api/location/verify/address 🔒

**Request Body**:
```json
{
  "address": "서울 마포구 합정동"
}
```

| 필드 | 타입 | 필수 | 검증 |
|------|------|------|------|
| address | String | ✅ | 1자 이상, 200자 이하 |

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "동네 인증이 완료되었습니다.",
  "data": {
    "locationName": "마포구 합정동",
    "locationRadius": 3
  }
}
```

**Response 400** (주소 좌표 변환 실패):
```json
{ "code": "KAKAO_ADDRESS_NOT_FOUND", "message": "입력한 주소로 좌표를 찾을 수 없습니다." }
```

---

### GET /api/products/nearby 🔒

**Query Parameters**:

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| page | Integer | ❌ | 기본값 0 (Offset 기반, 페이지 크기 20) |

**Response 200**:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {
    "content": [
      {
        "id": 42,
        "title": "침대 프레임 퀸사이즈",
        "price": 150000,
        "status": "SALE",
        "categoryName": "가구/인테리어",
        "sellerNickname": "민준",
        "thumbnailUrl": "https://...",
        "locationName": "마포구 합정동",
        "distanceKm": 0.8,
        "lat": 37.549,
        "lng": 126.913
      }
    ],
    "totalElements": 34,
    "totalPages": 2,
    "size": 20,
    "number": 0
  }
}
```

**Response 400** (동네 인증 미완료):
```json
{ "code": "LOCATION_NOT_VERIFIED", "message": "동네 인증이 필요합니다." }
```

> 거리 오름차순 정렬. 본인 상품·삭제 상품·탈퇴 회원 상품 제외. 반경은 인증 시 저장된 `locationRadius`(km) 기준.

---

### GET /search/popular

**Response 200** (Caffeine 캐시):
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": [
    { "keyword": "침대 프레임", "searchCount": 152 },
    { "keyword": "아이패드", "searchCount": 98 },
    { "keyword": "자전거", "searchCount": 75 }
  ]
}
```

---

## 4. 공통 에러 코드

| HTTP Status | Code | 설명 |
|-------------|------|------|
| 400 | INVALID_REQUEST | 요청 값 유효성 오류 |
| 400 | INVALID_STATUS_TRANSITION | 허용되지 않는 거래 상태 전이 |
| 400 | INVALID_RATING | 잘못된 평점 (0.5~5.0, 0.5 단위 아님) |
| 400 | SELF_RESERVATION | 본인 상품 예약 시도 |
| 400 | SELF_CHAT | 본인 상품에 채팅 시도 |
| 400 | SAME_AS_CURRENT_PASSWORD | 새 비밀번호가 현재 비밀번호와 동일 |
| 400 | CHAT_ALREADY_LEFT | 이미 나간 채팅방에 재요청 |
| 400 | KAKAO_ADDRESS_NOT_FOUND | 입력 주소로 좌표 변환 실패 |
| 400 | KAKAO_REGION_NOT_FOUND | 좌표로 행정구역명 변환 실패 |
| 400 | LOCATION_NOT_VERIFIED | 동네 인증 미완료 상태에서 근처 상품 조회 시도 |
| 401 | UNAUTHORIZED | 인증 실패 / 토큰 만료 |
| 401 | INVALID_CREDENTIALS | 이메일 또는 비밀번호 불일치 (로그인) |
| 401 | INVALID_PASSWORD | 현재 비밀번호 불일치 (비밀번호 변경) |
| 401 | TOKEN_EXPIRED | Refresh Token 만료 |
| 401 | TOKEN_INVALID | 토큰 위변조 또는 Redis 저장값 불일치 |
| 401 | LOGGED_OUT_TOKEN | 블랙리스트에 등록된 AccessToken 재사용 |
| 403 | TRADE_FORBIDDEN | 거래 권한 없음 |
| 403 | CHAT_FORBIDDEN | 채팅방 권한 없음 |
| 403 | PRODUCT_FORBIDDEN | 상품 권한 없음 |
| 404 | MEMBER_NOT_FOUND | 존재하지 않는 회원 |
| 404 | PRODUCT_NOT_FOUND | 존재하지 않는 상품 |
| 404 | TRADE_NOT_FOUND | 존재하지 않는 거래 |
| 404 | CHAT_ROOM_NOT_FOUND | 존재하지 않는 채팅방 |
| 404 | CATEGORY_NOT_FOUND | 존재하지 않는 카테고리 |
| 409 | DUPLICATE_EMAIL | 이메일 중복 |
| 409 | DUPLICATE_NICKNAME | 닉네임 중복 |
| 409 | ALREADY_RESERVED | 이미 예약된 상품 |
| 409 | LOCK_TIMEOUT | DB 락 대기 타임아웃 |
| 500 | INTERNAL_ERROR | 서버 오류 |
| 502 | KAKAO_API_ERROR | 카카오 지도 API 호출 실패 |

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

---

## 5. WebSocket/STOMP 명세

### 연결

```
wss://api.marketplace.com/ws-chat
STOMP CONNECT headers:
  Authorization: Bearer {AccessToken}
```

> **JWT 정책**: WebSocket은 STOMP CONNECT 시점에만 JWT를 검증한다. 연결이 수립된 뒤 AccessToken이 만료되어도 기존 세션은 유지되며, 이후 메시지 처리에서는 채팅방 참여자 여부만 확인한다.

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
  "isRead": false,
  "createdAt": "2025-06-01T10:05:00"
}
```

### 읽음 처리 이벤트

```json
{
  "type": "READ",
  "chatRoomId": 55,
  "readerId": 5
}
```

---

## 6. OpenAPI YAML (핵심 엔드포인트)

```yaml
openapi: 3.0.3
info:
  title: 중고거래 플랫폼 API
  version: 2.0.0
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
    ApiResponse:
      type: object
      properties:
        code:
          type: string
          example: "SUCCESS"
        message:
          type: string
        data:
          type: object
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
  /auth/signup:
    post:
      tags: [Auth]
      summary: 회원가입
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [email, password, nickname]
              properties:
                email:
                  type: string
                  format: email
                password:
                  type: string
                  minLength: 8
                nickname:
                  type: string
                  minLength: 2
                  maxLength: 20
                profileImageUrl:
                  type: string
      responses:
        '201':
          description: 회원가입 성공
        '409':
          description: 이메일 중복

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
                  code:
                    type: string
                  data:
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

  /auth/refresh:
    post:
      tags: [Auth]
      summary: 토큰 재발급 (RT Rotation)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [refreshToken]
              properties:
                refreshToken:
                  type: string
      responses:
        '200':
          description: 재발급 성공
        '401':
          description: RT 만료 또는 무효

  /products:
    get:
      tags: [Product]
      summary: 상품 목록/검색 (통합, Cursor 페이지네이션)
      parameters:
        - name: keyword
          in: query
          schema:
            type: string
        - name: categoryId
          in: query
          schema:
            type: integer
        - name: status
          in: query
          schema:
            type: string
            enum: [SALE, RESERVED, SOLD_OUT]
        - name: cursor
          in: query
          schema:
            type: string
        - name: size
          in: query
          schema:
            type: integer
            default: 20
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
              required: [title, description, price, categoryId]
              properties:
                title:
                  type: string
                  minLength: 2
                  maxLength: 40
                description:
                  type: string
                price:
                  type: integer
                  minimum: 100
                categoryId:
                  type: integer
                imageUrls:
                  type: array
                  items:
                    type: string
      responses:
        '201':
          description: 등록 성공
        '401':
          description: 인증 필요

  /products/{productId}:
    get:
      tags: [Product]
      summary: 상품 상세 조회
      parameters:
        - name: productId
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: 성공
    patch:
      tags: [Product]
      summary: 상품 수정
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
          description: 수정 성공
    delete:
      tags: [Product]
      summary: 상품 삭제
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
          description: 삭제 성공

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

  /products/{productId}/likes:
    post:
      tags: [Like]
      summary: 찜하기
      security:
        - bearerAuth: []
      parameters:
        - name: productId
          in: path
          required: true
          schema:
            type: integer
      responses:
        '201':
          description: 찜 등록
    delete:
      tags: [Like]
      summary: 찜 취소
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
          description: 찜 취소

  /trades/{tradeId}/status:
    patch:
      tags: [Trade]
      summary: 거래 상태 변경
      security:
        - bearerAuth: []
      parameters:
        - name: tradeId
          in: path
          required: true
          schema:
            type: integer
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [status]
              properties:
                status:
                  type: string
                  enum: [RESERVED, TRADING, SOLD, SALE]
      responses:
        '200':
          description: 상태 변경 성공

  /trades/{tradeId}/reviews:
    post:
      tags: [Review]
      summary: 리뷰 작성
      security:
        - bearerAuth: []
      parameters:
        - name: tradeId
          in: path
          required: true
          schema:
            type: integer
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [rating, content]
              properties:
                rating:
                  type: number
                  minimum: 0.5
                  maximum: 5.0
                content:
                  type: string
                  maxLength: 500
      responses:
        '201':
          description: 리뷰 작성 성공

  /chat-rooms:
    post:
      tags: [Chat]
      summary: 채팅방 생성 (멱등)
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [productId]
              properties:
                productId:
                  type: integer
      responses:
        '200':
          description: 기존 채팅방 반환
        '201':
          description: 신규 채팅방 생성
    get:
      tags: [Chat]
      summary: 내 채팅방 목록
      security:
        - bearerAuth: []
      responses:
        '200':
          description: 성공

  /search/popular:
    get:
      tags: [Search]
      summary: 인기 검색어 TOP 10
      responses:
        '200':
          description: 성공
          content:
            application/json:
              schema:
                type: object
                properties:
                  code:
                    type: string
                  data:
                    type: array
                    items:
                      type: object
                      properties:
                        keyword:
                          type: string
                        searchCount:
                          type: integer

  /api/location/verify/gps:
    post:
      tags: [Location]
      summary: GPS 동네 인증
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [lat, lng]
              properties:
                lat:
                  type: number
                  minimum: -90.0
                  maximum: 90.0
                  description: 위도
                lng:
                  type: number
                  minimum: -180.0
                  maximum: 180.0
                  description: 경도
      responses:
        '200':
          description: 동네 인증 성공
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: object
                    properties:
                      locationName:
                        type: string
                        example: "마포구 합정동"
                      locationRadius:
                        type: integer
                        example: 3
        '400':
          description: 좌표→행정구역 변환 실패 (KAKAO_REGION_NOT_FOUND)
        '502':
          description: 카카오 API 오류 (KAKAO_API_ERROR)

  /api/location/verify/address:
    post:
      tags: [Location]
      summary: 주소 텍스트 동네 인증
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [address]
              properties:
                address:
                  type: string
                  maxLength: 200
                  example: "서울 마포구 합정동"
      responses:
        '200':
          description: 동네 인증 성공
        '400':
          description: 주소 변환 실패 (KAKAO_ADDRESS_NOT_FOUND) 또는 행정구역 변환 실패 (KAKAO_REGION_NOT_FOUND)
        '502':
          description: 카카오 API 오류 (KAKAO_API_ERROR)

  /api/products/nearby:
    get:
      tags: [Location]
      summary: 근처 상품 조회 (Offset 페이지네이션, 거리 오름차순)
      security:
        - bearerAuth: []
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 0
            minimum: 0
          description: 페이지 번호 (페이지 크기 20 고정)
      responses:
        '200':
          description: 성공
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: object
                    properties:
                      content:
                        type: array
                        items:
                          type: object
                          properties:
                            id:
                              type: integer
                            title:
                              type: string
                            price:
                              type: integer
                            status:
                              type: string
                              enum: [SALE, RESERVED, SOLD_OUT]
                            locationName:
                              type: string
                            distanceKm:
                              type: number
                              example: 0.8
                            lat:
                              type: number
                            lng:
                              type: number
                      totalElements:
                        type: integer
                      totalPages:
                        type: integer
        '400':
          description: 동네 인증 미완료 (LOCATION_NOT_VERIFIED)
```
