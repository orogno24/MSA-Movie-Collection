아래는 **「01 Design Spec.md」의 MSA 설계**를 기준으로 작성한 **Function Signature 설계서**입니다.
코드는 작성하지 않고, 각 서비스가 외부/내부적으로 제공해야 할 **기능 계약**, **입력**, **출력**, **권한**, **이벤트 발행 여부**, **의존성 경계**만 정의합니다. 설계서는 서비스 분리 기준을 “데이터 소유권 + 변경 이유 + 비즈니스 책임”으로 두고, 각 서비스가 `userId`, `movieId`를 참조 ID로만 사용하며 다른 서비스 DB에 직접 접근하지 않는 구조를 전제로 합니다. 

# 영화 컬렉션 관리 시스템 Function Signature

## 1. Function Signature 설계 원칙

| 원칙                 | 설명                                                                 |
| ------------------ | ------------------------------------------------------------------ |
| Command / Query 분리 | 상태를 변경하는 함수와 조회 함수는 분리한다.                                          |
| 서비스 내부 데이터 소유      | 각 함수는 자기 서비스가 소유한 데이터만 직접 변경한다.                                    |
| 참조 ID 기반 연계        | 다른 서비스의 데이터는 `userId`, `movieId`, `collectionItemId` 같은 ID로만 참조한다. |
| 직접 DB 접근 금지        | 서비스 간 DB Join, 타 서비스 DB 조회는 금지한다.                                  |
| 이벤트 기반 확장          | 상태 변경 함수는 필요한 경우 이벤트 발행 계약을 포함한다.                                  |
| Query 조합 분리        | 통합 조회는 Query Aggregation Service가 담당한다.                            |
| 권한 명시              | 사용자 함수와 관리자 함수의 권한을 분리한다.                                          |

---

# 2. 공통 Signature 타입

## 2.1 공통 입력 컨텍스트

| 이름                | 설명                       |
| ----------------- | ------------------------ |
| UserContext       | 인증된 사용자 정보               |
| AdminContext      | 관리자 권한 사용자 정보            |
| SystemContext     | 내부 시스템 또는 이벤트 처리 컨텍스트    |
| PaginationRequest | 페이지 번호, 페이지 크기, 정렬 조건    |
| SearchCondition   | 검색어, 필터, 정렬 조건           |
| IdempotencyKey    | 중복 요청 방지용 키              |
| TraceContext      | traceId, spanId 등 관측성 정보 |

## 2.2 공통 출력 타입

| 이름                 | 설명                     |
| ------------------ | ---------------------- |
| CommandResult      | 생성, 수정, 삭제 명령의 처리 결과   |
| QueryResult        | 단건 조회 결과               |
| PageResult         | 목록 조회 결과               |
| ValidationResult   | 검증 결과                  |
| ErrorResult        | 실패 사유, 오류 코드, 사용자 메시지  |
| EventPublishResult | 이벤트 발행 또는 Outbox 저장 결과 |

---

# 3. Auth/User Service Function Signature

Auth/User Service는 사용자 식별, 인증, 권한만 책임집니다. 다른 서비스는 User DB를 직접 조회하지 않고 JWT의 `userId`, `role` 클레임을 사용합니다. 

## 3.1 Command Functions

| Function        | 입력                                  | 출력                | 권한        | 이벤트                |
| --------------- | ----------------------------------- | ----------------- | --------- | ------------------ |
| signUp          | SignUpCommand                       | AuthUserResult    | Anonymous | UserCreated        |
| login           | LoginCommand                        | TokenResult       | Anonymous | 없음                 |
| refreshToken    | RefreshTokenCommand                 | TokenResult       | Anonymous | 없음                 |
| updateMyProfile | UserContext, UpdateProfileCommand   | UserProfileResult | USER      | UserProfileUpdated |
| changePassword  | UserContext, ChangePasswordCommand  | CommandResult     | USER      | PasswordChanged    |
| changeUserRole  | AdminContext, ChangeUserRoleCommand | UserProfileResult | ADMIN     | UserRoleChanged    |
| deactivateUser  | AdminContext, DeactivateUserCommand | CommandResult     | ADMIN     | UserDeactivated    |

## 3.2 Query Functions

