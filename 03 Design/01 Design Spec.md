아래 설계는 첨부된 **Design Agent Prompt**의 산출물 구조에 맞춰 진행합니다. 해당 Prompt는 MSA 설계 시 **서비스 분해, API 설계, DB 소유권, 서비스 간 통신, 인증/인가, 장애 대응, 확장성, 관측성**을 포함하도록 요구하며, 특히 **Database per Service**, **서비스 간 직접 DB 접근 금지**, **동기/비동기 통신 구분**, **API Gateway 명시**, **과도한 서비스 분리 방지**를 원칙으로 둡니다. 
또한 요구사항 분석 결과는 영화 메타데이터와 사용자 개인 데이터를 분리하고, 각 서비스가 자기 데이터만 소유하며, 서비스 간 연결은 API 또는 이벤트로 제한하는 방향을 제시하고 있습니다. 

# 영화 컬렉션 관리 시스템 MSA Design

## 1. 전체 아키텍처 개요

“영화 컬렉션 관리 시스템”은 사용자가 영화 정보를 검색하고, 자신의 컬렉션에 영화를 추가하며, 시청 상태, 평점, 리뷰, 소장 위치를 관리하는 시스템입니다.

MSA 설계의 핵심 기준은 다음입니다.

```text
서비스 분리 기준 = 데이터 소유권 + 변경 이유 + 비즈니스 책임
```

따라서 영화 자체의 공통 정보와 사용자 개인의 컬렉션/평점/시청 상태/소장 정보는 분리합니다. 영화 메타데이터는 여러 사용자가 공유하는 기준 데이터이고, 컬렉션·평점·시청 상태·소장 위치는 사용자별 개인 데이터이기 때문입니다.

전체 구조는 다음과 같습니다.

```text
[Client / Web / Mobile]
          |
          v
[API Gateway]
          |
          +--------------------+
          |                    |
          v                    v
[Auth/User Service]      [Query Aggregation Service]
          |                    |
          |                    +---- read/API/event read model
          |
          +------------------------------------------------+
          |                |                 |             |
          v                v                 v             v
[Movie Catalog]   [Collection]     [Watch Status]   [Rating Review]
          |                |                 |             |
          |                +-----------------+-------------+
          |                                  |
          v                                  v
      [Catalog DB]                    [각 서비스별 DB]

[Ownership Service] ---- [Ownership DB]

[Event Broker]
  - MovieCreated
  - MovieUpdated
  - CollectionItemAdded
  - WatchStatusChanged
  - RatingCreated
  - RatingUpdated
  - OwnershipChanged
```

## 2. Micro Service 목록

| 서비스                       | 핵심 책임                        | DB 소유          | 주요 의존성             |
| ------------------------- | ---------------------------- | -------------- | ------------------ |
| API Gateway               | 외부 요청 진입점, 인증 토큰 검증, 라우팅     | 없음             | Auth/User Service  |
| Auth/User Service         | 사용자, 인증, 권한 관리               | User DB        | 독립                 |
| Movie Catalog Service     | 영화, 장르, 배우, 감독 등 공통 메타데이터 관리 | Catalog DB     | 독립                 |
| Collection Service        | 사용자별 컬렉션 포함 여부와 메모 관리        | Collection DB  | movieId 참조         |
| Watch Status Service      | 사용자별 시청 상태 관리                | WatchStatus DB | userId, movieId 참조 |
| Rating Review Service     | 사용자 평점, 리뷰, 평균 평점 집계         | Rating DB      | userId, movieId 참조 |
| Ownership Service         | 소장 형태, 보관 위치, 플랫폼 관리         | Ownership DB   | userId, movieId 참조 |
| Query Aggregation Service | 화면 조회용 데이터 조합, 읽기 모델 제공      | 선택적 Query DB   | 여러 서비스 API 또는 이벤트  |

초기 MVP에서는 운영 복잡도를 줄이기 위해 `Watch Status`, `Rating Review`, `Ownership`을 하나의 `User Movie Activity Service`로 묶을 수도 있습니다. 다만 이번 설계에서는 책임 경계와 변경 독립성을 우선하므로 별도 서비스로 분리합니다.

