# 10. ADR (Architecture Decision Records)

> **형식**: 결정마다 1개
> **버전**: v1.2 (ADR-001 블랙리스트·스케일아웃 전략 보완, ADR-002 즉시 RESERVED 정합성 명시)

---

## ADR-001. JWT AccessToken 단독 인증 채택

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-04-09 |
| **상태** | 승인됨 |
| **결정자** | 설계팀 전원 |

### 컨텍스트

회원 인증 방식을 결정해야 한다. 주요 옵션은 세션 기반, JWT AccessToken+RefreshToken, JWT AccessToken 단독이다.

### 비교

| 항목 | 세션 기반 (Spring Session + Redis) | JWT (AccessToken + RefreshToken) | JWT (AccessToken 단독) ✅ |
|------|-----------------------------------|----------------------------------|--------------------------|
| 서버 상태 | Stateful | Stateless | Stateless |
| 스케일아웃 | Redis 세션 공유 필요 | 추가 인프라 불필요 | 추가 인프라 불필요 |
| 구현 복잡도 | 낮음 | 높음 (Redis 관리 + 재발급 API) | 낮음 |
| 토큰 즉시 무효화 | 가능 (Redis 삭제) | 가능 (Redis RefreshToken 삭제) | 불가 (만료까지 대기) |
| 모바일 친화성 | 쿠키 의존 | 헤더 기반으로 유리 | 헤더 기반으로 유리 |
| 업계 표준 | 레거시 방향 | 현재 표준 | 현재 표준 |
| 3주 일정 적합성 | ✅ | ❌ (Redis TTL 동기화·탈취 대응 복잡) | ✅ |

### 결정

**JWT (AccessToken 15분 단독)** 채택

### 이유

1. 프론트엔드가 바이브코딩(웹)이나 향후 모바일 확장을 고려하면 쿠키보다 헤더 기반이 유리
2. **RefreshToken 미채택**: 로그아웃 시 Redis 삭제 무효화, 탈취 감지 시 블랙리스트 등록, Redis TTL 동기화 등 관리 복잡도가 3주 개발 일정 범위를 초과한다고 판단
3. AccessToken 만료시간을 15분으로 짧게 유지하여 보안 위협 범위를 최소화
4. 단일 EC2 기준이지만 Stateless 구조로 스케일아웃 전환 비용이 낮음

### 트레이드오프 — AccessToken 즉시 무효화 문제

AccessToken 만료 전 강제 무효화가 불가능하다. 로그아웃 시 클라이언트가 토큰을 폐기하지만, 탈취된 토큰은 만료(15분)까지 유효하다.

**이 트레이드오프를 수용하는 이유**:
```
블랙리스트 방식을 도입한다면:
  - 로그아웃 → Redis에 "blocked:{accessToken}" SET EX 900
  - 모든 API 요청 → Redis 블랙리스트 조회 추가
  → 매 요청마다 Redis I/O 발생 → 성능 저하
  → Stateless JWT의 핵심 장점 상실

∴ v1 선택: 만료시간 15분으로 위협 노출 구간 최소화
  로그아웃 = 클라이언트 토큰 폐기 (서버 처리 없음)
```

### 스케일아웃 전환 전략

| 단계 | 현재 (단일 EC2) | 스케일아웃 시 |
|------|----------------|--------------|
| 인증 | JWT AccessToken (변경 없음) | JWT AccessToken (변경 없음) — Stateless이므로 그대로 |
| 세션 공유 | 불필요 | 불필요 (JWT 덕분) |

JWT의 가장 큰 스케일아웃 이점: 서버 인스턴스가 늘어나도 인증 로직 변경 없음.

### v2 향후 계획

RefreshToken은 아래 조건이 갖춰지면 v2에서 도입한다.

| 항목 | v2 구현 내용 |
|------|------------|
| RefreshToken 발급 | UUID 생성, `refresh:{token}` → `memberId` Redis 저장 (TTL 7일) |
| 토큰 재발급 API | `POST /auth/reissue` — RefreshToken 검증 후 AccessToken 재발급 |
| 로그아웃 처리 | Redis에서 RefreshToken 삭제 → 즉시 무효화 |
| 탈취 대응 | 재사용 감지 시 해당 토큰 블랙리스트 등록 검토 |

> **v2 영향 문서**: 04-기능명세서, 06-API명세서, 03-유스케이스(UC03·로그아웃), 11-패키지구조(AuthService)

---

