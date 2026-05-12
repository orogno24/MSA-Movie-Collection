아래는 첨부된 **「02 Funtion Signature.md」**를 기준으로 작성한 **Code Skeleton**입니다. 해당 문서는 Command/Query 분리, 서비스 내부 데이터 소유, 참조 ID 기반 연계, 직접 DB 접근 금지, 이벤트 기반 확장, Query 조합 분리, 권한 명시를 핵심 원칙으로 정의하고 있습니다. 
또한 서비스별 Command/Query/Event Handler 그룹과 서비스 책임 경계가 이미 정의되어 있으므로, 아래 Skeleton은 그 계약을 그대로 코드 구조로 옮긴 형태입니다. 

전제는 다음과 같습니다.

```text
Style: Java-like Interface Skeleton
Scope: Application Service, Event Handler, Port 중심
Rule: No function body / No implementation logic
Description: English only inside function comments
```

---

# 1. Common Types Skeleton

```java
/**
 * Represents the authenticated user context propagated from the API Gateway.
 * This context should contain only identity and authorization-related information,
 * such as userId, role, and trace metadata. Domain services must not use this
 * object to access user database records directly.
 */
public interface UserContext {
}

/**
 * Represents an authenticated administrator context.
 * This context is required for operations that modify global catalog data,
 * user roles, or administrative resources.
 */
public interface AdminContext {
}

/**
 * Represents a trusted internal system context.
 * This context is used for scheduled jobs, event handlers, background processors,
 * and internal maintenance operations that are not initiated directly by an end user.
 */
public interface SystemContext {
}

/**
 * Represents pagination, sorting, and page-size information for list queries.
 * This object should be used consistently across services to avoid service-specific
 * pagination behavior leaking into clients.
 */
public interface PaginationRequest {
}

/**
 * Represents a request-level idempotency key.
 * Command operations that may be retried by clients or infrastructure should accept
 * this key to prevent duplicate creation, duplicate mutation, or repeated event emission.
 */
public interface IdempotencyKey {
}

/**
 * Represents distributed tracing information.
 * This context should be propagated across synchronous service calls and asynchronous
 * events so that requests can be traced end-to-end across the MSA.
 */
public interface TraceContext {
}

/**
 * Represents the generic result of a command operation.
 * This result should describe whether the command was accepted, completed, rejected,
 * or failed without exposing persistence implementation details.
 */
public interface CommandResult {
}

/**
 * Represents validation output.
 * This result should be used for existence checks, ownership checks, and policy checks
 * without forcing the caller to access another service's database directly.
 */
public interface ValidationResult {
}

/**
 * Represents a paginated query result.
 * This abstraction should include content, pagination metadata, and sorting metadata
 * without exposing internal database cursor or query implementation details.
 */
public interface PageResult<T> {
}

/**
 * Represents a standardized error result.
 * This object should carry an error code, user-safe message, technical reason,
 * and trace information for observability.
 */
public interface ErrorResult {
}

/**
 * Represents the result of handling an event.
 * Event handlers should use this result to indicate success, duplicate event detection,
 * retryable failure, non-retryable failure, or dead-letter routing.
 */
public interface EventHandlingResult {
}
```

---

# 2. Auth/User Service Skeleton

Auth/User Service는 사용자 식별, 인증, 권한만 담당하며, 다른 서비스는 User DB를 직접 조회하지 않고 JWT 클레임을 사용합니다. 

```java
public interface AuthUserApplicationService {

    /**
     * Registers a new user account.
     *
     * This function is responsible for validating the sign-up request,
     * creating a user identity, assigning the default role, and preparing
     * a UserCreated event if event publishing is enabled.
     *
     * This function must not create movie, collection, rating, watch status,
     * or ownership data because those responsibilities belong to other services.
     */
    AuthUserResult signUp(SignUpCommand command);

    /**
     * Authenticates a user using login credentials.
     *
     * This function verifies the provided credentials and returns access and
     * refresh tokens when authentication succeeds. It should not expose password
     * hashes or internal authentication storage details.
     */
    TokenResult login(LoginCommand command);

    /**
     * Issues a new access token using a valid refresh token.
     *
     * This function should validate token integrity, expiration, revocation state,
     * and user account status before returning a new token pair or access token.
     */
    TokenResult refreshToken(RefreshTokenCommand command);

    /**
     * Updates the authenticated user's own profile.
     *
     * This function allows a user to change profile-level information such as name.
     * It must enforce that the target user is the same as the authenticated user
     * represented by UserContext.
     */
    UserProfileResult updateMyProfile(UserContext userContext, UpdateProfileCommand command);

    /**
     * Changes the authenticated user's password.
     *
     * This function should validate the current password, apply password policy rules,
     * and complete the password update without exposing credential implementation details.
     */
    CommandResult changePassword(UserContext userContext, ChangePasswordCommand command);

    /**
     * Changes a user's role.
     *
     * This function is restricted to administrators. It should update authorization
     * metadata only and must not modify domain data owned by other services.
     */
    UserProfileResult changeUserRole(AdminContext adminContext, ChangeUserRoleCommand command);

    /**
     * Deactivates a user account.
     *
     * This function marks the user account as inactive or disabled. It should not
     * directly delete or mutate collection, rating, watch status, or ownership data
     * owned by other services.
     */
    CommandResult deactivateUser(AdminContext adminContext, DeactivateUserCommand command);

    /**
     * Returns the authenticated user's profile.
     *
     * This query should return identity and profile information only. It must not
     * aggregate movie collection, watch status, rating, or ownership information.
     */
    UserProfileResult getMyProfile(UserContext userContext);

    /**
     * Returns a user profile by userId for administrative use.
     *
     * This query is intended for administrator workflows and should enforce admin
     * authorization before returning user information.
     */
    UserProfileResult getUserById(AdminContext adminContext, String userId);

    /**
     * Searches users for administrative workflows.
     *
     * This query should support filtering and pagination for user management screens.
     * It must not be used by other domain services as a substitute for direct user
     * database access.
     */
    PageResult<UserSummary> searchUsers(
        AdminContext adminContext,
        UserSearchCondition condition,
        PaginationRequest pagination
    );

    /**
     * Validates an access token for gateway or internal service usage.
     *
     * This function should verify token validity and return identity claims that can
     * be propagated to downstream services. It should not return sensitive user data.
     */
    TokenValidationResult validateToken(AccessToken accessToken);
}
```