## 3. 서비스별 책임

## 3.1 API Gateway

### 책임

* 모든 외부 요청의 단일 진입점
* JWT 검증
* 사용자 ID 추출 후 내부 서비스에 전달
* 서비스별 라우팅
* Rate Limit 적용
* 공통 에러 응답 포맷 적용

### 포함하지 않는 책임

* 비즈니스 로직
* DB 접근
* 영화/컬렉션/평점 데이터 처리

### 예시 라우팅

```text
/api/users/**        -> Auth/User Service
/api/movies/**       -> Movie Catalog Service
/api/collections/**  -> Collection Service
/api/watch-status/** -> Watch Status Service
/api/ratings/**      -> Rating Review Service
/api/ownerships/**   -> Ownership Service
/api/me/library      -> Query Aggregation Service
```

---

## 3.2 Auth/User Service

### 책임

* 회원가입
* 로그인
* 사용자 프로필 관리
* 사용자 역할 관리
* JWT 발급
* 관리자 권한 검증

### 책임 경계

이 서비스는 사용자 식별과 권한만 책임집니다. 다른 서비스는 User DB를 직접 조회하지 않고, JWT의 `userId`, `role` 클레임을 사용합니다.

### 소유 데이터

```text
User
- userId
- email
- passwordHash
- name
- role
- status
- createdAt
- updatedAt
```

---

## 3.3 Movie Catalog Service

### 책임

* 영화 메타데이터 등록, 수정, 삭제, 조회
* 장르 관리
* 배우/감독/출연진 관리
* 영화 검색
* 외부 영화 API 연동 가능성 수용

### 책임 경계

이 서비스는 “공통 영화 정보”만 책임집니다. 사용자별 컬렉션, 평점, 시청 상태, 소장 위치는 절대 포함하지 않습니다.

### 소유 데이터

```text
Movie
- movieId
- title
- originalTitle
- releaseYear
- description
- runtime
- posterUrl
- createdAt
- updatedAt

Genre
- genreId
- name

Person
- personId
- name
- birthDate
- profileImageUrl

MovieGenre
- movieId
- genreId

MovieCredit
- movieId
- personId
- roleType
- characterName
```

---

## 3.4 Collection Service

### 책임

* 사용자가 자신의 컬렉션에 영화를 추가
* 컬렉션에서 영화 제거
* 컬렉션 메모 관리
* 사용자별 컬렉션 포함 여부 관리

### 책임 경계

Collection Service는 영화 상세 정보를 소유하지 않습니다. `movieId`만 참조합니다.
영화 제목, 장르, 포스터가 필요하면 Movie Catalog Service API를 호출하거나 Query Aggregation Service의 읽기 모델을 사용합니다.

### 소유 데이터

```text
CollectionItem
- collectionItemId
- userId
- movieId
- memo
- addedAt
- updatedAt
```

### 제약

```text
unique(userId, movieId)
```

같은 사용자가 같은 영화를 중복으로 컬렉션에 추가하지 못하게 합니다.

---

## 3.5 Watch Status Service

### 책임

* 사용자별 영화 시청 상태 관리
* 보고 싶음, 보는 중, 시청 완료, 보류 상태 관리
* 시청 완료일 관리

### 책임 경계

Watch Status Service는 컬렉션 소유 여부를 강하게 검증하지 않습니다.
즉, “컬렉션에 있어야만 시청 상태를 등록할 수 있다”는 강한 의존을 만들지 않습니다.

권장 정책은 다음입니다.

```text
시청 상태는 userId + movieId 기준으로 독립 관리한다.
컬렉션 포함 여부는 화면 조회에서 조합한다.
```

### 소유 데이터

```text
WatchStatus
- watchStatusId
- userId
- movieId
- status
- watchedAt
- updatedAt
```

### 상태값

```text
WANT_TO_WATCH
WATCHING
WATCHED
ON_HOLD
DROPPED
```