| Function      | 입력                                                   | 출력                      | 권한               | 설명                      |
| ------------- | ---------------------------------------------------- | ----------------------- | ---------------- | ----------------------- |
| getMyProfile  | UserContext                                          | UserProfileResult       | USER             | 내 프로필 조회                |
| getUserById   | AdminContext, userId                                 | UserProfileResult       | ADMIN            | 관리자용 사용자 조회             |
| searchUsers   | AdminContext, UserSearchCondition, PaginationRequest | PageResult<UserSummary> | ADMIN            | 사용자 목록 검색               |
| validateToken | AccessToken                                          | TokenValidationResult   | Gateway/Internal | API Gateway 또는 내부 인증 검증 |

## 3.3 주요 Request / Response

| 타입                | 필드                                      |
| ----------------- | --------------------------------------- |
| SignUpCommand     | email, password, name                   |
| LoginCommand      | email, password                         |
| TokenResult       | accessToken, refreshToken, userId, role |
| UserProfileResult | userId, email, name, role, status       |

---

# 4. Movie Catalog Service Function Signature

Movie Catalog Service는 영화, 장르, 배우, 감독 등 **공통 영화 메타데이터**만 책임집니다. 사용자별 컬렉션, 평점, 시청 상태, 소장 위치는 포함하지 않습니다. 

## 4.1 Movie Command Functions

| Function              | 입력                                           | 출력                | 권한    | 이벤트          |
| --------------------- | -------------------------------------------- | ----------------- | ----- | ------------ |
| createMovie           | AdminContext, CreateMovieCommand             | MovieDetailResult | ADMIN | MovieCreated |
| updateMovie           | AdminContext, movieId, UpdateMovieCommand    | MovieDetailResult | ADMIN | MovieUpdated |
| deleteMovie           | AdminContext, movieId                        | CommandResult     | ADMIN | MovieDeleted |
| addGenresToMovie      | AdminContext, movieId, GenreIdsCommand       | MovieDetailResult | ADMIN | MovieUpdated |
| removeGenreFromMovie  | AdminContext, movieId, genreId               | MovieDetailResult | ADMIN | MovieUpdated |
| addCreditToMovie      | AdminContext, movieId, AddMovieCreditCommand | MovieDetailResult | ADMIN | MovieUpdated |
| removeCreditFromMovie | AdminContext, movieId, creditId              | MovieDetailResult | ADMIN | MovieUpdated |

## 4.2 Movie Query Functions

| Function              | 입력                                      | 출력                       | 권한          | 설명                       |
| --------------------- | --------------------------------------- | ------------------------ | ----------- | ------------------------ |
| getMovieDetail        | movieId                                 | MovieDetailResult        | Public/USER | 영화 상세 조회                 |
| searchMovies          | MovieSearchCondition, PaginationRequest | PageResult<MovieSummary> | Public/USER | 제목, 장르, 배우, 감독 기준 검색     |
| getMoviesByIds        | movieIds                                | List<MovieSummary>       | Internal    | Query Aggregation용 벌크 조회 |
| existsMovie           | movieId                                 | ValidationResult         | Internal    | movieId 존재 여부 확인         |
| getMovieRatingDisplay | movieId                                 | MovieRatingDisplayResult | Public/USER | 평균 평점 표시용 조회, 선택 사항      |

## 4.3 Genre Functions

| Function    | 입력                                        | 출력                | 권한          | 이벤트          |
| ----------- | ----------------------------------------- | ----------------- | ----------- | ------------ |
| createGenre | AdminContext, CreateGenreCommand          | GenreResult       | ADMIN       | GenreCreated |
| updateGenre | AdminContext, genreId, UpdateGenreCommand | GenreResult       | ADMIN       | GenreUpdated |
| deleteGenre | AdminContext, genreId                     | CommandResult     | ADMIN       | GenreDeleted |
| getGenres   | 없음                                        | List<GenreResult> | Public/USER | 없음           |

## 4.4 Person / Credit Functions

| Function        | 입력                                          | 출력                        | 권한          | 이벤트           |
| --------------- | ------------------------------------------- | ------------------------- | ----------- | ------------- |
| createPerson    | AdminContext, CreatePersonCommand           | PersonResult              | ADMIN       | PersonCreated |
| updatePerson    | AdminContext, personId, UpdatePersonCommand | PersonResult              | ADMIN       | PersonUpdated |
| getPersonDetail | personId                                    | PersonDetailResult        | Public/USER | 없음            |
| searchPersons   | PersonSearchCondition, PaginationRequest    | PageResult<PersonSummary> | Public/USER | 없음            |

## 4.5 주요 Request / Response