---

# 3. Movie Catalog Service Skeleton

Movie Catalog Service는 영화, 장르, 인물, 크레딧 등 **공통 영화 메타데이터**만 담당합니다. 사용자별 컬렉션, 평점, 시청 상태, 소장 위치는 포함하지 않습니다. 

```java
public interface MovieCatalogApplicationService {

    /**
     * Creates a new movie metadata record.
     *
     * This function is restricted to administrators and owns only global movie
     * catalog data such as title, release year, runtime, poster URL, genres,
     * and credits. It must not create user-specific collection, rating, watch
     * status, or ownership records.
     */
    MovieDetailResult createMovie(AdminContext adminContext, CreateMovieCommand command);

    /**
     * Updates an existing movie metadata record.
     *
     * This function modifies only catalog-owned movie attributes. If the update
     * affects read models or search indexes, a MovieUpdated event should be
     * prepared for asynchronous propagation.
     */
    MovieDetailResult updateMovie(
        AdminContext adminContext,
        String movieId,
        UpdateMovieCommand command
    );

    /**
     * Deletes or deactivates a movie metadata record.
     *
     * This function should define whether deletion is physical or logical.
     * It must not directly delete user collection, rating, watch status, or
     * ownership data from other services.
     */
    CommandResult deleteMovie(AdminContext adminContext, String movieId);

    /**
     * Adds one or more genres to a movie.
     *
     * This function modifies the relationship between a movie and catalog-owned
     * genre records. It should emit or prepare a MovieUpdated event when the
     * public movie representation changes.
     */
    MovieDetailResult addGenresToMovie(
        AdminContext adminContext,
        String movieId,
        GenreIdsCommand command
    );

    /**
     * Removes a genre from a movie.
     *
     * This function changes only catalog metadata. It must not affect user-specific
     * filtering preferences or collection data owned by other services.
     */
    MovieDetailResult removeGenreFromMovie(
        AdminContext adminContext,
        String movieId,
        String genreId
    );

    /**
     * Adds a cast or crew credit to a movie.
     *
     * This function manages the association between a movie and a person in the
     * catalog domain. It should support actors, directors, and other credit roles
     * without leaking user-specific behavior into the catalog service.
     */
    MovieDetailResult addCreditToMovie(
        AdminContext adminContext,
        String movieId,
        AddMovieCreditCommand command
    );

    /**
     * Removes a cast or crew credit from a movie.
     *
     * This function modifies only movie credit metadata and should not affect
     * user reviews, ratings, collections, or watch statuses.
     */
    MovieDetailResult removeCreditFromMovie(
        AdminContext adminContext,
        String movieId,
        String creditId
    );

    /**
     * Returns detailed movie metadata.
     *
     * This query should include catalog-owned information such as title, description,
     * genres, credits, runtime, and poster URL. It must not include user-specific
     * collection membership, rating, watch status, or ownership data.
     */
    MovieDetailResult getMovieDetail(String movieId);

    /**
     * Searches movies by catalog metadata.
     *
     * This query supports searching by keyword, genre, actor, director, release year,
     * and pagination. It should remain independent from user-specific library data.
     */
    PageResult<MovieSummary> searchMovies(
        MovieSearchCondition condition,
        PaginationRequest pagination
    );

    /**
     * Returns movie summaries for multiple movie IDs.
     *
     * This query is intended for internal API composition or query aggregation.
     * It should provide only catalog-owned summary data and must not join with
     * other service databases.
     */
    List<MovieSummary> getMoviesByIds(List<String> movieIds);

    /**
     * Checks whether a movie exists in the catalog.
     *
     * This function provides a lightweight validation contract for other services.
     * Callers may use this to validate movieId references without accessing the
     * catalog database directly.
     */
    ValidationResult existsMovie(String movieId);

    /**
     * Returns rating display data for a movie if the catalog chooses to expose it.
     *
     * This function is optional because rating summary ownership may remain in the
     * Rating Review Service or Query Aggregation Service depending on the chosen
     * consistency and ownership policy.
     */
    MovieRatingDisplayResult getMovieRatingDisplay(String movieId);
}
```

```java
public interface GenreApplicationService {

    /**
     * Creates a new genre in the movie catalog.
     *
     * This function is restricted to administrators and should maintain genre
     * uniqueness within the catalog boundary.
     */
    GenreResult createGenre(AdminContext adminContext, CreateGenreCommand command);

    /**
     * Updates an existing genre.
     *
     * This function changes catalog-owned genre metadata only and should trigger
     * read model synchronization if genre names are denormalized elsewhere.
     */
    GenreResult updateGenre(
        AdminContext adminContext,
        String genreId,
        UpdateGenreCommand command
    );

    /**
     * Deletes or deactivates a genre.
     *
     * This function must handle the catalog policy for movies already associated
     * with the genre without modifying data owned by other services.
     */
    CommandResult deleteGenre(AdminContext adminContext, String genreId);

    /**
     * Returns all available genres.
     *
     * This query is public or user-accessible and should be optimized for frequent
     * reads because genre data changes relatively infrequently.
     */
    List<GenreResult> getGenres();
}
```