---

## 3.6 Rating Review Service

### 책임

* 사용자별 평점 등록, 수정, 삭제
* 리뷰 작성, 수정, 삭제
* 영화별 평균 평점 집계
* 사용자별 리뷰 목록 조회

### 책임 경계

Rating Review Service는 영화 제목이나 장르를 소유하지 않습니다.
평점은 `userId`, `movieId` 기준으로만 관리합니다.

평균 평점은 실시간 계산보다 이벤트 기반 집계를 권장합니다.

### 소유 데이터

```text
RatingReview
- reviewId
- userId
- movieId
- rating
- reviewText
- visibility
- createdAt
- updatedAt

MovieRatingSummary
- movieId
- averageRating
- ratingCount
- updatedAt
```

---

## 3.7 Ownership Service

### 책임

* 사용자의 영화 소장 형태 관리
* Blu-ray, DVD, 파일, 스트리밍 구매, 대여 등 관리
* 보관 위치 또는 플랫폼 관리

### 책임 경계

Ownership Service는 Collection Service에 종속되지 않습니다.
소장 정보는 `userId + movieId` 기준으로 독립 관리합니다.

### 소유 데이터

```text
Ownership
- ownershipId
- userId
- movieId
- mediaType
- location
- platform
- purchaseDate
- updatedAt
```

### mediaType 예시

```text
BLURAY
DVD
DIGITAL_FILE
STREAMING_PURCHASE
RENTAL
THEATER
```

---

## 3.8 Query Aggregation Service

### 책임

* 프론트엔드 화면에 필요한 통합 조회 제공
* 내 컬렉션 목록 조회
* 컬렉션 + 영화 정보 + 시청 상태 + 평점 + 소장 위치 조합
* 조회 성능 최적화

### 책임 경계

Query Aggregation Service는 원천 데이터를 수정하지 않습니다.
쓰기 요청은 반드시 각 도메인 서비스로 전달됩니다.

### 선택 가능한 구현 방식

#### 방식 A: 실시간 API Composition

```text
Query Service
  -> Collection Service
  -> Movie Catalog Service
  -> Watch Status Service
  -> Rating Review Service
  -> Ownership Service
```

장점은 단순합니다.
단점은 호출이 많고 일부 서비스 장애에 취약합니다.

#### 방식 B: 이벤트 기반 Read Model

```text
각 서비스 이벤트 발행
        ↓
Query Service가 이벤트 구독
        ↓
query_movie_library_view 생성
```

장점은 조회 성능과 장애 격리가 좋습니다.
단점은 최종적 일관성을 감수해야 합니다.

이번 설계에서는 **초기에는 API Composition**, 사용량 증가 후 **이벤트 기반 Read Model**로 확장하는 방식을 권장합니다.

## 4. 서비스별 데이터 모델

## 4.1 User DB

```text
users
- user_id PK
- email UNIQUE
- password_hash
- name
- role
- status
- created_at
- updated_at
```

## 4.2 Catalog DB

```text
movies
- movie_id PK
- title
- original_title
- release_year
- description
- runtime
- poster_url
- created_at
- updated_at

genres
- genre_id PK
- name UNIQUE

persons
- person_id PK
- name
- profile_image_url

movie_genres
- movie_id
- genre_id

movie_credits
- movie_id
- person_id
- role_type
- character_name
```

## 4.3 Collection DB

```text
collection_items
- collection_item_id PK
- user_id
- movie_id
- memo
- added_at
- updated_at

unique(user_id, movie_id)
```

## 4.4 WatchStatus DB

```text
watch_statuses
- watch_status_id PK
- user_id
- movie_id
- status
- watched_at
- updated_at

unique(user_id, movie_id)
```

## 4.5 Rating DB

```text
rating_reviews
- review_id PK
- user_id
- movie_id
- rating
- review_text
- visibility
- created_at
- updated_at

unique(user_id, movie_id)

movie_rating_summaries
- movie_id PK
- average_rating
- rating_count
- updated_at
```