| 타입                   | 필드                                                                                           |
| -------------------- | -------------------------------------------------------------------------------------------- |
| CreateMovieCommand   | title, originalTitle, releaseYear, description, runtime, posterUrl, genreIds, credits        |
| UpdateMovieCommand   | title, originalTitle, releaseYear, description, runtime, posterUrl                           |
| MovieSearchCondition | keyword, genreId, actorName, directorName, releaseYear, page, size                           |
| MovieSummary         | movieId, title, releaseYear, posterUrl, genres                                               |
| MovieDetailResult    | movieId, title, originalTitle, releaseYear, description, runtime, posterUrl, genres, credits |

---

# 5. Collection Service Function Signature

Collection Service는 사용자가 특정 영화를 자신의 컬렉션에 담았는지, 그리고 컬렉션 메모가 무엇인지만 책임집니다. 영화 제목, 장르, 포스터 등은 소유하지 않습니다. 설계서에서도 Collection Service는 `movieId`만 참조하고 영화 상세 정보는 API 조회 또는 읽기 모델을 통해 해결하도록 되어 있습니다. 

## 5.1 Command Functions

| Function                           | 입력                                                         | 출력                   | 권한   | 이벤트                   |
| ---------------------------------- | ---------------------------------------------------------- | -------------------- | ---- | --------------------- |
| addMovieToCollection               | UserContext, AddCollectionItemCommand, IdempotencyKey      | CollectionItemResult | USER | CollectionItemAdded   |
| updateCollectionMemo               | UserContext, collectionItemId, UpdateCollectionMemoCommand | CollectionItemResult | USER | CollectionItemUpdated |
| removeMovieFromCollection          | UserContext, collectionItemId                              | CommandResult        | USER | CollectionItemRemoved |
| removeMovieFromCollectionByMovieId | UserContext, movieId                                       | CommandResult        | USER | CollectionItemRemoved |

## 5.2 Query Functions

| Function                     | 입력                             | 출력                               | 권한            | 설명                       |
| ---------------------------- | ------------------------------ | -------------------------------- | ------------- | ------------------------ |
| getMyCollectionItems         | UserContext, PaginationRequest | PageResult<CollectionItemResult> | USER          | 내 컬렉션 목록 조회              |
| getCollectionItem            | UserContext, collectionItemId  | CollectionItemResult             | USER          | 컬렉션 항목 단건 조회             |
| getCollectionItemByMovieId   | UserContext, movieId           | CollectionItemResult             | USER          | 특정 영화의 컬렉션 포함 여부 조회      |
| existsInCollection           | UserContext, movieId           | ValidationResult                 | USER/Internal | 컬렉션 포함 여부 확인             |
| getCollectionItemsByMovieIds | UserContext, movieIds          | List<CollectionItemResult>       | Internal      | Query Aggregation용 벌크 조회 |

## 5.3 주요 Request / Response

| 타입                          | 필드                                                          |
| --------------------------- | ----------------------------------------------------------- |
| AddCollectionItemCommand    | movieId, memo                                               |
| UpdateCollectionMemoCommand | memo                                                        |
| CollectionItemResult        | collectionItemId, userId, movieId, memo, addedAt, updatedAt |

## 5.4 경계 규칙

| 규칙               | 설명                                                        |
| ---------------- | --------------------------------------------------------- |
| 영화 상세 정보 미포함     | CollectionItemResult에는 title, posterUrl, genre를 포함하지 않는다. |
| 중복 추가 방지         | userId + movieId는 유일해야 한다.                                |
| Catalog DB 접근 금지 | movieId 검증이 필요해도 Catalog DB를 직접 조회하지 않는다.                 |
| Query Service 위임 | 화면용 통합 정보는 Query Aggregation Service에서 조합한다.              |

---

# 6. Watch Status Service Function Signature

Watch Status Service는 사용자별 영화 시청 상태만 관리합니다. 설계서에서는 이 서비스가 Collection Service에 강하게 의존하지 않고 `userId + movieId` 기준으로 독립 관리하도록 권장합니다. 

## 6.1 Command Functions

| Function          | 입력                                           | 출력                | 권한   | 이벤트                |
| ----------------- | -------------------------------------------- | ----------------- | ---- | ------------------ |
| setWatchStatus    | UserContext, movieId, SetWatchStatusCommand  | WatchStatusResult | USER | WatchStatusChanged |
| updateWatchedAt   | UserContext, movieId, UpdateWatchedAtCommand | WatchStatusResult | USER | WatchStatusChanged |
| removeWatchStatus | UserContext, movieId                         | CommandResult     | USER | WatchStatusRemoved |

