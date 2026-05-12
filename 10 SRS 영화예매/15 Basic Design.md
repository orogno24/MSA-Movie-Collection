# 온라인 영화 예매 시스템 MSA 기본 설계서

## 0. 설계 범위

이번 단계는 **1. 기본 설계**입니다.
따라서 아직 함수 Signature나 Code Skeleton은 작성하지 않고, 다음 항목에 집중합니다.

```text
1. 서비스 경계 정의
2. 서비스별 책임 정의
3. 데이터 소유권 정의
4. 서비스 간 통신 방식 정의
5. 예매/좌석/결제 트랜잭션 경계 정의
6. 이벤트 기반 처리 구조 정의
7. 장애 대응 기본 전략 정의
8. 인프라 구성 방향 정의
```

---

# 1. 전체 아키텍처 개요

온라인 영화 예매 시스템은 다음과 같은 구조를 가진다.

```text
[Client]
  ├─ Web
  └─ Mobile App

      ↓

[API Gateway / BFF]
  ├─ 인증 토큰 검증
  ├─ 라우팅
  ├─ Rate Limiting
  ├─ API Aggregation
  └─ 공통 로깅

      ↓

[Microservices]
  ├─ Auth Service
  ├─ User Service
  ├─ Movie Service
  ├─ Theater Service
  ├─ Screening Service
  ├─ Seat Inventory Service
  ├─ Reservation Service
  ├─ Payment Service
  ├─ Ticket Service
  ├─ Benefit Service
  ├─ Notification Service
  ├─ Statistics Service
  └─ Admin Service

      ↓

[Infrastructure]
  ├─ Redis
  ├─ Message Broker
  ├─ Database per Service
  ├─ Config Server
  ├─ Service Discovery
  ├─ Observability
  └─ CI/CD
```

---

# 2. 핵심 설계 원칙

## 2.1 서비스별 데이터 소유권 분리

각 서비스는 자신의 DB만 직접 접근한다.

```text
Reservation Service → Reservation DB만 접근
Payment Service     → Payment DB만 접근
Seat Service        → Seat DB / Redis만 접근
```

다른 서비스의 DB를 직접 조회하거나 수정하지 않는다.

예를 들어 Reservation Service가 좌석 상태를 직접 변경하지 않는다.
좌석 상태 변경은 반드시 Seat Inventory Service API 또는 이벤트 처리로 수행한다.

---

## 2.2 동기 호출과 비동기 이벤트 분리

조회성 요청이나 즉시 응답이 필요한 요청은 동기 API로 처리한다.

```text
영화 조회
상영 일정 조회
좌석 현황 조회
좌석 선점 요청
예매 생성 요청
결제 요청
```

상태 변경 후 후속 처리는 비동기 이벤트로 처리한다.

```text
결제 승인 완료
예매 확정 완료
티켓 발급 완료
예매 취소 완료
환불 완료
알림 발송 요청
통계 반영
```

---

## 2.3 강한 정합성과 최종적 정합성 구분

| 영역       | 정합성 수준     | 이유                     |
| -------- | ---------- | ---------------------- |
| 좌석 선점    | 강한 정합성     | 같은 좌석이 동시에 선점되면 안 됨    |
| 좌석 예약 확정 | 강한 정합성     | 같은 좌석이 중복 예매되면 안 됨     |
| 예매-결제 상태 | 높은 정합성     | 결제 성공 후 예매 누락 방지 필요    |
| 티켓 발급    | 최종적 정합성 가능 | 재처리 가능                 |
| 알림       | 최종적 정합성 가능 | 실패해도 예매 자체는 유지         |
| 통계       | 최종적 정합성 가능 | 실시간 정합성보다 누락 없는 반영이 중요 |

---

# 3. Bounded Context 기준 서비스 분리

## 3.1 서비스 분리 요약

| 서비스                    | Bounded Context | 핵심 책임                             | 데이터 소유권         |
| ---------------------- | --------------- | --------------------------------- | --------------- |
| Auth Service           | 인증/인가           | 로그인, 토큰 발급, 권한 검증                 | 계정 인증 정보, 토큰/세션 |
| User Service           | 사용자             | 회원 프로필, 사용자 상태                    | 회원 프로필          |
| Movie Service          | 영화 카탈로그         | 영화 정보 관리/조회                       | 영화, 장르, 출연진     |
| Theater Service        | 극장/상영관          | 극장, 상영관, 기본 좌석 구조 관리              | 극장, 상영관, 좌석 템플릿 |
| Screening Service      | 상영 일정           | 영화별/극장별 상영 일정 관리                  | 상영 일정, 상영 가격    |
| Seat Inventory Service | 상영 회차별 좌석 재고    | 좌석 조회, 임시 선점, 예약 확정, 좌석 해제        | 상영 회차별 좌석 상태    |
| Reservation Service    | 예매              | 예매 생성, 상태 관리, 취소 처리, Saga 오케스트레이션 | 예매, 예매 좌석 스냅샷   |
| Payment Service        | 결제              | 결제 요청, 승인, 취소, 환불, PG 연동          | 결제, 환불 거래       |
| Ticket Service         | 티켓              | 모바일 티켓/QR 발급, 티켓 무효화              | 티켓              |
| Benefit Service        | 쿠폰/포인트          | 쿠폰 적용, 포인트 사용/복구                  | 쿠폰, 포인트         |
| Notification Service   | 알림              | 이메일/SMS/푸시 발송                     | 알림 발송 이력        |
| Statistics Service     | 통계/정산           | 예매/매출/환불 통계 생성                    | 통계 Read Model   |
| Admin Service          | 관리자 백오피스        | 관리자용 API 조합, 감사 로그                | 관리자 계정, 감사 로그   |

---

# 4. 서비스별 기본 설계

---

## 4.1 Auth Service

### 책임

* 로그인
* 로그아웃
* Access Token / Refresh Token 발급
* 토큰 검증
* 사용자 권한 확인
* 관리자 권한 확인

### 소유 데이터

```text
auth_user
refresh_token
login_history
role
permission
```

### 다른 서비스와의 경계

