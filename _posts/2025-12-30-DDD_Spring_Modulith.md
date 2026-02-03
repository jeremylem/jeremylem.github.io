---
layout: single
title: "Migrating Spring PetClinic to DDD with Spring Modulith and jMolecules"
date: 2025-12-30
categories: [blogging]
---

The classic Spring PetClinic is everyone's first Spring app. Simple, clean, easy to follow. But as codebases grow it might turn differently. Business logic scatters across controllers and services. Changes ripple unpredictably. New developers take months to become productive.

I modernized PetClinic as a reference implementation for Domain-Driven Design using Spring Modulith, jMolecules, ByteBuddy, and ArchUnit. Here's what I learned.

## Why DDD Now?

Eric Evans published the Blue Book in 2003. For years, applying DDD to Spring/Hibernate meant wrestling with anemic domain models, entities reduced to data containers with getters and setters, business logic scattered across service layers.

That's changed. Thanks to [Oliver Drotbohm](https://odrotbohm.de/) and the work on Spring Modulith and jMolecules, we can build rich domain models with proper encapsulation, enforce module boundaries at compile time, and prepare a monolith for eventual microservice extraction, all while keeping Spring productive.

DDD is seeing a renaissance for three reasons:

- **Microservices need boundaries.** Teams discovered that decomposing monoliths without clear domain boundaries leads to distributed monoliths. DDD's Bounded Contexts provide a principled way to define service boundaries.
- **Event-driven architecture.** Modern systems communicate through events. DDD's Domain Events pattern maps directly to event sourcing and message-driven microservices.
- **Better tooling.** Frameworks like Spring Modulith and jMolecules finally make DDD practical in Java without fighting the framework.

## The Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Spring PetClinic                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────┐         ┌──────────────────┐          │
│  │  Owner Module    │         │   Vet Module     │          │
│  │                  │ events  │                  │          │
│  │  - Owner         │────────▶│  - Vet           │          │
│  │  - Pet           │         │  - Specialty     │          │
│  │  - Visit         │         │  - Patient       │          │
│  │  - PetType       │         │    Tracking      │          │
│  └────────┬─────────┘         └────────┬─────────┘          │
│           │                            │                     │
│           ▼                            ▼                     │
│  ┌─────────────────────────────────────────────────┐        │
│  │              Shared Kernel (model)               │        │
│  │         Person, PersonName, NamedEntity          │        │
│  └─────────────────────────────────────────────────┘        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

Modules communicate through events, not direct calls. When you're ready to extract a microservice, the boundaries are already clean.

## The Stack

Four pieces work together:

| Tool | Purpose |
|------|---------|
| **Spring Modulith** | Defines and verifies module boundaries |
| **jMolecules** | DDD building blocks as interfaces (Entity, ValueObject, AggregateRoot) |
| **ByteBuddy** | Weaves JPA annotations at compile time so domain classes stay clean |
| **ArchUnit** | Enforces architecture rules in tests |

## Tactical DDD Building Blocks

### Type-Safe Identifiers

Primitive obsessionm using `Integer` or `Long` for IDs, leads to accidentally mixing different entity IDs. Wrap identifiers in type-safe records:

```java
public record PetId(@Column(name = "id") UUID value) implements Identifier {
    public PetId() { this(UUID.randomUUID()); }
}

public record OwnerId(@Column(name = "id") UUID value) implements Identifier {
    public OwnerId() { this(UUID.randomUUID()); }
}
```

Now you can't pass an `OwnerId` where a `PetId` is expected. Compile-time safety.

### Value Objects

Create immutable Value Objects that encapsulate both data and behavior:

```java
public record BirthDate(@Column(name = "birth_date") LocalDate date) implements ValueObject {

    public BirthDate {
        if (date == null) throw new IllegalArgumentException("Birth date must not be null");
        if (date.isAfter(LocalDate.now())) throw new IllegalArgumentException("Birth date cannot be in the future");
    }

    public int getAgeInYears() {
        return Period.between(date, LocalDate.now()).getYears();
    }

    public boolean isElderly() { return getAgeInYears() >= 7; }
    public boolean isPuppy() { return getAgeInYears() < 1; }
}
```

Self-validating. Behavior lives with the data. No more validation logic scattered across services.

### Entities and Aggregates

An **Entity** has identity that persists across state changes. An **Aggregate** is a cluster of entities treated as a single unit. One entity is the **Aggregate Root**—all access goes through it.

