## 단순 조회에도 읽기 전용 트랜잭션이 필요할까?

## 목적

서비스 계층에서 단순 조회만 수행하는 메서드에 `@Transactional(readOnly = true)`를 적용할지 판단하기 위한 기준을 마련한다.

또한 읽기 전용 트랜잭션을 사용한다면, 단순히 습관적으로 적용하는 것이 아니라 어떤 근거로 사용하는지 설명할 수 있도록 한다.

### 비교 대상

서비스 계층에서 단순 조회를 수행할 때 선택할 수 있는 두 가지 방식을 비교한다. 비교 범위는 **성능 관점**으로 한정한다.

```java
@Transactional(readOnly = true)
public ReservationResponse findReservation(Long id) {
    ...
}
```

```java
public ReservationResponse findReservation(Long id) {
    ...
}
```

| 선택지 | 의미 |
| --- | --- |
| 읽기 전용 트랜잭션 사용 | 서비스 조회 메서드 전체를 `readOnly = true` 트랜잭션으로 감싼다 |
| 트랜잭션 없음 | 서비스 조회 메서드에 `@Transactional`을 사용하지 않는다 |

즉, 이 글에서는 일반 트랜잭션과 읽기 전용 트랜잭션을 비교하지 않는다.

오직 **읽기 전용 트랜잭션을 사용하는 경우**와**트랜잭션을 아예 사용하지 않는 경우**만 비교한다.

---

## **읽기 전용 트랜잭션의** 장점

### 1. DB/JDBC에 read-only 힌트를 전달할 수 있다

`@Transactional(readOnly = true)`를 사용하면 **Spring**은 트랜잭션을 시작하면서 **JDBC Connection**에 **read-only** 힌트를 전달할 수 있다.

```java
connection.setReadOnly(true);
```

이 힌트를 실제로 어떻게 활용하는지는 **DB**와 **JDBC 드라이버**에 따라 다르다.

가능한 성능상 이점은 다음과 같다.

- DB가 읽기 전용 트랜잭션으로 인식해 일부 최적화를 적용할 수 있다.
- 쓰기 작업과 관련된 일부 비용을 줄일 가능성이 있다.
- `read-only connection`을 별도로 관리하는 구조에서 최적화 기준으로 사용할 수 있다.

다만 모든 DB에서 성능 향상이 보장되는 것은 아니다. 따라서 이 장점은 사용하는 DB, JDBC 드라이버, 커넥션 풀 설정에 따라 달라진다.

### 2. 읽기/쓰기 DB 분리 구조에서 Replica 라우팅 기준으로 사용할 수 있다

Primary DB와 Replica DB를 분리한 구조에서는 `readOnly = true`가 라우팅 기준으로 사용될 수 있다.

- `@Transactional(readOnly = true)` → Replica DB로 라우팅
- `@Transactional` 없음 → 별도 라우팅 기준이 없다면 Primary DB로 접근할 수 있음

이 경우 읽기 전용 트랜잭션의 장점은 단일 쿼리 속도 향상이 아니다. 핵심은 읽기 요청을 Replica DB로 분산하여 Primary DB의 부하를 줄일 수 있다는 점이다.

따라서 읽기/쓰기 DB 분리 구조를 사용한다면, 단순 조회라도 `readOnly = true`가 성능상 의미를 가질 수 있다.

### 3. JPA 사용 시 영속성 컨텍스트 관련 최적화 여지가 있다

**JPA**를 사용하는 경우, 읽기 전용 트랜잭션은 **Hibernate** 같은 **JPA 구현체**에 **읽기 전용 힌트**를 전달할 수 있다.

이 경우 상황에 따라 다음과 같은 최적화 여지가 생길 수 있다.

- 변경 감지를 위한 스냅샷 관리 비용 감소 가능
- `flush` 관련 비용 감소 가능
- 대량 엔티티 조회 시 관리 비용 감소 가능

