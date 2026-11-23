# @Transactional은 DB 연결을 어떻게 붙잡고 있을까?

[목차](README.md) · [이전: DispatcherServlet은 요청을 어떻게 내 메서드까지 보낼까?](dispatcher-servlet.md)

## 다루는 범위

어댑터가 부른 메서드 안, 우리가 직접 짜는 비즈니스 로직에서 일어나는 일을 정리한다. [전체 흐름](README.md#전체-흐름)의 7단계다.

- `@Transactional` 프록시와 트랜잭션 매니저가 하는 일
- DB 연결을 스레드에 묶어 두는 `ThreadLocal`
- HikariCP: 연결을 빌려주고 돌려받는 커넥션 풀
- JDBC와 드라이버, `PreparedStatement`
- 연결을 들고 있는 시간과, 외부 API가 느릴 때 스레드 200개·연결 10개가 바닥나는 이유
- 타임아웃, 서킷 브레이커, 벌크헤드
- Nest에는 `ThreadLocal`이 없어서 어떻게 하는지

예시는 주문 요청(`POST /api/orders/{shopId}`)이고, 코드와 숫자는 설명용이다.

## 시작한 질문

```java
@Service
class OrderService {
    @Transactional
    Order create(OrderRequest req) {
        var p = productRepo.findById(req.productId());
        p.decreaseStock(req.quantity());
        return orderRepo.save(new Order(p, req));
    }
}
```

1. 상품을 찾고, 재고를 줄이고, 주문을 저장하는 세 작업은 한 덩어리로 성공하거나 실패해야 한다. 리포지토리를 따로따로 부르는데 어떻게 하나의 트랜잭션이 될까?
2. 결제 같은 외부 API를 이 메서드 안에서 부르면 무엇이 문제일까?

## 내가 이해한 내용

### 1. 프록시가 호출을 가로채고, 트랜잭션 매니저가 연결을 빌려 스레드에 묶는다

`@Transactional`이 붙으면 Spring은 `OrderService` 대신 그걸 감싼 프록시를 빈으로 등록한다. 컨트롤러가 `create()`를 부르면 프록시가 먼저 받는다.

1. 트랜잭션 시작: 프록시가 호출을 가로챈다.
2. 연결 빌리기: 트랜잭션 매니저(`DataSourceTransactionManager`, JPA면 `JpaTransactionManager`)가 HikariCP에서 DB 연결을 하나 빌리고 `setAutoCommit(false)`로 바꾼다.
3. 스레드에 묶기: 그 연결을 **지금 스레드(exec-3)의 전용 보관함, `ThreadLocal`**에 넣는다(`TransactionSynchronizationManager`).
4. 같은 연결 쓰기: `productRepo`와 `orderRepo`는 새로 빌리지 않고 이 보관함에서 같은 연결을 꺼내 쓴다. 그래서 하나의 트랜잭션이 된다.
5. 끝: 정상으로 끝나면 커밋, 예외면 롤백하고, 연결을 보관함에서 꺼내 HikariCP에 돌려준다.

`ThreadLocal`에 묶는 방식이 되는 이유는 **요청 하나를 스레드 하나가 끝까지 처리하기 때문**이다([1장](request-to-thread.md#4-연결-하나가-아니라-요청-하나에-스레드-하나)). 같은 요청 안의 코드는 다 같은 스레드에서 돌아서, 그 스레드의 보관함을 열면 같은 연결이 나온다.

알아둘 점:

- 기본으로 롤백하는 건 `RuntimeException`(과 `Error`)일 때다. 체크 예외는 커밋된다. `rollbackFor`로 바꿀 수 있다.
- 같은 클래스 안에서 `this.create()`처럼 부르면 프록시를 거치지 않아 `@Transactional`이 동작하지 않는다(자기 호출 문제).

### 2. HikariCP: 연결을 빌려주고 돌려받기만 한다

- 서버가 켜질 때 DB와 연결을 미리 맺어 둔다. 기본 최대 10개(`maximumPoolSize`). TCP 연결과 DB 로그인까지 끝낸 상태라서 빌릴 때 빠르다.
- 빌릴 연결이 없으면 기다린다. 최대 30초(`connectionTimeout`)를 기다리다 못 빌리면 예외가 난다.
- 연결의 `close()`는 끊는 게 아니라 **풀로 반납**이다.
- 오래된 연결은 30분(`maxLifetime`)마다 새로 바꾼다.

### 3. JDBC: 빌린 연결로 실제 DB 작업을 한다

- JDBC는 자바가 DB와 대화하는 **표준 인터페이스**(`java.sql`)다.
- 실제 구현인 **드라이버는 DB 회사가 만든다**(MySQL Connector/J, PostgreSQL JDBC 등).
- JPA(Hibernate), MyBatis, JdbcTemplate은 JDBC를 대신 써 주는 쪽이다. 어떤 기술을 써도 결국 아래에서는 JDBC 코드가 돈다.

```
exec-3 ──① 연결 빌려줘──────────▶ HikariCP
       ◀─② 연결 #3 (미리 만든 것)──
       ③ ThreadLocal에 보관
       ──④ SQL 틀 + 파라미터 (PreparedStatement)──▶ DB
       ◀─⑤ 결과 행 (ResultSet) → 객체로 매핑────────
       ──⑥ COMMIT──────────────────────────────▶ DB
       ──⑦ 연결 반납 (close)──▶ HikariCP
```

①②⑦은 HikariCP의 일, ④⑤⑥은 JDBC의 일이다.

- `PreparedStatement`는 SQL 틀과 파라미터를 따로 보낸다. 파라미터가 SQL로 해석되지 않아서 SQL 인젝션을 막는다. MyBatis의 `#{}`는 이 방식이라 안전하고, `${}`는 문자열을 그대로 이어 붙여서 위험하다.
- JPA는 `save()`를 부른 순간이 아니라 보통 커밋 직전(flush)에 SQL을 한꺼번에 보낸다. 다만 ID를 DB가 만드는 `IDENTITY` 전략이면 `save()` 때 바로 INSERT한다.

### 4. 연결은 ②부터 ⑦까지 계속 붙잡혀 있다

트랜잭션이 시작될 때 빌린 연결은 메서드가 끝나서 반납될 때까지 계속 그 스레드가 쥐고 있다. 연결은 10개뿐이라 이 시간이 길어질수록 다른 요청이 연결을 기다린다. 그래서 **트랜잭션은 짧게** 가져간다.

### 5. 연결을 들고 있는 시간과 외부 API

외부 결제 API(PG)의 응답이 평소와 달리 10초씩 걸리고, 결제 요청이 초당 20건 들어온다고 해 보자. 기다리는 동안 exec-3에게 붙잡히는 게 두 가지다.

| 붙잡히는 것 | 계산 | 결과 | 대응 |
| --- | --- | --- | --- |
| 스레드 (기본 최대 200) | 초당 20건 × 10초 = 200개 | 새 요청이 줄을 서고, 결제와 상관없는 `/api/health`도 응답을 못 한다 | 외부 API를 부를 때 **기다리는 우리 쪽에서** 타임아웃을 건다 |
| DB 연결 (기본 10) | 트랜잭션 안에서 부르면 초당 20건이면 0.5초 만에 10개가 다 붙잡힌다 | DB를 쓰는 다른 요청이 전부 연결을 기다린다(최대 30초 후 예외). 스레드보다 연결이 먼저 바닥난다 | 외부 호출은 **트랜잭션 밖으로** 뺀다 |

더 나아간 대비책:

- **타임아웃**: 호출 하나하나에 "이 이상은 안 기다린다"를 건다. 기다리는 쪽이 자기를 지키려고 걸고, **안쪽일수록 짧게** 잡는다(예: 결제 API 3초 < nginx 60초). 반대면 nginx가 먼저 포기해서 사용자는 에러를 받는데, 서버는 모르고 계속 기다린다.
- **서킷 브레이커**: 실패가 계속되면 당분간 아예 부르지 않고 바로 실패로 처리한다. 가끔 한 번씩만 다시 시험해 본다.
- **벌크헤드(칸막이)**: 외부 호출이 동시에 쓸 수 있는 양을 제한한다(예: 결제 호출 최대 20개). 넘치면 기다리지 않고 바로 실패해서, 나머지 스레드는 다른 일을 계속한다.
- Spring에서는 Resilience4j로 셋 다 쓸 수 있다.

### 6. Nest와 비교

- Node에는 요청이 스레드에 묶이지 않아서 `ThreadLocal` 방식을 쓸 수 없다.
- 그래서 트랜잭션을 **명시적으로 넘기거나**(TypeORM의 `EntityManager`, Prisma의 `tx`를 함수 인자로 전달), **`AsyncLocalStorage`**로 요청 맥락을 이어 붙인다(typeorm-transactional 같은 라이브러리가 이 방식이다).
- `pg` 커넥션 풀도 기본 최대 10개이고, 파라미터를 `$1`로 따로 보낸다(`PreparedStatement`와 같은 생각).
- 외부 API를 `await`하는 동안 이벤트 루프는 다른 요청을 처리해서 스레드가 동나지는 않는다. 하지만 트랜잭션 안에서 기다리면 커넥션 풀은 똑같이 바닥나고, 타임아웃이 없으면 기다리는 요청과 소켓이 계속 쌓인다.

## 헷갈렸던 점

| 처음의 생각 | 정리한 내용 |
| --- | --- |
| 트랜잭션이 HikariCP를 통해 커넥션 하나를 ThreadLocal에 보관한다는데, 누가 하는 건지 | 프록시가 가로채면 트랜잭션 매니저가 HikariCP에서 빌려 지금 스레드의 `ThreadLocal`에 넣는다. 리포지토리는 거기서 꺼내 쓴다 |
| JDBC 드라이버는 JPA, MyBatis가 서로 구현한다 | 드라이버는 DB 회사가 JDBC 인터페이스를 구현한 것이다. JPA, MyBatis는 JDBC를 사용하는 쪽이다 |
| JDBC는 그냥 개념인데 드라이버를 DB가 구현하는 건가 | 맞다. JDBC는 `java.sql`의 표준 인터페이스이고, 각 DB 회사가 그 인터페이스의 구현체(드라이버)를 배포한다 |
| 연결의 `close()`를 부르면 연결이 끊긴다 | 커넥션 풀에서 `close()`는 반납이다. 실제 TCP 연결은 살아 있다 |
| "exec-3은 기다린다"를 "exec-3을 기다린다"로 읽었다 | exec-3이 `executeQuery`에서 DB 응답을 기다리며 멈춰 있다(블로킹)는 뜻이다 |
| 스레드가 200개나 있으니 외부 API가 조금 느려도 괜찮다 | 트랜잭션 안에서 부르면 연결 10개가 먼저 바닥난다. 초당 20건이면 0.5초다 |
| 타임아웃은 호출받는 쪽이 건다 | 기다리는 쪽(호출하는 쪽)이 자기를 지키려고 건다 |
| 타임아웃과 서킷 브레이커는 같은 것이다 | 타임아웃은 한 번의 호출을 끊는다. 서킷 브레이커는 실패가 이어지면 아예 부르지 않는다 |

### 공부하며 던진 질문

- **HikariCP는 연결을 언제 만들어?** 서버가 켜질 때 미리 만든다. TCP 연결과 로그인까지 끝내 둔다.
- **PreparedStatement는 왜 안전해?** SQL 틀을 먼저 보내고 파라미터는 값으로만 따로 보내서, 파라미터가 SQL 문법으로 해석되지 않는다.
- **외부 API 전용 스레드 풀을 따로 두면?** 그게 벌크헤드다. 그 칸만 차고 나머지 스레드는 살아남는다.

## 예제와 확인 결과

설명용이다. 직접 실행하지 않았다. JPA, MyBatis가 대신 써 주는 JDBC 코드는 대략 이렇다.

```java
Connection conn = dataSource.getConnection();   // HikariCP에서 빌리기
conn.setAutoCommit(false);                      // 트랜잭션 시작
PreparedStatement ps = conn.prepareStatement(
        "SELECT * FROM product WHERE id = ?");
ps.setLong(1, id);                              // 파라미터는 따로
ResultSet rs = ps.executeQuery();               // DB로 보내서 실행 (여기서 기다린다)
while (rs.next()) { … }                         // 결과 행 → 객체
conn.commit();                                  // 확정
conn.close();                                   // 끊지 않고 풀로 반납
```

## 남은 질문

- 트랜잭션 전파(`REQUIRES_NEW` 등)를 쓰면 연결을 하나 더 빌릴까? 그때 풀 고갈 위험은?
- `readOnly = true`는 실제로 무엇을 바꿀까?
- OSIV(Open Session In View)가 켜져 있으면 연결을 들고 있는 시간이 어디까지 늘어날까?
- AOP 프록시는 JDK 동적 프록시와 CGLIB 중 무엇으로 만들어질까?
- 가상 스레드에서도 `ThreadLocal` 방식이 그대로 괜찮을까?

## 참고 자료

이번 공부는 발표("요청 하나, 스레드 하나", 2026-11-24)를 준비하며 Claude와 대화로 진행했다. 더 읽어볼 자료:

- [Spring Framework - 트랜잭션 관리](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [HikariCP](https://github.com/brettwooldridge/HikariCP)
- [Resilience4j](https://resilience4j.readme.io/)

## 정리 이력

- 2026-11-23: 발표 준비로 공부한 내용을 노트로 정리했다.
