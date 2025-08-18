# Lecture 4 Summary

---

## Callables and Lambda Functions

### What Are Callables?

A "callable" is any object that can be invoked with the function call operator `()`. There are four main types of callables:

1. **Regular Functions**
2. **Lambda Expressions**
3. **Function Objects (Functors)**
4. **Member Functions** (with appropriate binding)

### Lambda Functions

Lambda expressions provide a concise way to define anonymous functions inline.

#### Basic Syntax
```cpp
[capture list](parameters) -> return_type { function body }
```

#### Examples

**Simple Lambda:**
```cpp
auto square = [](int x) { return x * x; };
std::cout << square(5); // Output: 25
```

**Lambda with Return Type Specification:**
```cpp
auto divide = [](double a, double b) -> double {
    if (b != 0) return a / b;
    return 0.0;
};
```

**Polymorphic Lambda (C++14+):**
```cpp
auto multiply = [](auto x, auto y) { return x * y; };
std::cout << multiply(3, 4);     // 12 (int)
std::cout << multiply(2.5, 4.0); // 10.0 (double)
```

### Capture Lists

Capture lists determine how variables from the surrounding scope are accessed within the lambda.

#### Capture Types
- `[=]` - Capture all variables by value (copy)
- `[&]` - Capture all variables by reference
- `[var]` - Capture `var` by value
- `[&var]` - Capture `var` by reference
- `[=, &var]` - Capture all by value except `var` by reference
- `[&, var]` - Capture all by reference except `var` by value

#### Capture Examples
```cpp
int main() {
    int x = 10;
    int y = 20;
    
    // Capture by value
    auto lambda1 = [=]() { 
        std::cout << x + y; // Uses copies of x and y
    };
    
    // Capture by reference
    auto lambda2 = [&]() { 
        x += y; // Modifies original x
    };
    
    // Mixed capture
    auto lambda3 = [x, &y](int z) { 
        return x + y + z; // x by value, y by reference
    };
    
    // Mutable lambda (allows modification of captured values)
    auto lambda4 = [=]() mutable { 
        x++; // Modifies the copy, not original
        return x;
    };
    
    return 0;
}
```

### Function Objects (Functors)

Functors are classes that overload the `operator()`, making instances callable like functions.

```cpp
struct Accumulator {
    int total = 0;
    
    int operator()(int value) {
        total += value;
        return total;
    }
};

// Usage
Accumulator acc;
std::cout << acc(5);  // 5
std::cout << acc(3);  // 8
std::cout << acc(2);  // 10
```

**Advanced Functor Example:**
```cpp
template<typename T>
class Multiplier {
private:
    T factor;
    
public:
    Multiplier(T f) : factor(f) {}
    
    T operator()(T value) const {
        return value * factor;
    }
};

// Usage
Multiplier<double> doubleIt(2.0);
Multiplier<int> tripleIt(3);

std::cout << doubleIt(4.5); // 9.0
std::cout << tripleIt(7);   // 21
```

---

## C++ Standard Library Algorithms

### Core Algorithm Categories

#### 1. Non-Modifying Sequence Operations
```cpp
#include <algorithm>
#include <vector>
#include <iostream>

std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

// all_of, any_of, none_of
bool allEven = std::all_of(data.begin(), data.end(), 
                          [](int n) { return n % 2 == 0; });

bool anyEven = std::any_of(data.begin(), data.end(), 
                          [](int n) { return n % 2 == 0; });

bool noneNegative = std::none_of(data.begin(), data.end(), 
                                [](int n) { return n < 0; });

// find_if and find_if_not
auto firstEven = std::find_if(data.begin(), data.end(), 
                             [](int n) { return n % 2 == 0; });

auto firstOdd = std::find_if_not(data.begin(), data.end(), 
                                [](int n) { return n % 2 == 0; });
```

