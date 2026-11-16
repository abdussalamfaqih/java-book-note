# Distributed System Design Patterns

Notes on 12 patterns for distributed systems, based on the article [12 Essential Distributed System Design Patterns Every Architect Should Know](https://antondevtips.com/blog/12-essential-distributed-system-design-patterns-every-architect-should-know) (Anton Martyniuk, Dec 2025). Each pattern is enriched with explanation, trade-offs, a small Java/Spring-flavoured snippet and references from architecture books and documentation.

## How to read the references

- **Online links** were opened and loaded when these notes were written (2026-11-16). Pages may change later.
- **Book references** (title, author, chapter/section) are from memory and were **not** checked against the books. Chapter numbers can differ between editions, so treat them as pointers and confirm in your copy.
- The snippets are illustrative drafts, not compiled or run.

## Patterns

| # | Pattern | Group |
|---|---------|-------|
| 1 | API Gateway | Communication / edge |
| 2 | Point-to-Point Async Integration | Messaging |
| 3 | Publish/Subscribe | Messaging |
| 4 | Outbox | Data consistency |
| 5 | CQRS | Data / queries |
| 6 | Saga | Data consistency |
| 7 | Sidecar | Deployment |
| 8 | Strangler Fig | Migration |
| 9 | Anti-Corruption Layer | Migration / DDD |
| 10 | Service Discovery | Infrastructure |
| 11 | Sharding | Data scaling |
| 12 | Replication | Data availability |

## Core books used

- **[MP]** Chris Richardson, *Microservices Patterns* (Manning, 2018)
- **[BM]** Sam Newman, *Building Microservices*, 2nd ed. (O'Reilly, 2021)
- **[DDIA]** Martin Kleppmann, *Designing Data-Intensive Applications* (O'Reilly, 2017)
- **[EIP]** Gregor Hohpe, Bobby Woolf, *Enterprise Integration Patterns* (Addison-Wesley, 2003)
- **[DDD]** Eric Evans, *Domain-Driven Design* (Addison-Wesley, 2003)
- **[DDS]** Brendan Burns, *Designing Distributed Systems* (O'Reilly, 2018)
- **[IDDD]** Vaughn Vernon, *Implementing Domain-Driven Design* (Addison-Wesley, 2013)

Online catalogues: [microservices.io](https://microservices.io/patterns/), [Azure Architecture Center: Cloud Design Patterns](https://learn.microsoft.com/en-us/azure/architecture/patterns/).

---

## 1. API Gateway

**Problem:** Clients would have to call many services directly, know their addresses, and each service would reimplement auth, rate limiting and request translation.

**How it works**
- A single entry point (reverse proxy) sits between clients and services.
- It handles cross-cutting concerns: authentication/authorization, rate limiting and throttling, TLS termination, routing by path or header.
- It can **aggregate** several service calls into one response (fewer round trips for mobile clients) and adapt the API per client type (Backend for Frontend, BFF).

**Trade-offs**
- Pros: simpler clients, hidden internal structure, one place for policies.
- Cons: single point of failure and potential bottleneck (run several instances), extra hop latency, risk of growing business logic inside it (keep it thin).

```yaml
# Spring Cloud Gateway route
spring.cloud.gateway.routes:
  - id: orders
    uri: lb://order-service
    predicates: [ Path=/api/orders/** ]
    filters:
      - StripPrefix=1
      - name: RequestRateLimiter
        args: { redis-rate-limiter.replenishRate: 10, redis-rate-limiter.burstCapacity: 20 }
```

**References**
- [MP] Ch. 8, *External API patterns* (API gateway, BFF).
- [BM] Ch. 5, *Implementing microservice communication* (API gateways and aggregation; see also the section on BFF).
- Online: [microservices.io: API Gateway / BFF](https://microservices.io/patterns/apigateway.html); [Azure: Gateway Routing](https://learn.microsoft.com/en-us/azure/architecture/patterns/gateway-routing) (the Azure catalogue also has *Gateway Aggregation* and *Gateway Offloading*).
- See also `microservices-spring-boot-cloud.md`, chapters 10 and 17.

---

## 2. Point-to-Point Async Integration

**Problem:** A sender must not block (or fail) when the receiver is slow or down.

**How it works**
- The sender puts a message on a **queue**; exactly **one** consumer processes each message.
- The broker buffers (and may persist) messages, so work waits if the consumer is down, and consumers can scale out as *competing consumers* to absorb load peaks.
- Messages that keep failing go to a **dead letter queue** after N retries.

**Trade-offs**
- Pros: temporal decoupling, load leveling, resilience, independent scaling.
- Cons: broker becomes a dependency, eventual consistency (the sender doesn't know the outcome), harder debugging, consumers must be **idempotent** since delivery is usually at-least-once.

```java
// Spring AMQP: producer and consumer with a dead-letter setup
rabbitTemplate.convertAndSend("orders.queue", new OrderPlaced(orderId));

@RabbitListener(queues = "orders.queue")
public void handle(OrderPlaced event) {
    fulfilmentService.start(event.orderId());   // must be idempotent
}
```

**References**
- [EIP] Ch. 3, *Messaging Systems*: *Point-to-Point Channel*, *Dead Letter Channel*, *Guaranteed Delivery*; Ch. 10, *Messaging Endpoints*: *Competing Consumers*, *Idempotent Receiver*.
- [MP] Ch. 3, *Interprocess communication in a microservice architecture* (asynchronous messaging).
- [DDIA] Ch. 11, *Stream Processing* (message brokers vs log-based brokers, acknowledgments and redelivery).
- Online: [EIP: Point-to-Point Channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PointToPointChannel.html).

---

## 3. Publish/Subscribe

**Problem:** One event must reach several independent consumers, and the publisher shouldn't know who they are.

**How it works**
- Publishers send events to a **topic**; the broker delivers a copy to **every** subscriber (each subscriber, or consumer group, gets its own copy).
- Subscribers are added or removed without changing the publisher; filters (by type or header) are possible.
- Difference from point-to-point: one event goes to *many* consumers; in point-to-point one message goes to *one* consumer.

**Trade-offs**
- Pros: loose coupling, easy to extend, parallel processing.
- Cons: harder to trace who reacted to what, ordering and duplicates must be handled, eventual consistency, broker is critical infrastructure.

```java
// Spring Cloud Stream: one function bean per subscriber, each with its own group
@Bean Consumer<OrderPlaced> sendEmail()    { return e -> emailService.confirm(e.orderId()); }
@Bean Consumer<OrderPlaced> updateStock()  { return e -> stock.reserve(e.items()); }
```
```yaml
spring.cloud.stream.bindings:
  sendEmail-in-0:   { destination: order-events, group: email }
  updateStock-in-0: { destination: order-events, group: stock }
```

**References**
- [EIP] Ch. 3, *Publish-Subscribe Channel*; Ch. 3, *Message Broker* and *Event Message*.
- [MP] Ch. 3 (publish/subscribe style, domain events); Ch. 5 (publishing domain events from aggregates).
- [DDIA] Ch. 11 (topics, consumer groups, partitions, log-based messaging).
- Online: [EIP: Publish-Subscribe Channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PublishSubscribeChannel.html); [Azure: Publisher-Subscriber](https://learn.microsoft.com/en-us/azure/architecture/patterns/publisher-subscriber).

---

## 4. Outbox

**Problem:** The **dual-write problem**: a service updates its database *and* publishes an event, but these are two systems and no shared transaction. If one fails, data and events disagree (order saved but no event, or event sent but order rolled back).

**How it works**
- In the **same local DB transaction** as the business change, insert the event into an `outbox` table.
- A relay (a poller, or **change data capture** such as Debezium reading the transaction log) publishes the rows to the broker and marks or deletes them.
- Delivery is **at-least-once**, so consumers must deduplicate (an **inbox** table or idempotent handlers).

**Trade-offs**
- Pros: no lost events if the commit succeeded; audit trail; enables replay.
- Cons: eventual consistency (relay delay), extra table and relay to run and monitor, duplicates possible, ordering needs care.

```java
@Transactional
public void placeOrder(Order order) {
    orders.save(order);
    outbox.save(new OutboxEvent(UUID.randomUUID(), "OrderPlaced", toJson(order)));
    // both rows commit atomically; a separate relay publishes outbox rows
}

@Scheduled(fixedDelay = 1000)
void relay() {
    for (OutboxEvent e : outbox.findUnpublished(100)) {
        broker.send(e.type(), e.payload());   // may be sent twice: consumers dedupe by e.id()
        e.markPublished();
    }
}
```

**References**
- [MP] Ch. 3, *Reliably publishing messages: Transactional outbox / polling publisher / transaction log tailing* (section on transactional messaging).
- [DDIA] Ch. 11, *Databases and Streams*: dual writes, change data capture, event sourcing; Ch. 9 for consistency guarantees.
- [EIP] *Guaranteed Delivery*, *Idempotent Receiver* (for the consumer side).
- Online: [microservices.io: Transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html); [Debezium: Reliable Microservices Data Exchange With the Outbox Pattern](https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/).

---

## 5. CQRS (Command Query Responsibility Segregation)

**Problem:** One model serves both writes (business rules, consistency) and reads (joins, projections, scale), and each pulls the design in a different direction.

**How it works**
- **Commands** change state through a *write model* that enforces invariants.
- **Queries** read from a *read model* shaped for the screens/reports; it can be a separate table, database or search index.
- The read model is updated from the write side, often asynchronously via events (so it is **eventually consistent**).
- Can be used without event sourcing; the two often appear together but are independent ideas.

**Trade-offs**
- Pros: scale and optimize reads and writes independently, simpler query code, simpler command validation.
- Cons: more moving parts, stale reads, sync infrastructure; overkill for plain CRUD.

```java
// write side
record PlaceOrderCommand(UUID customerId, List<Item> items) {}
class OrderCommandHandler { void handle(PlaceOrderCommand c) { /* validate, save, publish OrderPlaced */ } }

// read side: projection updated from events, queried without joins
@EventListener
void on(OrderPlaced e) { orderSummaryView.save(new OrderSummary(e.orderId(), e.customerName(), e.total())); }
```

**References**
- [MP] Ch. 7, *Implementing queries in a microservice architecture* (CQRS and API composition); Ch. 6 for event sourcing.
- [DDIA] Ch. 11 (derived data, materialized views, separating reads and writes).
- Greg Young's CQRS documents and Bertrand Meyer's *Object-Oriented Software Construction* (command-query separation, the idea CQRS grows from).
- Online: [Martin Fowler: CQRS](https://martinfowler.com/bliki/CQRS.html); [microservices.io: CQRS](https://microservices.io/patterns/data/cqrs.html).

---

## 6. Saga

**Problem:** A business transaction spans several services, each with its own database. Two-phase commit and distributed locks don't scale and aren't supported by most message brokers/NoSQL stores.

**How it works**
- Split the transaction into a sequence of **local transactions**, each in one service.
- Each step publishes an outcome; if one fails, **compensating transactions** undo the previous steps, in reverse order (e.g. `ReleaseStock` for `ReserveStock`).
- Two styles:
  - **Choreography**: services react to each other's events; no central coordinator (simple for 2-3 steps, hard to follow beyond).
  - **Orchestration**: a coordinator tells each participant what to do next (clearer flow, one more component).
- Sagas give **ACD** without isolation: intermediate states are visible, so you need countermeasures (semantic locks, versioning, re-reading values).

**Trade-offs**
- Pros: no distributed locks; long-running flows; each service stays autonomous.
- Cons: every step needs a compensation, lack of isolation, compensations can fail (and need retries), more complex monitoring.

```java
// Orchestrator sketch
class CreateOrderSaga {
    void start(Order o) {
        try {
            payment.charge(o);          // step 1
            stock.reserve(o);           // step 2
            shipping.schedule(o);       // step 3
            orders.approve(o);
        } catch (Exception e) {
            shipping.cancel(o);         // compensations, reverse order, each idempotent
            stock.release(o);
            payment.refund(o);
            orders.reject(o);
        }
    }
}
```
(In practice each call is an async command with a reply; the orchestrator persists its state.)

**References**
- Hector Garcia-Molina, Kenneth Salem, *Sagas*, ACM SIGMOD 1987 (the original paper).
- [MP] Ch. 4, *Managing transactions with sagas* (choreography, orchestration, countermeasures for lack of isolation).
- [BM] Ch. 6, *Workflow* (sagas, choreographed vs orchestrated, and compensating transactions).
- Pat Helland, *Life beyond Distributed Transactions: an Apostate's Opinion* (CIDR 2007).
- [DDIA] Ch. 7 and 9 (why distributed transactions and two-phase commit are costly).
- Online: [microservices.io: Saga](https://microservices.io/patterns/data/saga.html); [Azure: Saga](https://learn.microsoft.com/en-us/azure/architecture/patterns/saga).

---

## 7. Sidecar

**Problem:** Logging, monitoring, config, TLS and traffic management are needed by every service, in every language, and shouldn't be tangled with business code.

**How it works**
- Deploy a helper process/container **next to the app**, sharing its lifecycle and network namespace (they talk via `localhost`).
- The sidecar intercepts or augments traffic: proxying (Envoy in Istio), log shipping, config refresh, secrets, or a portable messaging API (Dapr).
- In Kubernetes: another container in the same Pod.

**Trade-offs**
- Pros: language-agnostic, separates infrastructure from business logic, upgraded independently, reuse.
- Cons: extra CPU/memory per instance, more complex deployment, extra network hop latency.

```yaml
# Pod with app + log-shipping sidecar sharing a volume
spec:
  containers:
    - name: app
      image: shop/order-service
      volumeMounts: [{ name: logs, mountPath: /var/log/app }]
    - name: log-shipper
      image: fluent/fluent-bit
      volumeMounts: [{ name: logs, mountPath: /var/log/app }]
  volumes: [{ name: logs, emptyDir: {} }]
```

**References**
- [DDS] Part II, *Single-node patterns*: sidecar, ambassador and adapter. Also Burns & Oppenheimer, *Design Patterns for Container-based Distributed Systems* (USENIX HotCloud 2016).
- [MP] Ch. 12, *Deploying microservices* (sidecar and service mesh).
- [BM] Ch. 5 and 8 (service meshes, and deployment).
- Online: [microservices.io: Sidecar](https://microservices.io/patterns/deployment/sidecar.html); [Azure: Sidecar](https://learn.microsoft.com/en-us/azure/architecture/patterns/sidecar).
- See also `microservices-spring-boot-cloud.md`, chapter 18 (Istio).

---

## 8. Strangler Fig

**Problem:** A big-bang rewrite of a legacy system is risky, takes long, and delivers value only at the end.

**How it works**
- Put a **facade** (router or gateway) in front of the legacy system.
- Build new functionality as new services; route requests per feature to old or new.
- Migrate slice by slice; both systems run in parallel until the legacy part has nothing left and is retired.
- Typical steps: identify a seam, build the new slice, route a % of traffic (feature flag), compare, switch fully, delete the old code.

**Trade-offs**
- Pros: incremental, reversible, continuous delivery, lower risk.
- Cons: two systems to run for a long time, data synchronization during transition, routing layer to maintain; needs discipline to finish.

```yaml
# Route migrated feature to the new service, everything else stays on the monolith
spring.cloud.gateway.routes:
  - id: new-invoices
    uri: lb://invoice-service
    predicates: [ Path=/invoices/** ]
  - id: legacy
    uri: http://legacy-monolith:8080
    predicates: [ Path=/** ]       # catch-all, order matters
```

**References**
- Martin Fowler, *StranglerFigApplication* (bliki, 2004) and the later article on legacy seams.
- [BM] Ch. 3, *Splitting the monolith* (strangler fig pattern, branch by abstraction, parallel run).
- [MP] Ch. 13, *Refactoring to microservices* (strangler application).
- Michael Feathers, *Working Effectively with Legacy Code* (seams) for the code-level side.
- Online: [Martin Fowler: Strangler Fig](https://martinfowler.com/bliki/StranglerFigApplication.html); [microservices.io: Strangler application](https://microservices.io/patterns/refactoring/strangler-application.html).

---

## 9. Anti-Corruption Layer (ACL)

**Problem:** Integrating with a legacy or third-party system whose model and terminology would leak into and "corrupt" your clean domain model.

**How it works**
- A translation layer between your **bounded context** and the external one: adapters, facades and translators that map external types to domain types (and back).
- Domain code depends only on its own interface (a port); the ACL implements it using the external API.
- It can combine several external calls into one domain operation, and hide quirks (bad naming, codes, error formats).
- Often used during a Strangler Fig migration so new services talk to the legacy through the ACL.

**Trade-offs**
- Pros: protected domain model, easy to mock in tests, centralized integration, external changes touch only one place.
- Cons: extra code and mapping overhead, another thing to maintain and monitor; can grow too big or become a bottleneck.

```java
// Domain port: speaks domain language
public interface CustomerDirectory { Customer findById(CustomerId id); }

// ACL adapter: translates the legacy CRM model
class LegacyCrmCustomerDirectory implements CustomerDirectory {
    private final LegacyCrmClient crm;
    public Customer findById(CustomerId id) {
        LegacyCustRec rec = crm.getCust(id.value());                 // CUST_NM, STAT_CD ...
        return new Customer(id, rec.CUST_NM, "A".equals(rec.STAT_CD) ? Status.ACTIVE : Status.CLOSED);
    }
}
```

**References**
- [DDD] Part IV, Ch. 14, *Maintaining model integrity* (Anti-Corruption Layer; also Context Map patterns).
- [IDDD] Ch. 3, *Context Maps* (integration with legacy and upstream contexts).
- [BM] Ch. 3 (ACL in the context of migrating from a monolith).
- Alistair Cockburn, *Hexagonal Architecture* (ports and adapters), the structure an ACL usually takes.
- Online: [Azure: Anti-Corruption Layer](https://learn.microsoft.com/en-us/azure/architecture/patterns/anti-corruption-layer).

---

## 10. Service Discovery

**Problem:** Instances are created, scaled, moved and replaced all the time, so hard-coded host/port values break.

**How it works**
- A **service registry** keeps the live instances of each service.
- Instances **register** at startup, send **heartbeats** and are removed when these stop (or on deregistration).
- Two styles:
  - **Client-side discovery**: the client asks the registry and load-balances itself (Eureka + Spring Cloud LoadBalancer).
  - **Server-side discovery**: a router/load balancer or the platform resolves the name (Kubernetes Service + DNS, cloud load balancers).
- The registry itself should be highly available (replicated); clients cache results to survive short outages.

**Trade-offs**
- Pros: dynamic scaling, automatic failover, no manual config.
- Cons: the registry is a critical dependency, stale entries are possible, extra moving part; on Kubernetes the platform already provides it.

```java
@Bean @LoadBalanced
RestClient.Builder restClientBuilder() { return RestClient.builder(); }

// logical name instead of host:port
String status = restClientBuilder.build().get().uri("http://payment-service/status").retrieve().body(String.class);
```

**References**
- [MP] Ch. 3 (*Service discovery*: client-side, server-side, self registration, third-party registration).
- [BM] Ch. 5 (DNS and service discovery).
- [DDIA] Ch. 9 (consensus and coordination services such as ZooKeeper and etcd, which registries often build on).
- Online: [microservices.io: Client-side discovery](https://microservices.io/patterns/client-side-discovery.html); [Kubernetes: Service](https://kubernetes.io/docs/concepts/services-networking/service/).
- See also `microservices-spring-boot-cloud.md`, chapters 9 and 17.

---

## 11. Sharding

**Problem:** Data volume or write throughput outgrows what one database node can handle.

**How it works**
- Split data **horizontally** into shards, each on its own node, chosen by a **shard key**.
- Strategies: **range** (efficient range scans, risk of hot spots), **hash** (even spread, no range queries), **directory/lookup** (flexible, extra lookup), **geo/tenant** (data locality, multi-tenant SaaS).
- A router (app logic, proxy or the database itself) maps key to shard; **consistent hashing** reduces data movement when shards are added.
- Queries across shards (scatter-gather) and transactions across shards are costly or limited.

**Trade-offs**
- Pros: horizontal scale of storage and throughput, fault isolation, smaller indexes.
- Cons: hard shard-key choice (hard to change later), hot shards, cross-shard joins and transactions, rebalancing, schema changes across all shards.

```java
// simple hash routing by tenant (illustrative)
int shard = Math.floorMod(tenantId.hashCode(), shardCount);
JdbcTemplate db = shards.get(shard);
db.update("INSERT INTO orders ...");
```
(Spring's `AbstractRoutingDataSource` can route per request.)

**References**
- [DDIA] Ch. 6, *Partitioning* (key-range, hash partitioning, secondary indexes, rebalancing, request routing).
- DeCandia et al., *Dynamo: Amazon's Highly Available Key-value Store*, SOSP 2007 (consistent hashing and partitioning).
- Pat Helland, *Life beyond Distributed Transactions* (entities as the unit of scale).
- [MP] Ch. 2/3 for *database per service* (which is not the same thing as sharding, but often confused).
- Online: [Azure: Sharding](https://learn.microsoft.com/en-us/azure/architecture/patterns/sharding).

---

## 12. Replication

**Problem:** A single copy of data is a single point of failure and a read bottleneck.

**How it works**
- Keep **copies** of the data on several nodes.
- **Single-leader (primary-replica)**: all writes go to the leader and are copied to followers; reads can go to followers. If the leader fails, a follower is **promoted** (failover).
- **Multi-leader** and **leaderless** (quorum reads/writes, e.g. Dynamo-style) support multi-region writes but need conflict resolution.
- **Synchronous** replication: safer, slower writes. **Asynchronous**: faster, but replicas lag and data can be lost on leader failure.
- Replication gives availability and read scale, **not** write scale. Combine with sharding (replicas per shard).
- Anomalies to know: reading stale data, reading your own writes, monotonic reads.

**Trade-offs**
- Pros: availability, disaster recovery, lower read latency, zero-downtime maintenance.
- Cons: replica lag (eventual consistency), failover complexity (split brain), conflict handling in multi-leader, storage cost.

```java
// Read/write splitting in Spring (illustrative)
@Transactional(readOnly = true)   // route to replica via a routing DataSource
public Order find(UUID id) { return orders.findById(id).orElseThrow(); }
```

**References**
- [DDIA] Ch. 5, *Replication* (leaders and followers, replication lag, multi-leader, leaderless); Ch. 9 for consistency models and consensus.
- DeCandia et al., *Dynamo* (leaderless replication, quorums).
- Brewer's CAP theorem and Gilbert & Lynch, *Brewer's conjecture and the feasibility of consistent, available, partition-tolerant web services* (2002).
- Online: [Kubernetes: StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) for running replicated databases, and your database's own replication guide (PostgreSQL, MySQL, MongoDB).

---

## How the patterns fit together

- **Edge and migration:** API Gateway fronts the system; Strangler Fig uses it (or a similar facade) to move traffic; an Anti-Corruption Layer protects new code from the legacy model.
- **Messaging and consistency:** Point-to-Point and Pub/Sub carry messages; Outbox makes publishing reliable; Saga uses those events to coordinate multi-service transactions; CQRS read models are fed the same way.
- **Platform:** Service Discovery and Sidecar (service mesh) handle runtime networking and cross-cutting concerns.
- **Data at scale:** Sharding spreads the data; Replication keeps copies of each shard for availability.
- Recurring trade-off: **eventual consistency, more infrastructure and harder debugging in exchange for scalability and resilience.** Add a pattern only when you actually have its problem.
