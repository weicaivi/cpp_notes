# Lecture 5 Summary

---

## Template Compilation and Instantiation

### Key Principle: Lazy Template Instantiation
Templates in C++ follow a "lazy instantiation" model where **only used methods are compiled**. This has important implications:

```cpp
template<int rows, int cols = rows>
class Matrix {
public:
    // This method won't be compiled unless called
    Matrix<rows-1, cols-1> minor(int r, int c) const {
        // Would fail for Matrix<1,1> but only if called
        Matrix<rows-1, cols-1> result;
        // ... implementation
        return result;
    }
};

// This compiles fine even though minor() would be invalid
Matrix<1,1> m;  // OK - minor() never called
```

**Benefits:**
- Allows creation of template classes even when some methods would be invalid
- Enables template specialization for edge cases
- Reduces compilation overhead

---

## Template Improvements

### 1. Static Assertions for Better Error Messages

**Problem:** Template errors often produce cryptic compiler messages.

```cpp
// Poor error message - compiler internals exposed
template<int rows, int cols>
class Matrix {
    double determinant() const {
        // Complex template error if not square
        return calculate_determinant();
    }
};
```

**Solution:** Use `static_assert` for clear compile-time checks:

```cpp
template<int rows, int cols>
class Matrix {
    double determinant() const {
        static_assert(rows == cols, "Determinant requires square matrix");
        // ... implementation
    }
};
```

### 2. The `*this` Idiom

Essential for method chaining and self-reference in templates:

```cpp
template<int rows, int cols>
class Matrix {
public:
    Matrix& operator+=(const Matrix& other) {
        *this = *this + other;  // *this refers to current object
        return *this;           // Enable chaining: m1 += m2 += m3;
    }
};
```

### 3. Inheriting Constructors

Reduces boilerplate when creating wrapper classes:

```cpp
template<typename T, int rows, int cols>
class MatrixBase {
public:
    MatrixBase() = default;
    MatrixBase(std::initializer_list<std::initializer_list<T>> init) { /* ... */ }
    // ... other constructors
};

template<typename T, int rows, int cols>
class Matrix : public MatrixBase<T, rows, cols> {
public:
    // Inherit all constructors from base class
    using MatrixBase<T, rows, cols>::MatrixBase;
    
    // Add specialized functionality
    T determinant() const { /* ... */ }
};
```

---

## Templates Before Concepts

### The Problem with Unconstrained Templates

Before C++20 Concepts, template errors occurred deep in instantiation:

```cpp
template<typename T>
void sort_container(T& container) {
    std::sort(container.begin(), container.end());  // May fail inside std::sort
}

std::vector<int> v = {3, 1, 4};
std::list<int> l = {5, 9, 2};

sort_container(v);  // OK - vector has random access iterators
sort_container(l);  // ERROR - cryptic message about list iterators
```

### C++20 Concepts Solution

```cpp
#include <concepts>

template<std::ranges::random_access_range T>
void sort_container(T& container) {
    std::sort(container.begin(), container.end());
}

// Now error is clear: "list does not satisfy random_access_range concept"
```

### Silent Failures: The GCD Example

Unconstrained templates can compile but produce wrong results:

```cpp
template<typename T>
T gcd(T a, T b) {
    if (b == 0) return a;
    return gcd(b, a - b * (a / b));
}

// Dangerous usage:
auto result1 = gcd(48, 30);     // OK: returns 6
auto result2 = gcd(4.0, 6.0);   // CRASH: infinite recursion
auto result3 = gcd(4.0, 8.0);   // WRONG: returns 4.0 (meaningless for floats)
```

---

## SFINAE and Template Constraints

### Understanding SFINAE
**"Substitution Failure Is Not An Error"** - Failed template substitutions remove candidates rather than causing compilation errors.