## ADR-002. 비관적 락 + Redisson 이중 방어 채택

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-04-09 |
| **상태** | 승인됨 |

### 컨텍스트

상품 예약 시 동시에 여러 사용자가 요청해도 1명만 성공해야 한다.
**예약 모델: 구매자 클릭 즉시 RESERVED (판매자 승인 없음)** — 이 결정이 동시성 제어 필요성을 높인다.
판매자 승인 단계가 있었다면 동시성 이슈가 줄어들지만, 즉시 RESERVED이므로 락은 필수다.

### 비교

| 전략 | 정합성 | 성능 | 구현 난이도 | 분산 환경 |
|------|--------|------|------------|-----------|
| 낙관적 락 | 재시도 시 보장 | 충돌 적을 때 우수 | 중간 | 부적합 |
| 비관적 락 | 확실 | 락 경합 시 지연 | 낮음 | 단일 DB 의존 |
| Redisson | 확실 | DB 접근 전 차단 | 중간 | 대응 가능 |

### 결정

**비관적 락(필수) + Redisson(도전)** 순차 적용

### 이유

1. 비관적 락은 구현이 단순하고 정합성이 확실하여 필수 기능으로 먼저 구현
2. Redisson은 Redis 락으로 DB 도달 전에 차단하여 DB 부하 감소
3. Redis 장애 시 비관적 락이 fallback으로 동작하는 이중 방어 구조
4. `tryLock(waitTime=0)`으로 사용자에게 즉시 실패 응답 → UX 개선

### 스케일아웃 전환 전략

| 단계 | 현재 (단일 EC2) | 스케일아웃 시 |
|------|----------------|--------------|
| 비관적 락 | ✅ 동작 | ✅ 동작 (DB가 단일이면 동일) |
| Redisson | ✅ 동작 | ✅ 동작 — Redis 연결만 Cluster로 전환 |
| 문제 발생 시 | 없음 | DB가 다중화되면 비관적 락 효과 감소 → Redisson 필수화 |

---

## ADR-003. Caffeine 로컬 캐시 (인기 검색어)

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-04-09 |
| **상태** | 승인됨 |

### 컨텍스트

인기 검색어는 매 요청마다 DB 집계 쿼리를 실행하면 부하가 크다. 캐싱이 필요하다.

### 비교

| 항목 | Caffeine (로컬) | Redis (분산) |
|------|----------------|-------------|
| 속도 | 나노초 (JVM 메모리) | 마이크로초 (네트워크) |
| 일관성 | 인스턴스별 독립 | 모든 인스턴스 공유 |
| 설정 복잡도 | 낮음 | 중간 |
| 인프라 의존 | 없음 | Redis 필요 |
| 적합 케이스 | 단일 서버, 일관성 허용 | 다중 서버, 강한 일관성 |

### 결정

**Caffeine 로컬 캐시** 채택

### 이유

1. 단일 EC2 배포 환경 → 인스턴스 간 불일치 문제 없음
2. 인기 검색어는 TTL 10분 허용 오차 가능 → 강한 일관성 불필요
3. Redis는 채팅·세션·분산락에 이미 사용 → 단순한 인기 검색어에 Redis 부하 추가 불필요
4. 구현이 단순하여 3개월 경력 팀에게 적합

### 트레이드오프

향후 다중 서버로 스케일아웃 시 각 서버마다 다른 인기 검색어를 반환할 수 있다. 이 경우 Redis로 교체 또는 Spring Cache 추상화로 전환 필요.

---

## ADR-004. Cursor 기반 페이지네이션 채택

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-04-09 |
| **상태** | 승인됨 |

### 컨텍스트

상품 목록은 무한 스크롤 UI를 사용한다. 페이지네이션 방식을 결정해야 한다.

### 비교

| 항목 | Offset 기반 (LIMIT/OFFSET) | Cursor 기반 (createdAt + id) |
|------|---------------------------|------------------------------|
| 구현 난이도 | 낮음 | 중간 |
| 성능 (대량 데이터) | OFFSET 커질수록 느려짐 | 일정 |
| 데이터 중복/누락 | 발생 가능 (데이터 삽입/삭제 시) | 없음 |
| 무한 스크롤 적합성 | 부적합 | ✅ 적합 |
| 페이지 직접 이동 | 가능 | 불가능 |

### 결정

**Cursor 기반 페이지네이션** 채택

### 이유