다만 이 장점은 **JPA**를 사용할 때의 이야기다. `JdbcTemplate`, `MyBatis`, `순수 JDBC`를 사용하는 경우에는 **dirty checking**이나 `flush` 자체가 없기 때문에 해당하지 않는다.

또한 OSIV 설정, 조회 방식, DTO projection 사용 여부에 따라 실제 이점은 달라진다. 따라서 이 장점은 “JPA를 사용하면 항상 성능이 좋아진다”가 아니라, “JPA 환경에서는 최적화 여지가 생길 수 있다” 정도로 이해하는 것이 적절하다.

## **읽기 전용 트랜잭션의** 단점

### 1. 트랜잭션 시작/종료 비용이 발생한다

읽기 전용이라도 트랜잭션은 트랜잭션이다. 따라서 다음과 같은 비용이 발생할 수 있다.

- 트랜잭션 인터셉터 실행
- 트랜잭션 시작
- 트랜잭션 종료
- `commit` 또는 `rollback` 처리
- `Connection` 상태 변경
- `readOnly` 설정 전달

단순 단건 조회에서는 이러한 부가 비용이 `readOnly` 힌트의 이점보다 클 수 있다.

### 2. 서비스 메서드 동안 리소스를 더 오래 유지할 수 있다

읽기 전용 트랜잭션을 사용하면 서비스 메서드가 끝날 때까지 트랜잭션이 유지된다. **JPA**를 사용하는 경우에는 이 범위 동안 영속성 컨텍스트도 함께 유지된다.

따라서 조회가 매우 단순한 경우에는 다음 비용이 불필요할 수 있다.

- 영속성 컨텍스트 유지 비용
- 조회 엔티티 관리 비용
- 커넥션 또는 트랜잭션 관련 리소스 유지 비용

즉, Repository나 DAO 한 번 호출로 필요한 데이터를 모두 조회하고 바로 반환하는 경우라면, 읽기 전용 트랜잭션이 오히려 불필요한 부가 비용을 만들 수 있다.

---

## 실험해보기

### 실험에 들어가며

앞서 정리한 읽기 전용 트랜잭션의 장단점은 일반적인 관점에서의 이야기다. 실제로 `@Transactional(readOnly = true)`가 성능상 유리한지는 실행 환경과 조회 방식에 따라 달라질 수 있다.

예를 들어 같은 단순 조회라도 다음 조건에 따라 결과가 달라질 수 있다.

- JPA를 사용하는가?
- JdbcTemplate이나 MyBatis처럼 JPA를 사용하지 않는가?
- JPA를 사용한다면 OSIV가 켜져 있는가?
- OSIV가 꺼져 있는가?
- DTO projection으로 바로 조회하는가?
- 엔티티를 조회한 뒤 연관 객체에 접근하는가?
- 읽기/쓰기 DB 분리 구조가 있는가?

따라서 지금부터는 몇 가지 조회 시나리오를 정해두고, 각 상황에서 **읽기 전용 트랜잭션을 사용하는 경우**와 **트랜잭션을 사용하지 않는 경우**를 비교해본다.

실험의 목표는 각 상황에서 두 방식 중 어느 쪽이 성능적으로 더 유리한지 확인하는 것으로 한다.

### 측정 기준

이번 실험에서는 **서비스 조회 메서드 전체 실행 시간**을 기준으로 성능을 측정한다.

측정 범위는 서비스 메서드가 호출된 시점부터 **응답 DTO**가 생성되어 반환되기 직전까지다.

#### 포함되는 비용

- 트랜잭션 시작/종료 비용
- `Repository/DAO` 호출 비용
- DB 조회 비용
- `Entity` 생성 및 매핑 비용
- 응답 DTO 변환 비용
- **JPA** 사용 시 영속성 컨텍스트 관련 비용
- **JPA** 사용 시 지연 로딩으로 인한 추가 조회 비용

