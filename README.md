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
   * [스프링 AOP 개념](#스프링-AOP-개념)
   * [스프링 AOP 구현](#스프링-AOP-구현)
   * [스프링 AOP 포인트컷](#스프링-AOP-포인트컷)
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
  - Context 는 내부에 `Strategy strategy` 필드를 가지고 있어서 이 필드에 변하는 부분인 Strategy 의 구현체를 주입
  - 전략 패턴의 핵심은 Context 는 Strategy 인터페이스에만 의존
- 전략 패턴 예제 코드
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
      ContextV1 context2 = new ContextV1(strategyLogic2); 
      context2.execute();
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

스프링이 지원하는 프록시
=======
- `프록시 팩토리`
  - 프록시 팩토리에 타겟을 설정하고 그 타겟이 인터페이스가 있으면 JDK 동적 프록시를 사용하고, 구체 클래스만 있다면 CGLIB를 사용
- 프록시 팩토리 예제 코드
```java
@Slf4j
public class TimeAdvice implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
      log.info("TimeProxy 실행");
      long startTime = System.currentTimeMillis();
      Object result = invocation.proceed();
      long endTime = System.currentTimeMillis();
      long resultTime = endTime - startTime; 
      log.info("TimeProxy 종료 resultTime={}ms", resultTime); 
      return result;
   }
}
```
```java
@Slf4j
public class ProxyFactoryTest {
   
   @Test
   @DisplayName("인터페이스가 있으면 JDK 동적 프록시 사용")
   void interfaceProxy() {

       ServiceInterface target = new ServiceImpl();
       ProxyFactory proxyFactory = new ProxyFactory(target); // 생성자에 프록시의 호출 대상을 함께 넘겨주어 인터페이스인지 구체 클래스인지에 따라 동적 프록시 생성 
       proxyFactory.addAdvice(new TimeAdvice()); // 프록시 팩토리를 통해서 만든 프록시가 사용할 부가 기능 로직을 설정

       ServiceInterface proxy = (ServiceInterface) proxyFactory.getProxy(); // 프록시 객체를 생성하고 그 결과를 받음
       log.info("targetClass={}", target.getClass()); 
       log.info("proxyClass={}", proxy.getClass());
       
       proxy.save(); // 프록시 객체를 통해 부가 기능 로직을 추가하여 target의 함수 실행
       
       assertThat(AopUtils.isAopProxy(proxy)).isTrue(); 
       assertThat(AopUtils.isJdkDynamicProxy(proxy)).isTrue(); 
       assertThat(AopUtils.isCglibProxy(proxy)).isFalse();
   } 
}
```
- `포인트컷`
  - `어디에` 부가 기능을 적용할지, 어디에 부가 기능을 적용하지 않을지 판단하는 필터링 로직
  - 특정 조건에 맞을 때 프록시 로직을 적용하는 기능도 공통으로 제공하기 위해 Pointcut 이라는 개념을 도입
- 스프링이 제공하는 포인트 컷
  - NameMatchMethodPointcut: 메서드 이름을 기반으로 매칭하며 내부에서는 PatternMatchUtils 를 사용
  - JdkRegexpMethodPointcut: JDK 정규 표현식을 기반으로 포인트컷을 매칭
  - TruePointcut: 항상 참을 반환
  - AnnotationMatchingPointcut: 애노테이션으로 매칭
  - `AspectJExpressionPointcut`: aspectJ 표현식으로 매칭, 대부분 이거만 사용
- `어드바이스`
  - 프록시가 호출(제공)하는 `부가 기능 로직`
  - Advice 는 프록시에 적용하는 부가 기능 로직으로 JDK 동적 프록시가 제공하는 InvocationHandler와 CGLIB가 제공하는 MethodInterceptor를 각각 중복으로 따로 만드는 것을 방지하기 위해 Advice를 만듦
  - `MethodInterceptor` 인터페이스를 구현하면 만들 수 있음
- `어드바이저`
  - 단순하게 하나의 포인트컷과 하나의 어드바이스를 가지고 있는 것
  - 스프링은 우리가 사용할 대부분의 포인트 컷을 제공하지만 직접 구현한 포인트 컷을 사용하고 싶을 경우 `new DefaultPointcutAdvisor(포인트컷, 어드바이스)`를 사용
- 포인트 컷, 어드바이스, 어드바이저 예제
```java
@Slf4j
public class ProxyFactoryTest {
   
   @Test
   @DisplayName("스프링이 제공하는 포인트컷")
   void advisorTest3() {
      ServiceImpl target = new ServiceImpl();
      ProxyFactory proxyFactory = new ProxyFactory(target); 

      NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut(); 
      pointcut.setMappedNames("save"); // 매칭할 메서드 이름을 지정
      DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(pointcut, new TimeAdvice());
      proxyFactory.addAdvisor(advisor);
      
      ServiceInterface proxy = (ServiceInterface) proxyFactory.getProxy();
      proxy.save();
      proxy.find(); 
   }

}
```

빈 후처리기
=======
- `빈 후처리기`
  - 빈 포스트 프로세서(BeanPostProcessor)는 번역하면 빈 후처리기인데, 이름 그대로 빈을 생성한 후에 무언가를 처리하는 용도로 사용
  - @Bean, 컴포넌트 스캔된 객체가 생성된 이후 조작 및 객체 스왑을 가능하게 함
  - `BeanPostProcessor`를 구현하면 빈 후처리기를 만들 수 있음
  - @PostConstruct도 같은 방식으로 동작
- 빈 후처리기가 해결한 문제
  - 프록시를 직접 스프링 빈으로 등록하는 설정 부분을 하나로 집중
  - 컴포넌트 스캔처럼 스프링이 직접 대상을 빈으로 등록하는 경우에도 중간에 빈 등록 과정을 가로채서 원본 대신에 프록시를 스프링 빈으로 등록
- 빈 후처리기 예제 코드
```java
public class BeanPostProcessorTest {

   @Test
   void postProcessor() {
      ApplicationContext applicationContext = new AnnotationConfigApplicationContext(BeanPostProcessorConfig.class);
      
      // beanA 이름으로 B 객체가 빈으로 등록된다.
      B b = applicationContext.getBean("beanA", B.class); 
      b.helloB();

      // A는 빈으로 등록되지 않는다.
      A a = applicationContext.getBean("beanA", A.class); 
      a.helloA();
      
      // B는 빈으로 등록되지 않는다. 
      Assertions.assertThrows(NoSuchBeanDefinitionException.class,
() -> applicationContext.getBean(B.class));
      
   @Slf4j
   @Configuration
   static class BasicConfig {

      @Bean(name = "beanA")
      public A a() {
         return new A();
      }

      @Bean
      public AToBPostProcessor helloPostProcessor() {
         return new AToBPostProcessor();
      }
   }
   
   @Slf4j
   static class A {
      public void helloA() { 
         log.info("hello A");
      } 
   }
   
   @Slf4j
   static class B {
      public void helloB() { 
         log.info("hello B");
      } 
   }

   @Slf4j
   static class AToBPostProcessor implements BeanPostProcessor {
      
      @Override
      public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
         log.info("beanName={} bean={}", beanName, bean);
         if (bean instanceof A) {
            return new B();
         }
         return bean;
      }
   }

}
```
- `스프링이 제공하는 빈 후처리기`
  - 스프링 부트 자동 설정으로 `AnnotationAwareAspectJAutoProxyCreator` 라는 빈 후처리기가 스프링 빈에 자동으로 등록
  - 이 빈 후처리기는 스프링 빈으로 등록된 Advisor 들을 자동으로 찾아서 프록시가 필요한 곳에 자동으로 프록시를 적용
- 빈 후처리기 예제 코드
```java
@Configuration
@Import({AppV1Config.class, AppV2Config.class}) 
public class AutoProxyConfig {

   @Bean
   public Advisor advisor1(LogTrace logTrace) {
      
      AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut(); 
      pointcut.setExpression("execution(* hello.proxy.app..*(..))"); // AspectJ가 제공하는 포인트컷 표현식
      LogTraceAdvice advice = new LogTraceAdvice(logTrace);
      
      // advisor = pointcut + advice
      return new DefaultPointcutAdvisor(pointcut, advice);
   }
}
```
```java
@Import(AutoProxyConfig.class) 
@SpringBootApplication(scanBasePackages = "hello.proxy.app") 
public class ProxyApplication {
   
   public static void main(String[] args) { 
      SpringApplication.run(ProxyApplication.class, args);
   }
      
   @Bean
   public LogTrace logTrace() {
      return new ThreadLocalLogTrace();
   }
}
```
- 포인트 컷과 `자동 프록시 생성기(빈 후처리기)`
  - 프록시 적용 여부 판단: 자동 프록시 생성기는 포인트컷을 사용해서 해당 빈이 프록시를 생성할 필요가 있는지 없는지 체크
  - 어드바이스 적용 여부 판단: 프록시가 호출되었을 때 부가 기능인 어드바이스를 적용할지 말지 포인트컷을 보고 판단
- 하나의 프록시에 여러 Advisor 적용하기
  - 프록시 팩토리가 생성하는 프록시는 내부에 여러 advisor 들을 포함 가능
  - 프록시 자동 생성기는 프록시를 하나만 만들지만 포인트컷의 조건을 모두 만족하면 모두 실행

@Aspect AOP
=======
- `@Aspect AOP`
  - @Aspect 는 관점 지향 프로그래밍(AOP)을 가능하게 하는 AspectJ 프로젝트에서 제공하는 애노테이션
  - @Aspect 애노테이션으로 매우 편리하게 `포인트컷과 어드바이스로 구성되어 있는 어드바이저 생성 기능을 지원`
  - 스프링은 이것을 차용해서 프록시를 통한 AOP를 가능하게함
- @Aspect 예제 코드
```java
@Slf4j
@Aspect // 애노테이션 기반 프록시를 적용
public class LogTraceAspect {

   private final LogTrace logTrace;

   public LogTraceAspect(LogTrace logTrace) { 
      this.logTrace = logTrace;
   }

   @Around("execution(* hello.proxy.app..*(..))") // @Around는 포인트 컷
   public Object execute(ProceedingJoinPoint joinPoint) throws Throwable { // excute는 어드바이스

      TraceStatus status = null;

      try {

         String message = joinPoint.getSignature().toShortString(); // joinPoint 안에 내부에 실제 호출 대상, 전달 인자, 그리고 어떤 객체와 어떤 메서드가 호출되었는지 정보가 포함되어 있음
         status = logTrace.begin(message);
         
         //실제 로직 호출
         Object result = joinPoint.proceed();
         
         logTrace.end(status);
         return result;
         
      } catch (Exception e) {
         logTrace.exception(status, e);
         throw e; 
      }
   }
}
```
```java
@Configuration
@Import({AppV1Config.class, AppV2Config.class}) // 수동 빈 등록
public class AopConfig {

   @Bean // @Aspect 가 있어도 스프링 빈으로 등록해야함
   public LogTraceAspect logTraceAspect(LogTrace logTrace) {
         return new LogTraceAspect(logTrace);
   }

}
```
```java
@Import(AopConfig.class) 
@SpringBootApplication(scanBasePackages = "hello.proxy.app") 
public class ProxyApplication {
   public static void main(String[] args) { 
      SpringApplication.run(ProxyApplication.class, args);
   }
   
   @Bean
   public LogTrace logTrace() {
         return new ThreadLocalLogTrace();
   }
}
```
- `@Aspect를 어드바이저로 변환해서 저장하는 과정`
  - 실행: 스프링 애플리케이션 로딩 시점에 자동 프록시 생성기를 호출한다.
  - 모든 @Aspect 빈 조회: 자동 프록시 생성기는 스프링 컨테이너에서 @Aspect 애노테이션이 붙은 스프링 빈을 모두 조회한다.
  - 어드바이저 생성: @Aspect 어드바이저 빌더를 통해 @Aspect 애노테이션 정보를 기반으로 어드바이저를 생성한다.
  - @Aspect 기반 어드바이저 저장: 생성한 어드바이저를 @Aspect 어드바이저 빌더 내부에 저장한다.
- `자동 프록시 생성기의 작동 과정`
  - 생성: 스프링 빈 대상이 되는 객체를 생성한다. ( @Bean , 컴포넌트 스캔 모두 포함)
  - 전달: 생성된 객체를 빈 저장소에 등록하기 직전에 빈 후처리기에 전달한다.
  - Advisor 빈 조회: 스프링 컨테이너에서 Advisor 빈을 모두 조회한다.
  - @Aspect Advisor 조회: @Aspect 어드바이저 빌더 내부에 저장된 Advisor 를 모두 조회한다. 
  - 프록시 적용 대상 체크: 앞서 조회한 Advisor들에 포함되어 있는 포인트컷을 사용해서 해당 객체가 프록시를 적용할 대상인지 아닌지 판단한다. 이때 객체의 클래스 정보는 물론이고, 해당 객체의 모든 메서드를 포인트컷에 하나하나 모두 매칭해본다. 그래서 조건이 하나라도 만족하면 프록시 적용 대상이 된다.
  - 프록시 생성: 프록시 적용 대상이면 프록시를 생성하고 프록시를 반환한다. 그래서 프록시를 스프링 빈으로 등록한다. 만약 프록시 적용 대상이 아니라면 원본 객체를 반환해서 원본 객체를 스프링 빈으로 등록한다.
  - 빈 등록: 반환된 객체는 스프링 빈으로 등록

스프링 AOP 개념
=======
- `AOP 핵심 기능과 부가기능`
  - 핵심 기능: 해당 객체가 제공하는 고유의 기능
  - 부가 기능: 핵심 기능을 보조하기 위해 제공되는 기능으로 다른 객체들과 공통적으로 사용될 가능성이 크므로 변경 지점이 하나가 될 수 있도록 잘 모듈화되어야함
- `Aspect`
  - Aspect: 관점이라는 뜻으로 이름 그대로 애플리케이션을 바라보는 관점을 하나하나의 기능에서 횡단 관심사(cross-cutting concerns) 관점으로 보는 것
  - AOP: 애스펙트를 사용한 프로그래밍 방식을 관점 지향 프로그래밍
  - AspectJ 프레임워크: 자바 프로그래밍 언어에 대한 완벽한 관점 지향 프레임워크로 스프링은 대부분 AspectJ의 문법을 차용하여 필요한 기능들만 편리하게 사용할 수 있는 AOP를 제공
  - 스프링 AOP는 AspectJ의 문법을 차용하고 프록시 방식의 AOP를 제공하는데 AspectJ를 직접 사용하는 것아니며 @Aspect 애노테이션 같은 AspectJ 프레임워크가 제공하는 애노테이션을 사용
- `AOP 적용 방식`
  - 컴파일 시점: .java 소스 코드를 컴파일러를 사용해서 .class 를 만드는 시점에 부가 기능 로직을 추가 (실제 코드에 부가 기능 코드 포함)
  - 클래스 로딩 시점: .class 를 JVM에 저장하기 전에 조작 (실제 코드에 부가 기능 코드 포함)
  - 런타임 시점(프록시): `스프링이 사용하는 방식`으로 컴파일이 다 끝나고, 클래스 로더에 클래스도 다 올라가서 이미 자바가 실행되고 난 다음인 런타임 시점에서 프록시를 통해 스프링 빈에 부가 기능을 적용
- `AOP 적용 위치`
  - 적용 가능 지점(조인 포인트): 생성자, 필드 값 접근, static 메서드 접근, 메서드 실행
  - 프록시를 사용하는 스프링 AOP의 조인 포인트는 메서드 실행으로 제한
- `스프링 AOP 용어 정리`
  - 조인 포인트(Join point)
    - 어드바이스가 적용될 수 있는 위치, 메소드 실행, 생성자 호출, 필드 값 접근, static 메서드 접근 같은 프로그램 실행 중 지점
    - AOP를 적용할 수 있는 모든 지점
    - 스프링 AOP는 프록시 방식을 사용하므로 조인 포인트는 항상 `메소드 실행 지점`으로 제한
  - 포인트컷(Pointcut)
    - 조인 포인트 중에서 어드바이스가 적용될 위치를 선별하는 기능
    - 주로 AspectJ 표현식을 사용해서 지정
    - 프록시를 사용하는 스프링 AOP는 `메서드 실행 지점`만 포인트컷으로 선별 가능
  - 타켓(Target)
    - 어드바이스를 받는 `실제 객체`, 포인트컷으로 결정됨
  - 어드바이스(Advice)
    - 부가 기능
    - 특정 조인 포인트에서 Aspect에 의해 취해지는 조치
    - Around(주변), Before(전), After(후)와 같은 다양한 종류의 어드바이스가 있음
  - 에스팩트
    - 어드바이스 + 포인트컷을 모듈화 한 것
    - 여러 어드바이스와 포인트 컷이 함께 존재
  - 어드바이저
    - 하나의 어드바이스와 하나의 포인트 컷으로 구성
  - 위빙
    - 원본 로직에 부가 기능 로직이 추가되는 것

스프링 AOP 구현
=======
- `스프링 AOP 기본 형태 예제 코드`
```java
@Slf4j
@Aspect // 애스펙트라는 표식이지 컴포넌트 스캔이 되는 것은 아니므로 @Bean, @Component, @Import 등으로 빈으로 등록해야함
public class AspectV1 {
   
   @Around("execution(* hello.aop.order..*(..))") // AspectJ 포인트컷 표현식 (hello.aop.order 패키지와 하위 패키지)
   public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable { // doLog는 어드바이스 함수
      log.info("[log] {}", joinPoint.getSignature()); //join point 시그니처
      return joinPoint.proceed(); 
   }
}
```
```java
@Import(AspectV1.class)
@SpringBootTest
public class AopTest {

   @Autowired
   OrderService orderService;
   
   @Autowired
   OrderRepository orderRepository;
   
   @Test
   void aopInfo() {
      System.out.println("isAopProxy, orderService=" + AopUtils.isAopProxy(orderService));
      System.out.println("isAopProxy, orderRepository=" + AopUtils.isAopProxy(orderRepository));
   }
   
   @Test
   void success() {
      orderService.orderItem("itemA"); 
   }

   @Test
   void exception() {
      assertThatThrownBy(() -> orderService.orderItem("ex")) 
               .isInstanceOf(IllegalStateException.class);
   }
}
```
- `포인트컷 분리`
  - @Around 어드바이스에서는 포인트컷을 직접 지정해도 되지만 @Pointcut 에 포인트컷 표현식을 사용하여 분리 가능
  - 메서드 이름과 파라미터를 합쳐서 포인트컷 시그니처이며 메서드의 반환 타입은 void 여야함
- 포인트컷 분리 예제 코드
```java
@Slf4j
@Aspect
public class AspectV2 {

   //hello.aop.order 패키지와 하위 패키지
   @Pointcut("execution(* hello.aop.order..*(..))") //pointcut expression
   private void allOrder(){} //pointcut signature
   
   @Around("allOrder()")
   public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
      log.info("[log] {}", joinPoint.getSignature());
      return joinPoint.proceed(); 
   }
}
```
- `어드바이스 추가`
  - 포인트컷은 조합할 수 있는데, && (AND), || (OR), ! (NOT) 3가지 조합이 가능
- 어드바이스 추가 예제 코드
```java
@Slf4j
@Aspect
public class AspectV3 {

   //hello.aop.order 패키지와 하위 패키지 
   @Pointcut("execution(* hello.aop.order..*(..))") 
   public void allOrder(){}
   
   //클래스 이름 패턴이 *Service 
   @Pointcut("execution(* *..*Service.*(..))") 
   private void allService(){}

   @Around("allOrder()")
   public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
      log.info("[log] {}", joinPoint.getSignature());
      return joinPoint.proceed(); 
   }
   
   //hello.aop.order 패키지와 하위 패키지 이면서 클래스 이름 패턴이 *Service 
   @Around("allOrder() && allService()")
   public Object doTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
      
      try {
         log.info("[트랜잭션 시작] {}", joinPoint.getSignature()); 
         Object result = joinPoint.proceed();
         log.info("[트랜잭션 커밋] {}", joinPoint.getSignature());
         return result;
      } catch (Exception e) {
         log.info("[트랜잭션 롤백] {}", joinPoint.getSignature());
         throw e;
      } finally {
         log.info("[리소스 릴리즈] {}", joinPoint.getSignature()); 
      }
   }
}
```
- `포인트컷 참조`
  - 포인트컷을 공용으로 사용하기 위해 별도의 외부 클래스에 모아두어야하며 참고로 외부에서 호출할 때는 포인트컷의 접근 제어자를 public 으로 열어야함
- 포인트컷 참조 예제 코드
```java
public class Pointcuts {

   //hello.springaop.app 패키지와 하위 패키지 
   @Pointcut("execution(* hello.aop.order..*(..))") 
   public void allOrder(){}
   
   //타입 패턴이 *Service
   @Pointcut("execution(* *..*Service.*(..))") 
   public void allService(){}
   
   //allOrder && allService
   @Pointcut("allOrder() && allService()")
   public void orderAndService(){}
}
```
```java
@Slf4j
@Aspect
public class AspectV4Pointcut {

   @Around("hello.aop.order.aop.Pointcuts.allOrder()")
   public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable { 
      log.info("[log] {}", joinPoint.getSignature());
      return joinPoint.proceed();
   }

   @Around("hello.aop.order.aop.Pointcuts.orderAndService()")
   public Object doTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
      
      try {
         log.info("[트랜잭션 시작] {}", joinPoint.getSignature());
         Object result = joinPoint.proceed(); 
         log.info("[트랜잭션 커밋] {}", joinPoint.getSignature());
         return result;
      } catch (Exception e) {
         log.info("[트랜잭션 롤백] {}", joinPoint.getSignature());
         throw e;
      } finally {
         log.info("[리소스 릴리즈] {}", joinPoint.getSignature()); 
      }
   }
}
```
- `어드바이스 순서`
  - 하나의 애스펙트에 여러 어드바이스가 있으면 순서를 보장 받을 수 없으므로 애스펙트를 별도의 클래스로 분리해야함
- 어드바이스 순서 예제 코드
```java
@Slf4j
public class AspectV5Order {
   
   @Aspect
   @Order(2)
   public static class LogAspect {
      @Around("hello.aop.order.aop.Pointcuts.allOrder()")
       public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable { 
          log.info("[log] {}", joinPoint.getSignature());
          return joinPoint.proceed();
   }

   @Aspect
   @Order(1)
   public static class TxAspect {
      
      @Around("hello.aop.order.aop.Pointcuts.orderAndService()")
      public Object doTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
         
         try {
            log.info("[트랜잭션 시작] {}", joinPoint.getSignature());
            Object result = joinPoint.proceed(); 
            log.info("[트랜잭션 커밋] {}", joinPoint.getSignature());
            return result;
         } catch (Exception e) {
            log.info("[트랜잭션 롤백] {}", joinPoint.getSignature());
            throw e;
         } finally {
            log.info("[리소스 릴리즈] {}", joinPoint.getSignature()); 
         }
      }     
   }
}
```
- `어드바이스 종류`
  - `@Around`: 메서드 호출 전후에 수행, 가장 강력한 어드바이스, 조인 포인트 실행 여부 선택, 반환 값 변환, 예외 변환 등이 가능
  - `@Before`: 조인 포인트 실행 이전에 실행
  - `@AfterReturning`: 조인 포인트가 정상 완료후 실행
  - `@AfterThrowing`: 메서드가 예외를 던지는 경우 실행
  - `@After`: 조인 포인트가 정상 또는 예외에 관계없이 실행(finally)
  > 모든 어드바이스는 `org.aspectj.lang.JoinPoint`를 첫번째 파라미터에 사용하나 @Around는 JoinPoint의 하위 타입인 ProceedingJoinPoint을 사용해야함
- @Around
  - 메서드의 실행의 주변에서 실행
  - 조인 포인트 실행 여부 선택 joinPoint.proceed() 
  - 호출 여부 선택 전달 값 변환: joinPoint.proceed(args[])
  - 반환 값 변환
  - 예외 변환
  - 트랜잭션 처럼 try catch finally 모두 들어가는 구문 처리 가능 
  - 어드바이스의 첫 번째 파라미터는 ProceedingJoinPoint 를 사용
  - proceed() 를 통해 대상을 실행
  - @Around 하나만 있어도 모든 기능을 수행 가능
- JoinPoint 인터페이스의 주요 기능
  - getArgs(): 메서드 인수를 반환
  - getThis(): 프록시 객체를 반환
  - getTarget(): 대상 객체를 반환
  - getSignature(): 조언되는 메서드에 대한 설명을 반환
  - toString(): 조언되는 방법에 대한 유용한 설명을 인쇄
- ProceedingJoinPoint 인터페이스의 주요 기능 
  - proceed(): 다음 어드바이스나 타켓을 호출
- 어드바이스 종류 예제 코드
```java
@Slf4j
@Aspect
public class AspectV6Advice {
   
   @Around("hello.aop.order.aop.Pointcuts.orderAndService()")
   public Object doTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
      try {
         
         // @Before
         log.info("[around][트랜잭션 시작] {}", joinPoint.getSignature()); 
         Object result = joinPoint.proceed();
         
         // @AfterReturning
         log.info("[around][트랜잭션 커밋] {}", joinPoint.getSignature());
         return result;

      } catch (Exception e) {
            
         // @AfterThrowing
         log.info("[around][트랜잭션 롤백] {}", joinPoint.getSignature());
         throw e;

      } finally {
         
         // @After
         log.info("[around][리소스 릴리즈] {}", joinPoint.getSignature()); 
      }
   }

   @Before("hello.aop.order.aop.Pointcuts.orderAndService()") 
   public void doBefore(JoinPoint joinPoint) {
      log.info("[before] {}", joinPoint.getSignature()); 
   }
   
   @AfterReturning(value = "hello.aop.order.aop.Pointcuts.orderAndService()", returning = "result")public void doReturn(JoinPoint joinPoint, Object result) { 
      log.info("[return] {} return={}", joinPoint.getSignature(), result);
   }
   
   @AfterThrowing(value = "hello.aop.order.aop.Pointcuts.orderAndService()", throwing = "ex")
   public void doThrowing(JoinPoint joinPoint, Exception ex) { 
      log.info("[ex] {} message={}", joinPoint.getSignature(), ex.getMessage()); 
   }
   
   @After(value = "hello.aop.order.aop.Pointcuts.orderAndService()") 
   public void doAfter(JoinPoint joinPoint) {
      log.info("[after] {}", joinPoint.getSignature()); 
   }
}
```

스프링 AOP 포인트컷
=======
- `포인트컷 지시자`
  - 포인트컷 표현식은 애스펙트J가 제공하는 포인트컷 표현식을 줄여서 말하는 것으로 execution 같은 `포인트컷 지시자`(Pointcut Designator)로 시작
- 포인트컷 지시자의 종류
  - `execution`: 메소드 실행 조인 포인트를 매칭 (스프링 AOP에서 가장 많이 사용)
  - within: 특정 타입 내의 조인 포인트를 매칭
  - args: 인자가 주어진 타입의 인스턴스인 조인 포인트
  - this: 스프링 빈 객체(스프링 AOP 프록시)를 대상으로 하는 조인 포인트
  - target: Target 객체(스프링 AOP 프록시가 가르키는 실제 대상)를 대상으로 하는 조인 포인트 
  - @target: 실행 객체의 클래스에 주어진 타입의 애노테이션이 있는 조인 포인트
  - @within: 주어진 애노테이션이 있는 타입 내 조인 포인트
  - @annotation: 메서드가 주어진 애노테이션을 가지고 있는 조인 포인트를 매칭
  - @args: 전달된 실제 인수의 런타임 타입이 주어진 타입의 애노테이션을 갖는 조인 포인트 
  - bean: 스프링 전용 포인트컷 지시자, 빈의 이름으로 포인트컷을 지정
- `execution`
  - execution 문법: `execution(접근제어자? 반환타입 선언타입?메서드이름(파라미터) 예외?)`
    - 메소드 실행 조인 포인트를 매칭
    - ?는 생략가능
    - * 같은 패턴 지정 가능 
    - * 은 아무 값이 들어와도 된다는 뜻
    - 파라미터에서 `..`은 파라미터의 타입과 파라미터 수가 상관없다는 뜻
    - 패키지에서 `.`은 정확하게 해당 위치의 패키지 뜻이고 `..`은 해당 위치의 패키지와 그 하위 패키지
  - execution 예시 => "execution(public String hello.aop.member.MemberServiceImpl.hello(String))"
    - 접근제어자?: public
    - 반환타입: String
    - 선언타입?: hello.aop.member.MemberServiceImpl 
    - 메서드이름: hello
    - 파라미터: (String)
    - 예외?: 생략
- execution 타입 매칭 예시
```java
// 타입 정보가 정확하게 일치
@Test
void typeExactMatch() {
   pointcut.setExpression("execution(* hello.aop.member.MemberServiceImpl.*(..))"); 
   assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue(); }

// 부모 타입을 선언해도 그 자식 타입은 매칭
@Test
void typeMatchSuperType() {
   pointcut.setExpression("execution(* hello.aop.member.MemberService.*(..))");
   assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}
```
- execution 파라미터 매칭 예시
```java
// String 타입의 파라미터 허용 
// (String)
@Test
void argsMatch() { 
   pointcut.setExpression("execution(* *(String))"); assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue(); 
}

// 파라미터가 없어야 함
// ()
@Test
void argsMatchNoArgs() {
   pointcut.setExpression("execution(* *())");
   assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isFalse();
}

// 정확히 하나의 파라미터 허용, 모든 타입 허용 
// (Xxx)
@Test
void argsMatchStar() {
   pointcut.setExpression("execution(* *(*))");
   assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

// 숫자와 무관하게 모든 파라미터, 모든 타입 허용 
// 파라미터가 없어도 됨
// (), (Xxx), (Xxx, Xxx)
@Test
void argsMatchAll() { 
   pointcut.setExpression("execution(* *(..))"); assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue(); 
}

// String 타입으로 시작, 숫자와 무관하게 모든 파라미터, 모든 타입 허용 
// (String), (String, Xxx), (String, Xxx, Xxx) 허용
@Test
void argsMatchComplex() {
   pointcut.setExpression("execution(* *(String, ..))");
   assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}
```

스프링 AOP 실무 주의사항
=======
- `내부 호출 문제`
  - AOP를 적용하려면 항상 프록시를 통해서 대상 객체(Target)을 호출해야하며 프록시를 스프링 빈으로 등록하여 프록시 객체가 주입되기 때문에 대상 객체를 직접 호출하는 문제는 일반적으로 발생하지 않는데, 대상 객체의 내부에서 메서드 호출이 발생하면 프록시를 거치지 않고 대상 객체를 직접 호출하기에 `프록시 방식의 AOP는 메서드 내부 호출에 프록시를 적용할 수 없음`
  > AspectJ를 사용하면 이런 문제가 발생하지 않으나 실무에서는 거의 사용하지 않음
- 내부 호출 문제 예시 코드
```java
@Slf4j
@Component
public class CallServiceV0 {
   public void external() {
      log.info("call external");
      internal(); //내부 메서드 호출(this.internal())
   }

   public void internal() { 
      log.info("call internal");
   } 
}
```
```java
@Slf4j
@Aspect
public class CallLogAspect {
   
   @Before("execution(* hello.aop.internalcall..*.*(..))") 
   public void doLog(JoinPoint joinPoint) {
      log.info("aop={}", joinPoint.getSignature()); 
   }
}
```
```java
@Import(CallLogAspect.class) 
@SpringBootTest
class CallServiceV0Test {

   @Autowired
   CallServiceV0 callServiceV0;
   
   @Test
   void external() {
      callServiceV0.external(); 
   }
   
   @Test
   void internal() {
      callServiceV0.internal(); 
   }
}
```
- `내부 호출 문제 대안 1`
```java
/**
* 자기 자신 주입
*/
@Slf4j
@Component
public class CallServiceV1 {
   
   private CallServiceV1 callServiceV1;
   
   @Autowired
   public void setCallServiceV1(CallServiceV1 callServiceV1) { // 생성자 주입은 순환 사이클을 만들기 때문에 실패하여 setter 주입
      this.callServiceV1 = callServiceV1; 
   }
   
   public void external() {
      log.info("call external"); 
      callServiceV1.internal(); //외부 메서드 호출
   }
   
   public void internal() { 
      log.info("call internal");
   } 
}
```
- `내부 호출 문제 대안 2`
```java
/**
* ObjectProvider(Provider)나 ApplicationContext를 사용해서 지연(LAZY) 조회
*/
@Slf4j
@Component
@RequiredArgsConstructor
public class CallServiceV2 {
  
  //  private final ApplicationContext applicationContext;
   private final ObjectProvider<CallServiceV2> callServiceProvider;
   
   public void external() { 
      log.info("call external");
      
      // CallServiceV2 callServiceV2 = applicationContext.getBean(CallServiceV2.class);
      CallServiceV2 callServiceV2 = callServiceProvider.getObject(); 
      callServiceV2.internal(); //외부 메서드 호출
   }
   
   public void internal() { 
      log.info("call internal");
   } 
}
```
- `내부 호출 문제 대안 3`
```java
/**
* 구조를 변경(분리)
*/
@Slf4j
@Component
@RequiredArgsConstructor
public class CallServiceV3 {
   
   private final InternalService internalService;
   
   public void external() {
      log.info("call external"); 
      internalService.internal(); //외부 메서드 호출
   } 
}
```
```java
@Slf4j
@Component
public class InternalService {

    public void internal() {
        log.info("call internal"); 
   }
}
```
- `프록시 기술과 한계`
  - `JDK 동적 프록시` 인터페이스 기반으로 프록시를 생성하기 때문에 구체 클래스로 타입 캐스팅이 불가능
  - `CGLIB `은 구체 클래스 기반으로 하기 때문에 final 키워드 클래스, 메서드 사용이 불가능