## 6.2 Query Functions

| Function                   | 입력                                                         | 출력                            | 권한       | 설명                       |
| -------------------------- | ---------------------------------------------------------- | ----------------------------- | -------- | ------------------------ |
| getMyWatchStatuses         | UserContext, WatchStatusSearchCondition, PaginationRequest | PageResult<WatchStatusResult> | USER     | 내 시청 상태 목록 조회            |
| getWatchStatusByMovieId    | UserContext, movieId                                       | WatchStatusResult             | USER     | 영화별 시청 상태 조회             |
| getWatchStatusesByMovieIds | UserContext, movieIds                                      | List<WatchStatusResult>       | Internal | Query Aggregation용 벌크 조회 |

## 6.3 주요 Request / Response

| 타입                         | 필드                                                           |
| -------------------------- | ------------------------------------------------------------ |
| SetWatchStatusCommand      | status, watchedAt                                            |
| UpdateWatchedAtCommand     | watchedAt                                                    |
| WatchStatusSearchCondition | status, watchedFrom, watchedTo                               |
| WatchStatusResult          | watchStatusId, userId, movieId, status, watchedAt, updatedAt |

## 6.4 상태값 계약

| 상태            | 의미    |
| ------------- | ----- |
| WANT_TO_WATCH | 보고 싶음 |
| WATCHING      | 보는 중  |
| WATCHED       | 시청 완료 |
| ON_HOLD       | 보류    |
| DROPPED       | 중단    |

## 6.5 경계 규칙

| 규칙               | 설명                                          |
| ---------------- | ------------------------------------------- |
| Collection 의존 금지 | 시청 상태 등록 시 Collection Service를 필수 호출하지 않는다. |
| movieId 참조만 유지   | 영화 제목, 장르, 포스터는 저장하지 않는다.                   |
| 독립 수정 가능         | 컬렉션에 없는 영화도 시청 상태를 가질 수 있도록 허용 가능하다.        |

---

# 7. Rating Review Service Function Signature

Rating Review Service는 사용자 평점, 리뷰, 평균 평점 집계를 책임집니다. 영화 제목이나 장르는 소유하지 않고, `movieId` 기준으로만 평점 데이터를 관리합니다. 설계서에서도 평균 평점은 이벤트 기반 집계를 권장합니다. 

## 7.1 Command Functions

| Function                      | 입력                                                  | 출력                       | 권한     | 이벤트                  |
| ----------------------------- | --------------------------------------------------- | ------------------------ | ------ | -------------------- |
| createOrUpdateRatingReview    | UserContext, movieId, UpsertRatingReviewCommand     | RatingReviewResult       | USER   | RatingReviewChanged  |
| updateReviewVisibility        | UserContext, movieId, UpdateReviewVisibilityCommand | RatingReviewResult       | USER   | RatingReviewChanged  |
| deleteRatingReview            | UserContext, movieId                                | CommandResult            | USER   | RatingReviewDeleted  |
| recalculateMovieRatingSummary | SystemContext, movieId                              | MovieRatingSummaryResult | SYSTEM | RatingSummaryUpdated |

## 7.2 Query Functions

| Function                   | 입력                                                          | 출력                             | 권한          | 설명                       |
| -------------------------- | ----------------------------------------------------------- | ------------------------------ | ----------- | ------------------------ |
| getMyRatingReview          | UserContext, movieId                                        | RatingReviewResult             | USER        | 내 평점/리뷰 조회               |
| getMyRatingReviews         | UserContext, RatingReviewSearchCondition, PaginationRequest | PageResult<RatingReviewResult> | USER        | 내 리뷰 목록 조회               |
| getPublicReviewsByMovieId  | movieId, PaginationRequest                                  | PageResult<PublicReviewResult> | Public/USER | 공개 리뷰 목록 조회              |
| getMovieRatingSummary      | movieId                                                     | MovieRatingSummaryResult       | Public/USER | 영화 평균 평점 조회              |
| getRatingReviewsByMovieIds | UserContext, movieIds                                       | List<RatingReviewResult>       | Internal    | Query Aggregation용 벌크 조회 |

## 7.3 주요 Request / Response