서비스 계층의 조회 메서드는 일반적으로 엔티티를 그대로 반환하지 않고 응답 DTO로 변환해 반환한다. 따라서 DTO 변환은 조회 유스케이스의 일부로 보고 측정 범위에 포함한다.

특히 **JPA** 환경에서 `OSIV`를 끈 경우에는 트랜잭션 경계 안에서 DTO 변환을 수행하는지 여부가 지연 로딩 가능 여부에 영향을 줄 수 있다. 따라서 DTO 변환 시간을 제외하지 않는다.

#### 수집할 측정값

각 실험에서는 단일 실행 시간이 아니라 반복 실행 결과를 수집한다.

- 평균
- 중간값
- 최소값
- 최대값
- p95

평균만으로 판단하지 않는다. 일부 실행 시간이 튀면 평균이 왜곡될 수 있기 때문이다. 따라서 중간값과 p95를 함께 확인한다.

#### 결과 해석 기준

결과는 단순히 한 번 더 빠른지를 기준으로 판단하지 않는다.

- 반복 측정에서 일관된 차이가 나타나는가?
- 평균뿐 아니라 중간값과 p95에서도 비슷한 경향이 나타나는가?
- 차이가 측정 오차나 실행 환경의 흔들림보다 충분히 큰가?
- 단건 조회와 대량 조회에서 경향이 다르게 나타나는가?

차이가 작거나 실행마다 방향이 바뀐다면, 성능상 유의미한 근거로 보기 어렵다.

반대로 평균, 중간값, p95에서 일관된 차이가 반복적으로 나타난다면, 해당 환경에서는 `@Transactional(readOnly = true)` 적용 여부가 성능에 영향을 준다고 볼 수 있다.

### 고정할 조건과 그 이유

이번 실험에서는 `@Transactional(readOnly = true)` 적용 여부에 따른 성능 차이를 확인하기 위해, 그 외의 조건은 최대한 동일하게 유지한다.

#### 1. 데이터베이스와 JDBC Driver

`@Transactional(readOnly = true)`는 JDBC `Connection`에 read-only 힌트를 전달할 수 있다.

하지만 이 힌트를 실제로 어떻게 처리하는지는 DB와 JDBC Driver에 따라 달라질 수 있다. 따라서 DB와 JDBC Driver 버전을 명시해야 실험 결과를 해석하기 쉽다.

- DB: MySQL 8.0.x
- JDBC Driver: MySQL Connector/J 8.0.x
- Schema: reservation, time 테이블
- Index: 기본 PK/FK 인덱스 외 추가 인덱스 없음
- Initial Data: 단건 조회용 데이터 1건, 대량 조회용 데이터 50,000건

#### 2. 애플리케이션 실행 환경

Java, Spring, JVM 옵션, 로깅 설정에 따라 실행 시간이 달라질 수 있다. 특히 SQL 로그가 켜져 있으면 측정 결과가 왜곡될 수 있으므로 비활성화한다.

- Java: 17
- Spring Boot: 3.x.x
- Spring Framework: 6.x.x
- Profile: test
- JVM Options: 별도 옵션 없음
- SQL Logging: off

#### 3. 커넥션 풀 설정

커넥션 획득 시간과 커넥션 상태 초기화 비용은 서비스 메서드 실행 시간에 영향을 줄 수 있다. 또한 읽기 전용 트랜잭션은 커넥션의 read-only 상태 설정 및 복구 과정과 관련될 수 있으므로 커넥션 풀 설정을 고정한다.

- Connection Pool: HikariCP
- maximumPoolSize: 10
- minimumIdle: 10
- connectionTimeout: 30000ms
- idleTimeout: 600000ms
- maxLifetime: 1800000ms

#### 4. 데이터 양과 데이터 상태

데이터 양과 데이터 형태는 DB 조회 시간, 객체 생성 시간, DTO 변환 시간에 영향을 준다. 따라서 같은 시나리오 안에서는 동일한 데이터 상태를 유지한다.