```java
public interface PersonApplicationService {

    /**
     * Creates a new person record in the catalog.
     *
     * This function is used for actors, directors, and other contributors.
     * It must not create user-specific preferences or review data.
     */
    PersonResult createPerson(AdminContext adminContext, CreatePersonCommand command);

    /**
     * Updates catalog-owned person metadata.
     *
     * This function changes person profile data such as name or profile image
     * without touching movie reviews, collections, or watch status data.
     */
    PersonResult updatePerson(
        AdminContext adminContext,
        String personId,
        UpdatePersonCommand command
    );

    /**
     * Returns detailed information for a person.
     *
     * This query should include only catalog-owned person metadata and associated
     * catalog credits if required by the response contract.
     */
    PersonDetailResult getPersonDetail(String personId);

    /**
     * Searches people in the catalog.
     *
     * This query supports actor, director, or contributor search flows without
     * depending on user-specific services.
     */
    PageResult<PersonSummary> searchPersons(
        PersonSearchCondition condition,
        PaginationRequest pagination
    );
}
```

---

# 4. Collection Service Skeleton

Collection Service는 사용자가 특정 영화를 컬렉션에 담았는지와 컬렉션 메모만 책임집니다. Function Signature 설계서에서도 `CollectionItemResult`에는 영화 제목, 포스터, 장르를 포함하지 않고 `movieId` 중심으로 관리하도록 정의되어 있습니다. 

```java
public interface CollectionApplicationService {

    /**
     * Adds a movie to the authenticated user's collection.
     *
     * This command creates a user-owned collection item using only userId and movieId
     * references. It must not store movie title, poster URL, genres, or other catalog
     * metadata. Duplicate additions should be prevented by the userId and movieId
     * uniqueness rule.
     */
    CollectionItemResult addMovieToCollection(
        UserContext userContext,
        AddCollectionItemCommand command,
        IdempotencyKey idempotencyKey
    );

    /**
     * Updates the memo of an existing collection item.
     *
     * This command modifies only collection-owned user memo data. It must verify that
     * the collection item belongs to the authenticated user before applying the update.
     */
    CollectionItemResult updateCollectionMemo(
        UserContext userContext,
        String collectionItemId,
        UpdateCollectionMemoCommand command
    );

    /**
     * Removes a movie from the authenticated user's collection by collection item ID.
     *
     * This command removes or deactivates only the collection item. It must not delete
     * movie metadata, ratings, watch status, or ownership records from other services.
     */
    CommandResult removeMovieFromCollection(
        UserContext userContext,
        String collectionItemId
    );

    /**
     * Removes a movie from the authenticated user's collection by movie ID.
     *
     * This command is useful when the client knows the movieId but not the collection
     * item ID. It must enforce user ownership before removing the collection record.
     */
    CommandResult removeMovieFromCollectionByMovieId(
        UserContext userContext,
        String movieId
    );

    /**
     * Returns the authenticated user's collection items.
     *
     * This query returns collection-owned data only, such as collectionItemId, userId,
     * movieId, memo, and addedAt. It must not enrich results with catalog metadata.
     */
    PageResult<CollectionItemResult> getMyCollectionItems(
        UserContext userContext,
        PaginationRequest pagination
    );

    /**
     * Returns a single collection item by collection item ID.
     *
     * This query must verify that the requested collection item belongs to the
     * authenticated user. It should not call or join with the Movie Catalog Service
     * for display enrichment.
     */
    CollectionItemResult getCollectionItem(
        UserContext userContext,
        String collectionItemId
    );

    /**
     * Returns a collection item by movie ID for the authenticated user.
     *
     * This query is used to determine whether a given movie is included in the
     * user's collection without exposing catalog metadata.
     */
    CollectionItemResult getCollectionItemByMovieId(
        UserContext userContext,
        String movieId
    );

    /**
     * Checks whether the authenticated user's collection contains a movie.
     *
     * This function provides a lightweight validation contract for policy or
     * aggregation use cases. It must not become a mandatory dependency for watch
     * status, rating, or ownership commands unless a business policy explicitly
     * requires it.
     */
    ValidationResult existsInCollection(
        UserContext userContext,
        String movieId
    );

    /**
     * Returns collection items for multiple movie IDs.
     *
     * This query is intended for Query Aggregation Service composition. It should
     * return only collection-owned fields and should not perform cross-service joins.
     */
    List<CollectionItemResult> getCollectionItemsByMovieIds(
        UserContext userContext,
        List<String> movieIds
    );
}
```

---

# 5. Watch Status Service Skeleton

Watch Status Service는 `userId + movieId` 기준으로 시청 상태를 독립 관리하며, Collection Service에 강하게 의존하지 않는 구조입니다. 

```java
public interface WatchStatusApplicationService {

    /**
     * Sets or replaces the watch status for a movie owned by the authenticated user context.
     *
     * This command manages only the user's watch status for the given movieId.
     * It must not require the movie to exist in the user's collection unless a separate
     * business policy explicitly introduces that rule. This prevents unnecessary coupling
     * between Watch Status Service and Collection Service.
     */
    WatchStatusResult setWatchStatus(
        UserContext userContext,
        String movieId,
        SetWatchStatusCommand command
    );

    /**
     * Updates the watched date for a movie.
     *
     * This command modifies the watchedAt field independently from collection membership.
     * It should validate that the requested update is consistent with the current or
     * requested watch status policy.
     */
    WatchStatusResult updateWatchedAt(
        UserContext userContext,
        String movieId,
        UpdateWatchedAtCommand command
    );

    /**
     * Removes the watch status associated with a movie.
     *
     * This command deletes or deactivates only watch status data. It must not delete
     * the movie, collection item, rating, review, or ownership information.
     */
    CommandResult removeWatchStatus(
        UserContext userContext,
        String movieId
    );

    /**
     * Returns the authenticated user's watch statuses.
     *
     * This query supports filtering by status and watched date range. It should return
     * watch-status-owned data only and must not enrich results with movie metadata.
     */
    PageResult<WatchStatusResult> getMyWatchStatuses(
        UserContext userContext,
        WatchStatusSearchCondition condition,
        PaginationRequest pagination
    );

    /**
     * Returns the watch status for a specific movie.
     *
     * This query is scoped to the authenticated user and the provided movieId. It should
     * not require collection membership to return a result.
     */
    WatchStatusResult getWatchStatusByMovieId(
        UserContext userContext,
        String movieId
    );

    /**
     * Returns watch statuses for multiple movie IDs.
     *
     * This query is intended for Query Aggregation Service composition. It should be
     * optimized for bulk lookup and return only watch status data.
     */
    List<WatchStatusResult> getWatchStatusesByMovieIds(
        UserContext userContext,
        List<String> movieIds
    );
}
```