```cpp
struct Test {
    using foo = int;
};

template<typename T>
void f(typename T::foo) {}  // Definition #1
// writing `typename T:foo` to disambiguate to mean it's a type
// if we meant a value, we write `T::foo`

template<typename T>
void f(T) {}               // Definition #2
// we write T because T is always a type guaranteed by the template parameter list
// no ambiguity, T cannot be a value, T can only be a type
// writing `typename` is not needed and not allowed 

int main() {
    f<Test>(10);  // Calls #1 - Test::foo exists, T=Test, Test::foo=int → matches f(typename T::foo)
    f<int>(10);   // Calls #2 - int::foo doesn't exist, but no error, T=int, int::foo doesn't exist → so only f(T) is viable
}
```

### std::enable_if 

#### What is std::enable_if?

`std::enable_if` is a powerful metaprogramming SFINAE-based tool that transforms boolean conditions into the presence or absence of types, leveraging C++'s template substitution rules to control overload resolution:

1. **Conditionally enables/disables** template instantiations
2. **Uses substitution failure** to remove candidates without errors
3. **Enables type-safe generic programming** before C++20 concepts
4. **Works through template specialization** - the false case has no `type` member

**When to use**:
- Pre-C++20 codebases
- Complex conditional logic that concepts can't express
- Library code that needs maximum compatibility

**Modern alternatives**:
- `if constexpr` for simpler conditional compilation
- Concepts for cleaner type constraints
- Requires clauses for more readable conditions

**Key Purpose**: Remove template overloads from consideration during overload resolution without causing compilation errors.

---

#### The Implementation

Here's how `std::enable_if` is actually implemented in the standard library:

```cpp
// Primary template - when condition is true
template<bool B, class T = void>
struct enable_if {
    using type = T;  // 'type' member exists
};

// Partial specialization - when condition is false
template<class T>
struct enable_if<false, T> {
    // No 'type' member when B is false
    // This creates a substitution failure
};

// C++14 convenience alias
template<bool B, typename T = void>
using enable_if_t = typename enable_if<B, T>::type;
```

##### How This Works

1. **When `B` is `true`**: Uses primary template → `enable_if<true, T>::type` exists and equals `T`
2. **When `B` is `false`**: Uses specialization → `enable_if<false, T>::type` doesn't exist → **substitution failure**

---

#### How SFINAE Works

When the compiler tries to instantiate a template and substitution fails, instead of producing a compilation error, it simply removes that template from the candidate set.

##### Visual Example

```cpp
template<typename T>
void func(typename std::enable_if<std::is_integral<T>::value, T>::type param) {
    std::cout << "Integer version: " << param << std::endl;
}

template<typename T>
void func(typename std::enable_if<std::is_floating_point<T>::value, T>::type param) {
    std::cout << "Float version: " << param << std::endl;
}

int main() {
    func(42);    // Calls integer version
    func(3.14);  // Calls float version
    // func("hello"); // Would cause compilation error - no matching overload
}
```

**What happens during compilation:**

For `func(42)` where `T = int`:
1. **First overload**: `std::is_integral<int>::value` = `true` → `enable_if<true, int>::type` = `int` → **SUCCESS**
2. **Second overload**: `std::is_floating_point<int>::value` = `false` → `enable_if<false, int>::type` = **SUBSTITUTION FAILURE** → removed from candidates

For `func(3.14)` where `T = double`:
1. **First overload**: `std::is_integral<double>::value` = `false` → **SUBSTITUTION FAILURE** → removed
2. **Second overload**: `std::is_floating_point<double>::value` = `true` → **SUCCESS**

---

#### Examples

##### Example 1: Basic Usage

```cpp
#include <iostream>
#include <type_traits>

// Only enabled for integral types
template<typename T>
typename std::enable_if<std::is_integral<T>::value, void>::type
print_value(T value) { 
    // func return type is:
    // std::enable_if<true, void>::type -> void 
    // ::type comes from line 215 "using type = T" (called a member typedef, or nested type alias, of the class template enable_if)
    // "conditionally declared void functions"
    std::cout << "Integer: " << value << std::endl;
}

// Only enabled for floating-point types
template<typename T>
typename std::enable_if<std::is_floating_point<T>::value, void>::type
print_value(T value) {
    std::cout << "Float: " << value << std::endl;
}

int main() {
    print_value(42);      // Calls integer version
    print_value(3.14);    // Calls float version
    // print_value("hi");  // Compilation error - no matching function
}
```