또한 실험 중 데이터가 변경되면 조회 결과와 실행 시간이 달라질 수 있으므로, 실험 중에는 조회 외의 쓰기 작업을 수행하지 않는다.

- Single Read: id = 1 데이터 1건 조회
- Bulk Read: reservation 50,000건 조회
- Related Data: reservation 1건당 time 1건 참조
- Data Change During Test: 없음

#### 5. 서비스 테스트 트랜젝션

서비스 테스트 메서드에는 `@Transactional`을 붙이지 않는다.

- 테스트 트랜잭션이 서비스 트랜잭션 비교를 오염시킬 수 있다.
- `@Transactional` 없는 서비스 메서드도 테스트 트랜잭션 안에서 실행될 수 있다.
- readOnly 있음/없음 비교가 정확하지 않아진다.

대신:

- given 데이터 삽입은 측정 구간 밖에서 수행한다.
- 테스트 전후 데이터 초기화는 직접 DELETE/TRUNCATE로 처리한다.

#### 6. 워밍업

첫 실행에는 클래스 로딩, JIT 컴파일, 커넥션 초기화, DB 캐시 상태 등의 영향이 섞일 수 있다. 이 영향이 특정 비교 대상에만 몰리면 측정 결과가 왜곡될 수 있으므로, 실제 측정 전에 워밍업 실행을 수행한다.

- Warm-up Count: 100회
- Warm-up Result: 최종 측정값에서 제외
- Warm-up Order: readOnly 있음 / 없음 교차 실행

#### 7. 측정 방식

측정 방식이 달라지면 결과를 비교하기 어렵다.

워밍업 없이 측정하면 클래스 로딩, JIT 컴파일, 커넥션 초기화, DB 캐시 상태의 영향이 섞일 수 있다. 또한 한쪽을 먼저 몰아서 실행하면 측정 순서에 따른 캐시 영향이 생길 수 있으므로 교차 실행한다.

- Measurement: `System.nanoTime()` 기반 반복 측정
- Warm-up: 100회
- Measurement Count: 1,000회
- Execution Order: readOnly 있음/없음 교차 실행
- Aggregation: 평균, 중간값, 최소값, 최대값, p95

### 시나리오 목록

이번 실험에서는 `@Transactional(readOnly = true)` 적용 여부에 따른 성능 차이를 확인하기 위해, 여러 조회 상황을 시나리오로 나누어 비교한다. 각 시나리오 안에서는 `@Transactional(readOnly = true)` 적용 여부만 다르게 두고, 나머지 조회 조건은 동일하게 유지한다.

#### 시나리오 설계 원칙

각 시나리오는 하나의 조회 유스케이스를 기준으로 구성한다.

같은 시나리오 안에서는 다음 조건을 동일하게 유지한다.

- 동일한 Service 메서드 구조
- 동일한 Repository/DAO 메서드 호출
- 동일한 SQL 또는 JPQL
- 동일한 조회 조건
- 동일한 DTO 클래스
- 동일한 DTO 변환 위치
- 동일한 DTO 변환 로직

이렇게 해야 같은 조회 유스케이스에서 관찰된 성능 차이를 `@Transactional(readOnly = true)` 적용 여부와 관련지어 해석할 수 있다.

#### 시나리오에서 비교할 변수

이번 실험에서는 다음 요소들을 시나리오 변수로 둔다.

- 데이터 접근 기술
    - JdbcTemplate
    - Spring Data JPA
- 조회 데이터 양
    - 단건 조회
    - 대량 조회
- 조회 유스케이스
    - 예약 기본 정보 조회
    - 예약 + 시간 정보 조회

### 1. JdbcTemplate 기반 조회

#### 시나리오 1-1. JdbcTemplate 단건 조회

