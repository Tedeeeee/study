# AOP 프록시는 어떻게 @Transactional을 감쌀까?

[목차](README.md) · [이전: 리플렉션은 어떻게 어노테이션을 읽고 메서드를 부를까?](reflection.md)

## 다루는 범위

[6번](bean-creation.md)에서 후처리기가 `@Transactional` 빈을 프록시로 바꿔치기한다는 것까지 봤다. 이 노트는 그 프록시가 무엇을 하고, 어떻게 만들어지고, 어디서 함정이 생기는지 정리한다.

- AOP: 공통으로 필요한 일을 떼어 내 메서드 앞뒤에 끼우기, 어드바이스와 포인트컷
- 프록시를 만드는 두 방법: JDK 동적 프록시와 CGLIB, Spring Boot가 CGLIB를 기본으로 쓰는 이유
- CGLIB 프록시의 실제 모양, 원본·프록시·컨트롤러 사이의 주입 관계
- `final`: 필드는 괜찮고 클래스·메서드가 문제인 이유, `final` 메서드가 NPE로 이어지는 과정
- 자기 호출 문제와 피하는 법
- Nest와 비교

코드는 설명용이고, Parent·Child 예제만 직접 실행했다.

## 시작한 질문

1. 프록시는 안에서 무슨 일을 할까? AOP는 정확히 뭘까?
2. 프록시는 어떻게 만들어질까? JDK 동적 프록시와 CGLIB는 뭐가 다를까?
3. `final`이 붙으면 왜 안 될까? 필드에 붙은 `final`도 문제일까?
4. 같은 클래스 안에서 부른 `@Transactional`은 왜 안 걸릴까?

## 내가 이해한 내용

### 1. AOP: 공통으로 필요한 일을 떼어 내 끼워 넣기

AOP 없이 트랜잭션을 직접 다루면 메서드마다 같은 틀이 붙는다.

```java
public void order() {
    TransactionStatus tx = txManager.getTransaction(new DefaultTransactionDefinition());
    try {
        orderRepository.save(...);        // 진짜 하고 싶은 일
        txManager.commit(tx);
    } catch (RuntimeException e) {
        txManager.rollback(tx);
        throw e;
    }
}
```

트랜잭션, 로그, 캐시, 권한 확인처럼 여러 곳에 똑같이 필요한 일을 비즈니스 코드에서 떼어 내 한곳에 두고, 필요한 메서드 앞뒤에 끼워 넣는 방식이 AOP다. Spring은 이걸 프록시로 한다. 그러면 `@Transactional`과 진짜 일만 남는다.

| 말 | 뜻 | 트랜잭션에서는 |
| --- | --- | --- |
| 어드바이스 (Advice) | 무엇을 끼워 넣나 | 트랜잭션 열고 커밋·롤백 (`TransactionInterceptor`) |
| 포인트컷 (Pointcut) | 어디에 끼워 넣나 | `@Transactional`이 붙은 메서드 |

```
컨트롤러 → 프록시.order()
            ├ 포인트컷에 걸리나? (@Transactional 있나?)
            ├ 걸리면 → TransactionInterceptor: 트랜잭션 시작 (커넥션을 ThreadLocal에, 4장)
            │              → 원본.order()
            │           ← 정상이면 커밋, RuntimeException이면 롤백
            └ 안 걸리면 → 바로 원본 메서드
```

끼울 일이 여러 개면(`@Transactional` + `@Cacheable`) 프록시 안에서 차례로 실행된다(인터셉터 체인). AOP인지 아닌지는 "읽고 끝나느냐, 부를 때마다 앞뒤에 일이 끼어드느냐"로 가른다([7번](reflection.md)).

### 2. 프록시를 만드는 두 방법

**JDK 동적 프록시**는 인터페이스를 구현한 가짜를 실행 중에 만든다. 자바 기본 기능이고, 원본을 부를 때 7번에서 본 `method.invoke`를 쓴다.

```java
interface OrderService { void order(); }
class OrderServiceImpl implements OrderService { ... }

OrderService proxy = (OrderService) Proxy.newProxyInstance(
    loader, new Class[]{ OrderService.class },
    (p, method, args) -> {
        트랜잭션_시작();
        Object result = method.invoke(target, args);   // 원본은 리플렉션으로
        커밋();
        return result;
    });
```

**CGLIB**는 클래스를 상속한 가짜(`OrderService$$SpringCGLIB$$0 extends OrderService`)를 만들고 메서드를 오버라이드해서 가로챈다.