## 4.6 Ownership DB

```text
ownerships
- ownership_id PK
- user_id
- movie_id
- media_type
- location
- platform
- purchase_date
- updated_at

unique(user_id, movie_id, media_type, platform)
```

## 4.7 Query DB, 선택 사항

```text
user_movie_library_view
- user_id
- movie_id
- title
- release_year
- poster_url
- genres
- collection_added_at
- collection_memo
- watch_status
- watched_at
- rating
- review_summary
- ownership_type
- ownership_location
- last_synced_at
```

이 테이블은 원천 데이터가 아닙니다. 언제든 이벤트를 다시 재생해서 재구성할 수 있는 조회 전용 모델입니다.

## 5. API 설계

## 5.1 Auth/User Service API

```http
POST /auth/signup
POST /auth/login
GET  /users/me
PATCH /users/me
GET  /admin/users/{userId}
PATCH /admin/users/{userId}/role
```

### 로그인 응답 예시

```json
{
  "accessToken": "jwt-token",
  "refreshToken": "refresh-token",
  "user": {
    "userId": "u-001",
    "email": "user@example.com",
    "role": "USER"
  }
}
```

---

## 5.2 Movie Catalog Service API

```http
GET    /movies
GET    /movies/{movieId}
POST   /admin/movies
PATCH  /admin/movies/{movieId}
DELETE /admin/movies/{movieId}

GET    /genres
POST   /admin/genres

GET    /persons
GET    /persons/{personId}
POST   /admin/persons
```

### 영화 검색

```http
GET /movies?keyword=matrix&genre=Sci-Fi&actor=Keanu%20Reeves&page=0&size=20
```

---

## 5.3 Collection Service API

```http
GET    /collections/me
POST   /collections/me/items
PATCH  /collections/me/items/{collectionItemId}
DELETE /collections/me/items/{collectionItemId}
GET    /collections/me/items/by-movie/{movieId}
```

### 컬렉션 추가 요청

```json
{
  "movieId": "m-001",
  "memo": "4K 리마스터 버전으로 다시 보기"
}
```

---

## 5.4 Watch Status Service API

```http
GET   /watch-status/me
GET   /watch-status/me/{movieId}
PUT   /watch-status/me/{movieId}
DELETE /watch-status/me/{movieId}
```

### 시청 상태 변경 요청

```json
{
  "status": "WATCHED",
  "watchedAt": "2026-05-12"
}
```

---

## 5.5 Rating Review Service API

```http
GET    /ratings/movies/{movieId}/summary
GET    /ratings/movies/{movieId}/reviews
GET    /ratings/me
GET    /ratings/me/{movieId}
PUT    /ratings/me/{movieId}
DELETE /ratings/me/{movieId}
```

### 평점 등록 요청

```json
{
  "rating": 4.5,
  "reviewText": "다시 봐도 훌륭한 SF 영화",
  "visibility": "PRIVATE"
}
```

---

## 5.6 Ownership Service API

```http
GET    /ownerships/me
GET    /ownerships/me/{movieId}
POST   /ownerships/me/{movieId}
PATCH  /ownerships/me/{ownershipId}
DELETE /ownerships/me/{ownershipId}
```

### 소장 정보 등록 요청

```json
{
  "mediaType": "BLURAY",
  "location": "거실 선반 A-3",
  "platform": null,
  "purchaseDate": "2025-11-10"
}
```

---

## 5.7 Query Aggregation Service API

```http
GET /me/library
GET /me/library/{movieId}
GET /me/dashboard
```

### 내 컬렉션 통합 조회 응답 예시

```json
{
  "items": [
    {
      "movieId": "m-001",
      "title": "The Matrix",
      "releaseYear": 1999,
      "posterUrl": "https://example.com/matrix.jpg",
      "genres": ["Sci-Fi", "Action"],
      "collection": {
        "addedAt": "2026-05-12",
        "memo": "소장판"
      },
      "watchStatus": {
        "status": "WATCHED",
        "watchedAt": "2026-05-10"
      },
      "rating": {
        "score": 4.5
      },
      "ownership": {
        "mediaType": "BLURAY",
        "location": "거실 선반 A-3"
      }
    }
  ]
}
```