1. 중고거래 특성상 상품이 자주 등록/삭제됨 → Offset 방식은 중복/누락 발생
2. 무한 스크롤 UI → 특정 페이지 직접 이동 불필요
3. 대량 데이터에서도 일정한 성능 유지

---

## ADR-005. 채팅 메시지 MySQL 영속 저장

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-04-09 |
| **상태** | 승인됨 |

### 컨텍스트

채팅 메시지를 어디에 저장할지 결정해야 한다.

### 비교

| 항목 | Redis Pub/Sub | MySQL 영속 저장 |
|------|---------------|----------------|
| 실시간 전송 | ✅ 우수 | ❌ (WebSocket별도) |
| 메시지 영속성 | ❌ (휘발성) | ✅ |
| 이전 메시지 조회 | ❌ 불가 | ✅ 가능 |
| 거래 증거 보존 | ❌ | ✅ |
| 구현 복잡도 | 중간 | 낮음 |

### 결정

**실시간 전송: WebSocket/STOMP + 영속 저장: MySQL** 채택

### 이유

1. 중고거래에서 채팅은 거래 증거 → 영속성 필수
2. 연결 끊김 후 재접속 시 이전 메시지 조회 필요
3. Redis Pub/Sub은 메시지 유실 위험 (구독자 없을 때 발행 시 소실)
4. Redis는 이미 분산락·세션에 활용 중이며 채팅 저장소로 추가 부담 불필요

---

## ADR-006. QueryDSL 동적 쿼리 채택

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-04-09 |
| **상태** | 승인됨 |

### 컨텍스트

상품 검색 시 키워드, 카테고리, 상태 등 조건이 선택적으로 조합된다. 동적 쿼리 생성 방법을 결정해야 한다.

### 비교

| 항목 | JPQL + @Query | Specification | QueryDSL |
|------|---------------|---------------|---------|
| 동적 조건 처리 | 어려움 (수동 문자열 조합) | 가능 | ✅ 우수 |
| 타입 안전성 | 없음 | 있음 | ✅ 있음 |
| 가독성 | 낮음 | 중간 | ✅ 높음 |
| 컴파일 타임 오류 검출 | ❌ | 부분 | ✅ |
| 초기 설정 비용 | 없음 | 낮음 | 중간 (QClass 생성) |

### 결정

**QueryDSL** 채택 (기본 CRUD는 Spring Data JPA)

### 이유

1. 검색 조건이 keyword, categoryId, status 등 선택적 조합 → `BooleanBuilder` 패턴으로 깔끔하게 처리
2. 타입 안전성으로 오타·컬럼명 오류를 컴파일 타임에 검출
3. 실무에서 가장 널리 쓰이는 동적 쿼리 솔루션 → 학습 가치 높음

---

## ADR-007. TRADE 테이블 분리

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-04-09 |
| **상태** | 승인됨 |

### 컨텍스트

거래 정보를 PRODUCT 테이블에 컬럼으로 둘지, 별도 TRADE 테이블로 분리할지 결정해야 한다.

### 비교

| 항목 | PRODUCT 컬럼 확장 | TRADE 테이블 분리 |
|------|------------------|------------------|
| 구현 단순성 | 단순 | 조인 필요 |
| 거래 이력 보존 | 1건만 저장 가능 | ✅ 이력 보존 |
| 재판매 지원 | 어려움 | ✅ 취소 후 재판매 가능 |
| 리뷰 연결 | 복잡 | ✅ trade_id 기준 |
| 데이터 정규화 | 위반 | ✅ 정규화 |

### 결정

**TRADE 테이블 분리** 채택

### 이유

1. 예약 취소 후 재판매가 가능해야 함 → PRODUCT.status만 SALE로 되돌리고 새 TRADE 생성
2. 거래 이력 보존 → 분쟁 대비, 리뷰 연결의 기준점
3. 리뷰가 trade_id 기준으로 연결되어야 "이 거래의 구매자만 리뷰 작성 가능" 검증 가능

---

## ADR-008. Redis Cache-Aside 패턴 채택 (도전 기능)

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-04-09 |
| **상태** | 승인됨 |
| **대상** | 상품 상세 조회 캐싱 |

### 컨텍스트

상품 상세 페이지는 동일한 productId에 대해 반복 조회가 많다. DB 부하를 줄이기 위해 Redis 캐싱을 도입한다. 캐싱 전략을 결정해야 한다.

### 비교

