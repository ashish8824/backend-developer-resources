# Java OOP — Complete Guide with Real-Life Examples (Basic to Advanced)

---

## Table of Contents

1. [What is OOP?](#1-what-is-oop)
2. [Class & Object](#2-class--object)
3. [Constructors](#3-constructors)
4. [this Keyword](#4-this-keyword)
5. [static Keyword](#5-static-keyword)
6. [Encapsulation](#6-encapsulation)
7. [Inheritance](#7-inheritance)
8. [super Keyword](#8-super-keyword)
9. [Method Overriding](#9-method-overriding)
10. [Polymorphism](#10-polymorphism)
    - [Compile-time (Method Overloading)](#101-compile-time-polymorphism--method-overloading)
    - [Runtime (Method Overriding)](#102-runtime-polymorphism--method-overriding)
11. [Abstraction](#11-abstraction)
    - [Abstract Class](#111-abstract-class)
    - [Interface](#112-interface)
    - [Abstract Class vs Interface](#113-abstract-class-vs-interface)
12. [final Keyword](#12-final-keyword)
13. [instanceof Operator](#13-instanceof-operator)
14. [Object Class Methods](#14-object-class-methods)
15. [Inner Classes](#15-inner-classes)
16. [Enums](#16-enums)
17. [Generics](#17-generics)
18. [SOLID Principles](#18-solid-principles)
19. [Design Patterns (OOP in Action)](#19-design-patterns-oop-in-action)
20. [Summary Cheat Sheet](#20-summary-cheat-sheet)

---

## 1. What is OOP?

**Object-Oriented Programming (OOP)** is a programming paradigm that organises code around **objects** — entities that bundle together **data (fields)** and **behaviour (methods)** — just like real-world things.

### Real-Life Analogy

Think of a **Car**:
- **Data (attributes):** brand, colour, speed, fuel level
- **Behaviour (actions):** start, accelerate, brake, refuel

In OOP, you model this as a `Car` class. Every actual car you drive (Honda Civic, Toyota Camry) is an **object** — an instance of that class.

### 4 Pillars of OOP

| Pillar          | Real-Life Analogy                                                       |
|-----------------|-------------------------------------------------------------------------|
| **Encapsulation** | ATM machine — you interact via buttons; internal cash logic is hidden |
| **Inheritance**   | A SportsCar IS-A Car — it inherits all car properties + adds its own  |
| **Polymorphism**  | A "shape" can be a circle, square, or triangle — draw() behaves differently |
| **Abstraction**   | Driving a car — you use the steering wheel without knowing engine internals |

---

## 2. Class & Object

### Class — The Blueprint

A **class** is a template or blueprint. It defines what data and behaviour objects of that type will have.

### Object — The Instance

An **object** is a concrete instance created from a class. You can create many objects from one class.

```
CLASS (Blueprint)           OBJECTS (Instances)
┌─────────────────┐        ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│    Employee     │  ───►  │ emp1: Alice  │  │ emp2: Bob    │  │ emp3: Charlie│
│  - name         │        │ name="Alice" │  │ name="Bob"   │  │ name="Charlie│
│  - salary       │        │ salary=50000 │  │ salary=60000 │  │ salary=75000 │
│  - department   │        │ dept="HR"    │  │ dept="Tech"  │  │ dept="Sales" │
│  + work()       │        └──────────────┘  └──────────────┘  └──────────────┘
│  + getSalary()  │
└─────────────────┘
```

### Real-Life Example: Employee Management System

```java
// CLASS — defines the blueprint of an Employee
public class Employee {

    // ── Fields (data / state) ──────────────────────
    String name;
    int    employeeId;
    double salary;
    String department;

    // ── Methods (behaviour) ───────────────────────
    void work() {
        System.out.println(name + " from " + department + " is working.");
    }

    void displayInfo() {
        System.out.println("──────────────────────────");
        System.out.println("ID         : " + employeeId);
        System.out.println("Name       : " + name);
        System.out.println("Department : " + department);
        System.out.println("Salary     : ₹" + salary);
    }

    double getAnnualSalary() {
        return salary * 12;
    }
}

public class Main {
    public static void main(String[] args) {

        // Creating OBJECTS from the Employee class
        Employee emp1 = new Employee();  // 'new' allocates memory
        emp1.name       = "Alice";
        emp1.employeeId = 101;
        emp1.salary     = 75000;
        emp1.department = "Engineering";

        Employee emp2 = new Employee();
        emp2.name       = "Bob";
        emp2.employeeId = 102;
        emp2.salary     = 60000;
        emp2.department = "Marketing";

        // Using object methods
        emp1.displayInfo();
        emp1.work();
        System.out.println("Annual Salary: ₹" + emp1.getAnnualSalary());

        emp2.displayInfo();
        emp2.work();
    }
}
```

**Output:**
```
──────────────────────────
ID         : 101
Name       : Alice
Department : Engineering
Salary     : ₹75000.0
Alice from Engineering is working.
Annual Salary: ₹900000.0
──────────────────────────
ID         : 102
Name       : Bob
Department : Marketing
Salary     : ₹60000.0
Bob from Marketing is working.
```

---

## 3. Constructors

A **constructor** is a special method that is automatically called when an object is created with `new`. It **initialises** the object's state.

### Rules
- Same name as the class
- No return type (not even void)
- Called automatically when object is created

### Types of Constructors

```java
public class BankAccount {

    String accountNumber;
    String holderName;
    double balance;
    String accountType;

    // ── 1. Default Constructor (no arguments) ─────────────
    // Java provides this automatically if you write none.
    // If you define any constructor, Java stops providing this.
    BankAccount() {
        accountNumber = "ACC000";
        holderName    = "Unknown";
        balance       = 0.0;
        accountType   = "Savings";
        System.out.println("Default account created.");
    }

    // ── 2. Parameterised Constructor ──────────────────────
    BankAccount(String accountNumber, String holderName, double balance) {
        this.accountNumber = accountNumber;
        this.holderName    = holderName;
        this.balance       = balance;
        this.accountType   = "Savings"; // default type
    }

    // ── 3. Constructor Overloading ────────────────────────
    BankAccount(String accountNumber, String holderName, double balance, String accountType) {
        this.accountNumber = accountNumber;
        this.holderName    = holderName;
        this.balance       = balance;
        this.accountType   = accountType;
    }

    // ── 4. Copy Constructor ───────────────────────────────
    BankAccount(BankAccount other) {
        this.accountNumber = other.accountNumber + "-COPY";
        this.holderName    = other.holderName;
        this.balance       = other.balance;
        this.accountType   = other.accountType;
    }

    void displayInfo() {
        System.out.printf("Account: %-12s | Holder: %-10s | Balance: ₹%-10.2f | Type: %s%n",
            accountNumber, holderName, balance, accountType);
    }

    void deposit(double amount) {
        balance += amount;
        System.out.println("Deposited ₹" + amount + ". New balance: ₹" + balance);
    }
}

public class Main {
    public static void main(String[] args) {

        BankAccount acc1 = new BankAccount();  // Default constructor
        BankAccount acc2 = new BankAccount("ACC101", "Alice", 50000.0);
        BankAccount acc3 = new BankAccount("ACC102", "Bob", 25000.0, "Current");
        BankAccount acc4 = new BankAccount(acc2);  // Copy constructor

        acc1.displayInfo();
        acc2.displayInfo();
        acc3.displayInfo();
        acc4.displayInfo();

        acc2.deposit(10000);
        // acc4 balance unchanged — it's a separate copy
        System.out.println("acc4 balance still: ₹" + acc4.balance);
    }
}
```

**Output:**
```
Default account created.
Account: ACC000       | Holder: Unknown    | Balance: ₹0.00       | Type: Savings
Account: ACC101       | Holder: Alice      | Balance: ₹50000.00   | Type: Savings
Account: ACC102       | Holder: Bob        | Balance: ₹25000.00   | Type: Current
Account: ACC101-COPY  | Holder: Alice      | Balance: ₹50000.00   | Type: Savings
Deposited ₹10000.0. New balance: ₹60000.0
acc4 balance still: ₹50000.0
```

---

## 4. this Keyword

`this` refers to the **current object** — the object whose method or constructor is currently being executed.

### 4 Uses of `this`

```java
public class Student {
    String name;
    int    rollNo;
    double gpa;

    // ── Use 1: Resolve naming conflict (field vs parameter) ──
    Student(String name, int rollNo, double gpa) {
        this.name   = name;    // 'this.name'  = field; 'name'  = parameter
        this.rollNo = rollNo;
        this.gpa    = gpa;
    }

    // ── Use 2: Call another constructor (constructor chaining) ──
    Student(String name, int rollNo) {
        this(name, rollNo, 0.0);  // calls 3-argument constructor
        // MUST be the FIRST statement in constructor
    }

    // ── Use 3: Pass current object as argument ───────────────
    void register(University university) {
        university.enroll(this);  // passing current Student object
    }

    // ── Use 4: Return current object (method chaining / builder) ──
    Student setName(String name) {
        this.name = name;
        return this;             // enables method chaining
    }

    Student setGpa(double gpa) {
        this.gpa = gpa;
        return this;
    }

    void display() {
        System.out.println("Roll: " + rollNo + " | Name: " + name + " | GPA: " + gpa);
    }
}

class University {
    void enroll(Student s) {
        System.out.println("Enrolling student: " + s.name + " (Roll: " + s.rollNo + ")");
    }
}

public class Main {
    public static void main(String[] args) {

        Student s1 = new Student("Alice", 101, 3.9);
        Student s2 = new Student("Bob", 102);    // uses 2-arg constructor

        s1.display();
        s2.display();

        // Method chaining using 'return this'
        Student s3 = new Student("", 103);
        s3.setName("Charlie").setGpa(3.7);       // chain calls on same object
        s3.display();

        // Pass current object
        University uni = new University();
        s1.register(uni);
    }
}
```

---

## 5. static Keyword

`static` members **belong to the class**, not to any individual object. They are shared across all instances.

### Real-Life Analogy
Think of a `Hospital` class:
- Each `Patient` object has its own `name`, `age`, `diagnosis` (instance fields)
- But `totalPatients` admitted is a property of the **hospital itself** — shared across all — that's `static`

```java
public class Hospital {

    // ── Static fields — shared by ALL instances ────────────
    static String hospitalName = "City General Hospital";
    static int    totalPatients = 0;

    // ── Instance fields — unique per object ────────────────
    String patientName;
    int    patientId;
    String ward;
    double billAmount;

    // ── Static block — runs ONCE when class is loaded ──────
    static {
        System.out.println("Hospital system initialised: " + hospitalName);
        // Could load config, connect to DB, etc.
    }

    // ── Constructor ───────────────────────────────────────
    Hospital(String patientName, String ward) {
        totalPatients++;            // increment shared counter
        this.patientId   = totalPatients;
        this.patientName = patientName;
        this.ward        = ward;
        this.billAmount  = 0.0;
        System.out.println("Patient admitted: " + patientName + " [ID: " + patientId + "]");
    }

    // ── Static method — can only use static fields/methods ─
    static void showHospitalStats() {
        System.out.println("Hospital  : " + hospitalName);
        System.out.println("Total Patients : " + totalPatients);
    }

    // ── Instance method ───────────────────────────────────
    void addCharge(double amount) {
        this.billAmount += amount;
    }

    void displayBill() {
        System.out.printf("Patient: %-15s | Ward: %-10s | Bill: ₹%.2f%n",
            patientName, ward, billAmount);
    }
}

public class Main {
    public static void main(String[] args) {

        Hospital.showHospitalStats();  // called on class, not object

        Hospital p1 = new Hospital("Alice",   "Cardiology");
        Hospital p2 = new Hospital("Bob",     "Orthopaedics");
        Hospital p3 = new Hospital("Charlie", "General");

        p1.addCharge(15000);
        p1.addCharge(3500);
        p2.addCharge(22000);

        p1.displayBill();
        p2.displayBill();
        p3.displayBill();

        System.out.println();
        Hospital.showHospitalStats();
    }
}
```

**Output:**
```
Hospital system initialised: City General Hospital
Hospital  : City General Hospital
Total Patients : 0
Patient admitted: Alice [ID: 1]
Patient admitted: Bob [ID: 2]
Patient admitted: Charlie [ID: 3]
Patient: Alice           | Ward: Cardiology  | Bill: ₹18500.00
Patient: Bob             | Ward: Orthopaedics| Bill: ₹22000.00
Patient: Charlie         | Ward: General     | Bill: ₹0.00

Hospital  : City General Hospital
Total Patients : 3
```

### static vs Instance — Quick Comparison

| Feature            | `static`                        | Instance                        |
|--------------------|---------------------------------|---------------------------------|
| Belongs to         | Class                           | Object                          |
| Memory             | Allocated once (class load)     | Allocated per object            |
| Access             | `ClassName.member`              | `object.member`                 |
| Can access         | Only other static members       | Both static and instance members|
| Use case           | Counters, constants, utilities  | Object-specific data            |

---

## 6. Encapsulation

**Encapsulation** = bundling data (fields) + methods into one unit (class) AND **restricting direct access** to the data using access modifiers.

### Real-Life Analogy

An **ATM machine**:
- You can deposit, withdraw, check balance (the **public interface**)
- You cannot directly reach in and grab cash or modify the database (the **hidden internals**)

This is encapsulation — controlled access through defined methods.

### Access Modifiers

| Modifier    | Same Class | Same Package | Subclass | Everywhere |
|-------------|:----------:|:------------:|:--------:|:----------:|
| `private`   | ✅          | ❌            | ❌        | ❌          |
| `default`   | ✅          | ✅            | ❌        | ❌          |
| `protected` | ✅          | ✅            | ✅        | ❌          |
| `public`    | ✅          | ✅            | ✅        | ✅          |

### Real-Life Example: Medical Record System

```java
public class MedicalRecord {

    // ── Private fields — NOBODY outside can access directly ──
    private String  patientName;
    private int     age;
    private double  weight;         // in kg
    private double  height;         // in cm
    private String  bloodGroup;
    private String  confidentialNotes; // very sensitive

    // ── Constructor ───────────────────────────────────────
    public MedicalRecord(String name, int age, double weight, double height, String bloodGroup) {
        this.patientName = name;
        setAge(age);        // use setter for validation even in constructor
        setWeight(weight);
        this.height      = height;
        this.bloodGroup  = bloodGroup;
    }

    // ── Getters — controlled READ access ─────────────────
    public String getName()      { return patientName; }
    public int    getAge()       { return age; }
    public String getBloodGroup(){ return bloodGroup; }

    public double getBMI() {
        double heightM = height / 100.0;
        return Math.round((weight / (heightM * heightM)) * 10.0) / 10.0;
    }

    // ── Setters — controlled WRITE access with validation ─
    public void setAge(int age) {
        if (age < 0 || age > 150) {
            throw new IllegalArgumentException("Invalid age: " + age);
        }
        this.age = age;
    }

    public void setWeight(double weight) {
        if (weight <= 0 || weight > 500) {
            throw new IllegalArgumentException("Invalid weight: " + weight);
        }
        this.weight = weight;
    }

    // confidentialNotes has NO getter — completely hidden
    public void addNote(String note, String doctorPassword) {
        if ("DOC#2024".equals(doctorPassword)) {
            this.confidentialNotes += " | " + note;
            System.out.println("Note added securely.");
        } else {
            System.out.println("Access denied! Invalid doctor password.");
        }
    }

    public void displayPublicInfo() {
        System.out.println("──────────────────────────────────");
        System.out.println("Patient : " + patientName);
        System.out.println("Age     : " + age);
        System.out.println("Blood   : " + bloodGroup);
        System.out.println("BMI     : " + getBMI() + " (" + getBMICategory() + ")");
    }

    private String getBMICategory() {  // private helper — internal use only
        double bmi = getBMI();
        if (bmi < 18.5) return "Underweight";
        if (bmi < 25.0) return "Normal";
        if (bmi < 30.0) return "Overweight";
        return "Obese";
    }
}

public class Main {
    public static void main(String[] args) {
        MedicalRecord record = new MedicalRecord("Alice", 30, 65.0, 165.0, "B+");

        record.displayPublicInfo();

        // ✅ Controlled access via setters with validation
        record.setWeight(70.0);
        System.out.println("Updated BMI: " + record.getBMI());

        // ❌ Direct field access blocked:
        // record.weight = -50;  // Compile error! Field is private

        // ❌ This validation would never be bypassed:
        try {
            record.setAge(-5);
        } catch (IllegalArgumentException e) {
            System.out.println("Caught: " + e.getMessage());
        }

        // Confidential notes — password-protected
        record.addNote("Patient has mild hypertension", "WRONG");
        record.addNote("Patient has mild hypertension", "DOC#2024");
    }
}
```

---

## 7. Inheritance

**Inheritance** allows a class (**child/subclass**) to acquire the properties and behaviours of another class (**parent/superclass**).

### Real-Life Analogy

```
Vehicle (parent)
  ├── Car
  │     ├── ElectricCar
  │     └── SportsCar
  ├── Truck
  └── Motorcycle
```

A `Car` IS-A `Vehicle`. It inherits wheels, engine, fuel. But it also has its own specific features like air conditioning, trunk space.

### Types of Inheritance in Java

```
Single       Multilevel        Hierarchical      (Multiple via interfaces)
A            A                 A
│            │                 ├─ B
B            B                 └─ C
             │
             C
```

> ⚠️ Java does NOT support **multiple inheritance** with classes (to avoid the Diamond Problem) — but supports it via **interfaces**.

### Real-Life Example: Vehicle System

```java
// ── PARENT class ────────────────────────────────────────────
public class Vehicle {

    protected String brand;
    protected String model;
    protected int    year;
    protected double fuelLevel;     // 0.0 to 100.0
    protected double maxSpeed;

    public Vehicle(String brand, String model, int year, double maxSpeed) {
        this.brand    = brand;
        this.model    = model;
        this.year     = year;
        this.maxSpeed = maxSpeed;
        this.fuelLevel = 100.0;
        System.out.println("[Vehicle] Created: " + brand + " " + model);
    }

    public void start() {
        System.out.println(brand + " " + model + " engine started. Vroom!");
    }

    public void refuel(double amount) {
        fuelLevel = Math.min(100.0, fuelLevel + amount);
        System.out.println("Refuelled. Fuel level: " + fuelLevel + "%");
    }

    public void displayInfo() {
        System.out.println("Brand: " + brand + " | Model: " + model
            + " | Year: " + year + " | Max Speed: " + maxSpeed + " km/h");
    }
}

// ── CHILD class 1: Car ──────────────────────────────────────
public class Car extends Vehicle {

    private int   numberOfDoors;
    private boolean hasAirConditioning;

    public Car(String brand, String model, int year, double maxSpeed,
               int doors, boolean ac) {
        super(brand, model, year, maxSpeed);  // call parent constructor
        this.numberOfDoors     = doors;
        this.hasAirConditioning = ac;
        System.out.println("[Car] Configured doors and AC.");
    }

    public void openTrunk() {
        System.out.println(model + " trunk opened.");
    }

    @Override
    public void displayInfo() {
        super.displayInfo();  // reuse parent's display
        System.out.println("Doors: " + numberOfDoors + " | AC: " + hasAirConditioning);
    }
}

// ── CHILD class 2: ElectricCar extends Car (MULTILEVEL) ────
public class ElectricCar extends Car {

    private double batteryCapacity;  // in kWh
    private double chargeLevel;      // 0-100%
    private int    rangePerCharge;   // km

    public ElectricCar(String brand, String model, int year,
                       double maxSpeed, double batteryCapacity, int range) {
        super(brand, model, year, maxSpeed, 4, true); // all electric cars have 4 doors + AC
        this.batteryCapacity = batteryCapacity;
        this.chargeLevel     = 100.0;
        this.rangePerCharge  = range;
    }

    @Override
    public void start() {
        // Electric cars don't "vroom" — override parent behaviour
        System.out.println(brand + " " + model + " silently powered on. ⚡");
    }

    public void charge(double hours) {
        double charged = hours * 20; // 20% per hour
        chargeLevel = Math.min(100.0, chargeLevel + charged);
        System.out.printf("%s charged for %.1f hours. Battery: %.1f%%%n", model, hours, chargeLevel);
    }

    @Override
    public void displayInfo() {
        super.displayInfo();
        System.out.printf("Battery: %.1f kWh | Range: %d km | Charge: %.1f%%%n",
            batteryCapacity, rangePerCharge, chargeLevel);
    }
}

// ── CHILD class 3: Truck extends Vehicle ────────────────────
public class Truck extends Vehicle {

    private double payloadCapacity;  // in tonnes
    private int    numberOfAxles;

    public Truck(String brand, String model, int year, double maxSpeed,
                 double payload, int axles) {
        super(brand, model, year, maxSpeed);
        this.payloadCapacity = payload;
        this.numberOfAxles   = axles;
    }

    public void loadCargo(double tonnes) {
        if (tonnes <= payloadCapacity) {
            System.out.println("Loaded " + tonnes + " tonnes onto " + model);
        } else {
            System.out.println("Overload! Max capacity: " + payloadCapacity + " tonnes");
        }
    }
}

public class Main {
    public static void main(String[] args) {

        System.out.println("=== Creating Vehicles ===\n");
        Car car         = new Car("Toyota", "Camry", 2022, 180, 4, true);
        ElectricCar ev  = new ElectricCar("Tesla", "Model 3", 2023, 250, 82.0, 570);
        Truck truck     = new Truck("Tata", "Prima", 2021, 120, 25.0, 6);

        System.out.println("\n=== Starting Vehicles ===");
        car.start();
        ev.start();
        truck.start();

        System.out.println("\n=== Display Info ===");
        car.displayInfo();
        System.out.println();
        ev.displayInfo();

        System.out.println("\n=== Special Actions ===");
        car.openTrunk();
        ev.charge(2.5);
        truck.loadCargo(20.0);
        truck.loadCargo(30.0);
    }
}
```

---

## 8. super Keyword

`super` refers to the **parent class** and is used to:
1. Call the parent constructor
2. Call an overridden parent method
3. Access a hidden parent field

```java
public class Animal {
    String name;
    String sound;

    Animal(String name, String sound) {
        this.name  = name;
        this.sound = sound;
        System.out.println("Animal created: " + name);
    }

    void makeSound() {
        System.out.println(name + " says: " + sound);
    }

    void eat() {
        System.out.println(name + " is eating.");
    }
}

public class Dog extends Animal {
    String breed;

    Dog(String name, String breed) {
        super(name, "Woof");    // ── Use 1: Call parent constructor (MUST be first)
        this.breed = breed;
    }

    @Override
    void makeSound() {
        super.makeSound();      // ── Use 2: Call parent version first
        System.out.println(name + " also wags its tail!");
    }

    void fetch() {
        super.eat();            // ── Use 3: Call parent method from child
        System.out.println(name + " fetches the ball!");
    }
}

public class GoldenRetriever extends Dog {
    GoldenRetriever(String name) {
        super(name, "Golden Retriever");  // calls Dog constructor → which calls Animal
    }

    @Override
    void makeSound() {
        super.makeSound();  // Dog's makeSound (which calls Animal's makeSound)
        System.out.println(name + " is very friendly!");
    }
}

public class Main {
    public static void main(String[] args) {
        GoldenRetriever goldie = new GoldenRetriever("Buddy");
        goldie.makeSound();
        System.out.println();
        goldie.fetch();
    }
}
```

**Output:**
```
Animal created: Buddy
Buddy says: Woof
Buddy also wags its tail!
Buddy is very friendly!

Buddy is eating.
Buddy fetches the ball!
```

---

## 9. Method Overriding

**Method Overriding** = a child class provides its **own implementation** of a method that is already defined in the parent class.

### Rules
- Same method name, same parameters, same return type (or covariant)
- Use `@Override` annotation (not mandatory but strongly recommended)
- Access modifier cannot be more restrictive than parent's
- Cannot override `final`, `static`, or `private` methods

### Real-Life Example: Payment Processing System

```java
// Parent
public class PaymentProcessor {
    String processorName;

    PaymentProcessor(String name) {
        this.processorName = name;
    }

    // Will be overridden by each payment type
    boolean processPayment(double amount) {
        System.out.println("Processing ₹" + amount + " via generic processor.");
        return true;
    }

    void generateReceipt(double amount, boolean success) {
        System.out.println("──────────────────────────────");
        System.out.println("Processor : " + processorName);
        System.out.println("Amount    : ₹" + amount);
        System.out.println("Status    : " + (success ? "SUCCESS ✅" : "FAILED ❌"));
        System.out.println("──────────────────────────────");
    }
}

// UPI Payment
public class UPIPayment extends PaymentProcessor {
    private String upiId;

    UPIPayment(String upiId) {
        super("UPI");
        this.upiId = upiId;
    }

    @Override
    boolean processPayment(double amount) {
        System.out.println("Initiating UPI transfer for ₹" + amount);
        System.out.println("Sending request to UPI ID: " + upiId);
        // Simulate UPI processing
        boolean success = amount <= 100000; // UPI limit
        if (!success) System.out.println("UPI limit exceeded! Max ₹1,00,000");
        return success;
    }
}

// Credit Card Payment
public class CreditCardPayment extends PaymentProcessor {
    private String cardNumber;
    private double creditLimit;
    private double usedCredit;

    CreditCardPayment(String cardNumber, double creditLimit) {
        super("Credit Card");
        this.cardNumber  = cardNumber;
        this.creditLimit = creditLimit;
        this.usedCredit  = 0;
    }

    @Override
    boolean processPayment(double amount) {
        System.out.println("Authorising credit card: ****" + cardNumber.substring(12));
        if (usedCredit + amount > creditLimit) {
            System.out.println("Credit limit exceeded! Available: ₹" + (creditLimit - usedCredit));
            return false;
        }
        usedCredit += amount;
        System.out.printf("Credit card charged. Used: ₹%.2f / ₹%.2f%n", usedCredit, creditLimit);
        return true;
    }
}

// Crypto Payment
public class CryptoPayment extends PaymentProcessor {
    private String walletAddress;
    private double ethRate = 250000; // 1 ETH in INR

    CryptoPayment(String walletAddress) {
        super("Crypto (ETH)");
        this.walletAddress = walletAddress;
    }

    @Override
    boolean processPayment(double amount) {
        double ethAmount = amount / ethRate;
        System.out.printf("Converting ₹%.2f → %.6f ETH%n", amount, ethAmount);
        System.out.println("Broadcasting to blockchain: " + walletAddress.substring(0, 10) + "...");
        System.out.println("Waiting for block confirmation...");
        return true;
    }
}

public class Main {
    public static void main(String[] args) {
        PaymentProcessor[] processors = {
            new UPIPayment("alice@okaxis"),
            new CreditCardPayment("1234567890123456", 50000),
            new CryptoPayment("0xAbC123dEf456789")
        };

        double amount = 15000;

        for (PaymentProcessor processor : processors) {
            System.out.println("\n>>> Paying ₹" + amount + " via " + processor.processorName);
            boolean result = processor.processPayment(amount);  // RUNTIME POLYMORPHISM
            processor.generateReceipt(amount, result);
        }
    }
}
```

---

## 10. Polymorphism

**Polymorphism** means "many forms." One interface, multiple implementations.

### 10.1 Compile-time Polymorphism — Method Overloading

**Same method name**, different parameters. Resolved at **compile time**.

### Real-Life Analogy: Calculator / Print function
- `print(int)` → prints integer
- `print(String)` → prints text
- `print(int, int)` → prints two integers

```java
public class SmartPrinter {

    // Overloaded print methods — same name, different parameters
    void print(String message) {
        System.out.println("[TEXT] " + message);
    }

    void print(int number) {
        System.out.println("[INT ] " + number);
    }

    void print(double number) {
        System.out.printf("[DBL ] %.2f%n", number);
    }

    void print(String label, int value) {
        System.out.println("[TAG ] " + label + " = " + value);
    }

    void print(String[] items) {
        System.out.print("[ARR ] ");
        for (String item : items) System.out.print(item + "  ");
        System.out.println();
    }

    // ── Method overloading in a real Math utility ────────
    static int add(int a, int b)             { return a + b; }
    static double add(double a, double b)    { return a + b; }
    static int add(int a, int b, int c)      { return a + b + c; }
    static String add(String a, String b)    { return a + b; }
}

public class Main {
    public static void main(String[] args) {
        SmartPrinter printer = new SmartPrinter();

        printer.print("Hello World");
        printer.print(42);
        printer.print(3.14159);
        printer.print("Score", 99);
        printer.print(new String[]{"Java", "Python", "Go"});

        System.out.println(SmartPrinter.add(5, 3));
        System.out.println(SmartPrinter.add(5.5, 3.3));
        System.out.println(SmartPrinter.add(1, 2, 3));
        System.out.println(SmartPrinter.add("Hello", " World"));
    }
}
```

---

### 10.2 Runtime Polymorphism — Method Overriding

The actual method called is decided at **runtime** based on the object's actual type, not the reference type.

### Real-Life Analogy: Shape Drawing Application

```java
public class Shape {
    String colour;

    Shape(String colour) { this.colour = colour; }

    // These will be overridden
    double area()      { return 0; }
    double perimeter() { return 0; }

    void draw() {
        System.out.println("Drawing a " + colour + " " + getClass().getSimpleName()
            + " | Area: " + String.format("%.2f", area())
            + " | Perimeter: " + String.format("%.2f", perimeter()));
    }
}

public class Circle extends Shape {
    double radius;

    Circle(String colour, double radius) {
        super(colour);
        this.radius = radius;
    }

    @Override double area()      { return Math.PI * radius * radius; }
    @Override double perimeter() { return 2 * Math.PI * radius; }
}

public class Rectangle extends Shape {
    double width, height;

    Rectangle(String colour, double width, double height) {
        super(colour);
        this.width  = width;
        this.height = height;
    }

    @Override double area()      { return width * height; }
    @Override double perimeter() { return 2 * (width + height); }
}

public class Triangle extends Shape {
    double a, b, c;

    Triangle(String colour, double a, double b, double c) {
        super(colour);
        this.a = a; this.b = b; this.c = c;
    }

    @Override double area() {
        double s = (a + b + c) / 2;
        return Math.sqrt(s * (s-a) * (s-b) * (s-c)); // Heron's formula
    }
    @Override double perimeter() { return a + b + c; }
}

public class Main {
    public static void main(String[] args) {
        // Parent reference, child objects — RUNTIME POLYMORPHISM
        Shape[] shapes = {
            new Circle("Red", 7),
            new Rectangle("Blue", 8, 5),
            new Triangle("Green", 3, 4, 5),
            new Circle("Yellow", 3.5)
        };

        double totalArea = 0;
        for (Shape shape : shapes) {
            shape.draw();             // JVM decides at RUNTIME which draw() to call
            totalArea += shape.area();
        }
        System.out.printf("%nTotal area of all shapes: %.2f%n", totalArea);
    }
}
```

**Output:**
```
Drawing a Red Circle       | Area: 153.94 | Perimeter: 43.98
Drawing a Blue Rectangle   | Area: 40.00  | Perimeter: 26.00
Drawing a Green Triangle   | Area: 6.00   | Perimeter: 12.00
Drawing a Yellow Circle    | Area: 38.48  | Perimeter: 21.99

Total area of all shapes: 238.42
```

---

## 11. Abstraction

**Abstraction** = hiding **implementation details** and showing only the **essential features** to the user.

### Real-Life Analogy

When you press the TV remote's power button:
- You don't know HOW the infrared signal is generated
- You don't know how the TV decodes it
- You just know: press button → TV turns on

That's abstraction — hiding complexity, exposing simplicity.

---

### 11.1 Abstract Class

- Cannot be instantiated (can't do `new AbstractClass()`)
- Can have `abstract` methods (no body — subclass MUST implement)
- Can have regular methods with implementation
- Can have constructors, fields, static methods

### Real-Life Example: Food Ordering System

```java
// ABSTRACT class — blueprint that can't be used directly
public abstract class MenuItem {

    protected String name;
    protected double basePrice;
    protected String category;

    // Constructor (abstract classes CAN have constructors)
    public MenuItem(String name, double basePrice, String category) {
        this.name      = name;
        this.basePrice = basePrice;
        this.category  = category;
    }

    // ── Abstract methods — MUST be implemented by subclasses ──
    public abstract double calculatePrice(); // pricing varies by item
    public abstract String getDescription(); // description varies

    // ── Concrete method — shared by all subclasses ────────────
    public void displayItem() {
        System.out.printf("%-20s | %-12s | ₹%-8.2f | %s%n",
            name, category, calculatePrice(), getDescription());
    }

    public String getName() { return name; }
}

// CONCRETE class 1: Pizza
public class Pizza extends MenuItem {
    private String size;       // Small, Medium, Large
    private boolean extraCheese;
    private int     toppings;

    public Pizza(String name, double basePrice, String size, boolean extraCheese, int toppings) {
        super(name, basePrice, "Pizza");
        this.size        = size;
        this.extraCheese = extraCheese;
        this.toppings    = toppings;
    }

    @Override
    public double calculatePrice() {
        double price = basePrice;
        if (size.equals("Medium")) price *= 1.3;
        if (size.equals("Large"))  price *= 1.6;
        if (extraCheese)           price += 50;
        price += toppings * 30;
        return price;
    }

    @Override
    public String getDescription() {
        return size + " | Cheese: " + (extraCheese ? "Extra" : "Normal")
            + " | Toppings: " + toppings;
    }
}

// CONCRETE class 2: Burger
public class Burger extends MenuItem {
    private String  pattyType;  // Veg, Chicken, Beef
    private boolean doublePatch;

    public Burger(String name, double basePrice, String pattyType, boolean doublePatch) {
        super(name, basePrice, "Burger");
        this.pattyType   = pattyType;
        this.doublePatch = doublePatch;
    }

    @Override
    public double calculatePrice() {
        double price = basePrice;
        if (pattyType.equals("Chicken")) price += 40;
        if (pattyType.equals("Beef"))    price += 80;
        if (doublePatch)                 price += 60;
        return price;
    }

    @Override
    public String getDescription() {
        return pattyType + " Patty | " + (doublePatch ? "Double" : "Single");
    }
}

// CONCRETE class 3: Drink
public class Drink extends MenuItem {
    private String size;    // Regular, Large
    private boolean isCold;

    public Drink(String name, double basePrice, String size, boolean cold) {
        super(name, basePrice, "Drink");
        this.size   = size;
        this.isCold = cold;
    }

    @Override
    public double calculatePrice() {
        return size.equals("Large") ? basePrice * 1.4 : basePrice;
    }

    @Override
    public String getDescription() {
        return size + " | " + (isCold ? "Cold 🧊" : "Hot ♨");
    }
}

public class Main {
    public static void main(String[] args) {

        // MenuItem item = new MenuItem("X", 100, "Y"); // ❌ Cannot instantiate abstract class

        MenuItem[] order = {
            new Pizza("Margherita",   199, "Large",  true,  2),
            new Pizza("Pepperoni",    249, "Medium", false, 3),
            new Burger("Zinger",      149, "Chicken", false),
            new Burger("Double Beef", 249, "Beef",    true),
            new Drink("Coke",          60, "Large",  true),
            new Drink("Green Tea",     80, "Regular", false)
        };

        System.out.println("─".repeat(80));
        System.out.printf("%-20s | %-12s | %-9s | %s%n", "Item", "Category", "Price", "Details");
        System.out.println("─".repeat(80));

        double total = 0;
        for (MenuItem item : order) {
            item.displayItem();
            total += item.calculatePrice();
        }
        System.out.println("─".repeat(80));
        System.out.printf("%-20s   %-12s   ₹%-8.2f%n", "TOTAL", "", total);
    }
}
```

---

### 11.2 Interface

An **interface** is a **100% abstract contract**. It defines WHAT a class must do, not HOW.

Key points:
- All methods are `public abstract` by default (before Java 8)
- All fields are `public static final` by default
- A class can **implement multiple interfaces**
- Java 8+: can have `default` and `static` methods
- Java 9+: can have `private` methods

### Real-Life Example: Smart Home System

```java
// ── Interfaces (contracts) ─────────────────────────────────

public interface Switchable {
    void turnOn();
    void turnOff();
    boolean isOn();

    // Default method (Java 8+) — optional override
    default void toggle() {
        if (isOn()) turnOff();
        else turnOn();
    }
}

public interface Dimmable {
    void setBrightness(int level); // 0-100
    int  getBrightness();
}

public interface Schedulable {
    void scheduleOn(String time);
    void scheduleOff(String time);
}

public interface EnergyMonitor {
    double getPowerConsumption(); // in watts
    static String getEnergyRating(double watts) { // static interface method
        if (watts < 10)  return "A+++ (Ultra Efficient)";
        if (watts < 20)  return "A++ (Very Efficient)";
        if (watts < 50)  return "A+ (Efficient)";
        return "B (Standard)";
    }
}

// ── Smart Light: implements multiple interfaces ─────────────
public class SmartLight implements Switchable, Dimmable, Schedulable, EnergyMonitor {

    private String  location;
    private boolean on = false;
    private int     brightness = 100;

    public SmartLight(String location) { this.location = location; }

    @Override public void turnOn()  { on = true;  System.out.println(location + " light ON 💡"); }
    @Override public void turnOff() { on = false; System.out.println(location + " light OFF"); }
    @Override public boolean isOn() { return on; }

    @Override
    public void setBrightness(int level) {
        brightness = Math.max(0, Math.min(100, level));
        System.out.println(location + " brightness: " + brightness + "%");
    }
    @Override public int getBrightness() { return brightness; }

    @Override
    public void scheduleOn(String time)  { System.out.println(location + " light scheduled ON at "  + time); }
    @Override
    public void scheduleOff(String time) { System.out.println(location + " light scheduled OFF at " + time); }

    @Override
    public double getPowerConsumption() {
        return on ? (brightness / 100.0) * 9 : 0; // 9W LED max
    }
}

// ── Smart AC: implements some interfaces ───────────────────
public class SmartAC implements Switchable, Schedulable, EnergyMonitor {

    private String  location;
    private boolean on = false;
    private int     temperature = 24; // degrees Celsius

    public SmartAC(String location) { this.location = location; }

    @Override public void turnOn()  { on = true;  System.out.println(location + " AC ON ❄️  Temp: " + temperature + "°C"); }
    @Override public void turnOff() { on = false; System.out.println(location + " AC OFF"); }
    @Override public boolean isOn() { return on; }

    public void setTemperature(int temp) {
        temperature = Math.max(16, Math.min(30, temp));
        System.out.println(location + " AC temperature set to " + temperature + "°C");
    }

    @Override
    public void scheduleOn(String time)  { System.out.println(location + " AC scheduled ON at "  + time); }
    @Override
    public void scheduleOff(String time) { System.out.println(location + " AC scheduled OFF at " + time); }

    @Override
    public double getPowerConsumption()  { return on ? 1500 : 0; }
}

public class Main {
    public static void main(String[] args) {

        SmartLight bedroomLight = new SmartLight("Bedroom");
        SmartLight kitchenLight = new SmartLight("Kitchen");
        SmartAC    livingRoomAC = new SmartAC("Living Room");

        // ── Using Switchable interface reference ───────────
        Switchable[] devices = { bedroomLight, kitchenLight, livingRoomAC };
        System.out.println("=== All On ===");
        for (Switchable d : devices) d.turnOn();

        // ── Dimmable specific methods ──────────────────────
        System.out.println("\n=== Dimming ===");
        bedroomLight.setBrightness(40);
        kitchenLight.setBrightness(80);

        // ── Toggle (default interface method) ─────────────
        System.out.println("\n=== Toggle Bedroom ===");
        bedroomLight.toggle(); // turns OFF
        bedroomLight.toggle(); // turns ON

        // ── Energy monitoring ──────────────────────────────
        System.out.println("\n=== Energy Report ===");
        EnergyMonitor[] monitors = { bedroomLight, kitchenLight, livingRoomAC };
        double total = 0;
        for (EnergyMonitor m : monitors) {
            double w = m.getPowerConsumption();
            System.out.printf("  %.1fW — %s%n", w, EnergyMonitor.getEnergyRating(w));
            total += w;
        }
        System.out.printf("Total: %.1fW%n", total);

        // ── Scheduling ─────────────────────────────────────
        System.out.println("\n=== Scheduling ===");
        livingRoomAC.setTemperature(22);
        livingRoomAC.scheduleOn("06:30");
        livingRoomAC.scheduleOff("09:00");
    }
}
```

---

### 11.3 Abstract Class vs Interface

| Feature                    | Abstract Class                   | Interface                              |
|----------------------------|----------------------------------|----------------------------------------|
| Instantiation              | ❌ Cannot                        | ❌ Cannot                              |
| Method types               | Abstract + Concrete              | Abstract + default + static + private  |
| Fields                     | Any type                         | `public static final` only             |
| Constructor                | ✅ Yes                           | ❌ No                                  |
| Multiple inheritance       | ❌ One class only                | ✅ Multiple interfaces                 |
| Access modifiers on methods| Any                              | `public` by default                    |
| State (instance variables) | ✅ Yes                           | ❌ No                                  |
| When to use                | IS-A with shared state/behaviour | CAN-DO contract across unrelated classes|

```java
// USE Abstract Class when: classes share code AND state
abstract class Animal { String name; void breathe() { ... } } // shared behaviour + state

// USE Interface when: unrelated classes share a CAPABILITY
interface Flyable { void fly(); }
class Bird    implements Flyable { ... } // birds can fly
class Airplane implements Flyable { ... } // planes can fly — but Airplane IS NOT an Animal
class Superman implements Flyable { ... } // Superman can fly — but is not an Animal either
```

---

## 12. final Keyword

`final` means "cannot be changed" — applied to variables, methods, and classes.

```java
// ── final VARIABLE — value cannot be reassigned ─────────────
public class Constants {
    static final double PI          = 3.14159265;
    static final int    MAX_RETRIES = 3;
    static final String APP_NAME    = "MyApp";

    // PI = 3.14;  // ❌ Compile error: cannot assign to final variable
}

// ── final METHOD — cannot be overridden in subclasses ────────
public class BankSecurity {
    final void authenticate(String pin) {
        System.out.println("Authenticating with PIN (cannot be bypassed!)");
    }
}

public class HackedBank extends BankSecurity {
    // @Override
    // void authenticate(String pin) { ... } // ❌ Compile error!
}

// ── final CLASS — cannot be subclassed (extended) ────────────
final class ImmutablePoint {
    final double x;
    final double y;

    ImmutablePoint(double x, double y) {
        this.x = x;
        this.y = y;
    }

    double distanceTo(ImmutablePoint other) {
        return Math.sqrt(Math.pow(x - other.x, 2) + Math.pow(y - other.y, 2));
    }
}

// class SubPoint extends ImmutablePoint { } // ❌ Compile error: cannot inherit from final

// ── final in practical context: blank final field ─────────────
public class Order {
    final String orderId;    // blank final — must be set in constructor
    String status;

    Order(String orderId) {
        this.orderId = orderId;   // ✅ Set here
        this.status  = "PENDING";
    }

    void updateStatus(String newStatus) {
        this.status  = newStatus;    // ✅ status can change
        // this.orderId = "NEW_ID"; // ❌ orderId cannot change
    }
}
```

---

## 13. instanceof Operator

`instanceof` checks if an object is an instance of a particular class or interface.

```java
public class InstanceofDemo {

    public static void main(String[] args) {
        // Creating various shapes
        Shape[] shapes = {
            new Circle("Red", 5),
            new Rectangle("Blue", 4, 6),
            new Circle("Green", 3)
        };

        for (Shape shape : shapes) {

            // ── Traditional instanceof ─────────────────────
            if (shape instanceof Circle) {
                Circle c = (Circle) shape;   // explicit cast needed
                System.out.println("Circle radius: " + c.radius);
            } else if (shape instanceof Rectangle) {
                Rectangle r = (Rectangle) shape;
                System.out.printf("Rectangle: %.0f x %.0f%n", r.width, r.height);
            }

            // ── Pattern Matching instanceof (Java 16+) ─────
            // Combines instanceof check + cast in ONE step
            if (shape instanceof Circle c) {   // 'c' is now a Circle — no cast needed!
                System.out.println("Pattern match — area: " + String.format("%.2f", c.area()));
            }

            System.out.println("Is Shape? " + (shape instanceof Shape));       // always true
            System.out.println("Is Object? " + (shape instanceof Object));     // always true
        }

        // null is never instanceof anything
        Shape nullShape = null;
        System.out.println("null instanceof Shape: " + (nullShape instanceof Shape)); // false
    }
}
```

---

## 14. Object Class Methods

Every Java class implicitly extends `java.lang.Object`. Key methods to override:

```java
import java.util.Objects;

public class Product {
    private String  productId;
    private String  name;
    private double  price;
    private String  category;

    public Product(String productId, String name, double price, String category) {
        this.productId = productId;
        this.name      = name;
        this.price     = price;
        this.category  = category;
    }

    // ── toString() — human-readable representation ──────────
    // Default: "Product@7852e922" (class name + hash) — not useful!
    @Override
    public String toString() {
        return String.format("Product{id='%s', name='%s', price=₹%.2f, category='%s'}",
            productId, name, price, category);
    }

    // ── equals() — logical equality ─────────────────────────
    // Default: reference equality (==) — same memory address
    // Override to compare by content
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;                 // same reference
        if (obj == null) return false;                // null check
        if (!(obj instanceof Product)) return false;  // type check
        Product other = (Product) obj;
        return Objects.equals(productId, other.productId); // compare by ID
    }

    // ── hashCode() — MUST override if equals() is overridden ─
    // Contract: if a.equals(b) → a.hashCode() == b.hashCode()
    // Used by HashMap, HashSet, etc.
    @Override
    public int hashCode() {
        return Objects.hash(productId);
    }

    // ── clone() — copy of the object ───────────────────────
    // Requires implementing Cloneable interface
}

public class Main {
    public static void main(String[] args) {
        Product p1 = new Product("P001", "iPhone 15", 89999, "Electronics");
        Product p2 = new Product("P001", "iPhone 15", 89999, "Electronics");
        Product p3 = new Product("P002", "Samsung S24", 79999, "Electronics");

        // toString()
        System.out.println(p1); // calls toString() automatically

        // equals()
        System.out.println("p1 == p2  : " + (p1 == p2));      // false (different refs)
        System.out.println("p1.equals(p2): " + p1.equals(p2)); // true (same productId)
        System.out.println("p1.equals(p3): " + p1.equals(p3)); // false

        // hashCode()
        System.out.println("p1 hash: " + p1.hashCode());
        System.out.println("p2 hash: " + p2.hashCode()); // same as p1 (equal objects)

        // Works correctly in collections because of equals/hashCode
        java.util.Set<Product> set = new java.util.HashSet<>();
        set.add(p1);
        set.add(p2); // duplicate — not added because equals() returns true
        set.add(p3);
        System.out.println("Set size: " + set.size()); // 2 (not 3)
    }
}
```

---

## 15. Inner Classes

A class defined **inside another class**.

```java
public class OuterCompany {

    private String companyName = "TechCorp";
    private int    revenue     = 1000000;

    // ── 1. Regular Inner Class ─────────────────────────────
    // Has access to all outer class members (even private)
    class Department {
        String deptName;

        Department(String name) { this.deptName = name; }

        void display() {
            // Can access outer class's private field directly
            System.out.println(deptName + " @ " + companyName + " | Revenue: " + revenue);
        }
    }

    // ── 2. Static Nested Class ─────────────────────────────
    // Does NOT have access to outer instance members
    static class CompanyPolicy {
        void display() {
            System.out.println("Policy: 9-to-5 work hours, 20 leaves/year");
            // System.out.println(companyName); // ❌ cannot access instance member
        }
    }

    // ── 3. Local Inner Class (inside a method) ────────────
    void performAudit() {
        final int AUDIT_YEAR = 2024;

        class Auditor {
            void audit() {
                System.out.println("Auditing " + companyName + " for year " + AUDIT_YEAR);
            }
        }
        new Auditor().audit();
    }

    // ── 4. Anonymous Inner Class ──────────────────────────
    // Created on-the-fly for one-time use, usually for interfaces
    void runEvent() {
        Runnable ceremony = new Runnable() {
            @Override
            public void run() {
                System.out.println("Annual ceremony of " + companyName + " has started!");
            }
        };
        ceremony.run();

        // Lambda equivalent (Java 8+)
        Runnable ceremony2 = () -> System.out.println("Ceremony 2 started at " + companyName);
        ceremony2.run();
    }
}

public class Main {
    public static void main(String[] args) {
        OuterCompany company = new OuterCompany();

        // Regular inner class — needs outer instance
        OuterCompany.Department eng = company.new Department("Engineering");
        eng.display();

        // Static nested — no outer instance needed
        OuterCompany.CompanyPolicy policy = new OuterCompany.CompanyPolicy();
        policy.display();

        company.performAudit();
        company.runEvent();
    }
}
```

---

## 16. Enums

An **enum** (enumeration) is a special class representing a fixed set of constants.

### Real-Life Example: Order Management System

```java
public enum OrderStatus {
    PENDING   ("Order placed, awaiting confirmation",   "🟡"),
    CONFIRMED ("Order confirmed by seller",             "🔵"),
    PACKED    ("Order packed and ready to ship",        "🟠"),
    SHIPPED   ("Order dispatched",                      "🚚"),
    DELIVERED ("Order delivered successfully",          "🟢"),
    CANCELLED ("Order cancelled",                       "🔴"),
    REFUNDED  ("Refund processed",                      "💰");

    private final String description;
    private final String icon;

    // Enum constructor (always private)
    OrderStatus(String description, String icon) {
        this.description = description;
        this.icon        = icon;
    }

    public String getDescription() { return description; }
    public String getIcon()        { return icon; }

    // Enum method
    public boolean isActive() {
        return this != CANCELLED && this != REFUNDED;
    }

    // Enum can have abstract methods too
    public boolean canTransitionTo(OrderStatus next) {
        return switch (this) {
            case PENDING   -> next == CONFIRMED || next == CANCELLED;
            case CONFIRMED -> next == PACKED    || next == CANCELLED;
            case PACKED    -> next == SHIPPED;
            case SHIPPED   -> next == DELIVERED;
            case DELIVERED -> next == REFUNDED;
            default        -> false;
        };
    }
}

public class Order {
    private String      orderId;
    private String      product;
    private OrderStatus status;

    public Order(String orderId, String product) {
        this.orderId = orderId;
        this.product = product;
        this.status  = OrderStatus.PENDING;
    }

    public void updateStatus(OrderStatus newStatus) {
        if (status.canTransitionTo(newStatus)) {
            System.out.printf("[%s] %s → %s %s%n",
                orderId, status, newStatus, newStatus.getIcon());
            status = newStatus;
        } else {
            System.out.printf("[%s] ❌ Cannot change from %s to %s%n",
                orderId, status, newStatus);
        }
    }

    public void printStatus() {
        System.out.printf("[%s] %s | Status: %s %s | Active: %b%n",
            orderId, product, status, status.getIcon(), status.isActive());
        System.out.println("        → " + status.getDescription());
    }
}

public class Main {
    public static void main(String[] args) {
        Order order = new Order("ORD-001", "iPhone 15 Pro");
        order.printStatus();

        order.updateStatus(OrderStatus.CONFIRMED);
        order.updateStatus(OrderStatus.SHIPPED); // ❌ skipping PACKED
        order.updateStatus(OrderStatus.PACKED);
        order.updateStatus(OrderStatus.SHIPPED);
        order.updateStatus(OrderStatus.DELIVERED);
        order.printStatus();

        // Enum utilities
        System.out.println("\n=== All Statuses ===");
        for (OrderStatus s : OrderStatus.values()) {
            System.out.printf("  %-12s %s %s%n", s.name(), s.getIcon(), s.getDescription());
        }

        // Enum from string
        OrderStatus fromString = OrderStatus.valueOf("PENDING");
        System.out.println("\nOrdinal of SHIPPED: " + OrderStatus.SHIPPED.ordinal()); // 4
    }
}
```

---

## 17. Generics

**Generics** enable classes and methods to work with **any data type** while maintaining **type safety** at compile time.

### Real-Life Analogy
A **storage box** — you can have `Box<Book>`, `Box<Toy>`, `Box<Electronics>`. The box itself doesn't care what's inside — but once you specify, you get type-safe access.

```java
// ── Generic Class ──────────────────────────────────────────
public class Warehouse<T> {
    private T[] items;
    private int count = 0;
    private String warehouseName;

    @SuppressWarnings("unchecked")
    public Warehouse(String name, int capacity) {
        this.warehouseName = name;
        this.items = (T[]) new Object[capacity];
    }

    public void store(T item) {
        if (count < items.length) {
            items[count++] = item;
            System.out.println("Stored in " + warehouseName + ": " + item);
        } else {
            System.out.println("Warehouse full!");
        }
    }

    public T retrieve(int index) {
        if (index < 0 || index >= count) throw new IndexOutOfBoundsException("No item at " + index);
        return items[index];
    }

    public int getCount() { return count; }
}

// ── Generic Method ─────────────────────────────────────────
public class SortUtils {

    // Works for any Comparable type (Integer, String, Double...)
    public static <T extends Comparable<T>> T findMax(T[] array) {
        if (array == null || array.length == 0) throw new IllegalArgumentException("Empty array");
        T max = array[0];
        for (T item : array) {
            if (item.compareTo(max) > 0) max = item;
        }
        return max;
    }

    public static <T extends Comparable<T>> void bubbleSort(T[] array) {
        for (int i = 0; i < array.length - 1; i++) {
            for (int j = 0; j < array.length - 1 - i; j++) {
                if (array[j].compareTo(array[j + 1]) > 0) {
                    T temp      = array[j];
                    array[j]    = array[j + 1];
                    array[j + 1] = temp;
                }
            }
        }
    }
}

// ── Bounded Wildcards ──────────────────────────────────────
class Statistics {
    // <? extends Number> — accepts List<Integer>, List<Double>, List<Float>
    public static double sum(java.util.List<? extends Number> list) {
        double total = 0;
        for (Number n : list) total += n.doubleValue();
        return total;
    }
}

public class Main {
    public static void main(String[] args) {
        // Generic class with different types
        Warehouse<String>  bookStore  = new Warehouse<>("Book Warehouse", 5);
        Warehouse<Integer> itemCodes  = new Warehouse<>("Code Warehouse", 5);

        bookStore.store("Clean Code");
        bookStore.store("Design Patterns");
        bookStore.store("Effective Java");

        itemCodes.store(1001);
        itemCodes.store(1002);

        System.out.println("Retrieved: " + bookStore.retrieve(0));
        System.out.println("Total books: " + bookStore.getCount());

        // Generic method
        Integer[] nums    = {5, 2, 9, 1, 7};
        String[]  words   = {"banana", "apple", "cherry"};

        System.out.println("Max int: "    + SortUtils.findMax(nums));
        System.out.println("Max string: " + SortUtils.findMax(words));

        SortUtils.bubbleSort(nums);
        System.out.print("Sorted: ");
        for (int n : nums) System.out.print(n + " ");
        System.out.println();

        // Wildcards
        java.util.List<Integer> ints    = java.util.Arrays.asList(1, 2, 3, 4, 5);
        java.util.List<Double>  doubles = java.util.Arrays.asList(1.1, 2.2, 3.3);

        System.out.println("Sum of ints: "    + Statistics.sum(ints));
        System.out.println("Sum of doubles: " + Statistics.sum(doubles));
    }
}
```

---

## 18. SOLID Principles

**SOLID** is a set of 5 design principles for writing maintainable, scalable OOP code.

### S — Single Responsibility Principle (SRP)

> A class should have **one and only one reason to change**.

```java
// ❌ BAD — UserService does too many things
class BadUserService {
    void createUser(String name) { System.out.println("Creating user: " + name); }
    void sendEmail(String email) { System.out.println("Sending email to: " + email); } // separate concern!
    void generateReport()        { System.out.println("Generating user report"); }      // separate concern!
}

// ✅ GOOD — Each class has one responsibility
class UserService {
    private EmailService   emailService  = new EmailService();
    private ReportService  reportService = new ReportService();

    void createUser(String name) {
        System.out.println("Creating user: " + name);
        emailService.sendWelcomeEmail(name + "@example.com");
    }
}

class EmailService {
    void sendWelcomeEmail(String email) { System.out.println("Welcome email → " + email); }
}

class ReportService {
    void generateUserReport() { System.out.println("Generating user report"); }
}
```

---

### O — Open/Closed Principle (OCP)

> Classes should be **open for extension, closed for modification**.

```java
// ❌ BAD — must modify existing class to add discount types
class BadDiscount {
    double apply(String type, double price) {
        if (type.equals("SEASONAL")) return price * 0.9;
        if (type.equals("STUDENT"))  return price * 0.85;
        // To add new type, you MODIFY this class ❌
        return price;
    }
}

// ✅ GOOD — add new discounts without modifying existing code
interface DiscountStrategy {
    double apply(double price);
    String getName();
}

class SeasonalDiscount implements DiscountStrategy {
    public double apply(double price) { return price * 0.90; }
    public String getName() { return "Seasonal (10% off)"; }
}

class StudentDiscount implements DiscountStrategy {
    public double apply(double price) { return price * 0.85; }
    public String getName() { return "Student (15% off)"; }
}

class FlashSaleDiscount implements DiscountStrategy { // NEW — no existing code changed!
    public double apply(double price) { return price * 0.50; }
    public String getName() { return "Flash Sale (50% off)"; }
}

class PriceCalculator {
    double calculate(double price, DiscountStrategy discount) {
        double finalPrice = discount.apply(price);
        System.out.printf("%-22s ₹%.2f → ₹%.2f%n", discount.getName(), price, finalPrice);
        return finalPrice;
    }
}
```

---

### L — Liskov Substitution Principle (LSP)

> Objects of a subclass should be **substitutable** for objects of the superclass without breaking the program.

```java
// ❌ BAD — Square breaks Rectangle contract
class BadRectangle {
    protected int width, height;
    void setWidth(int w)  { width  = w; }
    void setHeight(int h) { height = h; }
    int area() { return width * height; }
}

class BadSquare extends BadRectangle {
    @Override void setWidth(int w)  { width = height = w; } // breaks Rectangle behaviour!
    @Override void setHeight(int h) { width = height = h; }
}
// setWidth(5); setHeight(10); area() → should be 50 but Square returns 100 ❌

// ✅ GOOD — model correctly
abstract class Shape2 { abstract int area(); }

class Rectangle2 extends Shape2 {
    int width, height;
    Rectangle2(int w, int h) { width = w; height = h; }
    public int area() { return width * height; }
}

class Square2 extends Shape2 {
    int side;
    Square2(int s) { side = s; }
    public int area() { return side * side; }
}
// Both work as Shape2 correctly ✅
```

---

### I — Interface Segregation Principle (ISP)

> A class should not be forced to implement methods it **does not use**.

```java
// ❌ BAD — one giant interface
interface AllInOnePrinter {
    void print();
    void scan();
    void fax();       // old printers can't fax!
    void photocopy();
}

// ✅ GOOD — segregated interfaces
interface Printable   { void print(); }
interface Scannable   { void scan(); }
interface Faxable     { void fax(); }
interface Copyable    { void photocopy(); }

class BasicPrinter implements Printable {
    public void print() { System.out.println("Printing..."); }
    // Not forced to implement scan/fax/copy ✅
}

class OfficePrinter implements Printable, Scannable, Copyable {
    public void print()     { System.out.println("Printing..."); }
    public void scan()      { System.out.println("Scanning..."); }
    public void photocopy() { System.out.println("Copying..."); }
}

class AllInOneEnterprisePrinter implements Printable, Scannable, Faxable, Copyable {
    public void print()     { System.out.println("Printing..."); }
    public void scan()      { System.out.println("Scanning..."); }
    public void fax()       { System.out.println("Faxing..."); }
    public void photocopy() { System.out.println("Copying..."); }
}
```

---

### D — Dependency Inversion Principle (DIP)

> High-level modules should not depend on low-level modules. Both should depend on **abstractions**.

```java
// ❌ BAD — high-level class depends directly on low-level class
class MySQLDatabase {
    void save(String data) { System.out.println("Saving to MySQL: " + data); }
}

class BadUserRepository {
    private MySQLDatabase db = new MySQLDatabase(); // tightly coupled!
    void saveUser(String user) { db.save(user); }
}

// ✅ GOOD — depend on abstraction (interface)
interface Database {
    void save(String data);
    String load(String key);
}

class MySQLDatabase2 implements Database {
    public void save(String data) { System.out.println("MySQL: saved → " + data); }
    public String load(String key) { return "MySQL: loaded " + key; }
}

class MongoDatabase implements Database {
    public void save(String data) { System.out.println("MongoDB: saved → " + data); }
    public String load(String key) { return "MongoDB: loaded " + key; }
}

class UserRepository {
    private Database db; // depends on ABSTRACTION, not concrete class

    UserRepository(Database db) { this.db = db; } // injected from outside

    void saveUser(String user) { db.save(user); }
    String loadUser(String id) { return db.load(id); }
}

public class Main {
    public static void main(String[] args) {
        // Swap database without changing UserRepository ✅
        UserRepository mysqlRepo = new UserRepository(new MySQLDatabase2());
        UserRepository mongoRepo = new UserRepository(new MongoDatabase());

        mysqlRepo.saveUser("Alice");
        mongoRepo.saveUser("Bob");
    }
}
```

---

## 19. Design Patterns (OOP in Action)

### Singleton Pattern — Only one instance ever

```java
// Real-life: Application configuration, Logger, DB Connection Pool
public class AppConfig {
    private static AppConfig instance; // the single instance

    private String dbUrl      = "jdbc:mysql://localhost/mydb";
    private int    maxThreads = 10;
    private String appVersion = "2.4.1";

    private AppConfig() { // private constructor — nobody can do 'new AppConfig()'
        System.out.println("Config loaded from file (only happens once).");
    }

    // Thread-safe singleton
    public static synchronized AppConfig getInstance() {
        if (instance == null) {
            instance = new AppConfig();
        }
        return instance;
    }

    public String getDbUrl()      { return dbUrl; }
    public int    getMaxThreads() { return maxThreads; }
    public String getVersion()    { return appVersion; }
}

public class Main {
    public static void main(String[] args) {
        AppConfig config1 = AppConfig.getInstance();
        AppConfig config2 = AppConfig.getInstance();
        AppConfig config3 = AppConfig.getInstance();

        System.out.println("Same instance? " + (config1 == config2)); // true
        System.out.println("Version: " + config1.getVersion());
        System.out.println("Max Threads: " + config3.getMaxThreads());
    }
}
```

---

### Builder Pattern — Construct complex objects step by step

```java
// Real-life: Building a complex Pizza order, HTTP Request, SQL query
public class Pizza {
    private final String  size;           // required
    private final String  crust;          // required
    private final boolean extraCheese;    // optional
    private final boolean extraSauce;
    private final java.util.List<String> toppings;
    private final boolean glutenFree;

    private Pizza(Builder builder) {
        this.size        = builder.size;
        this.crust       = builder.crust;
        this.extraCheese = builder.extraCheese;
        this.extraSauce  = builder.extraSauce;
        this.toppings    = builder.toppings;
        this.glutenFree  = builder.glutenFree;
    }

    public static class Builder {
        private final String size;    // required
        private final String crust;   // required
        private boolean extraCheese = false;
        private boolean extraSauce  = false;
        private java.util.List<String> toppings = new java.util.ArrayList<>();
        private boolean glutenFree  = false;

        public Builder(String size, String crust) {
            this.size  = size;
            this.crust = crust;
        }

        public Builder extraCheese()          { this.extraCheese = true; return this; }
        public Builder extraSauce()           { this.extraSauce  = true; return this; }
        public Builder glutenFree()           { this.glutenFree  = true; return this; }
        public Builder addTopping(String top) { this.toppings.add(top); return this; }
        public Pizza build()                  { return new Pizza(this); }
    }

    @Override
    public String toString() {
        return String.format("Pizza[%s, %s crust, cheese=%b, sauce=%b, GF=%b, toppings=%s]",
            size, crust, extraCheese, extraSauce, glutenFree, toppings);
    }
}

public class Main {
    public static void main(String[] args) {
        // Fluent, readable construction
        Pizza pizza1 = new Pizza.Builder("Large", "Thin")
            .extraCheese()
            .addTopping("Mushrooms")
            .addTopping("Olives")
            .extraSauce()
            .build();

        Pizza pizza2 = new Pizza.Builder("Medium", "Thick")
            .glutenFree()
            .addTopping("Pepperoni")
            .build();

        System.out.println(pizza1);
        System.out.println(pizza2);
    }
}
```

---

### Observer Pattern — Event-driven notification

```java
import java.util.*;

// Real-life: YouTube notifications, stock price alerts, event listeners
interface Observer {
    void update(String event, Object data);
}

interface Observable {
    void subscribe(Observer observer);
    void unsubscribe(Observer observer);
    void notifyObservers(String event, Object data);
}

class StockMarket implements Observable {
    private List<Observer> observers = new ArrayList<>();
    private Map<String, Double> prices = new HashMap<>();

    public void subscribe(Observer o)   { observers.add(o); }
    public void unsubscribe(Observer o) { observers.remove(o); }

    public void notifyObservers(String event, Object data) {
        for (Observer o : observers) o.update(event, data);
    }

    public void updatePrice(String stock, double newPrice) {
        double oldPrice = prices.getOrDefault(stock, newPrice);
        prices.put(stock, newPrice);
        double change = ((newPrice - oldPrice) / oldPrice) * 100;
        String event = change >= 0 ? "PRICE_UP" : "PRICE_DOWN";
        notifyObservers(event, stock + ": ₹" + newPrice
            + String.format(" (%+.2f%%)", change));
    }
}

class StockAlert implements Observer {
    private String investorName;

    StockAlert(String name) { this.investorName = name; }

    @Override
    public void update(String event, Object data) {
        String icon = event.equals("PRICE_UP") ? "📈" : "📉";
        System.out.println(icon + " [" + investorName + "] Alert: " + data);
    }
}

class StockLogger implements Observer {
    @Override
    public void update(String event, Object data) {
        System.out.println("📋 [LOG] " + new java.util.Date() + " | " + event + " | " + data);
    }
}

public class Main {
    public static void main(String[] args) {
        StockMarket market = new StockMarket();

        Observer alice  = new StockAlert("Alice");
        Observer bob    = new StockAlert("Bob");
        Observer logger = new StockLogger();

        market.subscribe(alice);
        market.subscribe(bob);
        market.subscribe(logger);

        System.out.println("=== Market Open ===");
        market.updatePrice("RELIANCE", 2500.00);
        market.updatePrice("TCS",       3800.00);

        System.out.println("\n=== Bob unsubscribed ===");
        market.unsubscribe(bob);
        market.updatePrice("RELIANCE", 2480.00); // only Alice and logger notified
    }
}
```

---

## 20. Summary Cheat Sheet

### 4 Pillars

| Pillar          | Key Idea                          | Mechanism                        |
|-----------------|-----------------------------------|----------------------------------|
| Encapsulation   | Hide data, expose behaviour       | `private` fields + getters/setters|
| Inheritance     | Reuse and extend                  | `extends`, `super`               |
| Polymorphism    | One interface, many forms         | Overloading + Overriding         |
| Abstraction     | Hide complexity, show essentials  | `abstract` class + `interface`   |

### Quick Reference

```
Class             → blueprint/template
Object            → instance of a class (new ClassName())
Constructor       → initialises object; same name as class, no return type
this              → current object reference
super             → parent class reference
static            → belongs to class, not instance
final             → variable: constant | method: no override | class: no extend
abstract          → class: can't instantiate | method: no body, must override
interface         → 100% contract (default/static methods allowed in Java 8+)
instanceof        → checks object type at runtime
extends           → class inheritance (single)
implements        → interface implementation (multiple allowed)
@Override         → annotation to verify you're correctly overriding
```

### Inheritance vs Composition

```
Inheritance (IS-A)            Composition (HAS-A)
class Dog extends Animal      class Car { Engine engine; }   ← prefer this
→ Dog IS-A Animal             → Car HAS-A Engine
→ tight coupling              → loose coupling, more flexible
→ use when genuine IS-A       → use when IS-A feels forced
```

### Method Overloading vs Overriding

| Feature           | Overloading                   | Overriding                         |
|-------------------|-------------------------------|------------------------------------|
| Where             | Same class                    | Parent-child relationship          |
| Parameters        | Must differ                   | Must be same                       |
| Return type       | Can differ                    | Must be same (or covariant)        |
| Resolved at       | Compile time                  | Runtime                            |
| Polymorphism type | Compile-time (static)         | Runtime (dynamic)                  |
| `@Override`       | Not applicable                | Recommended                        |

---

*OOP mastery is a journey — the best way to cement these concepts is to build real projects. Model the world around you as classes and objects.* 🏗️