#### 2. Modifying Sequence Operations
```cpp
// transform
std::vector<int> source = {1, 2, 3, 4, 5};
std::vector<int> squares;

std::transform(source.begin(), source.end(), 
               std::back_inserter(squares),
               [](int x) { return x * x; });

// copy_n
std::vector<int> destination(3);
std::copy_n(source.begin(), 3, destination.begin());

// partition_copy
std::vector<int> evens, odds;
std::partition_copy(source.begin(), source.end(),
                   std::back_inserter(evens),
                   std::back_inserter(odds),
                   [](int n) { return n % 2 == 0; });
```

#### 3. Sorting and Related Operations
```cpp
#include <algorithm>
#include <vector>

std::vector<int> data = {3, 1, 4, 1, 5, 9, 2, 6};

// Basic sorting
std::sort(data.begin(), data.end());

// Custom comparison
std::sort(data.begin(), data.end(), std::greater<int>());

// Partial sorting
std::partial_sort(data.begin(), data.begin() + 3, data.end());

// Min/Max operations
auto [min_it, max_it] = std::minmax_element(data.begin(), data.end());
```

### Parallel Algorithms (C++17)

```cpp
#include <execution>
#include <algorithm>

std::vector<int> large_data(1000000);
std::iota(large_data.begin(), large_data.end(), 1);

// Sequential execution (default)
std::sort(large_data.begin(), large_data.end());

// Parallel execution
std::sort(std::execution::par, large_data.begin(), large_data.end());

// Parallel unsequenced execution (vectorization allowed)
std::sort(std::execution::par_unseq, large_data.begin(), large_data.end());
```

### Ranges (C++20 Preview)

Traditional algorithms require iterator pairs, which can be verbose and error-prone. Ranges provide a more natural interface:

```cpp
// Traditional approach
std::vector<int> data = {1, 2, 3, 4, 5};
std::vector<int> result;
std::transform(data.begin(), data.end(), 
               std::back_inserter(result),
               [](int x) { return x * x; });

// Ranges approach (C++20+)
// auto result = data | std::views::transform([](int x) { return x * x; })
//                   | std::ranges::to<std::vector>();
```

---

## Special Member Functions and RAII

### The Rule of Three/Five/Zero

#### Rule of Three (C++98)
If a class defines any of these, it should probably define all three:
1. **Copy Constructor**
2. **Copy Assignment Operator**
3. **Destructor**

#### Rule of Five (C++11)
Adds move semantics:
4. **Move Constructor**
5. **Move Assignment Operator**

#### Rule of Zero
Prefer using smart pointers and standard containers to avoid manual resource management.

### Implementation Example

```cpp
#include <memory>
#include <iostream>

class ResourceManager {
private:
    std::unique_ptr<int[]> data;
    size_t size;

public:
    // Constructor
    explicit ResourceManager(size_t s) 
        : size(s), data(std::make_unique<int[]>(s)) {
        std::cout << "Constructor called\n";
    }

    // Copy constructor
    ResourceManager(const ResourceManager& other) 
        : size(other.size), data(std::make_unique<int[]>(other.size)) {
        std::copy(other.data.get(), other.data.get() + size, data.get());
        std::cout << "Copy constructor called\n";
    }

    // Copy assignment operator
    ResourceManager& operator=(const ResourceManager& other) {
        if (this != &other) {
            size = other.size;
            data = std::make_unique<int[]>(size);
            std::copy(other.data.get(), other.data.get() + size, data.get());
        }
        std::cout << "Copy assignment called\n";
        return *this;
    }

    // Move constructor
    ResourceManager(ResourceManager&& other) noexcept 
        : size(other.size), data(std::move(other.data)) {
        other.size = 0;
        std::cout << "Move constructor called\n";
    }

    // Move assignment operator
    ResourceManager& operator=(ResourceManager&& other) noexcept {
        if (this != &other) {
            size = other.size;
            data = std::move(other.data);
            other.size = 0;
        }
        std::cout << "Move assignment called\n";
        return *this;
    }

    // Destructor (automatically handled by unique_ptr)
    ~ResourceManager() = default;

    // Access methods
    int& operator[](size_t index) { return data[index]; }
    const int& operator[](size_t index) const { return data[index]; }
    size_t getSize() const { return size; }
};
```