## 6. 서비스 간 통신 설계

## 6.1 동기 통신

동기 통신은 사용자의 요청 처리 흐름에서 즉시 결과가 필요한 경우에만 사용합니다.

| 호출 주체                     | 호출 대상                 | 목적                 |
| ------------------------- | --------------------- | ------------------ |
| API Gateway               | Auth/User Service     | 토큰 검증 또는 사용자 정보 확인 |
| Query Aggregation Service | Collection Service    | 내 컬렉션 목록 조회        |
| Query Aggregation Service | Movie Catalog Service | 영화 상세 정보 조회        |
| Query Aggregation Service | Watch Status Service  | 시청 상태 조회           |
| Query Aggregation Service | Rating Review Service | 평점 조회              |
| Query Aggregation Service | Ownership Service     | 소장 정보 조회           |

주의할 점은 Collection Service가 Movie Catalog Service를 매번 강하게 호출하지 않도록 하는 것입니다.
컬렉션 추가 시 movieId 존재 여부 검증은 선택적으로 수행하되, 장애 시 사용자 경험과 데이터 정합성 정책을 명확히 해야 합니다.

권장 정책은 다음입니다.

```text
쓰기 서비스 간 동기 의존은 최소화한다.
조회 조합은 Query Aggregation Service가 담당한다.
```

## 6.2 비동기 이벤트 통신

이벤트는 데이터 변경 사실을 다른 서비스에 알릴 때 사용합니다.

| 이벤트                   | 발행 서비스        | 구독 서비스                        | 목적               |
| --------------------- | ------------- | ----------------------------- | ---------------- |
| MovieCreated          | Movie Catalog | Query Aggregation             | 조회 모델 생성         |
| MovieUpdated          | Movie Catalog | Query Aggregation             | 영화 제목/포스터 변경 반영  |
| MovieDeleted          | Movie Catalog | Query Aggregation, Collection | 조회 모델 비활성화       |
| CollectionItemAdded   | Collection    | Query Aggregation             | 내 라이브러리 조회 모델 갱신 |
| CollectionItemRemoved | Collection    | Query Aggregation             | 조회 모델에서 제거       |
| WatchStatusChanged    | Watch Status  | Query Aggregation             | 시청 상태 반영         |
| RatingReviewChanged   | Rating Review | Query Aggregation             | 개인 평점 반영         |
| RatingSummaryUpdated  | Rating Review | Movie Catalog 또는 Query        | 평균 평점 표시         |
| OwnershipChanged      | Ownership     | Query Aggregation             | 소장 정보 반영         |

## 7. 이벤트 설계

## 7.1 이벤트 공통 구조

```json
{
  "eventId": "evt-001",
  "eventType": "WatchStatusChanged",
  "occurredAt": "2026-05-12T10:00:00Z",
  "producer": "watch-status-service",
  "version": 1,
  "payload": {}
}
```

## 7.2 CollectionItemAdded

```json
{
  "eventId": "evt-101",
  "eventType": "CollectionItemAdded",
  "occurredAt": "2026-05-12T10:00:00Z",
  "producer": "collection-service",
  "version": 1,
  "payload": {
    "userId": "u-001",
    "movieId": "m-001",
    "collectionItemId": "c-001",
    "memo": "소장판",
    "addedAt": "2026-05-12T09:59:00Z"
  }
}
```

## 7.3 WatchStatusChanged

```json
{
  "eventId": "evt-102",
  "eventType": "WatchStatusChanged",
  "occurredAt": "2026-05-12T10:05:00Z",
  "producer": "watch-status-service",
  "version": 1,
  "payload": {
    "userId": "u-001",
    "movieId": "m-001",
    "status": "WATCHED",
    "watchedAt": "2026-05-10"
  }
}
```

## 7.4 RatingReviewChanged

