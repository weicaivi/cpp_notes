# Advanced C++ Lecture 2 Summary

## Memory Management and Object Lifetime

### Code and Data Relationship
C++ programs refer to data objects through expressions, while the actual objects exist in computer memory at runtime. 

### Stack vs Heap Memory
```cpp
// Stack allocation - automatic lifetime management
int i = 5;  // Memory allocated on stack, automatically freed when out of scope

// Heap allocation - manual lifetime management required
int* ptr = new int(5);  // Memory allocated on heap
delete ptr;             // Must manually free memory
```

### Copying and Assignment Semantics
C++ uses value semantics by default - assignment creates copies:

```cpp
int i = 3;
int j = i;  // j gets its own copy of i's value
i = 5;      // Changing i doesn't affect j
cout << j;  // Prints 3
```

**Key Difference from Java/Python**: In C++, user-defined types are also copied by assignment, unlike reference semantics in Java/Python for objects.

---

## References vs Values

### Reference Basics
References provide an alias to existing objects without creating copies:

```cpp
int i = 3;
int& j = i;  // j is a reference to i (same memory location)
i = 5;       // Modifying i also affects j
cout << j;   // Prints 5
```

### Reference Characteristics
- **No separate storage**: References don't allocate new memory
- **Must be initialized**: Cannot declare uninitialized references
- **Cannot be reassigned**: Once bound, always refers to the same object
- **Same usage as values**: After initialization, use references like regular variables

### Lifetime Safety Warning
```cpp
// DANGEROUS: Returning reference to local variable
int& dangerous_function() {
    int local_var = 42;
    return local_var;  // local_var destroyed when function ends
}  // Undefined behavior when using returned reference
```

---

## Dynamic Memory Management

### The Problem with Manual Memory Management
Traditional C-style memory management is error-prone:
- **Early release**: Causes crashes from accessing freed memory
- **Memory leaks**: Forgetting to free memory leads to resource exhaustion

### RAII and Smart Pointers
C++ uses Resource Acquisition Is Initialization (RAII) for automatic memory management:

```cpp
#include <memory>

// Create object with automatic lifetime management
auto ui = std::make_unique<int>(5);
cout << *ui;  // Access the value

// Memory automatically freed when ui goes out of scope
// No explicit delete needed
```

### Smart Pointer Operations
```cpp
// Creating smart pointers
auto ui = std::make_unique<int>(5);
auto student = std::make_unique<Student_info>("Alice");

// Accessing the managed object
cout << *ui;                    // Dereference to get value
cout << student->name;          // Access member via ->
cout << (*student).name;        // Equivalent to above

// Transferring ownership
auto ui2 = std::move(ui);       // ui2 now owns the object, ui is null
// Don't use ui after move!
```

### Benefits of Smart Pointers
- **Automatic cleanup**: Memory freed when smart pointer is destroyed
- **Exception safety**: Memory cleaned up even if exceptions occur
- **Clear ownership**: `unique_ptr` indicates single ownership

---

## Classes and Object-Oriented Programming

### Class Definition Syntax
```cpp
// Using struct (members public by default)
struct Student_info {
    std::string name;
    double midterm = 0.0;
    double final = 0.0;
    std::vector<double> homework;
    
    // Constructor with member initializer list
    Student_info(const std::string& student_name) 
        : name(student_name), midterm(0), final(0) {}
    
    // Member function
    double grade() const {
        double avg = std::accumulate(homework.begin(), homework.end(), 0.0) / homework.size();
        return (midterm + final + avg) / 3.0;
    }
    
    // Input function
    std::istream& read(std::istream& is) {
        double hw;
        while (is >> hw && hw >= 0) {
            homework.push_back(hw);
        }
        return is;
    }
};

// Equivalent using class (members private by default)
class Student_info {
public:  // Must explicitly make members public
    std::string name;
    double midterm = 0.0;
    // ... rest of implementation
};
```