| 전략 | 설명 | 장점 | 단점 |
|------|------|------|------|
| **Cache-Aside** | 앱이 캐시 미스 시 DB 조회 후 직접 캐시에 PUT | 필요한 데이터만 캐시 | 캐시 미스 첫 요청 느림 |
| **Write-Through** | 쓰기 시 DB + 캐시 동시 갱신 | 캐시 항상 최신 | 모든 데이터가 캐시됨 (메모리 낭비) |
| **Write-Behind** | 쓰기를 캐시에 먼저, DB는 비동기 | 쓰기 성능 최고 | 데이터 유실 위험, 구현 복잡 |

### 결정

**Cache-Aside (Lazy Loading)** 패턴 채택

### 이유

1. 중고거래 특성상 전체 상품 중 일부만 집중 조회됨 → 필요한 데이터만 캐시하는 Cache-Aside가 메모리 효율적
2. Write-Through는 모든 상품 등록/수정 시 캐시를 항상 갱신해야 해서 불필요한 캐시 엔트리가 쌓임
3. Write-Behind는 데이터 유실 위험이 있어 거래 데이터에 부적합

### 구현 패턴

```java
// ProductService.java
public ProductDetailResponse getProduct(Long productId) {
    String cacheKey = "product:" + productId;

    // 1. 캐시 조회
    String cached = redisTemplate.opsForValue().get(cacheKey);
    if (cached != null) {
        return objectMapper.readValue(cached, ProductDetailResponse.class); // CACHE HIT
    }

    // 2. DB 조회 (CACHE MISS)
    Product product = productRepository.findById(productId)
        .orElseThrow(() -> new NotFoundException("상품을 찾을 수 없습니다."));
    ProductDetailResponse response = ProductDetailResponse.from(product);

    // 3. 캐시 저장 (TTL 10분)
    redisTemplate.opsForValue().set(cacheKey, objectMapper.writeValueAsString(response),
        Duration.ofMinutes(10));

    return response; // CACHE MISS → DB 조회 후 캐시 저장
}

// 상품 수정/삭제 시 캐시 무효화
public void updateProduct(Long productId, UpdateProductRequest request) {
    // ... 상품 수정 로직 ...
    redisTemplate.delete("product:" + productId); // Cache Eviction
}
```

### TTL 정책

| 캐시 대상 | TTL | 이유 |
|-----------|-----|------|
| 상품 상세 | 10분 | 상태 변경(예약/판매완료)이 빈번하지 않음, 수정 시 명시적 eviction |
| 상품 목록 | 미적용 | 목록은 변화가 잦고 Cursor 조합이 무한해 캐싱 비효율 |

### 트레이드오프

캐시와 DB 사이 최대 10분의 불일치 가능성이 있다. 상품 수정·삭제·상태 변경 시 즉시 `redisTemplate.delete(key)`로 eviction하여 불일치 구간을 최소화한다.

---

## ADR-009. EXPLAIN 기반 인덱스 최적화 전략

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-04-09 |
| **상태** | 승인됨 |
| **대상** | MySQL 8 인덱스 설계 |

### 컨텍스트

처음부터 모든 컬럼에 인덱스를 걸면 되는 것 아닌가? 라는 질문에 답해야 한다. 인덱스 설계 전략을 결정해야 한다.

### 왜 처음부터 모든 인덱스를 걸지 않는가?

| 항목 | 설명 |
|------|------|
| 쓰기 성능 저하 | 인덱스가 많을수록 INSERT/UPDATE/DELETE 시 B-Tree 갱신 비용 증가 |
| 불필요한 인덱스 | 실제 쿼리 패턴과 다른 인덱스는 옵티마이저 혼란, 메모리 낭비 |
| 복합 인덱스 순서 | 컬럼 순서가 틀리면 인덱스를 전혀 타지 않음 |

### 결정

**실제 쿼리 작성 후 EXPLAIN으로 실행 계획 확인 → 필요한 인덱스만 추가** 전략 채택

### EXPLAIN 분석 기준

```sql
-- 예시: 상품 검색 쿼리
EXPLAIN SELECT * FROM product
WHERE status = 'SALE' AND category_id = 12
ORDER BY created_at DESC
LIMIT 20;
```

| EXPLAIN 컬럼 | 확인 기준 | 나쁜 신호 |
|-------------|-----------|-----------|
| `type` | `ref`, `range`, `index` | `ALL` (풀 테이블 스캔) |
| `key` | 인덱스명 표시 | `NULL` (인덱스 미사용) |
| `rows` | 적을수록 좋음 | 전체 행 수에 근접 |
| `Extra` | `Using index` | `Using filesort`, `Using temporary` |