```json
{
  "eventId": "evt-103",
  "eventType": "RatingReviewChanged",
  "occurredAt": "2026-05-12T10:10:00Z",
  "producer": "rating-review-service",
  "version": 1,
  "payload": {
    "userId": "u-001",
    "movieId": "m-001",
    "rating": 4.5,
    "hasReview": true
  }
}
```

## 7.5 이벤트 처리 원칙

```text
이벤트는 상태 변경 사실만 전달한다.
다른 서비스의 내부 DB 구조를 이벤트에 노출하지 않는다.
이벤트 소비자는 멱등성을 보장한다.
eventId 기준 중복 처리를 방지한다.
실패한 이벤트는 DLQ로 보낸다.
```

## 8. 인증/인가 설계

## 8.1 인증 방식

* JWT 기반 인증
* API Gateway에서 1차 토큰 검증
* 내부 서비스는 Gateway가 전달한 사용자 컨텍스트를 신뢰하되, 민감 작업은 자체 검증
* 내부 서비스 간 통신은 mTLS 또는 내부 네트워크 정책 적용

## 8.2 JWT Claims 예시

```json
{
  "sub": "u-001",
  "email": "user@example.com",
  "role": "USER",
  "iat": 1778551200,
  "exp": 1778554800
}
```

## 8.3 권한 정책

| 기능                | 권한           |
| ----------------- | ------------ |
| 내 컬렉션 조회          | USER         |
| 내 컬렉션 수정          | USER         |
| 내 시청 상태 수정        | USER         |
| 내 평점/리뷰 수정        | USER         |
| 내 소장 정보 수정        | USER         |
| 영화 메타데이터 등록/수정/삭제 | ADMIN        |
| 장르/인물 정보 관리       | ADMIN        |
| 시스템 모니터링          | SYSTEM_ADMIN |

## 8.4 사용자 데이터 접근 규칙

각 서비스는 요청의 `userId`가 리소스의 `userId`와 일치하는지 확인합니다.

```text
/users/me 계열 API:
JWT.sub == resource.userId 이어야 한다.
```

관리자 API는 별도 경로로 분리합니다.

```text
/admin/**
```

## 9. 배포 구조

## 9.1 기본 배포 단위

각 서비스는 독립 컨테이너로 배포합니다.

```text
api-gateway
auth-user-service
movie-catalog-service
collection-service
watch-status-service
rating-review-service
ownership-service
query-aggregation-service
event-broker
```

## 9.2 인프라 구성

```text
Kubernetes Cluster
  ├── API Gateway / Ingress
  ├── Auth User Service + User DB
  ├── Movie Catalog Service + Catalog DB
  ├── Collection Service + Collection DB
  ├── Watch Status Service + WatchStatus DB
  ├── Rating Review Service + Rating DB
  ├── Ownership Service + Ownership DB
  ├── Query Aggregation Service + Query DB
  ├── Event Broker
  ├── Observability Stack
  └── Secret Manager
```

## 9.3 DB 배치

각 서비스는 논리적으로 독립된 DB를 갖습니다.

```text
user_db
catalog_db
collection_db
watch_status_db
rating_db
ownership_db
query_db
```

개발 초기에는 하나의 DB 인스턴스 안에 schema를 분리할 수 있지만, 논리적 소유권은 반드시 서비스별로 분리해야 합니다.
즉, 다른 서비스 schema를 직접 JOIN하거나 조회하는 것은 금지합니다.

## 10. 장애 대응 전략

## 10.1 장애 격리

| 장애 상황                    | 대응                                 |
| ------------------------ | ---------------------------------- |
| Rating Review Service 장애 | 컬렉션 조회에서 평점 영역만 null 또는 “일시 불가” 표시 |
| Movie Catalog Service 장애 | Query Read Model이 있으면 기존 캐시 데이터 제공 |
| Watch Status Service 장애  | 시청 상태 수정만 제한, 컬렉션 추가는 가능           |
| Ownership Service 장애     | 소장 위치 표시만 제한                       |
| Event Broker 장애          | Outbox에 이벤트 저장 후 재전송               |
| Query Service 장애         | 각 도메인 서비스 API를 통한 기본 조회로 우회 가능     |

