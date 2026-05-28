# 예약 대기 미션에서 발생할 수 있는 레이스 컨디션과 해결 방법

## 1. 예약 대기 미션에서 발생할 수 있는 레이스 컨디션

**예약**과 **예약 대기**는 같은 **슬롯**을 기준으로 **생성**, **수정**, **삭제**된다.

여기서 슬롯이란 다음 세 값을 조합한 **예약 가능 단위**를 의미한다.

- `reservation_date` + `time_id` + `theme_id`

이 슬롯에 대해 여러 요청이 동시에 들어오면 다음과 같은 **레이스 컨디션**이 발생할 수 있다.

### 1.1. 예약 생성 관련

> **같은 슬롯**에 대해 예약 생성 요청이 동시에 들어오는 상황
>

> **예약 대기가 있는 슬롯**에 일반 예약 생성 요청이 들어오는 동안, 다른 요청이 같은 슬롯에 예약 대기를 생성하는 상황
>

> 예약 생성 중 **예약 시간 또는 테마의 존재를 확인한 뒤**, 다른 요청이 해당 예약 시간 또는 테마를 삭제하는 상황
>

### 1.2. 예약 대기 생성 관련

> **같은 사용자가 같은 슬롯**에 대해 예약 대기 신청을 동시에 여러 번 보내는 상황
>

> 예약 대기 신청 중 **대상 예약 존재를 확인한 뒤**, 다른 요청이 해당 예약을 삭제하거나 다른 슬롯으로 변경하는 상황
>

> 예약 대기 생성 중 **예약 시간 또는 테마의 존재를 확인한 뒤**, 다른 요청이 해당 예약 시간 또는 테마를 삭제하는 상황
>

### 1.3. 예약 수정 관련

> **같은 예약**을 여러 요청이 동시에 수정하는 상황
>

> 예약 수정 중 **변경할 슬롯의 중복 예약 여부를 확인한 뒤**, 다른 요청이 해당 슬롯에 예약을 생성하거나 수정하는 상황
>

> 예약 수정 중 **변경할 슬롯의 예약 대기 존재 여부를 확인한 뒤**, 다른 요청이 해당 슬롯에 예약 대기를 생성하는 상황
>

> 예약 수정으로 **기존 슬롯이 비는 동안**, 첫 번째 예약 대기를 승격하는 과정에서 다른 요청이 같은 대기를 삭제하거나 승격하는 상황
>

### 1.4. 예약 삭제 관련

> **같은 예약**을 여러 요청이 동시에 삭제하는 상황
>

> 예약 삭제 후 **첫 번째 예약 대기를 승격하는 동안**, 다른 요청이 같은 대기를 삭제하거나 승격하는 상황
>

> **첫 번째 예약 대기를 삭제한 뒤 예약으로 저장하기 전에**, 다른 요청이 같은 슬롯에 예약을 생성하는 상황
>

### 1.5. 예약 대기 삭제 관련

> **같은 예약 대기**를 여러 요청이 동시에 삭제하는 상황
>

### 1.6. 예약 시간 / 테마 관련

> **같은 예약 시간**을 여러 요청이 동시에 생성하는 상황
>

> **같은 테마**를 여러 요청이 동시에 생성하는 상황
>

> 예약 시간 삭제 전 **존재 여부를 확인한 뒤**, 다른 요청이 해당 시간을 참조하는 예약 또는 예약 대기를 생성하는 상황
>

> 테마 삭제 전 **존재 여부를 확인한 뒤**, 다른 요청이 해당 테마를 참조하는 예약 또는 예약 대기를 생성하는 상황
>

> **같은 예약 시간 또는 같은 테마**를 여러 요청이 동시에 삭제하는 상황
>

---

## 2. 어떻게 해결할 수 있는가?

**레이스 컨디션**은 하나의 방식만으로 모두 해결하기보다, 상황에 따라 여러 **방어 수단**을 조합해야 한다.

대표적인 해결 방법은 다음과 같다.