Auth Service는 인증과 권한만 담당한다.
회원 이름, 휴대폰 번호, 이메일 프로필 같은 정보는 User Service가 소유한다.

### 제공 기능

```text
회원 로그인
관리자 로그인
토큰 갱신
토큰 검증
권한 검증
```

### 발행 이벤트

```text
UserLoggedInEvent
UserLoggedOutEvent
AuthFailedEvent
```

---

## 4.2 User Service

### 책임

* 회원 프로필 관리
* 회원 상태 관리
* 마이페이지 기본 정보 제공
* 회원 탈퇴 처리

### 소유 데이터

```text
user
user_profile
user_status
user_preference
```

### 다른 서비스와의 경계

User Service는 예매 내역을 직접 소유하지 않는다.
예매 내역은 Reservation Service가 소유한다.

User Service는 결제 내역을 직접 소유하지 않는다.
결제 내역은 Payment Service가 소유한다.

### 제공 기능

```text
회원가입
회원 정보 조회
회원 정보 수정
회원 탈퇴
사용자 상태 조회
```

### 발행 이벤트

```text
UserCreatedEvent
UserUpdatedEvent
UserDeletedEvent
```

---

## 4.3 Movie Service

### 책임

* 영화 정보 등록
* 영화 정보 수정
* 영화 목록 조회
* 영화 상세 조회
* 영화 검색
* 현재 상영작/상영 예정작 관리

### 소유 데이터

```text
movie
movie_genre
movie_cast
movie_director
movie_rating
movie_media
```

### 다른 서비스와의 경계

Movie Service는 상영 시간표를 소유하지 않는다.
상영 일정은 Screening Service가 소유한다.

Movie Service는 영화 정보만 소유한다.

### 제공 기능

```text
영화 목록 조회
영화 상세 조회
영화 검색
관리자 영화 등록
관리자 영화 수정
관리자 영화 비활성화
```

### 발행 이벤트

```text
MovieCreatedEvent
MovieUpdatedEvent
MovieDeactivatedEvent
```

---

## 4.4 Theater Service

### 책임

* 극장 정보 관리
* 상영관 정보 관리
* 상영관 기본 좌석 구조 관리
* 좌석 템플릿 관리

### 소유 데이터

```text
theater
auditorium
seat_template
seat_layout
```

### 중요한 경계

Theater Service가 소유하는 좌석은 **상영관의 물리적 좌석 구조**다.

예를 들어 다음 정보는 Theater Service 소유다.

```text
A1 좌석이 존재한다.
A열은 12석이다.
3관은 총 120석이다.
A1은 장애인석이다.
B5는 프리미엄석이다.
```

하지만 특정 상영 일정에서 A1이 예약되었는지는 Theater Service가 관리하지 않는다.

그 정보는 Seat Inventory Service가 관리한다.

### 제공 기능

```text
극장 목록 조회
극장 상세 조회
상영관 조회
상영관 좌석 구조 조회
극장 등록
상영관 등록
좌석 구조 등록
좌석 구조 수정
```

### 발행 이벤트

```text
TheaterCreatedEvent
AuditoriumCreatedEvent
SeatLayoutUpdatedEvent
```

---

## 4.5 Screening Service

### 책임

* 상영 일정 관리
* 영화, 극장, 상영관, 시간 연결
* 상영 가격 관리
* 상영 일정 중복 검증
* 상영 일정 취소

### 소유 데이터

```text
screening
screening_time
screening_price
screening_status
```

### 중요한 경계

Screening Service는 상영 일정 정보를 소유한다.

예를 들어 다음 정보는 Screening Service 소유다.

```text
영화 1번이 2026-05-20 14:00에 강남점 3관에서 상영된다.
상영 시작 시간은 14:00이다.
상영 종료 시간은 16:10이다.
기본 가격은 15,000원이다.
```

하지만 해당 상영 일정의 좌석별 예약 상태는 소유하지 않는다.
좌석 상태는 Seat Inventory Service가 소유한다.

### 제공 기능

```text
영화 기준 상영 일정 조회
극장 기준 상영 일정 조회
날짜 기준 상영 일정 조회
상영 일정 상세 조회
상영 일정 등록
상영 일정 수정
상영 일정 취소
```

### 발행 이벤트

```text
ScreeningCreatedEvent
ScreeningUpdatedEvent
ScreeningCancelledEvent
```

### 구독 이벤트

```text
MovieDeactivatedEvent
SeatLayoutUpdatedEvent
```

---

## 4.6 Seat Inventory Service

### 책임

* 특정 상영 일정의 좌석 상태 관리
* 좌석 현황 조회
* 좌석 임시 선점
* 좌석 선점 만료
* 좌석 예약 확정
* 좌석 해제
* 좌석 중복 예매 방지

### 소유 데이터

```text
screening_seat
seat_hold
seat_reservation
seat_state_history
```

### Redis 소유 데이터

```text
seat_hold:{screeningId}:{seatId}
seat_hold_group:{holdId}
```

### 중요한 경계

Seat Inventory Service는 **상영 회차별 좌석 재고**를 소유한다.

예를 들어 다음 정보는 Seat Inventory Service 소유다.

```text
상영 일정 100번의 A1 좌석은 AVAILABLE이다.
상영 일정 100번의 A2 좌석은 HELD 상태이다.
상영 일정 100번의 A3 좌석은 RESERVED 상태이다.
A2 좌석은 userId=10이 5분간 선점했다.
```

Reservation Service는 좌석 상태를 직접 변경하지 않는다.
반드시 Seat Inventory Service에 요청해야 한다.

### 좌석 상태

```text
AVAILABLE
HELD
RESERVED
BLOCKED
```

### 제공 기능

```text
상영 일정별 좌석 현황 조회
좌석 임시 선점
좌석 선점 연장
좌석 선점 해제
좌석 예약 확정
예매 취소에 따른 좌석 해제
관리자 좌석 차단
```

### 발행 이벤트

```text
SeatHeldEvent
SeatHoldFailedEvent
SeatHoldExpiredEvent
SeatReservedEvent
SeatReleasedEvent
SeatBlockedEvent
```

### 구독 이벤트