| 타입                            | 필드                                                                              |
| ----------------------------- | ------------------------------------------------------------------------------- |
| UpsertRatingReviewCommand     | rating, reviewText, visibility                                                  |
| UpdateReviewVisibilityCommand | visibility                                                                      |
| RatingReviewSearchCondition   | ratingFrom, ratingTo, visibility, createdFrom, createdTo                        |
| RatingReviewResult            | reviewId, userId, movieId, rating, reviewText, visibility, createdAt, updatedAt |
| MovieRatingSummaryResult      | movieId, averageRating, ratingCount, updatedAt                                  |
| PublicReviewResult            | reviewId, movieId, userDisplayName, rating, reviewText, createdAt               |

## 7.4 경계 규칙

| 규칙               | 설명                                          |
| ---------------- | ------------------------------------------- |
| 영화 정보 소유 금지      | RatingReviewResult에 영화 제목, 장르를 직접 저장하지 않는다. |
| 평균 평점 분리         | 개인 평점 변경 후 평균 평점은 별도 집계 함수 또는 이벤트로 갱신한다.    |
| Collection 의존 금지 | 평점 작성 시 컬렉션 포함 여부를 필수 검증하지 않는다.             |
| 사용자별 유일성         | userId + movieId 기준으로 하나의 평점/리뷰만 허용한다.      |

---

# 8. Ownership Service Function Signature

Ownership Service는 사용자가 영화를 어떤 형태로, 어디에 소장하고 있는지를 관리합니다. 설계서에서는 Ownership Service가 Collection Service에 직접 종속되지 않고 `userId + movieId` 기준으로 독립 관리하도록 되어 있습니다. 

## 8.1 Command Functions

| Function                  | 입력                                               | 출력              | 권한   | 이벤트              |
| ------------------------- | ------------------------------------------------ | --------------- | ---- | ---------------- |
| addOwnership              | UserContext, movieId, AddOwnershipCommand        | OwnershipResult | USER | OwnershipChanged |
| updateOwnership           | UserContext, ownershipId, UpdateOwnershipCommand | OwnershipResult | USER | OwnershipChanged |
| removeOwnership           | UserContext, ownershipId                         | CommandResult   | USER | OwnershipRemoved |
| removeOwnershipsByMovieId | UserContext, movieId                             | CommandResult   | USER | OwnershipRemoved |

## 8.2 Query Functions

| Function                | 입력                                                       | 출력                          | 권한       | 설명                       |
| ----------------------- | -------------------------------------------------------- | --------------------------- | -------- | ------------------------ |
| getMyOwnerships         | UserContext, OwnershipSearchCondition, PaginationRequest | PageResult<OwnershipResult> | USER     | 내 소장 정보 목록               |
| getOwnershipsByMovieId  | UserContext, movieId                                     | List<OwnershipResult>       | USER     | 특정 영화의 소장 정보 조회          |
| getOwnershipsByMovieIds | UserContext, movieIds                                    | List<OwnershipResult>       | Internal | Query Aggregation용 벌크 조회 |

## 8.3 주요 Request / Response

| 타입                       | 필드                                                                                   |
| ------------------------ | ------------------------------------------------------------------------------------ |
| AddOwnershipCommand      | mediaType, location, platform, purchaseDate                                          |
| UpdateOwnershipCommand   | mediaType, location, platform, purchaseDate                                          |
| OwnershipSearchCondition | mediaType, platform, locationKeyword                                                 |
| OwnershipResult          | ownershipId, userId, movieId, mediaType, location, platform, purchaseDate, updatedAt |

## 8.4 mediaType 계약

| 값                  | 의미       |
| ------------------ | -------- |
| BLURAY             | 블루레이     |
| DVD                | DVD      |
| DIGITAL_FILE       | 디지털 파일   |
| STREAMING_PURCHASE | 스트리밍 구매  |
| RENTAL             | 대여       |
| THEATER            | 극장 관람 기록 |

## 8.5 경계 규칙

| 규칙               | 설명                                          |
| ---------------- | ------------------------------------------- |
| Collection 종속 금지 | 컬렉션 항목이 없어도 소장 정보 등록 가능 여부를 정책적으로 허용할 수 있다. |
| 영화 메타데이터 미포함     | title, genre, posterUrl은 저장하지 않는다.          |
| 복수 소장 허용         | 같은 영화에 Blu-ray, 디지털 파일 등 복수 소장 정보를 허용한다.    |

---

# 9. Query Aggregation Service Function Signature