| | JDK 동적 프록시 | CGLIB |
| --- | --- | --- |
| 만드는 방식 | 인터페이스를 구현 | 클래스를 상속 |
| 필요 조건 | 인터페이스가 있어야 한다 | 상속할 수 있어야 한다 (`final` 클래스 ✕) |
| 프록시의 타입 | 인터페이스 타입만 | 원본 클래스 타입 그대로 |
| 못 가로채는 것 | 인터페이스에 없는 메서드 | `final`·`private` 메서드 |
| Spring Boot 기본값 | | ✅ (`spring.aop.proxy-target-class=true`, Boot 2.0부터) |

JDK 프록시라면 구현체 타입으로 주입받는 순간 서버가 안 뜬다. 빈은 `$Proxy123 implements OrderService`인 가짜라서 `OrderServiceImpl`이 아니기 때문이다.

```java
@Autowired
OrderServiceImpl orderService;   // 인터페이스가 아니라 구현 클래스 타입
```

```
The bean 'orderServiceImpl' could not be injected because it is a JDK dynamic proxy
```

CGLIB 프록시는 구현체를 상속한 자식이라 그 타입으로도 받을 수 있다. 그래서 Boot는 CGLIB가 기본이다.

### 3. CGLIB 프록시의 모양과 주입 관계

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;

    @Transactional
    public Long submit(OrderRequest request) {                  // public + @Transactional
        Product product = findProduct(request.productId());
        validateStock(product, request.quantity());
        product.decreaseStock(request.quantity());
        return orderRepository.save(createOrder(product, request)).getId();
    }

    public OrderResponse getOrder(Long orderId) { ... }         // public, @Transactional 없음

    @Transactional
    public final void cancel(Long orderId) {                    // public final + @Transactional
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.cancel();
    }

    private Product findProduct(Long productId) { ... }
    private void validateStock(Product product, int quantity) { ... }
    private Order createOrder(Product product, OrderRequest request) { ... }
}
```

```java
// CGLIB가 서버를 켜는 중에 바이트코드로 만드는 클래스 (개념)
public class OrderService$$SpringCGLIB$$0 extends OrderService {

    // 물려받은 필드 칸: orderRepository = null, productRepository = null (생성자를 안 불러서 비어 있음)

    private OrderService target;                  // 원본
    private TransactionInterceptor txInterceptor;

    @Override
    public Long submit(OrderRequest request) {                      // 트랜잭션 + 원본으로
        return txInterceptor.invoke(() -> target.submit(request));
    }

    @Override
    public OrderResponse getOrder(Long orderId) {                   // 끼울 일이 없으니 그냥 원본으로
        return target.getOrder(orderId);
    }

    // cancel()은 final이라 오버라이드 못 함 → 부모 코드가 "물려받은 그대로" 있다
    // private 메서드는 물려받지만 밖에서 부를 수 없다
}
```

실제로는 오버라이드된 메서드가 모두 Spring의 공용 가로채기 장치(`DynamicAdvisedInterceptor`)로 모이고, 거기서 메서드마다 끼울 일을 찾는다. 위 코드는 그 결과를 메서드별로 풀어 쓴 것이다.

주입은 서로 다른 세 곳에서 일어난다.

```
[orderService 빈의 벨트]
 ① new OrderService(orderRepository, productRepository)   ← repo는 원본에 주입
 ④ 프록시 담당 후처리기: 프록시를 생성자 없이 만들고, 프록시.target = 원본
 ⑤ 보관함 "orderService" = 프록시 (원본 대신)
[orderController 빈의 벨트]
 ① new OrderController(보관함의 orderService)             ← 컨트롤러에는 프록시가 주입
