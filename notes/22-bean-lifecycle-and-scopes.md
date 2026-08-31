# 22 — Spring Core: IoC, Dependency Injection, Bean Lifecycle & Scopes

## 1. IoC & Dependency Injection

### The problem

Without dependency injection, a class may construct its own dependencies:

```java
class OrderService {
    private final PaymentService paymentService;

    public OrderService() {
        this.paymentService = new PaymentService();
    }
}
```

This makes the class responsible for construction and tightly couples it to the concrete dependency.

### Dependency Injection

Instead, make the dependency explicit and let an external component provide it:

```java
class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
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
public class OrderService {
}
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

DI is especially useful when depending on abstractions. When multiple implementations exist, Spring provides mechanisms such as `@Primary` and `@Qualifier` to select one.

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
public class ReportGenerator {
}
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

`JobService` is a singleton. When Spring creates it, it resolves the `Worker` dependency and injects an instance once. The prototype scope does not magically create a new `Worker` on every method call.

If a fresh prototype instance is required each time, use a provider/factory mechanism such as `ObjectProvider` rather than relying on direct injection alone.

## 9. Spring Boot Auto-Configuration

Spring Boot's major contribution is making configuration largely automatic based on the application's dependencies and configuration.

For example, adding JPA and PostgreSQL dependencies can allow Boot to configure infrastructure such as a `DataSource`, JPA/Hibernate infrastructure, and transaction-related infrastructure without manually declaring every bean.

The basic model is:

```text
Dependencies on classpath
        +
Application configuration
        +
Existing beans
        ↓
Spring Boot Auto-Configuration
        ↓
Configure appropriate infrastructure
```

### Conditional auto-configuration

Auto-configuration is primarily **conditional**. Boot does not blindly create everything.

Conceptually:

```text
Relevant classes available?
        ↓
Required configuration present?
        ↓
User already defined the bean?
        ↓
Yes → configure
No  → don't configure / back off
```

A key principle is:

> **Convention by default, customization when needed.**

If you provide your own relevant configuration, Boot generally backs off rather than creating a competing default.

### Classpath-driven configuration

The classpath is a major input. Adding a starter/dependency makes relevant classes available, allowing Boot's conditions to activate appropriate auto-configuration.

```text
Add dependency
      ↓
Classes available on classpath
      ↓
Boot's conditions detect them
      ↓
Relevant auto-configuration activates
```

## 10. `@SpringBootApplication`

Typical application entry point:

```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

`@SpringBootApplication` is a convenience annotation combining:

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
```

### `@SpringBootConfiguration`

Identifies the class as a Spring Boot configuration class. It is effectively a specialized form of `@Configuration` for a Boot application.

### `@EnableAutoConfiguration`

Activates Spring Boot's auto-configuration mechanism.

### `@ComponentScan`

Tells Spring where to search for application components such as:

```java
@Component
@Service
@Repository
@Controller
@RestController
```

The default scan starts from the package of the application class and covers its subpackages. Therefore the main application class should normally be placed near the root package.

## 11. Component scanning vs auto-configuration

These are related but different.

**Component scanning:**

> Finds and registers your application's components.

Examples:

```text
@Service
@Repository
@RestController
@Component
```

**Auto-configuration:**

> Configures framework/infrastructure beans based on the environment.

Examples include infrastructure for:

```text
DataSource
JPA/Hibernate
MVC
Jackson
```

A useful startup mental model is:

```text
main()
  ↓
SpringApplication.run()
  ↓
Create ApplicationContext
  ↓
Component scanning
  ↓
Discover configuration/classes
  ↓
Apply auto-configuration
  ↓
Register bean definitions
  ↓
Create/wire beans
  ↓
Application ready
```

## 12. Autowiring

Autowiring is dependency resolution between beans.

Given:

```java
@Service
public class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

Spring looks in the container for a suitable `PaymentService` bean. If exactly one candidate exists, it can inject it.

### Multiple candidates

If multiple implementations exist:

```text
PaymentService
     ↑
 ┌───┴──────────────┐
 │                  │
Stripe          Razorpay
```

Spring cannot arbitrarily choose between them and can fail with a `NoUniqueBeanDefinitionException`.

### `@Primary`

```java
@Service
@Primary
public class StripePaymentService implements PaymentService {
}
```

`@Primary` marks an implementation as the default candidate when multiple candidates exist.

Think:

> **`@Primary` = "This is the default."**

### `@Qualifier`

When a specific implementation is required:

```java
@Service("stripe")
public class StripePaymentService implements PaymentService {
}

@Service("razorpay")
public class RazorpayPaymentService implements PaymentService {
}
```

```java
public OrderService(
        @Qualifier("razorpay") PaymentService paymentService) {
    this.paymentService = paymentService;
}
```

Think:

> **`@Qualifier` = "I specifically want this one."**

If both are applicable, the qualifier provides explicit selection.

### Do we need `@Autowired`?

With a single constructor, `@Autowired` is unnecessary:

```java
public OrderService(PaymentService paymentService) {
    this.paymentService = paymentService;
}
```

Spring recognizes the constructor for injection.

Constructor injection is preferred over field injection because dependencies are explicit, required dependencies can be `final`, and unit testing is straightforward.

## 13. Auto-configuration vs Autowiring

Do not conflate them.

**Auto-configuration:**

> What infrastructure/configuration should Spring Boot create based on the environment?

**Autowiring:**

> Which existing bean should be injected into another bean?

```text
Auto-configuration
→ configuration/creation of infrastructure

Autowiring
→ dependency resolution/injection
```

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

> **Auto-configuration** configures Spring Boot infrastructure based on classpath dependencies, configuration, and conditions, and generally backs off when you provide your own relevant configuration.

> **`@SpringBootApplication`** combines `@SpringBootConfiguration`, `@EnableAutoConfiguration`, and `@ComponentScan`.

> **Component scanning** finds your application's components; **auto-configuration** configures framework/infrastructure beans.

> **Autowiring** resolves dependencies between beans.

> Multiple matching beans require disambiguation such as `@Primary` or `@Qualifier`.

> With a single constructor, `@Autowired` is unnecessary.

> **Auto-configuration ≠ autowiring**: auto-configuration configures infrastructure; autowiring injects dependencies between existing beans.