Query Aggregation Service는 화면 조회에 필요한 데이터를 조합합니다. 설계서에서는 이 서비스가 원천 데이터를 수정하지 않고, 초기에는 API Composition, 이후 이벤트 기반 Read Model로 확장하는 방식을 권장합니다. 

## 9.1 Query Functions

| Function                   | 입력                                                     | 출력                              | 권한   | 조회 방식                         |
| -------------------------- | ------------------------------------------------------ | ------------------------------- | ---- | ----------------------------- |
| getMyLibrary               | UserContext, LibrarySearchCondition, PaginationRequest | PageResult<MyLibraryItemResult> | USER | API Composition 또는 Read Model |
| getMyLibraryItemDetail     | UserContext, movieId                                   | MyLibraryItemDetailResult       | USER | API Composition 또는 Read Model |
| getMyDashboard             | UserContext                                            | MyDashboardResult               | USER | API Composition 또는 Read Model |
| searchMyLibrary            | UserContext, LibrarySearchCondition, PaginationRequest | PageResult<MyLibraryItemResult> | USER | Read Model 권장                 |
| getRecentlyAddedMovies     | UserContext, limit                                     | List<MyLibraryItemResult>       | USER | Read Model 권장                 |
| getWatchingProgressSummary | UserContext                                            | WatchProgressSummaryResult      | USER | Read Model 권장                 |

## 9.2 Event Handler Functions

| Function                    | 입력                         | 출력                  | 구독 이벤트                | 목적                |
| --------------------------- | -------------------------- | ------------------- | --------------------- | ----------------- |
| handleMovieCreated          | MovieCreatedEvent          | EventHandlingResult | MovieCreated          | 조회 모델 영화 정보 생성    |
| handleMovieUpdated          | MovieUpdatedEvent          | EventHandlingResult | MovieUpdated          | 영화 제목, 포스터, 장르 반영 |
| handleMovieDeleted          | MovieDeletedEvent          | EventHandlingResult | MovieDeleted          | 조회 모델 비활성화        |
| handleCollectionItemAdded   | CollectionItemAddedEvent   | EventHandlingResult | CollectionItemAdded   | 내 라이브러리 항목 생성     |
| handleCollectionItemRemoved | CollectionItemRemovedEvent | EventHandlingResult | CollectionItemRemoved | 내 라이브러리 항목 제거     |
| handleWatchStatusChanged    | WatchStatusChangedEvent    | EventHandlingResult | WatchStatusChanged    | 시청 상태 갱신          |
| handleRatingReviewChanged   | RatingReviewChangedEvent   | EventHandlingResult | RatingReviewChanged   | 개인 평점/리뷰 갱신       |
| handleOwnershipChanged      | OwnershipChangedEvent      | EventHandlingResult | OwnershipChanged      | 소장 정보 갱신          |

## 9.3 주요 Request / Response

| 타입                         | 필드                                                                                         |
| -------------------------- | ------------------------------------------------------------------------------------------ |
| LibrarySearchCondition     | keyword, genreId, watchStatus, ratingFrom, ratingTo, mediaType, sort                       |
| MyLibraryItemResult        | movieId, title, releaseYear, posterUrl, genres, collection, watchStatus, rating, ownership |
| MyLibraryItemDetailResult  | movie, collection, watchStatus, ratingReview, ownerships                                   |
| MyDashboardResult          | totalCollectionCount, watchedCount, watchingCount, averageMyRating, recentlyAddedItems     |
| WatchProgressSummaryResult | totalCount, wantToWatchCount, watchingCount, watchedCount, onHoldCount, droppedCount       |

## 9.4 경계 규칙

| 규칙        | 설명                                            |
| --------- | --------------------------------------------- |
| 쓰기 기능 없음  | Query Aggregation Service는 원천 데이터를 수정하지 않는다.  |
| 원천 소유권 없음 | 조회 모델은 재생성 가능한 파생 데이터다.                       |
| 장애 허용     | 일부 서비스 장애 시 해당 영역만 null 또는 unavailable로 응답한다. |
| 이벤트 멱등성   | 같은 eventId가 중복 수신되어도 결과가 변하지 않아야 한다.          |

---

# 10. API Gateway Function Signature

API Gateway는 외부 요청 진입점, 인증 토큰 검증, 라우팅, Rate Limit, 공통 오류 처리를 담당합니다. 설계서에서도 Gateway는 비즈니스 로직과 DB 접근을 포함하지 않는다고 정의되어 있습니다. 

