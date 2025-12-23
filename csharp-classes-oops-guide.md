# Complete Guide to C# Classes and OOP Concepts

## Table of Contents
1. Classes and Objects
2. Constructors
3. Object Creation
4. Access Modifiers
5. Static Keyword
6. Properties
7. Virtual, Override, and Sealed
8. New Keyword (Method Hiding)
9. Object-Oriented Programming (OOP) Principles

---

## 1. Classes and Objects

### What is a Class?
A class is a **blueprint or template** for creating objects. It defines the structure (fields, properties) and behavior (methods) that objects of that class will have.

### Class Declaration Syntax
```csharp
public class ClassName
{
    // Fields (member variables)
    public int id;
    public string name;
    
    // Properties
    public int Age { get; set; }
    
    // Methods
    public void PrintDetails()
    {
        Console.WriteLine($"ID: {id}, Name: {name}");
    }
}
```

### Key Characteristics
- **Fields**: Data belonging to the class
- **Methods**: Behaviors or actions the class can perform
- **Properties**: Controlled access to data
- **Events**: Notifications the class can raise

### Example
```csharp
// Class Definition
public class Person
{
    public string Name;
    public int Age;
    
    public void DisplayInfo()
    {
        Console.WriteLine($"Name: {Name}, Age: {Age}");
    }
}

// Creating an object (instance)
Person person = new Person();
person.Name = "John";
person.Age = 30;
person.DisplayInfo(); // Output: Name: John, Age: 30
```

---

## 2. Constructors

### What is a Constructor?
A constructor is a special method that is **automatically called when an object is created**. It initializes the object's state and sets initial values for fields.

### Constructor Characteristics
- **Same name as the class**
- **No return type** (not even void)
- **Automatically invoked** when using the `new` keyword
- **Can be overloaded** with different signatures

### Types of Constructors

#### 2.1 Default Constructor (Parameterless)
If no constructor is defined, the compiler provides an implicit default constructor that initializes all fields to their default values.

```csharp
public class Student
{
    public string Name;
    public int RollNumber;
    
    // Explicit default constructor
    public Student()
    {
        Name = "Unknown";
        RollNumber = 0;
        Console.WriteLine("Default constructor called");
    }
}

// Usage
Student student = new Student(); // Output: Default constructor called
```

#### 2.2 Parameterized Constructor
Accepts parameters to initialize object properties with specific values.

```csharp
public class Car
{
    public string Brand;
    public int Year;
    
    // Parameterized constructor
    public Car(string brand, int year)
    {
        Brand = brand;
        Year = year;
        Console.WriteLine("Parameterized constructor called");
    }
}

// Usage
Car car = new Car("Toyota", 2023);
Console.WriteLine($"Brand: {car.Brand}, Year: {car.Year}");
// Output: Brand: Toyota, Year: 2023
```

#### 2.3 Constructor Overloading
Multiple constructors with different parameter signatures in the same class.

```csharp
public class Employee
{
    public string Name;
    public int EmployeeId;
    public string Department;
    
    // Constructor 1: No parameters
    public Employee()
    {
        Name = "Unknown";
        EmployeeId = 0;
        Department = "IT";
    }
    
    // Constructor 2: Name only
    public Employee(string name)
    {
        Name = name;
        EmployeeId = 0;
        Department = "IT";
    }
    
    // Constructor 3: Name and ID
    public Employee(string name, int id)
    {
        Name = name;
        EmployeeId = id;
        Department = "IT";
    }
    
    // Constructor 4: All parameters
    public Employee(string name, int id, string dept)
    {
        Name = name;
        EmployeeId = id;
        Department = dept;
    }
}

// Usage
Employee emp1 = new Employee();
Employee emp2 = new Employee("Alice");
Employee emp3 = new Employee("Bob", 101);
Employee emp4 = new Employee("Charlie", 102, "HR");
```

#### 2.4 Copy Constructor
A constructor that accepts an object of the same class as a parameter.

```csharp
public class Product
{
    public string Name;
    public decimal Price;
    
    public Product(string name, decimal price)
    {
        Name = name;
        Price = price;
    }
    
    // Copy constructor
    public Product(Product other)
    {
        Name = other.Name;
        Price = other.Price;
    }
}

// Usage
Product product1 = new Product("Laptop", 50000);
Product product2 = new Product(product1);
Console.WriteLine($"Name: {product2.Name}, Price: {product2.Price}");
```