---

# 6. Rating Review Service Skeleton

Rating Review Service는 사용자 평점, 리뷰, 평균 평점 집계를 담당하고, 영화 제목·장르 등 Catalog 데이터를 소유하지 않습니다. 평균 평점은 이벤트 기반 갱신을 고려합니다. 

```java
public interface RatingReviewApplicationService {

    /**
     * Creates or updates the authenticated user's rating and review for a movie.
     *
     * This command owns only user rating and review data. It must not store movie
     * title, poster URL, genres, or collection membership. The service should enforce
     * the rule that a user can have only one rating or review per movie.
     */
    RatingReviewResult createOrUpdateRatingReview(
        UserContext userContext,
        String movieId,
        UpsertRatingReviewCommand command
    );

    /**
     * Updates the visibility of the authenticated user's review.
     *
     * This command changes only review visibility metadata, such as public or private
     * visibility. It must verify that the review belongs to the authenticated user.
     */
    RatingReviewResult updateReviewVisibility(
        UserContext userContext,
        String movieId,
        UpdateReviewVisibilityCommand command
    );

    /**
     * Deletes the authenticated user's rating and review for a movie.
     *
     * This command removes only rating-review-owned data. It must not remove the movie,
     * collection item, watch status, or ownership information.
     */
    CommandResult deleteRatingReview(
        UserContext userContext,
        String movieId
    );

    /**
     * Recalculates the rating summary for a movie.
     *
     * This function is intended for internal system use, event-driven processing,
     * or scheduled repair jobs. It should update aggregate rating data such as average
     * rating and rating count without accessing catalog-owned movie metadata.
     */
    MovieRatingSummaryResult recalculateMovieRatingSummary(
        SystemContext systemContext,
        String movieId
    );

    /**
     * Returns the authenticated user's rating and review for a movie.
     *
     * This query returns only user-specific rating and review data. Movie display
     * information should be provided by Movie Catalog Service or Query Aggregation Service.
     */
    RatingReviewResult getMyRatingReview(
        UserContext userContext,
        String movieId
    );

    /**
     * Returns the authenticated user's rating and review list.
     *
     * This query supports filtering by rating range, visibility, and creation date.
     * It must not include catalog metadata unless explicitly provided by a query
     * aggregation layer.
     */
    PageResult<RatingReviewResult> getMyRatingReviews(
        UserContext userContext,
        RatingReviewSearchCondition condition,
        PaginationRequest pagination
    );

    /**
     * Returns public reviews for a movie.
     *
     * This query exposes only reviews that are allowed by visibility policy. It should
     * avoid leaking private user data and should not require access to the Movie Catalog
     * database.
     */
    PageResult<PublicReviewResult> getPublicReviewsByMovieId(
        String movieId,
        PaginationRequest pagination
    );

    /**
     * Returns the aggregate rating summary for a movie.
     *
     * This query provides summary-level rating data, such as average rating and rating
     * count. It should be suitable for catalog display or query aggregation use cases.
     */
    MovieRatingSummaryResult getMovieRatingSummary(String movieId);

    /**
     * Returns the authenticated user's rating and review data for multiple movies.
     *
     * This query is intended for Query Aggregation Service composition and should return
     * only rating-review-owned data.
     */
    List<RatingReviewResult> getRatingReviewsByMovieIds(
        UserContext userContext,
        List<String> movieIds
    );
}
```

---

# 7. Ownership Service Skeleton

Ownership Service는 사용자의 소장 형태, 위치, 플랫폼을 관리하며, CollectionItem에 종속되지 않고 독립 관리합니다. 

```java
public interface OwnershipApplicationService {

    /**
     * Adds ownership information for a movie.
     *
     * This command records how and where the authenticated user owns or accessed a movie,
     * such as Blu-ray, DVD, digital file, streaming purchase, rental, or theater record.
     * It must not require a CollectionItem unless explicitly required by business policy.
     */
    OwnershipResult addOwnership(
        UserContext userContext,
        String movieId,
        AddOwnershipCommand command
    );

    /**
     * Updates an existing ownership record.
     *
     * This command modifies only ownership-owned fields such as mediaType, location,
     * platform, and purchaseDate. It must verify that the ownership record belongs
     * to the authenticated user.
     */
    OwnershipResult updateOwnership(
        UserContext userContext,
        String ownershipId,
        UpdateOwnershipCommand command
    );

    /**
     * Removes a single ownership record.
     *
     * This command deletes or deactivates only ownership data. It must not delete the
     * movie, collection item, watch status, rating, or review.
     */
    CommandResult removeOwnership(
        UserContext userContext,
        String ownershipId
    );

    /**
     * Removes ownership records associated with a specific movie.
     *
     * This command is scoped to the authenticated user and movieId. It should be used
     * when the user wants to clear all ownership records for a movie without affecting
     * other user-movie activity data.
     */
    CommandResult removeOwnershipsByMovieId(
        UserContext userContext,
        String movieId
    );

    /**
     * Returns ownership records owned by the authenticated user.
     *
     * This query supports filtering by media type, platform, and location keyword.
     * It must not include movie title, genre, or poster URL.
     */
    PageResult<OwnershipResult> getMyOwnerships(
        UserContext userContext,
        OwnershipSearchCondition condition,
        PaginationRequest pagination
    );

    /**
     * Returns ownership records for a specific movie.
     *
     * This query is scoped to the authenticated user and movieId. It supports the
     * policy that one movie may have multiple ownership records.
     */
    List<OwnershipResult> getOwnershipsByMovieId(
        UserContext userContext,
        String movieId
    );

    /**
     * Returns ownership records for multiple movie IDs.
     *
     * This query is intended for Query Aggregation Service composition and should
     * return only ownership-owned data.
     */
    List<OwnershipResult> getOwnershipsByMovieIds(
        UserContext userContext,
        List<String> movieIds
    );
}
```

