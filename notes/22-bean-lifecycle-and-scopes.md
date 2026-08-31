# 22 — Spring Core: IoC, Dependency Injection, Bean Lifecycle & Scopes

## 1. IoC & Dependency Injection

### The problem

Without dependency injection, a class may construct its own dependencies:

```java
class OrderService {
    private final PaymentService paymentService;
    public OrderService() { this.paymentService = new PaymentService(); }
}
```

This makes the class responsible for construction and tightly couples it to the concrete dependency.

### Dependency Injection

Instead, make the dependency explicit and let an external component provide it:

```java
class OrderService {
    private final PaymentService paymentService;
    public OrderService(PaymentService paymentService) { this.paymentService = paymentService; }
}
```

> **Dependency Injection (DI) is a design pattern where dependencies are supplied externally rather than constructed by the dependent class.**

### IoC

**IoC = Inversion of Control.** Normally application code controls object creation. With Spring, control over object creation and wiring is transferred to the Spring container.

> **IoC is the broader principle: control is transferred to the framework/container. DI is a common mechanism for achieving IoC.**

## 2. Spring IoC Container

Spring's `ApplicationContext` acts as the IoC container. It discovers/configures beans, instantiates them, resolves dependencies, wires them together, manages their lifecycle, and makes them available to the application.

A useful mental model:

```text
Spring Container
   ↓
creates PaymentClient
   ↓
creates PaymentService
   ↓
creates OrderService
```

Conceptually this resembles:

```java
PaymentService paymentService = new PaymentService();
OrderService orderService = new OrderService(paymentService);
```

Real Spring also handles lifecycle management, proxies, scopes, configuration, and post-processing.

## 3. Spring Beans

A **bean is an object whose lifecycle is managed by the Spring IoC container.**

```java
@Service
public class OrderService { }
```

Beans can also be explicitly declared:

```java
@Configuration
public class AppConfig {
    @Bean
    public PaymentClient paymentClient() {
        return new PaymentClient();
    }
}
```

## 4. Why constructor injection is preferred

```java
@Service
public class OrderService {
    private final PaymentService paymentService;
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

With a single constructor, Spring can use it without `@Autowired`.

Constructor injection:

- makes dependencies explicit
- allows dependencies to be `final`
- prevents creating the object without required dependencies
- makes unit testing straightforward
- avoids hidden dependencies through field injection
- makes dependency relationships more visible

```java
OrderService service = new OrderService(mockPaymentService);
```

### Interface + DI

DI is especially useful when depending on abstractions:

```java
public interface PaymentService {
    void pay(Order order);
}
```

```java
@Service
public class StripePaymentService implements PaymentService { }
```

`OrderService` can depend on `PaymentService` rather than a concrete implementation. When multiple implementations exist, Spring provides mechanisms such as `@Primary` and `@Qualifier` to select one.

## 5. Bean lifecycle

Spring manages a bean through a lifecycle rather than simply calling its constructor and handing the object to the application.

```text
1. Instantiate bean
       ↓
2. Inject dependencies
       ↓
3. Aware callbacks (if applicable)
       ↓
4. BeanPostProcessor — before initialization
       ↓
5. @PostConstruct
       ↓
6. BeanPostProcessor — after initialization
       ↓
7. Bean is ready
       ↓
8. Application uses bean
       ↓
9. @PreDestroy
       ↓
10. Bean destroyed
```

This is an interview-level model, not an exhaustive list of every callback.

### Constructor vs `@PostConstruct`

The constructor runs first. With constructor injection, required dependencies are supplied through it. `@PostConstruct` runs after dependency injection, so it is appropriate for initialization that requires injected dependencies.

> **Constructor → dependency injection → initialization callbacks.**

Heavyweight startup work should not be placed casually in `@PostConstruct`, because it is part of bean initialization and can affect application startup.

## 6. Bean scopes

A bean scope answers:

> **How many instances should Spring create, and how long should they live?**

The most important scopes for SDE-2 are singleton and prototype.

### Singleton — default

Spring creates one instance per `ApplicationContext`:

```java
OrderService a = context.getBean(OrderService.class);
OrderService b = context.getBean(OrderService.class);
a == b  // true
```

Important interview trap:

> **Singleton means one instance per Spring `ApplicationContext`, not necessarily one instance for the entire JVM or all application contexts.**

### Prototype

```java
@Scope("prototype")
@Component
public class ReportGenerator { }
```

Spring creates a new instance when the bean is requested from the container. Separate lookups return different instances.

Spring manages creation/configuration of the prototype bean, but does not manage its destruction in the same way it manages singleton destruction.

### Web scopes

In web applications:

- **request** — one instance per HTTP request
- **session** — one instance per HTTP session
- **application** — one instance per web application's `ServletContext`

Know these conceptually; singleton/prototype matter more for this revision.

## 7. Singleton beans and mutable state

Singleton beans are shared, so multiple HTTP requests can execute against the same instance concurrently.

Bad:

```java
@Service
public class OrderService {
    private Order currentOrder;  // shared mutable state
    public void process(Order order) {
        this.currentOrder = order;
    }
}
```

Good:

```java
@Service
public class OrderService {
    public void process(Order order) {
        // use local variables / immutable state
    }
}
```

Shared mutable instance state in a singleton can introduce race conditions and thread-safety bugs.

## 8. Prototype injected into a singleton

A common trap is assuming a prototype dependency injected into a singleton is recreated every time the singleton uses it.

```java
@Component
@Scope("prototype")
class Worker { }

@Service
class JobService {
    private final Worker worker;
    JobService(Worker worker) {
        this.worker = worker;
    }
}
```

`JobService` is a singleton. When Spring creates it, it resolves the `Worker` dependency and injects an instance once. The prototype scope does not magically create a new `Worker` on every method call.

If a fresh prototype instance is required each time, use a provider/factory mechanism such as `ObjectProvider` rather than relying on direct injection alone.

## Interview Quick Recall

> **IoC** = control over object creation/wiring is transferred to the container/framework.

> **DI** = dependencies are supplied externally rather than constructed by the dependent class.

> **ApplicationContext** = Spring's IoC container that creates, wires, and manages beans.

> **Bean** = an object managed by the Spring container.

> Constructor injection makes dependencies explicit, supports `final` fields, and makes testing straightforward.

> Singleton is the default scope and means one instance per `ApplicationContext`.

> Prototype creates a new instance when obtained from the container, but Spring does not manage its destruction like a singleton bean.

> `@PostConstruct` runs after dependency injection and is useful when initialization requires injected dependencies.

> Singleton beans are shared across concurrent requests, so service beans should generally be stateless and avoid unsafe mutable instance state.

> A prototype dependency injected directly into a singleton is not recreated for every method call; it is resolved when the singleton is created.

> Spring's DI mechanism also enables framework features such as proxies, which becomes important for AOP and `@Transactional`.