##### Example 2: Using enable_if in Template Parameters

```cpp
// Method 1: In return type (shown above)
template<typename T>
typename std::enable_if<condition, ReturnType>::type func();

// Method 2: In template parameter list (cleaner)
// first template parameter: typename T
// second, unamed type parameter with a default argument, never used explicitly inside the function
// if T is integral, std::is_integral_v<T> is true, so std::enable_if_t<true> is just void
// second template parameter becomes typename = void, perfectly valid and function exists
// if false, SFINAE
template<typename T, typename = std::enable_if_t<std::is_integral_v<T>>>
void func(T value) {
    std::cout << "Integral: " << value << std::endl;
}

// Method 3: In function parameter
template<typename T>
void func(T value, std::enable_if_t<std::is_integral_v<T>>* = nullptr) {
    std::cout << "Integral: " << value << std::endl;
}
```

##### Example 3: Real-World GCD Function

```cpp
#include <type_traits>

// Constrained GCD function - only works with integral types
// the specifier constexpr makes the function eligible for compile-time eval, but doesn't force it
template<typename T, typename = std::enable_if_t<std::is_integral_v<T>>>
constexpr T gcd(T a, T b) {
    return b == 0 ? a : gcd(b, a % b);
}

// Usage:
int main() {
    auto result1 = gcd(48, 18);        // OK: returns 6
    auto result2 = gcd(100L, 35L);     // OK: works with long
    // auto result3 = gcd(4.5, 2.1);   // Compilation error: not integral
    
    std::cout << "GCD(48, 18) = " << result1 << std::endl;
    return 0;
}
```

##### Example 4: Container Type Detection

```cpp
#include <vector>
#include <list>
#include <iterator>

// Function for containers with random access iterators (like vector)
template<typename Container>
// the default second template argument of enable_if_t is void
// this is effectively void sort_container(Container& c)
std::enable_if_t<
    std::is_same_v<
        typename std::iterator_traits<typename Container::iterator>::iterator_category,
        std::random_access_iterator_tag
    >
> sort_container(Container& c) {
    std::sort(c.begin(), c.end());
    std::cout << "Sorted with std::sort (random access)" << std::endl;
}

// Function for containers without random access iterators (like list)
template<typename Container>
std::enable_if_t<
    !std::is_same_v<
        typename std::iterator_traits<typename Container::iterator>::iterator_category,
        std::random_access_iterator_tag
    >
> sort_container(Container& c) {
    c.sort();  // Use container's own sort method
    std::cout << "Sorted with container's sort method" << std::endl;
}

int main() {
    std::vector<int> vec = {3, 1, 4, 1, 5};
    std::list<int> lst = {9, 2, 6, 5, 3};
    
    sort_container(vec);  // Uses std::sort
    sort_container(lst);  // Uses list::sort
}
```

---

#### Common Usage Patterns

##### Pattern 1: Type Category Constraints

```cpp
// Only for arithmetic types
template<typename T, std::enable_if_t<std::is_arithmetic_v<T>, int> = 0>
// the point of `= 0` is just a dummy that defaults to 0
// without it, would be forced to write `multiply_by_two<int, 0>(5), which is ugly
T multiply_by_two(T value) {
    return value * 2;
}

// Only for pointer types
template<typename T, std::enable_if_t<std::is_pointer_v<T>, int> = 0>
auto dereference_safely(T ptr) -> decltype(*ptr) {
    // decltype(*ptr) means “deduce the type of the expression *ptr”
    // If T = int*, then *ptr is an int&, so the return type is int&
    // If T = const double*, then *ptr is const double&
    // The return type is exactly “whatever dereferencing the pointer yields.”
    return ptr ? *ptr : decltype(*ptr){};
    //  decltype(*ptr){} value-initializes an object of the dereferenced type
}
```

##### Pattern 2: Class Template Specialization

```cpp
template<typename T, typename Enable = void>
// Enable exists solely to be manipulated by enable_if_t in partial specializations
// sort of provides a hook for SFINAE
class Serializer {
public:
    static void serialize(const T& obj) {
        // Generic serialization
        std::cout << "Generic serialization" << std::endl;
    }
};