```text
ScreeningCreatedEvent
ReservationConfirmedEvent
ReservationCancelledEvent
ReservationExpiredEvent
```

### 정합성 전략

좌석 중복 예매 방지는 이 서비스의 핵심 책임이다.

기본 전략은 다음과 같다.

```text
1. Redis SET NX EX 기반 임시 선점
2. screeningId + seatId 기준 원자적 선점
3. DB unique constraint로 최종 RESERVED 중복 방지
4. 예약 확정 시 Redis hold 검증
5. 만료된 hold는 TTL 또는 스케줄러로 정리
```

---

## 4.7 Reservation Service

### 책임

* 예매 생성
* 예매 상태 관리
* 예매 상세 조회
* 예매 목록 조회
* 예매 취소
* 예매 만료 처리
* 예매 Saga 오케스트레이션

### 소유 데이터

```text
reservation
reservation_seat_snapshot
reservation_status_history
reservation_idempotency
```

### 중요한 경계

Reservation Service는 예매 정보를 소유한다.

예를 들어 다음 정보는 Reservation Service 소유다.

```text
사용자 10번이 상영 일정 100번에 대해 예매를 생성했다.
예매 상태는 PENDING_PAYMENT이다.
예매 좌석은 A1, A2이다.
예매 만료 시간은 2026-05-12 12:10이다.
```

하지만 실제 좌석 상태의 원천 데이터는 Seat Inventory Service가 소유한다.
결제 상태의 원천 데이터는 Payment Service가 소유한다.
티켓 상태의 원천 데이터는 Ticket Service가 소유한다.

Reservation Service는 프로세스의 중심이므로 Saga Orchestrator 역할을 가진다.

### 예매 상태

```text
CREATED
PENDING_PAYMENT
CONFIRMED
CANCELLED
EXPIRED
FAILED
```

### 제공 기능

```text
예매 생성
예매 상세 조회
예매 목록 조회
예매 취소
예매 만료 처리
```

### 발행 이벤트

```text
ReservationCreatedEvent
ReservationPendingPaymentEvent
ReservationConfirmedEvent
ReservationCancelledEvent
ReservationExpiredEvent
ReservationFailedEvent
```

### 구독 이벤트

```text
PaymentApprovedEvent
PaymentFailedEvent
PaymentCancelledEvent
PaymentRefundedEvent
SeatHoldExpiredEvent
TicketIssuedEvent
```

### Saga 책임

Reservation Service는 다음 흐름을 관리한다.

```text
좌석 선점 확인
→ 예매 생성
→ 결제 요청
→ 결제 성공 이벤트 수신
→ 예매 확정
→ 좌석 예약 확정 요청
→ 티켓 발급 이벤트 발행
```

---

## 4.8 Payment Service

### 책임

* 결제 요청 생성
* 결제 승인
* 결제 실패 처리
* 결제 취소
* 환불 요청
* PG 연동
* 결제 중복 방지

### 소유 데이터

```text
payment
payment_transaction
payment_refund
payment_status_history
pg_request_log
pg_callback_log
```

### 중요한 경계

Payment Service는 결제 상태를 소유한다.

예를 들어 다음 정보는 Payment Service 소유다.

```text
예매 1000번의 결제 요청이 생성되었다.
결제 금액은 32,000원이다.
PG 거래 ID는 pg_tx_123이다.
결제 상태는 APPROVED이다.
```

Payment Service는 예매 상태를 직접 변경하지 않는다.
결제 성공/실패 이벤트를 발행하고 Reservation Service가 예매 상태를 변경한다.

### 결제 상태

```text
READY
REQUESTED
APPROVED
FAILED
CANCELLED
REFUNDED
```

### 제공 기능

```text
결제 요청
결제 승인 콜백 처리
결제 상태 조회
결제 취소
환불 요청
환불 상태 조회
```

### 발행 이벤트

```text
PaymentRequestedEvent
PaymentApprovedEvent
PaymentFailedEvent
PaymentCancelledEvent
PaymentRefundRequestedEvent
PaymentRefundedEvent
PaymentRefundFailedEvent
```

### 구독 이벤트

```text
ReservationCreatedEvent
ReservationCancelledEvent
ReservationExpiredEvent
```

### 외부 의존성

```text
PG Provider
간편 결제 Provider
카드 결제 Provider
```

### 정합성 전략

```text
1. reservationId 기준 결제 중복 방지
2. idempotencyKey 기반 결제 요청 중복 방지
3. PG callback 중복 수신 대비
4. 결제 승인 이벤트 중복 발행 가능성을 전제로 consumer idempotency 필요
5. 결제 성공 후 예매 확정 실패 시 재처리 가능해야 함
```

---

## 4.9 Ticket Service

### 책임

* 티켓 발급
* QR 코드 생성
* 모바일 티켓 조회
* 티켓 사용 처리
* 예매 취소 시 티켓 무효화

### 소유 데이터

```text
ticket
ticket_qr
ticket_status_history
ticket_usage_log
```

### 중요한 경계

Ticket Service는 티켓의 생명주기를 소유한다.

예매가 확정되었다고 해서 Reservation Service가 티켓 데이터를 직접 만들지 않는다.
ReservationConfirmedEvent를 기반으로 Ticket Service가 티켓을 발급한다.

### 티켓 상태

```text
ISSUED
USED
CANCELLED
EXPIRED
```

### 제공 기능

```text
티켓 조회
QR 티켓 조회
티켓 사용 처리
티켓 무효화
```

### 발행 이벤트

```text
TicketIssuedEvent
TicketIssueFailedEvent
TicketUsedEvent
TicketCancelledEvent
```

### 구독 이벤트

```text
ReservationConfirmedEvent
ReservationCancelledEvent
ReservationExpiredEvent
```

---

## 4.10 Benefit Service

### 책임

* 쿠폰 조회
* 쿠폰 적용 가능 여부 검증
* 쿠폰 사용 처리
* 쿠폰 사용 취소
* 포인트 조회
* 포인트 사용
* 포인트 복구
* 포인트 적립

### 소유 데이터

