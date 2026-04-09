# 05. ERD (Entity Relationship Diagram)

> **버전**: v1.0
> **표기법**: Mermaid ERD

---

## 1. ERD
```mermaid
erDiagram

    MEMBER {
        bigint id PK
        varchar email UK
        varchar password
        varchar nickname
        float average_rating
        datetime created_at
        datetime updated_at
        boolean is_deleted
    }

    CATEGORY {
        bigint id PK
        bigint parent_id FK
        varchar name
        int depth
    }

    PRODUCT {
        bigint id PK
        bigint seller_id FK
        bigint category_id FK
        varchar title
        varchar description
        int price
        varchar status
        int like_count
        datetime created_at
        datetime updated_at
        boolean is_deleted
    }

    PRODUCT_IMAGE {
        bigint id PK
        bigint product_id FK
        varchar image_url
        int sort_order
    }

    TRADE {
        bigint id PK
        bigint product_id FK
        bigint buyer_id FK
        bigint seller_id FK
        varchar status
        datetime reserved_at
        datetime traded_at
        datetime sold_at
        datetime created_at
    }

    REVIEW {
        bigint id PK
        bigint trade_id FK
        bigint reviewer_id FK
        bigint reviewee_id FK
        int rating
        varchar content
        datetime created_at
    }

    LIKE {
        bigint id PK
        bigint member_id FK
        bigint product_id FK
        datetime created_at
    }

    CHAT_ROOM {
        bigint id PK
        bigint product_id FK
        bigint buyer_id FK
        bigint seller_id FK
        datetime created_at
        datetime last_message_at
    }

    CHAT_MESSAGE {
        bigint id PK
        bigint chat_room_id FK
        bigint sender_id FK
        varchar content
        boolean is_read
        datetime created_at
    }

    KEYWORD_LOG {
        bigint id PK
        varchar keyword
        int count
        date log_date
    }

    MEMBER ||--o{ PRODUCT : sells
    MEMBER ||--o{ TRADE : buys
    MEMBER ||--o{ LIKE : likes
    MEMBER ||--o{ REVIEW : writes
    MEMBER ||--o{ REVIEW : receives
    MEMBER ||--o{ CHAT_ROOM : joins_as_buyer
    MEMBER ||--o{ CHAT_ROOM : joins_as_seller
    MEMBER ||--o{ CHAT_MESSAGE : sends

    CATEGORY ||--o{ CATEGORY : parent_of
    CATEGORY ||--o{ PRODUCT : classifies

    PRODUCT ||--o{ PRODUCT_IMAGE : has
    PRODUCT ||--o{ TRADE : records
    PRODUCT ||--o{ LIKE : receives
    PRODUCT ||--o{ CHAT_ROOM : has

    TRADE ||--|| REVIEW : has

    CHAT_ROOM ||--o{ CHAT_MESSAGE : contains
```
---

## 2. 설계 결정 사항

### 2-1. TRADE 테이블 분리 이유

상품(PRODUCT)과 거래(TRADE)를 분리한 이유:
- 상품은 취소 후 재판매 가능 → PRODUCT.status만 초기화하면 됨
- 거래 이력은 별도 보존 필요 (리뷰, 분쟁 대비)
- PRODUCT.id와 TRADE.product_id는 UK이지만, 취소 시 TRADE.status = CANCELLED로 변경하고 새 TRADE 생성 허용

### 2-2. LIKE.like_count 비정규화

- PRODUCT 테이블에 `like_count` 컬럼을 두어 목록 조회 시 JOIN 없이 찜 수 표시
- 정합성 허용 오차 ±1 수준 (실시간 정확도 불필요)
- 증가: `UPDATE product SET like_count = like_count + 1 WHERE id = ?`

### 2-3. CATEGORY 자기 참조 (2뎁스)

- `parent_id IS NULL` → 대분류
- `parent_id IS NOT NULL` → 소분류
- PRODUCT는 소분류 ID만 저장 (depth=1)
- 대분류 조회 필요 시 JOIN 또는 `parent_id`를 통해 역추적

### 2-4. CHAT_MESSAGE 저장소

- Redis Pub/Sub이 아닌 MySQL 영속 저장
- 이유: 재연결 시 이전 메시지 조회, 거래 증거 보존
- 인덱스: `(chat_room_id, id DESC)` — 최신 메시지 빠른 조회

### 2-5. KEYWORD_LOG 설계

- 검색마다 실시간 INSERT 대신 `count` 컬럼 UPDATE 또는 일배치 집계
- `log_date` 컬럼으로 최근 7일 집계 → 인기 검색어 산출

---

## 3. 인덱스 계획 (EXPLAIN 기반 최적화 대상)

| 테이블 | 인덱스 | 이유 |
|--------|--------|------|
| PRODUCT | `(status, created_at DESC)` | 목록 조회 기본 정렬 |
| PRODUCT | `(category_id, status)` | 카테고리 필터 검색 |
| PRODUCT | `(seller_id)` | 내 판매 목록 |
| PRODUCT | `FULLTEXT(title)` | 키워드 검색 (LIKE → FULLTEXT) |
| TRADE | `(buyer_id)` | 구매 이력 조회 |
| LIKE | `UNIQUE(member_id, product_id)` | 중복 방지 + 찜 여부 조회 |
| CHAT_MESSAGE | `(chat_room_id, id DESC)` | 채팅 이력 최신순 |
| KEYWORD_LOG | `(keyword, log_date)` | 인기 검색어 집계 |