### 2.1. DB 제약 조건으로 방어한다

가장 먼저 고려할 수 있는 방법은 **DB 제약 조건**이다.

DB 제약 조건은 **애플리케이션 검증**을 통과한 뒤에도, **동시성 상황**에서 마지막으로 **데이터 정합성**을 지켜주는 방어선이 된다.

예를 들어 같은 슬롯에는 하나의 예약만 존재해야 한다면 다음과 같이 `unique 제약 조건`을 걸 수 있다.

```sql
CREATE TABLE reservation (
    id BIGINT NOT NULL AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    reservation_date DATE NOT NULL,
    time_id BIGINT NOT NULL,
    theme_id BIGINT NOT NULL,
    PRIMARY KEY (id),
    FOREIGN KEY (time_id) REFERENCES reservation_time (id),
    FOREIGN KEY (theme_id) REFERENCES theme (id),
    UNIQUE (reservation_date, time_id, theme_id)
);
```

이렇게 하면 **여러 요청이 동시에 같은 슬롯에 예약을 생성하더라도,** DB는 최종적으로 **하나의 예약만 허용**한다.

### 2.2. 서비스 검증으로 방어한다

서비스 계층에서는 **비즈니스 규칙**을 명시적으로 검증한다.

예를 들어 예약 시간을 삭제할 때, 해당 id의 데이터가 실제로 삭제되었는지 `affectedRow`를 통해 확인할 수 있다.

```java
int affectedRow = reservationTimeRepository.deleteById(id);
int nonAffected = 0;

if (affectedRow == nonAffected) {
    throw new TimeNotFoundException();
}
```

이 방식은 **이미 삭제된 데이터를 다시 삭제하려는 요청**이나, **존재하지 않는 데이터를 삭제하려는 요청**을 감지할 수 있다.

다만 서비스 검증만으로는 **동시성 문제를 완전히 막을 수 없다.**

검증 시점과 실제 변경 시점 사이에 다른 트랜잭션이 데이터를 변경할 수 있기 때문이다.

### 2.3. 데이터베이스 Lock으로 방어한다

**검증과 변경 사이에 다른 트랜잭션이 끼어들면 안 되는 경우**에는 **DB Lock**을 사용할 수 있다.

예를 들어 예약 시간을 삭제하기 전에 해당 시간이 존재하는지 확인했다고 하자.

그 직후 다른 요청이 같은 예약 시간을 참조하는 예약이나 예약 대기를 생성하면, 삭제 로직과 생성 로직 사이에서 충돌이 발생할 수 있다.

이런 경우 예약 시간 **row**에 **lock**을 걸어, **삭제 트랜잭션**이 끝날 때까지 다른 트랜잭션이 해당 예약 시간을 기준으로 예약을 생성하지 못하게 만들 수 있다.

```sql
SELECT id, start_at
FROM reservation_time
WHERE id = ?
FOR UPDATE;
```

---

## 3. 데이터베이스 Lock

### 3.1. 데이터베이스 Lock이란?

**데이터베이스 Lock**이란 여러 트랜잭션이 같은 데이터에 동시에 접근할 때, **데이터의 일관성**을 지키기 위해 특정 데이터나 자원에 대한 **접근을 일시적으로 제한**하는 기법이다.

### 3.2. Lock의 유형

#### 공유 락(Shared Lock)

공유 락은 **읽기용 락**이다.

공유 락이 걸린 데이터는 다른 트랜잭션도 읽을 수 있지만, 수정은 제한된다.

- 다른 트랜잭션의 읽기: `가능`
- 다른 트랜잭션의 수정: `제한`

즉, **읽는 동안 데이터가 변경되지 않도록 막는 락**이라고 볼 수 있다.

MySQL에서는 다음과 같은 방식으로 공유 락을 걸 수 있다.

```sql
SELECT *
FROM reservation
WHERE id = 1
FOR SHARE;
```

또는 MySQL 8.0 이전 스타일로는 다음과 같이 사용할 수 있다.

