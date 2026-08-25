---
layout: default
title: "Spring Interview — IoC Container, Beans & Dependency Injection"
---

# Spring Interview — IoC Container, Beans & Dependency Injection

This set covers the core of the Spring Framework and Spring Boot: the IoC
container, what a bean actually is, how dependency injection and
autowiring resolve at runtime, the full bean lifecycle, how Boot's
autoconfiguration decides what to wire up, and the proxy machinery
(AOP/`@Transactional`) that sits underneath a lot of "magic" behavior.
Spring MVC, Spring Data, and Spring Security are intentionally out of scope
here — they get their own future sets.

### Q1. What is Spring's IoC container, and what problem does it actually solve? {#q1}

**Question:**
What is Spring's IoC (Inversion of Control) container, and what problem does it actually solve?

**Good answer:**
The IoC container (an `ApplicationContext` in practice) is the runtime component that reads configuration metadata — annotations, Java `@Configuration` classes, or historically XML — and uses it to instantiate, configure, wire together, and manage the complete lifecycle of the objects in your application (called beans). The "inversion" is that instead of your code calling `new` and wiring its own collaborators, the container constructs objects and *pushes* their dependencies in. This decouples components from the concrete construction/wiring logic of their dependencies, which is what makes it practical to swap implementations, mock dependencies in tests, and manage cross-cutting concerns (transactions, security, lifecycle) declaratively instead of by hand in every class.

**Follow-up question:**
What's the difference between `BeanFactory` and `ApplicationContext`, and why do virtually all Spring applications use `ApplicationContext`?

**Follow-up good answer:**
`BeanFactory` is the root, minimal container interface — basic bean instantiation and wiring. `ApplicationContext` is a superset built on top of `BeanFactory` that adds enterprise features virtually every real application needs: automatic registration of `BeanPostProcessor`/`BeanFactoryPostProcessor`, easy access to `MessageSource` for i18n, event publication (`ApplicationEvent`), and integration with Spring's AOP for things like `@Transactional`. In practice `BeanFactory` is used internally/for very lightweight scenarios; application code targets `ApplicationContext`.

**Glossary:**
- **IoC (Inversion of Control)** — the general principle that a framework, not your code, controls object creation and flow.
- **ApplicationContext** — Spring's primary, feature-rich IoC container interface.

**Mental model:**
Tests whether the candidate can explain "the container" as more than a buzzword — do they understand it's specifically about *who* constructs and wires objects, and why that inversion has practical payoff (testability, decoupling), not just "Spring does dependency injection for you."

**TL;DR:**
The IoC container (`ApplicationContext`) constructs and wires your objects from configuration metadata instead of your code doing it with `new`, decoupling components from their dependencies' construction logic.

