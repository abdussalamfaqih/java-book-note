# Microservices with Spring Boot and Spring Cloud

Notes on *Microservices with Spring Boot 3 and Spring Cloud* (Magnus Larsson, Packt, 3rd edition). A hands-on book that builds a small system of cooperating microservices and adds, chapter by chapter, the pieces a real landscape needs: Docker, OpenAPI, persistence, reactive code, service discovery, an edge server, security, config, resilience, tracing, Kubernetes, a service mesh, logging and monitoring.

Notes are in my own words and written from memory, so chapter titles and details should be checked against the book. Code is a first draft and has not been run.

## The example system (used in every chapter)

- `product-service`: product info by `productId`
- `recommendation-service`: recommendations for a product
- `review-service`: reviews for a product
- `product-composite-service`: aggregates the three above into one response (the public API)

```
client -> product-composite -> product, recommendation, review
```
Core services are independent (own database each); the composite calls them over REST or messaging.

## Chapters

| Part | Ch | Title | Notes |
|------|----|-------|-------|
| Basics | 1 | Introduction to microservices | todo |
| | 2 | Introduction to Spring Boot | todo |
| | 3 | Creating a set of cooperating microservices | todo |
| | 4 | Deploying with Docker | todo |
| | 5 | Adding an API description using OpenAPI | todo |
| | 6 | Adding persistence | todo |
| | 7 | Developing reactive microservices | todo |
| Spring Cloud | 8 | Introduction to Spring Cloud | todo |
| | 9 | Service discovery with Netflix Eureka | todo |
| | 10 | Spring Cloud Gateway as an edge server | todo |
| | 11 | Securing access to APIs | todo |
| | 12 | Centralized configuration | todo |
| | 13 | Improving resilience with Resilience4j | todo |
| | 14 | Understanding distributed tracing | todo |
| Kubernetes | 15 | Introduction to Kubernetes | todo |
| | 16 | Deploying microservices to Kubernetes | todo |
| | 17 | Kubernetes features to simplify the landscape | todo |
| | 18 | Service mesh for observability and management (Istio) | todo |
| Operations | 19 | Centralized logging with the EFK stack | todo |
| | 20 | Monitoring microservices | todo |

(Later chapters/appendices cover installing tools on macOS and Windows.)

---

## Part 1: Basics

### Chapter 1: Introduction to microservices

**Explanation**
- A microservice is an independently deployable component, built around a business capability, with its own data and a well-defined API. Many of them together form a system landscape.
- Benefits: independent deployment and scaling, small teams owning a service end to end, technology freedom, easier to replace or rewrite one part, resilience against partial failure.
- Challenges: distributed systems are hard (network failures, latency, partial failure), many things to operate, testing across services, data consistency without shared transactions, and needing automation for everything.
- Compared with a monolith, you trade in-process simplicity for deployment flexibility. Don't start with microservices if you can't manage that complexity.
- Design guidelines: services should be loosely coupled and highly cohesive, with the right size (not too fine), and own their data. Use domain-driven design (bounded contexts) to find the boundaries.
- Patterns the book will implement: service discovery, edge server, reactive services, central configuration, circuit breaker, distributed tracing, centralized logging and monitoring.
- Mentions 12-factor app ideas (config in environment, stateless processes, disposability, logs as streams).

**Takeaways**
- Microservices are a trade-off, not a goal: they solve scaling teams and deployment, and create operational complexity.
- Most of what the rest of the book does is deal with that complexity using standard patterns and tools.

---

### Chapter 2: Introduction to Spring Boot

**Explanation**
- Spring Boot gives opinionated defaults: auto-configuration based on classpath, starter dependencies, an embedded server, and Actuator endpoints. You write mostly business code.
- Spring Boot 3 requires Java 17+ and moved from `javax.*` to `jakarta.*`. It supports native images via GraalVM.
- The application entry point is a class with `@SpringBootApplication` (= `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`).
- Spring WebFlux (reactive, Netty) or Spring MVC (servlet, Tomcat) are the two web stacks. REST controllers use `@RestController` and mapping annotations.
- Springdoc/Swagger, Spring Data and Spring Security are added later as starters.
- Testing: `@SpringBootTest` for full context, `@WebFluxTest`/`@WebMvcTest` for slices, `WebTestClient`/`MockMvc`; Testcontainers is used later for real databases.
- Build is Gradle with multi-project builds (a root build and one subproject per service, plus shared `api` and `util` projects).