### RAII (Resource Acquisition Is Initialization)

RAII ensures that resources are properly managed through object lifetime:

```cpp
class FileManager {
private:
    std::unique_ptr<std::FILE, decltype(&std::fclose)> file;

public:
    explicit FileManager(const char* filename) 
        : file(std::fopen(filename, "r"), &std::fclose) {
        if (!file) {
            throw std::runtime_error("Failed to open file");
        }
    }

    // File automatically closed when object is destroyed
    // No explicit destructor needed!

    std::FILE* get() { return file.get(); }
};

// Usage
void processFile() {
    FileManager fm("data.txt");
    // Use fm.get() to access file
    // File automatically closed when fm goes out of scope
}
```

---

## Templates and Generic Programming

### Function Templates

#### Basic Function Template
```cpp
template<typename T>
T max(T a, T b) {
    return (a > b) ? a : b;
}

// C++20 abbreviated syntax
auto max_modern(auto a, auto b) {
    return (a > b) ? a : b;
}

// Usage
int i = max(3, 7);           // T = int
double d = max(3.14, 2.71);  // T = double
```

#### Template with Multiple Parameters
```cpp
template<typename T, typename U>
auto add(T a, U b) -> decltype(a + b) {
    return a + b;
}

// C++14 simplified return type deduction
template<typename T, typename U>
auto multiply(T a, U b) {
    return a * b;
}
```

### Class Templates

#### Basic Class Template
```cpp
template<typename T, size_t N>
class Array {
private:
    T data[N];

public:
    constexpr size_t size() const { return N; }
    
    T& operator[](size_t index) { return data[index]; }
    const T& operator[](size_t index) const { return data[index]; }
    
    T* begin() { return data; }
    T* end() { return data + N; }
    const T* begin() const { return data; }
    const T* end() const { return data + N; }
};

// Usage
Array<int, 5> intArray;
Array<double, 10> doubleArray;
```

### Template Specialization

#### Full Specialization
```cpp
template<typename T>
class Storage {
    T data;
public:
    void set(const T& value) { data = value; }
    T get() const { return data; }
};

// Specialization for bool
template<>
class Storage<bool> {
    unsigned char data : 1;  // Bit field for space efficiency
public:
    void set(bool value) { data = value ? 1 : 0; }
    bool get() const { return data != 0; }
};
```

#### Partial Specialization
```cpp
template<typename T, typename U>
class Pair {
    T first;
    U second;
public:
    Pair(const T& f, const U& s) : first(f), second(s) {}
    T getFirst() const { return first; }
    U getSecond() const { return second; }
};

// Partial specialization for same types
template<typename T>
class Pair<T, T> {
    T first, second;
public:
    Pair(const T& f, const T& s) : first(f), second(s) {}
    T getFirst() const { return first; }
    T getSecond() const { return second; }
    bool areEqual() const { return first == second; }
};
```

### Concepts (C++20)

Concepts provide type constraints for templates:

```cpp
#include <concepts>

template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

template<Numeric T>
T add(T a, T b) {
    return a + b;
}

// Custom concept
template<typename T>
concept Printable = requires(T t) {
    std::cout << t;
};

template<Printable T>
void print(const T& value) {
    std::cout << value << std::endl;
}
```

---

## Matrix Library Implementation

### Basic Matrix Class