```java
// Owner is the Aggregate Root
public class Owner extends Person implements AggregateRoot<Owner, OwnerId> {
    private OwnerId id = new OwnerId();
    private Set<Pet> pets = new LinkedHashSet<>();

    public void addPet(Pet pet) {
        pets.add(pet);
    }

    public Pet getPet(String name) {
        return pets.stream()
            .filter(p -> p.getName().equals(name))
            .findFirst()
            .orElse(null);
    }
}

// Pet is an Entity within the Owner aggregate
public class Pet extends NamedEntity implements Entity<Owner, PetId> {
    private PetId id = new PetId();
    private BirthDate birthDateValue;
    private Set<Visit> visits = new LinkedHashSet<>();
}
```

Rules: Only the Aggregate Root has a repository. External objects reference the aggregate by ID only. Invariants are enforced within the aggregate boundary.

### Cross-Aggregate References with Association

Direct references between aggregates create tight coupling. If `Pet` holds a direct reference to `PetType`, changes to `PetType` can break `Pet`. Worse, JPA eagerly loads the entire object graph.

Use `Association<T, ID>` to hold only the ID reference:

```java
public class Pet implements Entity<Owner, PetId> {
    // Don't do this - crosses aggregate boundary
    // private PetType type;

    // Do this - store only the reference
    private Association<PetType, PetTypeId> type;

    public void setType(PetType type) {
        this.type = type != null ? Association.forAggregate(type) : null;
    }

    public PetTypeId getTypeId() {
        return this.type != null ? this.type.getId() : null;
    }
}
```

#### Resolving Associations with AssociationResolver

When you need the actual `PetType` object, resolve it explicitly using `AssociationResolver`:

```java
public class Pet implements Entity<Owner, PetId> {
    private Association<PetType, PetTypeId> type;

    // Resolve when needed - caller provides the resolver
    public PetType resolveType(AssociationResolver<PetType, PetTypeId> resolver) {
        return this.type != null ? resolver.resolve(this.type).orElse(null) : null;
    }
}
```

The repository implements `AssociationResolver`:

```java
@Repository
public interface PetTypeRepository
        extends JpaRepository<PetType, PetTypeId>,
                AssociationResolver<PetType, PetTypeId> {
}
```

Usage in application service:

```java
@Service
public class PetApplicationService {
    private final PetTypeRepository petTypes;

    public PetTypeInfo getPetTypeInfo(Pet pet) {
        PetType type = pet.resolveType(petTypes);  // Explicit resolution
        return new PetTypeInfo(type.getName());
    }
}
```

#### Why This Pattern Matters

| Benefit | Explanation |
|---------|-------------|
| **Aggregate boundaries respected** | `Pet` doesn't hold a direct reference to `PetType`—only its ID |
| **Type safety** | `AssociationResolver<PetType, PetTypeId>` ensures you can't accidentally resolve to wrong type |
| **Explicit dependencies** | Resolution requires injecting the resolver—no hidden database calls |
| **Testable** | Easy to mock `AssociationResolver` in unit tests |
| **Lazy loading** | Associations are resolved only when explicitly requested, not eagerly by JPA |
| **jMolecules standard** | Follows the framework's best practices for cross-aggregate references |

#### Testing with Mocked Resolver

```java
@Test
void shouldResolvePetType() {
    PetType dog = new PetType("Dog");
    Pet pet = new Pet();
    pet.setType(dog);

    // Mock the resolver
    AssociationResolver<PetType, PetTypeId> resolver = mock(AssociationResolver.class);
    when(resolver.resolve(any())).thenReturn(Optional.of(dog));

    PetType resolved = pet.resolveType(resolver);

    assertThat(resolved.getName()).isEqualTo("Dog");
}
```

No database needed. The domain logic is fully testable in isolation.

### Domain Events

Domain Events are the key to decoupling modules. When something significant happens in one module, it publishes an event. Other modules react without knowing about each other.

#### Defining Events

Events are immutable records of something that happened. Use past tense naming—`PetAdopted`, not `AdoptPet`:

```java
public record PetAdoptedEvent(
    PetId petId,
    PetTypeId petTypeId,
    OwnerId ownerId,
    LocalDate adoptionDate
) implements DomainEvent {

    public static PetAdoptedEvent of(PetId petId, PetTypeId petTypeId, OwnerId ownerId) {
        return new PetAdoptedEvent(petId, petTypeId, ownerId, LocalDate.now());
    }
}
```

Include only the data consumers need. Use IDs, not full entities—keeps events lightweight and avoids coupling to aggregate internals.

#### Publishing Events: Two Patterns

**Pattern 1: From the Aggregate (Pure DDD)**

The aggregate registers events internally. Spring Data publishes them when the aggregate is saved:

```java
public class Owner extends AbstractAggregateRoot<Owner>
        implements AggregateRoot<Owner, OwnerId> {

    public void addPet(Pet pet) {
        pets.add(pet);
        // Register event - published after save() completes
        registerEvent(PetAdoptedEvent.of(pet.getId(), pet.getTypeId(), this.id));
    }
}
```

The event is published automatically when `ownerRepository.save(owner)` commits. No explicit publisher needed.

**Pattern 2: From the Application Service (Pragmatic)**

The application service publishes events explicitly:

```java
@Service
public class PetApplicationService {
    private final OwnerRepository owners;
    private final ApplicationEventPublisher events;

    @Transactional
    public void adoptPet(OwnerId ownerId, Pet pet) {
        Owner owner = owners.findById(ownerId).orElseThrow();
        owner.addPet(pet);
        owners.save(owner);

        events.publishEvent(PetAdoptedEvent.of(pet.getId(), pet.getTypeId(), ownerId));
    }
}
```

**When to use which:**

| Pattern | Use When |
|---------|----------|
| Aggregate | Event is intrinsic to domain logic; you want pure domain model |
| Application Service | Event depends on application context; you need more control over timing |

#### Subscribing to Events

Use `@ApplicationModuleListener` for cross-module event handling:

```java
@Service
class VetPatientTrackingService {

    @ApplicationModuleListener
    void onPetAdopted(PetAdoptedEvent event) {
        log.info("New patient registered: Pet ID=" + event.petId());
        // Update vet module's view of patients
    }
}
```

`@ApplicationModuleListener` is Spring Modulith's annotation that combines `@EventListener` with `@Async` and `@Transactional`. It runs in a separate transaction after the publishing transaction commits.

#### Transactional Event Handling