```

| 무엇이 | 어디에 | 어떻게 |
| --- | --- | --- |
| repo들 | 원본의 필드 | 원본 생성자로 |
| 원본 | 프록시의 `target` | 후처리기가 직접 연결 |
| 프록시 | 컨트롤러의 필드 | 컨트롤러 생성자로 |

컨트롤러의 필드 타입은 우리가 쓴 `OrderService`이고, 실제 객체는 프록시다(`Parent p = new Child()`와 같은 다형성). 프록시 클래스 이름은 실행 중에 생겨서 우리 코드에 적을 수 없다.

`submit()` 안에서 부르는 `private` 메서드들은 프록시가 가로채지 못하지만, `submit()`이 프록시를 거쳐 연 트랜잭션 안에서 같은 스레드로 실행된다. 그래서 `validateStock()`에서 예외가 나면 `submit()` 전체가 롤백된다. 반대로 `private` 메서드에 따로 `@Transactional(propagation = REQUIRES_NEW)`를 붙이면 프록시가 가로채지 못해서 조용히 무시된다.

### 4. `final`: 필드는 괜찮고, 클래스·메서드가 문제

| `final`이 붙은 곳 | CGLIB 프록시 |
| --- | --- |
| 클래스 | ✕ 상속이 안 돼서 프록시를 못 만든다 |
| 메서드 | ✕ 오버라이드가 안 돼서 가로채지 못한다 |
| `private` 메서드 | ✕ 오버라이드가 안 돼서 가로채지 못한다 |
| 필드 | ○ 상관없다 |

프록시는 필드를 쓰지 않고 원본에게 넘기기만 한다. 원본은 생성자로 제대로 만들어져 `final` 필드가 다 채워져 있다. 그래서 `@RequiredArgsConstructor` + `final` 필드 + `@Transactional` 조합은 괜찮다.

`final` 메서드는 이렇게 된다.

- `final`은 **오버라이드 금지**이지 **상속 금지**가 아니다. 프록시도 `cancel()`을 물려받아 갖고 있고, 호출은 된다.
- 원본으로 넘기는 `target.xxx()`는 CGLIB가 **오버라이드한 메서드 안에 써 넣은 코드**다. `cancel()`에는 그 코드를 넣을 수 없다.
- 그래서 `OrderService` 클래스의 `cancel()` 코드가 **프록시 객체 위에서** 실행된다. 그 코드는 `target`을 모르고 `this.orderRepository`를 읽는데, `this`는 프록시이고 프록시의 필드 칸은 비어 있다. 결과는 트랜잭션 누락이 아니라 `NullPointerException`이다.
- 원본 객체는 끝까지 호출되지 않는다. 원본은 프록시의 "부모 부분"이 아니라 따로 만들어진 다른 객체다.

Spring은 이 상황을 DEBUG 로그로만 남긴다. 기본 설정(INFO)에서는 보이지 않는다. 인터페이스 메서드를 구현한 `final` 메서드일 때만 WARN이 나온다(Spring Framework 6.2.x `CglibAopProxy` 소스 확인).

```
Final method [...] cannot get proxied via CGLIB: Calls to this method will NOT be routed
to the target instance and might lead to NPEs against uninitialized fields in the proxy instance.
```

### 5. 자기 호출: 프록시를 통과한 호출에만 걸린다

```java
public void submitAll(List<OrderRequest> requests) {   // @Transactional 없음
    for (OrderRequest r : requests) {
        submit(r);                                     // = this.submit(r)
    }
}

@Transactional
public Long submit(OrderRequest request) { ... }
```

```
컨트롤러 → 프록시.submitAll(list)    ← 트랜잭션 없음 → 원본에게 넘김
            → 원본.submitAll(list)
                 this.submit(r1)     ← this = 원본 → 프록시를 안 거친다 → 트랜잭션 ✕