---

# 8. Query Aggregation Service Skeleton

Query Aggregation Service는 통합 조회만 담당하고 원천 데이터를 수정하지 않습니다. Function Signature 설계서에서도 이 서비스는 내 라이브러리, 상세, 대시보드 조회와 다수 이벤트 구독을 담당하도록 정의되어 있습니다. 

```java
public interface QueryAggregationApplicationService {

    /**
     * Returns the authenticated user's integrated movie library.
     *
     * This query composes collection data, movie catalog data, watch status, rating,
     * review summary, and ownership data into a client-oriented read response.
     * It must not modify source-of-truth data in any domain service.
     */
    PageResult<MyLibraryItemResult> getMyLibrary(
        UserContext userContext,
        LibrarySearchCondition condition,
        PaginationRequest pagination
    );

    /**
     * Returns an integrated detail view for a single movie in the user's library context.
     *
     * This query may use API composition or a precomputed read model. It should clearly
     * distinguish source-owned data from derived read-model data and tolerate partial
     * downstream service failures when possible.
     */
    MyLibraryItemDetailResult getMyLibraryItemDetail(
        UserContext userContext,
        String movieId
    );

    /**
     * Returns dashboard-level summary information for the authenticated user.
     *
     * This query provides aggregate user-facing metrics such as collection count,
     * watched count, watching count, average personal rating, and recently added items.
     * It must not perform command-side mutations.
     */
    MyDashboardResult getMyDashboard(UserContext userContext);

    /**
     * Searches within the authenticated user's integrated library.
     *
     * This query is best served by a read model when filtering spans multiple domains,
     * such as keyword, genre, watch status, rating range, and ownership media type.
     */
    PageResult<MyLibraryItemResult> searchMyLibrary(
        UserContext userContext,
        LibrarySearchCondition condition,
        PaginationRequest pagination
    );

    /**
     * Returns recently added movies from the authenticated user's library.
     *
     * This query should be optimized for dashboard and home-screen usage. It should
     * not mutate collection records or catalog records.
     */
    List<MyLibraryItemResult> getRecentlyAddedMovies(
        UserContext userContext,
        int limit
    );

    /**
     * Returns a watch progress summary for the authenticated user.
     *
     * This query aggregates watch status counts by status type. It should be based
     * on either the Watch Status Service or a synchronized read model.
     */
    WatchProgressSummaryResult getWatchingProgressSummary(UserContext userContext);
}
```

```java
public interface QueryAggregationEventHandler {

    /**
     * Handles MovieCreated events.
     *
     * This handler should create or update the movie portion of the read model.
     * It must be idempotent and should not call command functions on the Movie
     * Catalog Service.
     */
    EventHandlingResult handleMovieCreated(MovieCreatedEvent event);

    /**
     * Handles MovieUpdated events.
     *
     * This handler should update denormalized movie display fields such as title,
     * release year, poster URL, and genres in the read model. It must tolerate
     * duplicate event delivery.
     */
    EventHandlingResult handleMovieUpdated(MovieUpdatedEvent event);

    /**
     * Handles MovieDeleted events.
     *
     * This handler should deactivate or mark movie-related read model entries as
     * unavailable without deleting source data from other services.
     */
    EventHandlingResult handleMovieDeleted(MovieDeletedEvent event);

    /**
     * Handles CollectionItemAdded events.
     *
     * This handler should create or update the user's library read model entry for
     * the given userId and movieId. It should not modify the source collection record.
     */
    EventHandlingResult handleCollectionItemAdded(CollectionItemAddedEvent event);

    /**
     * Handles CollectionItemRemoved events.
     *
     * This handler should remove or mark the corresponding library read model entry
     * as no longer part of the user's collection.
     */
    EventHandlingResult handleCollectionItemRemoved(CollectionItemRemovedEvent event);

    /**
     * Handles WatchStatusChanged events.
     *
     * This handler should update the watch status portion of the user's read model.
     * It must not require collection membership to process the event.
     */
    EventHandlingResult handleWatchStatusChanged(WatchStatusChangedEvent event);

    /**
     * Handles RatingReviewChanged events.
     *
     * This handler should update the user's personal rating and review summary in
     * the read model while respecting review visibility rules.
     */
    EventHandlingResult handleRatingReviewChanged(RatingReviewChangedEvent event);

    /**
     * Handles OwnershipChanged events.
     *
     * This handler should update the ownership section of the user's read model,
     * including media type, location, platform, and related display fields.
     */
    EventHandlingResult handleOwnershipChanged(OwnershipChangedEvent event);
}
```

---

# 9. API Gateway Skeleton

API Gateway는 외부 요청 진입점, JWT 검증, 라우팅, Rate Limit, 공통 오류 처리를 담당하며 비즈니스 로직과 DB 접근은 하지 않습니다. 