```cpp
#include <array>
#include <initializer_list>
#include <iostream>
#include <iomanip>

template<typename T, int Rows, int Cols = Rows>
class Matrix {
private:
    std::array<std::array<T, Cols>, Rows> data;

public:
    // Default constructor
    Matrix() : data{} {}

    // Initializer list constructor
    Matrix(std::initializer_list<std::initializer_list<T>> init) {
        auto row_iter = data.begin();
        for (const auto& row : init) {
            std::copy(row.begin(), row.end(), row_iter->begin());
            ++row_iter;
        }
    }

    // Element access
    T& operator()(int row, int col) {
        return data[row][col];
    }

    const T& operator()(int row, int col) const {
        return data[row][col];
    }

    // Dimensions
    static constexpr int rows() { return Rows; }
    static constexpr int cols() { return Cols; }

    // Matrix addition
    Matrix operator+(const Matrix& other) const {
        Matrix result;
        for (int i = 0; i < Rows; ++i) {
            for (int j = 0; j < Cols; ++j) {
                result(i, j) = data[i][j] + other(i, j);
            }
        }
        return result;
    }

    // Matrix addition assignment
    Matrix& operator+=(const Matrix& other) {
        for (int i = 0; i < Rows; ++i) {
            for (int j = 0; j < Cols; ++j) {
                data[i][j] += other(i, j);
            }
        }
        return *this;
    }

    // Stream output
    friend std::ostream& operator<<(std::ostream& os, const Matrix& m) {
        os << "[\n";
        for (int i = 0; i < Rows; ++i) {
            os << "  ";
            for (int j = 0; j < Cols; ++j) {
                os << std::setw(8) << m(i, j);
            }
            os << "\n";
        }
        os << "]";
        return os;
    }
};
```

### Matrix Multiplication

```cpp
template<typename T, int A, int B, int C>
Matrix<T, A, C> operator*(const Matrix<T, A, B>& left, 
                         const Matrix<T, B, C>& right) {
    Matrix<T, A, C> result;
    for (int i = 0; i < A; ++i) {
        for (int j = 0; j < C; ++j) {
            T sum = T{};
            for (int k = 0; k < B; ++k) {
                sum += left(i, k) * right(k, j);
            }
            result(i, j) = sum;
        }
    }
    return result;
}
```

### Determinant Implementation

```cpp
template<typename T, int N>
class Matrix<T, N, N> {  // Specialization for square matrices
    // ... previous members ...

public:
    // Minor matrix (for determinant calculation)
    Matrix<T, N-1, N-1> minor(int skip_row, int skip_col) const {
        Matrix<T, N-1, N-1> result;
        int result_row = 0;
        
        for (int i = 0; i < N; ++i) {
            if (i == skip_row) continue;
            
            int result_col = 0;
            for (int j = 0; j < N; ++j) {
                if (j == skip_col) continue;
                
                result(result_row, result_col) = data[i][j];
                ++result_col;
            }
            ++result_row;
        }
        return result;
    }

    // Determinant calculation
    T determinant() const {
        if constexpr (N == 1) {
            return data[0][0];
        } else if constexpr (N == 2) {
            return data[0][0] * data[1][1] - data[0][1] * data[1][0];
        } else {
            T det = T{};
            for (int j = 0; j < N; ++j) {
                T cofactor = data[0][j] * minor(0, j).determinant();
                det += (j % 2 == 0) ? cofactor : -cofactor;
            }
            return det;
        }
    }
};
```

---

## Code Examples and Exercises

### Exercise 1: Enhanced Function Objects

Create a configurable function object for mathematical operations:

```cpp
#include <functional>
#include <iostream>
#include <vector>
#include <algorithm>

template<typename T>
class MathOperation {
private:
    std::function<T(T, T)> operation;
    T initial_value;

public:
    MathOperation(std::function<T(T, T)> op, T init) 
        : operation(op), initial_value(init) {}

    T operator()(const std::vector<T>& values) {
        return std::accumulate(values.begin(), values.end(), 
                              initial_value, operation);
    }
};

// Usage example
void demonstrate_math_operations() {
    std::vector<int> data = {1, 2, 3, 4, 5};

    // Sum operation
    MathOperation<int> sum(std::plus<int>(), 0);
    std::cout << "Sum: " << sum(data) << std::endl;

    // Product operation
    MathOperation<int> product(std::multiplies<int>(), 1);
    std::cout << "Product: " << product(data) << std::endl;

    // Max operation
    MathOperation<int> maximum([](int a, int b) { return std::max(a, b); }, 
                              std::numeric_limits<int>::min());
    std::cout << "Maximum: " << maximum(data) << std::endl;
}
```