```sql
SELECT *
FROM reservation
WHERE id = 1
LOCK IN SHARE MODE;
```

#### 배타 락(Exclusive Lock)

배타 락은 **쓰기용 락**이다.

배타 락이 걸린 데이터는 다른 트랜잭션이 수정할 수 없다.

- 다른 트랜잭션의 수정: `제한`
- 다른 트랜잭션의 읽기: `상황에 따라 제한`

즉, **내가 수정하는 동안 다른 트랜잭션이 해당 데이터를 변경하지 못하게 막는 락**이라고 볼 수 있다.

대표적으로 `FOR UPDATE`가 있다.

```sql
SELECT *
FROM reservation
WHERE id = 1
FOR UPDATE;
```

### 3.3. Lock 전략

데이터베이스 Lock 전략은 **비관적 락**과 **낙관적 락**으로 나누어 볼 수 있다.

#### 비관적 락

비관적 락은 **충돌이 발생할 가능성이 높다고 보고**, 데이터를 읽거나 수정하기 전에 미리 **DB Lock**을 거는 전략이다.

예를 들어 다음 쿼리는 조회한 row에 **배타 락**을 건다.

```sql
SELECT *
FROM reservation
WHERE id = 1
FOR UPDATE;
```

비관적 락은 실제 **DB Lock**을 사용하기 때문에, 다른 트랜잭션은 해당 Lock이 해제될 때까지 대기하거나 실패할 수 있다.

즉, 비관적 락은 DB에서 제공하는 **공유 락**이나 **배타 락**을 사용해 다른 트랜잭션의 데이터 접근을 제어하는 방식이다.

#### 낙관적 락

낙관적 락은 **충돌이 드물 것이라고 보고**, 데이터를 읽는 시점에는 Lock을 걸지 않는 전략이다.

대신 수정 시점에 **내가 읽은 이후 데이터가 변경되었는지** 확인한다.

보통 `version` 컬럼을 사용해 데이터 변경 여부를 감지하고 충돌을 처리한다.

```sql
UPDATE reservation
SET name = 'kim',
    version = version + 1
WHERE id = 1
  AND version = 3;
```

이때 수정된 `row` 수가 `0`이면, 내가 읽은 뒤에 다른 트랜잭션이 먼저 데이터를 수정했다는 뜻이다.

즉, 낙관적 락은 데이터를 미리 막는 방식이 아니라, **수정 시점에 충돌을 감지하는 방식**이다.

JPA를 사용한다면 `@Version` 애노테이션을 통해 낙관적 락을 적용할 수 있다.

```java
@Entity
@Builder
@Getter
@NoArgsConstructor
@AllArgsConstructor
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    private String subTitle;
    private String author;

    @Version
    private Integer version;
}
```

### 3.4. Lock 적용 범위

데이터베이스 **Lock**은 어느 범위를 잠그는지에 따라 다음과 같이 나눌 수 있다.

#### Row Lock

Row Lock은 **특정 행**에 거는 락이다.

```sql
SELECT *
FROM reservation
WHERE id = 1
FOR UPDATE;
```

동시성을 높일 수 있기 때문에 일반적인 상황에서는 **Table Lock**보다 유리하다.

#### Table Lock

Table Lock은 **테이블 전체**에 거는 락이다.

동시성은 낮지만, 테이블 전체에 대한 일괄 작업이나 특정 DDL 작업에서 사용될 수 있다.

#### Gap Lock

Gap Lock은 실제 `row`가 아니라, row와 row 사이의 **범위**에 거는 락이다.

예를 들어 `10 < id < 20` 범위에 대해 조회할 때, 그 사이에 새로운 `row`가 삽입되는 것을 막기 위해 사용될 수 있다.

즉, 기존 데이터를 잠그는 것이 아니라 **비어 있는 범위에 새로운 데이터가 끼어드는 것**을 막는 락이다.

#### Next-Key Lock

Next-Key Lock은 **Row Lock**과 **Gap Lock**이 합쳐진 락이다.