```java
public interface ApiGatewayService {

    /**
     * Authenticates an incoming HTTP request.
     *
     * This function validates the access token when present and creates either an
     * authenticated user context or an anonymous context. It should not perform
     * domain-specific business validation.
     */
    AuthenticationContext authenticateRequest(HttpRequest request);

    /**
     * Authorizes an authenticated request against route policy.
     *
     * This function checks whether the current identity has permission to access
     * the requested route. It should evaluate role and route policy only, not
     * domain ownership rules.
     */
    AuthorizationResult authorizeRequest(
        UserContext userContext,
        RoutePolicy routePolicy
    );

    /**
     * Routes the incoming request to the target downstream service.
     *
     * This function should forward identity, role, trace context, and request metadata
     * to downstream services. It must not execute domain business logic.
     */
    HttpResponse routeRequest(
        HttpRequest request,
        UserContext userContext
    );

    /**
     * Applies rate limiting based on client identity and route policy.
     *
     * This function protects downstream services from excessive traffic while keeping
     * rate limit rules outside domain service implementations.
     */
    RateLimitResult applyRateLimit(
        ClientIdentity clientIdentity,
        RoutePolicy routePolicy
    );

    /**
     * Enriches the request with distributed tracing metadata.
     *
     * This function creates or propagates traceId and spanId values so that downstream
     * service calls and asynchronous events can be correlated.
     */
    TraceContext enrichTraceContext(HttpRequest request);

    /**
     * Converts internal or downstream errors into standardized gateway responses.
     *
     * This function should normalize error structure and status codes without hiding
     * traceability information required for troubleshooting.
     */
    HttpResponse handleGatewayError(ErrorResult error);
}
```

---

# 10. Event Contract Skeleton

이벤트는 내부 DB 구조가 아니라 “상태 변경 사실”만 전달해야 하며, Query Aggregation Service 등이 이를 구독합니다. 

```java
/**
 * Represents the common envelope for all domain events.
 *
 * The envelope should carry event identity, event type, producer, occurrence time,
 * schema version, trace information, and a payload. It must not expose internal
 * database table structure or persistence-specific fields.
 */
public interface DomainEventEnvelope<TPayload> {
}

/**
 * Published when a user account is created.
 *
 * Consumers should use this event only for user-created side effects that do not
 * require direct access to the Auth/User Service database.
 */
public interface UserCreatedEvent {
}

/**
 * Published when a movie metadata record is created in the catalog.
 *
 * Consumers may use this event to create search indexes or read model entries.
 */
public interface MovieCreatedEvent {
}

/**
 * Published when movie metadata changes.
 *
 * Consumers may use this event to refresh denormalized movie display data in read
 * models. The event should identify changed fields without exposing database internals.
 */
public interface MovieUpdatedEvent {
}

/**
 * Published when a movie is deleted or deactivated in the catalog.
 *
 * Consumers should treat this event as a signal to hide, deactivate, or mark read
 * model entries as unavailable rather than directly deleting data they do not own.
 */
public interface MovieDeletedEvent {
}

/**
 * Published when a user adds a movie to their collection.
 *
 * Consumers may use this event to create or update user library read model entries.
 */
public interface CollectionItemAddedEvent {
}

/**
 * Published when a user's collection item is updated.
 *
 * Consumers may use this event to update collection-specific display data such as
 * user memo or added metadata in read models.
 */
public interface CollectionItemUpdatedEvent {
}

/**
 * Published when a movie is removed from a user's collection.
 *
 * Consumers may use this event to remove or deactivate the corresponding library
 * read model entry.
 */
public interface CollectionItemRemovedEvent {
}

/**
 * Published when a user's watch status changes.
 *
 * Consumers may use this event to update watch status projections without coupling
 * directly to the Watch Status Service database.
 */
public interface WatchStatusChangedEvent {
}

/**
 * Published when a user's rating or review changes.
 *
 * Consumers may use this event to update personal rating projections and review
 * summaries in read models.
 */
public interface RatingReviewChangedEvent {
}

/**
 * Published when a movie's rating summary changes.
 *
 * Consumers may use this event to refresh average rating and rating count display
 * fields in query models or catalog-facing views.
 */
public interface RatingSummaryUpdatedEvent {
}

/**
 * Published when a user's ownership information changes.
 *
 * Consumers may use this event to update ownership-related read model fields such
 * as media type, platform, and location.
 */
public interface OwnershipChangedEvent {
}

/**
 * Published when a user's ownership record is removed.
 *
 * Consumers may use this event to remove or deactivate ownership-related read model
 * fields without touching source ownership data.
 */
public interface OwnershipRemovedEvent {
}
```

---

# 11. Service Client / Port Skeleton

Service Client / Port는 서비스 간 직접 DB 접근을 막기 위한 계약입니다. Function Signature 설계서는 조회 조합을 Query Aggregation Service로 집중하고, 쓰기 서비스 간 의존을 최소화하도록 정의합니다. 

```java
public interface CatalogLookupPort {

    /**
     * Checks whether the given movieId exists in the Movie Catalog Service.
     *
     * This port allows other services to validate movie references without directly
     * querying the catalog database. Use of this port in command flows should remain
     * optional or policy-driven to avoid excessive synchronous coupling.
     */
    ValidationResult existsMovie(String movieId);

    /**
     * Retrieves movie summaries for multiple movie IDs.
     *
     * This port is primarily intended for Query Aggregation Service composition and
     * should return only catalog-owned summary fields.
     */
    List<MovieSummary> getMovieSummaries(List<String> movieIds);
}
```

```java
public interface CollectionLookupPort {

    /**
     * Retrieves collection items for a user and a list of movie IDs.
     *
     * This port supports query aggregation by returning collection-owned fields only.
     * It must not return catalog metadata such as title, poster URL, or genres.
     */
    List<CollectionItemResult> getCollectionItemsByMovieIds(
        String userId,
        List<String> movieIds
    );

    /**
     * Checks whether a movie exists in a user's collection.
     *
     * This port should be used carefully because making it mandatory for rating,
     * watch status, or ownership commands would increase service coupling.
     */
    ValidationResult existsInCollection(
        String userId,
        String movieId
    );
}
```

