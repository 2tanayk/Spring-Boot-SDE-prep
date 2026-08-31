# 23 — Spring Boot Auto-Configuration, Autowiring & `@SpringBootApplication`

## 1. Spring Boot Auto-Configuration

Spring Boot makes configuration largely automatic based on application dependencies, configuration, and the existing beans in the container.

For example, adding JPA and PostgreSQL dependencies can allow Boot to configure infrastructure such as a `DataSource`, JPA/Hibernate infrastructure, and transaction-related infrastructure without manually declaring every bean.

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

A key principle:

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

## 2. `@SpringBootApplication`

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

## 3. Component scanning vs auto-configuration

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

A useful startup mental model:

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

## 4. Autowiring

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

## 5. Auto-configuration vs Autowiring

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

> **Auto-configuration** configures Spring Boot infrastructure based on classpath dependencies, configuration, and conditions, and generally backs off when you provide your own relevant configuration.

> **`@SpringBootApplication`** combines `@SpringBootConfiguration`, `@EnableAutoConfiguration`, and `@ComponentScan`.

> **Component scanning** finds your application's components; **auto-configuration** configures framework/infrastructure beans.

> **Autowiring** resolves dependencies between beans.

> Multiple matching beans require disambiguation such as `@Primary` or `@Qualifier`.

> With a single constructor, `@Autowired` is unnecessary.

> **Auto-configuration ≠ autowiring**: auto-configuration configures infrastructure; autowiring injects dependencies between existing beans.