#### 2.5 Static Constructor
Initializes static (class-level) fields. Called only once before any instance is created or static member is accessed.

```csharp
public class Configuration
{
    public static string AppName;
    public static int MaxUsers;
    
    // Static constructor
    static Configuration()
    {
        AppName = "MyApp";
        MaxUsers = 100;
        Console.WriteLine("Static constructor called");
    }
}

// Usage
Console.WriteLine(Configuration.AppName); // Output: Static constructor called, then "MyApp"
```

**Key Points about Static Constructors:**
- Cannot have access modifiers (public, private, etc.)
- Cannot take parameters
- Cannot be overloaded
- Executed only once during the application lifetime
- Useful for initializing static fields and one-time operations

---

## 3. Object Creation

### The New Keyword
The `new` keyword allocates memory for an object and calls the constructor.

```csharp
// Basic syntax
ClassName objectName = new ClassName();

// With parameters
ClassName objectName = new ClassName(param1, param2);
```

### Object Creation Process
1. **Memory Allocation**: Space is allocated on the heap
2. **Constructor Call**: The appropriate constructor is invoked
3. **Reference Assignment**: A reference to the object is assigned to the variable

### Example
```csharp
public class Book
{
    public string Title;
    public string Author;
    
    public Book(string title, string author)
    {
        Title = title;
        Author = author;
    }
}

// Object creation
Book book = new Book("C# Programming", "John Doe");
Console.WriteLine($"Title: {book.Title}, Author: {book.Author}");
```

### Object Initializer Syntax
Sets properties directly during object creation without a parameterized constructor.

```csharp
public class Person
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public int Age { get; set; }
}

// Using object initializer
Person person = new Person 
{ 
    FirstName = "John", 
    LastName = "Doe", 
    Age = 30 
};
```

---

## 4. Access Modifiers

Access modifiers control the visibility of class members (fields, methods, properties).

| Modifier | Description | Accessibility |
|----------|-------------|-----------------|
| `public` | Accessible from anywhere | All classes, any assembly |
| `private` | Accessible only within the same class | Same class only |
| `protected` | Accessible in the same class and derived classes | Same class + derived classes |
| `internal` | Accessible within the same assembly | Same assembly only |
| `protected internal` | Accessible in same assembly or derived classes in other assemblies | Same assembly + derived classes |
| `private protected` | Accessible in same class or derived classes within same assembly | Same class + derived classes (same assembly only) |

### Examples

#### Public Access
```csharp
public class BankAccount
{
    public string AccountNumber { get; set; }
    
    public void Withdraw(decimal amount)
    {
        // Implementation
    }
}

// Can be accessed from any class
BankAccount account = new BankAccount();
account.Withdraw(1000);
```

#### Private Access
```csharp
public class Employee
{
    private decimal salary; // Only accessible within this class
    
    public void SetSalary(decimal amount)
    {
        if (amount > 0)
            salary = amount;
    }
}

// This will cause an error:
// Employee emp = new Employee();
// emp.salary = 50000; // Error: salary is private
```

#### Protected Access
```csharp
public class Animal
{
    protected string Name; // Accessible in Animal and derived classes
    
    protected void Eat()
    {
        Console.WriteLine($"{Name} is eating");
    }
}

public class Dog : Animal
{
    public void Feed()
    {
        Eat(); // Can access protected member
        Name = "Buddy"; // Can access protected field
    }
}
```

#### Internal Access
```csharp
internal class Utility // Only accessible within same assembly
{
    internal static void PrintMessage(string msg)
    {
        Console.WriteLine(msg);
    }
}
```

---

## 5. Static Keyword

The `static` keyword indicates that a member belongs to the **class itself** rather than to individual instances.

### Static Fields
```csharp
public class Student
{
    public static int TotalStudents = 0; // Shared by all instances
    public string Name;
    
    public Student(string name)
    {
        Name = name;
        TotalStudents++; // Increment shared counter
    }
}

// Usage
Student student1 = new Student("Alice");
Student student2 = new Student("Bob");
Student student3 = new Student("Charlie");

Console.WriteLine($"Total: {Student.TotalStudents}"); // Output: Total: 3
```