### Member Access Control
```cpp
class AccessExample {
private:
    int private_member;    // Only accessible within this class

protected:
    int protected_member;  // Accessible in this class and subclasses

public:
    int public_member;     // Accessible everywhere
    
    void member_function() {
        private_member = 1;    // OK
        protected_member = 2;  // OK
        public_member = 3;     // OK
    }
};

class Derived : public AccessExample {
    void derived_function() {
        // private_member = 1;    // ERROR: not accessible
        protected_member = 2;     // OK
        public_member = 3;        // OK
    }
};

void external_function(AccessExample& obj) {
    // obj.private_member = 1;    // ERROR
    // obj.protected_member = 2;  // ERROR
    obj.public_member = 3;        // OK
}
```

### Constructor Best Practices
```cpp
class BestPracticeClass {
private:
    int x, y;
    std::string name;
    
public:
    // Prefer member initializer lists
    BestPracticeClass(int a, int b, const std::string& n) 
        : x(a), y(b), name(n) {}  // Direct initialization
    
    // Avoid assignment in constructor body
    BestPracticeClass(int a) : x(a), y(0) {
        // name = "default";  // Assignment, less efficient than initialization
    }
};
```

**Important**: Members are initialized in declaration order, not initializer list order!

```cpp
class OrderExample {
    int x, y;  // Declaration order matters
public:
    OrderExample(int val) : y(val), x(y + 1) {}  // x initialized first with uninitialized y!
    // Correct: OrderExample(int val) : x(val), y(val + 1) {}
};
```

---

## Inheritance and Polymorphism

### Static vs Dynamic Types
- **Static type**: Type of expression known at compile time
- **Dynamic type**: Type of actual object, may differ due to inheritance

```cpp
class Animal {
public:
    void non_virtual_func() { std::cout << "Animal"; }
    virtual void virtual_func() { std::cout << "Animal"; }
};

class Dog : public Animal {
public:
    void non_virtual_func() { std::cout << "Dog"; }
    void virtual_func() override { std::cout << "Dog"; }
};

void demonstrate_types() {
    auto dog = std::make_unique<Dog>();
    Animal& animal_ref = *dog;  // Static type: Animal&, Dynamic type: Dog
    
    animal_ref.non_virtual_func();  // Prints "Animal" (uses static type)
    animal_ref.virtual_func();      // Prints "Dog" (uses dynamic type)
}
```

### Abstract Base Classes and Pure Virtual Functions
```cpp
struct Abstract_student_info {
    std::string name;
    double midterm = 0;
    double final = 0;
    std::vector<double> homework;
    
    Abstract_student_info(const std::string& student_name) : name(student_name) {}
    
    // Pure virtual function makes this class abstract
    virtual double grade() const = 0;
    
    virtual ~Abstract_student_info() = default;  // Virtual destructor important!
};

// Concrete implementations
struct BalancedGrading : public Abstract_student_info {
    using Abstract_student_info::Abstract_student_info;  // Inherit constructors
    
    double grade() const override {
        double avg = std::accumulate(homework.begin(), homework.end(), 0.0) / homework.size();
        return (midterm + final + avg) / 3.0;
    }
};

struct IgnoreHomework : public Abstract_student_info {
    using Abstract_student_info::Abstract_student_info;
    
    double grade() const override {
        return (midterm + final) / 2.0;
    }
};
```

### Polymorphic Usage
```cpp
void use_polymorphism() {
    std::vector<std::unique_ptr<Abstract_student_info>> students;
    
    students.push_back(std::make_unique<BalancedGrading>("Alice"));
    students.push_back(std::make_unique<IgnoreHomework>("Bob"));
    
    for (auto& student : students) {
        student->midterm = 85;
        student->final = 92;
        student->homework = {88, 76, 94};
        
        std::cout << student->name << ": " << student->grade() << std::endl;
        // Calls appropriate grade() method based on dynamic type
    }
}
```

---

## Virtual Function Performance

### Performance Characteristics
Virtual functions typically add minimal overhead (one extra indirection), but can impact performance in specific scenarios due to:

1. **Loss of inlining opportunities**
2. **Reduced compiler optimization**
3. **Memory layout changes**

### Performance Comparison Example
```cpp
class NonVirtual {
public:
    int compute(double i1, int i2) {
        return static_cast<int>(i1 * std::log(i1)) * i2;
    }
};

class Virtual {
public:
    virtual int compute(double i1, int i2) {
        return static_cast<int>(i1 * std::log(i1)) * i2;
    }
};

// Benchmark results can vary dramatically based on usage pattern
void benchmark_scenario1() {
    auto obj = std::make_unique<Virtual>();
    for (int i = 0; i < 100'000'000; ++i) {
        obj->compute(i, 10);  // Different values each call - minimal difference
    }
}

void benchmark_scenario2() {
    auto obj = std::make_unique<Virtual>();
    for (int i = 0; i < 100'000'000; ++i) {
        obj->compute(10, i);  // Constant first parameter - virtual can be 70x slower!
    }
}
```

### When the Performance Difference Matters
The second scenario is slower because:
- Non-virtual: Compiler can optimize `std::log(10)` to a constant
- Virtual: Compiler cannot assume the function won't be overridden, preventing optimization

### Best Practices for Virtual Functions
- Use virtual functions when polymorphism is needed
- Don't use virtual gratuitously in performance-critical code
- Consider the optimization implications in tight loops
- Profile before optimizing

---

## Design Patterns and Best Practices

### Strategy Pattern Implementation
two approaches to implementing the Strategy pattern:

#### Approach 1: Inheritance-Based Strategy
```cpp
// Multiple inheritance hierarchies lead to combinatorial explosion
// MPCS_Student + BalancedGrading = MPCS_BalancedGrading_Student
// MPCS_Student + IgnoreHomework = MPCS_IgnoreHomework_Student
// Undergraduate_Student + BalancedGrading = Undergraduate_BalancedGrading_Student
// This quickly becomes unmanageable!
```

#### Approach 2: Composition-Based Strategy (Preferred)
```cpp
// Forward declaration
struct NewStudent_info;

// Strategy interface
struct GradingMachine {
    virtual ~GradingMachine() = default;
    virtual double grade(const NewStudent_info& student) = 0;
};

// Context class
struct NewStudent_info {
    std::string name;
    double midterm = 0;
    double final = 0;
    std::vector<double> homework;
    std::unique_ptr<GradingMachine> grading_machine;
    
    NewStudent_info(const std::string& student_name, 
                   std::unique_ptr<GradingMachine> initial_machine)
        : name(student_name), grading_machine(std::move(initial_machine)) {}
    
    void update_grading_machine(std::unique_ptr<GradingMachine> new_machine) {
        grading_machine = std::move(new_machine);
    }
    
    double grade() const {
        return grading_machine->grade(*this);
    }
    
    std::istream& read(std::istream& is) {
        double hw;
        while (is >> hw && hw >= 0) {
            homework.push_back(hw);
        }
        return is;
    }
};

// Strategy implementations
struct BalancedGradingMachine : public GradingMachine {
    double grade(const NewStudent_info& student) override {
        if (student.homework.empty()) return (student.midterm + student.final) / 2.0;
        double avg = std::accumulate(student.homework.begin(), student.homework.end(), 0.0) 
                    / student.homework.size();
        return (student.midterm + student.final + avg) / 3.0;
    }
};

struct IgnoreHomeworkGradingMachine : public GradingMachine {
    double grade(const NewStudent_info& student) override {
        return (student.midterm + student.final) / 2.0;
    }
};
```