```java
@SpringBootApplication
public class ProductServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ProductServiceApplication.class, args);
    }
}

@RestController
class ProductController {
    @GetMapping("/product/{productId}")
    Product getProduct(@PathVariable int productId) {
        return new Product(productId, "name-" + productId, 123);
    }
}
```

**Takeaways**
- Convention over configuration: start fast, override only what you need.
- Keep services small, with their API separated from the implementation (an `api` module with interfaces and DTOs).

---

### Chapter 3: Creating a set of cooperating microservices

**Explanation**
- Builds the four services described above, with a shared `api` project (service interfaces + model classes) and a `util` project (common exceptions, error handling, `HttpErrorInfo`).
- The three core services return hard-coded data first. The composite service calls them with `RestTemplate` (blocking) via a `ProductCompositeIntegration` component, which is the single place that knows about downstream services.
- Error handling is centralized: custom exceptions (`NotFoundException`, `InvalidInputException`) are mapped to HTTP status codes (404, 422) by a global `@RestControllerAdvice`.
- The composite aggregates responses into `ProductAggregate` (product + lists of recommendations and reviews + service addresses).
- Configuration (ports, service URLs) is in `application.yml`, with Spring profiles for different environments.
- Testing: unit-style tests per service with `WebTestClient` and a composite test that mocks the integration layer; a manual `test-em-all.bash` script exercises the whole system end to end.

```java
@Component
public class ProductCompositeIntegration implements ProductService, RecommendationService, ReviewService {
    private final RestTemplate restTemplate;
    private final String productServiceUrl;

    public ProductCompositeIntegration(RestTemplate restTemplate,
            @Value("${app.product-service.host}") String host,
            @Value("${app.product-service.port}") int port) {
        this.restTemplate = restTemplate;
        this.productServiceUrl = "http://" + host + ":" + port + "/product/";
    }

    @Override public Product getProduct(int productId) {
        return restTemplate.getForObject(productServiceUrl + productId, Product.class);
    }
}
```

**Takeaways**
- Isolate calls to other services in one integration class so you can mock, change the protocol, or add resilience later.
- Synchronous calls chain failures: the composite is only as available as its dependencies (addressed in chapters 7 and 13).

---

### Chapter 4: Deploying our microservices using Docker

**Explanation**
- Docker packages each service with its runtime, so it runs identically everywhere. Containers are light and share the host kernel; images are built in layers.
- Each service gets a multi-stage `Dockerfile`: one stage extracts layers from the fat jar (layered jars), the next stage copies them into a slim JRE base image.
- Spring profile `docker` changes hostnames and ports (services refer to each other by container/service name).
- Docker Compose describes the whole landscape in `docker-compose.yml`: services, ports, environment, `depends_on`, healthchecks. One command starts everything.
- Only the composite service exposes a port to the host; core services are reachable only inside the Docker network (a basic form of protection).
- Container awareness in the JVM: set memory limits (`mem_limit`) and make sure the JVM respects them.
- Tests are run in the containers using `test-em-all.bash`, and Testcontainers is used later for integration tests.

```dockerfile
FROM eclipse-temurin:17-jre AS builder
WORKDIR extracted
ADD build/libs/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:17-jre
WORKDIR application
COPY --from=builder extracted/dependencies/ ./
COPY --from=builder extracted/spring-boot-loader/ ./
COPY --from=builder extracted/application/ ./
EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```
```yaml
# docker-compose.yml
services:
  product:
    build: microservices/product-service
    mem_limit: 512m
    environment: [SPRING_PROFILES_ACTIVE=docker]
  product-composite:
    build: microservices/product-composite-service
    ports: ["8080:8080"]
    depends_on: [product]
```

**Takeaways**
- Layered images make rebuilds faster since dependencies rarely change.
- Don't expose internal services; publish only the entry point.

---

### Chapter 5: Adding an API description using OpenAPI

**Explanation**
- OpenAPI (formerly Swagger) describes a REST API in a machine-readable format, which gives documentation, client generation and a try-it-out UI.
- The book uses the code-first style with `springdoc-openapi`: Java interfaces annotated with `@Operation`, `@ApiResponse`, `@Tag` produce the spec at `/v3/api-docs` and the UI at `/swagger-ui.html`.
- The API interface lives in the `api` project, with annotations on the interface rather than the controller, so the documentation stays close to the contract.
- General info (title, version, license, contact) is defined with an `OpenAPI` bean, configurable via properties.
- Only the composite service's API is public and documented in detail; core service APIs are internal.
- Swagger UI "Try it out" lets you call the running service directly from the browser.