// Specialization for integral types
template<typename T>
class Serializer<T, std::enable_if_t<std::is_integral_v<T>>> {
// condition is true, Enable = void (matching the primary’s second parameter kind), so specialization compiles
public:
    static void serialize(const T& obj) {
        std::cout << "Integer serialization: " << obj << std::endl;
    }
};

// Specialization for string types
template<typename T>
class Serializer<T, std::enable_if_t<std::is_same_v<T, std::string>>> {
public:
    static void serialize(const T& obj) {
        std::cout << "String serialization: \"" << obj << "\"" << std::endl;
    }
};
```

##### Pattern 3: CRTP (Curiously Recurring Template Pattern) Constraints

```cpp
template<typename Derived>
class Base {
    // Compilte-time enforcement: ensure Derived actually derives from Base<Derived>
    static_assert(std::is_base_of_v<Base<Derived>, Derived>, 
                  "CRTP violation: Derived must inherit from Base<Derived>");

 // Base is a class template that takes the derived class as its template parameter

public:
    template<typename T = Derived> // template parameter makes the function itself a function template
    // a trick to allow SFINAE with enable_if in the rturn type
    //  resolves to void if condition is true.
	// If T is not Derived, substitution fails and the function is removed from overload resolution
    // In this case, since T defaults to Derived, the condition is always true -> the function exists.
    std::enable_if_t<std::is_same_v<T, Derived>, void>
    call_derived_method() {
        static_cast<Derived*>(this)->derived_method();
        // `this` is a pointer to Base<Derived>
        // static_cast<Derived*>(this) converts it to a pointer to the actual derived class
        // then we can call methods defined only in Derived
        // !allows the base class to call methods of the derived class safely at compile time, without virtual functions
    }
};

class MyClass : public Base<MyClass> {
    // Derived = MyClass, derived class passes itself as a template argument to its base class
public:
    void derived_method() {
        std::cout << "Derived method called" << std::endl;
    }
};
```

---

#### Advanced Examples

##### Multiple Constraints with enable_if

```cpp
#include <type_traits>

// Function that only works with signed integral types
template<typename T>
std::enable_if_t<
    std::is_integral_v<T> && std::is_signed_v<T>,
    T
> abs_value(T value) {
    return value < 0 ? -value : value;
}

// Function that only works with unsigned integral types  
template<typename T>
std::enable_if_t<
    std::is_integral_v<T> && std::is_unsigned_v<T>,
    T
> abs_value(T value) {
    return value;  // Already positive
}

// Function for floating-point types
template<typename T>
std::enable_if_t<std::is_floating_point_v<T>, T>
abs_value(T value) {
    return std::abs(value);
}

int main() {
    std::cout << abs_value(-5) << std::endl;      // signed int version
    std::cout << abs_value(5u) << std::endl;      // unsigned int version  
    std::cout << abs_value(-3.14) << std::endl;   // floating-point version
}
```

##### Member Function enable_if
mimics the behavior of `std::optional<T>`
```cpp
template<typename T>
// Optional is a class template parametrized by a type T
// designed to optionally hold a value of type T
class Optional {
private:
    bool has_value_ = false; // tracks whether the Optional currently holds a valid T
    alignas(T) char storage_[sizeof(T)];
    // storage_: raw, untyped memory large enough to store a T
    // alignas(T): ensure the memory is correctly aligned for T
    // we do manual placement new and manual destructor calls to control construction and destruction
    
public:
    // Constructor only available for copy-constructible types
    template<typename U = T> // U = T allows using SFINAE on the constructor
    Optional(const U& value, 
             std::enable_if_t<std::is_copy_constructible_v<U>, int> = 0) {
        new(storage_) T(value); // placement new, constructs T in the raw storage
        has_value_ = true;
    }
    
    
    // Move constructor only available for move-constructible types
    template<typename U = T>
    Optional(U&& value,
             std::enable_if_t<std::is_move_constructible_v<U>, int> = 0) {
        new(storage_) T(std::move(value)); // move-construct the value into the storage
        has_value_ = true;
    }
    