### Usage Example
```cpp
void demonstrate_strategy_pattern() {
    // Create student with initial grading strategy
    auto lorenzo = std::make_unique<NewStudent_info>(
        "Lorenzo", 
        std::make_unique<BalancedGradingMachine>()
    );
    
    lorenzo->midterm = 80;
    lorenzo->final = 100;
    lorenzo->homework = {85, 90, 78};
    
    std::cout << "Balanced grading: " << lorenzo->grade() << std::endl;
    
    // Change strategy at runtime
    lorenzo->update_grading_machine(std::make_unique<IgnoreHomeworkGradingMachine>());
    std::cout << "Ignore homework: " << lorenzo->grade() << std::endl;
}
```

---

## Code Examples and Implementations

### Pascal's Triangle Implementation

#### Modern C++ Version
```cpp
#include <vector>
#include <iostream>
#include <string>
#include <algorithm>
#include <format>

using Row = std::vector<int>;
using Triangle = std::vector<Row>;

Row next_row(const Row& current_row) {
    Row result;
    result.reserve(current_row.size() + 1);
    
    int previous = 0;
    for (int element : current_row) {
        result.push_back(previous + element);
        previous = element;
    }
    result.push_back(previous);
    return result;
}

constexpr int ROWS = 10;

int num_digits(int number) {
    return std::to_string(number).size();
}

void print_row(std::ostream& os, const Row& row, int element_width) {
    for (int element : row) {
        os << std::format("{:^{}}", element, element_width) << " ";
    }
    os << std::endl;
}

std::ostream& operator<<(std::ostream& os, const Triangle& triangle) {
    if (triangle.empty()) return os;
    
    const Row& last_row = triangle.back();
    int max_element = *std::max_element(last_row.begin(), last_row.end());
    int element_width = num_digits(max_element);
    
    for (size_t i = 0; i < triangle.size(); ++i) {
        // Center the row
        std::string spaces((triangle.size() - i - 1) * (element_width + 1) / 2, ' ');
        os << spaces;
        print_row(os, triangle[i], element_width);
    }
    return os;
}

int main() {
    Triangle triangle;
    triangle.reserve(ROWS); 
    
    Row current_row = {1};
    for (int i = 0; i < ROWS; ++i) {
        triangle.push_back(current_row);
        current_row = next_row(current_row);
    }
    
    std::cout << triangle;
    return 0;
}
```

### Performance Benchmarking Utilities
```cpp
#include <chrono>
#include <iostream>
#include <functional>

template<typename Func>
auto benchmark(Func&& func, int iterations = 1) {
    using namespace std::chrono;
    
    auto start = high_resolution_clock::now();
    for (int i = 0; i < iterations; ++i) {
        func();
    }
    auto end = high_resolution_clock::now();
    
    return duration_cast<milliseconds>(end - start);
}

// Usage example
void example_benchmark() {
    auto duration = benchmark([]() {
        volatile int sum = 0;  // volatile prevents optimization
        for (int i = 0; i < 1000000; ++i) {
            sum += i * i;
        }
    }, 100);
    
    std::cout << "Operation took: " << duration.count() << "ms" << std::endl;
}
```

---

## Key Takeaways and Best Practices

### Memory Management
1. **Prefer stack allocation** when possible for automatic lifetime management
2. **Use smart pointers** (`std::unique_ptr`, `std::shared_ptr`) instead of raw pointers
3. **Avoid manual `new`/`delete`** in modern C++
4. **Be careful with reference lifetimes** - ensure referenced objects outlive references

### Class Design
1. **Use member initializer lists** in constructors for efficiency
2. **Make destructors virtual** in base classes intended for inheritance
3. **Follow Rule of Five** when managing resources manually
4. **Prefer composition over inheritance** to avoid combinatorial explosion

### Performance
1. **Don't optimize prematurely** - profile first to find actual bottlenecks
2. **Be aware of virtual function costs** in performance-critical code
3. **Use proper benchmarking tools** rather than simple timing code
4. **Understand compiler optimizations** and how they interact with language features

### Modern C++ Features
1. **Use `auto`** for type deduction when appropriate
2. **Prefer `constexpr`** over `const` for compile-time constants
3. **Use range-based for loops** when possible
4. **Leverage standard library algorithms** instead of manual loops