**InnoDB**에서는 특정 격리 수준에서 **팬텀 리드**를 막기 위해 **Next-Key Lock**을 사용할 수 있다.

---

## 4. 트랜잭션 격리 수준과 Lock

### 4.1. 트랜잭션 격리 수준이란?

트랜잭션 격리 수준은 **동시에 실행되는 여러 트랜잭션이 서로에게 어느 정도까지 영향을 줄 수 있는지**에 대한 설정이다.

즉, 트랜잭션 격리 수준은 다음 질문에 대한 **DB**의 정책이라고 볼 수 있다.

> 동시에 실행되는 트랜잭션 사이에서 어떤 동시성 문제까지 허용할 것인가?
>

### 4.2. 분리 기준

**SQL** 표준에서는 다음 이상 현상을 어디까지 방지하는지를 기준으로 격리 수준을 분리한다.

| 이상 현상 | 설명 |
| --- | --- |
| Dirty Read | 다른 트랜잭션이 아직 커밋하지 않은 데이터를 읽는 현상 |
| Non-repeatable Read | 같은 트랜잭션 안에서 같은 row를 다시 읽었는데 값이 달라지는 현상 |
| Phantom Read | 같은 조건으로 다시 조회했는데, 처음에는 없던 row가 생기거나 있던 row가 사라지는 현상 |

### 4.3. 트랜잭션 격리 수준 유형

**SQL 표준**에서 정의하는 트랜잭션 격리 수준은 다음 네 가지이다.

#### Read Uncommitted

**가장 낮은** 격리 수준이다.

다른 트랜잭션이 **아직 커밋하지 않은 데이터도** 읽을 수 있다.

| 이상 현상 | 발생 여부 |
| --- | --- |
| Dirty Read | 발생 가능 |
| Non-repeatable Read | 발생 가능 |
| Phantom Read | 발생 가능 |

#### Read Committed

**커밋된 데이터**만 읽을 수 있는 격리 수준이다.

따라서 **Dirty Read**는 막을 수 있다.

하지만 같은 트랜잭션 안에서 같은 데이터를 다시 읽을 때, 그 사이 다른 트랜잭션이 커밋했다면 다른 값을 읽을 수 있다.

| 이상 현상 | 발생 여부 |
| --- | --- |
| Dirty Read | 방지 |
| Non-repeatable Read | 발생 가능 |
| Phantom Read | 발생 가능 |

#### Repeatable Read

같은 트랜잭션 안에서 같**은 데이터를 반복해서 읽어도 같은 결과를 보장**하는 격리 수준이다.

따라서 **Dirty Read**와 **Non-repeatable Read**를 막을 수 있다.

다만 **Phantom Read**를 막는지는 DB의 구현 방식에 따라 다를 수 있다.

예를 들어 **MySQL InnoDB**는 기본 격리 수준이 **Repeatable Read**이며, 일반 `SELECT`는 트랜잭션 안에서 첫 번째 읽기 시점의 스냅샷을 기준으로 일관된 결과를 읽는다.

| 이상 현상 | 발생 여부 |
| --- | --- |
| Dirty Read | 방지 |
| Non-repeatable Read | 방지 |
| Phantom Read | DB 구현에 따라 다름 |

#### Serializable

**가장 강한** 격리 수준이다.

여러 트랜잭션이 동시에 실행되더라도, 결과가 마치 **트랜잭션들을 하나씩 순서대로 실행한 것**과 같아야 한다.

| 이상 현상 | 발생 여부 |
| --- | --- |
| Dirty Read | 방지 |
| Non-repeatable Read | 방지 |
| Phantom Read | 방지 |

### 4.4. 트랜잭션 격리 수준의 구현 방법

트랜잭션 격리 수준은 **개념적인 정책**이고, **DB**는 이를 구현하기 위해 **여러 동시성 제어 방식**을 사용한다.

대표적인 구현 방식은 다음과 같다.

#### Lock 기반