### Exercise 2: Template Metaprogramming

Compile-time factorial calculation:

```cpp
template<int N>
struct Factorial {
    static constexpr int value = N * Factorial<N-1>::value;
};

template<>
struct Factorial<0> {
    static constexpr int value = 1;
};

// C++14+ variable template
template<int N>
constexpr int factorial_v = Factorial<N>::value;

// Usage
constexpr int fact5 = factorial_v<5>;  // Computed at compile time
static_assert(fact5 == 120);
```

### Exercise 3: Advanced Lambda Usage

```cpp
#include <vector>
#include <algorithm>
#include <functional>

// Lambda factory
auto make_multiplier(int factor) {
    return [factor](int value) { return value * factor; };
}

// Higher-order function
template<typename Container, typename Predicate>
auto filter(const Container& container, Predicate pred) {
    Container result;
    std::copy_if(container.begin(), container.end(),
                 std::back_inserter(result), pred);
    return result;
}

// Composition example
void demonstrate_functional_style() {
    std::vector<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // Filter even numbers
    auto evens = filter(numbers, [](int x) { return x % 2 == 0; });
    
    // Transform with multiplier
    auto doubler = make_multiplier(2);
    std::transform(evens.begin(), evens.end(), evens.begin(), doubler);
    
    // Print results
    std::for_each(evens.begin(), evens.end(), 
                  [](int x) { std::cout << x << " "; });
}
```

### Common Pitfalls and Solutions

#### 1. Template Argument Deduction Issues
```cpp
// Problem: Ambiguous template instantiation
template<typename T>
void func(T value) { /* ... */ }

// func(0);  // Could be int, long, etc.

// Solution: Explicit instantiation or SFINAE
template<typename T>
std::enable_if_t<std::is_integral_v<T>, void> 
func(T value) { /* Only for integral types */ }
```

#### 2. Lambda Capture Gotchas
```cpp
// Dangerous: Capturing by reference
std::function<int()> make_counter_bad() {
    int count = 0;
    return [&count]() { return ++count; };  // Dangling reference!
}

// Safe: Capturing by value or using shared state
std::function<int()> make_counter_good() {
    auto count = std::make_shared<int>(0);
    return [count]() { return ++(*count); };
}
```

### Performance Considerations

#### 1. Template Instantiation Bloat
```cpp
// Avoid excessive template instantiations
template<typename T>
class Vector {
    // Implementation details
};

// Better: Use type erasure for common interface
class VectorBase {
public:
    virtual ~VectorBase() = default;
    virtual size_t size() const = 0;
    // Other non-template operations
};

template<typename T>
class Vector : public VectorBase {
    // Type-specific implementation
};
```

#### 2. Lambda vs Function Objects Performance
```cpp
// Generally equivalent performance after optimization
auto lambda = [](int x) { return x * x; };

struct Functor {
    int operator()(int x) const { return x * x; }
};

// Both typically inline to the same assembly code
```

---

## Best Practices Summary

1. **Use RAII**: Let destructors handle resource cleanup
2. **Prefer smart pointers**: Avoid manual memory management
3. **Use concepts**: Constrain template parameters meaningfully
4. **Leverage standard algorithms**: Don't reinvent the wheel
5. **Be mindful of template instantiation costs**: Use explicit instantiation when appropriate
6. **Use lambdas for local operations**: Keep function objects for reusable, stateful operations
7. **Follow the Rule of Zero**: Let the compiler generate special members when possible
8. **Use `constexpr` and `consteval`**: Enable compile-time computation
9. **Prefer composition over inheritance**: Especially for template designs
10. **Test with multiple types**: Ensure template code works with intended type categories
