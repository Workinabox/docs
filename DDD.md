# Domain-Driven Design Reference

This document is the authoritative DDD reference for the Workinabox project. All agents and contributors working on domain modeling, backend architecture, or code structure must read and follow this guide.

---

## Why DDD

Workinabox is an agentic system for running companies. The domain is complex, evolving, and the most valuable part of the system. DDD exists to keep that domain complexity manageable by making the code reflect how the business actually works.

---

## Part 1: Strategic Design

Strategic design is about the big picture — where boundaries go, how contexts relate, and how teams organize around the domain.

### Ubiquitous Language

A shared language between developers and domain experts that appears directly in code — struct names, method names, module names. Not a glossary. The living code.

Rules:

- If developers use different terms than domain experts, fix it. The translation layer is a bug source.
- The language is scoped to a Bounded Context. The same word can mean different things in different contexts.
- When the language changes (because understanding deepens), the code changes to match. They co-evolve.

### Bounded Contexts

An explicit boundary within which a particular domain model is defined and valid. The most important strategic pattern.

- Each Bounded Context has its own language, its own model, and ideally its own crate or module.
- A concept like "Agent" can exist in multiple contexts with different attributes. A meeting agent and a pipeline agent are not the same model — don't force them into one struct.
- Bounded Contexts are a linguistic boundary first, a technical boundary second. They emerge from where language diverges.
- They align well with team boundaries (Conway's Law).

**Bounded Context vs. Subdomain:** A subdomain is a problem-space concept (it exists whether you model it or not). A Bounded Context is a solution-space concept (a modeling choice). Ideally 1:1.

### Context Mapping

Identifying all Bounded Contexts and the relationships between them. The Context Map shows how contexts communicate and integrate.

#### Relationship Patterns

| Pattern | Description | When to use |
|---------|-------------|-------------|
| **Shared Kernel** | Two contexts share a subset of the model. Changes require agreement from both sides. | Closely related contexts, cooperating teams. Keep the kernel small. |
| **Customer/Supplier** | Upstream provides, downstream consumes. Downstream can negotiate. | Clear dependency direction, cooperative teams. |
| **Conformist** | Downstream must accept upstream's model as-is. No influence. | Third-party systems, legacy monoliths, uncooperative teams. |
| **Anti-Corruption Layer** | Translation layer isolating downstream from upstream's model. | Upstream model is significantly different, legacy, or external. Almost every external integration needs one. |
| **Open Host Service** | A context defines a well-known API for multiple consumers. | One-to-many integration. |
| **Published Language** | A documented, versioned data format for cross-context communication. | Stable contracts between contexts. Often paired with Open Host Service. |
| **Partnership** | Mutual dependency, both succeed or fail together. | Tightly collaborating teams. |
| **Separate Ways** | No integration. Solve needs independently, accept duplication. | Cost of integration exceeds benefit. |

### Subdomain Classification

Not everything deserves the same investment:

- **Core Domain**: The competitive advantage. Full DDD with rich models, events, careful aggregate design. This is where Workinabox's value lives.
- **Supporting Subdomain**: Necessary but not differentiating. Simpler patterns, less strict aggregate design.
- **Generic Subdomain**: Buy or use open-source. Don't invest DDD effort. Authentication, email delivery, payment processing.

---

## Part 2: Tactical Patterns (Building Blocks)

### Value Objects

Objects defined entirely by their attributes. No identity. Immutable.

Properties:

- Two value objects are equal if all attributes are equal
- Immutable — to "modify," create a new instance
- Self-validating — always in a valid state after construction
- Most concepts in most domains are value objects, not entities

Examples: Money, EmailAddress, Address, DateRange, Quantity, AgentId, TaskId.

**In Rust:**

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct EmailAddress(String);

impl EmailAddress {
    pub fn new(value: impl Into<String>) -> Result<Self, DomainError> {
        let value = value.into();
        if !value.contains('@') || value.len() < 3 {
            return Err(DomainError::InvalidEmail(value));
        }
        Ok(Self(value))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}
```

Private inner field + fallible constructor = invalid state is unrepresentable.

Derive `Clone`, `PartialEq`, `Eq`, `Hash` as appropriate. Derive `Copy` if small. Implement `Display` for presentation.

### The Newtype Pattern

The foundation of DDD in Rust. Wraps primitives to create distinct domain types with zero runtime cost.

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct OrderId(Uuid);

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct AgentId(String);
```

The compiler prevents mixing `OrderId` with `AgentId` even though both wrap common types.

For many ID types, a macro reduces boilerplate:

```rust
macro_rules! define_id {
    ($name:ident) => {
        #[derive(Debug, Clone, PartialEq, Eq, Hash)]
        pub struct $name(uuid::Uuid);

        impl $name {
            pub fn new() -> Self { Self(uuid::Uuid::new_v4()) }
        }

        impl std::fmt::Display for $name {
            fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
                write!(f, "{}", self.0)
            }
        }
    };
}
```

### Entities

Objects with a distinct identity that persists through time. Two entities are equal iff they have the same identity, regardless of attributes.

Properties:

- Unique identity (usually an ID value object)
- Mutable — state changes while identity remains
- Equality by identity, not attributes
- Behavior enforces invariants

**In Rust:**

```rust
#[derive(Debug)]
pub struct Customer {
    id: CustomerId,
    name: PersonName,
    email: EmailAddress,
}

impl PartialEq for Customer {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id // Identity-based equality
    }
}
impl Eq for Customer {}
```

Place behavior on entities when the behavior requires the entity's state and enforces its invariants. Use intention-revealing method names: `order.cancel()` not `order.set_status("cancelled")`.

### Aggregate Roots

An Aggregate is a cluster of domain objects treated as a single unit for data changes. The Aggregate Root is the single entity through which all external access passes.

Properties:

- External code can only hold references to the root, never internal entities
- The root enforces all invariants for the entire aggregate
- All modifications go through the root — it is the consistency boundary
- Each aggregate is a transactional boundary

**In Rust:**

```rust
pub struct Order {
    id: OrderId,
    customer_id: CustomerId,       // Reference by identity, not owned Customer
    items: Vec<OrderItem>,          // Owned internal entities
    status: OrderStatus,
    shipping_address: Address,      // Owned value object
}

impl Order {
    pub fn add_item(&mut self, product: ProductId, qty: Quantity, price: Money) -> Result<(), OrderError> {
        if self.status != OrderStatus::Draft {
            return Err(OrderError::CannotModifyNonDraftOrder);
        }
        self.items.push(OrderItem::new(product, qty, price));
        Ok(())
    }
}
```

Rust's ownership model naturally enforces aggregate boundaries — the root *owns* its internal entities. External code gets `&` references but cannot hold them across mutations.

#### Aggregate Design Rules

1. **Model true invariants in consistency boundaries.** If two things must be consistent *right now*, they belong in the same aggregate. If eventual consistency suffices, separate aggregates.

2. **Design small aggregates.** Prefer root entity + value objects. Add internal entities only when true invariants demand it. Large aggregates signal clustering by data relationships rather than invariants.

3. **Reference other aggregates by identity.** Store `customer_id: CustomerId`, not `customer: Customer`. This makes boundaries explicit and allows aggregates to live in different contexts.

4. **Update other aggregates via eventual consistency.** When a change in one aggregate requires a change in another, use domain events, not cross-aggregate transactions.

### Domain Services

Operations that are important domain concepts but don't belong on any entity or value object. Verbs in the ubiquitous language that don't fit as methods on existing objects.

Properties:

- Stateless
- Named using ubiquitous language verbs
- Live in the domain layer
- Used when an operation involves multiple aggregates

When to use vs. entity methods:

| Behavior belongs on **entity** when... | Behavior belongs on **domain service** when... |
|---|---|
| It requires the entity's own state | It involves multiple aggregates |
| It enforces the entity's invariants | Forcing it onto one entity creates artificial coupling |
| The ubiquitous language says "the order does X" | The ubiquitous language names it as a separate concept |

**Prefer entity methods first.** Only extract to a domain service when behavior truly doesn't fit on one object. Over-extraction leads to anemic models.

### Application Services

Thin orchestrators that coordinate domain objects to perform a use case. The entry point for application behavior.

Properties:

- Contain NO business logic — delegate to domain objects
- Handle cross-cutting concerns: transactions, security, logging, event publishing
- Translate between outside world (DTOs, commands) and domain model
- Live in the application layer, outside the domain

A typical flow:

1. Load aggregate(s) from repository
2. Call domain methods on the aggregate(s)
3. Save aggregate(s) through repository
4. Publish domain events
5. Return result

### Repositories

Provide a collection-like interface for accessing aggregates. The interface is defined in the domain layer; implementation lives in infrastructure.

Rules:

- One repository per aggregate root. Never for internal entities or value objects.
- Methods feel like collection operations and use ubiquitous language: `find_pending_orders()`, not `find_all("orders", { status: "pending" })`
- Not a DAO. Not a generic CRUD wrapper. Not the ORM.
- The domain layer does not know about databases, SQL, or ORMs.

**In Rust:**

```rust
// Domain layer — no infrastructure dependencies
pub trait OrderRepository: Send + Sync {
    fn find_by_id(&self, id: &OrderId) -> Result<Option<Order>, RepositoryError>;
    fn save(&self, order: &Order) -> Result<(), RepositoryError>;
}
```

For testing, an in-memory implementation is invaluable:

```rust
#[derive(Default)]
pub struct InMemoryOrderRepository {
    orders: std::sync::RwLock<HashMap<OrderId, Order>>,
}
```

### Domain Events

Something that happened in the domain that domain experts care about. Named in past tense using ubiquitous language: `OrderPlaced`, `TaskAssigned`, `MeetingEnded`.

Properties:

- Immutable facts — they happened, they cannot be undone
- First-class part of the domain model
- Enable decoupling between aggregates and between contexts
- Capture what happened and when

When to use:

- Cross-aggregate side effects within a context
- Cross-context integration
- Event sourcing
- Audit trails

**In Rust:**

```rust
#[derive(Debug, Clone)]
pub enum OrderEvent {
    Placed { order_id: OrderId, customer_id: CustomerId },
    ItemAdded { order_id: OrderId, product_id: ProductId },
    Cancelled { order_id: OrderId, reason: String },
}
```

Aggregates collect events during mutations, dispatched after persistence:

```rust
impl Order {
    pub fn take_events(&mut self) -> Vec<OrderEvent> {
        std::mem::take(&mut self.domain_events)
    }
}
```

### Factories

Encapsulate complex object creation. Use when constructing an aggregate requires more than a simple constructor.

In Rust, factory methods on the aggregate root or standalone `fn new(...)` with validation are the idiomatic approach. Builders (`typed-builder` crate) help with complex construction.

---

## Part 3: Rich vs. Anemic Domain Models

### Anemic Domain Model (anti-pattern)

Domain objects are data structures with getters/setters but no behavior. All business logic lives in service classes that manipulate these data bags.

Problems:

- Objects cannot protect their own invariants
- Business rules scatter across service classes, get duplicated
- Procedural code hidden behind a struct veneer
- New code easily violates business rules because there is no encapsulation

### Rich Domain Model

Behavior lives on the objects that own the data. Objects enforce their own invariants and express business operations.

```rust
// Anemic — anyone can break invariants
struct Order { pub status: OrderStatus, pub items: Vec<OrderItem> }
struct OrderService;
impl OrderService {
    fn cancel(order: &mut Order) { order.status = OrderStatus::Cancelled; }
}

// Rich — the object protects itself
struct Order { status: OrderStatus, items: Vec<OrderItem> }
impl Order {
    pub fn cancel(&mut self) -> Result<(), OrderError> {
        match self.status {
            OrderStatus::Shipped => Err(OrderError::CannotCancelShipped),
            _ => { self.status = OrderStatus::Cancelled; Ok(()) }
        }
    }
}
```

**This project uses rich domain models.** Behavior goes on the types that own the data. Services are introduced only at real seams where infrastructure is needed — and those become traits for dependency injection.

---

## Part 4: State Machines

Rust's enum system is exceptional for modeling domain state machines.

### Runtime enum (practical default)

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum TaskState {
    Created,
    Assigned { agent_id: AgentId },
    InProgress { started_at: String },
    Blocked { reason: String },
    Completed { result: String },
    Failed { error: String },
}
```

Validate transitions in methods:

```rust
impl Task {
    pub fn assign(&mut self, agent_id: AgentId) -> Result<(), TaskError> {
        match &self.state {
            TaskState::Created => {
                self.state = TaskState::Assigned { agent_id };
                Ok(())
            }
            _ => Err(TaskError::InvalidTransition),
        }
    }
}
```

### Typestate pattern (compile-time enforcement)

Use generic type parameters so invalid transitions are compile errors:

```rust
pub struct Order<State> { id: OrderId, _state: PhantomData<State> }
pub struct Draft;
pub struct Submitted;

impl Order<Draft> {
    pub fn submit(self) -> Order<Submitted> { /* ... */ }
    // add_item only exists on Draft orders
}
// Order<Submitted> has no add_item method — calling it is a compile error
```

Tradeoffs: Typestate is powerful but makes heterogeneous collections harder. Use the runtime enum approach by default; typestate for critical transitions where compile-time safety justifies the complexity.

---

## Part 5: Architectural Patterns

### Hexagonal Architecture (Ports & Adapters)

The domain sits at the center, defining ports (traits) for what it needs. Adapters implement those ports for specific technologies.

```text
                    ┌─────────────────────┐
   Inbound          │                     │         Outbound
   Adapters ──────► │   Domain + Use Cases │ ◄────── Adapters
   (HTTP, CLI,      │   (Ports = Traits)   │         (Postgres, Redis,
    gRPC, MQ)       │                     │          HTTP clients, MQ)
                    └─────────────────────┘
```

**Dependency rule:** Dependencies point inward. Adapters depend on ports. The domain depends on nothing external.

In Rust, **traits ARE ports**. The domain crate defines trait abstractions. The infrastructure crate provides implementations. The compiler enforces the dependency direction through crate dependencies in `Cargo.toml`.

### CQRS (Command Query Responsibility Segregation)

Separate the write model (commands, full domain, invariants) from the read model (queries, denormalized views, optimized for reads).

Use when:

- Read and write workloads scale differently
- Query complexity distorts the domain model
- You want to optimize reads independently

Don't use for simple CRUD — it adds unnecessary complexity.

### Event Sourcing

Store the sequence of domain events instead of current state. Current state is derived by replaying events.

Benefits: complete audit trail, temporal queries, projections can be rebuilt. Challenges: event schema evolution, eventual consistency, complexity.

Not every context needs it. Use where audit trails and temporal queries have business value.

---

## Part 6: DDD in Rust — Practical Patterns

### Ownership IS the aggregate boundary

Rust's ownership model naturally enforces that child entities are only accessed through the aggregate root. You cannot have two mutable references to the same aggregate — this mirrors the DDD rule that only one transaction modifies an aggregate at a time.

### Traits ARE ports

Trait definitions in the domain crate are ports. Implementations in the infrastructure crate are adapters. The compiler enforces the architecture.

### Enums over inheritance

Where OOP-DDD uses class hierarchies, Rust-DDD uses enums with associated data. The compiler's exhaustive match checking replaces runtime type checks.

### Validation at the boundary

If value object constructors are the only way to create instances, invalid data literally cannot exist in the domain.

### Error handling is first-class

`Result<T, DomainError>` makes failures explicit. Domain errors contain no infrastructure details. Use `thiserror` for domain errors, layer errors per architectural layer.

```rust
#[derive(Debug, Error)]
pub enum OrderError {
    #[error("Cannot modify a non-draft order")]
    CannotModifyNonDraftOrder,

    #[error("Order must have at least one item")]
    EmptyOrder,
}
```

### Dependency injection without frameworks

Rust doesn't need a DI container. Use generics for hot-path code, `Arc<dyn Trait>` for services and repositories:

```rust
// Generic (static dispatch, zero cost)
pub struct OrderService<R: OrderRepository> { repo: R }

// Trait object (dynamic dispatch, simpler types)
pub struct OrderService { repo: Arc<dyn OrderRepository> }
```

Composition happens in `main.rs` — the composition root.

### Module / crate organization

```text
backend/
  crates/
    wiab-core/        # Domain layer. Entities, value objects, aggregates,
                      # domain services, repository traits, domain events.
                      # Dependencies: only thiserror, uuid, serde.
    wiab-app/         # Application layer. Use case orchestration.
                      # Depends on: wiab-core.
    wiab-inf/         # Infrastructure layer. Repository implementations,
                      # external service clients, framework adapters.
                      # Depends on: wiab-core, wiab-app.
    wiab/             # API/entry point. HTTP routes, CLI, config.
                      # Depends on: wiab-app, wiab-inf.
```

Dependencies point inward. The domain crate depends on nothing. This is enforced by `Cargo.toml`.

### Testing

Domain logic is pure Rust with no infrastructure — trivially testable:

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn cannot_cancel_shipped_order() {
        let mut order = /* create shipped order */;
        assert!(matches!(order.cancel(), Err(OrderError::CannotCancelShipped)));
    }
}
```

Use in-memory repository implementations for integration tests. Use `mockall` for trait mocking when needed. Use `proptest` for property-based testing of domain invariants.

### Useful crates

| Crate | Purpose |
|-------|---------|
| `thiserror` | Domain error types |
| `uuid` | Entity and aggregate IDs |
| `serde` | Serialization (DTOs, persistence — not on domain types if avoidable) |
| `nutype` | Newtype derive with validation |
| `derive_more` | Reduce newtype boilerplate |
| `strum` | Enum utilities |
| `mockall` | Test doubles for traits |
| `proptest` | Property-based testing |

---

## Part 7: Modern DDD (2024-2026)

### DDD + Microservices

Bounded Context = service boundary is the accepted heuristic. Context Maps describe service interactions. Anti-Corruption Layers are adapters at service boundaries. Domain Events are the inter-service communication mechanism.

### Functional DDD

Strong trend toward algebraic types for domain modeling (enums, pattern matching, immutability). Rust is a natural fit. Aggregates become functions that take commands and return new state + events. Scott Wlaschin's "Domain Modeling Made Functional" (F#) has been influential, and the patterns map directly to Rust.

### Event Storming

The dominant collaborative modeling technique for domain discovery and Bounded Context identification. Three flavors: Big Picture, Process Modeling, Software Design.

### Team Topologies + DDD

Aligning Bounded Contexts with stream-aligned teams. Team Topologies provides organizational patterns; DDD provides domain decomposition.

### DDD and AI

Well-structured domain models benefit AI-assisted development:

- Rich types and intention-revealing method names carry semantic meaning
- Ubiquitous language helps AI understand intent
- Well-bounded contexts provide focused scope for AI to reason about

---

## Key Sources

- Eric Evans, "Domain-Driven Design" (2003) — the original
- Eric Evans, "DDD Reference" (2015) — free PDF, all patterns summarized
- Vaughn Vernon, "Implementing Domain-Driven Design" (2013)
- Vaughn Vernon, "Effective Aggregate Design" (three-part paper series)
- Scott Wlaschin, "Domain Modeling Made Functional" (2018)
- Luca Palmieri, "Zero to Production in Rust" — clean architecture patterns in Rust
- Alistair Cockburn, Hexagonal Architecture (2005)
- Greg Young, CQRS and Event Sourcing
- Alberto Brandolini, "Introducing EventStorming" (2021)