Lock 기반 동시성 제어는 데이터를 읽거나 수정할 때 실제 **Lock**을 걸어, 다른 트랜잭션의 접근을 제한하는 방식이다.

예를 들어 특정 `row`를 수정하려면 해당 `row`에 **배타 락**을 걸 수 있다.

```sql
SELECT *
FROM reservation
WHERE id = 1
FOR UPDATE;
```

또는 읽는 동안 다른 트랜잭션이 수정하지 못하게 공유 락을 걸 수 있다.

```sql
SELECT *
FROM reservation
WHERE id = 1
FOR SHARE;
```

Lock 기반 방식은 직관적이지만, **락 대기**나 **데드락**이 발생할 수 있고 동시성이 낮아질 수 있다.

#### MVCC(Multi-Version Concurrency Control) 기반

하나의 데이터를 하나의 값으로만 관리하는 것이 아니라**, 여러 버전**으로 관리하여 트랜잭션마다 적절한 버전을 읽게 하는 방식이다.

예를 들어 **A 트랜잭션**이 데이터를 **수정** 중이어도, **B 트랜잭션**은 A가 **수정하기 전의 커밋된 버전**을 읽을 수 있다.

- A 트랜잭션
    - reservation id = 1 수정 중, 아직 커밋하지 않음
- B 트랜잭션
    - reservation id = 1 조회
- 결과
    - B는 A가 수정하기 전의 커밋된 데이터를 읽음

따라서 **MVCC**에서는 일반적인 조회가 쓰기 작업을 막지 않고, 쓰기 작업도 일반적인 조회를 막지 않는다.

### 4.5. MySQL InnoDB의 구현 방식

MySQL InnoDB는 **MVCC**와 **Lock**을 함께 사용한다.

일반 `SELECT`는 보통 **MVCC**를 사용한 **consistent read**로 처리된다.

```sql
SELECT *
FROM reservation
WHERE id = 1;
```

이 경우 일반적으로 **row lock**을 걸지 않고, 트랜잭션 격리 수준에 맞는 데이터 버전을 읽는다.

반면 `FOR UPDATE`, `FOR SHARE`와 같은 locking read는 **실제 DB Lock**을 사용한다.

```sql
SELECT *
FROM reservation
WHERE id = 1
FOR UPDATE;
```

```sql
SELECT *
FROM reservation
WHERE id = 1
FOR SHARE;
```

즉, InnoDB에서는 다음과 같이 볼 수 있다.

- 일반 `SELECT`
    - **MVCC** 기반 **consistent read**
- `SELECT ... FOR UPDATE`
    - **배타 락**을 사용하는 **locking read**
- `SELECT ... FOR SHARE`
    - **공유 락**을 사용하는 **locking read**

---

## 5. 자바 Spring에서 동시성 테스트하기

### 5.1. 테스트의 구성

```java
private void executeConcurrently(
        int threadCount,
        Runnable task
) throws InterruptedException {
    ExecutorService executorService = Executors.newFixedThreadPool(threadCount);

    CountDownLatch readyLatch = new CountDownLatch(threadCount);
    CountDownLatch startLatch = new CountDownLatch(1);
    CountDownLatch doneLatch = new CountDownLatch(threadCount);

    for (int i = 0; i < threadCount; i++) {
        executorService.submit(() -> {
            try {
                readyLatch.countDown();
                startLatch.await();

                task.run();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException(e);
            } finally {
                doneLatch.countDown();
            }
        });
    }

    readyLatch.await();
    startLatch.countDown();
    doneLatch.await();

    executorService.shutdown();
}
```

### 5.2. 코드 설명

이 테스트 유틸 메서드는 여러 스레드가 **같은 작업을 최대한 동시에 실행**하도록 만들기 위한 코드이다.

전체 흐름은 다음과 같다.

1. 고정 크기의 **스레드 풀**을 만든다.
2. 각 스레드를 **준비 상태**로 만든다.
3. 모든 스레드가 **준비될 때까지 기다린다**.
4. **시작 신호**를 보내 모든 스레드를 **동시에 실행**시킨다.
5. 모든 스레드의 작업이 **끝날 때까지 기다린다.**
6. 스레드 풀을 종료한다.