### 우선 최적화 대상 쿼리 목록

| 쿼리 | 현재 인덱스 계획 | EXPLAIN 목표 |
|------|----------------|-------------|
| 상품 목록 (status + created_at) | `(status, created_at DESC)` | type=range, Using index |
| 카테고리 검색 (category_id + status) | `(category_id, status)` | type=ref |
| 제목 키워드 검색 | `FULLTEXT(title)` | type=fulltext (LIKE 대비 성능 대폭 향상) |
| 채팅 이력 최신순 | `(chat_room_id, id DESC)` | type=ref, Using index |

### FULLTEXT vs LIKE 비교

| 항목 | `LIKE '%키워드%'` | `FULLTEXT` |
|------|-----------------|------------|
| 인덱스 사용 | ❌ 앞 와일드카드 시 풀스캔 | ✅ 역색인 사용 |
| 성능 | 데이터 많을수록 급격히 저하 | 일정 수준 유지 |
| 설정 | 없음 | `FULLTEXT INDEX` 생성 필요 |
| 최소 토큰 길이 | 무관 | `innodb_ft_min_token_size=2` 설정 필요 (한글 2자) |

```sql
-- FULLTEXT 인덱스 생성
ALTER TABLE product ADD FULLTEXT INDEX ft_title (title) WITH PARSER ngram;

-- FULLTEXT 쿼리
SELECT * FROM product WHERE MATCH(title) AGAINST('침대 프레임' IN BOOLEAN MODE);
```

### 트레이드오프

FULLTEXT는 초기 설정(ngram parser, min_token_size)이 필요하다. 3주 일정 내 구현이 어려우면 `LIKE '키워드%'` (앞 와일드카드 제거)로 시작하고 FULLTEXT는 QA 주에 적용한다.

---

## ADR-010. WebSocket + STOMP 채택 (vs SSE, Long Polling)

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-04-09 |
| **상태** | 승인됨 |
| **대상** | 1:1 실시간 채팅 |

### 컨텍스트

구매자-판매자 간 실시간 채팅을 구현해야 한다. 실시간 통신 방식을 결정해야 한다.

### 비교

| 항목 | Long Polling | SSE | WebSocket + STOMP |
|------|-------------|-----|-------------------|
| 방향 | 단방향 (서버→클라이언트) | 단방향 (서버→클라이언트) | ✅ 양방향 |
| 연결 방식 | HTTP 반복 요청 | HTTP 지속 연결 | TCP 지속 연결 |
| 메시지 전송 | 클라이언트→서버: 별도 HTTP POST | 클라이언트→서버: 별도 HTTP POST | ✅ 단일 연결로 송수신 |
| 서버 부하 | 높음 (반복 HTTP) | 낮음 | 낮음 |
| 구현 복잡도 | 낮음 | 낮음 | 중간 |
| 채팅 적합성 | 부적합 | 부적합 (단방향) | ✅ 적합 |
| Spring 지원 | ✅ | ✅ | ✅ `spring-websocket` |

### 결정

**WebSocket + STOMP** 채택

### WebSocket 인증 정책

- JWT는 **STOMP CONNECT 시점에만 1회 검증**한다.
- 연결 후 AccessToken 만료만으로 기존 WebSocket 세션을 강제 종료하지 않는다.
- 이후 메시지 수신 시에는 토큰 재검증 대신 **chatRoom 참여자 여부**만 확인한다.
- 이유: 3주 일정 내 구현 복잡도를 낮추고, AccessToken 15분 단독 정책과 충돌하지 않는 최소 구현을 유지하기 위함이다.

### 이유

1. 채팅은 본질적으로 **양방향** 통신 — SSE는 서버→클라이언트 단방향만 지원하므로 메시지 전송에 별도 HTTP POST가 필요해 구조가 복잡해짐
2. Long Polling은 메시지마다 새 HTTP 연결을 맺어 불필요한 오버헤드 발생
3. STOMP 프로토콜은 발행/구독(pub/sub) 패턴을 추상화하여 채팅방 라우팅 구현이 간결함
4. `spring-websocket` + `spring-messaging` 의존성만으로 구현 가능

### STOMP 선택 이유 (순수 WebSocket 대비)