## 10.1 Gateway Functions

| Function            | 입력                          | 출력                              | 설명                   |
| ------------------- | --------------------------- | ------------------------------- | -------------------- |
| authenticateRequest | HttpRequest                 | UserContext 또는 AnonymousContext | JWT 검증 및 사용자 컨텍스트 생성 |
| authorizeRequest    | UserContext, RoutePolicy    | AuthorizationResult             | 경로별 권한 확인            |
| routeRequest        | HttpRequest, UserContext    | HttpResponse                    | 대상 서비스로 요청 전달        |
| applyRateLimit      | ClientIdentity, RoutePolicy | RateLimitResult                 | 요청 제한 확인             |
| enrichTraceContext  | HttpRequest                 | TraceContext                    | traceId 생성 및 전달      |
| handleGatewayError  | ErrorResult                 | HttpResponse                    | 공통 오류 응답 변환          |

## 10.2 경계 규칙

| 규칙          | 설명                                    |
| ----------- | ------------------------------------- |
| 비즈니스 로직 금지  | 컬렉션 추가, 평점 등록 같은 도메인 판단을 하지 않는다.      |
| DB 접근 금지    | Gateway는 DB를 소유하지 않는다.                |
| 사용자 컨텍스트 전달 | 내부 서비스에는 userId, role, traceId를 전달한다. |

---

# 11. Event Contract Signature

설계서에서는 상태 변경 이벤트를 통해 Query Aggregation Service와 관련 조회 모델을 갱신하는 구조를 정의합니다. 이벤트는 내부 DB 구조가 아니라 “상태 변경 사실”만 전달해야 합니다. 

## 11.1 공통 Event Envelope

| 필드         | 설명         |
| ---------- | ---------- |
| eventId    | 이벤트 고유 ID  |
| eventType  | 이벤트 타입     |
| occurredAt | 이벤트 발생 시각  |
| producer   | 이벤트 발행 서비스 |
| version    | 이벤트 스키마 버전 |
| traceId    | 분산 추적 ID   |
| payload    | 이벤트 본문     |

## 11.2 Event Signatures

| Event                 | Producer              | Payload                                                                | Consumer                            |
| --------------------- | --------------------- | ---------------------------------------------------------------------- | ----------------------------------- |
| UserCreated           | Auth/User Service     | userId, email, name, role, createdAt                                   | 필요한 내부 서비스                          |
| MovieCreated          | Movie Catalog Service | movieId, title, releaseYear, posterUrl, genres                         | Query Aggregation                   |
| MovieUpdated          | Movie Catalog Service | movieId, changedFields, updatedAt                                      | Query Aggregation                   |
| MovieDeleted          | Movie Catalog Service | movieId, deletedAt                                                     | Query Aggregation, Collection       |
| CollectionItemAdded   | Collection Service    | userId, movieId, collectionItemId, memo, addedAt                       | Query Aggregation                   |
| CollectionItemUpdated | Collection Service    | userId, movieId, collectionItemId, memo, updatedAt                     | Query Aggregation                   |
| CollectionItemRemoved | Collection Service    | userId, movieId, collectionItemId, removedAt                           | Query Aggregation                   |
| WatchStatusChanged    | Watch Status Service  | userId, movieId, status, watchedAt, updatedAt                          | Query Aggregation                   |
| RatingReviewChanged   | Rating Review Service | userId, movieId, rating, hasReview, visibility, updatedAt              | Query Aggregation                   |
| RatingSummaryUpdated  | Rating Review Service | movieId, averageRating, ratingCount, updatedAt                         | Query Aggregation, Movie Catalog 선택 |
| OwnershipChanged      | Ownership Service     | userId, movieId, ownershipId, mediaType, location, platform, updatedAt | Query Aggregation                   |
| OwnershipRemoved      | Ownership Service     | userId, movieId, ownershipId, removedAt                                | Query Aggregation                   |

---

# 12. Service Client / Port Signature

서비스 간 직접 DB 접근을 막기 위해, 필요한 경우 내부 API Client 또는 Port 계약만 둡니다. 핵심은 **쓰기 서비스 간 의존을 최소화하고, 조회 조합은 Query Aggregation Service로 집중**하는 것입니다.

## 12.1 Catalog Lookup Port