- Data Access: JdbcTemplate
- 조회 유스케이스: 예약 시간 정보 조회
- Read Size: 단건 조회
- Transaction: readOnly 있음 / 없음
- DTO Conversion: Service 내부

#### 시나리오 1-2. JdbcTemplate 대량 조회

- Data Access: JdbcTemplate
- 조회 유스케이스: 예약 시간 정보 조회
- Read Size: 대량 조회
- Transaction: readOnly 있음 / 없음
- DTO Conversion: Service 내부

### 2. Spring Data JPA 기반 조회

#### 시나리오 2-1. JPA 단건 조회 - 예약 기본 정보 조회

- Data Access: Spring Data JPA
- 조회 유스케이스: 예약 시간 정보 조회
- Read Size: 단건 조회
- OSIV: off
- Transaction: readOnly 있음 / 없음
- DTO Conversion: Service 내부

#### 시나리오 2-2. JPA 대량 조회 - 예약 기본 정보 조회

- Data Access: Spring Data JPA
- 조회 유스케이스: 예약 시간 정보 조회
- Read Size: 대량 조회
- OSIV: off
- Transaction: readOnly 있음 / 없음
- DTO Conversion: Service 내부

#### 시나리오 2-3. JPA 단건 조회 - 예약 + 시간 정보 조회

- Data Access: Spring Data JPA
- 조회 유스케이스: 예약 + 시간 정보 조회
- Read Size: 단건 조회
- OSIV: off
- Transaction: readOnly 있음 / 없음
- DTO Conversion: Service 내부

#### 시나리오 2-4. JPA 대량 조회 - 예약 + 시간 정보 조회

- Data Access: Spring Data JPA
- 조회 유스케이스: 예약 + 시간 정보 조회
- Read Size: 대량 조회
- OSIV: off
- Transaction: readOnly 있음 / 없음
- DTO Conversion: Service 내부

연관 객체 접근 과정에서 추가 쿼리가 발생할 수 있으므로, 결과를 해석할 때 다음 요소를 함께 고려해야 한다.

- 트랜잭션 유지 여부
- 영속성 컨텍스트 유지 여부
- 지연 로딩으로 인한 추가 쿼리 발생 여부
- N+1 발생 여부

### 최종 시나리오 목록

| 번호 | 데이터 접근 기술 | 조회 유스케이스 | 조회 데이터 양 | OSIV | 비교 대상 |
| --- | --- | --- | --- | --- | --- |
| 1-1 | JdbcTemplate | 예약 시간 정보 조회 | 단건 | 해당 없음 | readOnly 있음 / 없음 |
| 1-2 | JdbcTemplate | 예약 시간 정보 조회 | 대량 | 해당 없음 | readOnly 있음 / 없음 |
| 2-1 | Spring Data JPA | 예약 시간 정보 조회 | 단건 | off | readOnly 있음 / 없음 |
| 2-2 | Spring Data JPA | 예약 시간 정보 조회 | 대량 | off | readOnly 있음 / 없음 |
| 2-3 | Spring Data JPA | 예약 + 시간 정보 조회 | 단건 | off | readOnly 있음 / 없음 |
| 2-4 | Spring Data JPA | 예약 + 시간 정보 조회 | 대량 | off | readOnly 있음 / 없음 |

### 시나리오 1-1. 단건 조회 결과

### readOnly

- avg(ms) = 0.294431174
- median(ms) = 0.282042
- min(ms) = 0.197625
- max(ms) = 2.187833
- p95(ms) = 0.390542

### no transaction

- avg(ms) = 0.07263040300000001
- median(ms) = 0.063958
- min(ms) = 0.040833
- max(ms) = 0.460125
- p95(ms) = 0.118584

결론:

- 단건 조회에서는 no transaction이 readOnly 대비 일관되게 더 빠르게 나타났다.
- 단순 조회에서는 readOnly 트랜잭션의 시작/종료 부가 비용이 상대적으로 크게 작용할 수 있다.