```java
@Tag(name = "ProductComposite", description = "REST API for composite product information.")
public interface ProductCompositeService {
    @Operation(summary = "${api.product-composite.get-composite-product.description}")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "${api.responseCodes.ok.description}"),
        @ApiResponse(responseCode = "404", description = "${api.responseCodes.notFound.description}")
    })
    @GetMapping(value = "/product-composite/{productId}", produces = "application/json")
    ProductAggregate getProduct(@PathVariable int productId);
}
```

**Takeaways**
- Put docs in the API interface, text in `application.yml`/properties, so code stays readable.
- A documented API doubles as the contract between teams.

---

### Chapter 6: Adding persistence

**Explanation**
- Each microservice owns its database (database per service), so no other service touches its tables or collections.
- Product and recommendation services use MongoDB through Spring Data MongoDB; the review service uses MySQL with Spring Data JPA (relational data suits reviews).
- Entities map to DTOs with MapStruct: the API model is kept separate from the persistence model.
- Spring Data repositories need only an interface (`CrudRepository`, `ReactiveCrudRepository`) with derived query methods.
- Unique indexes enforce business keys; optimistic locking with `@Version` detects concurrent updates.
- Persistence tests use Testcontainers to run real MongoDB/MySQL in Docker, so tests match production behavior better than in-memory fakes.
- Docker Compose gets new containers for the databases, and the services' configs get connection settings per profile.

```java
@Document(collection = "products")
public class ProductEntity {
    @Id private String id;
    @Version private Integer version;
    @Indexed(unique = true) private int productId;
    private String name;
    private int weight;
}

public interface ProductRepository extends CrudRepository<ProductEntity, String> {
    Optional<ProductEntity> findByProductId(int productId);
}

@Mapper(componentModel = "spring")
public interface ProductMapper {
    Product entityToApi(ProductEntity entity);
    ProductEntity apiToEntity(Product api);
}
```

**Takeaways**
- Polyglot persistence: choose the store that fits each service.
- Test with the real database engine using Testcontainers.

---

### Chapter 7: Developing reactive microservices

**Explanation**
- Two things to decide: synchronous vs asynchronous communication, and blocking vs non-blocking code.
- Non-blocking code (Project Reactor `Mono<T>` for 0..1 items and `Flux<T>` for 0..n items) lets a small number of threads handle many requests, which is valuable when services wait on other services.
- The composite service uses `WebClient` (non-blocking) instead of `RestTemplate` and combines results with `Mono.zip`.
- Blocking libraries (JPA/JDBC) can't be made reactive, so the review service wraps blocking calls and runs them on a dedicated `Scheduler` (`publishOn`) so they don't block the event loop.
- Event-driven asynchronous communication: Spring Cloud Stream publishes `create`/`delete` events to a message broker (RabbitMQ or Kafka) and the core services consume them. This decouples services in time.
- Messaging concepts: topics, consumer groups (one instance of a group handles each message), partitions (preserve ordering per key), retries and dead-letter queues for poison messages.
- The API stays synchronous for reads; writes (create/delete) go through the broker, giving eventual consistency.
- With functional style in Spring Cloud Stream, the consumer is a `Consumer<Event<K,T>>` bean bound by naming convention to a destination.

```java
// Non-blocking aggregation in the composite service
public Mono<ProductAggregate> getProduct(int productId) {
    return Mono.zip(
        values -> createProductAggregate((Product) values[0],
                                         (List<Recommendation>) values[1],
                                         (List<Review>) values[2]),
        integration.getProduct(productId),
        integration.getRecommendations(productId).collectList(),
        integration.getReviews(productId).collectList());
}

// Messaging: consumer bean, bound to a topic via configuration
@Bean
public Consumer<Event<Integer, Product>> messageProcessor() {
    return event -> {
        switch (event.getEventType()) {
            case CREATE -> productService.createProduct(event.getData());
            case DELETE -> productService.deleteProduct(event.getKey());
        }
    };
}
```
```yaml
spring.cloud.stream:
  bindings.messageProcessor-in-0:
    destination: products
    group: productsGroup
```

**Takeaways**
- Reactive isn't free: harder to debug and write. Use it where the concurrency benefit is real.
- Asynchronous events increase resilience and scalability, but the system becomes eventually consistent.