#### `ExecutorService`

```java
ExecutorService executorService = Executors.newFixedThreadPool(threadCount);
```

`ExecutorService`는 여러 작업을 **별도의 스레드**에서 실행하기 위한 도구이다.

여기서는 `threadCount`만큼 고정된 크기의 **스레드 풀**을 생성한다.

예를 들어 `threadCount`가 `10`이면, 최대 10개의 작업을 동시에 실행할 수 있다.

즉, 동시성 테스트에서 여러 요청을 동시에 보내기 위한 기반이 된다.

#### `readyLatch`

```java
CountDownLatch readyLatch = new CountDownLatch(threadCount);
```

`readyLatch`는 모든 스레드가 **실행 준비를 마쳤는지 확인**하기 위한 장치이다.

각 스레드는 작업을 시작하기 전에 다음 코드를 실행한다.

```java
readyLatch.countDown();
```

이는 해당 스레드가 **실행 준비를 마쳤다**고 알리는 역할을 한다.

그리고 메인 테스트 스레드는 다음 코드에서 모든 작업 스레드가 **준비될 때까지 기다린다**.

```java
readyLatch.await();
```

이 과정이 없다면 일부 스레드만 준비된 상태에서 작업이 먼저 시작될 수 있다.

따라서 `readyLatch`는 모든 스레드를 출발선에 세우기 위한 역할을 한다.

#### `startLatch`

```java
CountDownLatch startLatch = new CountDownLatch(1);
```

`startLatch`는 모든 스레드를 동시에 출발시키기 위한 장치이다.

각 작업 스레드는 다음 코드에서 대기한다.

```java
startLatch.await();
```

그리고 메인 테스트 스레드가 다음 코드를 실행하면,

```java
startLatch.countDown();
```

대기 중이던 모든 작업 스레드가 동시에 `task.run()`을 실행한다.

```
task.run();
```

즉, `startLatch`는 실제 레이스 컨디션을 재현하기 위한 핵심 장치이다.

#### `doneLatch`

```java
CountDownLatch doneLatch = new CountDownLatch(threadCount);
```

`doneLatch`는 모든 작업 스레드가 끝날 때까지 기다리기 위한 장치이다.

각 작업 스레드는 작업이 끝나면 `finally` 블록에서 다음 코드를 실행한다.

```java
doneLatch.countDown();
```

메인 테스트 스레드는 다음 코드에서 모든 작업이 끝날 때까지 기다린다.

```java
doneLatch.await();
```

이 과정이 없다면, 작업 스레드가 아직 실행 중인데도 검증 코드가 먼저 실행될 수 있다.

따라서 `doneLatch`는 모든 동시 작업이 끝난 뒤 검증을 수행하도록 보장한다.

