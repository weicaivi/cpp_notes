# Lecture 17 Summary

## Part 1: Killing the Preprocessor

### The Problems with Preprocessor

The C++ preprocessor is a text-based macro system that causes numerous issues in modern development:

#### 1. **Namespace Ignorance**
Macros operate at the text level before compilation, completely bypassing C++'s namespace system.

```cpp
// Problem: Some GNU headers define 'minor' as a macro
#define minor(x) ((x) & 0xFF)  // System macro

namespace matrix {
    class Matrix {
        int minor(int i, int j);  // Error: macro interferes
    };
}

// Solution: Must use #undef
#undef minor
```

#### 2. **Side Effects and Multiple Evaluation**
Macros can evaluate arguments multiple times, leading to unexpected behavior:

```cpp
// Dangerous macro
#define min(x, y) x < y ? x : y

// Problem demonstration
int i = 3, j = 5;
cout << min(i++, j);  // Expands to: i++ < j ? i++ : j
                      // Result: 4 (not 3!), and i incremented twice

// Safe alternative with templates
template<typename T>
constexpr const T& min(const T& x, const T& y) {
    return x < y ? x : y;
}
```

#### 3. **IDE Refactoring Impediment**
Preprocessor macros prevent automated refactoring tools from working effectively because they can't predict macro expansions.

#### 4. **Debugging Difficulties**
Macros expand to single lines, making step-by-step debugging impossible.

#### 5. **Type Safety Absence**
Function-like macros accept any type without type checking:

```cpp
// Macro - no type safety
#define SQUARE(x) ((x) * (x))
SQUARE("hello");  // Compiles but nonsensical

// Template - type safe
template<typename T>
constexpr T square(T x) requires std::is_arithmetic_v<T> {
    return x * x;
}
square("hello");  // Compilation error with clear message
```

#### 6. **Compilation Time Issues**
Header files can expand to tens of thousands of lines due to nested includes, significantly increasing compilation time.

### Modern C++ Replacements for Preprocessor

#### **Templates Instead of Macros**
```cpp
// Old macro approach
#define MAX(a, b) ((a) > (b) ? (a) : (b))

// Modern template approach
template<typename T>
constexpr const T& max(const T& a, const T& b) {
    return a > b ? a : b;
}

// C++20 abbreviated template
constexpr auto max(const auto& a, const auto& b) 
    requires std::same_as<decltype(a), decltype(b)> {
    return a > b ? a : b;
}
```

#### **constexpr Variables for Constants**
```cpp
// Old way
#define PI 3.14159265359
#define BUFFER_SIZE 1024

// Modern way
inline constexpr double PI = 3.14159265359;
inline constexpr size_t BUFFER_SIZE = 1024;

// Can be namespaced
namespace physics {
    inline constexpr double SPEED_OF_LIGHT = 299792458.0;  // m/s
}
```

#### **if constexpr for Conditional Compilation**
```cpp
// Old preprocessor approach
#ifdef USE_FAST_ALGORITHM
    fast_implementation();
#else
    safe_implementation();
#endif

// Modern C++17 approach
template<bool UseFast>
void algorithm() {
    if constexpr (UseFast) {
        fast_implementation();
    } else {
        safe_implementation();
    }
}
```

#### **std::source_location for Debug Info**
```cpp
// Old macro way
#define LOG(msg) fprintf(stderr, "%s:%d: %s\n", __FILE__, __LINE__, msg)

// Modern C++20 way
#include <source_location>

void log(std::string_view message, 
         std::source_location loc = std::source_location::current()) {
    std::cerr << loc.file_name() << ':' 
              << loc.line() << ':' 
              << loc.function_name() << ": " 
              << message << '\n';
}
```

#### **std::numeric_limits Instead of Limit Macros**
```cpp
// Old way
#include <climits>
int max_int = INT_MAX;

// Modern way
#include <limits>
constexpr int max_int = std::numeric_limits<int>::max();

// Works in templates
template<typename T>
void print_range() {
    std::cout << "Range: [" << std::numeric_limits<T>::min() 
              << ", " << std::numeric_limits<T>::max() << "]\n";
}
```

### Feature Test Macros (Still Necessary)

While we aim to eliminate most macros, feature test macros remain essential:

```cpp
// Check C++ version
#if __cplusplus >= 202002L
    // C++20 code
#endif

// Check for specific features
#ifdef __cpp_concepts
    template<std::integral T>
    T gcd(T a, T b);
#else
    template<typename T, 
             typename = std::enable_if_t<std::is_integral_v<T>>>
    T gcd(T a, T b);
#endif
```