Understanding transaction boundaries is critical:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Publishing Transaction                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. owner.addPet(pet)                                         │   │
│  │ 2. ownerRepository.save(owner)                               │   │
│  │ 3. Event stored in publication registry (if enabled)        │   │
│  │ 4. COMMIT                                                     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Listener Transaction (separate)                              │   │
│  │ 5. @ApplicationModuleListener receives event                 │   │
│  │ 6. Listener does its work                                    │   │
│  │ 7. COMMIT (or ROLLBACK - doesn't affect publisher)          │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

Key insight: listener failures don't roll back the publishing transaction. The pet is adopted even if the vet notification fails. This is usually what you want—but you need to handle listener failures.

#### Reliable Event Publication

What happens if the listener fails? Or the application crashes between publishing and processing? Spring Modulith's **Event Publication Registry** solves this.

Add the dependency and configure a database-backed registry:

```xml
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-jpa</artifactId>
</dependency>
```

```java
@Configuration
class ModulithConfig {

    @Bean
    ApplicationRunner eventPublicationRegistrar(EventPublicationRegistry registry) {
        return args -> {
            // On startup, retry any incomplete publications
            registry.resubmitIncompletePublications();
        };
    }
}
```

How it works:

```
┌─────────────────────────────────────────────────────────────────────┐
│  1. Event Published                                                  │
│     └─> Stored in EVENT_PUBLICATION table with status INCOMPLETE    │
│                                                                      │
│  2. Listener Invoked                                                 │
│     └─> If SUCCESS: Mark publication COMPLETED                      │
│     └─> If FAILURE: Publication remains INCOMPLETE                  │
│                                                                      │
│  3. On Application Restart                                           │
│     └─> resubmitIncompletePublications() retries failed events      │
└─────────────────────────────────────────────────────────────────────┘
```

The registry guarantees at-least-once delivery. Listeners must be idempotent—they might receive the same event twice after a crash recovery.

#### Async vs Sync Processing

By default, `@ApplicationModuleListener` is async—listeners run in a separate thread after the transaction commits. For synchronous processing within the same transaction:

```java
@TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
void onPetAdoptedSync(PetAdoptedEvent event) {
    // Runs in same transaction as publisher
    // If this fails, the whole transaction rolls back
}
```

Use sync listeners sparingly. They couple the modules more tightly—a listener failure affects the publisher.

#### Error Handling in Listeners

Async listeners need explicit error handling:

```java
@ApplicationModuleListener
void onPetAdopted(PetAdoptedEvent event) {
    try {
        vetPatientService.registerPatient(event.petId());
    } catch (Exception e) {
        log.error("Failed to register patient for pet {}", event.petId(), e);
        // Options:
        // 1. Rethrow - event stays INCOMPLETE, retried on restart
        // 2. Swallow - event marked COMPLETED, lost
        // 3. Send to dead letter queue for manual handling
        throw e;  // Prefer rethrowing for automatic retry
    }
}
```

#### Exposing Events as a Module API

Events are the public API between modules. Expose them through a named interface:

```
owner/
├── package-info.java          # @ApplicationModule
├── Owner.java                 # Internal
├── OwnerRepository.java       # Internal
└── events/
    ├── package-info.java      # Named interface
    └── PetAdoptedEvent.java   # Public API
```

Other modules declare dependency on events only:

```java
@ApplicationModule(
    displayName = "Vet Management",
    allowedDependencies = { "model", "owner::events" }
)
package org.springframework.samples.petclinic.vet;
```

The vet module can listen to `PetAdoptedEvent` but cannot access `Owner`, `OwnerRepository`, or any other internal class. True decoupling.

#### Testing Events

Spring Modulith provides testing support:

```java
@ApplicationModuleTest
class OwnerModuleTests {

    @Test
    void petAdoptionPublishesEvent(Scenario scenario) {
        scenario.stimulate(() -> petService.adoptPet(ownerId, pet))
            .andWaitForEventOfType(PetAdoptedEvent.class)
            .matching(event -> event.petId().equals(pet.getId()))
            .toArriveAndVerify(event -> {
                assertThat(event.ownerId()).isEqualTo(ownerId);
            });
    }
}
```

The `Scenario` API lets you verify events are published with the right data, without coupling tests to listener implementations.

## Spring Modulith: Defining Boundaries

Use `package-info.java` to declare modules and their allowed dependencies:

```java
@ApplicationModule(
    displayName = "Owner Management",
    allowedDependencies = "model"
)
package org.springframework.samples.petclinic.owner;
```

```java
@ApplicationModule(
    displayName = "Vet Management",
    allowedDependencies = { "model", "owner::events" }  // Named interface - only events subpackage
)
package org.springframework.samples.petclinic.vet;
```

The `owner::events` syntax is a **named interface**—it exposes only the `events` subpackage while keeping `Owner`, `OwnerRepository`, and other internals hidden. Combined with the event publication registry described above, this creates truly independent modules that communicate only through events.

Verify structure in tests:

```java
class ModulithStructureTest {
    ApplicationModules modules = ApplicationModules.of("org.springframework.samples.petclinic");

    @Test
    void verifiesModularStructure() {
        modules.verify();  // Fails if boundaries are violated
    }

    @Test
    void generateDocumentation() {
        new Documenter(modules)
            .writeModulesAsPlantUml()
            .writeIndividualModulesAsPlantUml();
    }
}
```

## ArchUnit: Enforcing Architecture

Spring Modulith uses ArchUnit under the hood. The `modules.verify()` call checks:
- No cycles between modules
- Modules only access their declared dependencies
- Internal packages are not accessed from outside

jMolecules adds DDD-specific rules and layered architecture enforcement:

```
┌─────────────────────────────────────────────────────────┐
│                   @InterfaceLayer                        │
│              Controllers, REST endpoints                 │
├─────────────────────────────────────────────────────────┤
│                   @ApplicationLayer                      │
│           Application services, use cases                │
├─────────────────────────────────────────────────────────┤
│                     @DomainLayer                         │
│         Entities, Value Objects, Domain Services         │
├─────────────────────────────────────────────────────────┤
│                 @InfrastructureLayer                     │
│            Repositories, external services               │
└─────────────────────────────────────────────────────────┘

        Dependencies flow DOWN only (enforced by ArchUnit)
```

Run all rules in a test:

```java
@AnalyzeClasses(packages = "org.springframework.samples.petclinic")
public class JMoleculesRulesUnitTest {

    @ArchTest
    ArchRule dddRules = JMoleculesDddRules.all();

    @ArchTest
    ArchRule layeredArchitecture = JMoleculesArchitectureRules.ensureLayering();
}
```

When a rule is violated, your build fails:

```
java.lang.AssertionError: Architecture Violation [Priority: MEDIUM] -
Rule 'classes that implement Entity should have identity' was violated (1 times):
    Class Pet does not have an @Id annotated field
```

## ByteBuddy: Keeping Domain Classes Clean

The magic that makes this work smoothly. ByteBuddy weaves JPA annotations at compile time based on jMolecules interfaces. Your domain classes stay clean—no `@Entity`, no `@Id` annotations polluting the model.

Configure the Maven plugin:

```xml
<plugin>
    <groupId>net.bytebuddy</groupId>
    <artifactId>byte-buddy-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals><goal>transform-extended</goal></goals>
        </execution>
    </executions>
    <configuration>
        <classPathDiscovery>true</classPathDiscovery>
    </configuration>
</plugin>
```

Write clean domain classes. ByteBuddy adds the JPA infrastructure.

## The Path to Microservices

This architecture is a stepping stone:

```
                     Monolith with Modules
┌─────────────────────────────────────────────────────────┐
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Owner     │  │     Vet     │  │   Billing   │     │
│  │  Context    │──│   Context   │──│   Context   │     │
│  │  (Module)   │  │  (Module)   │  │  (Module)   │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│         │                │                │             │
│         └────── Events ──┴──── Events ────┘             │
└─────────────────────────────────────────────────────────┘
                            │
                            │ Extract when ready
                            ▼
                    Microservices
┌─────────────────┐  ┌─────────────┐  ┌─────────────┐
│   Owner         │  │     Vet     │  │   Billing   │
│  Service        │──│   Service   │──│   Service   │
└─────────────────┘  └─────────────┘  └─────────────┘
        │                   │                   │
        └────── Kafka/RabbitMQ ─────────────────┘
```

| DDD Concept | Monolith | Microservices |
|-------------|----------|---------------|
| Bounded Context | Spring Modulith Module | Separate Service |
| Aggregate | Transactional boundary | Service boundary |
| Domain Event | `ApplicationEventPublisher` | Kafka/RabbitMQ message |
| Anti-Corruption Layer | Module adapter | API Gateway / BFF |

Module boundaries are already defined and verified. Events decouple modules. Aggregate boundaries map naturally to service boundaries. Type-safe IDs prevent accidental coupling. When a module needs independent scaling, extract it—the work is already done.

## When NOT to Use DDD

DDD is not free. It adds concepts, abstractions, and ceremony.

| Scenario | Why DDD Is Overkill |
|----------|---------------------|
| **Simple CRUD apps** | If your app is mostly forms over data with little business logic, a simple layered architecture suffices. |
| **Short-lived projects** | Prototypes, MVPs, or throwaway code don't benefit from the upfront investment. |
| **Small teams without domain experts** | DDD assumes collaboration with domain experts. Without them, you're guessing. |
| **Stable, simple domains** | If the domain is unlikely to change, the flexibility DDD provides isn't needed. |

The pragmatic approach: Use DDD **tactically** for complex domain logic. Use DDD **strategically** when you have multiple teams or are planning microservices.

## Project Setup

```xml
<!-- Spring Modulith -->
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-core</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- jMolecules DDD -->
<dependency>
    <groupId>org.jmolecules</groupId>
    <artifactId>jmolecules-ddd</artifactId>
</dependency>
<dependency>
    <groupId>org.jmolecules</groupId>
    <artifactId>jmolecules-layered-architecture</artifactId>
</dependency>
<dependency>
    <groupId>org.jmolecules.integrations</groupId>
    <artifactId>jmolecules-jpa</artifactId>
</dependency>

<!-- ByteBuddy for JPA annotation weaving -->
<dependency>
    <groupId>org.jmolecules.integrations</groupId>
    <artifactId>jmolecules-bytebuddy-nodep</artifactId>
    <scope>provided</scope>
</dependency>

<!-- ArchUnit for architecture verification -->
<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit-junit5</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.jmolecules.integrations</groupId>
    <artifactId>jmolecules-archunit</artifactId>
    <scope>test</scope>
</dependency>
```

## Quick Reference

| Pattern | jMolecules Type | Purpose |
|---------|-----------------|---------|
| Aggregate Root | `AggregateRoot<T, ID>` | Entry point to aggregate, owns repository |
| Entity | `Entity<AggregateRoot, ID>` | Has identity, belongs to aggregate |
| Value Object | `ValueObject` | Immutable, equality by value |
| Identifier | `Identifier` | Type-safe ID wrapper |
| Association | `Association<T, ID>` | Cross-aggregate reference (ID only) |
| Domain Event | `DomainEvent` | Notification of state change |
| Repository | `Repository<T, ID>` | Aggregate persistence |

## Running the Project

```bash
git clone https://github.com/jeremylem/petclinic-exploration.git
cd petclinic-exploration
./mvnw spring-boot:run
```

Access at http://localhost:8080

## References

- [Domain-Driven Design](https://www.domainlanguage.com/ddd/) — Eric Evans (2003)
- [Implementing DDD Building Blocks in Java](https://odrotbohm.de/2020/03/Implementing-DDD-Building-Blocks-in-Java/) — Oliver Drotbohm
- [Spring Modulith Reference](https://docs.spring.io/spring-modulith/reference/)
- [jMolecules GitHub](https://github.com/xmolecules/jmolecules)
- [Tactical DDD Workshop](https://github.com/odrotbohm/tactical-ddd-workshop)