### Static Methods
```csharp
public class MathHelper
{
    public static int Add(int a, int b)
    {
        return a + b;
    }
    
    public static int Multiply(int a, int b)
    {
        return a * b;
    }
}

// Usage - no instance needed
int sum = MathHelper.Add(5, 10); // Output: 15
int product = MathHelper.Multiply(5, 10); // Output: 50
```

### Static vs Instance Members
```csharp
public class Counter
{
    public static int StaticCount = 0; // Shared
    public int InstanceCount = 0; // Individual per instance
    
    public void Increment()
    {
        StaticCount++; // All instances share this
        InstanceCount++; // Each instance has its own
    }
}

// Usage
Counter c1 = new Counter();
Counter c2 = new Counter();

c1.Increment();
c1.Increment();
c2.Increment();

Console.WriteLine($"Static: {Counter.StaticCount}"); // Output: 3 (shared)
Console.WriteLine($"Instance c1: {c1.InstanceCount}"); // Output: 2
Console.WriteLine($"Instance c2: {c2.InstanceCount}"); // Output: 1
```

### Static Classes
A static class contains only static members and cannot be instantiated.

```csharp
public static class Logger
{
    public static void Log(string message)
    {
        Console.WriteLine($"[LOG] {DateTime.Now}: {message}");
    }
    
    public static void LogError(string error)
    {
        Console.WriteLine($"[ERROR] {DateTime.Now}: {error}");
    }
}

// Usage
Logger.Log("Application started");
Logger.LogError("An error occurred");
```

**Key Points:**
- Static members are created once and shared by all instances
- No instance needed to access static members
- Useful for utility functions and shared data
- Cannot access instance members from static context

---

## 6. Properties

Properties provide controlled access to fields using getters and setters.

### Auto-Implemented Properties
```csharp
public class Person
{
    // Auto-implemented property
    public string Name { get; set; }
    public int Age { get; set; }
}

// Usage
Person person = new Person { Name = "John", Age = 30 };
Console.WriteLine($"Name: {person.Name}, Age: {person.Age}");
```

### Property with Backing Field
```csharp
public class BankAccount
{
    private decimal balance; // Backing field
    
    public decimal Balance
    {
        get { return balance; }
        set 
        { 
            if (value >= 0)
                balance = value;
        }
    }
}

// Usage
BankAccount account = new BankAccount();
account.Balance = 1000;
Console.WriteLine($"Balance: {account.Balance}");
```

### Read-Only Properties
```csharp
public class Product
{
    public string Name { get; set; }
    public decimal Price { get; set; }
    
    // Read-only property (only getter)
    public decimal DiscountedPrice 
    { 
        get { return Price * 0.9m; } 
    }
}

// Usage
Product product = new Product { Name = "Laptop", Price = 50000 };
Console.WriteLine($"Discounted: {product.DiscountedPrice}");
// product.DiscountedPrice = 45000; // Error: no setter
```

### Write-Only Properties
```csharp
public class SecureField
{
    private string encryptedValue;
    
    public string Value
    {
        set { encryptedValue = Encrypt(value); }
    }
    
    private string Encrypt(string value)
    {
        return Convert.ToBase64String(System.Text.Encoding.UTF8.GetBytes(value));
    }
}
```

### Property with Different Access Levels
```csharp
public class User
{
    private string password;
    
    public string Username { get; set; }
    
    // Public getter, private setter
    public string Email { get; private set; }
    
    public void SetEmail(string email)
    {
        Email = email;
    }
}
```

---

## 7. Virtual, Override, and Sealed

These keywords control method behavior in inheritance hierarchies.

### Virtual Methods
A `virtual` method can be overridden in derived classes.

```csharp
public class Animal
{
    // Virtual method - can be overridden
    public virtual void Speak()
    {
        Console.WriteLine("Animal speaks");
    }
}

public class Dog : Animal
{
    // Override the virtual method
    public override void Speak()
    {
        Console.WriteLine("Dog barks");
    }
}

// Usage
Animal animal = new Animal();
animal.Speak(); // Output: Animal speaks

Dog dog = new Dog();
dog.Speak(); // Output: Dog barks

Animal animalRef = new Dog(); // Reference is Animal, object is Dog
animalRef.Speak(); // Output: Dog barks (polymorphism)
```

