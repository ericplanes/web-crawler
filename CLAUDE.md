# Project — Claude Instructions

## Purpose

Write production-ready code: simple to explain, structured to maintain, and tested enough to trust.
Avoid fake enterprise complexity.

## Reviewability

- Keep diffs small and easy to explain.
- Do not make unrelated refactors.
- Do not add new dependencies without asking first.
- Prefer explicit code over clever code.
- Be ready to explain why each abstraction exists.

## How to work with me

- Do NOT use compound tool calls that require my approval — one simple action at a time.
- Do NOT use complex shell commands. Commands must be simple and safe.
- While working, explain what you are doing at each step. Short sentences, not paragraphs.
- I will be the one doing git commits, not you. Never commit on my behalf.
- Do not use superpowers skills (brainstorming, writing-plans, etc.) — we need speed.

## Development principles

- **Build only what is asked.** Do not anticipate future phases.
- **TDD always.** Write the test first, then implement the minimum to make it pass.
- **Verify with tests, not app restarts.** Unit tests are faster than starting the app. Save the live curl for a final sanity check.
- **One curl per new external endpoint.** Store the JSON at `src/main/resources/json/` to understand the shape, then build DTOs from it.
- **Don't over-layer early.** Create the client + service first. Add the controller only when it's needed.

## Commands

```bash
mvn test                        # Run all tests
mvn test -pl . -Dtest=MyTest    # Run a single test class
mvn spring-boot:run             # Start the application
lsof -i :8080                   # Check if port is in use
```

## Architecture

Use feature-based packaging by default.

Group code by business feature first, then by architectural responsibility.

Example:

```text
com.interview
├── Application.java
├── config
└── <feature>
    ├── api
    ├── application
    ├── domain
    ├── client
    └── repository
```

Responsibilities:

* `api`: inbound HTTP layer. Controllers, API DTOs, and API mappers.
* `application`: use cases and orchestration services.
* `domain`: domain models and reusable domain rules.
* `client`: outbound adapter for external APIs. Client DTOs, API contracts, HTTP clients, and client mappers.
* `repository`: outbound adapter for cache/storage. DBOs, cache keys, repositories, and repository mappers.
* `config`: application-wide Spring configuration.

Dependency direction:

```text
api -> application -> domain
application -> client
application -> repository
client -> domain
repository -> domain
```

Rules:

* API layer exposes our application to callers.
* Application layer orchestrates use cases.
* Domain layer contains the model and business/domain rules.
* Client layer hides external API DTOs and maps them to domain models.
* Repository layer hides DBOs/cache details and maps them to domain models.
* Services should work with domain models, not DTOs or DBOs.
* Controllers return API DTOs, not domain models.
* Clients return domain models, not client DTOs.
* Repositories return domain models, not DBOs.
* DTOs, DBOs, and domain models must stay data-only.
* Do not create a package unless there is at least one meaningful class for it.
* Only create event/listener/publisher packages when the feature actually needs event-driven behavior.

### Naming conventions

- Use `Operations` suffix for application/use-case interfaces.
- Use `Service` suffix for application/use-case implementations.
- Avoid unnecessary `Impl` suffixes when the concrete name is already clear.
- Good: `OrderSearchOperations` -> `OrderSearchService`.
- Good: `OrderSearchApi` -> `OrderSearchController`.
- Good: `OrderApiOperations` -> `OrderApiClient`.
- Good: `OrderRepository` -> `InMemoryOrderRepository` or `OrderCacheRepository`.

Use specific names for domain components instead of calling everything a service.

Good:
- `PriceCalculator`
- `OrderEligibilityChecker`

Avoid:
- `OrderFilterService` if it only does one thing.
- `CommonUtils`
- `AppUtils`

## Services and orchestration

Application services can have two roles:

1. Use-case/orchestration services
   - Coordinate repositories, clients, and domain components.
   - Represent a full application use case.
   - Example: `OrderSearchService`.

2. Focused operation services/components
   - Perform one focused business/domain operation.
   - Prefer placing reusable domain behavior under `domain` with specific names.
   - Example: `PriceCalculator`.

Rules:

- A service can be an orchestrator when it represents one application use case.
- Do not create a separate orchestrator class just because a service has multiple dependencies.
- Create an orchestrator only when coordinating multiple independent use cases.
- Services may depend on repositories, clients, and domain components.
- A service becomes too large when it starts doing the work of its collaborators instead of delegating to them.

## Coding style

