# Spring Boot에서 SQL 데이터베이스 사용하기

## DBMS 의존성 추가하기

### h2 의존성 추가하기

```groovy
dependencies {
		runtimeOnly 'com.h2database:h2'
    testRuntimeOnly 'com.h2database:h2' // 테스트에서 h2 사용 시 추가
}
```

### Mysql 의존성 추가하기

```groovy
dependencies {
		runtimeOnly 'com.mysql:mysql-connector-j'
}
```

- `com.mysql.jdbc.Driver` 도 MySQL 라이브러리로 사용 가능하지만, MySQL 8.x 부턴 최신 버전인 `com.mysql.cj.jdbc.Driver` 를 사용하는 것이 좋다.

### 참고: 왜 `runtimeonly`로 의존성을 추가해야할까?

- 애플리케이션 코드는`javax.sql.DataSource`, `JdbcTemplate` 같은 표준 **JDBC/Spring** 추상화에 의존하고, **DB 드라이버**는 **컴파일 시** 코드에서 직접 참조하기보다, **실행 시점**에 실제 **DB** 연결을 만들기 위해 필요하기 때문이다.

### 참고: Gradle Configuration 종류

| 방식 | 의미 | 대표 예시 |
| --- | --- | --- |
| `implementation` | 컴파일과 런타임에 모두 필요하지만, 다른 모듈에 노출하지 않음 | 일반 라이브러리 |
| `api` | 컴파일과 런타임에 필요하고, 다른 모듈에도 노출함 | 라이브러리 모듈의 공개 API |
| `compileOnly` | 컴파일할 때만 필요하고 런타임에는 필요 없음 | Lombok |
| `runtimeOnly` | 컴파일에는 필요 없고 런타임에만 필요함 | DB 드라이버 |
| `testImplementation` | 테스트 컴파일/실행에 필요 | JUnit, AssertJ |
| `testCompileOnly` | 테스트 컴파일에만 필요 | 테스트용 어노테이션 |
| `testRuntimeOnly` | 테스트 실행 시에만 필요 | JUnit Platform Launcher 등 |
| `annotationProcessor` | 어노테이션 처리기에 사용 | Lombok annotation processor |

## DataSource 구성하기

### 참고: DataSource란?

- 물리적인 데이터 소스에 대한 커넥션을 생성하는 팩토리 클래스
    - 애플리케이션이 데이터베이스와 연결할 때 직접 `DriverManager`를 사용하기보다, `DataSource`를 통해 커넥션을 얻는 방식이 권장된다.
    - Spring Boot에서는 보통 하나의 `DataSource` 빈을 전역에서 사용하며, 이를 기반으로 `JdbcTemplate`, JPA, 트랜잭션 매니저 등이 동작한다.

### Spring Boot에서 DataSource 설정 및 등록하기

- Spring Boot는 `application.yml` 또는 `application.properties` 같은 설정 파일을 바탕으로 `DataSource`를 자동으로 설정하고 빈으로 등록해준다.
- 일반적으로 다음 속성을 지정한다.
    - `driver-class-name`
    - `url`
    - `username`
    - `password`

#### `application.yml` 예시

```yaml
spring:
  datasource:
    driver-class-name: org.h2.Driver
    url: jdbc:h2:mem:database
    username: username
    password: password
```

#### `application.properties` 예시

```
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.url=jdbc:h2:mem:database
spring.datasource.username=username
spring.datasource.password=password
```

### Embedded In-Memory DB 설정하기

- Spring Boot는 `H2`, `HSQL`, `Derby` 같은 Embedded DB에 대해 필요한 설정을 자동으로 구성할 수 있다.
    - 따라서 해당 DB 의존성이 클래스패스에 있고, 별도의 `spring.datasource.url`을 지정하지 않으면 Spring Boot가 자동으로 Embedded DB용 `DataSource`를 구성한다.
- 이 경우 다음 값들을 직접 설정하지 않아도 된다.
    - `spring.datasource.url`
    - `spring.datasource.username`
    - `spring.datasource.password`
    - `spring.datasource.driver-class-name`
- 자동 구성 시 Spring Boot는 임의의 데이터베이스 이름을 사용해 JDBC URL을 생성한다. 또한 기본 사용자 이름은 `SA`, 비밀번호는 빈 값으로 설정된다.

#### 확인 코드

```java
@Test
void embedded_db_자동_설정() {
    try (Connection connection = jdbcTemplate.getDataSource().getConnection()) {
        assertThat(connection).isNotNull();

        System.out.println("driverName: " + connection.getMetaData().getDriverName());
        System.out.println("url: " + connection.getMetaData().getURL());
        System.out.println("username: " + connection.getMetaData().getUserName());
    } catch (SQLException e) {
        throw new RuntimeException(e);
    }
}
```

#### 자동 구성 결과 예시

```
driverName: H2 JDBC Driver
url: jdbc:h2:mem:3510d138-2ae3-4e04-bf41-3569e638c925
username: SA
```

### DataSource 추가 등록하기

