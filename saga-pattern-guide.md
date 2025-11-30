# Event-Driven Saga Pattern with Azure Service Bus and .NET

## Table of Contents
1. [What is the Saga Pattern?](#what-is-the-saga-pattern)
2. [SAGA Abbreviation](#saga-abbreviation)
3. [Saga Pattern Libraries](#saga-pattern-libraries)
4. [Saga Orchestrator Deployment](#saga-orchestrator-deployment)
5. [Service Communication: Sequential vs Parallel](#service-communication-sequential-vs-parallel)
6. [Implementation Guide](#implementation-guide)

---

## What is the Saga Pattern?

The **Saga Pattern** is a design pattern for managing **distributed transactions** across microservices. It breaks down a large transaction into a sequence of smaller, independent local transactions that can be compensated if a failure occurs.

### Key Concept
Instead of using traditional ACID transactions (which are difficult in distributed systems), the Saga pattern ensures **eventual consistency** by:
1. Breaking down a business process into multiple local transactions
2. Each step updates its own database and triggers the next step via events/messages
3. If a step fails, compensating transactions undo previous changes

### Why Not Use Traditional Transactions?

In microservices, traditional **two-phase commit** is problematic because:
- Requires strict ACID compliance across multiple databases
- Creates high coupling and coordination overhead
- Services often aren't available simultaneously
- Doesn't align with microservices independence principle

---

## Two Implementation Approaches

### 1. Orchestration (Centralized)

**How it works:** A central **Saga Orchestrator** coordinates the entire workflow, telling each service what to do in sequence.

**Characteristics:**
- ✅ Easier to understand and debug (centralized control flow)
- ✅ Clear ordering of steps
- ✅ Simpler individual service logic
- ✅ Better for complex workflows with dependencies
- ❌ Orchestrator is a single point of failure
- ❌ More complex to implement initially

**Use Case:** Order processing with sequential steps (Reserve Inventory → Process Payment → Ship Order)

### 2. Choreography (Event-Driven)

**How it works:** Each service listens to events from others and acts independently, publishing new events when done.

**Characteristics:**
- ✅ Decentralized and loosely coupled
- ✅ No single point of failure
- ✅ Easier to incrementally adopt
- ❌ Difficult to track overall workflow
- ❌ Risk of cyclic dependencies
- ❌ Harder to debug

**Use Case:** Simple workflows with independent services

---

## SAGA Abbreviation

**SAGA** stands for:
- **S**equence **A**cross **G**eographically-distributed **A**pplications

However, some interpretations define it as:
- **S**teps **A**cross **G**eographic **A**rchitectures
- Simply a design pattern named after the literary term "saga" (a long narrative of interconnected events)

**Origin:** The term was coined by Hector Garcia-Molina and Kenneth Salem in 1987 to describe long-lived transactions. In modern distributed systems, it represents a sequence of local transactions that maintain eventual consistency across services.

---

## Saga Pattern Libraries

Several libraries provide excellent saga pattern implementations for .NET:

| Library | Best For | Messaging Support | Learning Curve |
|---------|----------|-------------------|-----------------|
| **MassTransit** | Production-grade orchestration & choreography | RabbitMQ, Azure Service Bus, AWS SQS, Kafka | Medium |
| **NServiceBus** | Enterprise saga patterns with licensing | Specific transports | Medium-High |
| **Brighter** | Command Query Responsibility Segregation (CQRS) | Multiple transports | Medium |
| **Azure Durable Functions** | Serverless orchestration | Azure Service Bus, Queues | Low-Medium |
| **Rebus** | Lightweight, open-source | RabbitMQ, Azure Service Bus | Low |
| **Akka.NET** | Actor-based distributed computing | Native support | High |

**Recommendation:** MassTransit with Azure Service Bus is the most production-ready and flexible choice for enterprise .NET applications.

---

## Saga Orchestrator Deployment

### Is the Saga Orchestrator a Separate Application?

**YES - the orchestrator is typically deployed as a separate microservice/application.**

### Why Separate Application?
- ✅ Single Responsibility Principle
- ✅ Independent scaling
- ✅ Easier testing and debugging
- ✅ Can be reused across multiple services
- ✅ Clearer dependency graph
- ❌ Adds deployment complexity

### Deployment Options:

**1. As a Separate Microservice (Recommended)**
```
Service 1: OrderService.API
Service 2: OrderSaga.Orchestrator (Separate App)
Service 3: InventoryService.API
Service 4: PaymentService.API
Service 5: ShippingService.API
```

**2. As Part of a Larger Service**
```
Service 1: OrderService.API (contains orchestrator)
Service 2: InventoryService.API
Service 3: PaymentService.API
```

**3. Hybrid Approach**
```
Service 1: OrderService.API (initiates saga, contains some logic)
Service 2: OrderSaga.Orchestrator (separate, coordinates complex workflows)
Service 3: Other services...
```

### Complete Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CLIENT / API GATEWAY                             │
└──────────────────────────┬──────────────────────────────────────────────┘
                           │ POST /api/orders/place-order
                           ↓
┌──────────────────────────────────────────────────────────────────────────┐
│                    ORDER SERVICE (Microservice 1)                         │
│ - Validates order                                                        │
│ - Publishes OrderCreated event to Azure Service Bus Topic               │
└──────────────────────────┬──────────────────────────────────────────────┘
                           │
        ┌──────────────────┴──────────────────┐
        │                                     │
        ↓                                     ↓
   ┌─────────────────────────────────────────────────────┐
   │      AZURE SERVICE BUS (Message Broker)             │
   │  - Topic: OrderCreated                              │
   │  - Subscriptions: OrderSaga, InventoryService      │
   └──────────┬──────────────────────────────────────────┘
              │
    ┌─────────┼──────────┬──────────┐
    │         │          │          │
    ↓         ↓          ↓          ↓
┌───────────────────┐ ┌─────────────────────────────────────┐
│ Inventory Service │ │  ORDER SAGA ORCHESTRATOR (Separate) │
│                   │ │  - Tracks OrderSagaData state       │
│ Consumers:        │ │  - Coordinates workflow             │
│ - ReserveInventory│ │  - Manages compensation             │
│ - ReleaseInventory│ │  - Persists to SQL Database         │
│                   │ └─────────────────────────────────────┘
└───────────────────┘

    │ InventoryReserved         │
    │ Event Published           │
    │                           │
    └──────────────────────────→ OrderSaga receives event
                                and sends ProcessPaymentCommand
                                to queue:process-payment
                                │
                                ↓
┌──────────────────────────────────────────────────────────────┐
│              PAYMENT SERVICE (Microservice 3)                 │
│ - Receives ProcessPaymentCommand from Service Bus           │
│ - Processes payment                                          │
│ - Publishes PaymentProcessed or PaymentFailed              │
└──────────────────────────────────────────────────────────────┘

    │ PaymentProcessed
    │ Event Published to Service Bus
    │
    └──────────────────────────→ OrderSaga receives event
                                and sends ScheduleShippingCommand
                                to queue:schedule-shipping
                                │
                                ↓
┌──────────────────────────────────────────────────────────────┐
│             SHIPPING SERVICE (Microservice 4)                 │
│ - Receives ScheduleShippingCommand                          │
│ - Schedules shipping                                        │
│ - Publishes ShippingScheduled or ShippingFailed            │
└──────────────────────────────────────────────────────────────┘

    │ ShippingScheduled
    │ Event Published to Service Bus
    │
    └──────────────────────────→ OrderSaga receives event
                                Publishes OrderCompleted
                                Marks saga as COMPLETE
```

---

## Service Communication: Sequential vs Parallel

### Quick Answer

**Order Service DOES NOT directly communicate with Inventory, Payment, and other services.** Instead:

- **Order Service** publishes events to **Azure Service Bus**
- **Order Saga Orchestrator** (separate app) receives these events
- **Saga Orchestrator coordinates the flow** - it can execute steps **sequentially or in parallel**
- Individual services listen to commands/events and publish results back

**Communication Pattern:** Asynchronous event-driven (NOT direct service-to-service calls)

### Execution Models

#### 1. Sequential Execution (Default)

**How it works:** Each step waits for the previous one to complete before starting.

```
Step 1: Reserve Inventory (100ms)
    ↓ (waits for completion)
Step 2: Process Payment (150ms)
    ↓ (waits for completion)
Step 3: Schedule Shipping (200ms)

Total Time: 450ms
```

**Use Case:** When later steps depend on earlier ones (e.g., Can't charge payment without reserved inventory)

**Timeline:**
```
T=0ms    : OrderCreated event received
T=0ms    : Send ReserveInventoryCommand
T=100ms  : InventoryReserved event received ✓
T=100ms  : Send ProcessPaymentCommand
T=250ms  : PaymentProcessed event received ✓
T=250ms  : Send ScheduleShippingCommand
T=450ms  : ShippingScheduled event received ✓
T=450ms  : Order Completed
```

#### 2. Parallel Execution

**How it works:** Multiple independent steps execute simultaneously.

```
Step 1: Reserve Inventory (100ms)  ─┐
Step 2: Process Payment (150ms)    ─┼→ Wait for ALL to complete
Step 3: Schedule Shipping (200ms)  ─┘

Total Time: 250ms (max of all steps, not sum)
```

**Use Case:** When steps are independent and don't depend on each other

**Timeline:**
```
T=0ms    : OrderCreated event received
T=0ms    : Send ReserveInventoryCommand         ─┐
T=0ms    : Send ProcessPaymentCommand           ├→ All sent in parallel
T=0ms    : Send ScheduleShippingCommand         ─┘
T=100ms  : InventoryReserved event received ✓ (1/3 done)
T=150ms  : PaymentProcessed event received ✓ (2/3 done)
T=200ms  : ShippingScheduled event received ✓ (3/3 done)
T=200ms  : Order Completed (all parallel steps done)
```

#### 3. Hybrid Approach (Most Realistic)

**How it works:** Some steps in parallel, some sequential.

```
Order Created
    ↓
Send Inventory + Payment in PARALLEL
    ↓ (wait for both)
Then Schedule Shipping (depends on both)
    ↓
Complete
```

**Scenario:**
- Inventory and Payment are independent → run in parallel
- Shipping depends on both → wait for both before shipping

**Timeline for Hybrid Approach:**
```
T=0ms    : OrderCreated event received
T=0ms    : Send ReserveInventoryCommand (PARALLEL START) ─┐
T=0ms    : Send ProcessPaymentCommand (PARALLEL START)   ├→ Phase 1: Parallel
T=100ms  : InventoryReserved ✓                           │
T=150ms  : PaymentProcessed ✓                            ─┘
T=150ms  : Send ScheduleShippingCommand (SEQUENTIAL)     ← Phase 2: Sequential
T=350ms  : ShippingScheduled ✓
T=350ms  : Order Completed
```

### Visual Comparison

| Aspect | Sequential | Parallel | Hybrid |
|--------|-----------|----------|--------|
| **Execution** | One step at a time | All steps simultaneously | Mixed approach |
| **Total Time** | Sum of all steps | Max step duration | Sum of phases |
| **Use Case** | Dependent steps | Independent steps | Realistic scenario |
| **Example Time** | 100+150+200 = 450ms | max(100,150,200) = 200ms | (100\|150)+200 = 350ms |
| **Complexity** | Simple to implement | More complex | Medium complexity |
| **Failure Handling** | Easy (linear flow) | Complex (multiple failure points) | Moderate |

---

## Implementation Guide

### Step 1: Install NuGet Packages

```bash
# For all microservices
dotnet add package MassTransit
dotnet add package MassTransit.Azure.ServiceBus.Core
dotnet add package Azure.Messaging.ServiceBus

# For saga orchestrator
dotnet add package MassTransit.Persistence.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
```

### Step 2: Define Shared Events/Messages

**File: SharedMessages/OrderEvents.cs**

```csharp
namespace SharedMessages.Events
{
    // Commands (Requests)
    public interface CreateOrderCommand
    {
        public Guid OrderId { get; }
        public string CustomerId { get; }
        public string ProductId { get; }
        public int Quantity { get; }
        public decimal Amount { get; }
    }

    public interface ReserveInventoryCommand
    {
        public Guid OrderId { get; }
        public string ProductId { get; }
        public int Quantity { get; }
    }

    public interface ProcessPaymentCommand
    {
        public Guid OrderId { get; }
        public string CustomerId { get; }
        public decimal Amount { get; }
    }

    public interface ScheduleShippingCommand
    {
        public Guid OrderId { get; }
        public string CustomerId { get; }
    }

    // Events (Results)
    public interface OrderCreated
    {
        public Guid OrderId { get; }
        public string CustomerId { get; }
        public string ProductId { get; }
        public int Quantity { get; }
        public decimal Amount { get; }
    }

    public interface InventoryReserved
    {
        public Guid OrderId { get; }
        public string ProductId { get; }
        public int Quantity { get; }
    }

    public interface InventoryReservationFailed
    {
        public Guid OrderId { get; }
        public string Reason { get; }
    }

    public interface PaymentProcessed
    {
        public Guid OrderId { get; }
        public decimal Amount { get; }
    }

    public interface PaymentFailed
    {
        public Guid OrderId { get; }
        public string Reason { get; }
    }

    public interface ShippingScheduled
    {
        public Guid OrderId { get; }
    }

    public interface ShippingFailed
    {
        public Guid OrderId { get; }
        public string Reason { get; }
    }

    public interface InventoryReleased
    {
        public Guid OrderId { get; }
    }

    public interface PaymentRefunded
    {
        public Guid OrderId { get; }
    }

    public interface ShippingCancelled
    {
        public Guid OrderId { get; }
    }

    public interface OrderCompleted
    {
        public Guid OrderId { get; }
    }

    public interface OrderFailed
    {
        public Guid OrderId { get; }
        public string Reason { get; }
    }
}
```

### Step 3: Define Saga State

**File: Orchestrator/OrderSagaData.cs**

```csharp
namespace OrderSaga.Orchestrator
{
    using MassTransit;
    using System;

    public class OrderSagaData : SagaStateMachineInstance
    {
        public Guid CorrelationId { get; set; }
        public string CurrentState { get; set; }

        // Order Details
        public Guid OrderId { get; set; }
        public string CustomerId { get; set; }
        public string ProductId { get; set; }
        public int Quantity { get; set; }
        public decimal Amount { get; set; }

        // Saga Status Flags
        public bool InventoryReserved { get; set; }
        public bool PaymentProcessed { get; set; }
        public bool ShippingScheduled { get; set; }

        // Timestamps
        public DateTime CreatedAt { get; set; }
        public DateTime? CompletedAt { get; set; }
        public DateTime? FailedAt { get; set; }

        // Failure Details
        public string FailureReason { get; set; }
    }
}
```

### Step 4: Create Sequential Saga State Machine

**File: Orchestrator/SequentialOrderSagaStateMachine.cs**

```csharp
namespace OrderSaga.Orchestrator
{
    using MassTransit;
    using SharedMessages.Events;

    public class SequentialOrderSagaStateMachine : MassTransitStateMachine<OrderSagaData>
    {
        public State WaitingForInventory { get; private set; }
        public State WaitingForPayment { get; private set; }
        public State WaitingForShipping { get; private set; }
        public State OrderCompleted { get; private set; }

        public Event<OrderCreated> OrderCreatedEvent { get; private set; }
        public Event<InventoryReserved> InventoryReservedEvent { get; private set; }
        public Event<PaymentProcessed> PaymentProcessedEvent { get; private set; }
        public Event<ShippingScheduled> ShippingScheduledEvent { get; private set; }

        public SequentialOrderSagaStateMachine()
        {
            InstanceState(x => x.CurrentState);

            // Initial: Order Created → Send ReserveInventory Command
            Initially(
                When(OrderCreatedEvent)
                    .Then(context =>
                    {
                        context.Saga.OrderId = context.Message.OrderId;
                        context.Saga.CustomerId = context.Message.CustomerId;
                        context.Saga.CreatedAt = DateTime.UtcNow;
                    })
                    // Send command and wait for response
                    .Send(new Uri("queue:reserve-inventory"), context =>
                        (ReserveInventoryCommand)new
                        {
                            OrderId = context.Message.OrderId,
                            ProductId = context.Message.ProductId,
                            Quantity = context.Message.Quantity
                        })
                    .TransitionTo(WaitingForInventory)
            );

            // STEP 1: Inventory Reserved → Send Payment Command
            During(WaitingForInventory,
                When(InventoryReservedEvent)
                    .Then(context => context.Saga.InventoryReserved = true)
                    // Only after inventory is reserved, process payment
                    .Send(new Uri("queue:process-payment"), context =>
                        (ProcessPaymentCommand)new
                        {
                            OrderId = context.Message.OrderId,
                            CustomerId = context.Saga.CustomerId,
                            Amount = context.Saga.Amount
                        })
                    .TransitionTo(WaitingForPayment)
            );

            // STEP 2: Payment Processed → Send Shipping Command
            During(WaitingForPayment,
                When(PaymentProcessedEvent)
                    .Then(context => context.Saga.PaymentProcessed = true)
                    // Only after payment succeeds, schedule shipping
                    .Send(new Uri("queue:schedule-shipping"), context =>
                        (ScheduleShippingCommand)new
                        {
                            OrderId = context.Message.OrderId,
                            CustomerId = context.Saga.CustomerId
                        })
                    .TransitionTo(WaitingForShipping)
            );

            // STEP 3: Shipping Scheduled → Complete
            During(WaitingForShipping,
                When(ShippingScheduledEvent)
                    .Then(context =>
                    {
                        context.Saga.ShippingScheduled = true;
                        context.Saga.CompletedAt = DateTime.UtcNow;
                    })
                    .TransitionTo(OrderCompleted)
            );

            During(OrderCompleted);
        }
    }
}
```

### Step 5: Create Parallel Saga State Machine

**File: Orchestrator/ParallelOrderSagaStateMachine.cs**

```csharp
namespace OrderSaga.Orchestrator
{
    using MassTransit;
    using SharedMessages.Events;

    public class ParallelOrderSagaStateMachine : MassTransitStateMachine<OrderSagaData>
    {
        public State WaitingForAllServicesCompletion { get; private set; }
        public State OrderCompleted { get; private set; }

        public Event<OrderCreated> OrderCreatedEvent { get; private set; }
        public Event<InventoryReserved> InventoryReservedEvent { get; private set; }
        public Event<PaymentProcessed> PaymentProcessedEvent { get; private set; }
        public Event<ShippingScheduled> ShippingScheduledEvent { get; private set; }

        public ParallelOrderSagaStateMachine()
        {
            InstanceState(x => x.CurrentState);

            // Initial: Order Created → Send ALL commands in PARALLEL
            Initially(
                When(OrderCreatedEvent)
                    .Then(context =>
                    {
                        context.Saga.OrderId = context.Message.OrderId;
                        context.Saga.CustomerId = context.Message.CustomerId;
                        context.Saga.ProductId = context.Message.ProductId;
                        context.Saga.Quantity = context.Message.Quantity;
                        context.Saga.Amount = context.Message.Amount;
                        context.Saga.CreatedAt = DateTime.UtcNow;
                    })
                    // Send ALL commands in parallel - NOT sequential!
                    .Send(new Uri("queue:reserve-inventory"), context =>
                        (ReserveInventoryCommand)new
                        {
                            OrderId = context.Message.OrderId,
                            ProductId = context.Message.ProductId,
                            Quantity = context.Message.Quantity
                        })
                    .Send(new Uri("queue:process-payment"), context =>
                        (ProcessPaymentCommand)new
                        {
                            OrderId = context.Message.OrderId,
                            CustomerId = context.Message.CustomerId,
                            Amount = context.Message.Amount
                        })
                    .Send(new Uri("queue:schedule-shipping"), context =>
                        (ScheduleShippingCommand)new
                        {
                            OrderId = context.Message.OrderId,
                            CustomerId = context.Message.CustomerId
                        })
                    .TransitionTo(WaitingForAllServicesCompletion)
            );

            // Handle all responses coming in (could arrive in ANY order)
            During(WaitingForAllServicesCompletion,
                When(InventoryReservedEvent)
                    .Then(context => context.Saga.InventoryReserved = true)
                    .If(context => AllStepsComplete(context.Saga),
                        x => x.TransitionTo(OrderCompleted)
                    ),

                When(PaymentProcessedEvent)
                    .Then(context => context.Saga.PaymentProcessed = true)
                    .If(context => AllStepsComplete(context.Saga),
                        x => x.TransitionTo(OrderCompleted)
                    ),

                When(ShippingScheduledEvent)
                    .Then(context => context.Saga.ShippingScheduled = true)
                    .If(context => AllStepsComplete(context.Saga),
                        x => x.TransitionTo(OrderCompleted)
                    )
            );

            During(OrderCompleted);
        }

        private static bool AllStepsComplete(OrderSagaData saga)
        {
            return saga.InventoryReserved && saga.PaymentProcessed && saga.ShippingScheduled;
        }
    }
}
```

### Step 6: Create Hybrid Saga State Machine

**File: Orchestrator/HybridOrderSagaStateMachine.cs**

```csharp
namespace OrderSaga.Orchestrator
{
    using MassTransit;
    using SharedMessages.Events;

    public class HybridOrderSagaStateMachine : MassTransitStateMachine<OrderSagaData>
    {
        public State WaitingForInventoryAndPayment { get; private set; }
        public State WaitingForShipping { get; private set; }
        public State OrderCompleted { get; private set; }

        public Event<OrderCreated> OrderCreatedEvent { get; private set; }
        public Event<InventoryReserved> InventoryReservedEvent { get; private set; }
        public Event<PaymentProcessed> PaymentProcessedEvent { get; private set; }
        public Event<ShippingScheduled> ShippingScheduledEvent { get; private set; }

        public HybridOrderSagaStateMachine()
        {
            InstanceState(x => x.CurrentState);

            // PHASE 1: Send Inventory + Payment in PARALLEL
            Initially(
                When(OrderCreatedEvent)
                    .Then(context =>
                    {
                        context.Saga.OrderId = context.Message.OrderId;
                        context.Saga.CustomerId = context.Message.CustomerId;
                        context.Saga.ProductId = context.Message.ProductId;
                        context.Saga.Quantity = context.Message.Quantity;
                        context.Saga.Amount = context.Message.Amount;
                        context.Saga.CreatedAt = DateTime.UtcNow;
                    })
                    // Send both in parallel
                    .Send(new Uri("queue:reserve-inventory"), context =>
                        (ReserveInventoryCommand)new
                        {
                            OrderId = context.Message.OrderId,
                            ProductId = context.Message.ProductId,
                            Quantity = context.Message.Quantity
                        })
                    .Send(new Uri("queue:process-payment"), context =>
                        (ProcessPaymentCommand)new
                        {
                            OrderId = context.Message.OrderId,
                            CustomerId = context.Message.CustomerId,
                            Amount = context.Message.Amount
                        })
                    .TransitionTo(WaitingForInventoryAndPayment)
            );

            // PHASE 2: Wait for BOTH Inventory and Payment
            During(WaitingForInventoryAndPayment,
                When(InventoryReservedEvent)
                    .Then(context => context.Saga.InventoryReserved = true)
                    // Only transition if BOTH are done
                    .If(context => context.Saga.InventoryReserved && context.Saga.PaymentProcessed,
                        x => x.Send(new Uri("queue:schedule-shipping"), context =>
                            (ScheduleShippingCommand)new
                            {
                                OrderId = context.Saga.OrderId,
                                CustomerId = context.Saga.CustomerId
                            })
                            .TransitionTo(WaitingForShipping)
                    ),

                When(PaymentProcessedEvent)
                    .Then(context => context.Saga.PaymentProcessed = true)
                    // Only transition if BOTH are done
                    .If(context => context.Saga.InventoryReserved && context.Saga.PaymentProcessed,
                        x => x.Send(new Uri("queue:schedule-shipping"), context =>
                            (ScheduleShippingCommand)new
                            {
                                OrderId = context.Saga.OrderId,
                                CustomerId = context.Saga.CustomerId
                            })
                            .TransitionTo(WaitingForShipping)
                    )
            );

            // PHASE 3: Shipping (Sequential - happens after both Phase 2 steps complete)
            During(WaitingForShipping,
                When(ShippingScheduledEvent)
                    .Then(context =>
                    {
                        context.Saga.ShippingScheduled = true;
                        context.Saga.CompletedAt = DateTime.UtcNow;
                    })
                    .TransitionTo(OrderCompleted)
            );

            During(OrderCompleted);
        }
    }
}
```

### Step 7: Setup Saga Persistence

**File: Orchestrator/OrderSagaDbContext.cs**

```csharp
namespace OrderSaga.Orchestrator
{
    using MassTransit;
    using Microsoft.EntityFrameworkCore;

    public class OrderSagaDbContext : DbContext
    {
        public DbSet<OrderSagaData> OrderSagas { get; set; }

        public OrderSagaDbContext(DbContextOptions<OrderSagaDbContext> options)
            : base(options)
        {
        }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);

            var map = modelBuilder.Entity<OrderSagaData>();
            
            map.HasKey(p => p.CorrelationId);
            map.Property(p => p.CurrentState).HasMaxLength(64);
            map.Property(p => p.CustomerId).HasMaxLength(128);
            map.Property(p => p.ProductId).HasMaxLength(128);
            map.Property(p => p.FailureReason).HasMaxLength(1024);

            map.HasIndex(p => p.CreatedAt);
            map.HasIndex(p => p.CurrentState);
        }
    }

    public class OrderSagaDefinition : SagaDefinition<OrderSagaData>
    {
        protected override void ConfigureSaga(IReceiveEndpointConfigurator endpointConfigurator,
            ISagaConfigurator<OrderSagaData> sagaConfigurator)
        {
            endpointConfigurator.UseMessageRetry(r => r.Intervals(100, 200, 500, 1000));
        }
    }
}
```

### Step 8: Setup Orchestrator Service

**File: Orchestrator/Program.cs**

```csharp
using MassTransit;
using Microsoft.EntityFrameworkCore;
using OrderSaga.Orchestrator;

var builder = WebApplication.CreateBuilder(args);

// Add DbContext
builder.Services.AddDbContext<OrderSagaDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"))
);

// Add MassTransit with Azure Service Bus
builder.Services.AddMassTransit(x =>
{
    // Add Saga (choose one based on your need)
    x.AddSagaStateMachine<SequentialOrderSagaStateMachine, OrderSagaData>()
        .EntityFrameworkRepository(r =>
        {
            r.ConcurrencyToken = p => p.RowVersion;
            r.ExistingDbContext<OrderSagaDbContext>();
        });

    // Configure Transport: Azure Service Bus
    x.UsingAzureServiceBus((context, cfg) =>
    {
        var connectionString = builder.Configuration.GetConnectionString("AzureServiceBus");
        cfg.Host(connectionString);

        // Configure Saga Endpoint
        cfg.ReceiveEndpoint("order-saga", e =>
        {
            e.ConfigureSaga<OrderSagaData>(context);
        });

        // Configure other endpoints
        cfg.ReceiveEndpoint("order-service", e =>
        {
            e.PrefetchCount = 20;
        });
    });
});

builder.Services.AddControllers();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Auto-create database
using (var scope = app.Services.CreateScope())
{
    var dbContext = scope.ServiceProvider.GetRequiredService<OrderSagaDbContext>();
    dbContext.Database.Migrate();
}

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

### Step 9: Inventory Service Consumer

**File: InventoryService/Consumers/ReserveInventoryConsumer.cs**

```csharp
namespace InventoryService.Consumers
{
    using MassTransit;
    using SharedMessages.Events;
    using Microsoft.Extensions.Logging;

    public class ReserveInventoryConsumer : IConsumer<ReserveInventoryCommand>
    {
        private readonly ILogger<ReserveInventoryConsumer> _logger;
        private readonly IInventoryRepository _inventoryRepository;
        private readonly IPublishEndpoint _publishEndpoint;

        public ReserveInventoryConsumer(
            ILogger<ReserveInventoryConsumer> logger,
            IInventoryRepository inventoryRepository,
            IPublishEndpoint publishEndpoint)
        {
            _logger = logger;
            _inventoryRepository = inventoryRepository;
            _publishEndpoint = publishEndpoint;
        }

        public async Task Consume(ConsumeContext<ReserveInventoryCommand> context)
        {
            _logger.LogInformation($"Reserving inventory for Order {context.Message.OrderId}");

            try
            {
                var reserved = await _inventoryRepository.ReserveAsync(
                    context.Message.ProductId,
                    context.Message.Quantity
                );

                if (reserved)
                {
                    _logger.LogInformation($"Inventory reserved successfully for Order {context.Message.OrderId}");
                    
                    await _publishEndpoint.Publish<InventoryReserved>(new
                    {
                        OrderId = context.Message.OrderId,
                        ProductId = context.Message.ProductId,
                        Quantity = context.Message.Quantity
                    });
                }
                else
                {
                    _logger.LogError($"Failed to reserve inventory for Order {context.Message.OrderId}");
                    
                    await _publishEndpoint.Publish<InventoryReservationFailed>(new
                    {
                        OrderId = context.Message.OrderId,
                        Reason = "Insufficient inventory available"
                    });
                }
            }
            catch (Exception ex)
            {
                _logger.LogError($"Error reserving inventory: {ex.Message}");
                
                await _publishEndpoint.Publish<InventoryReservationFailed>(new
                {
                    OrderId = context.Message.OrderId,
                    Reason = ex.Message
                });
            }
        }
    }
}
```

### Step 10: Order Service Entry Point

**File: OrderService/Controllers/OrdersController.cs**

```csharp
namespace OrderService.Controllers
{
    using MassTransit;
    using Microsoft.AspNetCore.Mvc;
    using SharedMessages.Events;
    using System;
    using System.Threading.Tasks;

    [ApiController]
    [Route("api/[controller]")]
    public class OrdersController : ControllerBase
    {
        private readonly IPublishEndpoint _publishEndpoint;
        private readonly ILogger<OrdersController> _logger;

        public OrdersController(
            IPublishEndpoint publishEndpoint,
            ILogger<OrdersController> logger)
        {
            _publishEndpoint = publishEndpoint;
            _logger = logger;
        }

        [HttpPost("place-order")]
        public async Task<IActionResult> PlaceOrder([FromBody] PlaceOrderRequest request)
        {
            if (request == null)
                return BadRequest("Order request is required");

            var orderId = Guid.NewGuid();

            _logger.LogInformation($"Publishing OrderCreated event for Order {orderId}");

            // Publish event to trigger saga
            await _publishEndpoint.Publish<OrderCreated>(new
            {
                OrderId = orderId,
                CustomerId = request.CustomerId,
                ProductId = request.ProductId,
                Quantity = request.Quantity,
                Amount = request.Amount
            });

            return Accepted(new { OrderId = orderId, Message = "Order placed successfully" });
        }
    }

    public class PlaceOrderRequest
    {
        public string CustomerId { get; set; }
        public string ProductId { get; set; }
        public int Quantity { get; set; }
        public decimal Amount { get; set; }
    }
}
```

### Configuration: appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=OrderSagaDb;Trusted_Connection=true;",
    "AzureServiceBus": "Endpoint=sb://yournamespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=your-key-here"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "MassTransit": "Debug"
    }
  }
}
```

---

## Key Takeaways

1. **Order Service does NOT directly communicate** with other services
2. **Communication is asynchronous event-driven** via Azure Service Bus
3. **Saga Orchestrator controls the flow** - Sequential, Parallel, or Hybrid
4. **Separate orchestrator deployment** provides scalability and separation of concerns
5. **Choose execution model based on business logic dependencies**

---

## Best Practices

✅ Use **Orchestration** for complex workflows with dependencies  
✅ Use **Choreography** for simple, independent operations  
✅ Make all transactions **idempotent** (safe to retry)  
✅ Log all saga steps for debugging  
✅ Implement **timeouts and retries**  
✅ Use persistent storage for saga state  
✅ Implement **dead-letter queues** for failed messages  
✅ Monitor saga execution with observability tools  

---

## Advantages of Event-Driven Saga Architecture

✅ **Event-Driven:** Services communicate asynchronously via Azure Service Bus  
✅ **Loosely Coupled:** Services don't depend on each other directly  
✅ **Scalable:** Each service scales independently  
✅ **Resilient:** MassTransit provides automatic retries, dead-letter queues  
✅ **Observable:** Full audit trail of all saga steps  
✅ **Testable:** Easy to test individual consumers and saga logic  
✅ **Maintainable:** Separation of concerns (saga orchestrator is separate)  

---

## Summary Table

| Question | Answer |
|----------|--------|
| **SAGA Abbreviation** | Sequence Across Geographically-distributed Applications |
| **Libraries** | MassTransit (recommended), NServiceBus, Brighter, Azure Durable Functions, Rebus, Akka.NET |
| **Orchestrator Deployment** | YES - typically a separate microservice for scalability, testability, and separation of concerns |
| **Communication Model** | Asynchronous event-driven (NOT direct service calls) |
| **Execution Patterns** | Sequential (dependent steps), Parallel (independent steps), Hybrid (mixed approach) |
| **Performance** | Parallel > Hybrid > Sequential in terms of speed (depends on use case) |

---

**Last Updated:** November 30, 2025