### Override
The `override` keyword explicitly states that a method overrides a virtual method from the base class.

```csharp
public class Vehicle
{
    public virtual void Start()
    {
        Console.WriteLine("Vehicle starts");
    }
}

public class Car : Vehicle
{
    public override void Start()
    {
        Console.WriteLine("Car engine starts");
    }
}

public class Motorcycle : Vehicle
{
    public override void Start()
    {
        Console.WriteLine("Motorcycle engine starts");
    }
}

// Usage
Vehicle car = new Car();
car.Start(); // Output: Car engine starts

Vehicle motorcycle = new Motorcycle();
motorcycle.Start(); // Output: Motorcycle engine starts
```

### Sealed Classes
A `sealed` class cannot be inherited.

```csharp
public sealed class FinalClass
{
    public void DoSomething()
    {
        Console.WriteLine("Doing something");
    }
}

// This will cause an error:
// public class DerivedClass : FinalClass { } // Error: cannot inherit sealed class
```

### Sealed Methods
A `sealed` method prevents further overriding in derived classes.

```csharp
public class Base
{
    public virtual void Method1()
    {
        Console.WriteLine("Base.Method1");
    }
    
    public virtual void Method2()
    {
        Console.WriteLine("Base.Method2");
    }
}

public class Derived : Base
{
    // Seal this override - no further override allowed
    public sealed override void Method1()
    {
        Console.WriteLine("Derived.Method1");
    }
    
    public override void Method2()
    {
        Console.WriteLine("Derived.Method2");
    }
}

public class DoublyDerived : Derived
{
    // This is OK
    public override void Method2()
    {
        Console.WriteLine("DoublyDerived.Method2");
    }
    
    // Error: cannot override sealed method
    // public override void Method1() { }
}
```

---

## 8. New Keyword (Method Hiding)

The `new` keyword hides a base class method instead of overriding it.

### Difference: Override vs New (Hiding)
```csharp
public class BaseClass
{
    public virtual void PrintMessage()
    {
        Console.WriteLine("Base message");
    }
}

public class DerivedClassOverride : BaseClass
{
    // Override: provides new implementation
    public override void PrintMessage()
    {
        Console.WriteLine("Derived override message");
    }
}

public class DerivedClassHiding : BaseClass
{
    // New: hides the base method
    public new void PrintMessage()
    {
        Console.WriteLine("Derived new message");
    }
}

// Usage
BaseClass ref1 = new DerivedClassOverride();
ref1.PrintMessage(); // Output: Derived override message

BaseClass ref2 = new DerivedClassHiding();
ref2.PrintMessage(); // Output: Base message (hides new method)

DerivedClassHiding derived = new DerivedClassHiding();
derived.PrintMessage(); // Output: Derived new message
```

### Key Differences
```csharp
public class Animal
{
    public void MakeSound()
    {
        Console.WriteLine("Animal sound");
    }
}

public class Cat : Animal
{
    public new void MakeSound()
    {
        Console.WriteLine("Meow");
    }
}

// Usage
Cat cat = new Cat();
cat.MakeSound(); // Output: Meow

Animal animalRef = new Cat();
animalRef.MakeSound(); // Output: Animal sound (calls base method)
```

---

## 9. Object-Oriented Programming (OOP) Principles

OOP is built on four main principles:

### 9.1 Encapsulation
Bundling data and methods together and hiding internal details from the outside world.

```csharp
public class BankAccount
{
    // Private field - hidden from outside
    private decimal balance;
    
    // Public property - controlled access
    public decimal Balance
    {
        get { return balance; }
        private set { balance = value; }
    }
    
    // Public methods - interface to interact with the class
    public void Deposit(decimal amount)
    {
        if (amount > 0)
            balance += amount;
    }
    
    public bool Withdraw(decimal amount)
    {
        if (amount > 0 && amount <= balance)
        {
            balance -= amount;
            return true;
        }
        return false;
    }
}

// Usage
BankAccount account = new BankAccount();
account.Deposit(5000);
account.Withdraw(1000);
Console.WriteLine($"Balance: {account.Balance}"); // Can only access through property
// account.balance = -500; // Error: cannot access private field
```

