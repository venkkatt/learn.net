# Saga Pattern in .NET 8 - Interview Guide

## PART 1: UNDERSTANDING THE SAGA PATTERN

### 1.1 What is the Saga Pattern?

"The Saga Pattern is a design pattern for managing distributed transactions across microservices. Here's the problem it solves:

**TRADITIONAL MONOLITH:**
- Single database
- ACID transactions guaranteed
- All-or-nothing guarantee

**MICROSERVICES PROBLEM:**
- Multiple databases (one per service)
- Can't use traditional ACID transactions
- Need to maintain data consistency across services
- If one service fails, need to rollback previous services

**Example:**
When a customer places an order, we need to:
1. Reserve inventory
2. Process payment
3. Create shipment

If payment fails → we must undo inventory reservation.

**The Saga Solution:**
Break the transaction into smaller local transactions, each within a single service's database. If any fails, execute compensating transactions to undo previous steps. It's like a distributed transaction without 2PC (Two-Phase Commit)."

### 1.2 Key Characteristics

| Characteristic | Description |
|---|---|
| Sequence of Local Transactions | Each step is an independent transaction in one service |
| Event/Message Driven | Services communicate via events or messages |
| Compensating Transactions | Rollback actions if a step fails |
| Eventual Consistency | Data becomes consistent eventually, not immediately |
| No Central Lock | No blocking - services work independently |

## PART 2: TWO APPROACHES TO SAGA IMPLEMENTATION

### 2.1 Choreography vs Orchestration

#### Choreography
- Each microservice listens for events and emits events, with no central coordinator.
- Pros: Very decoupled, simple flows.
- Cons: Hard to track flow for complex processes, debugging is difficult.

#### Orchestration
- Uses a central saga (orchestrator) to coordinate all steps and compensations explicitly.
- Pros: Explicit workflow, clearer for debugging, better for complex flows, easy to test.
- Cons: The orchestrator can be a bottleneck or a single point of failure.

## PART 3: .NET 8 IMPLEMENTATION EXAMPLE

### Orchestration-Based Saga (Recommended)

**Scenario:** Order involves Inventory, Payment, and Shipping services.
If any step fails, previous steps are compensated by sending specific compensation commands.

**Key Implementation:**
- Use libraries like MassTransit (or NServiceBus) with RabbitMQ/Azure Service Bus for messaging.
- Saga state is stored in a relational DB (e.g. via Entity Framework Core).

**Events, States, and Commands Example:**
- Order Service starts the saga.
- Orchestrator sends ReserveInventory, then ProcessPayment, then CreateShipment.
- Services reply with InventoryReserved, PaymentProcessed, ShipmentCreated events.
- Compensation: ReleaseInventory, RefundPayment, CancelShipment commands sent on failure.

**Core Components:**
1. Commands (ReserveInventory, ProcessPayment, etc.)
2. Events (InventoryReserved, PaymentProcessed, etc.)
3. Compensation commands/events
4. Saga State (Order/Payment/Inventory/Shipment info and saga status)
5. Consumers and orchestrator logic

**Note:**
- Add idempotency (OrderId/CorrelationId as unique key for retries).
- Persist saga state for recovery and reliability.

### Choreography-Based Saga (For Simple Flows)
- Services react to events from the previous steps – harder to debug and trace.

## PART 4: INTERVIEW Q&A - CRITICAL QUESTIONS

**1. Explain the Saga Pattern and how it solves distributed transactions**

The Saga Pattern solves the problem of distributed transactions in microservices by splitting them into a sequence of smaller, local transactions. Each services updates its own DB; if any step fails, compensation/rollback logic ensures eventual consistency.

**2. Orchestration vs Choreography - which is better?**

- Orchestration: Central orchestrator manages the flow and compensation. Better for complex, production-grade scenarios.
- Choreography: Each service listens and reacts to events; best for simple, linear workflows.

**3. How do you handle eventual consistency and stale data?**

- With idempotent operations, retry logic, event versioning, and audit trails (event sourcing)

**4. How to test a saga?**

- Unit test saga logic/state-machines
- Integration test with message brokers and fake consumers

**5. How do you handle saga persistence and recovery?**

- Store saga state in DB (EF Core with MassTransit/NServiceBus support)
- On crash/restart, resume from last saved state, using correlation IDs to ensure safe recovery.

## PART 5: WHEN TO USE SAGA PATTERN

**Use Saga Pattern When:**
- Multiple microservices should participate in business process
- You need to coordinate distributed updates but can't have immediate consistency
- Business rollback/compensation is required

**Avoid Saga Pattern If:**
- You have a single service/database (can use local transactions)
- Real-time, strong consistency with no delay is required

## PART 6: INTERVIEW TIPS

- Mention distributed transaction problem and why ACID/2PC is not possible in microservices
- Be clear about eventual consistency model
- Be ready to explain orchestration and choreography and say why orchestration is usually better for larger systems
- Reference idempotency, message brokers (RabbitMQ, Azure Service Bus), and libraries (MassTransit, NServiceBus)

---