    // Assignment only available for assignable types
    template<typename U = T>
    std::enable_if_t<std::is_assignable_v<T&, const U&>, Optional&>
    operator=(const U& value) {
        if (has_value_) { // assign to the existing object
            reinterpret_cast<T&>(storage_) = value;
        } else {
            new(storage_) T(value);
            has_value_ = true;
        }
        return *this;
    }
    
    ~Optional() {
        if (has_value_) {
            reinterpret_cast<T&>(storage_).~T(); // Uses reinterpret_cast<T&> to access the object in raw storage
        }
    }
};
```

---

#### Modern Alternatives

##### C++17: if constexpr

```cpp
// Old way with enable_if
template<typename T>
std::enable_if_t<std::is_integral_v<T>, void>
process_old(T value) {
    std::cout << "Processing integer: " << value << std::endl;
}

template<typename T>
std::enable_if_t<std::is_floating_point_v<T>, void>
process_old(T value) {
    std::cout << "Processing float: " << value << std::endl;
}

// New way with if constexpr (single function)
template<typename T>
void process_new(T value) {
    if constexpr (std::is_integral_v<T>) {
        std::cout << "Processing integer: " << value << std::endl;
    } else if constexpr (std::is_floating_point_v<T>) {
        std::cout << "Processing float: " << value << std::endl;
    } else {
        std::cout << "Processing other type" << std::endl;
    }
}
```

##### C++20: Concepts

```cpp
#include <concepts>

// Old way
template<typename T, std::enable_if_t<std::is_integral_v<T>, int> = 0>
void func_old(T value) { /* ... */ }

// New way with concepts
template<std::integral T>
void func_new(T value) { /* ... */ }

// Custom concepts
template<typename T>
concept Numeric = std::is_arithmetic_v<T>;

template<Numeric T>
T add(T a, T b) {
    return a + b;
}
```

---

#### Common Pitfalls and Solutions

##### Pitfall 1: Ambiguous Overloads

```cpp
// PROBLEM: Both functions could match
template<typename T>
std::enable_if_t<std::is_arithmetic_v<T>, void> func(T) { }

template<typename T>  
std::enable_if_t<std::is_integral_v<T>, void> func(T) { }

// SOLUTION: Make conditions mutually exclusive
template<typename T>
std::enable_if_t<std::is_arithmetic_v<T> && !std::is_integral_v<T>, void> 
func(T) { /* floating-point version */ }

template<typename T>
std::enable_if_t<std::is_integral_v<T>, void> 
func(T) { /* integral version */ }
```

##### Pitfall 2: Complex Error Messages

```cpp
// PROBLEM: Complex enable_if expressions create unreadable errors
template<typename T>
std::enable_if_t<
    std::is_class_v<T> && 
    std::is_default_constructible_v<T> &&
    std::has_virtual_destructor_v<T> &&
    !std::is_abstract_v<T>,
    std::unique_ptr<T>
> create_object() {
    return std::make_unique<T>();
}

// SOLUTION: Use type traits for clarity
template<typename T>
struct is_valid_class : std::conjunction<
    std::is_class<T>,
    std::is_default_constructible<T>,
    std::has_virtual_destructor<T>,
    std::negation<std::is_abstract<T>>
> {};

template<typename T>
std::enable_if_t<is_valid_class_v<T>, std::unique_ptr<T>>
create_object() {
    return std::make_unique<T>();
}
```

---

## Constexpr Programming

### Compile-Time vs Runtime Computation

`constexpr` enables computation during compilation, improving performance and enabling template parameters:

```cpp
// Runtime computation
double pi_runtime() {
    return 4.0 * std::atan(1.0);
}

// Compile-time computation
constexpr double pi_compiletime() {
    return 4.0 * std::atan(1.0);  // Computed during compilation
}

