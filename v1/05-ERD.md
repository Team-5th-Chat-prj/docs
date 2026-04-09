# 05. ERD (Entity Relationship Diagram)

> **버전**: v1.1
> **표기법**: Mermaid ERD
>
> **개념 정의**: `PRODUCT` = 중고 판매글(Listing). 카탈로그 상품이 아님. 판매자가 올린 "이 물건을 팝니다" 게시글 1건 = PRODUCT 1건.

---

## 1. ERD

```mermaid
erDiagram

    MEMBER {
        bigint id PK
        varchar email UK "NOT NULL"
        varchar password "BCrypt 해시"
        varchar nickname
        float average_rating "판매자 평점, 기본값 0.0"
        datetime created_at
        datetime updated_at
        boolean is_deleted "소프트 삭제"
    }

    CATEGORY {
        bigint id PK
        bigint parent_id FK "NULL이면 대분류"
        varchar name
        int depth "0=대분류, 1=소분류"
    }

    PRODUCT {
        bigint id PK
        bigint seller_id FK "MEMBER.id"
        bigint category_id FK "CATEGORY.id (소분류)"
        varchar title "2~40자"
        text description
        int price
        enum status "SALE|RESERVED|TRADING|SOLD|REVIEWED|DELETED — 판매글 공개 상태"
        int like_count "비정규화 캐싱"
        datetime created_at
        datetime updated_at
        boolean is_deleted "소프트 삭제"
    }

    PRODUCT_IMAGE {
        bigint id PK
        bigint product_id FK
        varchar image_url
        int sort_order "정렬 순서"
    }

    TRADE {
        bigint id PK
        bigint product_id FK "PRODUCT.id — UK 없음. 취소 후 재거래 허용"
        bigint buyer_id FK "MEMBER.id"
        bigint seller_id FK "MEMBER.id"
        enum status "RESERVED|TRADING|SOLD|REVIEWED|CANCELLED — 거래 진행 상태"
        datetime reserved_at
        datetime traded_at
        datetime sold_at
        datetime created_at
    }

    REVIEW {
        bigint id PK
        bigint trade_id FK UK "거래당 1건"
        bigint reviewer_id FK "구매자"
        bigint reviewee_id FK "판매자"
        int rating "1~5"
        text content
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
        bigint product_id FK "어떤 판매글에 대한 채팅인지"
        bigint buyer_id FK "채팅 시작한 구매자"
        bigint seller_id FK "판매자"
        datetime created_at "채팅방 생성 = 구매자가 상품 상세에서 채팅하기 클릭 시"
        datetime last_message_at
    }

    CHAT_MESSAGE {
        bigint id PK
        bigint chat_room_id FK
        bigint sender_id FK
        text content
        boolean is_read
        datetime created_at
    }

    KEYWORD_LOG {
        bigint id PK
        varchar keyword
        int count "검색 횟수"
        date log_date "일별 집계"
    }

    MEMBER ||--o{ PRODUCT : "판매자"
    MEMBER ||--o{ TRADE : "구매자"
    MEMBER ||--o{ LIKE : "찜"
    MEMBER ||--o{ REVIEW : "작성자(reviewer)"
    MEMBER ||--o{ REVIEW : "대상자(reviewee)"
    MEMBER ||--o{ CHAT_ROOM : "구매자"
    MEMBER ||--o{ CHAT_ROOM : "판매자"
    MEMBER ||--o{ CHAT_MESSAGE : "발신자"

    CATEGORY ||--o{ CATEGORY : "parent(대분류)"
    CATEGORY ||--o{ PRODUCT : "카테고리"

    PRODUCT ||--o{ PRODUCT_IMAGE : "이미지"
    PRODUCT ||--o{ TRADE : "거래 (취소 후 재거래 허용 → 다건 가능)"
    PRODUCT ||--o{ LIKE : "찜"
    PRODUCT ||--o{ CHAT_ROOM : "채팅"

    TRADE ||--o| REVIEW : "리뷰"

    CHAT_ROOM ||--o{ CHAT_MESSAGE : "메시지"
```

---

## 2. 설계 결정 사항

### 2-0. PRODUCT = 중고 판매글(Listing) 개념 명확화

PRODUCT는 상품 카탈로그(e-커머스의 SKU)가 아니다.
**판매자가 "이 물건을 팝니다"라고 올린 게시글 1건 = PRODUCT 1건**이다.

- 같은 모델의 아이폰을 2대 팔려면 → PRODUCT 2건 등록
- PRODUCT가 삭제(DELETED)되면 해당 판매글이 비공개 처리됨
- PRODUCT.status는 "판매글의 공개/거래 상태", TRADE.status는 "실제 거래 진행 상태"로 역할이 다름

---

### 2-1. PRODUCT.status vs TRADE.status 역할 분리

| 상태 컬럼 | 역할 | 값 |
|-----------|------|-----|
| `PRODUCT.status` | 판매글 공개 상태 — 목록 노출 여부 제어 | SALE / RESERVED / TRADING / SOLD / REVIEWED / DELETED |
| `TRADE.status` | 거래 진행 상태 — 거래 이력 관리 | RESERVED / TRADING / SOLD / REVIEWED / CANCELLED |

