## 문제상황

사이드 프로젝트에서 소셜 로그인 기능 구현 중 토큰 발급에 동시성 이슈가 발생했다. 여러 요청이 거의 동시에 들어올 때 `Duplicate entry for key 'refresh_token'` 오류가 발생하는 문제였다.

이 문제를 해결하기 위해 비관적 락(Pessimistic Lock)을 사용하는 방법을 선택했다:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT rt FROM RefreshToken rt WHERE rt.userId = :userId")
Optional<RefreshToken> findByUserIdWithLock(@Param("userId") Long userId);
```

비관적 락으로 문제는 해결했지만, 다른 해결 방안으로 '트랜잭션 격리 수준 조정'이라는 방법도 있다는 것을 알게 됐다. 이에 트랜잭션 격리 수준과 ACID 속성에 대해 더 깊이 이해하고자 했다.

## 해결방안

트랜잭션의 기본 개념과 ACID 속성, 그리고 다양한 격리 수준에 대한 학습을 통해 데이터베이스 트랜잭션을 더 깊이 이해하기로 했다.

## 동작원리

### 트랜잭션의 ACID 속성

#### 1. 원자성(Atomicity)

원자성은 트랜잭션이 **더 이상 분할할 수 없는 최소 작업 단위**라는 의미다. 트랜잭션에 포함된 모든 연산은 전부 성공하거나 전부 실패해야 한다.

예를 들어, 계좌 이체 트랜잭션은:

- 출금 계좌에서 금액 차감
- 입금 계좌에 금액 추가

이 두 작업이 분리될 수 없는 하나의 단위로 취급되어야 한다. 출금만 되고 입금이 안 되는 상황은 절대 발생하면 안 된다.

#### 2. 일관성(Consistency)

일관성은 트랜잭션 전후로 **데이터베이스의 모든 규칙과 제약조건이 위반되지 않아야 한다**는 의미다.

계좌 이체 예시의 규칙들:

- 모든 계좌 잔액은 0원 미만이 될 수 없다
- 시스템 내 모든 돈의 총합은 항상 일정해야 한다

A계좌에서 10,000원을 출금하고 B계좌에 10,000원을 입금하는 트랜잭션은 이러한 규칙들을 모두 만족하므로 일관성을 유지한다.

#### 3. 격리성(Isolation)

격리성은 동시에 실행되는 여러 트랜잭션이 **서로 영향을 미치지 않아야 한다**는 의미다. 여러 트랜잭션이 동시에 처리될 때도 각 트랜잭션은 마치 혼자서 실행되는 것처럼 동작해야 한다.

이를 위해 데이터베이스는 다음과 같은 메커니즘을 사용한다:

- **락킹(Locking)**: 데이터에 접근할 때 락을 걸어 충돌을 방지
- **MVCC(Multi-Version Concurrency Control)**: 데이터의 여러 버전을 유지해 읽기가 쓰기를 방해하지 않도록 함

#### 4. 지속성(Durability)

지속성은 트랜잭션이 성공적으로 커밋되면, 그 결과는 **시스템이 다운되더라도 영구적으로 보존**되어야 한다는 의미다.

이를 위한 방법:

- **WAL(Write-Ahead Logging)**: 실제 데이터 변경 전에 로그에 먼저 기록
- **체크포인트**: 주기적으로 메모리 내용을 디스크에 기록
- **다중화(Replication)**: 여러 서버에 같은 데이터 복제

### 트랜잭션 격리 수준(Isolation Level)

트랜잭션 격리 수준은 "여러 트랜잭션이 동시에 실행될 때 서로 얼마나 영향을 주고받을 수 있는지" 정하는 규칙이다. 일상적인 예로 비유하면, 공유 문서를 여러 사람이 편집하는 상황에서의 권한 설정과 비슷하다.

#### 1. READ UNCOMMITTED (가장 낮은 수준)

- **특징**: 다른 트랜잭션이 커밋하지 않은 변경사항도 읽을 수 있음
- **일상 비유**: 다른 사람이 아직 "저장"을 안 눌렀는데도 그 사람의 타이핑이 실시간으로 보이는 상태
- **발생 가능한 문제**: Dirty Read (더티 리드) - 나중에 롤백될 수 있는 데이터를 읽음

```sql
-- MySQL 예시
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT * FROM accounts WHERE id = 1; -- 다른 트랜잭션에서 변경 중인 데이터도 읽음
COMMIT;
```

#### 2. READ COMMITTED (일반적인 수준)

- **특징**: 커밋된 데이터만 읽을 수 있음
- **일상 비유**: 다른 사람이 "저장" 버튼을 누른 내용만 볼 수 있음
- **발생 가능한 문제**: Non-repeatable Read (반복 불가능한 읽기) - 같은 쿼리를 두 번 실행하면 결과가 달라질 수 있음

```sql
-- PostgreSQL 기본 격리 수준
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1; -- 첫 번째 조회: 10,000원
-- 다른 트랜잭션에서 balance를 5,000원으로 변경하고 커밋
SELECT balance FROM accounts WHERE id = 1; -- 두 번째 조회: 5,000원으로 변경됨
COMMIT;
```

#### 3. REPEATABLE READ (높은 수준)

- **특징**: 트랜잭션 시작 시점의 스냅샷을 기준으로 데이터를 읽음
- **일상 비유**: 내가 문서를 연 시점의 저장된 내용만 계속 보임
- **발생 가능한 문제**: Phantom Read (유령 읽기) - 조건에 맞는 새 행이 추가되어도 감지하지 못함

```sql
-- MySQL InnoDB 기본 격리 수준
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT * FROM accounts WHERE balance > 1000; -- 10개 행 반환
-- 다른 트랜잭션에서 balance가 1500인 계좌 추가하고 커밋
SELECT * FROM accounts WHERE balance > 1000; -- 여전히 10개 행만 반환
COMMIT;
```

#### 4. SERIALIZABLE (가장 높은 수준)

- **특징**: 트랜잭션이 순차적으로 실행되도록 보장
- **일상 비유**: 한 번에 한 사람만 문서를 편집할 수 있음
- **장점**: 모든 동시성 문제 해결
- **단점**: 성능 저하, 대기 시간 증가

```sql
-- 모든 주요 DBMS에서 지원
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;
SELECT * FROM accounts WHERE owner_id = 100; -- 읽기 잠금 설정
UPDATE accounts SET balance = balance - 1000 WHERE id = 1; -- 여기서 다른 트랜잭션은 대기
COMMIT;
```

### 격리 수준과 발생 가능한 문제의 관계

|격리 수준|Dirty Read|Non-repeatable Read|Phantom Read|
|---|---|---|---|
|READ UNCOMMITTED|발생|발생|발생|
|READ COMMITTED|방지|발생|발생|
|REPEATABLE READ|방지|방지|발생(일부 DBMS는 방지)|
|SERIALIZABLE|방지|방지|방지|

### 격리 수준별 동작 메커니즘

#### READ UNCOMMITTED

- 락을 사용하지 않고 다른 트랜잭션의 변경 중인 데이터를 그대로 읽음
- 가장 빠르지만 데이터 일관성 보장 못함

#### READ COMMITTED

- 각 SELECT 쿼리마다 새로운 스냅샷 생성
- 커밋된 데이터만 읽을 수 있어 Dirty Read 방지
- 공유 락(Shared Lock)을 사용하여 읽는 동안만 락 유지

#### REPEATABLE READ

- 트랜잭션 시작 시점에 스냅샷 생성
- 트랜잭션 내에서 일관된 데이터 보장
- MySQL InnoDB에서는 MVCC와 갭 락(Gap Lock)을 사용하여 일부 Phantom Read도 방지

#### SERIALIZABLE

- 읽기에도 공유 락을 설정하고 트랜잭션 종료까지 유지
- 범위 조회 시 범위 락(Range Lock)을 사용하여 새 데이터 삽입도 차단
- 완벽한 격리를 제공하지만 동시성이 크게 저하됨

## 활용예시

### Spring Transaction에서의 격리 수준 설정

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void transferMoney(String fromAccount, String toAccount, BigDecimal amount) {
    Account from = accountRepository.findByAccountNumber(fromAccount);
    Account to = accountRepository.findByAccountNumber(toAccount);

    from.withdraw(amount);
    to.deposit(amount);

    accountRepository.save(from);
    accountRepository.save(to);
}
```

