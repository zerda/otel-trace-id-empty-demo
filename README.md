# otel-trace-id-empty-demo

# Bug report — `otel.javaagent.logging=application` drops `trace_id`/`span_id` from logback output

---

### Describe the bug

With `otel.javaagent.logging=application`, logback log lines render `%X{trace_id}` / `%X{span_id}` empty inside an otherwise active server-span request, when both of these hold:

1. The Spring Boot main class holds a `static final Logger` (no `log.warn()` call is required — the declaration alone triggers it); and
2. an auto-discovered `logback.xml` is on the classpath with `<root level="WARN">` as the **only** source of the WARN root level.

It appears to be a "write succeeds / read fails" signature of the `logback-mdc` read-side instrumentation: the advice on `Logger.callAppenders()` correctly stamps the current `Context` into the `VirtualField`, but `LoggingEvent.getMDCPropertyMap()` / `getMdc()` does not receive its return-advice, so the logback encoder reads the un-augmented MDC.

### Steps to reproduce

A minimal Spring Boot 3.5 app (package `com.example.demo`):

`src/main/java/com/example/demo/DemoApplication.java`
```java
@SpringBootApplication
public class DemoApplication {
    // Condition 1: a static final Logger in the main class. No log.warn() needed.
    private static final Logger log = LoggerFactory.getLogger(DemoApplication.class);
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

`src/main/java/com/example/demo/DemoController.java`
```java
@RestController
public class DemoController {
    private static final Logger log = LoggerFactory.getLogger(DemoController.class);
    @GetMapping("/")
    public String hello() {
        log.warn("IN_REQUEST_MARKER inside server span"); // emitted inside the server span
        return "hello";
    }
}
```

`src/main/resources/logback.xml`
```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] [%X{trace_id}] [%X{span_id}] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>
  <!-- Condition 2: auto-discovered logback.xml, WARN as the root level -->
  <root level="WARN"><appender-ref ref="STDOUT"/></root>
</configuration>
```

`src/main/resources/application.properties`
```properties
spring.application.name=otel-demo
server.port=8080
# IMPORTANT: do NOT set logging.level.root here (see Additional context).
```

Run with `application` mode and trigger one request:
```bash
java -javaagent:opentelemetry-javaagent.jar \
  -Dotel.service.name=otel-demo \
  -Dotel.javaagent.logging=application \
  -Dotel.traces.exporter=none -Dotel.metrics.exporter=none -Dotel.logs.exporter=none \
  -jar app.jar