**왜 두 곳에 상태가 있는가?**
- PRODUCT.status가 없으면 목록 조회 시 매번 TRADE JOIN 필요 → 성능 저하
- TRADE.status가 없으면 취소 이력, 리뷰 연결 기준점 없음
- 두 상태는 항상 동기화되어야 하며, `TradeService` 내에서 한 트랜잭션으로 함께 변경

---

### 2-2. TRADE.product_id — UK 제거 이유

**이전 설계의 버그**: `product_id FK UK`로 설정하면 취소 후 재거래 불가능.

**현재 설계**:
- TRADE.product_id에 UK 없음 → 1개 상품에 여러 TRADE 레코드 가능
- 활성 거래 유일성은 **서비스 레이어**에서 보장:
```java
// 예약 시도 시 활성 거래 존재 여부 확인
boolean hasActiveTrade = tradeRepository
    .existsByProductIdAndStatusIn(productId,
        List.of(TradeStatus.RESERVED, TradeStatus.TRADING));
if (hasActiveTrade) throw new ConflictException("ALREADY_RESERVED", ...);
```
- DB 최후 방어선: `(product_id, status)` 복합 인덱스로 활성 거래 조회 최적화

---

### 2-3. 채팅방 생성 정책

**정책: 예약 전 자유 채팅 허용**

- 구매자가 상품 상세에서 [채팅하기] 클릭 → 채팅방 생성 (`POST /chat-rooms`)
- 같은 (buyer_id, product_id) 조합의 채팅방이 이미 있으면 기존 채팅방으로 진입 (중복 생성 방지)
- 예약은 채팅방 내 [예약하기] 버튼 또는 상품 상세에서 직접 가능

**왜 예약 전 채팅을 허용하는가?**
실제 중고거래에서 "아직 판매 중인가요?", "직거래 가능한가요?" 확인 없이 예약 버튼부터 누르는 사람은 없다. 채팅을 예약 이후로 제한하면 UX가 현실과 맞지 않는다.

---

### 2-4. TRADE 테이블 분리 이유

상품(PRODUCT)과 거래(TRADE)를 분리한 이유:
- 취소 후 재판매: PRODUCT.status = SALE로 되돌리고 새 TRADE 생성 가능
- 거래 이력 보존: 분쟁 대비, 리뷰 기준점
- 리뷰가 trade_id 기준 → "이 거래의 구매자만 리뷰 작성 가능" 검증

---

### 2-5. LIKE.like_count 비정규화

- PRODUCT 테이블에 `like_count` 컬럼을 두어 목록 조회 시 JOIN 없이 찜 수 표시
- 정합성 허용 오차 ±1 수준 (실시간 정확도 불필요)
- 증가: `UPDATE product SET like_count = like_count + 1 WHERE id = ?`

---

### 2-6. CATEGORY 자기 참조 (2뎁스)

- `parent_id IS NULL` → 대분류
- `parent_id IS NOT NULL` → 소분류
- PRODUCT는 소분류 ID만 저장 (depth=1)
- 대분류 조회 필요 시 JOIN 또는 `parent_id`를 통해 역추적

---

### 2-7. CHAT_MESSAGE 저장소

- Redis Pub/Sub이 아닌 MySQL 영속 저장
- 이유: 재연결 시 이전 메시지 조회, 거래 증거 보존
- 인덱스: `(chat_room_id, id DESC)` — 최신 메시지 빠른 조회

---

### 2-8. KEYWORD_LOG 설계

- 검색마다 실시간 INSERT 대신 `count` 컬럼 UPDATE 또는 일배치 집계
- `log_date` 컬럼으로 최근 7일 집계 → 인기 검색어 산출

---

## 3. 인덱스 계획 (EXPLAIN 기반 최적화 대상)

| 테이블 | 인덱스 | 이유 |
|--------|--------|------|
| PRODUCT | `(status, created_at DESC)` | 목록 조회 기본 정렬 |
| PRODUCT | `(category_id, status)` | 카테고리 필터 검색 |
| PRODUCT | `(seller_id)` | 내 판매 목록 |
| PRODUCT | `FULLTEXT(title)` | 키워드 검색 (LIKE → FULLTEXT ngram) |
| TRADE | `(product_id, status)` | 활성 거래 존재 여부 확인 (UK 대체) |
| TRADE | `(buyer_id)` | 구매 이력 조회 |
| LIKE | `UNIQUE(member_id, product_id)` | 중복 방지 + 찜 여부 조회 |
| CHAT_ROOM | `UNIQUE(buyer_id, product_id)` | 동일 채팅방 중복 생성 방지 |
| CHAT_MESSAGE | `(chat_room_id, id DESC)` | 채팅 이력 최신순 |
| KEYWORD_LOG | `(keyword, log_date)` | 인기 검색어 집계 |