**References:**
- [Introduction to the Spring IoC Container and Beans](https://docs.spring.io/spring-framework/reference/core/beans/introduction.html)

---

### Q2. What is a Spring bean, precisely? {#q2}

**Question:**
What is a "bean" in Spring, precisely — is it just any Java object?

**Good answer:**
A bean is specifically an object that is instantiated, assembled, and otherwise managed by a Spring IoC container. It's not "any object" — a `new Foo()` you create yourself inside a method is not a bean; it becomes a bean only when the container owns its creation, based on a `BeanDefinition`. That `BeanDefinition` carries the metadata the container needs: the fully-qualified class name, scope, lifecycle callback references (init/destroy methods), and its dependencies — regardless of whether that metadata came from an `@Component`-annotated class picked up by component scanning, a `@Bean` method in a `@Configuration` class, or (historically) an XML `<bean>` element.

**Code example:**
```java
@Component
public class OrderService {
    // Once this class is picked up by component scanning,
    // the container creates and owns this instance — it's now a "bean".
}
```

**Follow-up question:**
Can two different beans in the same container be instances of the exact same class?

**Follow-up good answer:**
Yes — nothing prevents registering multiple bean definitions with the same class, each under a different bean name (e.g. via multiple `@Bean` methods returning the same type, or `@Component` classes extended for different named instances). The container tracks beans by name/id, not by class identity, and each bean definition is independent, so you could have `primaryDataSource` and `secondaryDataSource` beans that are both plain `DataSource` instances, configured differently.

**Glossary:**
- **BeanDefinition** — the container's internal metadata object describing how to create and configure a bean.
- **Component scanning** — the mechanism that discovers `@Component`-annotated (and stereotype-annotated) classes on the classpath and registers them as bean definitions automatically.

**Mental model:**
Checks whether the candidate conflates "object" with "bean" — a common shallow answer. The correct framing is about *container ownership*, which sets up later questions about lifecycle and scope correctly.

**TL;DR:**
A bean is an object whose creation, wiring, and lifecycle are owned by the Spring container, described internally by a `BeanDefinition` — not just any object you instantiate yourself.

**References:**
- [Introduction to the Spring IoC Container and Beans](https://docs.spring.io/spring-framework/reference/core/beans/introduction.html)

---

### Q3. Why does the Spring team recommend constructor injection over field or setter injection? {#q3}

**Question:**
Constructor, field, and setter injection all work in Spring. Why does the Spring team specifically recommend constructor injection?

**Good answer:**
Per the official documentation, constructor injection lets you implement components as immutable objects (fields can be `final`) and guarantees required dependencies are never `null` — the object literally cannot be constructed in an incomplete state, so it's always handed to calling code fully initialized. Setter injection, by contrast, is best reserved for genuinely optional dependencies that can have a sensible default, because otherwise you need null-checks scattered everywhere the field is used. Field injection (`@Autowired` directly on a field) is convenient but has none of these guarantees, hides the dependency list from the constructor signature, and makes plain-Java unit testing harder since you can't construct the object with mocks without reflection or a Spring test context. A large constructor parameter list is also a useful design smell — the framework's docs explicitly call it out as a sign the class has too many responsibilities.

**Code example:**
```java
@Service
public class OrderService {
    private final PaymentClient paymentClient; // final: guaranteed non-null once constructed

    public OrderService(PaymentClient paymentClient) {
        this.paymentClient = paymentClient;
    }
}
```

**Follow-up question:**
If constructor injection is preferred, when (if ever) is field injection actually acceptable?

**Follow-up good answer:**
Mainly in test code — injecting mocks into `@InjectMocks`-style test fixtures where you don't care about immutability or testability of the test class itself — and in some legacy/framework-constrained contexts where you can't control object construction (e.g. certain JPA entity or servlet lifecycle classes the container doesn't fully own). For ordinary application beans, there's essentially no production-code justification for field injection over constructor injection today, and most style guides / static analysis rules flag it.

**Glossary:**
- **Constructor injection** — dependencies passed as constructor arguments; enables `final` fields.
- **Field injection** — `@Autowired` applied directly to an instance field, bypassing the constructor.

**Mental model:**
Separates candidates who've memorized "constructor injection is best practice" from those who understand *why* — immutability, guaranteed initialization, and testability — and can identify the narrow legitimate exceptions.

**TL;DR:**
Constructor injection guarantees immutable, fully-initialized, null-safe dependencies and is easiest to unit-test without a container; setter injection suits optional dependencies; field injection has none of these benefits and is mostly a test-code convenience today.

**References:**
- [Dependency Injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html)

---

### Q4. Walk through the full lifecycle of a singleton bean, from definition to destruction. {#q4}

**Question:**
Walk through the full lifecycle of a singleton bean, from container startup to shutdown.

**Good answer:**
In order: (1) the bean is instantiated from its `BeanDefinition`; (2) its properties/dependencies are populated (constructor args resolved and injected, or setters called); (3) `Aware` callbacks run if implemented — `BeanNameAware.setBeanName()`, `BeanFactoryAware.setBeanFactory()`, `ApplicationContextAware.setApplicationContext()`, and others; (4) any registered `BeanPostProcessor.postProcessBeforeInitialization()` runs on the raw instance; (5) initialization callbacks run, in this order if multiple are present: `@PostConstruct` methods, then `InitializingBean.afterPropertiesSet()`, then a custom `init-method`; (6) `BeanPostProcessor.postProcessAfterInitialization()` runs — this is the hook where AOP proxies actually get created and returned in place of the raw bean, so initialization callbacks in step 5 run on the *unproxied* target, not the final proxy; (7) the bean is ready for use; on container shutdown, `@PreDestroy`, then `DisposableBean.destroy()`, then a custom `destroy-method` run, in that order.

**Follow-up question:**
Why does it matter, practically, that AOP proxies are created in `postProcessAfterInitialization` — after `@PostConstruct` already ran?

**Follow-up good answer:**
It means self-invocation from inside an `@PostConstruct` method (or any code path during initialization) calls the *unproxied* target object, so any AOP advice — most commonly `@Transactional` or `@Cacheable` — will silently not apply to calls made during that initialization method, even if the method being called is itself annotated. This is the same class of bug as ordinary self-invocation bypassing the proxy, just surfacing specifically at startup, and it's a real source of "why isn't my transaction working" bugs when logic is triggered from `@PostConstruct`.

**Glossary:**
- **BeanPostProcessor** — a container extension hook that can inspect/modify every bean instance before and after its init callbacks.
- **Aware interfaces** — marker interfaces (`BeanNameAware`, `ApplicationContextAware`, etc.) that let a bean receive container infrastructure objects via a setter callback.

**Mental model:**
Tests precise, ordered knowledge of the lifecycle rather than a vague "it gets created and Spring calls some methods" — and whether the candidate can connect that ordering to a real, non-obvious bug (proxying happening after init callbacks).

**TL;DR:**
Instantiation → dependency injection → Aware callbacks → `BeanPostProcessor` before-init → `@PostConstruct`/`InitializingBean`/init-method → `BeanPostProcessor` after-init (where AOP proxies get created) → ready → `@PreDestroy`/`DisposableBean`/destroy-method on shutdown.

**References:**
- [Customizing the Nature of a Bean](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html)

---

### Q5. How does Spring decide which bean to inject when multiple candidates of the same type exist? {#q5}

**Question:**
If there are three beans implementing the same interface, how does Spring's `@Autowired` resolve which one to inject?

**Good answer:**
Spring first narrows candidates by type. If more than one bean matches, it looks for `@Primary` (a bean explicitly marked as the default choice among its type). If a `@Qualifier` is present at the injection point, that further narrows the type-matched candidates by the qualifier value — qualifiers always have narrowing semantics within the type-matched set. If none of that disambiguates, Spring falls back to matching the injection point's field or parameter name against bean names (since Spring 6.1, name-based parameter matching requires the `-parameters` javac flag to be present, since parameter names aren't retained by default otherwise). If it still can't resolve to exactly one bean, it throws `NoUniqueBeanDefinitionException`.

**Code example:**
```java
@Autowired
@Qualifier("main")
private MovieCatalog movieCatalog; // narrows to the bean qualified "main"
```

**Follow-up question:**
What's the actual difference in intent between `@Primary` and `@Qualifier`, since both can resolve ambiguity?

**Follow-up good answer:**
`@Primary` declares a single "default" bean for a type globally — useful when one implementation should win in the common case, with no per-injection-site opinion needed. `@Qualifier` is explicit, per-injection-site selection — useful when different call sites genuinely need *different* implementations of the same type, not just a fallback default. Mixing them is fine: `@Qualifier` at an injection point overrides what `@Primary` would otherwise select, since qualifiers have narrowing semantics regardless of primary status.

**Glossary:**
- **@Primary** — marks one bean as the default when multiple candidates of a type exist.
- **@Qualifier** — an annotation used to explicitly select among multiple beans of the same type at an injection point.

**Mental model:**
Probes whether the candidate actually knows the resolution *order* (type → primary → qualifier → name) rather than just "you use `@Qualifier` for that," which misses how `@Primary` and name-fallback also participate.

**TL;DR:**
Spring narrows by type, then `@Primary`, then `@Qualifier`, then bean-name-matching as a last resort, throwing `NoUniqueBeanDefinitionException` if still ambiguous.

**References:**
- [Fine-tuning Annotation-based Autowiring with Qualifiers](https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/autowired-qualifiers.html)

---

### Q6. How does Spring Boot's autoconfiguration actually decide which beans to create? {#q6}

**Question:**
Spring Boot seems to "magically" configure a `DataSource`, a `DispatcherServlet`, etc. based on what's on your classpath. How does that actually work?

**Good answer:**
`@SpringBootApplication` includes `@EnableAutoConfiguration`, which tells Spring Boot to look up a list of autoconfiguration classes — today located via `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (the older mechanism, still supported for backward compatibility, was `META-INF/spring.factories`). Each autoconfiguration class is itself a normal `@Configuration` class, but guarded by conditional annotations: `@ConditionalOnClass` only activates the configuration if a given class is present on the classpath (e.g. `DataSourceAutoConfiguration` requires `DataSource` to be on the classpath), and `@ConditionalOnMissingBean` means the autoconfigured bean backs off entirely if you've already defined your own bean of that type. This is why defining your own `DataSource` `@Bean` silently disables Boot's embedded-database autoconfiguration — it's non-invasive by design, never overriding an explicit user bean.

**Code example:**
```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
public class DataSourceAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource() { /* ... */ }
}
```

**Follow-up question:**
How would you actually debug why an autoconfiguration class you expected to run didn't, or one you didn't expect did?

**Follow-up good answer:**
Run the application with `--debug` (or `debug=true` in properties), which enables Spring Boot's auto-configuration report — it logs a "positive matches" and "negative matches" list showing exactly which conditions each autoconfiguration class evaluated and why it was applied or skipped. You can also explicitly exclude a class via `@SpringBootApplication(exclude = {SomeAutoConfiguration.class})` or the `spring.autoconfigure.exclude` property, which is useful both for debugging and for deliberately opting out.

**Glossary:**
- **@ConditionalOnClass** — activates a configuration only if a given class is present on the classpath.
- **@ConditionalOnMissingBean** — activates a `@Bean` method only if no bean of that type is already registered.

**Mental model:**
Distinguishes candidates who've only used Spring Boot from those who understand the mechanism well enough to debug it when the "magic" doesn't do what they expect — a very common real-world need.

**TL;DR:**
`@EnableAutoConfiguration` loads a list of conditional `@Configuration` classes gated by `@ConditionalOnClass`/`@ConditionalOnMissingBean`, so autoconfiguration only fires when the relevant classpath dependency is present and you haven't already defined the bean yourself.

**References:**
- [Auto-configuration](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)

---

### Q7. How does Spring AOP create proxies, and what's the difference between JDK dynamic proxies and CGLIB? {#q7}

**Question:**
How does Spring AOP actually apply advice like `@Transactional` to a bean, and what's the difference between JDK dynamic proxies and CGLIB proxies?

**Good answer:**
Spring AOP works by wrapping the target bean in a proxy at container startup (specifically during `BeanPostProcessor.postProcessAfterInitialization`). If the target class implements at least one interface, Spring defaults to a JDK dynamic proxy — a lightweight, JDK-built-in proxy that implements the same interface(s) as the target and delegates calls through the advice chain. If the target class implements no interfaces, Spring falls back to CGLIB, which generates a real subclass of the target class at runtime to intercept calls. CGLIB has real limitations: it cannot proxy `final` classes or methods (can't subclass/override them), and it cannot advise `private` methods, since a subclass can't override those either. You can also force CGLIB even when interfaces exist via `proxyTargetClass = true` on `@EnableAspectJAutoProxy`/`@EnableTransactionManagement`.

**Code example:**
```java
@EnableTransactionManagement(proxyTargetClass = true) // force CGLIB (class-based) proxies
```

**Follow-up question:**
Why doesn't `@Transactional` apply when a method calls another `@Transactional` method on `this` within the same class?

**Follow-up good answer:**
Because the proxy sits *outside* the target object — external callers invoke methods on the proxy, which applies advice and then delegates to the real target. But once execution is already inside the target object, `this` refers to the raw, unproxied instance, so a self-invocation (`this.otherMethod()`) never passes back through the proxy and the advice (transaction demarcation, caching, etc.) is simply skipped. Fixes include refactoring to avoid self-invocation (e.g. splitting into two beans), injecting a self-reference bean and calling through that, or using `AopContext.currentProxy()` with `exposeProxy=true` (discouraged, couples the code to Spring AOP). AspectJ compile-time/load-time weaving avoids the problem entirely because it modifies bytecode directly rather than proxying.

**Glossary:**
- **JDK dynamic proxy** — a proxy implementing the target's interfaces, built into the JDK.
- **CGLIB proxy** — a runtime-generated subclass of the target class, used when no interface is available.

**Mental model:**
This is one of the most common "gotcha" interview questions for a reason — it tests whether the candidate understands Spring AOP is proxy-based (not bytecode weaving by default), which has direct, practical consequences for self-invocation.

**TL;DR:**
Spring AOP proxies the target bean (JDK dynamic proxy if it implements an interface, CGLIB subclass otherwise); because the proxy wraps the object from the outside, calling an advised method via `this` from inside the same class bypasses the proxy and the advice never runs.

**References:**
- [Proxying Mechanisms](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)

---

### Q8. What are Spring's bean scopes, and what problem do scoped proxies solve? {#q8}

**Question:**
What bean scopes does Spring support, and what problem does a "scoped proxy" solve?

**Good answer:**
Spring supports six scopes: `singleton` (default — one instance per container), `prototype` (a new instance every time the bean is requested), and, in a web-aware context, `request`, `session`, `application`, and `websocket`, each tied to the corresponding web lifecycle. The tricky case is injecting a narrower-scoped bean into a wider-scoped one — e.g. a `prototype` bean into a `singleton`. Because dependencies are resolved once, at the singleton's construction time, a plain injection would hand the singleton exactly one prototype instance forever, defeating the point of prototype scope entirely. A scoped proxy (`@Scope(proxyMode = ScopedProxyMode.TARGET_CLASS)` or `<aop:scoped-proxy/>`) solves this by injecting a proxy that looks like the target type but, on every method call, fetches a fresh instance from the actual scope and delegates to it — so the singleton effectively gets a new prototype instance per call without knowing it's talking to a proxy.

**Code example:**
```java
@Component
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserPreferences { /* ... */ }
```

**Follow-up question:**
Besides a scoped proxy, what's another way to get a fresh prototype instance from within a singleton, and how does it compare?

**Follow-up good answer:**
Inject an `ObjectProvider<T>` (Spring's enhanced `ObjectFactory<T>`, or the standard JSR-330 `Provider<T>`) instead of the bean type directly, and call `.getObject()` (or `.get()`) whenever you need a fresh instance. This is more explicit about intent at the call site — the code visibly asks for a new instance each time — and avoids proxy overhead and CGLIB's limitations (can't intercept `final`/`private` methods). The trade-off is the calling code is now aware it's dealing with a factory rather than treating the dependency transparently like any other injected collaborator.

**Glossary:**
- **Scoped proxy** — a proxy substituted for a narrower-scoped bean so a wider-scoped bean can hold a stable reference that transparently resolves to a fresh instance per use.
- **ObjectProvider** — Spring's extended `ObjectFactory` for programmatically requesting bean instances, including prototypes, on demand.

**Mental model:**
Checks understanding that dependency resolution timing (once, at construction) is what creates the mismatch between singleton and prototype scopes — not just rote memorization of scope names.

**TL;DR:**
Singleton/prototype/request/session/application/websocket are Spring's bean scopes; a scoped proxy lets a singleton hold a reference to a narrower-scoped (e.g. prototype) bean that transparently fetches a fresh instance on every call instead of freezing on the one instance resolved at construction time.

**References:**
- [Bean Scopes](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html)

---

### Q9. Your Spring Boot app's startup time doubled after adding a new module. How do you diagnose why? {#q9}

**Question:**
Your Spring Boot application's startup time doubled after adding a new internal module. How do you find out why?

**Good answer:**
Start with `--debug` to get the autoconfiguration report — it's common for a newly added dependency to pull in autoconfiguration you didn't intend (e.g. an embedded database, a full Actuator stack, or a needless `DataSource`) which adds real initialization cost. Beyond that, Spring Boot Actuator exposes a `startup` endpoint (backed by `ApplicationStartup`/`BufferingApplicationStartup`) that records timed steps of the startup sequence, letting you see exactly which bean instantiations or autoconfiguration phases are slow. Component scanning breadth also matters — an overly broad `@ComponentScan` base package forces the container to inspect far more classes than necessary. And for a quick before/after comparison, running with `-Dspring.main.lazy-initialization=true` defers bean creation until first use, which — if it dramatically shortens startup — points at bean instantiation cost (heavy `@PostConstruct` work, blocking I/O in a constructor, etc.) rather than classpath scanning as the bottleneck.

**Follow-up question:**
Why is `lazy-initialization=true` a good diagnostic tool but a risky thing to leave enabled in production?

**Follow-up good answer:**
As a diagnostic, it isolates "cost of creating beans eagerly at startup" from other Boot startup overhead, cheaply. But in production it defers failures: a misconfigured bean (bad DB credentials, missing external service) that would normally fail fast at startup will now only fail the first time a request actually needs it, turning a clear startup-time error into a confusing runtime error under live traffic. It also removes Boot's "readiness" guarantee — a lazily-initialized app can report itself as up and accepting traffic before critical beans have actually been constructed and validated.

**Glossary:**
- **ApplicationStartup** — Spring's SPI for recording timed steps during application startup, exposed via Actuator's `startup` endpoint.
- **Lazy initialization** — deferring bean creation until first use instead of eagerly at container startup.

**Mental model:**
This is the trending "performance diagnosis" style question applied to Spring specifically — testing whether the candidate has an actual methodology (autoconfig report, startup tracing, lazy-init as an isolation technique) rather than guessing.

**TL;DR:**
Diagnose slow Boot startup with `--debug`'s autoconfiguration report (unexpected autoconfig pulled in), Actuator's `ApplicationStartup`/startup endpoint for timed steps, and `spring.main.lazy-initialization=true` as a quick way to isolate eager-bean-creation cost — but don't leave lazy-init on in production.

**References:**
- [Auto-configuration](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)

---

### Q10. You get "circular reference" errors on startup. What's happening and how do you fix it? {#q10}

**Question:**
Your app fails to start with a `BeanCurrentlyInCreationException` about an unresolvable circular reference. What's going on, and how do you fix it?

**Good answer:**
Two (or more) beans depend on each other, and the container can't finish constructing either one first. With setter/field injection Spring can sometimes resolve this via an early-reference cache — exposing a not-yet-fully-initialized bean reference so the other side can grab it — but with constructor injection there's no way to hand out a reference before the constructor call completes, so it fails outright. Since Spring Boot 2.6, circular references are disallowed by default even where the early-reference trick would have worked, specifically to stop this from silently masking bad design — you now have to opt back into the old behavior via `spring.main.allow-circular-references=true` if you're not ready to fix it immediately. The real fix is almost always to break the cycle: introduce a third bean/interface that both depend on instead of depending on each other directly, use constructor injection for one side and `@Lazy` setter/field injection for the other (deferring resolution), or refactor the shared logic out of both classes.

**Follow-up question:**
Why did the Spring team consider this worth breaking backward compatibility for in 2.6, given `allow-circular-references=true` restores the old behavior anyway?

**Follow-up good answer:**
A resolvable circular dependency isn't free even when it "works" — it usually means two components are more tightly coupled than they should be, and depending on Spring's early-reference mechanism to paper over that hides an architectural smell rather than fixing it. Making it opt-in forces developers to consciously decide "yes, I understand this is circular and I'm choosing not to fix it right now" instead of it happening invisibly, which is consistent with Boot's general philosophy of surfacing configuration problems early and loudly rather than silently working around them.

**Glossary:**
- **BeanCurrentlyInCreationException** — thrown when the container detects it cannot complete constructing a bean due to an unresolvable circular dependency.
- **@Lazy** — defers resolution of a dependency until it's first actually used, which can break certain circular-dependency deadlocks.

**Mental model:**
Tests both mechanical knowledge (why constructor-injected cycles fail harder than setter-injected ones) and awareness of a real, fairly recent breaking change that trips up developers upgrading Boot versions.

**TL;DR:**
Circular bean dependencies fail outright with constructor injection (no early reference possible) and, since Boot 2.6, are disallowed by default even for setter/field injection — fix by breaking the cycle (extract shared logic, use `@Lazy`) rather than re-enabling `allow-circular-references`.

**References:**
- [Spring Boot 2.6 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.6-Release-Notes)

---

### Q11. You get "no qualifying bean of type X" at startup. What causes this and how do you debug it? {#q11}

**Question:**
You get `NoSuchBeanDefinitionException: No qualifying bean of type 'X' available`. What causes this, and how would you debug it methodically?

**Good answer:**
This means the container has finished scanning/configuration and, when trying to satisfy an injection point for type `X`, found zero matching beans (contrast with `NoUniqueBeanDefinitionException`, which means it found *too many*). Common causes: the class isn't annotated with a stereotype annotation (`@Component`/`@Service`/`@Repository`) or isn't declared via a `@Bean` method at all; it exists but lives outside the package tree covered by component scanning (`@ComponentScan`/`@SpringBootApplication`'s implicit base package); the autoconfiguration that would have provided it didn't fire because a `@ConditionalOnClass` dependency is missing from the classpath; or you're injecting an interface type with no implementation registered. Debug by checking exactly where the target class lives relative to your `@SpringBootApplication` class (component scanning is package-tree-relative by default), and — if you expected autoconfiguration to supply it — rerun with `--debug` to see whether that autoconfiguration class was excluded by a failed condition.

**Follow-up question:**
Why is `@ComponentScan`'s default base-package behavior a common source of this exact error in larger, multi-module projects?

**Follow-up good answer:**
`@SpringBootApplication` implicitly scans the package of the class it's on and everything below it. In a multi-module project, if your main application class lives in `com.example.app` but a module's components live in `com.example.shared`, those beans are simply invisible to component scanning by default — no error at the point they're skipped, just a `NoSuchBeanDefinitionException` later at the injection point that needed one of them, which can be confusing because nothing about the missing-bean error itself points at "wrong package" as the cause.

**Glossary:**
- **NoSuchBeanDefinitionException** — thrown when zero beans match a required injection point.
- **Component scanning base package** — the package (and sub-packages) Spring scans for stereotype-annotated classes, defaulting to the main application class's package.

**Mental model:**
A very common real-world debugging scenario — checks whether the candidate has an actual triage order (annotation present? in scan path? autoconfiguration condition failed?) rather than "I'd just Google the error."

**TL;DR:**
`NoSuchBeanDefinitionException` means zero beans matched the requested type — check for a missing stereotype annotation, a class outside the component-scan package tree (a frequent multi-module gotcha), or a `@ConditionalOnClass` autoconfiguration that silently didn't fire.

**References:**
- [Auto-configuration](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)

---

### Q12. How does Dependency Injection relate to the Dependency Inversion Principle? {#q12}

**Question:**
How does what Spring calls "Dependency Injection" relate to the (SOLID) Dependency Inversion Principle?

**Good answer:**
They're related but distinct: the Dependency Inversion Principle (DIP) is a design principle — high-level modules shouldn't depend on low-level modules, both should depend on abstractions, and abstractions shouldn't depend on details. Dependency Injection is one concrete *technique* for satisfying DIP: instead of a class constructing its own dependency (which would couple it to a concrete implementation), the dependency — typically expressed as an interface — is supplied from outside. You can follow DIP without a DI framework at all (manually passing interfaces into constructors, "poor man's DI"); Spring's contribution is automating the *wiring* of that pattern at scale so you don't hand-assemble an object graph yourself, plus adding lifecycle/scope/AOP machinery on top.

**Follow-up question:**
Give a concrete example of code that uses field-level `@Autowired` but still violates DIP.

**Follow-up good answer:**
```java
@Service
public class OrderService {
    @Autowired
    private StripePaymentClient paymentClient; // depends on a CONCRETE class, not an abstraction
}
```
Even though Spring is injecting the dependency, `OrderService` still depends directly on the concrete `StripePaymentClient` class rather than a `PaymentClient` interface — so DIP is violated even though DI (the mechanism) is in use. The two are orthogonal: using Spring's `@Autowired` doesn't automatically make a design correct if what's being injected is a concrete implementation rather than an abstraction the high-level module actually owns the contract for.

**Glossary:**
- **Dependency Inversion Principle (DIP)** — the "D" in SOLID: depend on abstractions, not concretions.
- **Dependency Injection (DI)** — a technique for supplying an object's dependencies from outside rather than having it construct them itself.

**Mental model:**
Distinguishes candidates who can articulate the SE-theory grounding behind "Spring does DI" from those who treat the two terms as interchangeable — DI is a mechanism, DIP is a design goal, and one doesn't guarantee the other.

**TL;DR:**
DIP is the design principle (depend on abstractions); DI is one implementation technique for it — using `@Autowired` doesn't automatically satisfy DIP if the injected type is a concrete class rather than an abstraction.

**References:**
- [Introduction to the Spring IoC Container and Beans](https://docs.spring.io/spring-framework/reference/core/beans/introduction.html)

---

### Q13. What real-world testing problem does dependency injection actually solve? {#q13}

**Question:**
Beyond "it's a best practice," what concrete testing problem does dependency injection solve in a real codebase?

**Good answer:**
Without DI, a class that constructs its own collaborators (e.g. `new StripePaymentClient()` inside a method) can only be unit-tested against the *real* dependency — which for something like a payment gateway, a database, or an external HTTP call means your "unit" test is actually an integration test: slow, flaky, dependent on network/credentials, and unable to easily simulate edge cases (timeouts, error responses) on demand. With DI, the dependency is supplied as an interface from outside, so a test can construct the class under test directly with a hand-written stub or a mocking-framework mock (Mockito, etc.) standing in for the collaborator — making the test fast, deterministic, and able to exercise failure paths that would be impractical to trigger against the real thing.

**Follow-up question:**
Does using Spring's `@SpringBootTest` to load the full application context for a unit test defeat this benefit?

**Follow-up good answer:**
Largely, yes, for what should be a unit test — `@SpringBootTest` boots the entire (or a large slice of the) application context, which is slow and reintroduces exactly the kind of heavyweight, integration-style test DI was meant to let you avoid for testing a single class's logic. The DI-enabled alternative is to instantiate the class under test directly in plain Java (`new OrderService(mockPaymentClient)`) with no Spring context involved at all — Spring isn't even required to get the testability benefit, since the class was already designed to accept its dependencies from outside. `@SpringBootTest` (or slice tests like `@WebMvcTest`/`@DataJpaTest`) still has real value, just for integration-level tests, not for testing a single class's business logic in isolation.

**Glossary:**
- **Mock/stub** — a test double standing in for a real dependency, allowing controlled, deterministic behavior in tests.
- **@SpringBootTest** — a test annotation that boots some or all of the Spring application context for integration-style testing.

**Mental model:**
Moves past the abstract "DI enables testing" claim to see if the candidate can articulate the specific mechanism (substitutability of collaborators) and recognize when a Spring-specific testing tool ironically undermines that same benefit.

**TL;DR:**
DI lets test code substitute a mock/stub for a real dependency at construction time, turning what would otherwise be a slow, flaky integration test (hitting a real payment gateway, database, etc.) into a fast, deterministic unit test — a benefit that plain-Java construction with mocks captures better than booting a full `@SpringBootTest` context.

**References:**
- [Dependency Injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html)

---

### Q14. Two beans depend on each other via constructor injection. What happens, and how do you resolve it? {#q14}

**Question:**
`ServiceA` takes a `ServiceB` in its constructor, and `ServiceB` takes a `ServiceA` in its constructor. What happens when the container tries to start, and how do you fix it?

**Good answer:**
The container fails to start with a `BeanCurrentlyInCreationException`. To construct `ServiceA`, it needs a fully-built `ServiceB` first; but constructing `ServiceB` needs a fully-built `ServiceA` — and because constructor injection requires the argument to exist *before* the constructor runs, there's no point at which either side can hand out a partially-built (or "early") reference the way setter injection sometimes can. The cleanest fix is a design fix: extract the shared responsibility both services need into a third component that each depends on independently, removing the cycle entirely. If that's not immediately feasible, mark one side `@Lazy` (e.g. the constructor parameter), which injects a lazily-resolving proxy instead of forcing eager construction, breaking the deadlock — though this is generally a stopgap, since the underlying tight coupling between `ServiceA` and `ServiceB` usually indicates a design problem worth addressing.

**Code example:**
```java
public class ServiceA {
    public ServiceA(@Lazy ServiceB serviceB) { /* ... */ } // defers resolution, breaking the cycle
}
```

**Follow-up question:**
Would changing both sides to field injection "fix" this? Is that a good idea?

**Follow-up good answer:**
It would likely let the app start, because field injection happens after both objects are already constructed (the early-reference mechanism can hand out a not-yet-fully-populated instance for the other side to hold onto) — though note Spring Boot 2.6+ disallows even this by default now, requiring `spring.main.allow-circular-references=true`. It's not a good idea as a real fix: it doesn't remove the underlying tight coupling between the two services, it just lets the container paper over it, and it trades a fast, clear startup-time failure for a design smell that will keep causing friction (e.g. making the two classes hard to test or reason about independently).

**Glossary:**
- **@Lazy** — defers a dependency's resolution to first use, which can be used to break certain circular-dependency deadlocks at the cost of some indirection.

**Mental model:**
A very concrete, scenario-based version of the earlier circular-dependency question — tests whether the candidate can apply the general concept to a specific two-class example and reason about trade-offs of each fix, not just "use `@Lazy`."

**TL;DR:**
Mutual constructor injection between two beans always fails with `BeanCurrentlyInCreationException` since neither can be built first; fix by removing the cycle via a shared abstraction, or as a stopgap mark one side `@Lazy` — switching to field injection can "work" but just hides the design problem.

**References:**
- [Spring Boot 2.6 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.6-Release-Notes)

---

### Q15. Why is field injection considered bad for testability even though it's the shortest to write? {#q15}

**Question:**
Field injection (`@Autowired` on a field) is the least code to write. Why is it still considered an anti-pattern, specifically from a testability angle?

**Good answer:**
A class using field injection has no public constructor that accepts its dependencies — the dependency is set by the container via reflection after construction. That means plain-Java code (including a test) cannot simply do `new OrderService(mockPaymentClient)`; there's no constructor parameter to pass a mock into. To supply a mock, the test either needs reflection-based tricks (some mocking frameworks provide this, e.g. Mockito's `@InjectMocks` field-injection support) or has to spin up a Spring test context just to get the wiring machinery involved — both are worse than plain construction. It also hides the dependency list: a reviewer or future maintainer can't see everything a class needs by reading its constructor signature; they have to scan the whole class body for `@Autowired` fields.

**Follow-up question:**
Mockito's `@InjectMocks` supports field injection directly. Does that undercut this argument?

**Follow-up good answer:**
Only partially — `@InjectMocks` does let you inject mocks into `@Autowired` fields via reflection, so it removes the *mechanical* blocker to testing. But it doesn't restore the other benefits constructor injection gives you for free: the dependency list is still not visible from a constructor signature, the object can still exist in a partially-initialized state (nothing prevents `new OrderService()` with all fields null outside the Mockito-managed path), and `final` fields — which give you a compiler-enforced immutability guarantee — aren't usable with field injection at all. So it addresses "can I still test it," not "is the design itself as safe."

**Glossary:**
- **@InjectMocks** — a Mockito annotation that injects declared mocks into the fields (or constructor/setters) of the object under test.

**Mental model:**
Pushes past the generic "constructor injection is better" answer into the specific mechanical reason (no constructor param = no easy mock injection in plain Java) and whether the candidate can evaluate a common counterargument (`@InjectMocks`) fairly rather than dismissing it.

**TL;DR:**
Field injection removes the constructor parameter a test would use to pass in a mock, forcing reflection tricks or a full Spring test context, and it hides the dependency list and forfeits `final`-field immutability — `@InjectMocks` papers over the mechanical testing problem but not the underlying design weaknesses.

**References:**
- [Dependency Injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html)

---

### Q16. You inject a prototype-scoped bean into a singleton via constructor. What actually happens at runtime? {#q16}

**Question:**
You inject a `prototype`-scoped bean directly (no scoped proxy) into a `singleton`-scoped bean's constructor. What actually happens the first, second, and hundredth time the singleton uses it?

**Good answer:**
Exactly one prototype instance is ever created and used. The container resolves the singleton's constructor dependencies once, at the moment the singleton itself is instantiated — at that point it asks for a prototype bean, gets a fresh instance, and injects it. But because the singleton is only constructed once for the lifetime of the container, it only ever asks for that dependency once, so every subsequent use — the first call, the hundredth call, forever — reuses that same single prototype instance. This silently defeats prototype scope's entire purpose (a new instance per use) without throwing any error; it just behaves like the prototype bean was actually a singleton from that injection point's perspective.

**Follow-up question:**
If you need a genuinely fresh instance per call, which fix would you reach for and why, between a scoped proxy and `ObjectProvider`?

**Follow-up good answer:**
Either works; the choice is mostly about explicitness and constraints. `ObjectProvider<PrototypeBean>` injected into the constructor, with `.getObject()` called at each use site, makes the "I need a new one each time" intent visible in the code and avoids any proxy limitations (works even if the prototype bean's class is `final`, or the fresh-instance call needs to happen inside a private method). A scoped proxy is preferable when you want the dependency to remain a plain, transparent `PrototypeBean`-typed field used exactly like any other injected collaborator, with no call-site awareness that it's actually re-resolving each time — better when you don't control every call site or want minimal code changes to existing usage.

**Glossary:**
- **Prototype scope** — a bean scope in which a new instance is created every time the bean is requested from the container.

**Mental model:**
Forces the candidate to reason precisely about *when* dependency resolution happens (construction time, once) rather than assuming injection is somehow re-evaluated per method call — a subtle but important mental model for scope-mismatch bugs.

**TL;DR:**
Injecting a prototype bean directly into a singleton's constructor resolves it exactly once at singleton-construction time, so the singleton is stuck reusing that single prototype instance forever — fix with a scoped proxy or an `ObjectProvider`/`ObjectFactory` to get a genuinely fresh instance per use.

**References:**
- [Bean Scopes](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html)

---

### Q17. What's the difference between `BeanFactoryPostProcessor` and `BeanPostProcessor`? {#q17}

**Question:**
What's the difference between `BeanFactoryPostProcessor` and `BeanPostProcessor`, and when would you write a custom one of each?

**Good answer:**
`BeanFactoryPostProcessor` runs before any regular bean is instantiated, and it operates on `BeanDefinition` metadata — the configuration, not yet an object. A canonical built-in example is `PropertySourcesPlaceholderConfigurer`, which resolves `${...}` placeholders in bean definitions before those beans are ever created. `BeanPostProcessor`, by contrast, runs per-bean, after each bean is instantiated, and operates on the actual bean instance via `postProcessBeforeInitialization`/`postProcessAfterInitialization` hooks — this is the extension point Spring's own AOP auto-proxying machinery uses to wrap beans in proxies. You'd write a custom `BeanFactoryPostProcessor` to programmatically alter bean definitions before creation (e.g. registering additional property sources or tweaking definition metadata); you'd write a custom `BeanPostProcessor` to inspect, validate, or wrap actual bean instances uniformly (logging every bean created, applying a custom proxy, enforcing a naming convention).

**Follow-up question:**
Why is calling `BeanFactory.getBean()` inside a `BeanFactoryPostProcessor` dangerous?

**Follow-up good answer:**
It forces that bean to be instantiated prematurely, before the normal container startup sequence would have created it — potentially before other `BeanFactoryPostProcessor`s have finished modifying bean definitions that bean depends on, and before Spring's bean creation ordering/caching guarantees are in their normal state. This can bypass property placeholder resolution, produce a bean that's inconsistent with what the fully-processed definition would have produced, or cause subtle ordering bugs, which is why the documentation explicitly warns against it.

**Glossary:**
- **BeanFactoryPostProcessor** — a hook that runs before bean instantiation, operating on `BeanDefinition` metadata.
- **BeanPostProcessor** — a hook that runs after each bean's instantiation, operating on the actual instance.

**Mental model:**
Checks whether the candidate can name the actual mechanical distinction (definitions vs. instances, before-all vs. per-bean) rather than a vague "one is for the factory and one is for beans," and whether they understand a specific documented pitfall.

**TL;DR:**
`BeanFactoryPostProcessor` modifies bean *definitions* before any beans are created; `BeanPostProcessor` intercepts each bean *instance* right after it's created (this is how Spring's own AOP proxying is implemented) — calling `getBean()` inside a `BeanFactoryPostProcessor` is dangerous because it forces premature, out-of-order instantiation.

**References:**
- [Customizing the Nature of a Bean](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html)

---

### Q18. What's the difference between `@Value` and `@ConfigurationProperties`, and why does Spring Boot support both? {#q18}

**Question:**
Spring supports both `@Value("${some.property}")` and `@ConfigurationProperties`. What's the actual difference, and why keep both around?

**Good answer:**
`@Value` binds a single external property to a single field or constructor parameter — simple, but with no type-safety beyond the target field's own type, no support for binding nested/hierarchical structures, and no relaxed-binding conveniences (it expects the exact key you name). `@ConfigurationProperties` binds an entire prefix of configuration (e.g. everything under `my.service.*`) onto a POJO in one step, including nested objects and lists, using Spring Boot's relaxed binding rules (`my.service.remote-address`, `MY_SERVICE_REMOTE_ADDRESS`, and `my.service.remoteAddress` all bind to the same `remoteAddress` field), and it plays well with `@Validated` for constraint validation and with IDE metadata for autocompletion. Both remain useful: `@Value` is fine for a single, simple, standalone property; `@ConfigurationProperties` is the better choice once you have a meaningfully-structured group of related settings.

**Code example:**
```java
@ConfigurationProperties("my.service")
public record MyServiceProperties(boolean enabled, String remoteAddress) {}
```

**Follow-up question:**
How do profile-specific property files interact with `@ConfigurationProperties` binding?

**Follow-up good answer:**
They don't interact any differently than with any other externalized configuration — `@ConfigurationProperties` just binds to whatever the *effective*, already-resolved `Environment` contains, and profile-specific files (`application-{profile}.properties`) are one of the sources that contribute to that effective environment. Boot loads `application.properties` as the base, then layers on `application-{profile}.properties` for each active profile (last-active-profile-wins when multiple are active), so by the time `@ConfigurationProperties` binds, it's simply reading the final merged values — it has no special profile-awareness of its own, the layering already happened upstream in property source resolution.

**Glossary:**
- **Relaxed binding** — Spring Boot's lenient property-name matching (kebab-case, camelCase, SCREAMING_SNAKE_CASE all resolve to the same property).
- **Profile-specific properties** — `application-{profile}.properties`/`.yml` files that override base configuration when a given profile is active.

**Mental model:**
Tests whether the candidate understands these as different tools for different granularities of configuration rather than treating them as interchangeable, and whether they understand profiles as a property-source-layering mechanism rather than something `@ConfigurationProperties` handles specially.

**TL;DR:**
`@Value` binds one simple property with no relaxed binding or structure support; `@ConfigurationProperties` binds a whole prefixed, potentially nested configuration block with relaxed binding and validation support — profile-specific files just layer into the same underlying `Environment` both read from.

**References:**
- [Externalized Configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html)

---

### Q19. When would you write a custom `@Conditional` annotation instead of using Spring Boot's built-in ones? {#q19}

**Question:**
Spring Boot ships `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`, and others. When would you write your own custom `@Conditional`?

**Good answer:**
When the activation logic genuinely doesn't reduce to "is a class present," "is a bean missing," or "does a property equal some value" — e.g. conditionally enabling a configuration based on the presence of a specific combination of properties evaluated together, the resolved value of a custom `Environment` abstraction, or a check against external system state resolvable only at startup (like whether a particular native library loaded successfully). You implement the `Condition` interface's `matches(ConditionContext, AnnotatedTypeMetadata)` method with your custom logic and wrap it with `@Conditional(YourCondition.class)`. In practice this is relatively rare in application code — most real needs are covered by Boot's built-in conditions — but it's exactly the mechanism Boot's own autoconfiguration conditions are built on, so understanding it demystifies how `@ConditionalOnClass` etc. work internally (they're just pre-built `Condition` implementations).

**Follow-up question:**
Is writing a custom `@Conditional` in ordinary application code (not a library) generally a good sign or a warning sign?

**Follow-up good answer:**
Usually a mild warning sign, or at least worth pausing on — it's a strong tool most commonly appropriate for *library/autoconfiguration* authors who need to make a component optionally activate based on the consuming application's environment, which they don't control. In ordinary application code, if you find yourself reaching for a custom `Condition`, it's often worth asking whether a simpler mechanism — `@Profile`, a plain `@ConditionalOnProperty`, or even just an `if` check inside a `@Bean` method returning `null`/throwing — would express the same intent more simply and be easier for the next developer to follow.

**Glossary:**
- **Condition** — the interface backing all of Spring's `@Conditional`-based annotations; implement `matches()` for custom activation logic.

**Mental model:**
Tests whether the candidate treats the more obscure/advanced parts of the framework with judgment (when is this actually warranted vs. over-engineering) rather than just knowing the feature exists.

**TL;DR:**
Write a custom `@Conditional` when activation logic can't be expressed by Boot's built-in conditions (class/bean/property presence) — most needed by library/autoconfiguration authors, and often a sign to look for a simpler mechanism when reached for in ordinary application code.

**References:**
- [Auto-configuration](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)

---

### Q20. Constructor injection vs. field injection: is this purely a style preference, or are there real correctness trade-offs? {#q20}

**Question:**
Is the constructor-vs-field-injection debate purely a style/readability preference, or are there real correctness trade-offs?

**Good answer:**
There are real correctness trade-offs, not just style. Constructor injection makes it structurally impossible to have a bean in a partially-initialized state — if a required dependency is missing, the object simply cannot be constructed, so the failure happens immediately and loudly at startup. Field injection allows an object to exist with `null` dependencies (e.g., if wiring is somehow skipped or happens out of expected order in edge cases like certain testing setups), deferring failure to whenever that field is first used — a `NullPointerException` far from the actual root cause. Constructor injection also enables `final` fields, giving the compiler (not just convention) a guarantee that the reference can't be reassigned after construction, which field injection cannot offer at all. So while readability and boilerplate are part of the conversation, "fails fast and loud with compiler-enforced immutability" versus "can silently exist half-wired" is a genuine correctness distinction, not just taste.

**Follow-up question:**
Can constructor injection also fail in a way that's hard to diagnose, or is it strictly safer in every respect?

**Follow-up good answer:**
It can still fail in confusing ways — most notably the circular-dependency case discussed earlier, where mutual constructor injection between two beans throws `BeanCurrentlyInCreationException` at startup. It's a loud, startup-time failure (better than a runtime NPE), but the error message and stack trace can be non-obvious to a developer unfamiliar with the cycle, especially in a large object graph with many intermediate beans. So "strictly safer" is fair for the null-safety/immutability dimension specifically, but constructor injection doesn't make every failure mode trivially diagnosable — it just moves them earlier and makes them loud rather than silent.

**Glossary:**
- **Fail fast** — a design principle where errors surface immediately, close to their root cause, rather than being deferred.

**Mental model:**
A synthesizing question that asks the candidate to weigh in on a topic they've likely already discussed piecemeal (Q3, Q14) and articulate the actual correctness argument — not just repeat "it's best practice" as received wisdom.

**TL;DR:**
It's a real correctness trade-off, not just style: constructor injection makes partial initialization structurally impossible and enables compiler-enforced `final` immutability, failing loudly at startup instead of allowing a silently half-wired object to fail later at an unrelated call site — though it can still produce non-obvious startup failures of its own (e.g. circular dependencies).

**References:**
- [Dependency Injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html)

---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=spring&tags=ioc-container-beans-and-dependency-injection&autostart=1" | relative_url }})