---

## Part 2: Spring Cloud

### Chapter 8: Introduction to Spring Cloud

**Explanation**
- Spring Cloud is a set of projects that implement common distributed-system patterns on top of Spring Boot.
- Mapping from need to component: service discovery to Netflix Eureka (or Spring Cloud LoadBalancer on top); edge server to Spring Cloud Gateway; central config to Spring Cloud Config Server; resilience to Resilience4j (circuit breaker, retry, rate limiter); distributed tracing to Micrometer Tracing with Zipkin (earlier: Sleuth); messaging to Spring Cloud Stream.
- Older Netflix components (Hystrix, Ribbon, Zuul) are in maintenance or replaced; the book uses the current replacements.
- Goal: use these only where needed, and later see how Kubernetes can replace several of them (chapter 17).
- Versions: Spring Cloud release trains are matched to Spring Boot versions, so use the BOM (`spring-cloud-dependencies`) rather than individual versions.

```groovy
dependencyManagement {
    imports { mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}" }
}
```

**Takeaways**
- Pick components by the problem they solve, not because they exist.
- Always use the release train BOM to avoid version conflicts.

---

### Chapter 9: Adding service discovery using Netflix Eureka

**Explanation**
- Hard-coded host/port can't work when instances start, stop and move. A discovery server keeps a registry of live instances per service name.
- Pieces: Eureka *server* (registry), Eureka *clients* (services that register themselves and send heartbeats), and client-side load balancing (Spring Cloud LoadBalancer) when a service calls another by logical name.
- Callers use `http://product/...` and the load balancer resolves `product` to one of the registered instances.
- Eureka favors availability over consistency: clients cache the registry, and during network partitions it enters self-preservation mode and keeps stale entries rather than dropping them.
- Scaling: start more instances (`docker-compose up --scale review=2`) and they register automatically; traffic is balanced between them.
- Failure demo: stop an instance and see clients stop calling it after the heartbeat times out; there is a window where calls can fail (so retries/circuit breakers are needed).
- Config knobs: lease renewal, registry fetch intervals (shortened in dev/tests for fast reaction, keep defaults in production).
- Should be protected by security (basic auth) since the registry is sensitive.

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication { ... }
```
```yaml
# client
eureka.client.serviceUrl.defaultZone: http://eureka:8761/eureka/
```
```java
@Bean @LoadBalanced
WebClient.Builder loadBalancedWebClientBuilder() { return WebClient.builder(); }
// now: webClient.get().uri("http://product/product/" + id)
```

**Takeaways**
- Use logical service names everywhere and let discovery plus load balancing do the rest.
- Discovery data is eventually consistent, so callers must tolerate stale entries.

---

### Chapter 10: Using Spring Cloud Gateway to hide microservices behind an edge server

**Explanation**
- An edge server is the single entry point: it hides internal services, handles cross-cutting concerns (routing, TLS termination, authentication, rate limiting) in one place.
- Spring Cloud Gateway is reactive (built on WebFlux/Netty) and based on *routes* made of a predicate (when to match), filters (modify request/response) and a target URI.
- It integrates with Eureka: `lb://product-composite` routes to a discovered service with load balancing.
- Routes are declared in config (or Java DSL), including path rewriting (`RewritePath`), so the public URL needn't match the internal one.
- It can expose consolidated Swagger UI and proxy internal tooling such as the Eureka UI.
- Only the gateway has a published port; everything else lives on the internal network.
- Trace ID headers and HTTPS are configured here later.

```yaml
spring.cloud.gateway.routes:
  - id: product-composite
    uri: lb://product-composite
    predicates:
      - Path=/product-composite/**
  - id: eureka-api
    uri: http://${app.eureka-server}:8761
    predicates:
      - Path=/eureka/api/{segment}
    filters:
      - SetPath=/eureka/{segment}
```

**Takeaways**
- The edge server is the right place for common policies; it should have no business logic.
- It is the one component exposed to the internet, so it must be secured and highly available.

---

### Chapter 11: Securing access to APIs