// Usage in templates
template<int N = static_cast<int>(pi_compiletime() * 100)>
class CircleCalculator {
    static constexpr double pi_scaled = N / 100.0;
};
```

### Constexpr Functions: Rules and Behavior

```cpp
constexpr int factorial(int n) {
    return (n <= 1) ? 1 : n * factorial(n - 1);
}

void demonstrate_constexpr() {
    // Compile-time evaluation (required for template parameter)
    std::array<int, factorial(5)> arr;  // Size = 120
    
    // May be compile-time or runtime (compiler's choice)
    constexpr int result = factorial(6);
    
    // Runtime evaluation (input not known at compile-time)
    int x;
    std::cin >> x;
    int runtime_result = factorial(x);
}
```

### Constexpr Restrictions

Constexpr functions cannot contain:
- Uninitialized variables
- `static` or `thread_local` variables
- Calls to non-constexpr functions
- Virtual function calls
- Non-literal types

### if constexpr: Conditional Compilation

Solves the template compilation problem:

```cpp
template<int rows, int cols>
class Matrix {
public:
    double determinant() const {
        if constexpr (rows == 1 && cols == 1) {
            return data[0][0];  // Simple case
        } else {
            // Complex case - not compiled for 1x1 matrices
            double result = 0;
            for (int i = 0; i < rows; ++i) {
                result += (i % 2 ? -1 : 1) * data[i][0] * minor(i, 0).determinant();
            }
            return result;
        }
    }
private:
    std::array<std::array<double, cols>, rows> data;
    Matrix<rows-1, cols-1> minor(int r, int c) const;  // Only compiled when needed
};
```

---

## Type Deduction with decltype

### Basic Usage

```cpp
template<typename T, typename U>
auto multiply(T t, U u) -> decltype(t * u) {
    return t * u;
}

// Modern C++14+ style (return type deduction)
template<typename T, typename U>
auto multiply(T t, U u) {
    return t * u;  // Type deduced from return expression
}
```

### Advanced decltype Patterns

```cpp
template<typename Container>
void process_container(Container& c) {
    using value_type = decltype(*c.begin());  // Deduces element type
    using size_type = decltype(c.size());     // Deduces size type
    
    // Create matrix based on container's value type
    Matrix<decltype(c.size()), 1> result;
}
```

---

## Exception Handling and RAII

### The Resource Management Problem

```cpp
// DANGEROUS: Resources not released if exception thrown
void dangerous_function() {
    acquire_resource();
    risky_operation();  // May throw exception
    release_resource(); // Never called if exception thrown
}
```

### RAII Solution

**Resource Acquisition Is Initialization** - Use object lifetimes to manage resources:

```cpp
class ResourceManager {
public:
    ResourceManager() { acquire_resource(); }
    ~ResourceManager() { release_resource(); }  // Always called
    
    // Prevent copying for unique ownership
    ResourceManager(const ResourceManager&) = delete;
    ResourceManager& operator=(const ResourceManager&) = delete;
    
    // Enable moving
    ResourceManager(ResourceManager&& other) noexcept : resource(other.resource) {
        other.resource = nullptr;
    }
};

void safe_function() {
    ResourceManager manager;  // Resource acquired
    risky_operation();        // Exception-safe
    // Resource automatically released when manager goes out of scope
}
```

### Standard RAII Classes

```cpp
#include <memory>
#include <mutex>
#include <thread>

void demonstrate_raii() {
    // Memory management
    auto ptr = std::make_unique<ExpensiveObject>();
    
    // Lock management
    std::mutex mtx;
    {
        std::lock_guard<std::mutex> lock(mtx);
        // Critical section
        // Lock automatically released at end of scope
    }
    
    // Thread management (C++20)
    std::jthread t([]{ /* work */ });
    // Thread automatically joined in destructor
}
```

---

## Multithreading and Concurrency

### Basic Threading

```cpp
#include <thread>
#include <iostream>
#include <mutex>

std::mutex io_mutex;

void thread_function(int id) {
    std::lock_guard<std::mutex> lock(io_mutex);
    std::cout << "Thread " << id << " executing\n";
}