curl http://localhost:8080/
```

The same binary with only `-Dotel.javaagent.logging=simple` (`baseline`) is the contrast.

Behavior is **deterministic**: with the configuration above, `application` consistently yields empty ids across repeated runs.

### Expected behavior

The in-request WARN line should carry the server span's trace/span ids (identical to `simple` mode — only `otel.javaagent.logging` differs):
```
00:35:34.388 [http-nio-8080-exec-1] [8928ce725c92d11d8492658e2d858603] [9b642137a5d95ffe] WARN  com.example.demo.DemoController - IN_REQUEST_MARKER inside server span
```

### Actual behavior

In `application` mode both trace_id and span_id render empty:
```
00:34:53.799 [http-nio-8080-exec-1] [] [] WARN  com.example.demo.DemoController - IN_REQUEST_MARKER inside server span
```

### Javaagent or library instrumentation version

v2.30.0

### Environment

**JDK**: OpenJDK 25.0.4
**OS**: macOS 15.7 (Darwin 24.6.0, arm64)
**App libraries**: Spring Boot 3.5.16 (`spring-boot-starter-web`), Logback 1.5.34, SLF4J 2.0.18
**Agent config**: `-Dotel.javaagent.logging=application`; `-Dotel.{traces,metrics,logs}.exporter=none` (trace_id is generated locally; exporters are irrelevant to MDC injection)

### Additional context

**This is an in-request, in-span log line, not a startup log.** The server span is created and the `Logger.callAppenders()` instrumentation has run (see analysis), so the span/write path is healthy — the emptiness comes from the read side.

**The `logback-mdc` instrumentation — two cooperating parts:**
- `LoggerInstrumentation`: advises `ch.qos.logback.classic.Logger.callAppenders(ILoggingEvent)` on enter and does `CONTEXT.set(event, Java8BytecodeBridge.currentContext())` — stamps the current `Context` (which holds the span) onto a `VirtualField<ILoggingEvent, Context>`. **(This part works.)**
- `LoggingEventInstrumentation`: advises `getMDCPropertyMap()` / `getMdc()` on return, for any class `implements ILoggingEvent`, reading `CONTEXT.get(event)` and injecting `trace_id`/`span_id`/`trace_flags` into the returned map. **(This is the part that renders `%X{trace_id}`; under the trigger conditions this advice is not present on `LoggingEvent`, so the logback encoder calls the un-augmented getter.)**

Since only `-Dotel.javaagent.logging` differs and the span-creation and write paths demonstrably work in `simple` mode (trace_id populated), the emptiness in `application` mode points to the read-side transform being missed.

**Analysis of the trigger conditions.** In `application` mode the agent's internal logs are buffered in an in-memory store until the app's SLF4J binding is ready, then flushed/replayed. The bridge has two mutually exclusive install triggers (in `internal-application-logger`): one on `LoggerFactory.getILoggerFactory()`, and one for Spring Boot on `LoggingApplicationListener.initialize()` (the latter exists because *"in Spring Boot, SLF4J's LoggerFactory is touched early but is not usable until LoggingApplicationListener finishes initializing"*). The Spring Boot one is normally armed by `SpringApplication`'s `<clinit>`.

When the main class has a `static final Logger`, its `<clinit>` runs `LoggerFactory.getLogger(...)` before the body of `main()` — i.e. before `SpringApplication` is loaded. The `LoggerFactory` trigger therefore fires during `<clinit>`, before Spring Boot has rerouted to the (correct) `LoggingApplicationListener` trigger. With an auto-discovered `logback.xml` on the classpath, that early `getLogger` then actually configures Logback (builds the appender, sets the WARN root level), and the replay flushes buffered agent logs into a Logback context that is not yet configured.

Evidence of this is the logback-emitted warning that appears only in `application` mode (absent in `simple`), the same root as #14889:
```
WARN in Logger[io.opentelemetry.javaagent.instrumentation.executors.ExecutorMatchers]
  - No appenders present in context [default] for logger [...].
```
(`[default]` is the default `LoggerContext` name; the message comes from `LoggerContext.noAppenderDefinedWarning(...)`, fired by `Logger.callAppenders(...)` when an event has no appender anywhere in the hierarchy — i.e. at replay time, Logback is bound but not configured.)

Our inference is that `LoggingEvent` first loads during this premature replay window, and the `logback-mdc` read-side transform on its `getMDCPropertyMap()` is not applied; later request logs then invoke the un-instrumented getter, so `%X{trace_id}` renders empty. (This last step is inferred from the symptom plus the no-appenders window; we have not yet confirmed the exact reason for the transformer skip with `-Dotel.javaagent.debug=true`, but can do so if helpful.)

**One detail essential to reproduction** (otherwise the bug is invisible with a typical Spring Boot setup): the WARN root level must come *only* from `logback.xml`. If `logging.level.root=warn` is also set in `application.properties`, Spring Boot's `LoggingApplicationListener` re-applies the root level mid-`SpringApplication.run()`, and in our testing the bug is **not** reproducible (trace_id populated). The repro above intentionally omits `logging.level.root` from `application.properties`.

Removing either of the two necessary conditions makes the bug disappear:
- remove the `static final Logger` from the main class (or the logback.xml entirely) → trace_id populated; or
- switch to `-Dotel.javaagent.logging=simple` → trace_id populated.

**Related issues:**
- #14889 – `otel.javaagent.logging=application` causes logback re-init / "No appenders present in context [default]".
- #13069 – context lost with logback async appender (related MDC-instrumentation + VirtualField path).
- #1201 – historical move from `LoggingEventWrapper` to the `VirtualField`-based MDC instrumentation.

This report is intentionally symptom- and analysis-focused; no fix is proposed here.