### 웹 애플리케이션에서 격리 수준 선택 예시

#### 1. 게시판 조회 기능 (READ COMMITTED)

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public List<Post> getRecentPosts() {
    return postRepository.findTop10ByOrderByCreatedAtDesc();
}
```

- 최신 게시글을 조회하는 함수에서는 최신 데이터를 보여주는 것이 중요
- Dirty Read만 방지하면 되므로 READ_COMMITTED 적절

#### 2. 송금 기능 (REPEATABLE READ)

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
    Account from = accountRepository.findById(fromId).orElseThrow();
    Account to = accountRepository.findById(toId).orElseThrow();

    if (from.getBalance().compareTo(amount) < 0) {
        throw new InsufficientBalanceException();
    }

    from.withdraw(amount);
    to.deposit(amount);
}
```

- 계좌 잔액 확인 후 출금할 때, 중간에 잔액이 변경되면 안 됨
- 따라서 Non-repeatable Read를 방지하는 REPEATABLE_READ 필요

#### 3. 좌석 예약 시스템 (SERIALIZABLE)

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public boolean reserveSeat(Long showId, String seatNumber, Long userId) {
    // 해당 좌석이 이미 예약되었는지 확인
    boolean isAlreadyReserved = reservationRepository
            .existsByShowIdAndSeatNumber(showId, seatNumber);

    if (isAlreadyReserved) {
        return false; // 이미 예약됨
    }

    // 새 예약 생성
    Reservation reservation = new Reservation();
    reservation.setShowId(showId);
    reservation.setSeatNumber(seatNumber);
    reservation.setUserId(userId);

    reservationRepository.save(reservation);
    return true; // 예약 성공
}
```

- 콘서트 좌석 예약 같은 경우, 동시 예약을 완벽히 방지해야 함
- 다른 방법으로는 비관적 락이나 분산 락도 고려 가능

## 주의사항

1. **격리 수준과 성능은 트레이드오프 관계**
    - 높은 격리 수준 → 더 정확한 데이터, 더 낮은 동시성(성능 저하)
    - 낮은 격리 수준 → 더 높은 동시성(성능 향상), 데이터 일관성 문제 가능성
2. **DB별로 기본 격리 수준이 다름**
    - MySQL(InnoDB): REPEATABLE READ
    - PostgreSQL, Oracle, SQL Server: READ COMMITTED
    - 여러 DB를 사용하는 프로젝트에서는 명시적 설정 필요
3. **격리 수준만으로 모든 동시성 문제를 해결할 수 없음**
    - 특정 상황(예: 재고 관리)에서는 비관적 락이나 낙관적 락 필요
    - 분산 환경에서는 데이터베이스 외부 락 메커니즘 고려
4. **트랜잭션 범위 최적화**
    - 트랜잭션 격리 수준을 높이면 락의 유지 시간도 길어짐
    - 필요한 작업만 트랜잭션에 포함시켜 범위 최소화
5. **데드락 가능성**
    - 높은 격리 수준에서는 데드락 발생 확률 증가
    - 타임아웃 설정과 재시도 메커니즘 고려

## 결론

트랜잭션 격리 수준에 대한 학습을 통해 나는 동시성 제어의 다양한 방법과 각 방법의 장단점을 생각해볼 수 있었다. 내가 비관적 락으로 해결했던 문제는 트랜잭션 격리 수준 조정으로도 해결 가능했으나, 성능과 정확성의 트레이드오프를 고려했을 때 비관적 락이 더 적합했다고 판단했다.

ACID 속성의 이해는 데이터베이스 트랜잭션을 설계할 때 필수적이며, 특히 격리성(Isolation)을 어느 수준까지 보장할지는 시스템 요구사항과 성능 목표에 따라 신중히 결정해야 할 것 같다.

앞으로 동시성 문제를 마주할 때는 다음과 같은 순서로 접근해보려 한다:

1. 문제 상황 정확히 파악 (어떤 동시성 문제가 발생하는지)
2. 가능한 해결 방법 검토 (트랜잭션 격리 수준, 비관적 락, 낙관적 락 등)
3. 성능과 정확성 요구사항 고려하여 최적의 방법 선택
4. 선택한 방법 구현 및 테스트
5. 모니터링 및 필요시 조정