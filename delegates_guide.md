# Complete Guide to Delegates in .NET

## Table of Contents
1. [Introduction](#introduction)
2. [Core Concepts](#core-concepts)
3. [Types of Delegates](#types-of-delegates)
4. [Usage Patterns](#usage-patterns)
5. [Best Practices](#best-practices)
6. [Real-World Examples](#real-world-examples)

## Introduction

A **delegate** is a type-safe reference type that holds a reference to methods with a specific signature. Delegates in .NET are similar to function pointers in C/C++, but they are fully object-oriented and type-safe. They allow you to:

- Treat methods as values
- Pass methods as parameters to other methods
- Store methods in collections
- Dynamically invoke methods at runtime
- Implement callback mechanisms and event handling

### Why Use Delegates?

- **Type Safety**: Ensures method signatures match at compile time
- **Flexibility**: Enable dynamic method invocation and behavioral composition
- **Event Handling**: Foundation of the event system in .NET
- **Code Reusability**: Reduce code duplication through parameterized behavior
- **Loose Coupling**: Allow components to communicate without direct references

## Core Concepts

### Declaring a Delegate

A delegate type is declared using the `delegate` keyword, specifying the method signature it can reference:

```csharp
// Syntax
[modifier] delegate [return_type] [delegate_name]([parameters]);

// Example: Delegate that takes no parameters and returns void
public delegate void NotificationHandler(string message);

// Example: Delegate that takes parameters and returns a value
public delegate int MathOperation(int a, int b);

// Example: Generic delegate
public delegate T GenericDelegate<T>(T input);
```

### Instantiating a Delegate

Once a delegate type is declared, you can create instances by pointing to compatible methods:

```csharp
public delegate void Notify(string msg);

static void Main()
{
    // Method that matches the delegate signature
    Notify del1 = DisplayMessage;  // Method group conversion
    Notify del2 = new Notify(DisplayMessage);  // Explicit instantiation
    Notify del3 = (msg) => Console.WriteLine(msg);  // Lambda expression
    
    del1("Hello"); // Invoke the delegate
}

static void DisplayMessage(string msg)
{
    Console.WriteLine(msg);
}
```

### Invoking a Delegate

Delegates can be invoked in three ways:

```csharp
Func<int, int> square = x => x * x;

// Method 1: Direct invocation
int result1 = square(5);

// Method 2: Using Invoke() method
int result2 = square.Invoke(5);

// Method 3: Null-safe invocation (C# 6.0+)
int result3 = square?.Invoke(5) ?? 0;
```

## Types of Delegates

### 1. Custom Delegates

Delegates you define yourself with specific signatures:

```csharp
public delegate void WorkCompletedHandler(string workType);
public delegate int CalculationDelegate(int a, int b);

class Calculator
{
    public static int Add(int x, int y) => x + y;
    public static int Multiply(int x, int y) => x * y;
}

// Usage
CalculationDelegate operation = Calculator.Add;
int result = operation(5, 3); // result = 8
```

### 2. Built-in Generic Delegates

#### Action<T>
Represents a method that **performs an action** (no return value, returns void):

```csharp
// Action with no parameters
Action greet = () => Console.WriteLine("Hello!");
greet();

// Action with parameters
Action<string, int> displayInfo = (name, age) => 
    Console.WriteLine($"Name: {name}, Age: {age}");
displayInfo("John", 30);

// Can take up to 16 parameters
Action<int, int, int> calculate = (a, b, c) => 
    Console.WriteLine($"Sum: {a + b + c}");
```

#### Func<T>
Represents a method that **returns a value**. The last type parameter is always the return type:

```csharp
// Func<input1, input2, ..., return_type>

// Func with no parameters
Func<DateTime> getCurrentTime = () => DateTime.Now;
var now = getCurrentTime();

// Func with parameters
Func<int, int, int> add = (x, y) => x + y;
int sum = add(5, 3); // sum = 8

// Func with return type
Func<string, int> getLength = str => str.Length;
int length = getLength("Hello"); // length = 5

// Can take up to 16 parameters
Func<int, int, int, int> calculate = (a, b, c) => a + b + c;
```

#### Predicate<T>
Represents a method that **tests a condition** and returns a boolean:

```csharp
// Predicate<T> - returns bool for a single parameter

Predicate<int> isPositive = x => x > 0;
bool result1 = isPositive(5);  // true
bool result2 = isPositive(-3); // false

Predicate<string> isEmpty = str => string.IsNullOrEmpty(str);
bool result3 = isEmpty("");    // true

// Common use with collections
List<int> numbers = new List<int> { 1, -2, 3, -4, 5 };
List<int> positiveNumbers = numbers.FindAll(isPositive);
```

### 3. Singlecast vs. Multicast Delegates

#### Singlecast Delegates
Hold a reference to **one method** at a time. Assigning a new method **replaces** the previous reference:

```csharp
public delegate void Handler(string msg);

Handler del = Method1;
del("First");    // Calls Method1

del = Method2;   // Replaces reference to Method2
del("Second");   // Calls Method2 (Method1 is no longer referenced)

static void Method1(string msg) => Console.WriteLine($"Method1: {msg}");
static void Method2(string msg) => Console.WriteLine($"Method2: {msg}");
```

#### Multicast Delegates
Hold references to **multiple methods**. All methods are invoked in the order they were added (FIFO - First In First Out):

```csharp
public delegate void Handler(string msg);

Handler del = Method1;
del += Method2;  // Add Method2
del += Method3;  // Add Method3

del("Execute"); 
// Output:
// Method1: Execute
// Method2: Execute
// Method3: Execute

// Remove a method
del -= Method2;
del("Execute");
// Output:
// Method1: Execute
// Method3: Execute

static void Method1(string msg) => Console.WriteLine($"Method1: {msg}");
static void Method2(string msg) => Console.WriteLine($"Method2: {msg}");
static void Method3(string msg) => Console.WriteLine($"Method3: {msg}");
```

### 4. Generic Delegates

Delegates with type parameters for flexibility:

```csharp
// Custom generic delegate
public delegate void GenericHandler<T>(T data);

GenericHandler<string> stringHandler = msg => Console.WriteLine($"String: {msg}");
stringHandler("Hello");

GenericHandler<int> intHandler = num => Console.WriteLine($"Number: {num}");
intHandler(42);

// Generic delegate with return type
public delegate TResult GenericFunc<T, TResult>(T input);

GenericFunc<int, int> square = x => x * x;
GenericFunc<string, int> getLength = str => str.Length;
```

## Usage Patterns

### 1. Callbacks

Delegates enable callback mechanisms where a method notifies another part of the code when work is complete:

```csharp
public delegate void WorkCompletedHandler(int hoursWorked, string workType);
public delegate void WorkFinishedHandler(string workType);

public class Worker
{
    public void DoWork(int hours, string workType, 
        WorkCompletedHandler onProgress, 
        WorkFinishedHandler onComplete)
    {
        for (int i = 0; i < hours; i++)
        {
            Thread.Sleep(1000); // Simulate work
            onProgress?.Invoke(i + 1, workType); // Notify progress
        }
        
        onComplete?.Invoke(workType); // Notify completion
    }
}

// Usage
Worker worker = new Worker();
worker.DoWork(3, "Development",
    (hours, type) => Console.WriteLine($"Progress: {hours} hours ({type})"),
    (type) => Console.WriteLine($"Work completed: {type}"));
```

### 2. Filtering and Transformation

Using predicates and functions for LINQ-like operations:

```csharp
// Filter with Predicate
List<int> numbers = new List<int> { 1, 2, 3, 4, 5, 6 };

Predicate<int> isEven = n => n % 2 == 0;
List<int> evenNumbers = numbers.FindAll(isEven);

// Transform with Func
Func<int, int> square = x => x * x;
List<int> squared = numbers.ConvertAll(x => square(x));
```

### 3. Event Handling

Delegates are fundamental to event handling:

```csharp
public delegate void ButtonClickedHandler(object sender, EventArgs e);

public class Button
{
    public event ButtonClickedHandler OnClick;
    
    public void Click()
    {
        OnClick?.Invoke(this, EventArgs.Empty);
    }
}

// Usage
Button btn = new Button();
btn.OnClick += (sender, e) => Console.WriteLine("Button clicked!");
btn.OnClick += (sender, e) => Console.WriteLine("Handling another action!");
btn.Click();
```

### 4. Strategy Pattern

Delegates implement the strategy pattern for flexible behavior:

```csharp
public class PaymentProcessor
{
    private Func<decimal, bool> _paymentStrategy;
    
    public PaymentProcessor(Func<decimal, bool> strategy)
    {
        _paymentStrategy = strategy;
    }
    
    public void ProcessPayment(decimal amount)
    {
        if (_paymentStrategy(amount))
            Console.WriteLine($"Payment of ${amount} processed successfully.");
        else
            Console.WriteLine($"Payment of ${amount} failed.");
    }
}

// Usage
Func<decimal, bool> creditCardPayment = amount => 
{
    // Credit card processing logic
    return amount > 0 && amount < 100000;
};

Func<decimal, bool> bankTransferPayment = amount =>
{
    // Bank transfer logic
    return amount > 0 && amount < 50000;
};

PaymentProcessor processor1 = new PaymentProcessor(creditCardPayment);
processor1.ProcessPayment(50); // Credit card

PaymentProcessor processor2 = new PaymentProcessor(bankTransferPayment);
processor2.ProcessPayment(30); // Bank transfer
```

### 5. Asynchronous Operations

Delegates for asynchronous method invocation:

```csharp
public delegate int AsyncOperation(int x);

void PerformAsyncOperation()
{
    AsyncOperation operation = num =>
    {
        Thread.Sleep(2000); // Simulate long operation
        return num * 2;
    };
    
    IAsyncResult asyncResult = operation.BeginInvoke(5, null, null);
    
    // Do other work while operation completes
    Console.WriteLine("Operation started...");
    
    int result = operation.EndInvoke(asyncResult);
    Console.WriteLine($"Result: {result}");
}
```

## Best Practices

### 1. Null-Safety

Always check for null before invoking:

```csharp
// Bad
public void RaiseEvent(EventHandler handler)
{
    handler(this, EventArgs.Empty); // Can throw NullReferenceException
}

// Good - Traditional approach
public void RaiseEvent(EventHandler handler)
{
    if (handler != null)
        handler(this, EventArgs.Empty);
}

// Good - Null-coalescing operator
handler?.Invoke(this, EventArgs.Empty);

// Good - Traditional with local variable (thread-safe)
var temp = handler;
if (temp != null)
    temp(this, EventArgs.Empty);
```

### 2. Memory Leaks with Delegates

Be careful with captured variables and unsubscribing:

```csharp
// Potential memory leak
public class EventSubscriber
{
    private EventPublisher _publisher;
    
    public void Subscribe()
    {
        _publisher.OnEvent += (data) => 
            Console.WriteLine(this.GetHashCode()); // Captures 'this'
    }
    
    // Must unsubscribe or the EventSubscriber instance will be held in memory
    public void Unsubscribe()
    {
        _publisher.OnEvent -= (data) => 
            Console.WriteLine(this.GetHashCode());
        // This won't work! The lambda is a different instance
    }
}

// Better approach - store reference
public class EventSubscriber
{
    private EventPublisher _publisher;
    private EventHandler _handler;
    
    public void Subscribe()
    {
        _handler = (sender, e) => 
            Console.WriteLine(this.GetHashCode());
        _publisher.OnEvent += _handler;
    }
    
    public void Unsubscribe()
    {
        _publisher.OnEvent -= _handler;
    }
}
```

### 3. Use Built-in Delegates When Possible

Prefer `Action<T>`, `Func<T>`, and `Predicate<T>` over custom delegates:

```csharp
// Less ideal - custom delegate
public delegate void ProcessDataHandler(string data);

// Better - use Action<T>
public void ProcessData(Action<string> handler)
{
    handler("Some data");
}

// Even better - use Func<T> if a value is returned
public void ProcessData(Func<string, bool> handler)
{
    bool result = handler("Some data");
}
```

### 4. Performance Considerations

Delegates have minimal performance overhead in modern .NET:

```csharp
// Delegates are typically comparable to or faster than method calls
// The invocation cost is negligible for most real-world scenarios

// Prefer storing delegate references outside loops
Action<int> action = x => Console.WriteLine(x);
for (int i = 0; i < 1000000; i++)
{
    action(i); // More efficient than creating new delegate each iteration
}

// Instead of:
for (int i = 0; i < 1000000; i++)
{
    Action<int> action2 = x => Console.WriteLine(x);
    action2(i);
}
```

### 5. Thread Safety with Delegates

When invoking delegates in multithreaded scenarios:

```csharp
public class EventPublisher
{
    private EventHandler _handler;
    
    public event EventHandler OnEvent
    {
        add { _handler += value; }
        remove { _handler -= value; }
    }
    
    public void RaiseEvent()
    {
        // Thread-safe approach: read into local variable
        var handler = _handler;
        handler?.Invoke(this, EventArgs.Empty);
    }
}
```

## Real-World Examples

### Example 1: Notification System

```csharp
public delegate void NotificationHandler(string message);

public class NotificationCenter
{
    public event NotificationHandler OnNotification;
    
    public void SendNotification(string message)
    {
        OnNotification?.Invoke(message);
    }
}

public class EmailService
{
    public void SendEmail(string message)
    {
        Console.WriteLine($"Email sent: {message}");
    }
}

public class SMSService
{
    public void SendSMS(string message)
    {
        Console.WriteLine($"SMS sent: {message}");
    }
}

// Usage
NotificationCenter center = new NotificationCenter();
EmailService email = new EmailService();
SMSService sms = new SMSService();

center.OnNotification += email.SendEmail;
center.OnNotification += sms.SendSMS;

center.SendNotification("System Alert!");
// Output:
// Email sent: System Alert!
// SMS sent: System Alert!
```

### Example 2: Employee Promotion System

```csharp
public delegate bool EligibilityChecker(Employee employee);

public class Employee
{
    public int ID { get; set; }
    public string Name { get; set; }
    public int Experience { get; set; }
    public decimal Salary { get; set; }
}

public class HRDepartment
{
    public void ProcessPromotions(List<Employee> employees, 
        EligibilityChecker eligibilityCheck)
    {
        foreach (var employee in employees)
        {
            if (eligibilityCheck(employee))
            {
                Console.WriteLine($"{employee.Name} is promoted!");
                employee.Salary *= 1.10m; // 10% raise
            }
        }
    }
}

// Usage
HRDepartment hr = new HRDepartment();
List<Employee> employees = new List<Employee>
{
    new Employee { ID = 1, Name = "Alice", Experience = 5, Salary = 50000 },
    new Employee { ID = 2, Name = "Bob", Experience = 10, Salary = 70000 }
};

// Promotion criteria
EligibilityChecker managerPromotion = emp => emp.Experience >= 8;
hr.ProcessPromotions(employees, managerPromotion);

// Or using lambda
hr.ProcessPromotions(employees, emp => emp.Salary < 60000);
```

### Example 3: Data Processing Pipeline

```csharp
public class DataProcessor
{
    private List<Func<string, string>> _processors = new List<Func<string, string>>();
    
    public void AddProcessor(Func<string, string> processor)
    {
        _processors.Add(processor);
    }
    
    public string Process(string data)
    {
        foreach (var processor in _processors)
        {
            data = processor(data);
        }
        return data;
    }
}

// Usage
DataProcessor pipeline = new DataProcessor();
pipeline.AddProcessor(str => str.ToUpper());
pipeline.AddProcessor(str => str.Replace(" ", "_"));
pipeline.AddProcessor(str => $"[{str}]");

string result = pipeline.Process("hello world");
Console.WriteLine(result); // Output: [HELLO_WORLD]
```

### Example 4: Multicast Delegate with Return Values

```csharp
public delegate int MathDelegate(int a, int b);

class Program
{
    static void Main()
    {
        MathDelegate del1 = Add;
        MathDelegate del2 = Subtract;
        MathDelegate multicast = del1 + del2;
        
        // Note: Only the last method's return value is returned
        int result = multicast(10, 5);
        Console.WriteLine(result); // Output: 5 (from Subtract)
        
        // To get all results, iterate through invocation list
        Delegate[] invocationList = multicast.GetInvocationList();
        foreach (MathDelegate del in invocationList)
        {
            Console.WriteLine(del(10, 5));
        }
        // Output:
        // 15 (Add)
        // 5 (Subtract)
    }
    
    static int Add(int a, int b) => a + b;
    static int Subtract(int a, int b) => a - b;
}
```

## Important Delegate Properties and Methods

```csharp
Action<int> action = Console.WriteLine;

// Properties
MethodInfo method = action.Method;           // Gets method information
object target = action.Target;               // Gets target object (null for static)

// Methods
Delegate[] list = action.GetInvocationList(); // Gets all delegates in multicast chain
action.DynamicInvoke(42);                    // Dynamic invocation with object[]

// Combine and Remove
Action handler1 = () => Console.WriteLine("1");
Action handler2 = () => Console.WriteLine("2");
Action combined = (Action)Delegate.Combine(handler1, handler2);
Action removed = (Action)Delegate.Remove(combined, handler1);
```

## Summary

| Feature | Purpose |
|---------|---------|
| **Custom Delegates** | Define specific method signatures for your domain |
| **Action<T>** | Execute methods that return void |
| **Func<T>** | Execute methods that return a value |
| **Predicate<T>** | Test conditions and return boolean |
| **Multicast** | Call multiple methods with single invoke |
| **Events** | Publish-subscribe pattern with delegates |
| **Lambda** | Inline anonymous functions with delegates |

Delegates are fundamental to C# and provide powerful abstraction for method encapsulation, enabling flexible, maintainable, and scalable code architecture.