```text
coupon
user_coupon
point_wallet
point_transaction
benefit_usage
```

### 중요한 경계

Benefit Service는 할인/혜택의 원천 데이터를 소유한다.

Reservation Service는 쿠폰을 직접 사용 처리하지 않는다.
Payment Service도 포인트를 직접 차감하지 않는다.

할인 계산 결과는 예매/결제 시점에 스냅샷으로 저장할 수 있지만, 원천 데이터는 Benefit Service에 있다.

### 제공 기능

```text
사용 가능 쿠폰 조회
쿠폰 적용 검증
쿠폰 사용 예약
쿠폰 사용 확정
쿠폰 사용 취소
포인트 사용 예약
포인트 사용 확정
포인트 복구
```

### 발행 이벤트

```text
CouponReservedEvent
CouponUsedEvent
CouponReleasedEvent
PointReservedEvent
PointUsedEvent
PointRestoredEvent
```

### 구독 이벤트

```text
ReservationCreatedEvent
PaymentApprovedEvent
PaymentFailedEvent
ReservationCancelledEvent
ReservationExpiredEvent
```

---

## 4.11 Notification Service

### 책임

* 예매 완료 알림
* 결제 실패 알림
* 예매 취소 알림
* 환불 완료 알림
* 상영 일정 변경 알림
* 이메일/SMS/푸시 발송

### 소유 데이터

```text
notification
notification_template
notification_delivery_log
notification_failure_log
```

### 중요한 경계

Notification Service 장애가 예매/결제 흐름을 실패시키면 안 된다.

따라서 Notification Service는 비동기 이벤트 기반으로 동작한다.

### 제공 기능

```text
알림 발송 요청
알림 발송 이력 조회
알림 템플릿 관리
재발송
```

### 구독 이벤트

```text
ReservationConfirmedEvent
ReservationCancelledEvent
PaymentFailedEvent
PaymentRefundedEvent
ScreeningCancelledEvent
TicketIssuedEvent
```

### 발행 이벤트

```text
NotificationSentEvent
NotificationFailedEvent
```

---

## 4.12 Statistics Service

### 책임

* 예매 통계 생성
* 매출 통계 생성
* 환불 통계 생성
* 영화별/극장별/상영 일정별 집계
* 정산용 데이터 생성

### 소유 데이터

```text
reservation_statistics
sales_statistics
refund_statistics
screening_statistics
settlement_summary
```

### 중요한 경계

Statistics Service는 원천 데이터를 소유하지 않는다.

예매 원천은 Reservation Service, 결제 원천은 Payment Service, 상영 원천은 Screening Service에 있다.

Statistics Service는 이벤트를 구독해 통계용 Read Model을 생성한다.

### 구독 이벤트

```text
ReservationConfirmedEvent
ReservationCancelledEvent
PaymentApprovedEvent
PaymentRefundedEvent
ScreeningCreatedEvent
ScreeningCancelledEvent
```

---

## 4.13 Admin Service

### 책임

* 관리자 백오피스 API 제공
* 관리자 인증 연계
* 여러 도메인 서비스 API 조합
* 관리자 감사 로그 관리
* 운영자 수동 처리 화면 지원

### 소유 데이터

```text
admin_account
admin_role
admin_audit_log
admin_operation_log
```

### 중요한 경계

Admin Service는 다른 도메인의 원천 데이터를 직접 소유하지 않는다.

예를 들어 관리자가 영화를 등록하더라도 Admin Service DB에 영화가 저장되는 것이 아니다.

```text
관리자 요청
→ Admin Service
→ Movie Service API 호출
→ Movie Service DB에 저장
```

Admin Service는 백오피스용 진입점이자 조합 계층이다.

---

# 5. 데이터 소유권 설계

## 5.1 서비스별 DB 분리

```text
Auth Service DB
User Service DB
Movie Service DB
Theater Service DB
Screening Service DB
Seat Inventory Service DB
Reservation Service DB
Payment Service DB
Ticket Service DB
Benefit Service DB
Notification Service DB
Statistics Service DB
Admin Service DB
```

---

## 5.2 데이터 소유권 표

| 데이터          | 소유 서비스                 | 다른 서비스 접근 방식           |
| ------------ | ---------------------- | ---------------------- |
| 인증 정보        | Auth Service           | Auth API               |
| 회원 프로필       | User Service           | User API               |
| 영화 정보        | Movie Service          | Movie API / 이벤트 복제     |
| 극장 정보        | Theater Service        | Theater API / 이벤트 복제   |
| 상영관 좌석 구조    | Theater Service        | Theater API            |
| 상영 일정        | Screening Service      | Screening API / 이벤트 복제 |
| 상영 회차별 좌석 상태 | Seat Inventory Service | Seat API               |
| 예매           | Reservation Service    | Reservation API / 이벤트  |
| 결제           | Payment Service        | Payment API / 이벤트      |
| 티켓           | Ticket Service         | Ticket API / 이벤트       |
| 쿠폰           | Benefit Service        | Benefit API / 이벤트      |
| 포인트          | Benefit Service        | Benefit API / 이벤트      |
| 알림 이력        | Notification Service   | Notification API       |
| 통계 데이터       | Statistics Service     | Statistics API         |
| 감사 로그        | Admin Service          | Admin API              |

---

# 6. 핵심 도메인 경계 상세

## 6.1 Theater Service와 Seat Inventory Service 경계

이 둘은 헷갈리기 쉬우므로 명확히 분리한다.

### Theater Service

극장과 물리 좌석 구조를 관리한다.

```text
강남점
3관
A1 좌석
A2 좌석
B1 좌석
프리미엄석 여부
장애인석 여부
```

### Seat Inventory Service

상영 회차별 좌석 상태를 관리한다.

```text
상영 일정 100번의 A1 = AVAILABLE
상영 일정 100번의 A2 = HELD
상영 일정 100번의 A3 = RESERVED
상영 일정 101번의 A1 = RESERVED
```

즉, 같은 A1 좌석이라도 상영 일정마다 상태가 다르다.

```text
강남점 3관 A1
  ├─ 14:00 상영에서는 RESERVED
  ├─ 17:00 상영에서는 AVAILABLE
  └─ 20:00 상영에서는 HELD
```

