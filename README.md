# Spring Framework 정리 자료
스프링 핵심 원리 - 고급편 정리 문서

Table of contents
=================
<!--ts-->
   * [쓰레드 로컬](#쓰레드-로컬)
   * [템플릿 메서드 패턴과 콜백 패턴](#템플릿-메서드-패턴과-콜백-패턴)
   * [프록시 패턴과 데코레이터 패턴](#프록시-패턴과-데코레이터-패턴)
   * [동적 프록시 기술](#동적-프록시-기술)
   * [스프링이 지원하는 프록시](#스프링이-지원하는-프록시)
   * [빈 후처리기](#빈-후처리기)
   * [@Aspect AOP](#@Aspect-AOP)
   * [스프링 AOP 구현](#스프링-AOP-구현)
   * [스프링 AOP 포인트컷](#스프링-AOP-포인트컷)
   * [스프링 AOP 실전예제](#스프링-AOP-실전예제)
   * [스프링 AOP 실무 주의사항](#스프링-AOP-실무-주의사항)
<!--te-->

쓰레드 로컬
=======
- `쓰레드 로컬`
   - `해당 쓰레드`만 접근할 수 있는 `특별한 저장소`
   - WAS는 쓰레드 풀에서 쓰레드 단위로 요청에 할당하여 처리하기 때문에 동시성 이슈가 있는 저장 항목은 쓰레드 로컬을 통해서 각각 처리해야함
   - 특히 스프링 빈 처럼 싱글톤 객체의 필드나 static 같은 공용
필드를 변경하며 사용할 때 이러한 동시성 문제가 자주 발생
   - 동시성 문제는 값을 읽기만 하면 발생하지 않고 어디선가 값을 변경하기 때문에 발생
   - 바는 언어차원에서 쓰레드 로컬을 지원하기 위한 `java.lang.ThreadLocal` 클래스를 제공
- 동시성 문제가 있는 코드
```java
@Slf4j
public class FieldLogTrace implements LogTrace {

   private static final String START_PREFIX = "-->"; 
   private static final String COMPLETE_PREFIX = "<--"; 
   private static final String EX_PREFIX = "<X-";
   
   private TraceId traceIdHolder; //traceId 동기화, 동시성 이슈 발생
   
   @Override
   public TraceStatus begin(String message) {
      syncTraceId();
      TraceId traceId = traceIdHolder;
      Long startTimeMs = System.currentTimeMillis();
      log.info("[{}] {}{}", traceId.getId(), addSpace(START_PREFIX, traceId.getLevel()), message);
      
      return new TraceStatus(traceId, startTimeMs, message);
   }

   @Override
   public void end(TraceStatus status) {
        complete(status, null);
   }
   
   @Override
   public void exception(TraceStatus status, Exception e) {
        complete(status, e);
   }
   
   private void complete(TraceStatus status, Exception e) { 
      Long stopTimeMs = System.currentTimeMillis();
      long resultTimeMs = stopTimeMs - status.getStartTimeMs(); 
      TraceId traceId = status.getTraceId();
      
      if (e == null) {
         log.info("[{}] {}{} time={}ms", traceId.getId(), addSpace(COMPLETE_PREFIX, traceId.getLevel()), status.getMessage(), resultTimeMs);   
      } else {
         log.info("[{}] {}{} time={}ms ex={}", traceId.getId(), addSpace(EX_PREFIX, traceId.getLevel()), status.getMessage(), resultTimeMs, e.toString());
      }
      releaseTraceId();
    }

    private void syncTraceId() {
      if (traceIdHolder == null) {
         traceIdHolder = new TraceId();
      } else {
         traceIdHolder = traceIdHolder.createNextId(); 
      }
    }

    private void releaseTraceId() {
       if (traceIdHolder.isFirstLevel()) { 
          traceIdHolder = null; //destroy
      } else {
         traceIdHolder = traceIdHolder.createPreviousId();
      }
    }

    private static String addSpace(String prefix, int level) {
      StringBuilder sb = new StringBuilder();
      
      for (int i = 0; i < level; i++) {
         sb.append( (i == level - 1) ? "|" + prefix : "| "); 
      }
      
      return sb.toString(); 
   }
}
```
  > TraceId traceIdHolder 필드를 공통으로 사용하므로 여러 스레드에서 값을 갱신하는 경우 동시성 문제가 생김

- 쓰레드 로컬이 적용된 코드
```java
@Slf4j
public class ThreadLocalLogTrace implements LogTrace {

   private static final String START_PREFIX = "-->"; 
   private static final String COMPLETE_PREFIX = "<--"; 
   private static final String EX_PREFIX = "<X-";

   private ThreadLocal<TraceId> traceIdHolder = new ThreadLocal<>();
   
   @Override
   public TraceStatus begin(String message) {
      syncTraceId();
      TraceId traceId = traceIdHolder.get();
      Long startTimeMs = System.currentTimeMillis();
      log.info("[{}] {}{}", traceId.getId(), addSpace(START_PREFIX, traceId.getLevel()), message);
      return new TraceStatus(traceId, startTimeMs, message);
   }
   
   @Override
   public void end(TraceStatus status) {
      complete(status, null);
   }
   
   @Override
   public void exception(TraceStatus status, Exception e) {
      complete(status, e);
   }

   private void complete(TraceStatus status, Exception e) { 
      Long stopTimeMs = System.currentTimeMillis();
      long resultTimeMs = stopTimeMs - status.getStartTimeMs(); 
      TraceId traceId = status.getTraceId();

      if (e == null) {
         log.info("[{}] {}{} time={}ms", traceId.getId(), addSpace(COMPLETE_PREFIX, traceId.getLevel()), status.getMessage(), resultTimeMs);
      } else {
         log.info("[{}] {}{} time={}ms ex={}", traceId.getId(), addSpace(EX_PREFIX, traceId.getLevel()), status.getMessage(), resultTimeMs, e.toString());
      }
      releaseTraceId();
   }

   private void syncTraceId() {
      TraceId traceId = traceIdHolder.get(); 
      if (traceId == null) {
         traceIdHolder.set(new TraceId()); 
      } else {
         traceIdHolder.set(traceId.createNextId()); 
      }
   }
   
   private void releaseTraceId() {
      TraceId traceId = traceIdHolder.get(); 
      if (traceId.isFirstLevel()) {
         traceIdHolder.remove(); //destroy 
      } else {
         traceIdHolder.set(traceId.createPreviousId()); 
      }
   }
   
   private static String addSpace(String prefix, int level) {
      StringBuilder sb = new StringBuilder();
   
      for (int i = 0; i < level; i++) {
         sb.append( (i == level - 1) ? "|" + prefix : "| "); 
      }
      
      return sb.toString(); 
   }
}
```
> 쓰레드 로컬이 적용되니 ThreadLocal<TraceId> traceIdHolder를 공통으로 사용하므로 여러 스레드에서 값을 갱신하는 경우 동시성 문제가 생기지 않음
- 쓰레드 로컬 사용법
  - 값 저장: ThreadLocal.set(value) 
  - 값 조회: ThreadLocal.get()
  - 값 제거: ThreadLocal.remove()
- 쓰레 로컬 주의 사항
  - 쓰레드를 생성하는 비용은 비싸기 때문에 쓰레드를 제거하지 않고 보통 `쓰레드 풀`을 통해서 쓰레드를 재사용하는데 쓰레드 로컬에 저장된 값을 제거하지 않으면 다른 요청에 의해 쓰레드가 재사용될 수 있고 그때 쓰레드 로컬에 저장된 값이 노출될 수 있음
  - 쓰레드 로컬을 모두 사용하고 나면 꼭 `ThreadLocal.remove()` 를 호출해서 쓰레드 로컬에 저장된 값을 제거해주어야함
  
템플릿 메서드 패턴과 콜백 패턴
=======
- `템플릿 메서드 패턴`
  - 상위 클래스의 템플릿 메서드에서 하위 클래스가 오버라이딩한 메서드를 호출하는 패턴
  - 템플릿 메서드 패턴은 상속을 통해 `동일한 부분(핵심 기능)`은 상위 클래스로, `달라지는 부분(부가 기능)`만 하위 클래스로 분할하여 사용하는 패턴
  - 템플릿 메서드 패턴은 상속을 사용해서 따라서 상속에서 오는 단점들을 그대로 가지고 있음(부모-자식 의존)
- 템플릿 메서드 패턴 예제 코드
```java
@Slf4j
public abstract class AbstractTemplate {
   
   // 변하지 않는 부분
   public void execute() {
      long startTime = System.currentTimeMillis();
      call(); //상속
      long endTime = System.currentTimeMillis(); 
      long resultTime = endTime - startTime; 
      log.info("resultTime={}", resultTime);
   }

   // 변하는 부분
   protected abstract void call();
}
```
```java
@Slf4j
public class SubClassLogic1 extends AbstractTemplate {
   @Override
   protected void call() {
      log.info("비즈니스 로직1 실행"); 
   }
}
```
```java
@Slf4j
public class SubClassLogic2 extends AbstractTemplate {
   @Override
   protected void call() { 
      log.info("비즈니스 로직2 실행");
   } 
}
```
```java
public class TemplateMethodTest {
   @Test
   void templateMethodV1() {
      AbstractTemplate template1 = new SubClassLogic1(); 
      template1.execute();
      AbstractTemplate template2 = new SubClassLogic2();
      template2.execute(); 
   }
}
```
- `전략 패턴`
  - 클라이언트가 전략을 생성해 전략을 실행한 컨텍스트에 주입하는 패턴
  - 템플릿 메서드 패턴과 비슷한 역할을 하면서 상속의 단점을 제거할 수 있는 디자인 패턴
  -  변하지 않는 부분을 `Context` 라는 곳에 두고, 변하는 부분을 `Strategy` 라는 인터페이스를 만들고 해당 인터페이스를 구현하도록 해서 문제를 해결
  - Context 는 내부에 Strategy strategy 필드를 가지고 있어서 이 필드에 변하는 부분인 Strategy 의 구현체를 주입
  - 전략 패턴의 핵심은 Context 는 Strategy 인터페이스에만 의존
- 전략 패턴 패턴 예제 코드
```java
public interface Strategy {
   void call();
}
```
```java
@Slf4j
public class StrategyLogic1 implements Strategy {
   
   @Override
   public void call() {
      log.info("비즈니스 로직1 실행"); 
   }
}
```
```java
@Slf4j
public class StrategyLogic2 implements Strategy {

   @Override
   public void call() {
      log.info("비즈니스 로직2 실행"); 
   }
}
```
```java
@Slf4j
public class ContextV1 {
   
   private Strategy strategy;
   
   public ContextV1(Strategy strategy) { 
      this.strategy = strategy;
   }
   
   public void execute() {
      long startTime = System.currentTimeMillis(); 
      
      //비즈니스 로직 실행
      strategy.call(); //위임
      //비즈니스 로직 종료
      
      long endTime = System.currentTimeMillis(); 
      long resultTime = endTime - startTime; 
      log.info("resultTime={}", resultTime);
   } 
}
```
```java
public class ContextV1Test {

   @Test
   void strategyV1() {
      Strategy strategyLogic1 = new StrategyLogic1(); 
      ContextV1 context1 = new ContextV1(strategyLogic1); 
      
      context1.execute();
      
      Strategy strategyLogic2 = new StrategyLogic2(); 
      ContextV1 context2 = new ContextV1(strategyLogic2); context2.execute();
   }
}
```
- `템플릿 콜백 패턴`
  - 전략 패턴 중에서 Strategy를 내부 익명 클래스로 정의해서 만든 템플릿 콜백 패턴
  - 정식 GOF 패턴은 아니고, 스프링 내부에서 이런 방식을 자주 사용
  - 스프링에서 이름에 XxxTemplate 가 있다면 템플릿 콜백 패턴
- 템플릿 콜백 패턴 예제 코드
```java
public interface Callback {
   void call();
}
```
```java
@Slf4j
public class TimeLogTemplate {
   
   public void execute(Callback callback) {
      
      long startTime = System.currentTimeMillis(); 
      
      //비즈니스 로직 실행
      callback.call(); //위임
      //비즈니스 로직 종료
      
      long endTime = System.currentTimeMillis(); 
      long resultTime = endTime - startTime; 
      log.info("resultTime={}", resultTime);
} 
```
```java
@Slf4j
public class TemplateCallbackTest {

   @Test
   void callbackV1() {
      TimeLogTemplate template = new TimeLogTemplate();
      template.execute(new Callback() { 
         @Override
         public void call() {
             log.info("비즈니스 로직1 실행"); 
         }
      });
      
      template.execute(new Callback() { 
         @Override
         public void call() { 
            log.info("비즈니스 로직2 실행");
         } 
      });
   }

   @Test
   void callbackV2() {
      
      TimeLogTemplate template = new TimeLogTemplate(); 
      template.execute(() -> log.info("비즈니스 로직1 실행")); 
      template.execute(() -> log.info("비즈니스 로직2 실행"));
   } 
}
```

프록시 패턴과 데코레이터 패턴
=======
- `프록시 패턴`
  - 프록시 패턴은 접근 제어가 목적이고 데코레이터 패턴 새로운 기능 추가가 목적으로 둘다 프록시를 사용하긴함
  - 프록시 패턴은 실제 서비스 메서드의 반환 값을 수정하는 게 아니라 `별도 로직을 수행`하기 위해 사용
- 프록시 패턴 예제 코드
```java
public interface Subject {
   String operation();
}
```
```java
@Slf4j
public class RealSubject implements Subject {
   
   @Override
   public String operation() {
      log.info("실제 객체 호출"); 
      sleep(1000);
      return "data";
   }

   private void sleep(int millis) {
      try {
         Thread.sleep(millis);
      } catch (InterruptedException e) {
          e.printStackTrace(); 
      }
   } 
}
```
```java
@Slf4j
public class CacheProxy implements Subject {
   private Subject target;
   private String cacheValue;

   public CacheProxy(Subject target) { 
      this.target = target;
   }
   
   @Override
   public String operation() {
      log.info("프록시 호출");
      
      if (cacheValue == null) {
         cacheValue = target.operation(); 
      }
      return cacheValue;
   }
}

```
```java
public class ProxyPatternClient {

   private Subject subject;
   
   public ProxyPatternClient(Subject subject) { 
      this.subject = subject;
   }
   
   public void execute() { 
      subject.operation();
   } 
}
```
```java
 public class ProxyPatternTest {

   @Test
   void noProxyTest() {
      RealSubject realSubject = new RealSubject();
      ProxyPatternClient client = new ProxyPatternClient(realSubject); 
      client.execute();
      client.execute();
      client.execute();
   } 

   @Test
   void cacheProxyTest() {
      
      Subject realSubject = new RealSubject();
      Subject cacheProxy = new CacheProxy(realSubject); // 프록시
      
      ProxyPatternClient client = new ProxyPatternClient(cacheProxy); 
      client.execute();
      client.execute();
      client.execute();
   }
}
```
```java

```
- `데코레이터 패턴`
  - 프록시 패턴은 클라이언트가 최종적으로 돌려 받는 반환 값을 조작하지 않고 전달하는 반면, 데코레이터 패턴은 클라이언트가 받는 반환 값에 장식을 덧붙임
- 데코레이터 패턴 예제 코드
```java
@Slf4j
public class RealComponent implements Component {
   
   @Override
   public String operation() {
      log.info("RealComponent 실행");
      return "data";
   }
}
```
```java
@Slf4j
public class MessageDecorator implements Component {
   private Component component;
   
   public MessageDecorator(Component component) { 
      this.component = component;
   }
   
   @Override
   public String operation() {
      log.info("MessageDecorator 실행");
      String result = component.operation();
      String decoResult = "*****" + result + "*****"; 
      log.info("MessageDecorator 꾸미기 적용 전={}, 적용 후={}", result, decoResult);
      return decoResult;
   }
}
```
```java
@Slf4j
public class DecoratorPatternClient {
   private Component component;
   
   public DecoratorPatternClient(Component component) { 
      this.component = component;
   }
   
   public void execute() {
      String result = component.operation(); 
      log.info("result={}", result);
   } 
}
```
```java
@Slf4j
public class DecoratorPatternTest {
   
   @Test
   void noDecorator() {
      Component realComponent = new RealComponent();
      DecoratorPatternClient client = new DecoratorPatternClient(realComponent);
      client.execute(); 
   }
   
   @Test
   void decorator1() {
      Component realComponent = new RealComponent();
      Component messageDecorator = new MessageDecorator(realComponent);
      DecoratorPatternClient client = new DecoratorPatternClient(messageDecorator); client.execute();
   }
}
```
동적 프록시 기술
=======
- `리플랙션`
  - 클래스나 메서드의 메타정보를 동적으로 획득하고, 코드도 동적으로 호출하는 기술
  - 리플렉션 기술은 런타임에 동작하기 때문에, 컴파일 시점에 오류를 잡을 수 없으므로 일반적으로 사용하면 안됨
- 리플랙션 예제 코드
```java
@Slf4j
public class ReflectionTest {

   @Slf4j
   static class Hello {
      public String callA() { 
         log.info("callA"); 
         return "A";
      }
      
      public String callB() {
         log.info("callB");
         return "B"; 
      }
   }

   @Test
   void reflection_test() throws Exception {
      //클래스 정보
      Class classHello = Class.forName("hello.proxy.jdkdynamic.ReflectionTest$Hello");
      Hello target = new Hello(); 
      
      //callA 메서드 정보 
      Method methodCallA = classHello.getMethod("callA"); 
      Object result1 = methodCallA.invoke(target); 
      log.info("result1={}", result1);
      
      //callB 메서드 정보
      Method methodCallB = classHello.getMethod("callB"); 
      Object result2 = methodCallB.invoke(target); 
      log.info("result2={}", result2);
   }
}
```
- `JDK 동적 프록시`
  - 프록시를 적용하기 위해 적용 대상의 숫자 만큼 많은 프록시 클래스가 필요한 문제가 있었는데, 동적 프록시 기술을 사용하면 개발자가 직접 프록시 클래스를 만들지 않게 만들어줌
  - 리플랙션을 사용하는 JDK 동적 프록시는 인터페이스를 기반으로 프록시를 동적으로 만들어줌
- JDK 동적 프록시 사용법
  - JDK 동적 프록시에 적용할 로직은 InvocationHandler 인터페이스를 구현해서 작성
  ```java
  public interface InvocationHandler {
       public Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
  }
  ```
  - 제공되는 파라미터 목록
    - Object proxy : 프록시 자신
    - Method method : 호출한 메서드
    - Object[] args : 메서드를 호출할 때 전달한 인수
- JDK 동적 프록시 예제 코드
```java
public interface AInterface {
   String call();
}
```
```java
@Slf4j
public class AImpl implements AInterface {
   @Override
   public String call() {
      log.info("A 호출");
      return "a"; 
   }
}
```
```java
public interface BInterface {
   String call();
}
```
```java
@Slf4j
public class BImpl implements BInterface {
   @Override
   public String call() {
      log.info("B 호출");
      return "b"; 
   }
}
```
```java
@Slf4j
public class TimeInvocationHandler implements InvocationHandler {

   private final Object target;
   
   public TimeInvocationHandler(Object target) { 
      this.target = target;
   }

   @Override
   public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      log.info("TimeProxy 실행");
      long startTime = System.currentTimeMillis();
      Object result = method.invoke(target, args);
      long endTime = System.currentTimeMillis();
      long resultTime = endTime - startTime; 
      log.info("TimeProxy 종료 resultTime={}", resultTime); return result;
   }
}
```
```java
@Slf4j
public class JdkDynamicProxyTest {

    @Test
    void dynamicA() {
      AInterface target = new AImpl();
      TimeInvocationHandler handler = new TimeInvocationHandler(target);
      AInterface proxy = (AInterface) Proxy.newProxyInstance(AInterface.class.getClassLoader(), new Class[] {AInterface.class}, handler); 
      
      proxy.call();
      
      log.info("targetClass={}", target.getClass());
      log.info("proxyClass={}", proxy.getClass()); 
   }
      
   @Test
   void dynamicB() {
      BInterface target = new BImpl();
      TimeInvocationHandler handler = new TimeInvocationHandler(target);
      BInterface proxy = (BInterface) Proxy.newProxyInstance(BInterface.class.getClassLoader(), new Class[] {BInterface.class}, handler);

      proxy.call();
         
      log.info("targetClass={}", target.getClass());
      log.info("proxyClass={}", proxy.getClass()); 
   }
}
```
- JDK 동적 프록시 - 한계
  - JDK 동적 프록시는 인터페이스가 필수적이라 클래스만 있는 경우에는 어떻게 동적 프록시를 적용할 수 없음
- `CGLIB`
  - CGLIB는 바이트코드를 조작해서 동적으로 클래스를 생성하는 기술을 제공하는 라이브러리
  - CGLIB를 사용하면 인터페이스가 없어도 구체 클래스만 가지고 동적 프록시를 만듦
  - 스프링 프레임워크가 스프링 내부 소스 코드에 포함되어 있어 사용 가능
- CGLIB 사용법
  - JDK 동적 프록시에서 실행 로직을 위해 InvocationHandler 를 제공했듯이, CGLIB는 MethodInterceptor 를 제공
  ```java
  public interface MethodInterceptor extends Callback {
   Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable;
  }
  ```
  - 제공되는 파라미터 목록
    - obj : CGLIB가 적용된 객체
    - method : 호출된 메서드
    - args : 메서드를 호출하면서 전달된 인수
    - proxy : 메서드 호출에 사용
- CGLIB 예제 코드
```java
@Slf4j
public class TimeMethodInterceptor implements MethodInterceptor {
   private final Object target;
   
   public TimeMethodInterceptor(Object target) {
      this.target = target; 
   }

   @Override
   public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
      
      log.info("TimeProxy 실행");
      long startTime = System.currentTimeMillis();
      Object result = proxy.invoke(target, args);
      long endTime = System.currentTimeMillis();
      long resultTime = endTime - startTime; 
      log.info("TimeProxy 종료 resultTime={}", resultTime); 
      return result;
   }
}
```java
@Slf4j
public class CglibTest {
   
   @Test
   void cglib() {
      ConcreteService target = new ConcreteService();
      Enhancer enhancer = new Enhancer(); 
      enhancer.setSuperclass(ConcreteService.class); 
      enhancer.setCallback(new TimeMethodInterceptor(target)); 
      ConcreteService proxy = (ConcreteService)enhancer.create(); 
      log.info("targetClass={}", target.getClass()); 
      log.info("proxyClass={}", proxy.getClass());
      proxy.call(); 
   }
}
```
- `CGLIB 제약`
  - 클래스 기반 프록시는 상속을 사용하기 때문에 몇가지 제약 존재
    - 부모 클래스의 생성자를 체크해야함 -> CGLIB는 자식 클래스를 동적으로 생성하기 때문에 기본 생성자가 필요
    - 클래스에 final 키워드가 붙으면 상속이 불가능 -> CGLIB에서는 예외가 발생
    - 메서드에 final 키워드가 붙으면 해당 메서드를 오버라이딩 할 수 없음 -> CGLIB에서는 프록시 로직이 동작하지 않음
- JDK 동적 프록시와 CGLIB
  - 인터페이스가 있는 경우에는 JDK 동적 프록시를 적용하고 그렇지 않은 경우에는 CGLIB를 적용하지만 InvocationHandler와 CGLIB가 제공하는 MethodInterceptor를 각각 중복으로 만들어서 관리하는 것은 불편하기에 주로 이 두 가지가 통합된 `스프링이 제공하는 프록시 기능을 사용함`