## Part 2: Deep Dive into C++20 Concepts

### Understanding Concepts

Concepts are compile-time predicates on template parameters that make generic programming safer and more expressive.

#### Basic Concept Usage

```cpp
// Simple concept constraint
template<std::integral T>
T factorial(T n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}

// Terse notation with auto
auto add(std::integral auto a, std::integral auto b) {
    return a + b;
}

// Multiple constraints with requires clause
template<typename T, typename U>
requires std::integral<T> && std::floating_point<U>
auto mixed_operation(T integer, U floating) {
    return integer * floating;
}
```

### The optimized_copy Case Study

Let's examine both SFINAE and Concepts approaches to implementing an optimized copy function:

#### SFINAE Approach (optimized_copy_SFINAE.h)

The SFINAE version has several issues:

1. **Complex type trait for contiguous iterators** - The custom `is_contiguous_iterator` trait is verbose and error-prone
2. **Difficult to read enable_if conditions**
3. **Template parameter ordering issues**

Here's a corrected and improved version:

```cpp
#ifndef OPTIMIZED_COPY_SFINAE_H
#define OPTIMIZED_COPY_SFINAE_H

#include <type_traits>
#include <iterator>
#include <cstring>

namespace mpcs {
    // Helper trait to detect contiguous iterators
    template<typename It, typename = void>
    struct is_contiguous_iterator : std::false_type {};
    
    // Specialization for pointers (always contiguous)
    template<typename T>
    struct is_contiguous_iterator<T*, void> : std::true_type {};
    
    // Specialization for iterators with iterator_concept
    template<typename It>
    struct is_contiguous_iterator<It, 
        std::void_t<typename std::iterator_traits<It>::iterator_concept>>
        : std::is_same<typename std::iterator_traits<It>::iterator_concept,
                       std::contiguous_iterator_tag> {};
    
    template<typename It>
    inline constexpr bool is_contiguous_iterator_v = is_contiguous_iterator<It>::value;
    
    // Generic version for non-optimizable cases
    template<typename InputIt, typename OutputIt>
    std::enable_if_t<!is_contiguous_iterator_v<InputIt> ||
                     !is_contiguous_iterator_v<OutputIt> ||
                     !std::is_trivially_copyable_v<
                         typename std::iterator_traits<OutputIt>::value_type>,
                     OutputIt>
    optimized_copy(InputIt first, InputIt last, OutputIt out) {
        while (first != last) {
            *out++ = *first++;
        }
        return out;
    }
    
    // Optimized version using memcpy
    template<typename InputIt, typename OutputIt>
    std::enable_if_t<is_contiguous_iterator_v<InputIt> &&
                     is_contiguous_iterator_v<OutputIt> &&
                     std::is_trivially_copyable_v<
                         typename std::iterator_traits<OutputIt>::value_type>,
                     OutputIt>
    optimized_copy(InputIt first, InputIt last, OutputIt out) {
        using value_type = typename std::iterator_traits<OutputIt>::value_type;
        if (first != last) {
            std::memcpy(&*out, &*first, (last - first) * sizeof(value_type));
        }
        return out + (last - first);
    }
}

#endif
```

#### Concepts Approach (optimized_copy.h)

The Concepts version is much cleaner:

```cpp
#ifndef OPTIMIZED_COPY_H
#define OPTIMIZED_COPY_H

#include <type_traits>
#include <iterator>
#include <concepts>
#include <cstring>

namespace mpcs {
    // Helper alias template
    template<typename T>
    using iter_value_t = std::iter_value_t<T>;
    
    // Generic version - default case
    template<typename InputIt, typename OutputIt>
    OutputIt optimized_copy(InputIt first, InputIt last, OutputIt out) {
        while (first != last) {
            *out++ = *first++;
        }
        return out;
    }
    
    // Optimized version with clear constraints
    template<std::contiguous_iterator InputIt, std::contiguous_iterator OutputIt>
    requires std::is_trivially_copyable_v<iter_value_t<OutputIt>> &&
             std::same_as<std::remove_const_t<iter_value_t<InputIt>>, 
                         iter_value_t<OutputIt>>
    OutputIt optimized_copy(InputIt first, InputIt last, OutputIt out) {
        if (first != last) {
            std::memcpy(&*out, &*first, 
                       (last - first) * sizeof(iter_value_t<OutputIt>));
        }
        return out + (last - first);
    }
}

#endif
```

### Advanced Concept Techniques

#### Creating Custom Concepts

