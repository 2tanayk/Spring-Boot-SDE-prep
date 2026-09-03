# Spring Profiles & Configuration

## 1. Externalized Configuration

Spring Boot keeps environment-specific settings outside application code.

```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: app
    password: secret
```

Both `application.properties` and `application.yml` are supported.

### `@Value`

Useful for injecting individual properties:

```java
@Value("${app.payment.timeout}")
private int timeout;
```

### `@ConfigurationProperties`

Preferred for grouped, typed configuration:

```yaml
app:
  payment:
    timeout: 5000
    retry-count: 3
```

```java
@ConfigurationProperties(prefix = "app.payment")
public class PaymentProperties {
    private int timeout;
    private int retryCount;
}
```

It gives a structured configuration object instead of scattering individual `@Value` fields.

---

## 2. Profiles

Profiles allow different configuration and bean definitions for different environments.

Typical files:

```text
application.yml          # common/default
application-dev.yml      # development
application-test.yml     # testing
application-prod.yml     # production
```

Profile-specific configuration overrides common configuration where applicable.

### Activating a Profile

```properties
spring.profiles.active=dev
```

Or at runtime:

```bash
--spring.profiles.active=prod
```

```bash
SPRING_PROFILES_ACTIVE=prod
```

Multiple profiles can be active:

```text
prod,metrics
```

---

## 3. `@Profile`

`@Profile` conditionally creates beans based on the active profile.

```java
@Profile("dev")
@Bean
PaymentClient mockPaymentClient() { ... }

@Profile("prod")
@Bean
PaymentClient realPaymentClient() { ... }
```

This is useful when different environments need different implementations.

### Profile Groups

A logical profile can activate several profiles together:

```yaml
spring:
  profiles:
    group:
      production:
        - prod-db
        - metrics
        - monitoring
```

Activating `production` activates all three grouped profiles.

---

## 4. Configuration Precedence

Spring Boot can receive configuration from multiple sources. More specific/higher-precedence sources override lower-precedence values.

For example, a default value:

```yaml
server:
  port: 8080
```

can be overridden at runtime with:

```bash
--server.port=9090
```

This allows the same application JAR to run with different runtime configuration.

**Never commit real secrets** such as database passwords or API keys to `application.yml`. Use environment variables or a secret-management system.

---

## 5. Practical Structure

```text
src/main/resources/
├── application.yml
├── application-dev.yml
├── application-test.yml
└── application-prod.yml
```

Keep shared configuration in `application.yml` and environment-specific differences in profile files.

---

## Mental Model

- `application.yml` → common/default configuration
- `application-{profile}.yml` → profile-specific configuration
- `spring.profiles.active` → selects active profiles
- `@Profile` → conditionally creates beans
- `@Value` → injects an individual property
- `@ConfigurationProperties` → binds grouped properties into a typed object
- Environment variables → deployment-time configuration/overrides
- CLI arguments → high-precedence runtime overrides

## Interview Answer

**Spring Boot externalizes configuration so settings don't have to be hardcoded in application code. Profiles provide different configuration and bean definitions for different environments. Common settings live in `application.yml`, while `application-{profile}.yml` contains environment-specific overrides. `@Profile` conditionally enables beans, and `@ConfigurationProperties` provides type-safe binding for structured configuration.**
