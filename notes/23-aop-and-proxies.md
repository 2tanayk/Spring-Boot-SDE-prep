# 23 — AOP & Proxies

## 1. Why AOP exists

AOP (Aspect-Oriented Programming) separates **cross-cutting concerns** from business logic.

Examples of cross-cutting concerns:

- transactions
- authorization
- caching
- logging
- metrics

Without AOP, the same infrastructure logic would have to be repeated around many business methods.

## 2. Core AOP terms

- **Aspect** — the cross-cutting behavior, e.g. transaction or logging logic.
- **Pointcut** — defines which methods should be intercepted.
- **Advice** — defines what should happen when a matching method is intercepted.
- **Join point** — a point during execution where advice can apply. With Spring's proxy-based AOP, method execution is the practical case to remember.

Common advice types include `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`, and `@Around`.

`@Around` is especially important because it can execute behavior before and after the target method.

## 3. Spring AOP and proxies

Spring commonly implements AOP using a **proxy** around the target bean.

For example:

```java
@Service
public class TransferService {

    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        accountRepository.debit(fromId, amount);
        accountRepository.credit(toId, amount);
    }
}
```

The object obtained from Spring can conceptually look like:

```text
Caller
   ↓
Spring Proxy
   ↓
transaction logic
   ↓
actual TransferService
   ↓
transfer()
```

The proxy allows Spring to add behavior around the method without putting that infrastructure logic inside the business method.

For a transaction, the conceptual behavior is roughly:

```java
beginTransaction();
try {
    target.transfer(...);
    commit();
} catch (Exception e) {
    rollback();
    throw e;
}
```

This is a mental model of the interceptor behavior, not the literal generated source code.

## 4. Runtime type vs declared type

Suppose another bean receives:

```java
private final TransferService transferService;
```

The **declared/compile-time type** is `TransferService`, but the actual runtime object can be a Spring-generated proxy wrapping the real `TransferService` target.

Conceptually:

```text
transferService
      ↓
Spring Proxy
      ↓
actual TransferService
```

This is why an injected bean can appear to have a generated proxy class when inspecting its runtime class.

The key idea:

> **The reference can be declared as the target type while the runtime object is a proxy that wraps the target.**

## 5. External call vs self-invocation

This is one of the most important Spring AOP interview traps.

An external call enters through the proxy:

```text
Controller
    ↓
Spring Proxy
    ↓
TransferService
    ↓
transfer()
```

The proxy can therefore intercept the call and apply transaction/security/cache advice.

But an internal call such as:

```java
public void processTransfer(...) {
    validate();
    this.transfer();
}
```

does not go through the proxy.

The flow is effectively:

```text
Controller
    ↓
Spring Proxy
    ↓
processTransfer()
    ↓
this.transfer()
    ↓
actual TransferService directly
```

`this` refers to the **actual target object**, not the outer Spring proxy.

Therefore the proxy does not get a chance to intercept the `transfer()` call.

> **Spring proxy-based AOP only gets a chance to intercept calls that cross the proxy boundary.**

## 6. Self-invocation example with `@Transactional`

```java
@Service
public class TransferService {

    public void processTransfer(...) {
        validate();
        transfer();
    }

    @Transactional
    public void transfer(...) {
        accountRepository.debit(...);
        accountRepository.credit(...);
    }
}
```

The `transfer()` call is effectively:

```java
this.transfer();
```

so it bypasses the proxy. The `@Transactional` interceptor on that method therefore does not get invoked through the normal proxy mechanism.

## 7. Cleaner solution to self-invocation

A common clean solution is to move the separately advised operation into another bean:

```java
@Service
class PaymentService {

    private final TransferOperations transferOperations;

    public void process() {
        transferOperations.transfer();
    }
}
```

```java
@Service
class TransferOperations {

    @Transactional
    public void transfer() {
        // database work
    }
}
```

Now the call crosses a bean/proxy boundary:

```text
PaymentService
      ↓
TransferOperations proxy
      ↓
@Transactional advice
      ↓
transfer()
```

This is generally cleaner than designing business code around manually obtaining or calling the proxy.

## 8. JDK vs class-based proxies

For interview purposes:

- **JDK dynamic proxy** — interface-based proxy.
- **Class-based/CGLIB-style proxy** — subclass-based proxy.

The exact proxy mechanism can depend on Spring configuration and the bean being proxied. Do not spend time on bytecode-generation internals unless specifically asked.

## 9. AOP beyond transactions

The same proxy/interception idea is relevant to several Spring features:

```text
Spring Proxy
     │
     ├── transactions
     ├── security / authorization
     ├── caching
     └── other interceptors/aspects
```

For example, a method with:

```java
@PreAuthorize("hasRole('ADMIN')")
```

can be intercepted before the target method executes, while a cached method can be intercepted to check the cache before invoking the target.

## Interview Quick Recall

> **AOP** separates cross-cutting concerns such as transactions, authorization, caching, logging, and metrics from business logic.

> Spring commonly implements AOP using **proxies**.

> The injected reference can be declared as the target type while the runtime object is a proxy wrapping the actual target bean.

> External call: `Caller → Proxy → Target` — advice can run.

> Self-invocation: `Target → this.method()` — proxy is bypassed, so proxy-based advice does not get a chance to intercept that internal call.

> `@Around` advice can execute logic before and after the target and uses `proceed()` to continue to the target method.

> **JDK proxy = interface-based; class-based/CGLIB-style proxy = subclass-based.**

> The key mental model: **Spring proxy-based AOP only intercepts calls that cross the proxy boundary.**