int main() {
    std::vector<std::thread> threads;
    
    // Create threads
    for (int i = 0; i < 4; ++i) {
        threads.emplace_back(thread_function, i);
    }
    
    // Wait for all threads to complete
    for (auto& t : threads) {
        t.join();
    }
    
    return 0;
}
```

### Thread Argument Gotchas

```cpp
void modify_value(int& value) {
    value = 42;
}

void demonstrate_thread_args() {
    int x = 10;
    
    // WRONG: Copies x, doesn't modify original
    std::thread t1(modify_value, x);
    
    // CORRECT: Uses reference wrapper
    std::thread t2(modify_value, std::ref(x));
    
    t1.join();
    t2.join();
    
    std::cout << x << std::endl;  // Prints 42
}
```

### Mutex Types and Lock Management

```cpp
#include <shared_mutex>

class ThreadSafeContainer {
private:
    std::vector<int> data;
    mutable std::shared_mutex mtx;
    
public:
    // Writer - exclusive access
    void add(int value) {
        std::unique_lock<std::shared_mutex> lock(mtx);
        data.push_back(value);
    }
    
    // Reader - shared access
    bool contains(int value) const {
        std::shared_lock<std::shared_mutex> lock(mtx);
        return std::find(data.begin(), data.end(), value) != data.end();
    }
    
    // Reader - shared access
    size_t size() const {
        std::shared_lock<std::shared_mutex> lock(mtx);
        return data.size();
    }
};
```

### Deadlock Prevention

```cpp
#include <mutex>

class BankAccount {
private:
    std::mutex mtx;
    double balance;
    
public:
    BankAccount(double initial_balance) : balance(initial_balance) {}
    
    // DEADLOCK-PRONE VERSION
    void transfer_unsafe(BankAccount& to, double amount) {
        std::lock_guard<std::mutex> lock1(mtx);      // Lock this account
        std::lock_guard<std::mutex> lock2(to.mtx);   // Lock target account
        // Deadlock if another thread transfers in opposite direction!
        
        balance -= amount;
        to.balance += amount;
    }
    
    // DEADLOCK-SAFE VERSION
    void transfer_safe(BankAccount& to, double amount) {
        std::scoped_lock lock(mtx, to.mtx);  // Locks both mutexes safely
        balance -= amount;
        to.balance += amount;
    }
};
```

---

## Code Analysis and Examples

### Matrix Template Implementation Analysis

The provided Matrix class demonstrates several advanced concepts:

```cpp
template<int rows, int cols = rows>
class Matrix {
private:
    std::array<std::array<double, cols>, rows> data;
    
public:
    // Default initialization
    Matrix() : data{} {}
    
    // Initializer list constructor
    Matrix(std::initializer_list<std::initializer_list<double>> init) {
        auto dp = data.begin();
        for (auto row : init) {
            std::copy(row.begin(), row.end(), dp->begin());
            ++dp;
        }
    }
    
    // Access operators
    double& operator()(int x, int y) { return data[x][y]; }
    double operator()(int x, int y) const { return data[x][y]; }
    
    // Compound assignment
    Matrix& operator+=(const Matrix& other) {
        *this = *this + other;
        return *this;
    }
    
    // Minor matrix calculation
    Matrix<rows-1, cols-1> minor(int r, int c) const {
        Matrix<rows-1, cols-1> result;
        for (int i = 0; i < rows; ++i) {
            if (i == r) continue;
            for (int j = 0; j < cols; ++j) {
                if (j == c) continue;
                result(i < r ? i : i-1, j < c ? j : j-1) = data[i][j];
            }
        }
        return result;
    }
    
    // Determinant with compile-time constraints
    double determinant() const {
        static_assert(rows == cols, "Determinant requires square matrix");
        
        if constexpr (rows == 1) {
            return data[0][0];
        } else if constexpr (rows == 2) {
            return data[0][0] * data[1][1] - data[0][1] * data[1][0];
        } else {
            double result = 0;
            for (int i = 0; i < rows; ++i) {
                result += (i % 2 ? -1 : 1) * data[i][0] * minor(i, 0).determinant();
            }
            return result;
        }
    }
};
```

### Distributed Counter Implementation

Analysis of the three distributed counter versions shows evolution of thread-safe design:

```cpp
// Version 1: Simple mutex (high contention)
// every increment by any thread must acquire the same lock
// If many threads increment at once, they block each other -> poor scalability
class DistributedCounter1 {
    std::shared_mutex mtx;
    long long count = 0;
    
public:
    // takes a unique lock for writing
    void operator++() {
        std::unique_lock lock(mtx);
        ++count;
    }
    