- Spring Boot에서 기본 `DataSource` 외에 추가 `DataSource`를 등록할 수도 있다.
    - 이때 기본 `DataSource`와 추가 `DataSource`가 서로 충돌하지 않도록, 추가로 등록하는 `DataSource` 관련 빈에는 `defaultCandidate = false` 옵션을 지정할 수 있다.

#### `application.yml` 예시

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost/first
    username: user1
    password: pwd1
    configuration:
      maximum-pool-size: 30

app:
  datasource:
    url: jdbc:mysql://localhost/second
    username: user2
    password: pwd2
    configuration:
      maximum-pool-size: 30
```

#### 추가 `DataSource` 등록 예시

```java
@Configuration(proxyBeanMethods = false)
public class AdditionalDataSourceConfiguration {

    @Qualifier("second")
    @Bean(defaultCandidate = false)
    @ConfigurationProperties("app.datasource")
    public DataSourceProperties secondDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Qualifier("second")
    @Bean(defaultCandidate = false)
    @ConfigurationProperties("app.datasource.configuration")
    public HikariDataSource secondDataSource(
            @Qualifier("second") DataSourceProperties dataSourceProperties
    ) {
        return dataSourceProperties.initializeDataSourceBuilder()
                .type(HikariDataSource.class)
                .build();
    }
}
```

이렇게 하면 `spring.datasource` 기반의 기본 `DataSource`와 별도로, `app.datasource` 기반의 추가 `DataSource`를 등록할 수 있다.

## Database 초기화하기

### Basic SQL 스크립트로 DB 초기화하기

- **Spring Boot**는 **SQL 스크립트**를 사용해 **애플리케이션 시작 시점**에 **데이터베이스를 초기화**할 수 있다.
    - 기본적으로 Spring Boot는 다음 위치의 SQL 파일을 찾아 실행한다.
        - `optional:classpath*:schema.sql`
        - `optional:classpath*:data.sql`
    - `schema.sql`은 테이블 생성과 같은 **스키마 초기화**에 사용하고, `data.sql`은 **초기 데이터 삽입**에 사용한다.

#### 기본 스크립트 위치

- Spring Boot는 기본적으로 다음 경로에서 SQL 스크립트를 찾는다.

```
optional:classpath*:schema.sql
optional:classpath*:data.sql
```

- 여기서 `optional:` prefix는 해당 파일이 없어도 애플리케이션을 정상적으로 시작한다는 의미이다.
    - 즉, `schema.sql`이나 `data.sql`이 존재하지 않아도 애플리케이션 시작이 실패하지 않는다.

#### 스크립트 위치 변경하기

- 기본 위치가 아닌 다른 경로에 SQL 스크립트를 두고 싶다면 설정 파일에서 위치를 변경할 수 있다.
- `application.yml` 예시

    ```yaml
    spring:
      sql:
        init:
          schema-locations: classpath:db/schema.sql
          data-locations: classpath:db/data.sql
    ```

- 위 설정을 사용하면 다음 위치의 파일을 사용해 DB를 초기화한다.

    ```
    src/main/resources/db/schema.sql
    src/main/resources/db/data.sql
    ```


#### SQL 초기화 실행 모드 설정하기

- SQL 스크립트 기반 초기화를 실행하려면 `spring.sql.init.mode` 설정을 사용할 수 있다.

```yaml
spring:
  sql:
    init:
      mode: always
```

- `spring.sql.init.mode`의 주요 값은 다음과 같다.

| 값 | 설명 |
| --- | --- |
| `always` | 항상 SQL 스크립트를 사용해 DB를 초기화한다. |
| `embedded` | Embedded DB를 사용할 때만 초기화한다. |
| `never` | SQL 스크립트 초기화를 사용하지 않는다. |
- Embedded In-Memory DB를 사용하는 경우에는 기본값이 `embedded`이므로 별도의 설정 없이도 `schema.sql`, `data.sql`이 실행될 수 있다.
- 반면 MySQL 같은 외부 DB를 사용하는 경우에는 보통 다음과 같이 `always`로 명시해야 SQL 스크립트가 실행된다.

#### JPA와 함께 사용하는 경우

- SQL 스크립트 기반 초기화는 JPA의 `EntityManagerFactory` 빈이 생성되기 전에 실행된다.
    - 따라서 JPA를 사용하는 경우에도 `schema.sql`, `data.sql`을 사용해 데이터베이스를 초기화할 수 있다.

### 데이터베이스 마이그레이션 도구 사용하기

- 운영 환경에서 스키마 변경 이력을 관리해야 한다면 `schema.sql`, `data.sql`보다 **데이터베이스 마이그레이션 도구**를 사용하는 것이 좋다.
    - **Spring Boot**는 대표적으로 `Flyway`, `Liquibase`를 지원한다.
    - **마이그레이션 도구**를 사용하면 `테이블 생성`, `컬럼 추가`, `인덱스 추가` 등의 변경 사항을 버전 단위로 관리할 수 있다.

#### Spring Boot에서 Flyway를 처리하는 방식

- **Spring Boot**는 `Flyway` **의존성**이 클래스패스에 있으면, `Flyway`를 자동으로 설정한다.
    - 즉, 별도의 **설정 클래스**를 만들지 않아도 된다.
    - 기본적으로 **애플리케이션 시작 시점**에 마이그레이션을 실행한다.

#### Flyway 실행 시점

- Spring Boot는 애플리케이션 시작 과정에서 `Flyway.migrate()`를 호출한다.
    - 이때 아직 DB에 적용되지 않은 **마이그레이션 파일**만 실행된다.
    - 이미 실행된 마이그레이션 정보는 `Flyway`가 별도 테이블로 관리한다.

```groovy
애플리케이션 시작
→ DataSource 설정
→ Flyway 자동 설정
→ Flyway.migrate() 호출
→ 미적용 마이그레이션 실행
```

#### Flyway 사용하기

- **Spring Boot**에서 `Flyway`를 사용하려면 `Flyway` 의존성을 추가해야 한다.
    - **In-memory DB**나 **file-based DB**는 **기본 Flyway 의존성**만으로 사용할 수 있다.
    - **MySQL**, **PostgreSQL** 같은 DB는 **데이터베이스별 Flyway 모듈**을 추가로 등록해야 한다.

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-flyway'
    runtimeOnly 'org.flywaydb:flyway-mysql' // MySQL 사용 시 추가
}
```