```
순수 WebSocket:
  - 텍스트 프레임을 직접 처리해야 함
  - 채팅방 라우팅, 브로드캐스트 로직을 직접 구현

STOMP over WebSocket:
  - SUBSCRIBE /topic/room.{id} → 자동 라우팅
  - SEND /app/chat.sendMessage → @MessageMapping으로 수신
  - SimpleBroker가 구독자 관리 자동화
```

### 트레이드오프

WebSocket은 HTTP/2 멀티플렉싱과 달리 연결당 하나의 소켓을 유지한다. 동시 접속자가 수천 명이 되면 서버 소켓 리소스가 문제될 수 있다. 현재 규모(수십~수백 명)에서는 문제없으며, 스케일 필요 시 STOMP External Broker(RabbitMQ/Redis)로 전환한다.

---

## ADR-011. GitHub Actions + ECR + EC2 SSH 배포 자동화

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-04-09 |
| **상태** | 승인됨 |
| **대상** | CI/CD 파이프라인 |

### 컨텍스트

코드 변경 → 프로덕션 반영까지의 배포 파이프라인을 자동화해야 한다. AWS 배포 방식을 결정해야 한다.

### 비교

| 항목 | GitHub Actions + SSH | AWS CodeDeploy | ECS (Fargate) |
|------|---------------------|---------------|---------------|
| 설정 복잡도 | 낮음 | 중간 | 높음 |
| 비용 | Actions 무료(공개) / EC2 비용만 | CodeDeploy 무료 + EC2 | Fargate 태스크 비용 추가 |
| Blue/Green 배포 | ❌ 수동 구성 필요 | ✅ 내장 | ✅ 내장 |
| 경력 3개월 팀 적합성 | ✅ 높음 | 중간 | 낮음 |
| 학습 목적 | ✅ 전체 흐름 이해에 적합 | 블랙박스 많음 | 블랙박스 많음 |

### 결정

**GitHub Actions + ECR + EC2 SSH 방식** 채택

### 이유

1. **학습 목적에 최적**: 빌드 → 이미지 빌드 → ECR push → EC2 pull → 컨테이너 재시작의 전 흐름을 직접 작성하므로 CI/CD 전체를 이해할 수 있음
2. **추가 서비스 비용 없음**: CodeDeploy agent 설정, ECS 추가 비용 불필요
3. **3주 일정에 적합**: GitHub Actions YAML은 공식 문서와 예시가 풍부해 빠르게 구현 가능

### 왜 Blue/Green을 적용하지 않는가?

```
Blue/Green 미적용 이유:
  - 단일 EC2 인스턴스 환경에서 Blue/Green은 서버 2대 필요
  - 3주 프로젝트에서 다운타임 1~2분은 허용 범위
  - docker-compose down → up 방식으로 재시작 시간 최소화

향후 확장 시:
  - ALB(Application Load Balancer) + Auto Scaling Group + CodeDeploy로 전환
  - 또는 ECS Fargate + ECR + CodePipeline으로 전환
```

### 배포 단계별 Secrets 관리

| Secret 키 | 용도 | 저장 위치 |
|-----------|------|-----------|
| `AWS_ACCESS_KEY_ID` | ECR 로그인, EC2 접근 | GitHub Secrets |
| `AWS_SECRET_ACCESS_KEY` | 위와 동일 | GitHub Secrets |
| `EC2_HOST` | SSH 접속 IP | GitHub Secrets |
| `EC2_SSH_KEY` | PEM 키 내용 | GitHub Secrets |
| `DB_PASSWORD` | RDS 비밀번호 | EC2 `.env` 파일 또는 AWS Secrets Manager |
| `JWT_SECRET` | JWT 서명 키 | EC2 `.env` 파일 또는 AWS Secrets Manager |

> **보안 원칙**: DB 비밀번호, JWT 시크릿은 GitHub Secrets에 저장하지 않는다. EC2 인스턴스의 `.env` 파일에만 저장하고 GitHub Actions는 SSH로 `docker-compose up`만 실행한다.

### 트레이드오프

SSH 방식은 EC2가 재기동되면 IP가 바뀔 수 있다. 반드시 **Elastic IP**를 EC2에 연결하여 고정 IP를 사용해야 한다. 또한 SSH 포트(22)는 GitHub Actions IP 대역이 아닌 개발자 고정 IP만 허용하는 것이 원칙이나, GitHub Actions의 IP는 동적이므로 `appleboy/ssh-action`을 쓸 때는 일시적으로 22포트를 열어두거나 AWS Systems Manager Session Manager로 대체하는 것을 권장한다.