    // takes a shared lock for reading 
    long long get() const {
        std::shared_lock lock(mtx);
        return count;
    }
};

// Version 2: Bucket-based (reduced contention)
// multiple “buckets”, each with its own counter and mutex
// Each thread hashes its thread::id to choose a bucket
// Still some possible collisions if threads hash to the same bucket
// Buckets may be stored contiguously in memory, can suffer false sharing
// which is when multiple threads write to variables on the same cache line,
// causing performance degradation even if the variables are logically independent
class DistributedCounter2 {
    struct Bucket {
        std::shared_mutex mtx;
        size_t count = 0;
    };
    
    static constexpr size_t BUCKET_COUNT = 128;
    std::vector<Bucket> buckets{BUCKET_COUNT};
    
public:
    // Increment only locks one bucket, not the entire counter
    void operator++() {
        size_t index = std::hash<std::thread::id>{}(std::this_thread::get_id()) % BUCKET_COUNT;
        // std::hash is a functor, produces a size_t hash for a thread ID
        // {} constructs a temporary hash object
        // operator()(...) immediately calls the hash object with the thread ID
        std::unique_lock lock(buckets[index].mtx);
        ++buckets[index].count;
    }
    
    // sums all buckets with shared locks
    size_t get() const {
        return std::accumulate(buckets.begin(), buckets.end(), size_t{0},
            [](size_t acc, const Bucket& b) {
                std::shared_lock lock(b.mtx);
                return acc + b.count;
            });
    }
};

// Version 3: Cache-line padding (avoid false sharing)
// Same idea as Version 2: multiple buckets, each with its own mutex
// New addition: Each bucket is aligned to and thu occupies a full cache line (usually 64 bytes)
// Avoids cache-coherence bottlenecks, better scaling under high concurrency
class DistributedCounter3 {
    struct alignas(64) Bucket {  // Align to cache line
        std::shared_mutex mtx;
        size_t count = 0;
        char padding[64 - sizeof(mtx) - sizeof(count)];  // Prevent false sharing
        // char padding[...] ensures that each Bucket occupies exactly one cache line
        // Calculates how many bytes remain in the cache line after mtx and count
        // Fills the remainder with a dummy char array (padding)
    };
    
    static constexpr size_t BUCKET_COUNT = 128;
    std::vector<Bucket> buckets{BUCKET_COUNT};
    
public:
    // Same interface as Version 2
    void operator++() { /* ... */ }
    size_t get() const { /* ... */ }
};
```

### Performance Considerations

1. **False Sharing Prevention**: Version 3 uses padding to ensure each bucket occupies its own cache line
2. **Lock Granularity**: Distributed approach reduces contention compared to single mutex
3. **Thread-Local Hashing**: Uses thread ID to distribute load across buckets

### Best Practices Summary

1. **Template Design**:
   - Use `static_assert` for clear error messages
   - Leverage `if constexpr` for conditional compilation
   - Apply SFINAE/concepts for type constraints

2. **Resource Management**:
   - Always use RAII for resource cleanup
   - Prefer smart pointers over raw pointers
   - Use standard RAII classes (`lock_guard`, `unique_ptr`, etc.)

3. **Concurrency**:
   - Avoid data races with proper synchronization
   - Use `scoped_lock` for multiple mutexes
   - Consider lock-free data structures for high-performance scenarios
   - Be aware of false sharing in multi-threaded environments

4. **Modern C++ Features**:
   - Leverage `constexpr` for compile-time computation
   - Use `auto` and `decltype` for type deduction
   - Apply structured bindings and range-based loops where appropriate