## 10.2 Circuit Breaker

Query Aggregation Service가 여러 서비스를 호출할 때는 Circuit Breaker를 적용합니다.

```text
Movie Catalog 호출 실패 -> 기본 영화 ID만 표시 또는 캐시 사용
Rating 호출 실패 -> rating: null
Ownership 호출 실패 -> ownership: null
```

## 10.3 Timeout

서비스 간 동기 호출에는 짧은 timeout을 적용합니다.

```text
internal API timeout: 500ms ~ 2s
```

조회 화면에서 일부 정보가 누락되더라도 전체 응답을 실패시키지 않는 것이 좋습니다.

## 10.4 Retry

Retry는 읽기 요청이나 멱등성 있는 요청에만 적용합니다.

```text
GET /movies/{movieId}
GET /ratings/movies/{movieId}/summary
```

쓰기 요청은 중복 처리 문제가 있으므로 idempotency key를 사용합니다.

## 10.5 Outbox Pattern

각 서비스는 DB 트랜잭션과 이벤트 발행 사이의 불일치를 막기 위해 Outbox Pattern을 사용합니다.

```text
비즈니스 데이터 저장
    +
outbox_event 저장
    =
하나의 DB 트랜잭션

별도 publisher가 outbox_event를 Event Broker로 발행
```

## 11. 확장성 설계

## 11.1 서비스별 확장 포인트

| 서비스                       | 확장 기준               |
| ------------------------- | ------------------- |
| Movie Catalog Service     | 영화 검색량, 관리자 등록량     |
| Collection Service        | 사용자 수, 컬렉션 추가/삭제 빈도 |
| Watch Status Service      | 시청 상태 변경 빈도         |
| Rating Review Service     | 리뷰 작성량, 평균 평점 집계량   |
| Ownership Service         | 소장 정보 변경 빈도         |
| Query Aggregation Service | 내 라이브러리 조회 트래픽      |
| Event Broker              | 이벤트 처리량             |

## 11.2 캐싱 전략

| 대상          | 캐싱 위치                      | 이유       |
| ----------- | -------------------------- | -------- |
| 영화 상세       | Catalog Service 또는 Gateway | 변경 빈도 낮음 |
| 장르 목록       | Catalog Service            | 거의 정적    |
| 내 컬렉션 통합 조회 | Query Service              | 조회 빈도 높음 |
| 영화 평균 평점    | Rating Summary Cache       | 집계 비용 감소 |

## 11.3 검색 확장

초기에는 Catalog DB 검색으로 충분합니다.
향후 영화 수가 많아지면 Search Service 또는 검색 엔진을 분리합니다.

```text
Movie Catalog Service
    -> MovieCreated / MovieUpdated
        -> Search Indexer
            -> Search Engine
```

검색은 별도 서비스로 너무 일찍 분리하지 않는 것이 좋습니다. 지금 단계에서 분리하면 운영 복잡도만 커질 수 있습니다.

## 12. 관측성 설계

## 12.1 로그

모든 서비스는 구조화 로그를 남깁니다.

```json
{
  "traceId": "tr-001",
  "spanId": "sp-001",
  "service": "collection-service",
  "userId": "u-001",
  "path": "/collections/me/items",
  "status": 201,
  "durationMs": 38
}
```

## 12.2 메트릭

서비스별로 다음 지표를 수집합니다.

```text
request_count
request_latency
error_rate
db_query_latency
event_publish_count
event_consume_lag
circuit_breaker_open_count
```

## 12.3 분산 트레이싱

API Gateway에서 traceId를 생성하고 모든 내부 호출과 이벤트에 전달합니다.

```text
Client Request
  -> API Gateway
    -> Query Service
      -> Collection Service
      -> Movie Catalog Service
      -> Rating Review Service
```

