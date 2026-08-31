# 22 — Spring Bean Lifecycle & Scopes

## 1. Bean lifecycle

Spring manages a bean through a lifecycle rather than simply calling its constructor and leaving the object alone.

A useful interview-level model is:

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

Do not treat this as an exhaustive list of every internal callback. It is the useful mental model for an SDE-2 interview.

## 2. Constructor vs @PostConstruct

```java
@Service
public class PaymentService {

    private final PaymentClient client;

    public PaymentService(PaymentClient client) {
        this.client = client;
    }

    @PostConstruct
    void init() {
        // initialization logic
    }
}
```

The constructor runs first. With constructor injection, required dependencies are available to the bean through the constructor.

`@PostConstruct` runs after dependency injection, so it is appropriate for initialization that requires the bean's dependencies to already be available.

Interview distinction:

> Constructor → dependency injection → initialization callbacks.

Heavyweight startup work should not be placed casually in `@PostConstruct`, because it is part of bean initialization and can affect application startup.

## 3. Bean scopes

A bean scope answers:

> **How many instances should Spring create, and how long should they live?**

The most important scopes for an SDE-2 interview are singleton and prototype.

### Singleton — default

A typical Spring bean such as:

```java
@Service
public class OrderService {
}
```

is singleton-scoped by default.

Spring creates one instance per `ApplicationContext`:

```text
ApplicationContext
       │
       └── OrderService
              ↑
        one instance
```

Conceptually:

```java
OrderService a = context.getBean(OrderService.class);
OrderService b = context.getBean(OrderService.class);

a == b  // true
```

Important interview trap:

> Singleton means one instance per Spring `ApplicationContext`, not necessarily one instance for the entire JVM or all application contexts.

### Prototype

```java
@Scope("prototype")
@Component
public class ReportGenerator {
}
```

Spring creates a new instance when the bean is requested from the container:

```text
getBean()
   ↓
ReportGenerator #1

getBean()
   ↓
ReportGenerator #2
```

So separate lookups return different instances.

Spring manages creation/configuration of the prototype bean, but does not manage its complete lifecycle after handing it to the caller in the same way it manages singleton destruction.

### Web scopes

In web applications, other scopes include:

- request — one instance per HTTP request
- session — one instance per HTTP session
- application — one instance per web application's `ServletContext`

Know these conceptually; do not spend significant revision time memorizing every scope.

## 4. Singleton beans and mutable state

Because singleton beans are shared, multiple HTTP requests can execute against the same instance concurrently:

```text
Request A ──┐
             ├──> same OrderService instance
Request B ──┘
```

Therefore singleton service beans should generally be stateless.

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

## 5. Prototype injected into a singleton

A common trap is assuming that a prototype dependency injected into a singleton is recreated every time the singleton uses it.

Example:

```java
@Component
@Scope("prototype")
class Worker {
}

@Service
class JobService {

    private final Worker worker;

    JobService(Worker worker) {
        this.worker = worker;
    }
}
```

`JobService` is a singleton. When Spring creates it, it resolves the `Worker` dependency and injects an instance once:

```text
Application startup

Worker #1
   ↓
JobService singleton
   ↓
keeps Worker #1
```

The prototype scope does not magically create a new `Worker` on every method call.

If a fresh prototype instance is needed each time, use a provider/factory mechanism such as `ObjectProvider` rather than relying on direct injection alone.

## Interview Quick Recall

> Spring manages beans through a lifecycle: construction, dependency injection, initialization callbacks, use, and destruction.

> `@PostConstruct` runs after dependency injection and is useful when initialization requires injected dependencies.

> Singleton is the default scope and means one instance per `ApplicationContext`.

> Prototype creates a new instance when obtained from the container, but Spring does not manage its destruction like a singleton bean.

> Singleton beans are shared across concurrent requests, so service beans should generally be stateless and avoid unsafe mutable instance state.

> A prototype dependency injected directly into a singleton is not recreated for every method call; it is resolved when the singleton is created.