```

프록시는 바깥에서 빈을 받아 간 쪽 앞에만 서 있다. 원본 안에서 `this`는 원본 자신이다. 6번의 "`@PostConstruct` 안에서는 `@Transactional`이 안 걸린다"도 같은 이유다(그때는 프록시가 아직 없다는 이유까지 겹친다). JDK 프록시든 CGLIB든 프록시 방식이면 똑같다.

| 피하는 법 | 설명 |
| --- | --- |
| 다른 빈으로 분리 (가장 흔함) | `submit()`을 다른 빈으로 옮기면 부르는 쪽이 그 빈의 프록시를 거친다 |
| 바깥 메서드에 `@Transactional` | `submitAll()` 전체를 트랜잭션 하나로 묶어도 될 때 |
| `TransactionTemplate` | 코드로 직접 트랜잭션을 연다. 범위를 세밀하게 정할 때 쓰는 정식 방법 |

클래스 자체를 고쳐 넣는 AspectJ 위빙 방식에는 이 문제가 없지만, Spring 기본은 프록시 방식이다.

### 6. Nest와 비교

- Nest의 Interceptor는 주로 컨트롤러 핸들러 단위로 감싼다. 서비스 메서드를 프록시로 바꿔치기하는 기능은 Nest 코어에 없다.
- 서비스 메서드에 트랜잭션을 거는 표준 데코레이터도 Nest 코어에는 없다. TypeORM의 `dataSource.transaction()`처럼 코드로 열거나 별도 라이브러리를 쓴다. 그래서 Spring의 자기 호출 같은 함정은 같은 모양으로 생기지 않는다.

## 헷갈렸던 점

| 처음의 생각 | 정리한 내용 |
| --- | --- |
| JDK 프록시에서 구현체 타입으로 받아도 될 것 같다 | 프록시는 인터페이스를 구현한 가짜라 구현체 타입에 못 들어가고, 서버가 안 뜬다 |
| `final` 필드가 있으면 프록시가 못 본다 | 필드의 `final`은 상관없다. 문제는 클래스와 메서드의 `final`이다 |
| `@RequiredArgsConstructor`로 필드가 전부 `final`인데 `@Transactional`이 될까 | 된다. 프록시는 필드를 안 쓰고 원본에게 넘긴다 |
| (1차 답) CGLIB이 만들어지면 어노테이션이 걷혀서 `submit()`에 트랜잭션이 없다 | 어노테이션은 그대로다. 원본 안의 `this.submit()`이 프록시를 안 거치는 자기 호출이 이유다 |
| `TransactionTemplate`은 절대 추천하지 않는다 | 범위를 세밀하게 정할 때 쓰는 정식 방법이다 |
| 원본은 repo를 잘 받았는데 왜 `null`이지? | 프록시와 원본은 별개 객체다. `final` 메서드는 원본까지 가지 않고 프록시 객체 위에서 돈다 |
| 오버라이드를 못 하면 프록시에 메서드가 없거나, 호출 자체가 안 되거나, 바로 원본으로 가야 한다 | `final`도 상속된다. 자식은 부모의 public 메서드를 없앨 수 없어서 호출은 된다. 원본으로 넘기는 건 CGLIB가 오버라이드 안에 써 넣은 코드라, 그게 없으면 원본으로 못 간다 |
| 컨트롤러는 `OrderService$$SpringCGLIB$$0 o = new ...()`로 들고 있다 | 변수 타입은 우리가 쓴 `OrderService`, 실제 객체가 프록시다. 프록시는 `new`가 아니라 생성자 없이 만들어진다 |
| `final`이면 OrderService 본체를 호출한다 | 실행되는 건 OrderService **클래스의 코드**지만, 실행되는 곳은 원본 **객체**가 아니라 프록시 **객체**다 |
| `target`에 주입된 원본이 있으니 그 repo를 쓰지 않나 | 원본 클래스의 코드는 `target`을 모른다. 언제나 `this.orderRepository`를 읽는다 |

### 확인 문제

**문제 1.** `OrderService`에 `submitAll(list)`(트랜잭션 없음, 안에서 `submit(r)` 반복), `@Transactional submit()`, `@Transactional public final cancel()`(`orderRepository` 사용)이 있다. ① Boot 기본이면 프록시는 어떤 방식으로 어떻게 만들어지고, 왜 그 방식이 기본인가 ② 컨트롤러가 `submitAll(list)`를 부르면 `submit()`에 트랜잭션이 걸리나, 왜, 어떻게 고치나 ③ 컨트롤러가 `cancel(1L)`을 부르면 어떻게 되나

내 답 (첫 시도):

> 1. Spring Boot 의 기본 설정으로라면 기본적으로 CGLIB으로 진행됩니다. JDK 도 좋은 방식이지만 기본 셋팅이 인터페이스를 구현하는 방식이기 때문에 의존을 할때 주의를 해야하고 잘못했을때 서버가 안뜨는 불상사가 생길 수있습니다.
> 2. 그렇지 않습니다. 트랜잭션의 어노테이션 기반으로 만들어지는걸라서 CGLIB이 만들어지면 어노테이션은 모두 겉힌 service 만 남기때문에 submit 은 트랜잭션이 없습니다. 이래서 시작점이 중요합니다. 그래서 수정하려면 트랜잭션을 애초에 submitAll 으로 호출하던지 아니면 클래스를 분리하는 방법이 있습니다. 굳이 하겠다면 코드로 트랜잭션을 여는 방법도 있지만 절대 추천하지 않습니다
> 3. 트랜잭션이 실행되지 않고 작업이 잘못되었을때 작업을 보장할 수 없게 됩니다. 이는 CGLIB의 특성때문인데 CGLIB은 JDK와 달리 상속을 기반으로 하고 있어 메서드에 final 이 있다면 해당 메서드를 프록시화 하지 못하기 때문에 문제가 됩니다

채점: 미통과. ①은 "어떻게"(후처리기가 원본을 상속한 하위 클래스를 만들어 public 메서드를 오버라이드하고 원본에게 넘긴다)가 빠졌다. ②는 결론과 고치는 법은 맞지만 이유가 틀렸다. 어노테이션이 걷히는 게 아니라 자기 호출이다. "시작점이 중요하다"는 감각은 맞다. ③은 결과가 트랜잭션 누락에서 끝나지 않고 프록시 객체의 빈 필드 때문에 NPE가 난다는 점이 빠졌다.

내 답 (재시도):

> 1. 프록시는 후처리 과정에서 제작이 됩니다. 뭐...Spring 이 하겠죠?? CGLIB은 기존의 구현체를 상속하는 방향으로 되어 해당 클래스의 이름 뒤에 CGLIB 이라는 이름이 붙어서 만들어집니다
> 2. submitAll() 에는 트랜잭션이 붙어있지 않아서 바로 구현채로 들어오는데 구현체에서는 트랜잭션이 걸린 프록시를 호출하는게 아니라 본인의 함수를 호출하기 때문에 걸리지 않습니다.
> 3. 어?? 이거 왜 null ㅇ야? 실제 구현체 코드에서는 repo 잘 받았잖아

채점: ①② 통과. ③은 답 대신 질문이 나와서, 위 "헷갈렸던 점"의 질문들을 하나씩 풀고 Parent·Child 예제를 직접 실행해 확인했다.

내 답 (③ 마지막):

> 아 그렇구나... 부모의 final 메서드를 못 물려받은건 맞아 그래서 컨트롤러에서 요청을 받아서 부모의 함수를 호출은 했는데 그 호출한 함수에서 사용하는 repo 가 부모의 repo 가 아니라 작식의 repo를 본다는거야?

채점: 통과. 다듬을 말은 "못 물려받은" → "물려받았지만 고쳐 쓰지 못한"이다. "부모의 repo"는 원본 객체의 repo, "자식의 repo"는 지금 실행 중인 프록시 객체의 빈 칸이다. 그래서 NPE가 난다.

## 예제와 확인 결과

`final` 메서드가 프록시 객체 위에서 도는 상황을 Parent(=OrderService)와 Child(=프록시)로 만들어 직접 실행했다(OpenJDK 21.0.12.1, `java Main.java`, 2026-11-28).

```java
public class Main {
    public static void main(String[] args) {
        Parent original = new Parent(new Repo());   // 원본: repo 있음
        Child proxy = new Child();                  // 프록시: repo = null
        proxy.target = original;
        Parent p = proxy;                           // 컨트롤러가 받는 것
        System.out.println(p.submit());
        System.out.println(p.cancel());
    }
}