- Java 21, Spring Boot 3.4, Maven
- Always use **final** and **var** when possible
- Use **UPPER_SNAKE_CASE** for all `static final` fields (constants): `LOGGER`, `BASE_URL`, `DELIMITER`, etc.
- Prefer **records** over classes whenever the type has no mutable state — DTOs, DBOs, constant holders, value objects, etc.
- Use **BigDecimal** for all decimal fields, **Integer** for integers, **String** for strings. Create from strings, not doubles: `new BigDecimal("4.8")`, not `new BigDecimal(4.8)`. Use `compareTo(...)` for business comparisons — `equals(...)` only when scale matters.
- Use **@JsonProperty** for JSON field names, use proper descriptive names for record fields (e.g., `@JsonProperty("lng") BigDecimal longitude`)
- Names must be readable and descriptive. Class names must be simple and clear.
- Use **Lombok** `@UtilityClass` for static-only classes (mappers, utility classes). Prefer meaningful names like `DistanceCalculator` over generic `Utils`. Do not manually write `final class` + private constructor — let the annotation handle it.
- Follow **SLAP**: Single Level of Abstraction Principle. Methods are either **doers** (one focused responsibility) or **orchestrators** (call doers at the same abstraction level). Never both. Top-level methods should read like a workflow — detailed logic belongs in focused private methods or dedicated collaborators.
- Validation goes in a dedicated `validate(...)` method that throws `IllegalArgumentException` on failure. The orchestrator calls it, never inlines the checks.
- Use **Lombok** `@Builder(toBuilder = true)` on records. This enables `.toBuilder().fieldName(newValue).build()` for clean test data creation and partial copies
- **Method declarations**: keep params on one line when the signature fits comfortably (~120 chars). Split one-per-line only when it doesn't fit or when annotations force it (e.g., `@Value` on constructor params).
- **Constructor/method calls with many args** (e.g., record instantiation): one argument per line, closing parenthesis on its own line:
  ```java
  final var product = new Product(
          "p001",
          "Wireless Mouse",
          new BigDecimal("29.99"),
          "Electronics",
          true
  );
  ```
- **Streams: one operation per line**, each `.method()` on its own line. If a lambda or comparator is hard to read inline, extract it to a named method:
  ```java
  // Bad: hard to read inline
  return items.stream()
          .filter(item -> item.price().compareTo(minPrice) >= 0)
          .sorted(Comparator.comparing(Item::name))
          .toList();

  // Good: extracted to named methods
  return items.stream()
          .filter(this::meetsMinimumPrice)
          .sorted(byName())
          .toList();
  ```

## DTOs, DBOs, and domain models

- Client DTOs represent external API contracts and live under `client/dto`.
- API DTOs represent our API contract and live under `api/dto`.
- DBOs represent cache/storage shape and live under `repository/dbo`.
- Domain models live under `domain/model`.

Rules:

- External API JSON maps to client DTOs.
- Client DTOs map to domain models inside the client layer.
- Domain models are used by application services.
- Domain models map to API DTOs inside the API layer.
- Domain models map to DBOs inside the repository layer.
- Do not leak client DTOs, API DTOs, or DBOs into application services.

Mapping flow:

```text
External API JSON -> client/dto -> domain/model -> api/dto
                                      |
                                      v
                                repository/dbo
```

## Javadoc

- Add Javadoc to **API interface methods**. These are system boundaries — document what the endpoint does, its parameters, and its return value.
- Do not add Javadoc to controllers, services, repositories, or internal classes. The code should be self-explanatory there.
- Keep Javadoc concise: one summary sentence, `@param` for each parameter, `@return` for the result. No `@throws` unless the exception is part of the contract.

## Null handling

- Public methods must not return null.
- Prefer empty collections over null collections.
- Use `Optional<T>` as a return type when absence is a valid result.
- Do not use `Optional<T>` for fields or method parameters.
- Prefer plain `if (x != null)` over `Optional.ofNullable(x).ifPresent(...)` for simple null-conditional logic. `Optional` is for return types that express absence, not for inline control flow.
- Do not return `Optional<List<T>>`; return an empty list when there are no results.
- Validate required inputs early and fail fast.

## Exceptions

- Exception messages must explain what failed.
- Do not leak secrets, API keys, tokens, or sensitive data in exception messages.
- Use `IllegalArgumentException` for invalid input.
- Use `IllegalStateException` for invalid application state or invalid external responses.

## Test approach

- Always follow **TDD**: write failing tests first, then implement the minimum to make them pass.
- **Unit tests only** by default. No integration tests or @SpringBootTest unless explicitly asked.

## Test style

- Naming: `methodName_whenCondition_thenExpectedResult`
- **Always** use `@DisplayName` with a clear description on every test — non-negotiable for readability
- Structure every test body with `// given`, `// when`, `// then` comments
- Use **AssertJ** for assertions (`assertThat`, `assertThatThrownBy`)
- Use **parametrized tests** (`@ParameterizedTest`) when multiple inputs test the same logic
- Use **standalone MockMvc** for controller tests (not @WebMvcTest)
- Mock **interfaces**, never concrete classes. This keeps tests simple and avoids Mockito configuration issues.
- Use **TestConstants** only for data shared across multiple test classes. Prefer local test data when it is only used once.

## Test coverage expectations

Tests should cover:

- successful path
- validation failures
- cache hit / cache miss (when caching is used)
- external API empty/invalid response
- optional parameters present and absent
- repository save after fetch