따라서 상영 회차별 좌석 상태는 반드시 Seat Inventory Service가 소유한다.

---

## 6.2 Screening Service와 Reservation Service 경계

### Screening Service

상영 일정 자체를 관리한다.

```text
어떤 영화가
어느 극장
어느 상영관에서
몇 시에
얼마에
상영되는가
```

### Reservation Service

사용자의 예매 행위를 관리한다.

```text
어떤 사용자가
어떤 상영 일정을
어떤 좌석으로
어떤 상태로
예매했는가
```

Reservation Service는 상영 일정 정보를 직접 수정하지 않는다.

---

## 6.3 Reservation Service와 Payment Service 경계

### Reservation Service

예매 상태를 관리한다.

```text
PENDING_PAYMENT
CONFIRMED
CANCELLED
EXPIRED
FAILED
```

### Payment Service

결제 상태를 관리한다.

```text
READY
REQUESTED
APPROVED
FAILED
CANCELLED
REFUNDED
```

결제 성공이 곧바로 예매 DB 업데이트를 의미하지 않는다.

```text
Payment Service
  → PaymentApprovedEvent 발행

Reservation Service
  → PaymentApprovedEvent 수신
  → 예매 상태 CONFIRMED로 변경
```

---

## 6.4 Reservation Service와 Ticket Service 경계

Reservation Service는 예매 확정을 관리한다.
Ticket Service는 티켓 발급을 관리한다.

```text
ReservationConfirmedEvent
  → Ticket Service 수신
  → Ticket 생성
  → TicketIssuedEvent 발행
```

예매가 취소되면 티켓은 무효화된다.

```text
ReservationCancelledEvent
  → Ticket Service 수신
  → Ticket 상태 CANCELLED
```

---

# 7. 서비스 간 통신 설계

## 7.1 동기 통신

동기 API는 사용자가 즉시 결과를 알아야 하는 경우에 사용한다.

| 호출 주체               | 대상 서비스                 | 목적           |
| ------------------- | ---------------------- | ------------ |
| Client/API Gateway  | Movie Service          | 영화 목록/상세 조회  |
| Client/API Gateway  | Theater Service        | 극장/상영관 조회    |
| Client/API Gateway  | Screening Service      | 상영 일정 조회     |
| Client/API Gateway  | Seat Inventory Service | 좌석 현황 조회     |
| Client/API Gateway  | Seat Inventory Service | 좌석 임시 선점     |
| Client/API Gateway  | Reservation Service    | 예매 생성        |
| Reservation Service | Seat Inventory Service | 좌석 선점 검증     |
| Reservation Service | Screening Service      | 상영 일정 유효성 검증 |
| Reservation Service | Benefit Service        | 쿠폰/포인트 검증    |
| Client/API Gateway  | Payment Service        | 결제 요청        |
| Client/API Gateway  | Reservation Service    | 예매 조회/취소     |

---

## 7.2 비동기 통신

비동기 이벤트는 상태 변경 후 후속 처리를 위해 사용한다.

| 이벤트                       | 발행 서비스              | 구독 서비스                                                                                            |
| ------------------------- | ------------------- | ------------------------------------------------------------------------------------------------- |
| ReservationCreatedEvent   | Reservation Service | Payment Service, Benefit Service, Statistics Service                                              |
| PaymentApprovedEvent      | Payment Service     | Reservation Service                                                                               |
| PaymentFailedEvent        | Payment Service     | Reservation Service, Notification Service                                                         |
| ReservationConfirmedEvent | Reservation Service | Seat Inventory Service, Ticket Service, Notification Service, Statistics Service                  |
| ReservationCancelledEvent | Reservation Service | Seat Inventory Service, Payment Service, Ticket Service, Notification Service, Statistics Service |
| ReservationExpiredEvent   | Reservation Service | Seat Inventory Service, Benefit Service                                                           |
| PaymentRefundedEvent      | Payment Service     | Reservation Service, Notification Service, Statistics Service                                     |
| TicketIssuedEvent         | Ticket Service      | Notification Service                                                                              |
| ScreeningCancelledEvent   | Screening Service   | Reservation Service, Notification Service, Statistics Service                                     |

---

# 8. 핵심 예매 흐름 설계

## 8.1 정상 예매 흐름

```text
1. 사용자가 영화/극장/날짜 기준으로 상영 일정을 조회한다.
2. 사용자가 특정 상영 일정의 좌석 현황을 조회한다.
3. 사용자가 좌석을 선택한다.
4. Seat Inventory Service가 좌석을 임시 선점한다.
5. Reservation Service가 선점 정보를 검증하고 예매를 생성한다.
6. 예매 상태는 PENDING_PAYMENT가 된다.
7. 사용자가 결제를 요청한다.
8. Payment Service가 PG 결제를 진행한다.
9. 결제 성공 시 PaymentApprovedEvent가 발행된다.
10. Reservation Service가 이벤트를 수신하고 예매를 CONFIRMED로 변경한다.
11. ReservationConfirmedEvent가 발행된다.
12. Seat Inventory Service가 좌석을 RESERVED로 확정한다.
13. Ticket Service가 티켓을 발급한다.
14. Notification Service가 예매 완료 알림을 발송한다.
15. Statistics Service가 통계 데이터를 갱신한다.
```

---

## 8.2 정상 흐름 시퀀스

```text
Client
  → API Gateway
  → Seat Inventory Service: 좌석 임시 선점 요청
  ← holdId 반환

Client
  → API Gateway
  → Reservation Service: 예매 생성 요청

Reservation Service
  → Seat Inventory Service: holdId 유효성 검증
  → Screening Service: 상영 일정 유효성 검증
  → Benefit Service: 쿠폰/포인트 검증
  → Reservation DB: PENDING_PAYMENT 저장
  → ReservationCreatedEvent 발행

Client
  → Payment Service: 결제 요청

Payment Service
  → PG Provider: 결제 승인 요청
  ← PG Provider: 결제 승인 성공
  → Payment DB: APPROVED 저장
  → PaymentApprovedEvent 발행

Reservation Service
  ← PaymentApprovedEvent
  → Reservation DB: CONFIRMED 변경
  → ReservationConfirmedEvent 발행

Seat Inventory Service
  ← ReservationConfirmedEvent
  → 좌석 RESERVED 변경

Ticket Service
  ← ReservationConfirmedEvent
  → 티켓 발급
  → TicketIssuedEvent 발행

Notification Service
  ← ReservationConfirmedEvent / TicketIssuedEvent
  → 알림 발송
```

