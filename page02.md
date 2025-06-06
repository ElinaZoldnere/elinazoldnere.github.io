---
layout: default
title: June 6, 2025 Should Now Be Now()
description: How Java's Clock abstraction improves control over time handling and testability in Spring Boot application.
nav_order: 2
---

#  Should Now Be Now()
### Java Clock
In my previous blog entry, I shared an issue with hardcoded test dates turning into past dates,
which led to test failures at some point even without any code changes.  
That made me rethink my approach to writing tests and handling dates and time in general.

The first and most obvious conclusion was that I needed more control over the notion of "current
time" in my tests. In unit tests, this should not be an issue at all, as a proper unit test is
performed in isolation and does not depend on any functionality outside the direct test scope.  
But with integration tests, where the functionality is tested by bringing up Spring's
application context, lacking control over the time represented as now became an issue that
required a solution.

The most obvious solution for time alteration seemed libraries that intercept system time
globally via bytecode manipulation or JVM agents. Yet I wasn't quite happy with their offered
approach — time manipulation is usually achieved by intercepting calls to the system clock and
overriding them at the JVM level, without control over what exactly is overridden. It didn’t
seem like an approach I would want to incorporate into my project, even if we are talking only
about tests.

Next, I came across the concept of the **Java Clock**, which is part of the `java.time` package
introduced in Java 8, offers a wrapper around the system clock and provides factory methods to
create fixed clocks, offset clocks, and others. Although the first and most obvious benefit
seems to be the desired control over time in tests, looking deeper, it provides additional
advantages for production code as well.

> *Code that relies on a direct method call like `LocalDate.now()` without 
> arguments is always tightly coupled to the actual system clock.* 

It does not allow easy control over it, other than modifying the host system clock or relying
on less clean workarounds. The `Clock` abstraction, on the other hand, ensures full control
over what time settings are provided to LocalDate when calling `LocalDate.now(clock)` (or other
time classes like `Instant.now(clock)`). It allows performing future or past simulations (for
example in test environments) or using a time source synchronized with an external authority,
if needed.

### Example of Usage in Spring Boot Application
> *Consistency in time management within an application can be ensured by 
> defining **a single shared `Clock` instance**.* 

If multiple time zones or strategies are involved, a separate `Clock` would be provided for
each logical time source.  
Since `Clock` is an abstract class, it cannot be instantiated directly but must be created
through its static factory methods, which can be exposed via a Spring `@Bean`.

Some of the commonly used methods are `Clock.systemUTC()`, `Clock.fixed(Instant, ZoneId)`,
`Clock.offset(Clock, Duration)`.
Overview of `Clock` with examples: [Guide to the Java Clock Class (Baeldung)](https://www.baeldung.com/java-clock).

To use a shared `Clock` across an application define a custom configuration for `Clock`:
```java
@Configuration
public class TimeConfig {

    @Bean
    public Clock defaultClock() {
        return Clock.system(ZoneId.of("Europe/Riga"));
    }

}
```
Then inject wherever needed:
```java
@Component
@RequiredArgsConstructor
public class DateTimeUtil {

    private final Clock defaultClock;

    public LocalDate currentDate() { // today 00:00:00 EET (Europe/Riga)
        return LocalDate.now(defaultClock);
    }

}
```
In tests instead of using a simple fixed clock I chose to simulate a dynamic, naturally ticking
clock always remaining within a specified year regardless of an actual date.  
```java
@TestConfiguration
public class TestTimeConfig {
    
    @Bean
    @Primary
    public Clock testClock() {
        ZoneId zone = ZoneId.of("Europe/Riga");
        LocalDate today = LocalDate.now(Clock.system(zone));
        LocalDate simulatedDate = today.withYear(2024);

        long daysOffset = ChronoUnit.DAYS.between(today, simulatedDate);
        Duration offset = Duration.ofDays(daysOffset);

        // Provides a ticking Clock shifted to the same day and month in the year 2024.
        return Clock.offset(Clock.system(zone), offset);
    }

}
```
Since `@TestConfiguration` classes are not picked up by component scanning, the test
configuration must be explicitly imported into the test context. To ensure that test `Clock`
bean is picked for injecting in the application context, it is marked with `@Primary`.
```java
@SpringBootTest
@AutoConfigureMockMvc
@Import(TestTimeConfig.class)
class ValidateAgreementDateIntegrationTest {
    ...

}
```

To sum up, I would probably avoid calling the `Clock` abstraction an absolute must-have for
every project, especially when relying on the host system clock is sufficient and the
application logic does not require extensive testing of time-dependent behavior.

That said, I still see `Clock` as a good practice to decouple application logic from the system
clock, improving design, offering flexibility and much better testability, when that matters.