**Explanation**
- Layers of protection: HTTPS for external traffic (certificate on the edge server), authentication of users and clients, and authorization of what they may do.
- The book uses OAuth 2.0 and OpenID Connect (OIDC). Roles: resource owner (user), client (app), authorization server (issues tokens), resource server (API).
- A local authorization server is built with Spring Authorization Server. Tokens are JWTs signed by the server; resource servers validate them with the public keys (JWK Set).
- Grant types: *authorization code* (with PKCE) for user apps, *client credentials* for machine-to-machine; the implicit grant is outdated.
- The gateway and the composite service act as OAuth2 resource servers; scopes (`product:read`, `product:write`) are checked per endpoint.
- Internal communication in the book remains unsecured inside the trusted network (a later chapter uses a service mesh for mTLS).
- Swagger UI can be integrated with the authorization code flow to test protected endpoints.
- Secrets (passwords, keys) must not be committed in plain text; chapter 12 covers encrypting configuration.

```java
@EnableWebFluxSecurity
public class SecurityConfig {
    @Bean
    SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
        http.authorizeExchange(ex -> ex
                .pathMatchers("/actuator/**").permitAll()
                .pathMatchers(POST, "/product-composite/**").hasAuthority("SCOPE_product:write")
                .pathMatchers(GET,  "/product-composite/**").hasAuthority("SCOPE_product:read")
                .anyExchange().authenticated())
            .oauth2ResourceServer(o -> o.jwt(withDefaults()));
        return http.build();
    }
}
```
```bash
# client credentials flow
curl -s -u writer:secret -d grant_type=client_credentials -d scope="product:read product:write" https://localhost:8443/oauth2/token
```

**Takeaways**
- Don't build your own auth: use standard protocols and a trusted authorization server (Keycloak, Auth0, Spring Authorization Server).
- Use scopes to limit what each client can do (least privilege), and always use HTTPS.

---

### Chapter 12: Centralized configuration

**Explanation**
- With many services and instances, managing config files per service is error-prone. Spring Cloud Config Server serves configuration from a central place (a Git repo or file system).
- Clients fetch their config at startup using `spring.config.import=configserver:...`. Files are named by application and profile (`product.yml`, `application-docker.yml`).
- Both *push* (via refresh) and *pull* models: the `/actuator/refresh` or Spring Cloud Bus can update running instances, but the book mostly restarts for simplicity.
- Sensitive values can be stored encrypted (`{cipher}...`) with a symmetric or asymmetric key on the Config Server, decrypted before being delivered.
- The config server itself is secured with HTTP basic authentication, and clients provide credentials.
- Docker Compose gets a `config-server` and services depend on it; environment-specific settings go in profiles.
- Config structure: shared `application.yml`, per-service files, per-profile overrides; later in Kubernetes ConfigMaps and Secrets replace this.

```yaml
# client: bootstrap config
spring:
  config.import: "configserver:http://${CONFIG_SERVER_USR}:${CONFIG_SERVER_PWD}@config-server:8888"
```
```bash
# encrypt a secret value
curl -s -u dev-usr:dev-pwd --data-urlencode "my-password" http://localhost:8888/encrypt
# put result in config as: password: '{cipher}f4...'
```

**Takeaways**
- Keep config out of images; versioned config in Git gives history and audit.
- Encrypt secrets at rest and protect the config server.

---

### Chapter 13: Improving resilience using Resilience4j

**Explanation**
- In a landscape of synchronous calls, a slow or failing service can bring down callers (cascading failure). Resilience patterns limit the damage.
- Resilience4j is a lightweight library with annotations/decorators for: **circuit breaker**, **time limiter**, **retry**, **rate limiter**, **bulkhead**.
- Circuit breaker: states closed (calls pass), open (fail fast, no calls), half-open (a few trial calls to see if the downstream recovered). Opens when failure rate exceeds a threshold within a sliding window.
- Time limiter: stop waiting after a timeout so threads aren't tied up.
- Retry: re-attempt transient failures with backoff, only for idempotent operations (reads), never blindly for writes.
- A *fallback* returns a cached or default value when the circuit is open, so the user gets degraded service rather than an error.
- Configuration is in `application.yml` (window size, failure threshold, wait duration in open state); state changes are published as events and exposed through Actuator and metrics.
- Test by making the downstream slow (`delay` parameter) or failing (`faultPercent`) and watching the breaker open and close.