---

# 9. Saga 설계

## 9.1 Saga 방식

예매 프로세스는 **Orchestration Saga**를 기본으로 한다.

### 이유

예매는 다음과 같이 상태 전이가 복잡하다.

```text
좌석 선점
예매 생성
쿠폰/포인트 사용
결제
예매 확정
좌석 예약 확정
티켓 발급
알림
```

이 흐름을 완전한 Choreography 방식으로만 처리하면 상태 추적이 어려워진다.

따라서 Reservation Service가 Saga Orchestrator 역할을 담당한다.

---

## 9.2 Saga 참여 서비스

| 참여 서비스                 | 역할                |
| ---------------------- | ----------------- |
| Reservation Service    | Saga Orchestrator |
| Seat Inventory Service | 좌석 선점/확정/해제       |
| Payment Service        | 결제 승인/취소/환불       |
| Benefit Service        | 쿠폰/포인트 예약/확정/복구   |
| Ticket Service         | 티켓 발급/무효화         |
| Notification Service   | 알림 발송             |
| Statistics Service     | 통계 반영             |

---

## 9.3 예매 Saga 상태

```text
STARTED
SEAT_HELD
RESERVATION_CREATED
PAYMENT_REQUESTED
PAYMENT_APPROVED
RESERVATION_CONFIRMED
SEAT_RESERVED
TICKET_ISSUED
COMPLETED
FAILED
COMPENSATING
COMPENSATED
```

---

## 9.4 보상 트랜잭션

| 실패 지점             | 보상 처리                                     |
| ----------------- | ----------------------------------------- |
| 좌석 선점 실패          | 예매 생성하지 않음                                |
| 예매 생성 실패          | 좌석 선점 해제                                  |
| 쿠폰 예약 실패          | 예매 생성 중단, 좌석 선점 해제                        |
| 결제 실패             | 예매 FAILED 또는 EXPIRED 처리, 좌석 해제, 쿠폰/포인트 복구 |
| 결제 타임아웃           | PG 상태 재조회 후 승인/실패 결정                      |
| 결제 성공 후 예매 확정 실패  | 예매 확정 재시도, 실패 시 운영자 보정 대상                 |
| 좌석 RESERVED 확정 실패 | 재시도, 실패 시 결제 취소/환불 검토                     |
| 티켓 발급 실패          | 티켓 발급 재시도, 예매 자체는 유지                      |
| 알림 발송 실패          | 알림 재시도, 예매 자체는 유지                         |
| 환불 실패             | 환불 재시도, 관리자 수동 처리                         |

---

# 10. 좌석 정합성 설계

좌석은 이 시스템에서 가장 강한 정합성이 필요한 영역이다.

## 10.1 좌석 임시 선점

좌석 임시 선점은 Redis를 사용한다.

```text
Key:
seat_hold:{screeningId}:{seatId}

Value:
{
  "holdId": "HOLD-123",
  "userId": 10,
  "reservationId": null,
  "expiresAt": "2026-05-12T12:10:00"
}

TTL:
5분
```

선점 시에는 원자적 연산을 사용한다.

```text
SET seat_hold:{screeningId}:{seatId} value NX EX 300
```

`NX` 조건으로 이미 선점된 좌석은 다시 선점할 수 없다.

---

## 10.2 여러 좌석 동시 선점

사용자가 A1, A2, A3를 동시에 선택하는 경우 일부만 성공하면 안 된다.

따라서 다음 전략 중 하나를 사용한다.

### 기본 전략

```text
1. 좌석 목록을 정렬한다.
2. 각 좌석에 대해 Redis SET NX EX 수행한다.
3. 하나라도 실패하면 이미 성공한 좌석 선점을 모두 해제한다.
4. 모두 성공하면 holdId를 반환한다.
```

### 이유

분산락보다 구현과 운영이 단순하고, Redis 원자 연산으로 충분히 빠르게 처리할 수 있다.

---

## 10.3 최종 예약 확정

결제 성공 후 좌석을 RESERVED로 확정할 때는 DB에서도 최종 중복 방지를 수행한다.

```text
Unique Constraint:
(screening_id, seat_id)
```

예약 확정 테이블에 동일 상영 일정/좌석 조합이 중복 저장될 수 없도록 한다.

즉, Redis는 빠른 선점을 담당하고, DB Unique Constraint는 최종 안전장치 역할을 한다.

---

# 11. 이벤트 설계 기본 원칙

## 11.1 이벤트 발행 기준

주요 상태 변경은 이벤트로 발행한다.

```text
예매 생성
예매 확정
예매 취소
예매 만료
결제 승인
결제 실패
환불 완료
티켓 발급
좌석 선점 만료
```

---

## 11.2 이벤트 신뢰성

이벤트 발행에는 Outbox Pattern을 적용한다.

### 이유

DB 저장은 성공했는데 이벤트 발행이 실패하면 서비스 간 상태 불일치가 발생할 수 있다.

예를 들어 Payment Service에서 결제 성공 저장은 됐는데 PaymentApprovedEvent 발행이 실패하면 Reservation Service는 예매를 확정하지 못한다.

따라서 다음 방식으로 처리한다.

```text
1. 비즈니스 DB 상태 변경
2. 같은 트랜잭션에서 outbox 테이블에 이벤트 저장
3. 별도 publisher가 outbox 이벤트를 Message Broker로 발행
4. 발행 성공 시 outbox 상태 변경
```

---

## 11.3 이벤트 중복 처리

모든 Consumer는 이벤트 중복 수신을 전제로 설계한다.

```text
processed_event
```