#### `InterruptedException` 처리

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    throw new RuntimeException(e);
}
```

`await()` 중인 스레드는 인터럽트될 수 있다.

이때 단순히 예외를 무시하면, 현재 스레드가 인터럽트되었다는 상태가 사라질 수 있다.

그래서 다음 코드로 인터럽트 상태를 다시 설정한다.

```java
Thread.currentThread().interrupt();
```

그 후 테스트 실패로 이어질 수 있도록 `RuntimeException`으로 감싸서 던진다.

#### `finally`에서 `doneLatch.countDown()`을 호출하는 이유

```java
finally {
doneLatch.countDown();
}
```

작업 중 예외가 발생하더라도 `doneLatch.countDown()`은 반드시 호출되어야 한다.

만약 예외가 발생했는데 `doneLatch.countDown()`이 호출되지 않으면, 메인 테스트 스레드는 다음 코드에서 영원히 대기할 수 있다.

```
doneLatch.await();
```

따라서 작업 성공 여부와 관계없이 `doneLatch`를 감소시키기 위해 `finally` 블록을 사용한다.

#### `executorService.shutdown()`

```java
executorService.shutdown();
```

모든 작업이 끝난 뒤 스레드 풀을 종료한다.

`shutdown()`을 호출하지 않으면 테스트가 끝난 뒤에도 스레드 풀이 살아 있을 수 있다.

따라서 동시성 테스트가 끝나면 명시적으로 스레드 풀을 종료하는 것이 좋다.

### 5.3. 동시성 테스트 작성 시 주의점

#### 1. 테스트 메서드에 `@Transactional`을 붙이지 않는 것이 좋다

```java
@Test
void concurrencyTest() {
}
```

동시성 테스트에서는 여러 스레드가 각각 서비스 로직을 실행한다.

테스트 메서드에 `@Transactional`을 붙이면 테스트 트랜잭션과 작업 스레드의 트랜잭션 경계가 달라져서, 예상과 다르게 동작할 수 있다.

따라서 동시성 테스트에서는 보통 테스트 메서드에 `@Transactional`을 붙이지 않는다.

#### 2. 테스트 데이터 정리는 직접 한다

`@Transactional` 롤백을 쓰지 않는 대신, 테스트 전후에 데이터를 직접 정리하는 편이 안전하다.

```java
@BeforeEach
void setUp() {
    reservationRepository.deleteAll();
}

@AfterEach
void tearDown() {
    reservationRepository.deleteAll();
}
```

단, 외래 키 관계가 있다면 삭제 순서에 주의해야 해.

예를 들어 예약이 예약 시간을 참조한다면,

1. reservation 삭제
2. reservation_time 삭제

순서로 지워야 한다.

#### 3. 예외 타입을 구체적으로 검증하면 더 좋다

단순히 실패 개수만 세는 것보다, 어떤 예외로 실패했는지도 확인하면 테스트 의도가 더 명확해진다.

```java
Queue<Throwable> exceptions = new ConcurrentLinkedQueue<>();
```

```java
catch (Exception e) {
    exceptions.add(e);
    failCount.incrementAndGet();
}
```

검증은 이렇게 할 수 있다.

```java
assertThat(exceptions)
        .allMatch(e -> e instanceof DuplicatedReservationException
                || e instanceof DataIntegrityViolationException);
```

---

## 6. 데이터베이스 Lock을 코드로 적용해보기

예약 시간 삭제와 예약/예약 대기 생성을 예로 들어보자.

문제 상황은 다음과 같다.

1. **예약 시간 삭제 요청**이 `reservation_time` 존재 여부를 확인한다.
2. 동시에 다른 요청이 같은 `reservation_time`을 **참조하는 예약을 생성**한다.
3. **삭제 요청**은 해당 `reservation_time`을 삭제하려고 한다.
4. **예약 생성 요청**은 이미 삭제될 수 있는 `reservation_time`을 참조하게 된다.

이 상황을 막기 위해, 예약 시간 삭제와 예약 생성이 같은 `reservation_time` `row`를 기준으로 **lock**을 획득하도록 만든다.

### 6.1. Repository에 lock 조회 메서드를 추가한다

```java
@Override
public Optional<ReservationTime> findByIdForUpdate(Long id) {
    String sql = """
           SELECT id, start_at
           FROM reservation_time
           WHERE id = ?
           FOR UPDATE
           """;

    return jdbcTemplate.query(sql, RESERVATION_TIME_ROW_MAPPER, id)
            .stream()
            .findFirst();
}
```

이 메서드는 단순히 예약 시간을 조회하는 것이 아니라, 해당 row에 배타 락을 건다.

따라서 같은 `reservation_time` row에 대해 다른 트랜잭션이 `FOR UPDATE`로 접근하려고 하면, 현재 트랜잭션이 끝날 때까지 대기하게 된다.

### 6.2. 예약 시간 삭제에서 lock 조회를 사용한다

```java
@Transactional
public void removeReservationTimeById(Long id) {
    if (reservationTimeRepository.findByIdForUpdate(id).isEmpty()) {
        throw new TimeNotFoundException();
    }

    try {
        int affectedRow = reservationTimeRepository.deleteById(id);
        int nonAffected = 0;

        if (affectedRow == nonAffected) {
            throw new TimeNotFoundException();
        }
    } catch (DataIntegrityViolationException e) {
        throw new TimeInUseException();
    }
}
```

여기서 중요한 점은 `@Transactional` 안에서 lock 조회와 삭제가 함께 수행되어야 한다는 것이다.

`FOR UPDATE`로 획득한 락은 트랜잭션이 종료될 때 해제된다.

따라서 lock 조회만 하고 트랜잭션이 바로 끝나버리면, 이후 삭제 시점에는 lock의 의미가 사라진다.

### 6.3. 예약 생성에서도 같은 reservation_time row를 lock 조회한다

```java
@Transactional
public Reservation makeReservation(ReservationCommand command) {
    ReservationTime time = getReservationTimeForUpdate(command.timeId());

    // 예약 생성 검증 및 저장 로직
    // ...

    return reservation;
}