```java
@CircuitBreaker(name = "product", fallbackMethod = "getProductFallback")
@Retry(name = "product")
@TimeLimiter(name = "product")
public Mono<Product> getProduct(int productId, int delay, int faultPercent) { ... }

private Mono<Product> getProductFallback(int productId, int delay, int faultPercent, CallNotPermittedException ex) {
    return Mono.just(new Product(productId, "Fallback product" + productId, 0));
}
```
```yaml
resilience4j:
  circuitbreaker.instances.product:
    slidingWindowSize: 5
    failureRateThreshold: 50
    waitDurationInOpenState: 10000
    permittedNumberOfCallsInHalfOpenState: 3
  timelimiter.instances.product.timeoutDuration: 2s
  retry.instances.product:
    maxAttempts: 3
    waitDuration: 1000
```

**Takeaways**
- Fail fast and degrade gracefully; decide per call what a sensible fallback is.
- Order matters: retry wraps the circuit breaker, which wraps the time limiter; tune values for your traffic.

---

### Chapter 14: Understanding distributed tracing

**Explanation**
- A single user request crosses many services, so logs from one service don't explain what happened. Distributed tracing links the pieces.
- Concepts: a **trace** is the whole request; a **span** is one unit of work (e.g. one service call); each has IDs, timing and tags. Trace context is propagated in HTTP headers (W3C `traceparent` or B3) and in message headers.
- In Spring Boot 3, **Micrometer Tracing** (replacing Spring Cloud Sleuth) instruments HTTP, messaging and logs, and exports spans to **Zipkin** (or another backend via OpenTelemetry).
- Trace and span IDs are added to log lines (MDC), so you can search all logs for one trace ID.
- Zipkin UI shows a timeline of spans per trace and a dependency graph, which makes slow services and failures easy to find.
- Works with messaging: the trace continues across the broker, so asynchronous flows are shown as one trace.
- Sampling controls overhead: `management.tracing.sampling.probability=1.0` in dev; lower in production.
- Custom spans and tags can be added with the `Tracer` or `@Observed`.

```yaml
management.tracing.sampling.probability: 1.0
management.zipkin.tracing.endpoint: http://zipkin:9411/api/v2/spans
logging.pattern.level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```
```gradle
implementation 'io.micrometer:micrometer-tracing-bridge-brave'
implementation 'io.zipkin.reporter2:zipkin-reporter-brave'
```

**Takeaways**
- Tracing answers "where did the time go / where did it fail" across services; it is essential in microservices.
- Propagate trace context everywhere (HTTP, messages) and include trace IDs in logs.

---

## Part 3: Kubernetes

### Chapter 15: Introduction to Kubernetes

**Explanation**
- Kubernetes (K8s) is a container orchestrator: it schedules containers on a cluster of nodes, restarts failed ones, scales, and handles networking and rolling updates.
- Main API objects: **Pod** (one or more containers; smallest unit), **Deployment** (desired state, replicas, rolling updates; creates a ReplicaSet), **Service** (stable name/IP and load balancing in front of pods), **Ingress** (HTTP routing from outside), **ConfigMap/Secret** (config), **Namespace** (isolation), **StatefulSet**, **Job**.
- Runtime components: control plane (API server, scheduler, controller manager, etcd) and nodes (kubelet, container runtime, kube-proxy).
- Declarative model: describe the desired state in YAML; controllers continuously reconcile reality with it.
- Probes: liveness (restart if failing), readiness (receive traffic only when ready), startup (slow starters). Spring Boot Actuator has health groups for these.
- The book uses a local cluster (Minikube / Docker Desktop) driven with `kubectl`.
- Compared with Docker Compose: K8s provides scaling, self-healing, service discovery and rolling updates natively.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: product }
spec:
  replicas: 2
  selector: { matchLabels: { app: product } }
  template:
    metadata: { labels: { app: product } }
    spec:
      containers:
        - name: product
          image: hands-on/product-service
          ports: [{ containerPort: 8080 }]
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
---
apiVersion: v1
kind: Service
metadata: { name: product }
spec:
  selector: { app: product }
  ports: [{ port: 80, targetPort: 8080 }]