#### Flyway 마이그레이션 파일 작성하기

- Flyway 마이그레이션 파일은 기본적으로 다음 경로에 둔다.

```
src/main/resources/db/migration
```

- 파일 이름은 다음 형식을 따른다.

```
V<버전>__<이름>.sql
```

- 예시

```
V1__create_member_table.sql
V2__add_email_column.sql
V3__insert_initial_data.sql
```

- Spring Boot는 애플리케이션 시작 시점에 아직 적용되지 않은 마이그레이션 파일을 자동으로 실행한다.

#### Flyway 마이그레이션 경로 변경하기

- 기본 경로가 아닌 다른 위치에 마이그레이션 파일을 두고 싶다면 `spring.flyway.locations` 설정을 사용한다.

```yaml
spring:
  flyway:
    locations: classpath:db/migration
```

- 여러 경로를 함께 지정할 수도 있다.

```yaml
spring:
  flyway:
    locations: classpath:db/migration,classpath:dev/db/migration
```

#### 테스트 전용 Flyway 마이그레이션 사용하기

- 테스트에서만 사용할 마이그레이션 파일은 `src/test/resources/db/migration` 경로에 둘 수 있다.

```
src/test/resources/db/migration/V9999__test_data.sql
```

- 이 파일은 테스트 실행 시에만 적용되고, 실제 애플리케이션 `jar`에는 포함되지 않는다.

### 참고: Liquibase

- `Liquibase`도 **Spring Boot**에서 지원하는 **데이터베이스 마이그레이션 도구**이다.
- `Flyway`가 **SQL 파일 기반**의 단순한 버전 관리에 적합하다면, `Liquibase`는 **YAML**, **XML**, **JSON**, **SQL** 등 다양한 형식으로 변경 이력을 관리할 수 있다.

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-liquibase'
}
```

## 추가: 존재 여부 쿼리는 어떻게 작성할까?

### 1. `SELECT 1 ... LIMIT 1`

#### 의미

- 조건에 맞는 **row**를 1 개 조회한다.

#### 예시

```java
public boolean existsByDateAndTimeId(LocalDate date, Long timeId) {
    String sql = """
            SELECT 1
            FROM reservation
            WHERE date = ?
              AND time_id = ?
            LIMIT 1
            """;

    return jdbcTemplate.query(sql, rs -> rs.next(), date, timeId);
}
```

#### 장점

- JDBC에서 `rs.next()`로 처리할 수 있기 때문에, 타입 매핑에 신경쓸 필요가 없다.

### 2. `SELECT EXISTS(SELECT 1 ... LIMIT 1)`

#### 의미

- 조건에 맞는 **row**가 존재하는지 확인한다.

#### 예시

```java
public boolean existsByDateAndTimeId(LocalDate date, Long timeId) {
    String sql = """
            SELECT EXISTS (
                SELECT 1
                FROM reservation
                WHERE date = ?
                  AND time_id = ?
            )
            """;

    Boolean exists = jdbcTemplate.queryForObject(sql, Boolean.class, date, timeId);
    return Boolean.TRUE.equals(exists);
}
```

#### 장점

- 존재 여부 확인 의도가 SQL에 직접 드러난다.

### 3. `SELECT COUNT(*)`

#### 의미

- 조건에 맞는 **row**의 개수를 조회한다.

#### 예시

```java
public boolean existsByDateAndTimeId(LocalDate date, Long timeId) {
    String sql = """
            SELECT COUNT(*)
            FROM reservation
            WHERE date = ?
              AND time_id = ?
            """;

    Integer count = jdbcTemplate.queryForObject(sql, Integer.class, date, timeId);
    return count != null && count > 0;
}
```

#### 장점

- 구현이 간단하다.