| Function          | 입력       | 출력                 | 사용 주체                                         | 목적              |
| ----------------- | -------- | ------------------ | --------------------------------------------- | --------------- |
| existsMovie       | movieId  | ValidationResult   | Collection, Rating, WatchStatus, Ownership 선택 | movieId 유효성 검증  |
| getMovieSummaries | movieIds | List<MovieSummary> | Query Aggregation                             | 통합 조회용 영화 정보 조회 |

## 12.2 Collection Lookup Port

| Function                     | 입력               | 출력                         | 사용 주체                      | 목적         |
| ---------------------------- | ---------------- | -------------------------- | -------------------------- | ---------- |
| getCollectionItemsByMovieIds | userId, movieIds | List<CollectionItemResult> | Query Aggregation          | 내 라이브러리 조합 |
| existsInCollection           | userId, movieId  | ValidationResult           | Query Aggregation 또는 정책 검증 | 포함 여부 확인   |

## 12.3 Watch Status Lookup Port

| Function                   | 입력               | 출력                      | 사용 주체             | 목적              |
| -------------------------- | ---------------- | ----------------------- | ----------------- | --------------- |
| getWatchStatusesByMovieIds | userId, movieIds | List<WatchStatusResult> | Query Aggregation | 통합 조회용 시청 상태 조회 |

## 12.4 Rating Lookup Port

| Function                   | 입력               | 출력                             | 사용 주체                         | 목적              |
| -------------------------- | ---------------- | ------------------------------ | ----------------------------- | --------------- |
| getRatingReviewsByMovieIds | userId, movieIds | List<RatingReviewResult>       | Query Aggregation             | 통합 조회용 개인 평점 조회 |
| getMovieRatingSummaries    | movieIds         | List<MovieRatingSummaryResult> | Query Aggregation, Catalog 선택 | 평균 평점 표시        |

## 12.5 Ownership Lookup Port

| Function                | 입력               | 출력                    | 사용 주체             | 목적              |
| ----------------------- | ---------------- | --------------------- | ----------------- | --------------- |
| getOwnershipsByMovieIds | userId, movieIds | List<OwnershipResult> | Query Aggregation | 통합 조회용 소장 정보 조회 |

---

# 13. 최종 서비스별 Function 그룹 요약

| 서비스                       |                  Command |                  Query | Event Handler |
| ------------------------- | -----------------------: | ---------------------: | ------------: |
| Auth/User Service         | 회원가입, 로그인, 프로필 수정, 권한 변경 |    내 정보, 사용자 조회, 토큰 검증 |            선택 |
| Movie Catalog Service     |        영화/장르/인물 생성·수정·삭제 |    영화 검색, 영화 상세, 벌크 조회 |            선택 |
| Collection Service        |             컬렉션 추가·수정·삭제 |    내 컬렉션, 포함 여부, 벌크 조회 |            선택 |
| Watch Status Service      |              시청 상태 설정·삭제 | 내 시청 상태, 영화별 상태, 벌크 조회 |            선택 |
| Rating Review Service     |   평점/리뷰 작성·삭제, 평균 평점 재계산 |     내 리뷰, 공개 리뷰, 평균 평점 |            선택 |
| Ownership Service         |           소장 정보 추가·수정·삭제 |     내 소장 정보, 영화별 소장 정보 |            선택 |
| Query Aggregation Service |                       없음 |      내 라이브러리, 상세, 대시보드 |     다수 이벤트 구독 |
| API Gateway               |                       없음 |           라우팅/인증/인가 관점 |            없음 |

## 핵심 결론

Function Signature의 핵심은 다음입니다.

| 경계                | 설계 결정                            |
| ----------------- | -------------------------------- |
| Collection        | 영화 상세를 반환하지 않고 `movieId` 중심으로 관리 |
| Watch Status      | Collection 포함 여부에 강하게 의존하지 않음    |
| Rating Review     | 영화 정보가 아니라 평점/리뷰만 소유             |
| Ownership         | CollectionItem에 종속되지 않고 독립 관리    |
| Query Aggregation | 통합 조회만 담당하고 쓰기 기능 없음             |
| Event             | 상태 변경 사실만 전달하고 내부 DB 구조는 노출하지 않음 |

이렇게 정의하면 이후 코딩 단계에서 서비스별 Controller, Application Service, Domain Service, Repository Interface를 만들 때도 책임 경계가 흐려지지 않습니다. 특히 “조회 편의 때문에 Collection Service가 Movie Catalog 데이터를 끌어안는 문제”를 피할 수 있습니다.