이렇게 하면 통합 조회가 느릴 때 어느 서비스가 병목인지 추적할 수 있습니다.

## 13. 설계상 트레이드오프

## 13.1 서비스 분리 vs 운영 복잡도

현재 설계는 책임 경계를 명확히 하기 위해 여러 서비스를 분리했습니다.

장점:

```text
서비스별 독립 배포
데이터 소유권 명확
장애 격리 가능
기능별 확장 가능
```

단점:

```text
운영 복잡도 증가
서비스 간 통신 비용 증가
로컬 개발 환경 복잡
트랜잭션 처리 어려움
```

대안:

```text
초기 MVP에서는 Watch Status, Rating Review, Ownership을 User Movie Activity Service로 묶고,
트래픽이나 변경 요구가 커질 때 분리한다.
```

## 13.2 강한 일관성 vs 최종적 일관성

예를 들어 영화 제목이 변경되었을 때 Query Service의 조회 모델에 즉시 반영되지 않을 수 있습니다.

장점:

```text
서비스 간 결합도 감소
장애 격리 향상
쓰기 성능 개선
```

단점:

```text
사용자가 잠깐 오래된 데이터를 볼 수 있음
이벤트 재처리/동기화 로직 필요
```

권장:

```text
사용자 개인 데이터는 즉시 반영한다.
영화 메타데이터, 평균 평점, 조회 모델은 최종적 일관성을 허용한다.
```

## 13.3 API Composition vs CQRS Read Model

### API Composition

장점:

```text
구현이 단순
데이터 중복 적음
초기 개발 빠름
```

단점:

```text
조회 시 여러 서비스 호출
장애 전파 위험
응답 지연 가능
```

### CQRS Read Model

장점:

```text
조회 성능 우수
장애 격리 좋음
프론트엔드 최적화 쉬움
```

단점:

```text
이벤트 설계 필요
동기화 지연 발생
운영 난이도 증가
```

권장:

```text
초기: API Composition
성장 후: 이벤트 기반 CQRS Read Model
```

## 13.4 Collection과 WatchStatus 의존성

시청 상태를 반드시 컬렉션에 포함된 영화에만 허용할지 결정해야 합니다.

강한 의존 설계:

```text
Watch Status Service -> Collection Service 확인
```

문제:

```text
Watch Status Service가 Collection Service 장애에 영향받음
서비스 독립성이 낮아짐
```

권장 설계:

```text
Watch Status Service는 userId + movieId 기준으로 독립 저장
화면에서 컬렉션 포함 여부와 시청 상태를 조합
```

이 방식이 의존성 최소화에 더 적합합니다.

# 최종 권장 아키텍처

```text
[Client]
   |
[API Gateway]
   |
   +-- Auth/User Service -------- User DB
   |
   +-- Movie Catalog Service ---- Catalog DB
   |
   +-- Collection Service ------- Collection DB
   |
   +-- Watch Status Service ----- WatchStatus DB
   |
   +-- Rating Review Service ---- Rating DB
   |
   +-- Ownership Service -------- Ownership DB
   |
   +-- Query Aggregation Service - Query DB(optional)
   |
[Event Broker]
```

## 핵심 설계 결론

이 시스템의 가장 중요한 경계는 다음입니다.

```text
Movie Catalog Service = 공통 영화 정보
Collection Service = 사용자가 내 컬렉션에 담았는가
Watch Status Service = 사용자가 봤는가
Rating Review Service = 사용자가 어떻게 평가했는가
Ownership Service = 사용자가 어디에/어떤 형태로 소장하는가
Query Service = 화면에 어떻게 보여줄 것인가
```

따라서 각 서비스는 `userId`, `movieId`를 참조 ID로만 사용하고, 다른 서비스의 DB에는 접근하지 않습니다.
조회 편의를 위한 데이터 조합은 Query Aggregation Service가 담당하며, 원천 데이터 변경 책임은 각 도메인 서비스에만 둡니다. 이게 서비스 책임 경계와 의존성 최소화를 동시에 만족하는 가장 안정적인 설계입니다.
