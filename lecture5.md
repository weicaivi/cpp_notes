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

template<typename T>
void f(T) {}               // Definition #2

int main() {
    f<Test>(10);  // Calls #1 - Test::foo exists
    f<int>(10);   // Calls #2 - int::foo doesn't exist, but no error
}
```

### std::enable_if Implementation

```cpp
template<bool B, class T = void>
struct enable_if {
    using type = T;
};

template<class T>
struct enable_if<false, T> {};  // No 'type' member when B is false

template<bool B, typename T = void>
using enable_if_t = typename enable_if<B, T>::type;
```

### Constraining the GCD Function

```cpp
#include <type_traits>

template<typename T, typename = std::enable_if_t<std::is_integral_v<T>>>
T gcd(T a, T b) {
    if (b == 0) return a;
    return gcd(b, a % b);  // Fixed: use modulo instead of subtraction
}

// Usage:
auto result1 = gcd(48, 30);      // OK: int is integral
auto result2 = gcd(4.0, 6.0);    // Compilation error: double not integral
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
class DistributedCounter1 {
    std::shared_mutex mtx;
    long long count = 0;
    
public:
    void operator++() {
        std::unique_lock lock(mtx);
        ++count;
    }
    
    long long get() const {
        std::shared_lock lock(mtx);
        return count;
    }
};

// Version 2: Bucket-based (reduced contention)
class DistributedCounter2 {
    struct Bucket {
        std::shared_mutex mtx;
        size_t count = 0;
    };
    
    static constexpr size_t BUCKET_COUNT = 128;
    std::vector<Bucket> buckets{BUCKET_COUNT};
    
public:
    void operator++() {
        size_t index = std::hash<std::thread::id>{}(std::this_thread::get_id()) % BUCKET_COUNT;
        std::unique_lock lock(buckets[index].mtx);
        ++buckets[index].count;
    }
    
    size_t get() const {
        return std::accumulate(buckets.begin(), buckets.end(), size_t{0},
            [](size_t acc, const Bucket& b) {
                std::shared_lock lock(b.mtx);
                return acc + b.count;
            });
    }
};

// Version 3: Cache-line padding (avoid false sharing)
class DistributedCounter3 {
    struct alignas(64) Bucket {  // Align to cache line
        std::shared_mutex mtx;
        size_t count = 0;
        char padding[64 - sizeof(mtx) - sizeof(count)];  // Prevent false sharing
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