private ReservationTime getReservationTimeForUpdate(Long timeId) {
    return reservationTimeRepository.findByIdForUpdate(timeId)
            .orElseThrow(TimeNotFoundException::new);
}
```

예약 생성 시에도 같은 `reservation_time` row를 `FOR UPDATE`로 조회한다.

이렇게 하면 예약 시간 삭제 트랜잭션과 예약 생성 트랜잭션이 같은 row를 기준으로 순서를 맞추게 된다.

### 6.4. 예약 대기 생성에서도 같은 reservation_time row를 lock 조회한다

```java
@Transactional
public ReservationWaiting makeReservationWaiting(ReservationWaitingCommand command) {
    ReservationTime time = reservationTimeRepository.findByIdForUpdate(command.timeId())
            .orElseThrow(TimeNotFoundException::new);

    // 예약 대기 생성 검증 및 저장 로직
    // ...

    return reservationWaiting;
}
```

예약 대기 생성도 예약 생성과 마찬가지로 `reservation_time`을 참조한다.

따라서 예약 시간 삭제와 충돌하지 않도록 같은 기준으로 lock을 획득해야 한다.

### 6.5. 동시성 테스트하기

```java
@SpringBootTest
class ReservationConcurrencyTest {

    @Autowired
    private ReservationService reservationService;

    @Autowired
    private ReservationRepository reservationRepository;

    @Test
    void 같은_슬롯에_동시에_예약을_생성하면_하나만_성공한다() throws InterruptedException {
        // given
        int threadCount = 10;

        ExecutorService executorService = Executors.newFixedThreadPool(threadCount);

        CountDownLatch readyLatch = new CountDownLatch(threadCount);
        CountDownLatch startLatch = new CountDownLatch(1);
        CountDownLatch doneLatch = new CountDownLatch(threadCount);

        AtomicInteger successCount = new AtomicInteger();
        AtomicInteger failCount = new AtomicInteger();

        ReservationCreateRequest request = new ReservationCreateRequest(
                "밤밤",
                LocalDate.of(2026, 5, 28),
                1L,
                1L
        );

        // when
        for (int i = 0; i < threadCount; i++) {
            executorService.submit(() -> {
                try {
                    readyLatch.countDown();
                    startLatch.await();

                    reservationService.create(request);

                    successCount.incrementAndGet();
                } catch (Exception e) {
                    failCount.incrementAndGet();
                } finally {
                    doneLatch.countDown();
                }
            });
        }

        readyLatch.await();
        startLatch.countDown();
        doneLatch.await();

        executorService.shutdown();

        // then
        assertThat(successCount.get()).isEqualTo(1);
        assertThat(failCount.get()).isEqualTo(threadCount - 1);

        long reservationCount = reservationRepository.countByDateAndTimeIdAndThemeId(
                LocalDate.of(2026, 5, 28),
                1L,
                1L
        );

        assertThat(reservationCount).isEqualTo(1);
    }
}
```