## Configuration

- Base URL, API key, ports, and timeouts must come from configuration.
- Do not hardcode configuration values in business code.
- Inject configuration through constructors.
- `@Value` is acceptable for simple cases.
- Prefer `@ConfigurationProperties` for grouped or complex configuration.

## HTTP clients

- Put external API calls behind a dedicated **client class**, separate from the service.
- **Services** orchestrate: validation, cache/repository lookup, calling the client, persistence.
- **Client classes** own: external HTTP calls, URL paths, query params, headers, client DTOs, response validation, and mapping to domain models.
- Clients return domain models, not client DTOs.
- Do not scatter raw HTTP contract strings (`"lat"`, `"/api/v1/venues/search"`, `"X-Api-Key"`) across services.
- Use **feature-local contract classes** with nested groups (`Path`, `QueryParam`, `Header`). Not a global `Constants` class.
- Use `UriComponentsBuilder.fromUriString()` and `queryParamIfPresent()` for optional params.
- Fail explicitly on invalid/empty responses: `throw new IllegalStateException(...)`, not `Objects.requireNonNull(...)`.
- Do not return null from clients. Do not hide external API failures.
- Do not catch exceptions unless adding useful context.
- RestTemplate is fine for now. Prefer RestClient for new work when practical.

Example structure:
```
<Feature>ApiContract — owns string constants for the external API
<Feature>ApiClient   — builds HTTP request, calls API, validates response
<Feature>Service     — validates input, checks cache, calls client, saves to cache
```

## Logging

- Use a logger, not `System.out`.
- Add logs only when useful for debugging behavior.
- Do not log secrets, API keys, tokens, or sensitive data.
- Do not add noisy logs inside simple methods or tests.

## Caching

- Repository classes own cache/storage access, DBOs, cache keys, and mapping to domain models.
- Repositories return domain models, not DBOs.
- Do not use `containsKey()` + `find()` as separate calls. Use `find()` and check the result — avoids double lookups and ensures both hit/miss log paths execute.

## Cache keys

- Do not build cache keys with ad-hoc string concatenation in business logic.
- Cache key construction belongs close to the repository/cache implementation.
- Prefer a small value object when the key has formatting, rounding, defaults, or normalization rules.
- Cache key rules must be explicit and testable.
- Use named constants for scale, delimiter, and default values.
- Use `_` (underscore) as the standard cache key delimiter. It is unambiguous — no collisions with decimal points, negative signs, or common data formats.
- Use `BigDecimal.toPlainString()` for decimal values in keys.
- Discuss cache key behavior before changing it.

## Java 21 + Spring Boot 3.4 rules

- Depend on **interfaces** for injectable application boundaries. Inject `RestOperations` not `RestTemplate`. Create interfaces for services and other collaborators that need to be mocked.
- If a client is injected into a service and mocked in tests, expose it through an interface.
- Use `@Mock` on interfaces only. Never `@Mock` a concrete class.
- Use **standalone MockMvc** with `MockMvcBuilders.standaloneSetup()` for controller tests. Instantiate the controller manually with mocked interfaces.
- Use `@ExtendWith(MockitoExtension.class)` and plain `@Mock` for all tests. Do not use `@MockBean` or `@MockitoBean`.
- Use `UriComponentsBuilder.fromUriString()` for building URLs.
- The pom.xml must include surefire `--add-opens` args for Java 21 reflection. Do not remove them.
- Use `@Service` for application-layer services. Use `@Component` for domain-layer components.
- Constructor injection only, no `@Autowired`. Spring Boot 3.4 with a single constructor does implicit injection.
- Before starting the app, check whether the target port is already in use: `lsof -i :8080`
- Do not kill processes automatically.
- If a process must be killed, show me the PID and command first. I will decide.

### Controller test template

```java
@ExtendWith(MockitoExtension.class)
class MyControllerTest {
    @Mock private MyServiceOperations service;
    private MockMvc mockMvc;

    @BeforeEach
    void setUp() {
        mockMvc = MockMvcBuilders
                .standaloneSetup(new MyController(service))
                .build();
    }
}
```

### Service test template

```java
@ExtendWith(MockitoExtension.class)
class MyServiceTest {
    @Mock private MyDependencyOperations dependency;
    private MyService service;

    @BeforeEach
    void setUp() {
        service = new MyService(dependency, "http://localhost:7001", "api-key");
    }
}
```

## Git workflow

- Do not add `.gitkeep` files. Empty directories will be tracked once real classes are added.
- When creating a new Java class or test class, `git add` the file immediately.
- Do not `git add` files under `src/main/resources/json/`.

## Definition of done

Before saying the task is complete:

1. Run the relevant tests.
2. Run `mvn test` if the change affects multiple layers.
3. Confirm the app starts if configuration or wiring changed.
4. Check that no secrets, debug logs, unrelated refactors, or unused code were added.
5. Summarize what changed and which tests were run.