```

**Takeaways**
- Think in desired state: you declare it and Kubernetes keeps it true.
- Health probes are what make rolling updates and self-healing safe.

---

### Chapter 16: Deploying our microservices to Kubernetes

**Explanation**
- Manual YAML for each service gets repetitive, so the book uses **Helm**: a package manager that templates Kubernetes manifests with values per environment.
- Chart structure: a shared *common* library chart with templates for Deployment, Service, ConfigMap, Secret; one chart per component that uses the common one; and an environment chart (e.g. `dev-env`, `prod-env`) that includes all components.
- Values files (`values.yaml`, overridden per environment) configure image, replicas, resources, env, probes.
- Images are built for the cluster's Docker daemon (or pushed to a registry) and referenced in the chart; Spring profile `docker` is replaced by a `kubernetes` profile.
- Configuration comes from ConfigMaps (files mounted as `application.yml`) and Secrets (credentials as env variables).
- Dependencies like MongoDB, MySQL, RabbitMQ are deployed as Deployments/StatefulSets in the cluster for dev; production would use managed services.
- Commands: `helm dependency update`, `helm install`, `helm upgrade`, `helm uninstall`; test with `kubectl get pods`, and the `test-em-all.bash` script.
- Rolling upgrade: change the image tag, Helm updates the Deployment, and K8s replaces pods gradually with no downtime while readiness probes pass.

```bash
helm dependency update kubernetes/helm/environments/dev-env
helm install hands-on-dev-env kubernetes/helm/environments/dev-env -n hands-on --create-namespace
kubectl get pods -n hands-on
helm upgrade hands-on-dev-env kubernetes/helm/environments/dev-env -n hands-on
```
```yaml
# component values.yaml
image: { name: product, tag: v1 }
replicaCount: 1
configmap:
  enabled: true
env:
  SPRING_PROFILES_ACTIVE: docker
```

**Takeaways**
- Helm templates remove duplication and let one chart deploy in several environments.
- Treat the deployment config as code in the repo; use rolling upgrades with probes for zero downtime.

---

### Chapter 17: Implementing Kubernetes features to simplify the system landscape

**Explanation**
- Kubernetes can replace several Spring Cloud components, so the landscape becomes simpler:
  - **Eureka and client-side load balancing** are replaced by Kubernetes **Service** (DNS name and load balancing).
  - **Spring Cloud Gateway as edge server** is replaced by **Ingress** (with an ingress controller such as NGINX), terminating TLS.
  - **Config Server** is replaced by **ConfigMaps** and **Secrets**.
- TLS certificates are provisioned automatically by **cert-manager**; Ingress then exposes HTTPS endpoints.
- Services are now called by Kubernetes Service names, so the `@LoadBalanced` setup is removed; Spring Cloud dependencies like Eureka and Config clients are disabled by profile.
- Kubernetes health probes use Actuator's readiness/liveness endpoints; graceful shutdown allows in-flight requests to finish when pods stop.
- Trade-off: fewer moving parts and less Java code, but behaviour is tied to Kubernetes and some Spring Cloud features (e.g. gateway filters, rate limiting) are no longer there.
- The book keeps a Spring Cloud Gateway variant only where needed (e.g. complex routing logic); otherwise Ingress is enough.
- Verified with the same test script on the new setup; the resulting system has fewer containers.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gateway
  annotations: { cert-manager.io/issuer: selfsigned-issuer }
spec:
  tls:
    - hosts: [minikube.me]
      secretName: tls-certificate
  rules:
    - host: minikube.me
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: product-composite, port: { number: 80 } } }
```

**Takeaways**
- Use the platform for platform problems: discovery, routing, config and certificates are built into Kubernetes.
- Fewer Spring Cloud components means less to run, but more reliance on Kubernetes.

---

### Chapter 18: Using a service mesh to improve observability and management