```cpp
// Concept for types with a size() method returning integral
template<typename T>
concept Sizeable = requires(T t) {
    { t.size() } -> std::convertible_to<std::size_t>;
};

// Concept for containers with begin/end
template<typename T>
concept Container = requires(T t) {
    { t.begin() } -> std::input_or_output_iterator;
    { t.end() } -> std::sentinel_for<decltype(t.begin())>;
};

// Compound concept
template<typename T>
concept SizeableContainer = Container<T> && Sizeable<T>;

// Usage
template<SizeableContainer C>
void process(const C& container) {
    std::cout << "Processing " << container.size() << " elements\n";
    for (const auto& elem : container) {
        // Process element
    }
}
```

#### Concept Subsumption and Overloading

```cpp
// Base concept
template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

// More refined concept (subsumes Numeric when T is integral)
template<typename T>
concept SignedNumeric = Numeric<T> && std::is_signed_v<T>;

// Even more refined
template<typename T>
concept SignedIntegral = std::integral<T> && std::is_signed_v<T>;

// Overloading with subsumption
template<Numeric T>
void process(T value) {
    std::cout << "Processing numeric: " << value << '\n';
}

template<SignedIntegral T>  // More specific, will be preferred
void process(T value) {
    std::cout << "Processing signed integral: " << value << '\n';
}

// Usage
process(42);      // Calls SignedIntegral version
process(42u);     // Calls Numeric version
process(3.14);    // Calls Numeric version
```

### Four Notations for Concept Constraints

```cpp
// 1. Requires clause after template header
template<typename T>
requires std::copyable<T>
T clone(const T& original) { return original; }

// 2. Trailing requires clause
template<typename T>
T clone(const T& original) requires std::copyable<T> { 
    return original; 
}

// 3. Constrained template parameter
template<std::copyable T>
T clone(const T& original) { return original; }

// 4. Terse notation with constrained auto
auto clone(const std::copyable auto& original) { 
    return original; 
}
```

### Best Practices and Common Patterns

#### GCD Example - Evolution from Unconstrained to Concepts

```cpp
// Problem: Unconstrained template
template<typename T>
T gcd_bad(T a, T b) {
    if (b == 0) return a;
    return gcd_bad(b, a - b * (a / b));  // Crashes with floating point!
}

// SFINAE solution (pre-C++20)
template<typename T, 
         typename = std::enable_if_t<std::is_integral_v<T>>>
T gcd_sfinae(T a, T b) {
    if (b == 0) return a;
    return gcd_sfinae(b, a - b * (a / b));
}

// Concepts solution (C++20)
template<std::integral T>
T gcd(T a, T b) {
    if (b == 0) return a;
    return gcd(b, a - b * (a / b));
}

// Or with terse notation
std::integral auto gcd(std::integral auto a, 
                       std::integral auto b) {
    if (b == 0) return a;
    return gcd(b, a - b * (a / b));
}
```

## Key Takeaways

1. **Preprocessor macros should be avoided** in favor of modern C++ features like templates, constexpr, and concepts
2. **Concepts provide type-safe, readable constraints** that are far superior to SFINAE
3. **Feature test macros remain necessary** for conditional compilation based on compiler support
4. **Concept subsumption enables proper overloading** based on constraint specificity
5. **Multiple notation styles for concepts** provide flexibility for different use cases

## Additional Examples and Patterns

### Complex Requires Expressions

```cpp
template<typename T>
concept Printable = requires(T t, std::ostream& os) {
    { os << t } -> std::same_as<std::ostream&>;
};

template<typename T>
concept Arithmetic = requires(T a, T b) {
    { a + b } -> std::convertible_to<T>;
    { a - b } -> std::convertible_to<T>;
    { a * b } -> std::convertible_to<T>;
    { a / b } -> std::convertible_to<T>;
};

template<typename T>
concept Comparable = requires(T a, T b) {
    { a < b } -> std::convertible_to<bool>;
    { a <= b } -> std::convertible_to<bool>;
    { a > b } -> std::convertible_to<bool>;
    { a >= b } -> std::convertible_to<bool>;
    { a == b } -> std::convertible_to<bool>;
    { a != b } -> std::convertible_to<bool>;
};
```

### Module System (C++20)

While still being adopted, modules will eventually replace the preprocessor-based include system:

```cpp
// math.cppm (module interface)
export module math;

export template<std::integral T>
T factorial(T n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}

// main.cpp
import math;

int main() {
    auto result = factorial(5);  // No #include needed!
}
```

This approach eliminates header duplication, speeds compilation, and prevents symbol leakage between translation units.