### 시나리오 1-2. 대량 조회 결과

### readOnly

- avg(ms) = 25.155393925000002
- median(ms) = 23.294875
- min(ms) = 22.082541
- max(ms) = 105.316667
- p95(ms) = 32.36

### no transaction

- avg(ms) = 24.27762814
- median(ms) = 22.828917
- min(ms) = 21.457667
- max(ms) = 151.167375
- p95(ms) = 30.359583

결론:

- 대량 조회에서도 no transaction이 평균/중간값/p95 기준으로 소폭 우세했다.
- 두 방식의 차이는 단건 조회에 비해 작으므로, 환경 변화나 반복 측정에 따라 결과가 달라질 여지가 있다.
- 데이터의 양이 크면 클수록 트랜잭션으로 인한 오버헤드의 비중이 작아지기 때문인 것으로 판단된다.

### 시나리오 2-1. JPA 단건 조회(예약 시간 정보 조회) 결과

### readOnly

- avg(ms) = 0.236382122
- median(ms) = 0.230291
- min(ms) = 0.187208
- max(ms) = 0.501458
- p95(ms) = 0.2765

### no transaction

- avg(ms) = 0.237616057
- median(ms) = 0.229792
- min(ms) = 0.187
- max(ms) = 2.046458
- p95(ms) = 0.272041

결론:

- 평균/중간값/p95 기준으로 두 방식의 차이가 매우 작게 나타났다.
- 이 시나리오에서는 readOnly 적용 여부가 성능에 미치는 영향이 거의 없다고 볼 수 있다.

### 시나리오 2-2. JPA 대량 조회(예약 시간 정보 조회) 결과

### readOnly

- avg(ms) = 45.545437485
- median(ms) = 44.868958
- min(ms) = 39.4845
- max(ms) = 54.152167
- p95(ms) = 51.227375

### no transaction

- avg(ms) = 45.807414825
- median(ms) = 45.444334
- min(ms) = 39.398667
- max(ms) = 54.81075
- p95(ms) = 51.007583

결론:

- 평균/중간값 기준으로는 readOnly가 아주 근소하게 우세했지만, p95는 no transaction이 근소하게 우세했다.
- 두 방식의 차이가 매우 작아, 이 시나리오에서는 성능상 우열이 뚜렷하다고 보기 어렵다.

### 시나리오 2-3. JPA 단건 조회(예약 + 시간 정보 조회) 결과

### readOnly

- avg(ms) = 0.46205617
- median(ms) = 0.372292
- min(ms) = 0.253708
- max(ms) = 2.263334
- p95(ms) = 0.908542

### no transaction

- avg(ms) = 0.462280906
- median(ms) = 0.372
- min(ms) = 0.255292
- max(ms) = 1.697125
- p95(ms) = 0.932417

결론:

- 평균/중간값은 사실상 동일하고, p95는 readOnly가 아주 근소하게 우세했다.
- 차이가 작기 때문에 단건 조회에서는 readOnly 적용 유무보다 다른 요인(쿼리/매핑/환경)이 결과에 더 크게 작용할 수 있다.

### 시나리오 2-4. JPA 대량 조회(예약 + 시간 정보 조회) 결과

### readOnly

- avg(ms) = 119.465370795
- median(ms) = 117.083708
- min(ms) = 100.966
- max(ms) = 269.674791
- p95(ms) = 141.486875

### no transaction

- avg(ms) = 121.878623495
- median(ms) = 116.382875
- min(ms) = 103.136958
- max(ms) = 350.338834
- p95(ms) = 151.961

결론:

- 평균과 p95 기준으로는 readOnly가 더 안정적으로 나타났고, max에서도 readOnly의 스파이크가 더 작았다.
- 중간값은 no transaction이 근소하게 우세했지만, 전체적으로는 일부 요청 구간의 지연 시간(p95, max) 측면에서 readOnly가 더 유리한 결과로 해석할 수 있다.