**Explanation**
- A service mesh moves cross-cutting network concerns out of application code into the platform. Every pod gets a **sidecar proxy** (Envoy) that intercepts traffic; a control plane (**Istio**'s `istiod`) configures the proxies.
- Capabilities: traffic management (routing, weighted splits, canary, retries, timeouts, fault injection), security (automatic mTLS between services, authorization policies, JWT validation), and observability (metrics, traces and a service graph with **Kiali**, plus Prometheus, Grafana and Jaeger/Zipkin integrations).
- Istio objects: `Gateway` + `VirtualService` (edge routing, replacing Ingress), `DestinationRule` (subsets, load balancing, circuit breaker settings), `PeerAuthentication` (mTLS mode), `RequestAuthentication` and `AuthorizationPolicy` (JWT and access rules).
- Resilience can be configured in the mesh (retries, timeouts, outlier detection) instead of in Resilience4j, and fault injection can test it.
- Zero-downtime deployments with traffic shifting between v1 and v2 (blue/green or canary) by changing weights.
- Cost: more resources per pod, added complexity and latency; sidecar-less ambient mode is a newer option.
- The book converts the previous setup: remove Ingress, add the Istio gateway, enable mTLS (STRICT), and check traffic in Kiali.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata: { name: product-composite }
spec:
  hosts: [product-composite]
  http:
    - route:
        - destination: { host: product-composite, subset: old }
          weight: 90
        - destination: { host: product-composite, subset: new }
          weight: 10
---
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata: { name: default }
spec:
  mtls: { mode: STRICT }
```

**Takeaways**
- A mesh gives uniform security, traffic control and visibility without changing service code.
- It is powerful but heavy; adopt it when you need those features at scale.

---

## Part 4: Operations

### Chapter 19: Centralized logging with the EFK stack

**Explanation**
- With many pods that come and go, logs must be collected centrally; otherwise they disappear with the container and can't be searched together.
- **EFK stack**: **Elasticsearch** (store and index), **Fluentd** (collect and forward logs from each node), **Kibana** (search and visualize).
- Fluentd runs as a DaemonSet (one per node), reads container logs from the node, parses them (for Spring Boot: multi-line stack traces and JSON), adds Kubernetes metadata (pod, namespace, labels) and ships them to Elasticsearch.
- Services should write logs to **stdout** (12-factor), so the platform can pick them up; no log files in containers.
- Include the trace ID in logs (from chapter 14) so Kibana can show all logs of one request across services.
- In Kibana: define an index pattern, filter by namespace/pod/container, use KQL for searching (e.g. `kubernetes.labels.app: "product-composite" and log: "ERROR"`), and build dashboards.
- Elasticsearch needs lots of memory; for local use, run a small single-node setup and expect to tune resources.
- Parsing multi-line stack traces is a common problem; handle it in Fluentd config (regex on the start of each log record).

```yaml
# Fluentd: parse Spring Boot log lines and keep traceId
<filter kubernetes.**>
  @type parser
  key_name log
  <parse>
    @type regexp
    expression /^(?<time>[^ ]+) +(?<level>[A-Z]+) \[(?<service>[^,]*),(?<traceId>[^,]*),(?<spanId>[^\]]*)\] (?<message>.*)$/
  </parse>
</filter>
```

**Takeaways**
- Log to stdout, centralize with a collector, and always correlate with trace IDs.
- Centralized logging is a requirement for debugging distributed systems, not a nice-to-have.

---

### Chapter 20: Monitoring microservices

**Explanation**
- Monitoring shows the health and performance of the system over time, so you can alert before users notice problems. Logs and traces explain individual requests; metrics show trends.
- **Micrometer** is the metrics facade in Spring Boot; Actuator exposes `/actuator/prometheus`, so **Prometheus** can scrape numbers like JVM memory, CPU, HTTP request counts and latencies, and Resilience4j state.
- **Prometheus** stores time series and answers queries with PromQL; **Grafana** builds dashboards on it; **Alertmanager** sends alerts (e-mail, Slack, etc.) based on rules.
- In Kubernetes, the Prometheus Operator / `kube-prometheus-stack` is installed with Helm and uses `ServiceMonitor` objects to find the endpoints to scrape.
- Dashboards: import community dashboards (JVM, Spring Boot, Kubernetes) and create custom ones, such as the status of circuit breakers or the request rate per service.
- Custom metrics: define counters, gauges, timers with `MeterRegistry` for business events (e.g. number of orders created).
- Alerts should be based on symptoms users feel (error rate, latency) with sensible thresholds, not on every metric.
- The three pillars of observability combined: metrics (Prometheus/Grafana), logs (EFK), traces (Zipkin/Micrometer).

```yaml
management.endpoints.web.exposure.include: health,info,prometheus
management.metrics.tags.application: ${spring.application.name}
```
```java
@Component
class OrderMetrics {
    private final Counter created;
    OrderMetrics(MeterRegistry registry) {
        this.created = Counter.builder("orders.created").register(registry);
    }
    void onCreated() { created.increment(); }
}
```
```
# PromQL
rate(http_server_requests_seconds_count{application="product-composite", status=~"5.."}[1m])
resilience4j_circuitbreaker_state{name="product", state="open"} == 1
```
```yaml
# alert rule
- alert: HighErrorRate
  expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.05
  for: 2m
  labels: { severity: critical }
```

**Takeaways**
- Expose metrics from the app; let Prometheus pull them, and Grafana show them.
- Alert on user-visible symptoms, and use metrics, logs and traces together to diagnose.