### 9.2 Abstraction
Simplifying complex systems by showing only essential features while hiding implementation details.

```csharp
// Abstract base class
public abstract class Database
{
    // Abstract method - must be implemented by derived classes
    public abstract void Connect();
    public abstract void Execute(string query);
    
    // Concrete method
    public void LogConnection()
    {
        Console.WriteLine("Connecting to database...");
    }
}

public class SqlDatabase : Database
{
    public override void Connect()
    {
        Console.WriteLine("Connecting to SQL Server");
    }
    
    public override void Execute(string query)
    {
        Console.WriteLine($"Executing SQL: {query}");
    }
}

public class MongoDatabase : Database
{
    public override void Connect()
    {
        Console.WriteLine("Connecting to MongoDB");
    }
    
    public override void Execute(string query)
    {
        Console.WriteLine($"Executing MongoDB query: {query}");
    }
}

// Usage
Database db = new SqlDatabase();
db.LogConnection();
db.Connect();
db.Execute("SELECT * FROM Users");
```

### 9.3 Inheritance
Creating a new class based on an existing class, inheriting its properties and methods.

```csharp
// Base class
public class Vehicle
{
    public string Brand { get; set; }
    public string Color { get; set; }
    
    public virtual void Start()
    {
        Console.WriteLine("Vehicle starting");
    }
}

// Derived class
public class Car : Vehicle
{
    public int NumberOfDoors { get; set; }
    
    public override void Start()
    {
        Console.WriteLine($"{Brand} car starting with {NumberOfDoors} doors");
    }
}

// Usage
Car car = new Car 
{ 
    Brand = "Toyota", 
    Color = "Red", 
    NumberOfDoors = 4 
};
car.Start(); // Output: Toyota car starting with 4 doors
```

### 9.4 Polymorphism
Objects of different types can be treated through the same interface, and each responds in its own way.

```csharp
// Base class
public class Shape
{
    public virtual void Draw()
    {
        Console.WriteLine("Drawing shape");
    }
}

// Derived classes
public class Circle : Shape
{
    public override void Draw()
    {
        Console.WriteLine("Drawing circle");
    }
}

public class Square : Shape
{
    public override void Draw()
    {
        Console.WriteLine("Drawing square");
    }
}

public class Triangle : Shape
{
    public override void Draw()
    {
        Console.WriteLine("Drawing triangle");
    }
}

// Usage
List<Shape> shapes = new List<Shape>
{
    new Circle(),
    new Square(),
    new Triangle()
};

foreach (Shape shape in shapes)
{
    shape.Draw();
    // Output:
    // Drawing circle
    // Drawing square
    // Drawing triangle
}
```

---

## Summary

| Concept | Purpose | Usage |
|---------|---------|-------|
| **Class** | Blueprint for objects | Define structure and behavior |
| **Constructor** | Initialize objects | Set initial values |
| **Access Modifiers** | Control visibility | Protect data, manage access |
| **Static** | Class-level members | Shared across all instances |
| **Properties** | Controlled field access | Getters and setters |
| **Virtual/Override** | Polymorphic behavior | Enable method overriding |
| **Sealed** | Prevent inheritance/overriding | Restrict extension |
| **New** | Hide base method | Alternative to override |
| **Encapsulation** | Bundle data and methods | Hide implementation details |
| **Abstraction** | Simplify complexity | Focus on essentials |
| **Inheritance** | Code reuse | Create class hierarchies |
| **Polymorphism** | Multiple forms | Same interface, different behaviors |

---

## Best Practices

1. **Use access modifiers wisely** - Keep fields private, expose through properties
2. **Leverage encapsulation** - Hide implementation details
3. **Prefer composition over inheritance** - Consider object composition when appropriate
4. **Use virtual/override for polymorphism** - Mark methods as virtual if they might be overridden
5. **Static for utilities** - Use static classes and methods for utility functions
6. **Properties for controlled access** - Use properties instead of public fields
7. **Constructor overloading** - Provide multiple constructors for flexibility
8. **Comment complex logic** - Document abstract and inherited behaviors