class Repo { String find() { return "주문 1번"; } }

class Parent {
    Repo repo;
    Parent(Repo repo) { this.repo = repo; }
    Parent() { }
    public String submit()       { return "submit: " + repo.find(); }
    public final String cancel() { return "cancel: " + repo.find(); }
}

class Child extends Parent {
    Parent target;
    @Override
    public String submit() {
        System.out.println("[트랜잭션 시작]");
        String r = target.submit();
        System.out.println("[커밋]");
        return r;
    }
}
```

```
[트랜잭션 시작]
[커밋]
submit: 주문 1번
Exception in thread "main" java.lang.NullPointerException: Cannot invoke "Repo.find()" because "this.repo" is null
	at Parent.cancel(Main.java:25)
	at Main.main(Main.java:10)
```

`at Parent.cancel`은 실행된 코드가 부모 클래스의 것이라는 뜻이고, `"this.repo" is null`은 그 코드의 `this`가 원본이 아니라 repo가 빈 프록시 객체라는 뜻이다.

- 그 밖의 코드는 설명용이다.
- `final` 메서드 로그의 레벨(DEBUG, 인터페이스 구현 메서드면 WARN)은 Spring Framework 6.2.x `CglibAopProxy`, JDK 프록시 주입 실패 메시지는 Spring Boot 3.3.x `BeanNotOfRequiredTypeFailureAnalyzer` 소스에서 확인했다.

## 남은 질문

- 이번 범위에서 물어본 것은 모두 정리했다. 이어서 본다면 트랜잭션 전파(`REQUIRES_NEW` 등)가 프록시 경계와 어떻게 맞물리는지다.

## 참고 자료

이번 공부는 Claude와 대화로 진행했다(2026-11-28).

- 김영한, 인프런 "스프링 입문 - 코드로 배우는 스프링 부트, 웹 MVC, DB 접근 기술" - [AOP 적용](https://www.inflearn.com/courses/lecture?courseId=325630&unitId=49601)
- [Spring Framework - Proxying Mechanisms](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)
- [Spring Framework - Using @Transactional](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
- [CglibAopProxy 소스 (6.2.x)](https://github.com/spring-projects/spring-framework/blob/6.2.x/spring-aop/src/main/java/org/springframework/aop/framework/CglibAopProxy.java)

## 정리 이력

- 2026-11-28: 8번 AOP 프록시를 공부하고 확인 문제를 통과해 정리했다. Part 2 완료.