테이블을 두고 `eventId` 기준으로 이미 처리한 이벤트인지 확인한다.

```text
eventId가 이미 처리됨
→ 무시

eventId가 처음 수신됨
→ 처리 후 processed_event 저장
```

---

# 12. 주요 이벤트 목록

| 이벤트                       | Producer       | Consumer                                        | 목적               |
| ------------------------- | -------------- | ----------------------------------------------- | ---------------- |
| SeatHeldEvent             | Seat Inventory | Reservation                                     | 좌석 선점 완료         |
| SeatHoldExpiredEvent      | Seat Inventory | Reservation                                     | 선점 만료로 예매 만료 처리  |
| ReservationCreatedEvent   | Reservation    | Payment, Benefit, Statistics                    | 예매 생성 후 후속 처리    |
| ReservationExpiredEvent   | Reservation    | Seat, Benefit                                   | 예매 만료 후 자원 해제    |
| PaymentApprovedEvent      | Payment        | Reservation                                     | 결제 성공 후 예매 확정    |
| PaymentFailedEvent        | Payment        | Reservation, Notification                       | 결제 실패 처리         |
| ReservationConfirmedEvent | Reservation    | Seat, Ticket, Notification, Statistics          | 예매 확정 후 후속 처리    |
| ReservationCancelledEvent | Reservation    | Seat, Payment, Ticket, Notification, Statistics | 예매 취소 후 후속 처리    |
| PaymentRefundedEvent      | Payment        | Reservation, Notification, Statistics           | 환불 완료 반영         |
| TicketIssuedEvent         | Ticket         | Notification                                    | 티켓 발급 알림         |
| ScreeningCancelledEvent   | Screening      | Reservation, Notification                       | 상영 취소 후 예매 취소/환불 |

---

# 13. 장애 대응 기본 설계

## 13.1 결제 실패

```text
PaymentFailedEvent 발행
→ Reservation Service가 예매 FAILED 처리
→ Seat Inventory Service가 좌석 해제
→ Benefit Service가 쿠폰/포인트 복구
→ Notification Service가 결제 실패 알림
```

---

## 13.2 결제 타임아웃

결제 타임아웃은 즉시 실패로 단정하지 않는다.

```text
1. Payment Service가 PG 상태 재조회
2. 승인 상태면 PaymentApprovedEvent 발행
3. 실패 상태면 PaymentFailedEvent 발행
4. 알 수 없는 상태면 UNKNOWN 상태로 보관 후 재시도
```

---

## 13.3 결제 성공 후 예매 확정 실패

가장 위험한 장애 상황이다.

```text
결제는 성공했지만 예매가 CONFIRMED가 되지 않은 상태
```

대응 전략:

```text
1. PaymentApprovedEvent 재발행 가능
2. Reservation Service는 이벤트 중복 처리 가능
3. 미확정 예매 보정 배치 실행
4. 일정 시간 이상 불일치 시 운영자 알림
5. 필요 시 자동 환불 또는 수동 처리
```

---

## 13.4 티켓 발급 실패

티켓 발급 실패는 예매 확정 자체를 취소하지 않는다.

```text
ReservationConfirmed 상태 유지
Ticket Service에서 티켓 발급 재시도
실패 지속 시 운영자 알림
```

---

## 13.5 알림 발송 실패

알림 실패는 예매 상태에 영향을 주지 않는다.

```text
NotificationFailedEvent 기록
재시도
최대 재시도 실패 시 발송 실패 상태 저장
```

---

## 13.6 환불 실패

환불 실패는 반드시 추적 가능해야 한다.

```text
PaymentRefundFailedEvent 발행
관리자 확인 필요 상태로 변경
재시도 큐 등록
운영자 알림
```

---

# 14. API Gateway / BFF 설계

## 14.1 책임

API Gateway는 다음 책임을 가진다.

```text
인증 토큰 검증
서비스 라우팅
Rate Limiting
요청/응답 로깅
공통 에러 응답 변환
CORS 처리
API 버전 관리
```

---

## 14.2 BFF 역할

사용자 화면에 필요한 데이터를 여러 서비스에서 조합한다.

예를 들어 영화 상세 예매 화면은 다음 데이터를 필요로 한다.

```text
Movie Service      → 영화 정보
Screening Service  → 상영 일정
Theater Service    → 극장/상영관 정보
Seat Service       → 좌석 현황
```

클라이언트가 네 개 서비스를 직접 호출하지 않고 BFF가 조합한다.

---

# 15. 인프라 기본 설계

## 15.1 구성 요소

| 구성 요소                   | 용도            |
| ----------------------- | ------------- |
| API Gateway             | 외부 요청 진입점     |
| Service Discovery       | 서비스 위치 탐색     |
| Config Server           | 환경별 설정 관리     |
| Message Broker          | 이벤트 전달        |
| Redis                   | 좌석 선점, 캐시     |
| Database per Service    | 서비스별 데이터 독립성  |
| Object Storage          | 포스터, QR 이미지 등 |
| Observability Stack     | 로그, 메트릭, 트레이싱 |
| CI/CD Pipeline          | 자동 빌드/배포      |
| Container Orchestration | 서비스 배포/확장     |

---

## 15.2 Message Broker

Kafka 또는 RabbitMQ를 사용할 수 있다.

이 시스템에서는 다음 특성 때문에 Kafka가 적합하다.

```text
이벤트 기반 통계 생성
서비스 간 상태 변경 이벤트 기록
이벤트 재처리 가능성
높은 처리량
```

다만 초기 구현이 단순해야 한다면 RabbitMQ도 가능하다.

---

## 15.3 Redis

Redis는 다음 용도로 사용한다.

```text
좌석 임시 선점
상영 일정/좌석 조회 캐시
API Rate Limiting
짧은 TTL 기반 임시 데이터
```

중요한 점은 Redis만 최종 정합성 저장소로 사용하지 않는다는 것이다.
최종 예약 확정은 DB Unique Constraint로 한 번 더 보호한다.

---

# 16. 보안 기본 설계

## 16.1 인증/인가

```text
사용자 인증: Auth Service
API 인증 검증: API Gateway
권한 검증: Auth Service 또는 Gateway Policy
관리자 권한: Role 기반 접근 제어
```

