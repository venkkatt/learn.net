# Complete Guide: Async/Await & Task Parallel Library (TPL) in .NET

---

## Table of Contents
1. [Async/Await Fundamentals](#asyncawait-fundamentals)
2. [How Async/Await Works](#how-asyncawait-works)
3. [Task Parallel Library (TPL)](#task-parallel-library-tpl)
4. [Async vs TPL: When to Use Each](#async-vs-tpl-when-to-use-each)
5. [Error Handling](#error-handling)
6. [Real-World Examples](#real-world-examples)
7. [Common Mistakes to Avoid](#common-mistakes-to-avoid)
8. [Interview Questions](#interview-questions)

---

## Async/Await Fundamentals

### What is Async/Await?

**Async/await** is a language feature that makes asynchronous, non-blocking code look synchronous and easy to read.

**Core Principle:** When a thread encounters an `await` on an I/O operation, it **doesn't block**. Instead, it:
1. Registers a callback with the operation
2. **Returns immediately to the thread pool**
3. The operation completes outside your thread pool
4. When done, any free thread resumes execution

### Restaurant Kitchen Analogy

**Synchronous (Bad):**
- Chef takes order → boils water (5 min standing idle) → cooks pasta (8 min idle) → serves
- Only helps next customer after completely done
- 10 customers = 130+ minutes

**Asynchronous (Smart):**
- Chef takes order → puts water on → helps Customer 2
- When water boils → back to pasta → helps Customer 3
- When pasta cooks → serves Customer 1
- 10 customers = ~30 minutes with 1 chef

**Key Insight:** The chef (thread) isn't idle while water boils (I/O). The stove (DB/Network) does the waiting.

---

## How Async/Await Works

### The State Machine

When you write an `async` method, the C# compiler **transforms it into a state machine** at compile time. This allows the method to be paused and resumed.

**Your Code:**
```csharp
public async Task<string> GetOrderDetailsAsync(int orderId)
{
    Console.WriteLine("Starting fetch");           // Section 1
    
    var order = await FetchOrderFromDbAsync(orderId);  // AWAIT POINT 1
    Console.WriteLine($"Got order: {order}");     // Section 2
    
    var customer = await FetchCustomerAsync(order.CustomerId);  // AWAIT POINT 2
    Console.WriteLine($"Got customer: {customer}");   // Section 3
    
    return $"{order.Name} - {customer.Name}";     // Section 4
}
```

**Compiler Transforms Into (Conceptually):**
```csharp
public class GetOrderDetailsAsync_StateMachine
{
    private int _state = 0;
    private TaskAwaiter _awaiter;
    private Order _order;
    private Customer _customer;
    
    public void MoveNext()
    {
        switch (_state)
        {
            case 0:  // Before first await
                Console.WriteLine("Starting fetch");
                _state = 1;
                _awaiter = FetchOrderFromDbAsync(orderId).GetAwaiter();
                if (!_awaiter.IsCompleted)
                {
                    _awaiter.OnCompleted(MoveNext);  // Resume when done
                    return;
                }
                goto case 1;
                
            case 1:  // After first await
                _order = _awaiter.GetResult();
                Console.WriteLine($"Got order: {_order}");
                _state = 2;
                _awaiter = FetchCustomerAsync(_order.CustomerId).GetAwaiter();
                if (!_awaiter.IsCompleted)
                {
                    _awaiter.OnCompleted(MoveNext);
                    return;
                }
                goto case 2;
                
            case 2:  // After second await
                _customer = _awaiter.GetResult();
                Console.WriteLine($"Got customer: {_customer}");
                _state = 3;
                goto case 3;
                
            case 3:  // Final section
                return $"{_order.Name} - {_customer.Name}";
        }
    }
}
```

Each `await` is a **bookmark**. The state machine resumes from bookmarks when operations complete.

### Thread Execution Flow

```
Timeline:
0ms:     Thread1: Runs "Starting fetch" → Hits first await
0ms:     Thread1: Registers callback with DB operation
0ms:     Thread1: Returns to thread pool (FREED)

[Meanwhile, database processes query on DB server - NOT using Thread1]

1000ms:  DB returns result
1000ms:  Any free thread (could be Thread1, could be Thread5) grabs work
1000ms:  Runs "Got order: ..." → Hits second await
1000ms:  Registers callback with customer fetch
1000ms:  Thread returns to pool (FREED)

[Meanwhile, customer fetch happens on another server]

1500ms:  Customer fetch returns
1500ms:  Any free thread resumes
1500ms:  Runs "Got customer: ..." → Runs final section
1500ms:  Method complete, returns result

Total method execution: 1.5 seconds of wall time
Total thread busy time: ~5 milliseconds (just setup/cleanup)
```

### The Critical Truth About I/O Operations

**"The actual I/O work happens OUTSIDE your thread pool"**

- **Database query** → Runs on database server (your thread is freed)
- **HTTP call** → Runs on remote server (your thread is freed)
- **File read** → OS kernel handles it (your thread is freed)
- **Email send** → Mail server processes it (your thread is freed)

Your thread pool threads are only busy for:
1. Starting the operation (microseconds)
2. Registering the callback (microseconds)
3. Resuming after completion (milliseconds)

### Order Guarantee: Always Sequential

```csharp
await Step1Async();  // Always 1st
await Step2Async();  // Always 2nd
await Step3Async();  // Always 3rd
```

**Execution order is 100% guaranteed.** The state machine ensures this. No parallelism unless you explicitly use `Task.WhenAll()`.

### Synchronization Context: Where Execution Resumes

**In UI apps (WPF/WinForms):**
```csharp
public async void Button_Click()
{
    label.Text = "Loading...";  // UI thread
    
    var data = await FetchAsync();  // Thread freed
    
    label.Text = data;  // Magically back on UI thread!
}
```
The sync context **remembers** you were on the UI thread and brings you back.

**In ASP.NET Core:**
```csharp
public async Task<ActionResult> GetData()
{
    var data = await FetchAsync();
    return Ok(data);  // Might run on different thread pool thread
}
```
No UI thread context, so any free thread continues.

---

## Task Parallel Library (TPL)

### What is TPL?

**TPL** is a set of APIs for **parallel** and **task-based** programming. It intelligently manages thread pool threads to execute multiple tasks simultaneously across CPU cores.

### How TPL Works with Threads

```
TPL Thread Pool (e.g., 50 threads available)

Your Code:
Task.Run(Calc1())  →  Thread4 grabs task
Task.Run(Calc2())  →  Thread5 grabs task
Task.Run(Calc3())  →  Thread6 grabs task

CPU Execution (4-core CPU):
Core1: Calc1()
Core2: Calc2()
Core3: Calc3()
Core4: (other tasks or system work)

When done:
Thread4, Thread5, Thread6 return to pool for reuse
```

### TPL Components

1. **Thread Pool**: Repository of reusable threads (100s of them)
2. **Task**: Unit of work to execute
3. **Task Scheduler**: Decides which thread executes which task
4. **Work-Stealing Queues**: Each thread has local queue; busy threads "steal" work from idle ones for load balancing

### Key TPL APIs

#### `Task.Run()` - Fire-and-Forget CPU Work
```csharp
Task.Run(() => CPUIntensiveCalculation());
```
- Grabs a thread pool thread
- Executes the work
- Returns thread when done

#### `Parallel.For()` - Data Parallelism
```csharp
Parallel.For(0, 1_000_000, i => 
{
    ProcessItem(i);
});
```
- Splits range into chunks (one per CPU core)
- Each chunk runs on different thread in parallel
- Waits for all to complete
- Example: 8 cores → 8 chunks processed simultaneously

#### `Parallel.ForEach()` - Data Parallelism Over Collections
```csharp
var items = GetMillionItems();
Parallel.ForEach(items, item => Process(item));
```

#### `Task.WhenAll()` - Wait for Multiple Tasks
```csharp
var t1 = Task.Run(() => Calc1());
var t2 = Task.Run(() => Calc2());
var t3 = Task.Run(() => Calc3());

Task.WaitAll(t1, t2, t3);  // Wait for all
```

#### `Task.WhenAny()` - Wait for First Completion
```csharp
var fastest = await Task.WhenAny(task1, task2, task3);
```

### Manual Threads vs TPL

| Aspect | Manual `new Thread()` | TPL |
|--------|----------------------|-----|
| **Creating threads** | `new Thread(Work).Start()` × N | `Task.Run(Work)` × N |
| **Thread count** | You create 1 per task (wasteful) | Reuses pool threads (~50) |
| **Efficiency** | 1MB memory per thread | Minimal overhead |
| **Load balancing** | Manual (error-prone) | Automatic work-stealing |
| **Scheduling** | None (OS decides) | Smart TPL scheduler |

**Example - The Cost:**
```csharp
// ❌ Manual: Creates 10,000 threads = ~10GB memory = crashes
for (int i = 0; i < 10_000; i++)
    new Thread(() => Calc(i)).Start();

// ✅ TPL: Uses ~50-100 threads efficiently
Parallel.For(0, 10_000, i => Calc(i));
```

### Thread Lifecycle with TPL

```
Application Start:
Thread Pool initialized with ~50 threads (CPU-dependent)

During Execution:
Task1 arrives → Thread4 grabs it → Executes Calc1()
Task2 arrives → Thread5 grabs it → Executes Calc2()
Task1 done → Thread4 returns to pool
Task3 arrives → Thread4 (available) grabs Task3

If all threads busy:
New task waits in queue → Thread picks it when free

Application End:
Threads don't die, they persist for future work
```

---

## Async vs TPL: When to Use Each

### The Decision Matrix

| Scenario | Tool | Why |
|----------|------|-----|
| **Database calls** | Async/Await | Thread freed during network wait |
| **HTTP requests** | Async/Await | Network/server does waiting |
| **File I/O** | Async/Await | OS kernel handles I/O |
| **Mathematical calculations on 1M items** | TPL | Parallel across CPU cores |
| **Image processing** | TPL | Pixel ops parallelize well |
| **Sorting large dataset** | TPL | PLINQ (Parallel LINQ) |
| **Web API endpoint** | Async/Await | Scale to 10,000s concurrent users |
| **Desktop app responsiveness** | Async/Await | Frees UI thread |

### Side-by-Side Comparison

**I/O Bound: Async/Await**
```csharp
public async Task<User> GetUserAsync(int id)
{
    // Thread freed during network wait
    var data = await _db.Users.FindAsync(id);
    return data;
}
```

**CPU Bound: TPL**
```csharp
public int ProcessMillionItems()
{
    // Parallel across CPU cores
    return Parallel.For(0, 1_000_000, 
        (i, _) => SomeCalculation(i)
    ).Count();
}
```

**Hybrid: Both**
```csharp
public async Task<ProcessResult> ProcessOrderAsync(int orderId)
{
    // Async I/O
    var order = await _db.Orders.FindAsync(orderId);
    
    // TPL CPU parallelism
    var processed = await Task.Run(() => 
        Parallel.For(0, order.Items.Count, i => 
            order.Items[i].Price = ExpensiveCalc(order.Items[i])
        )
    );
    
    // Async I/O
    await _email.SendAsync(order.CustomerEmail);
    return new ProcessResult { Success = true };
}
```

---

## Error Handling

### Method 1: Try-Catch with Await (Recommended - 95% of cases)

```csharp
public async Task ProcessOrderAsync(int orderId)
{
    try
    {
        var order = await FetchOrderAsync(orderId);
        var customer = await FetchCustomerAsync(order.CustomerId);
        await SendEmailAsync(customer.Email);
        
        Console.WriteLine("Success!");
    }
    catch (HttpRequestException ex) when (ex.StatusCode == 404)
    {
        Console.WriteLine("Order not found");
    }
    catch (DbException ex)
    {
        Console.WriteLine($"Database error: {ex.Message}");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Unexpected error: {ex.Message}");
        _logger.LogError(ex, "Order processing failed for {OrderId}", orderId);
    }
}
```

**How it works:**
- Exception from **any awaited task** is rethrown
- Caught by `catch` block just like synchronous code
- Multiple `catch` blocks work normally with type-specific handling

### Method 2: Task.WhenAll with Multiple Exceptions

```csharp
public async Task<(Order, Customer)> FetchOrderAndCustomerAsync(int orderId)
{
    try
    {
        var orderTask = FetchOrderAsync(orderId);
        var customerTask = FetchCustomerAsync(123);
        
        await Task.WhenAll(orderTask, customerTask);
        
        return (orderTask.Result, customerTask.Result);
    }
    catch (AggregateException ex)
    {
        // Multiple tasks might have failed
        foreach (var innerEx in ex.InnerExceptions)
        {
            Console.WriteLine($"Task failed: {innerEx.Message}");
        }
        throw;
    }
}
```

**Important:** When `Task.WhenAll()` is awaited and any task fails, it throws `AggregateException` containing all failures.

### Method 3: Handle Individual Task Exceptions

```csharp
public async Task ProcessMultipleOrdersAsync(int[] orderIds)
{
    var tasks = orderIds
        .Select(async id => 
        {
            try
            {
                return await FetchOrderAsync(id);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to fetch order {Id}", id);
                return null;
            }
        })
        .ToArray();
    
    var results = await Task.WhenAll(tasks);
    var successfulOrders = results.Where(x => x != null).ToList();
}
```

### Method 4: ContinueWith for Unobserved Tasks

```csharp
public void FireAndForgetWithErrorHandling(Task task)
{
    task.ContinueWith(t =>
    {
        if (t.IsFaulted)
        {
            foreach (var ex in t.Exception!.InnerExceptions)
            {
                _logger.LogError(ex, "Unobserved task failed");
            }
        }
        else if (t.IsCompletedSuccessfully)
        {
            Console.WriteLine("Task completed successfully");
        }
    });
}
```

### Method 5: Task.Run with Error Handling

```csharp
public async Task<int> CalculateAsync(int input)
{
    try
    {
        return await Task.Run(() =>
        {
            if (input < 0) throw new ArgumentException("Negative input");
            return ExpensiveCalculation(input);
        });
    }
    catch (ArgumentException ex)
    {
        Console.WriteLine($"Validation failed: {ex.Message}");
        return -1;
    }
}
```

---

## Real-World Examples

### Example 1: Web API Endpoint

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;
    private readonly IEmailService _emailService;
    private readonly ILogger<OrdersController> _logger;
    
    [HttpGet("{id}")]
    public async Task<ActionResult<OrderDto>> GetOrderAsync(int id)
    {
        try
        {
            // Async I/O: Thread freed during DB call
            var order = await _orderService.GetOrderAsync(id);
            
            if (order == null)
                return NotFound();
            
            return Ok(order);
        }
        catch (UnauthorizedAccessException)
        {
            return Unauthorized();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to get order {OrderId}", id);
            return StatusCode(500, "Internal server error");
        }
    }
    
    [HttpPost]
    public async Task<ActionResult<OrderDto>> CreateOrderAsync(CreateOrderRequest request)
    {
        try
        {
            // Parallel process items
            var processingTasks = request.Items
                .Select(async item => 
                {
                    // CPU-bound: TPL
                    item.ProcessedPrice = await Task.Run(() => 
                        CalculateTax(item.Price, item.Category)
                    );
                    return item;
                })
                .ToArray();
            
            await Task.WhenAll(processingTasks);
            
            // Async save
            var order = await _orderService.CreateOrderAsync(request);
            
            // Fire-and-forget email (but handle errors)
            _ = _emailService.SendOrderConfirmationAsync(order.Id)
                .ContinueWith(t => 
                {
                    if (t.IsFaulted)
                        _logger.LogError(t.Exception, "Email send failed for order {OrderId}", order.Id);
                });
            
            return CreatedAtAction(nameof(GetOrderAsync), new { id = order.Id }, order);
        }
        catch (ValidationException ex)
        {
            return BadRequest(ex.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to create order");
            return StatusCode(500);
        }
    }
}
```

### Example 2: Service Layer with Complex Operations

```csharp
public class OrderService : IOrderService
{
    private readonly IOrderRepository _orderRepo;
    private readonly IInventoryService _inventory;
    private readonly IPaymentGateway _payment;
    
    public async Task<Order> ProcessCompleteOrderAsync(int orderId, PaymentInfo payment)
    {
        try
        {
            // Step 1: Fetch order (async I/O)
            var order = await _orderRepo.GetOrderAsync(orderId);
            if (order == null)
                throw new OrderNotFoundException(orderId);
            
            // Step 2: Parallel tasks - some CPU, some I/O
            var (inventoryValid, paymentValid) = await Task.WhenAll(
                Task.Run(() => ValidateInventory(order)),  // CPU
                _payment.ValidateAsync(payment)             // I/O
            );
            
            if (!inventoryValid)
                throw new InsufficientInventoryException();
            
            if (!paymentValid)
                throw new PaymentFailedException();
            
            // Step 3: Process payment (I/O)
            var paymentResult = await _payment.ProcessAsync(payment, order.Total);
            
            // Step 4: Parallel updates
            var updateTasks = new List<Task>
            {
                _inventory.DeductItemsAsync(order.Items),
                _orderRepo.UpdateOrderStatusAsync(orderId, OrderStatus.Processing),
                RecordPaymentAsync(paymentResult)
            };
            
            await Task.WhenAll(updateTasks);
            
            // Step 5: Fetch updated order
            var completedOrder = await _orderRepo.GetOrderAsync(orderId);
            
            return completedOrder;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to process order {OrderId}", orderId);
            throw;
        }
    }
    
    private bool ValidateInventory(Order order)
    {
        // CPU-bound validation
        return order.Items.All(i => i.Quantity > 0);
    }
    
    private async Task RecordPaymentAsync(PaymentResult result)
    {
        // Async operation
        await Task.Delay(100);  // Simulate recording
    }
}
```

### Example 3: Background Job with TPL

```csharp
public class DataProcessingJob : IHostedService
{
    private readonly IDataRepository _repo;
    private CancellationTokenSource _cts;
    
    public async Task StartAsync(CancellationToken cancellationToken)
    {
        _cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
        _ = ProcessDataContinuouslyAsync(_cts.Token);
        await Task.CompletedTask;
    }
    
    private async Task ProcessDataContinuouslyAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try
            {
                // Fetch batch of data (async I/O)
                var batch = await _repo.GetUnprocessedBatchAsync(1000);
                
                if (batch.Count == 0)
                {
                    await Task.Delay(5000, ct);
                    continue;
                }
                
                // Process in parallel (CPU-bound)
                await Task.Run(() =>
                {
                    Parallel.ForEach(batch, item =>
                    {
                        item.ProcessedValue = ComplexCalculation(item.RawValue);
                        item.Status = ProcessStatus.Completed;
                    });
                }, ct);
                
                // Save results (async I/O)
                await _repo.UpdateBatchAsync(batch);
                
                _logger.LogInformation("Processed {Count} items", batch.Count);
            }
            catch (OperationCanceledException)
            {
                break;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in data processing job");
                await Task.Delay(10000, ct);  // Wait before retry
            }
        }
    }
    
    public async Task StopAsync(CancellationToken cancellationToken)
    {
        _cts?.Cancel();
        await Task.CompletedTask;
    }
    
    private int ComplexCalculation(int value)
    {
        // CPU-intensive work
        int result = 0;
        for (int i = 0; i < 1_000_000; i++)
            result += value * i;
        return result;
    }
}
```

---

## Common Mistakes to Avoid

### ❌ Mistake 1: Using .Result (Deadlock)

```csharp
// WRONG - Deadlock!
public ActionResult GetOrder(int id)
{
    var order = GetOrderAsync(id).Result;  // Blocks calling thread
    return Ok(order);
}

// RIGHT
public async Task<ActionResult> GetOrder(int id)
{
    var order = await GetOrderAsync(id);
    return Ok(order);
}
```

**Why:** `.Result` blocks the current thread waiting for the task. In UI/ASP.NET request contexts, if the task's continuation needs that context, deadlock occurs.

### ❌ Mistake 2: Fire-and-Forget Without Error Handling

```csharp
// WRONG - Unobserved exception will crash app (pre-.NET Core)
Task.Run(() => { throw new Exception("Boom!"); });

// RIGHT - Option 1: Await it
await Task.Run(() => { throw new Exception("Boom!"); });

// RIGHT - Option 2: Handle in ContinueWith
Task.Run(() => { throw new Exception("Boom!"); })
    .ContinueWith(t => 
    {
        if (t.IsFaulted)
            _logger.LogError(t.Exception, "Task failed");
    });
```

### ❌ Mistake 3: Wrapping I/O in Task.Run

```csharp
// WRONG - Wastes thread pool thread
public async Task<User> GetUserAsync(int id)
{
    return await Task.Run(async () => 
        await _db.Users.FindAsync(id)
    );
}

// RIGHT - I/O doesn't need Task.Run
public async Task<User> GetUserAsync(int id)
{
    return await _db.Users.FindAsync(id);
}
```

### ❌ Mistake 4: Blocking Inside Async Method

```csharp
// WRONG
public async Task ProcessAsync()
{
    Thread.Sleep(1000);  // Blocks thread for 1 second!
    await FetchAsync();
}

// RIGHT
public async Task ProcessAsync()
{
    await Task.Delay(1000);  // Frees thread, waits 1 second
    await FetchAsync();
}
```

### ❌ Mistake 5: Returning void from Async

```csharp
// WRONG - Can't await, hard to handle errors
public async void ProcessOrderAsync(int id)
{
    await FetchAsync();
}

// RIGHT - Always return Task or Task<T>
public async Task ProcessOrderAsync(int id)
{
    await FetchAsync();
}

// Exception: Event handlers are OK to return void
private async void Button_Click(object sender, EventArgs e)
{
    await ProcessOrderAsync(123);
}
```

### ❌ Mistake 6: Race Conditions with Shared State

```csharp
// WRONG - Race condition!
private List<int> results = new();
Parallel.For(0, 1000, i => results.Add(i));  // Concurrent access!

// RIGHT - Use thread-safe collection
private ConcurrentBag<int> results = new();
Parallel.For(0, 1000, i => results.Add(i));
```

### ❌ Mistake 7: Sequential When You Mean Parallel

```csharp
// WRONG - Runs sequentially, wastes time
await FetchUserAsync(id);
await FetchOrdersAsync(id);
await FetchPaymentsAsync(id);
// Total time: sum of all three

// RIGHT - Run in parallel
await Task.WhenAll(
    FetchUserAsync(id),
    FetchOrdersAsync(id),
    FetchPaymentsAsync(id)
);
// Total time: duration of longest operation
```

---

## Interview Questions

### Q1: Explain How Async/Await Works

**Answer:**
Async/await is a compiler feature that converts your async method into a state machine. Each `await` point creates a state transition. When an `await` is encountered on an incomplete task, the method:
1. Registers a callback with the task
2. Returns the control to the caller (doesn't block)
3. The thread returns to the thread pool for reuse
4. When the awaited operation completes, the state machine continues from the bookmark
5. An available thread (same or different) resumes execution from that point

This allows thousands of concurrent I/O operations to be handled with just a few threads.

### Q2: What's the Difference Between Async/Await and Task-Based Programming?

**Answer:**
- **Async/await**: Language feature providing convenient syntax for working with tasks asynchronously
- **Task-based programming**: Older pattern using callbacks (ContinueWith) and manual task management
- Async/await is just syntactic sugar on top of tasks, but dramatically improves readability
- For I/O-bound: async/await is preferred. For CPU-bound: TPL with Task.Run/Parallel is better

### Q3: What Causes Deadlocks with Async/Await?

**Answer:**
Deadlocks occur when you synchronously block on an async method using `.Result` or `.Wait()` in a single-threaded synchronization context:

```csharp
public ActionResult GetOrder(int id)
{
    var order = GetOrderAsync(id).Result;  // Deadlock!
    return Ok(order);
}
```

**Why it deadlocks:**
1. Main thread calls `.Result` and blocks
2. Main thread waits for the task to complete
3. Task's continuation tries to resume on the same context (requires the main thread)
4. Main thread is blocked, continuation can't run → Deadlock

**Solution:** Use `async all the way` - make the calling method async too.

### Q4: When Should You Use TPL vs Async/Await?

**Answer:**
- **Async/await**: I/O-bound operations (DB, HTTP, file I/O). Frees threads during waits.
- **TPL (Task.Run, Parallel.For)**: CPU-bound operations (calculations, data processing). Parallelizes across CPU cores.

**Example Decision:**
```csharp
var user = await _db.GetUserAsync(id);    // Async/await - I/O
var scores = await Task.Run(() =>         // TPL - CPU
    Parallel.For(0, 1_000_000, i => 
        CalculateScore(i)
    )
);
```

### Q5: What Does ConfigureAwait(false) Do?

**Answer:**
`ConfigureAwait(false)` prevents the continuation from capturing the current synchronization context. Instead of resuming on the original context (e.g., UI thread), it can resume on any thread pool thread.

**Use in:** Library code, to avoid forcing consumers to handle context switching

```csharp
// Library code
public async Task<Data> GetDataAsync()
{
    var response = await httpClient.GetAsync(url)
        .ConfigureAwait(false);
    return await response.Content.ReadAsAsync<Data>()
        .ConfigureAwait(false);
}
```

### Q6: What's the Difference Between Task.WhenAll and Task.WhenAny?

**Answer:**
- **Task.WhenAll**: Waits for ALL tasks to complete. If any fail, throws AggregateException.
- **Task.WhenAny**: Returns as soon as ANY task completes. Useful for racing operations.

```csharp
// Wait for all
await Task.WhenAll(t1, t2, t3);

// Race - return fastest
var fastest = await Task.WhenAny(t1, t2, t3);
```

### Q7: Is Async/Await Always Faster?

**Answer:**
No. Async/await is more scalable and responsive, not necessarily faster. It's about:
- **Responsiveness**: Application remains responsive (UI doesn't freeze)
- **Scalability**: Can handle thousands of concurrent I/O operations with few threads
- **Speed**: If an operation is fast anyway, the overhead of async might make it slightly slower

**When to use async:**
- Latency-sensitive applications
- High concurrency scenarios
- I/O-bound operations

**When async doesn't help:**
- Local, fast CPU operations
- Where responsiveness doesn't matter

### Q8: What's an AggregateException?

**Answer:**
`AggregateException` is thrown by `Task.WhenAll` when multiple tasks fail. It contains all exceptions from failed tasks in its `InnerExceptions` collection.

```csharp
try
{
    await Task.WhenAll(task1, task2, task3);
}
catch (AggregateException ex)
{
    foreach (var innerEx in ex.InnerExceptions)
    {
        Console.WriteLine($"Task failed: {innerEx.Message}");
    }
}
```

### Q9: Should You Always Return Task from Async Methods?

**Answer:**
Yes, except for event handlers. Returning `void` makes it impossible to await the method or catch exceptions properly.

```csharp
// ❌ WRONG (except event handlers)
public async void ProcessAsync() { }

// ✅ RIGHT
public async Task ProcessAsync() { }

// ✅ RIGHT
public async Task<int> CalculateAsync() { }

// Exception: Event handlers
private async void Button_Click(object sender, EventArgs e) { }
```

### Q10: How Does the Thread Pool Manage Thread Creation?

**Answer:**
The thread pool maintains a set of worker threads (typically 100s) and manages their reuse:

1. **On startup**: Creates initial threads based on CPU core count
2. **When work arrives**: Available thread picks it up; otherwise queues it
3. **When thread busy**: Scheduler might create new thread (if load is high)
4. **When thread done**: Returns to pool for reuse (doesn't destroy)
5. **Auto-scaling**: Adds threads gradually as demand increases, removes them as demand decreases

This avoids the 1MB per thread overhead of manual thread creation and ensures efficient resource usage.

---

## Key Takeaways

1. **Async/await** = I/O magic (frees threads during waits)
2. **TPL** = CPU parallelism (splits work across cores)
3. **Order preserved** but threads not blocked
4. **I/O work happens outside** your thread pool
5. **Always await**, never use `.Result`
6. **Handle errors** with try-catch or ContinueWith
7. **Sequential by default**, parallel with `Task.WhenAll`
8. **Thread pool reuses threads** for efficiency
9. **Responsive apps** come from async/await
10. **Scalable apps** come from proper threading models

---

## Resources for Further Learning

- Microsoft Docs: Async/Await
- Microsoft Docs: Task Parallel Library
- "Concurrency in C#" by Stephen Cleary
- "Threading in C#" by Albahari & Albahari