```java
public interface WatchStatusLookupPort {

    /**
     * Retrieves watch statuses for a user and a list of movie IDs.
     *
     * This port supports query aggregation and should return only watch-status-owned
     * fields such as status, watchedAt, and updatedAt.
     */
    List<WatchStatusResult> getWatchStatusesByMovieIds(
        String userId,
        List<String> movieIds
    );
}
```

```java
public interface RatingLookupPort {

    /**
     * Retrieves the authenticated user's rating and review data for multiple movies.
     *
     * This port supports integrated library views without requiring the Query
     * Aggregation Service to access the Rating Review Service database directly.
     */
    List<RatingReviewResult> getRatingReviewsByMovieIds(
        String userId,
        List<String> movieIds
    );

    /**
     * Retrieves rating summaries for multiple movies.
     *
     * This port provides aggregate rating display data such as average rating and
     * rating count. Ownership of this data should remain aligned with the rating
     * summary policy defined by the architecture.
     */
    List<MovieRatingSummaryResult> getMovieRatingSummaries(List<String> movieIds);
}
```

```java
public interface OwnershipLookupPort {

    /**
     * Retrieves ownership records for a user and a list of movie IDs.
     *
     * This port supports query aggregation and should return only ownership-owned
     * fields such as media type, platform, location, and purchase date.
     */
    List<OwnershipResult> getOwnershipsByMovieIds(
        String userId,
        List<String> movieIds
    );
}
```

---

# 12. Repository Interface Skeleton

Repository는 각 서비스 내부에서만 사용합니다. 다른 서비스는 Repository를 호출할 수 없습니다.

```java
public interface UserRepository {

    /**
     * Finds a user by userId within the Auth/User Service boundary.
     *
     * This repository must be used only inside the Auth/User Service and must not
     * be exposed as a cross-service access mechanism.
     */
    UserEntity findById(String userId);

    /**
     * Finds a user by email for authentication and account management workflows.
     *
     * This method must not expose password hashes or credential internals outside
     * the Auth/User Service.
     */
    UserEntity findByEmail(String email);

    /**
     * Saves a user aggregate within the Auth/User Service database.
     *
     * This method persists only user-owned identity and authorization data.
     */
    UserEntity save(UserEntity user);
}
```

```java
public interface MovieCatalogRepository {

    /**
     * Finds movie metadata by movieId within the Movie Catalog Service boundary.
     *
     * This repository must never be called by Collection, Rating, Watch Status,
     * Ownership, or Query Aggregation services directly.
     */
    MovieEntity findMovieById(String movieId);

    /**
     * Searches catalog-owned movie metadata.
     *
     * This method should query only catalog-owned tables or indexes.
     */
    PageResult<MovieEntity> searchMovies(
        MovieSearchCondition condition,
        PaginationRequest pagination
    );

    /**
     * Saves movie metadata owned by the Movie Catalog Service.
     *
     * This method must not persist user-specific activity or collection data.
     */
    MovieEntity saveMovie(MovieEntity movie);
}
```

```java
public interface CollectionRepository {

    /**
     * Finds collection items owned by a specific user.
     *
     * This repository should return only collection-owned records and must not join
     * with catalog, rating, watch status, or ownership databases.
     */
    PageResult<CollectionItemEntity> findByUserId(
        String userId,
        PaginationRequest pagination
    );

    /**
     * Finds a collection item by userId and movieId.
     *
     * This method supports collection membership checks within the Collection Service
     * boundary only.
     */
    CollectionItemEntity findByUserIdAndMovieId(
        String userId,
        String movieId
    );

    /**
     * Saves a collection item.
     *
     * This method persists only collection-owned fields such as userId, movieId,
     * memo, addedAt, and updatedAt.
     */
    CollectionItemEntity save(CollectionItemEntity collectionItem);
}
```

```java
public interface WatchStatusRepository {

    /**
     * Finds a watch status record by userId and movieId.
     *
     * This repository should remain internal to the Watch Status Service and must
     * not be exposed for direct cross-service access.
     */
    WatchStatusEntity findByUserIdAndMovieId(
        String userId,
        String movieId
    );

    /**
     * Finds watch statuses for a user according to search conditions.
     *
     * This method should filter only watch-status-owned data.
     */
    PageResult<WatchStatusEntity> findByUserId(
        String userId,
        WatchStatusSearchCondition condition,
        PaginationRequest pagination
    );

    /**
     * Saves a watch status record.
     *
     * This method persists only userId, movieId, status, watchedAt, and update metadata.
     */
    WatchStatusEntity save(WatchStatusEntity watchStatus);
}
```

```java
public interface RatingReviewRepository {

    /**
     * Finds a user's rating and review for a movie.
     *
     * This repository should be used only inside the Rating Review Service and must
     * not fetch movie title, genre, poster URL, or collection membership.
     */
    RatingReviewEntity findByUserIdAndMovieId(
        String userId,
        String movieId
    );

    /**
     * Finds public reviews for a movie.
     *
     * This method should apply visibility rules and return only review data that
     * is allowed for public or user-facing display.
     */
    PageResult<RatingReviewEntity> findPublicReviewsByMovieId(
        String movieId,
        PaginationRequest pagination
    );

    /**
     * Saves a rating and review record.
     *
     * This method persists only rating-review-owned fields and should support the
     * one-review-per-user-per-movie invariant.
     */
    RatingReviewEntity save(RatingReviewEntity ratingReview);
}
```