---

## 16.2 개인정보 보호

```text
개인정보는 User Service에 집중
서비스 간 개인정보 전달 최소화
필요 시 userId만 전달
로그에 개인정보 출력 금지
```

---

## 16.3 결제 정보 보호

```text
카드번호 직접 저장 금지
PG 거래 ID와 결제 승인 번호만 저장
민감 결제 정보는 PG Provider에 위임
```

---

# 17. 운영/모니터링 기본 설계

## 17.1 필수 모니터링 지표

| 영역   | 지표                                   |
| ---- | ------------------------------------ |
| API  | 응답 시간, 오류율, 요청 수                     |
| 좌석   | 선점 성공률, 선점 실패율, 만료 건수                |
| 예매   | 예매 생성 수, 확정 수, 만료 수, 실패 수            |
| 결제   | 승인률, 실패율, 타임아웃 수, 환불 실패 수            |
| 메시지  | Consumer Lag, DLQ 건수                 |
| 인프라  | CPU, Memory, DB Connection, Redis 상태 |
| 비즈니스 | 영화별 예매율, 상영별 좌석 점유율                  |

---

## 17.2 분산 추적

예매 전체 흐름은 하나의 Trace로 추적되어야 한다.

```text
좌석 선점
→ 예매 생성
→ 결제 요청
→ 결제 승인
→ 예매 확정
→ 좌석 확정
→ 티켓 발급
→ 알림 발송
```

각 요청에는 다음 ID를 포함한다.

```text
traceId
correlationId
reservationId
paymentId
eventId
```

---

# 18. 서비스 경계 최종 정리

이 설계에서 가장 중요한 경계는 다음이다.

## 18.1 영화와 상영 일정은 다르다

```text
Movie Service     → 영화 자체 정보
Screening Service → 언제, 어디서 상영되는지
```

---

## 18.2 상영관 좌석과 상영 회차 좌석 상태는 다르다

```text
Theater Service        → 물리 좌석 구조
Seat Inventory Service → 특정 상영 일정의 좌석 상태
```

---

## 18.3 예매와 결제는 다르다

```text
Reservation Service → 예매 상태
Payment Service     → 결제 상태
```

---

## 18.4 예매와 티켓은 다르다

```text
Reservation Service → 예매 확정
Ticket Service      → 입장 가능한 티켓 발급
```

---

## 18.5 관리자 서비스는 도메인 데이터를 소유하지 않는다

```text
Admin Service → 관리자 화면/감사 로그/운영 API 조합
각 도메인 데이터 → 해당 도메인 서비스가 소유
```

---

# 19. 초기 MVP 기준 서비스 구성 제안

처음부터 모든 서비스를 완전히 분리하면 운영 복잡도가 커질 수 있다.
따라서 MVP 단계에서는 다음 구성을 권장한다.

## 19.1 반드시 분리할 서비스

```text
Auth Service
User Service
Movie Service
Theater Service
Screening Service
Seat Inventory Service
Reservation Service
Payment Service
Ticket Service
Notification Service
```

## 19.2 초기에는 통합하거나 후순위로 둘 수 있는 서비스

```text
Benefit Service
Statistics Service
Admin Service
```

Benefit Service는 쿠폰/포인트가 핵심 정책이면 초기에 분리한다.
그렇지 않다면 2차 단계로 분리해도 된다.

Statistics Service는 이벤트 기반 Read Model이므로 초기에는 간단한 배치/리포트 구조로 시작할 수 있다.

Admin Service는 처음에는 API Gateway + 각 도메인 관리자 API 조합으로 시작하고, 운영 기능이 커지면 별도 서비스로 분리할 수 있다.

---

# 20. 코딩 Agent에게 전달할 기본 구현 지침

다음 단계에서 함수 Signature 설계와 Code Skeleton 작성 시 아래 원칙을 따라야 한다.

```text
1. 서비스별 패키지와 DB를 분리한다.
2. 다른 서비스 DB에 직접 접근하지 않는다.
3. 예매 생성은 Reservation Service에서 담당한다.
4. 좌석 상태 변경은 Seat Inventory Service에서만 담당한다.
5. 결제 상태 변경은 Payment Service에서만 담당한다.
6. 티켓 생성은 Ticket Service에서만 담당한다.
7. 결제 성공 후 예매 확정은 이벤트 기반으로 처리한다.
8. 주요 상태 변경은 Outbox Pattern으로 이벤트를 발행한다.
9. 모든 이벤트 Consumer는 idempotent하게 구현한다.
10. 좌석 선점은 Redis SET NX EX 기반으로 구현한다.
11. 최종 좌석 예약 확정은 DB Unique Constraint로 보호한다.
12. 결제 요청에는 idempotencyKey를 사용한다.
13. 장애 복구를 위해 Saga 상태와 상태 변경 이력을 저장한다.
14. 알림/통계는 핵심 예매 흐름과 분리하여 비동기로 처리한다.
```

---

# 21. 기본 설계 결론

온라인 영화 예매 시스템의 MSA 설계에서 핵심은 다음 네 가지다.

```text
1. 서비스 경계를 도메인 기준으로 명확히 나눈다.
2. 좌석 상태는 Seat Inventory Service가 독점적으로 관리한다.
3. 예매-결제 흐름은 Reservation Service 중심의 Saga로 관리한다.
4. 상태 변경은 이벤트 기반으로 전파하되, 중복 처리와 재처리를 전제로 설계한다.
```

가장 중요한 서비스 경계는 아래와 같다.

```text
Movie Service
  → 영화 정보

Screening Service
  → 상영 일정

Theater Service
  → 극장/상영관/좌석 구조

Seat Inventory Service
  → 상영 회차별 좌석 상태

Reservation Service
  → 예매 상태

Payment Service
  → 결제 상태

Ticket Service
  → 티켓 상태
```

이 경계를 지키면 이후 함수 Signature 설계와 Code Skeleton 작성 단계에서도 서비스 책임이 섞이지 않고, MSA 구조에 맞는 구현으로 이어질 수 있다.