```java
public interface OwnershipRepository {

    /**
     * Finds ownership records for a user and movie.
     *
     * This repository should support multiple ownership records for the same movie,
     * such as Blu-ray and digital purchase, without depending on Collection Service.
     */
    List<OwnershipEntity> findByUserIdAndMovieId(
        String userId,
        String movieId
    );

    /**
     * Finds ownership records for a user according to search conditions.
     *
     * This method filters ownership-owned data such as media type, platform, and
     * location keyword.
     */
    PageResult<OwnershipEntity> findByUserId(
        String userId,
        OwnershipSearchCondition condition,
        PaginationRequest pagination
    );

    /**
     * Saves an ownership record.
     *
     * This method persists only ownership-owned fields and must not store movie
     * catalog metadata.
     */
    OwnershipEntity save(OwnershipEntity ownership);
}
```

---

# 13. Outbox / Event Publisher Skeleton

설계서에서는 이벤트 발행과 DB 트랜잭션 불일치를 막기 위해 Outbox Pattern을 사용하도록 되어 있습니다. 

```java
public interface OutboxEventRepository {

    /**
     * Stores a domain event in the local service outbox.
     *
     * This method should be called within the same transaction as the business data
     * change so that state mutation and event recording remain consistent.
     */
    OutboxEventEntity save(OutboxEventEntity event);

    /**
     * Finds unpublished outbox events.
     *
     * This method is used by an internal publisher process to locate events that
     * still need to be delivered to the event broker.
     */
    List<OutboxEventEntity> findUnpublishedEvents();

    /**
     * Marks an outbox event as published.
     *
     * This method records successful publication without modifying the original
     * business entity that produced the event.
     */
    CommandResult markAsPublished(String eventId);
}
```

```java
public interface DomainEventPublisher {

    /**
     * Publishes a domain event to the event broker.
     *
     * This function should publish the event envelope without exposing internal
     * database structure. Delivery should be compatible with retry and idempotency
     * requirements.
     */
    EventPublishResult publish(DomainEventEnvelope<?> eventEnvelope);

    /**
     * Publishes all pending outbox events.
     *
     * This function is intended for background publishing workflows. It should handle
     * retryable failures, non-retryable failures, and dead-letter routing according
     * to infrastructure policy.
     */
    EventPublishResult publishPendingOutboxEvents(SystemContext systemContext);
}
```

---

# 14. DTO / Result Name Skeleton

아래 타입들은 상세 구현 없이 이름만 계약으로 둡니다.

```java
public interface SignUpCommand {}
public interface LoginCommand {}
public interface RefreshTokenCommand {}
public interface UpdateProfileCommand {}
public interface ChangePasswordCommand {}
public interface ChangeUserRoleCommand {}
public interface DeactivateUserCommand {}

public interface AuthUserResult {}
public interface TokenResult {}
public interface UserProfileResult {}
public interface UserSummary {}
public interface UserSearchCondition {}
public interface TokenValidationResult {}
public interface AccessToken {}

public interface CreateMovieCommand {}
public interface UpdateMovieCommand {}
public interface GenreIdsCommand {}
public interface AddMovieCreditCommand {}
public interface MovieSearchCondition {}
public interface MovieSummary {}
public interface MovieDetailResult {}
public interface MovieRatingDisplayResult {}

public interface CreateGenreCommand {}
public interface UpdateGenreCommand {}
public interface GenreResult {}

public interface CreatePersonCommand {}
public interface UpdatePersonCommand {}
public interface PersonSearchCondition {}
public interface PersonResult {}
public interface PersonSummary {}
public interface PersonDetailResult {}

public interface AddCollectionItemCommand {}
public interface UpdateCollectionMemoCommand {}
public interface CollectionItemResult {}

public interface SetWatchStatusCommand {}
public interface UpdateWatchedAtCommand {}
public interface WatchStatusSearchCondition {}
public interface WatchStatusResult {}

public interface UpsertRatingReviewCommand {}
public interface UpdateReviewVisibilityCommand {}
public interface RatingReviewSearchCondition {}
public interface RatingReviewResult {}
public interface MovieRatingSummaryResult {}
public interface PublicReviewResult {}

public interface AddOwnershipCommand {}
public interface UpdateOwnershipCommand {}
public interface OwnershipSearchCondition {}
public interface OwnershipResult {}

public interface LibrarySearchCondition {}
public interface MyLibraryItemResult {}
public interface MyLibraryItemDetailResult {}
public interface MyDashboardResult {}
public interface WatchProgressSummaryResult {}

public interface HttpRequest {}
public interface HttpResponse {}
public interface AuthenticationContext {}
public interface AnonymousContext {}
public interface AuthorizationResult {}
public interface RoutePolicy {}
public interface ClientIdentity {}
public interface RateLimitResult {}

public interface UserEntity {}
public interface MovieEntity {}
public interface CollectionItemEntity {}
public interface WatchStatusEntity {}
public interface RatingReviewEntity {}
public interface OwnershipEntity {}
public interface OutboxEventEntity {}
```

---

# 핵심 정리

이번 Code Skeleton은 다음 의도를 유지합니다.

| 영역                   | Skeleton 설계 방향                   |
| -------------------- | -------------------------------- |
| Function Body        | 작성하지 않음                          |
| Description          | 각 함수에 영어 설명 추가                   |
| Service Boundary     | 서비스별 Application Service로 분리     |
| DB 접근                | Repository는 각 서비스 내부 전용          |
| Cross-Service Access | Port/Client 계약으로만 표현             |
| Query 조합             | Query Aggregation Service에 집중    |
| Event                | Event Handler와 Event Contract 분리 |
| Outbox               | 이벤트 발행 안정성을 위한 별도 Skeleton 제공    |

이 상태에서 다음 단계로 넘어가면, 각 서비스별로 `Controller Skeleton`, `DTO Field Skeleton`, `Entity Skeleton`, `Repository Skeleton`, `Event Handler Skeleton`을 실제 프로젝트 디렉터리 구조에 맞춰 세분화할 수 있습니다.
