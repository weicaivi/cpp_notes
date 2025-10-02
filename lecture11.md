# Lecture 11 Summary

## Fundamental Types Deep Dive

### Philosophy: Hardware-Agnostic Design

C++ fundamental types are **implementation-defined** rather than fixed-size to optimize for different hardware architectures. This design principle reflects C++'s philosophy of "zero-overhead abstraction" - you shouldn't pay for what you don't use. C++'s approach allows optimal performance on diverse platforms from embedded microcontrollers to high-performance computing clusters, while Java's fixed sizes may be suboptimal on certain architectures.

```cpp
// Java approach: Fixed sizes everywhere
byte    // Always 8 bits
short   // Always 16 bits
int     // Always 32 bits
long    // Always 64 bits

// C++ approach: Flexible to hardware
short   // At least 16 bits
int     // At least 16 bits (typically matches processor word size)
long    // At least 32 bits
long long // At least 64 bits
```

---

## Behavior Specification Categories

### 1. Undefined Behavior (UB)
**Definition**: The program can do anything, including "halt and catch fire."

```cpp
// Classic UB examples
int arr[5];
int x = arr[10];        // Buffer overflow - undefined behavior
int* p = nullptr;
*p = 42;               // Null pointer dereference - undefined behavior

int i = 0;
int result = ++i + i++; // Sequence point violation - undefined behavior
```

**Modern C++ Improvements**: Newer features reduce UB:
- `std::variant` is safer than C unions
- Smart pointers reduce raw pointer UB
- Ranges reduce iterator UB

### 2. Unspecified Behavior
**Definition**: Implementation has choices, but all are valid.

```cpp
// Compile-time vs runtime calculation
double result = 0.1 + 0.2;  // May be computed at compile-time or runtime

// This comparison might fail due to different precision!
bool equal = (0.1 + 0.2) == (0.1 + 0.2);  // Unspecified!

// Function argument evaluation order
f(++i, ++i);  // Unspecified which ++i executes first
```

### 3. Implementation-Defined Behavior
**Definition**: Implementation must document its choice.

```cpp
#include <iostream>
#include <limits>

// Check your platform's choices
std::cout << "char is " << (std::numeric_limits<char>::is_signed ? "signed" : "unsigned") << std::endl;
std::cout << "int size: " << sizeof(int) << " bytes" << std::endl;
std::cout << "pointer size: " << sizeof(void*) << " bytes" << std::endl;
```

**Best Practice**: Prefer implementation-defined over unspecified, and avoid undefined behavior entirely.

---

## Integer Types and Hardware Compatibility

### Standard Integer Types
```cpp
// Basic types with minimum guarantees
short       // ≥ 16 bits
int         // ≥ 16 bits (usually processor word size)
long        // ≥ 32 bits  
long long   // ≥ 64 bits

// Each has unsigned variant
unsigned short, unsigned int, unsigned long, unsigned long long
```

### Common Data Models in Practice
```cpp
// Platform-specific sizes (in bits)
//           LP32  ILP32  LLP32  LLP64
// short      16     16     16     16
// int        16     32     32     32
// long       32     32     32     64
// long long  64     64     64     64
// pointer    32     32     32     64

// Modern platforms typically use LLP64 (Windows) or LP64 (Unix)
```

### Fixed-Width Integer Types (C++11)
```cpp
#include <cstdint>

// Exact width (may not be available on all platforms)
int8_t, uint8_t      // Exactly 8 bits
int16_t, uint16_t    // Exactly 16 bits  
int32_t, uint32_t    // Exactly 32 bits
int64_t, uint64_t    // Exactly 64 bits

// Minimum width (always available)
int_least8_t, uint_least8_t    // At least 8 bits
int_least16_t, uint_least16_t  // At least 16 bits
int_least32_t, uint_least32_t  // At least 32 bits
int_least64_t, uint_least64_t  // At least 64 bits

// Fastest with at least specified width
int_fast8_t, uint_fast8_t      // Fastest with ≥ 8 bits
int_fast16_t, uint_fast16_t    // Fastest with ≥ 16 bits
int_fast32_t, uint_fast32_t    // Fastest with ≥ 32 bits  
int_fast64_t, uint_fast64_t    // Fastest with ≥ 64 bits
```

### Special Purpose Integer Types
```cpp
#include <cstddef>
#include <cstdint>

size_t     // Unsigned, can represent size of any object
ptrdiff_t  // Signed, difference between pointers
intptr_t   // Can hold a pointer value
uintptr_t  // Unsigned version
intmax_t   // Largest supported signed integer
uintmax_t  // Largest supported unsigned integer
```

**Best Practice Example**:
```cpp
#include <vector>
#include <cstdint>

class Buffer {
    std::vector<uint8_t> data_;  // Exact byte storage
    
public:
    void resize(size_t new_size) {      // size_t for sizes
        data_.resize(new_size);
    }
    
    ptrdiff_t distance(const uint8_t* a, const uint8_t* b) {
        return b - a;  // ptrdiff_t for pointer arithmetic
    }
    
    uint_fast32_t checksum() const {    // Fast type for computations
        uint_fast32_t sum = 0;
        for (auto byte : data_) {
            sum += byte;
        }
        return sum;
    }
};
```

---

## Floating Point and Character Types

### Floating Point Types
```cpp
float       // Usually IEEE-754 32-bit (7 decimal digits precision)
double      // Usually IEEE-754 64-bit (15 decimal digits precision) 
long double // Implementation-defined (often 80-bit x87 on x86/x64)

// Literals
auto f = 3.14f;        // float literal
auto d = 3.14;         // double literal (default)
auto ld = 3.14L;       // long double literal
```

**Critical Warning**: Never compare floating-point for equality!
```cpp
// WRONG
if (0.1 + 0.2 == 0.3) { /* May be false! */ }

// CORRECT
#include <cmath>
bool nearly_equal(double a, double b, double epsilon = 1e-9) {
    return std::abs(a - b) < epsilon;
}

if (nearly_equal(0.1 + 0.2, 0.3)) { /* Reliable */ }
```

### Character Types Complexity

#### Basic char
```cpp
char c = 'A';  // Implementation-defined signedness!

// Check your platform
#include <limits>
bool is_signed = std::numeric_limits<char>::is_signed;
// Usually signed on x86, unsigned on ARM
```

#### Explicit Signedness
```cpp
signed char sc = -1;    // Always signed
unsigned char uc = 255; // Always unsigned

// char, signed char, and unsigned char are THREE DISTINCT TYPES!
void f(char);           // Different function
void f(signed char);    // Different function  
void f(unsigned char);  // Different function
```

#### Unicode Character Types
```cpp
char8_t   // UTF-8 code unit (C++20)
char16_t  // UTF-16 code unit  
char32_t  // UTF-32 code unit
wchar_t   // Wide character (avoid - size varies by platform!)

// Modern string literals
auto u8str = u8"Hello";    // UTF-8 string
auto u16str = u"Hello";    // UTF-16 string
auto u32str = U"Hello";    // UTF-32 string
```

**Best Practice**: Use `char` for ASCII and byte operations, `char32_t` for Unicode code points.

---

## Advanced Variadics

### Variadic Template Fundamentals
```cpp
// Traditional approach - verbose recursion
template<typename T>
T adder(T v) {
    return v;  // Base case
}

template<typename T, typename... Args>
T adder(T first, Args... args) {
    return first + adder(args...);  // Recursive case
}

// Usage
auto sum = adder(1, 2, 3, 4);  // Returns 10
auto str = adder("Hello"s, " "s, "World"s);  // Returns "Hello World"
```

### Pack Expansion Patterns
```cpp
// Basic expansion
template<typename... Args>
void print_args(Args... args) {
    // Each element in args gets printed
    ((std::cout << args << " "), ...);  // C++17 fold expression
    std::cout << std::endl;
}

// Expansion with expressions
template<typename... Args>
auto multiply_by_two(Args... args) {
    return std::make_tuple((args * 2)...);  // Each arg multiplied by 2
}

// Parallel expansion (packs must be same length)
template<typename... Ts, typename... Us>
auto add_pairs(std::tuple<Ts...> t1, std::tuple<Us...> t2) {
    return std::make_tuple((std::get<Ts>(t1) + std::get<Us>(t2))...);
}
```

### C++17 Fold Expressions
```cpp
// Left fold: (((1 + 2) + 3) + 4)
template<typename... Args>
auto sum_left(Args... args) {
    return (... + args);  // Left fold
}

// Right fold: (1 + (2 + (3 + 4)))
template<typename... Args>
auto sum_right(Args... args) {
    return (args + ...);  // Right fold
}

// Fold with initial value
template<typename... Args>
auto sum_with_init(Args... args) {
    return (0 + ... + args);  // Starts with 0
}

// Logical operations
template<typename... Args>
bool all_true(Args... args) {
    return (args && ...);  // true if all args are true
}

template<typename... Args>
bool any_true(Args... args) {
    return (args || ...);  // true if any arg is true
}
```

### Perfect Forwarding with Variadics
```cpp
// Universal reference + variadic perfect forwarding
template<typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

// Usage - constructs object with perfect forwarding
struct Person {
    Person(const std::string& name, int age) 
        : name_(name), age_(age) {}
    std::string name_;
    int age_;
};

auto person = make_unique<Person>("Alice", 30);  // Perfect forwarding
```

---

## Tuple Implementation and Metaprogramming

### Tuple1: Recursive Composition
Has-a: stores a member of type Tuple1<Ts...> inside each node
```cpp
// forward declaration of a Tuple1 template with parameter pack <Ts...>
template<typename ...Ts> 
struct Tuple1; 

// specialization for the empty pack <> (base case)
template<> 
struct Tuple1<> {};

template<typename T, typename ...Ts>
struct Tuple1<T, Ts...> {
    Tuple1(T const &t, Ts const &... ts) 
        : val(t), restOfVals(ts...) {}
    
    T val;
    Tuple1<Ts...> restOfVals;  // Recursive composition
};
```

### Tuple2: Empty Base Optimization
Is-a: derives from Tuple2<Ts...>

- If a stored type is an empty class, the base class subobject can occupy zero bytes (per the standard’s EBO rule)
- Composition can’t normally take advantage of this — an empty member still consumes at least 1 byte
```cpp
template<typename ...Ts> 
struct Tuple2;

template<> 
struct Tuple2<> {};

template<typename T, typename ...Ts>
struct Tuple2<T, Ts...> : public Tuple2<Ts...> {  // Inheritance
    Tuple2(T const &t, Ts const &... ts) 
        : Tuple2<Ts...>(ts...), val(t) {}
    //  recursively constructs the base class part first, passing along all remaining values
    T val;
};
/**
So for Tuple2<int,double,char>:
	- Top level stores an int and inherits from Tuple2<double,char>.
	- That level stores a double and inherits from Tuple2<char>.
	- And so on until the empty specialization.
*/
```

**Memory Layout Comparison**:
```cpp
// Tuple1<int, double> layout:
// [int val][padding][Tuple1<double> restOfVals]
//                   [double val][Tuple1<> restOfVals (empty)]

// Tuple2<int, double> layout (empty base optimization):
// [int val][padding][double val] // No storage for empty Tuple2<>

std::cout << sizeof(Tuple1<int>) << std::endl;  // Typically 8 bytes
std::cout << sizeof(Tuple2<int>) << std::endl;  // Typically 4 bytes
```

### Advanced Getter Implementation
template recursion + base casting to walk the inheritance chain
```cpp
// Getter by index

// declaration of a family of helper structs
// i is the index of the element we want
// Ts... are the types stored in the tuple
// actual logic is provided by partial specializations
template<int i, typename ...Ts>
struct Getter2;

// base case - index 0
template<typename T, typename ...Ts>
struct Getter2<0, T, Ts...> {
    static auto& get(Tuple2<T, Ts...>& tup) {
        return tup.val;  // Found the element at index 0
    }
};

template<int i, typename T, typename ...Ts>
struct Getter2<i, T, Ts...> {
    // Tuple2<T, Ts...> inherits from Tuple2<Ts...>
    // We treat tup as a reference to its base (Tuple2<Ts...>) and recurse with i-1
    // Each step peels off one type and decreases the index until it hits zero
    static auto& get(Tuple2<T, Ts...>& tup) { 
        Tuple2<Ts...>& restOfVals = tup;  // Cast to base
        return Getter2<i-1, Ts...>::get(restOfVals);  // Recurse
    }
};

// Convenience function, User-facing, get<i>(tuple)
// Just forwards to the appropriate specialization of Getter2 to hide the recursion details
template<int i, typename ...Ts>
auto& get(Tuple2<Ts...>& tup) {
    return Getter2<i, Ts...>::get(tup);
}
```

### Getter by Type
```cpp
// Type-based getter implementation
template<typename Target, typename... Ts>
struct TypeGetter;

template<typename Target, typename... Ts>
struct TypeGetter<Target, Target, Ts...> {
    static auto& get(Tuple2<Target, Ts...>& tup) {
        return tup.val;  // Found the target type
    }
};

template<typename Target, typename Head, typename... Ts>
struct TypeGetter<Target, Head, Ts...> {
    static auto& get(Tuple2<Head, Ts...>& tup) {
        Tuple2<Ts...>& rest = tup;
        return TypeGetter<Target, Ts...>::get(rest);
    }
};

// Type-based get function
template<typename Target, typename... Ts>
auto& get(Tuple2<Ts...>& tup) {
    return TypeGetter<Target, Ts...>::get(tup);
}

// Usage example
Tuple2<int, double, std::string> t(42, 3.14, "hello");
auto& str = get<std::string>(t);  // Gets the string element
```

### Compile-Time Metaprogramming Utilities
```cpp
// Length calculation
template<typename T> 
struct Length;

template<>
struct Length<std::tuple<>> {
    static constexpr int value = 0;
};

template<class T, typename... Us>
struct Length<std::tuple<T, Us...>> {
    static constexpr int value = 1 + Length<std::tuple<Us...>>::value;
};

// IndexOf implementation  
template<class List, class Target> 
struct IndexOf;

template<class Target>
struct IndexOf<std::tuple<>, Target> {
    static constexpr int value = -1;  // Not found
};

template<typename Target, typename... Tail>
struct IndexOf<std::tuple<Target, Tail...>, Target> {
    static constexpr int value = 0;  // Found at position 0
};

template<class Head, typename... Tail, class Target>
struct IndexOf<std::tuple<Head, Tail...>, Target> {
private:
    static constexpr int temp = IndexOf<std::tuple<Tail...>, Target>::value;
public:
    static constexpr int value = (temp == -1) ? -1 : 1 + temp;
};

// Modern alias template approach
template<typename T, typename A, typename B>
using Replace_t = typename Replace<T, A, B>::type;
```

---

## IndentStream Case Study

### Custom Stream Buffer Implementation
```cpp
class IndentStreamBuf : public std::streambuf {
private:
    std::streambuf& wrappedStreambuf;
    bool isLineStart = true;
    
public:
    size_t myIndent = 0;
    
    IndentStreamBuf(std::ostream& stream) 
        : wrappedStreambuf(*stream.rdbuf()) {}
    
    virtual int overflow(traits_type::int_type outputVal) override {
        if (outputVal == traits_type::eof()) {
            return traits_type::eof();
        }
        
        if (outputVal == '\n') {
            isLineStart = true;
        } else if (isLineStart) {
            // Insert indentation at start of line
            for (size_t i = 0; i < myIndent; i++) {
                wrappedStreambuf.sputc(' ');
            }
            isLineStart = false;
        }
        
        return wrappedStreambuf.sputc(static_cast<char>(outputVal));
    }
};
```

### Stream Wrapper Class
```cpp
class IndentStream : public std::ostream {
public:
    IndentStream(std::ostream& wrappedStream)
        : std::ostream(new IndentStreamBuf(wrappedStream)) {}
    
    ~IndentStream() { 
        delete this->rdbuf();  // Clean up custom buffer
    }
};
```

### Stream Manipulators
```cpp
namespace mpcs {
    // Safe manipulator with exception handling
    std::ostream& indent(std::ostream& ostr) {
        try {
            IndentStreamBuf& out = dynamic_cast<IndentStreamBuf&>(*ostr.rdbuf());
            out.myIndent += 4;
        } catch (const std::bad_cast&) {
            // Not an IndentStream - ignore or throw
        }
        return ostr;
    }
    
    // Safe manipulator that silently ignores non-IndentStreams
    std::ostream& unindent(std::ostream& ostr) {
        IndentStreamBuf* out = dynamic_cast<IndentStreamBuf*>(ostr.rdbuf());
        if (out && out->myIndent >= 4) {
            out->myIndent -= 4;
        }
        return ostr;
    }
}
```

### Usage Example
```cpp
#include <iostream>
#include "IndentStream.h"

using namespace std;
using namespace mpcs;

int main() {
    IndentStream ins(cout);
    
    ins << "/*\n" 
        << indent << "This function calculates Fibonacci numbers\n"
        << indent << "Implementation details:\n"
        << "- Uses recursion\n"
        << "- Base cases: 0 and 1\n"
        << unindent << unindent << "*/\n\n";
        
    ins << "int fib(int n) {" << indent << endl;
    ins << "if (n == 0) return 0;" << endl;
    ins << "if (n == 1) return 1;" << endl; 
    ins << "return fib(n-1) + fib(n-2);" << unindent << endl;
    ins << "}" << endl;
    
    return 0;
}
```

**Output**:
```
/*
    This function calculates Fibonacci numbers
        Implementation details:
        - Uses recursion
        - Base cases: 0 and 1
*/

int fib(int n) {
    if (n == 0) return 0;
    if (n == 1) return 1;
    return fib(n-1) + fib(n-2);
}
```

---

## Practical Code Examples

### Enhanced Regex Example
```cpp
#include <regex>
#include <iostream>
#include <string>
#include <vector>

class NumberExtractor {
private:
    std::regex decimal_pattern{R"((\d+)\.(\d+))"};
    std::regex integer_pattern{R"(\b(\d+)\b)"};
    
public:
    struct DecimalNumber {
        int integer_part;
        int fractional_part;
        std::string original;
    };
    
    std::vector<DecimalNumber> extract_decimals(const std::string& text) {
        std::vector<DecimalNumber> results;
        std::string search_text = text;
        std::smatch match;
        
        while (std::regex_search(search_text, match, decimal_pattern)) {
            results.push_back({
                std::stoi(match[1].str()),
                std::stoi(match[2].str()), 
                match[0].str()
            });
            search_text = match.suffix();
        }
        return results;
    }
    
    void print_analysis(const std::string& text) {
        auto decimals = extract_decimals(text);
        std::cout << "Found " << decimals.size() << " decimal numbers:\n";
        
        for (const auto& num : decimals) {
            std::cout << "  " << num.original 
                      << " -> integer: " << num.integer_part 
                      << ", fractional: " << num.fractional_part << "\n";
        }
    }
};

int main() {
    NumberExtractor extractor;
    std::string text = "Measurements: 1.23, 4, 5.6, 7.89, and 10.0";
    extractor.print_analysis(text);
    return 0;
}
```

### Modern Format Implementation
```cpp
#include <iostream>
#include <string>
#include <string_view>
#include <sstream>
#include <stdexcept>

// Modern C++20-style format implementation
template<typename T>
std::string to_string_impl(const T& value) {
    if constexpr (std::is_arithmetic_v<T>) {
        return std::to_string(value);
    } else {
        std::ostringstream oss;
        oss << value;
        return oss.str();
    }
}

// Base case - no more arguments
std::string simple_format(std::string_view fmt) {
    if (fmt.find("{}") != std::string_view::npos) {
        throw std::invalid_argument("Too few arguments for format string");
    }
    return std::string(fmt);
}

// Variadic case - process one argument and recurse
template<typename T, typename... Args>
std::string simple_format(std::string_view fmt, const T& value, const Args&... args) {
    auto pos = fmt.find("{}");
    if (pos == std::string_view::npos) {
        throw std::invalid_argument("Too many arguments for format string");
    }
    
    return std::string(fmt.substr(0, pos)) 
         + to_string_impl(value)
         + simple_format(fmt.substr(pos + 2), args...);
}

// C++17 fold expression version (more efficient)
template<typename... Args>
std::string format_fold(std::string_view fmt, const Args&... args) {
    std::string result;
    result.reserve(fmt.size() + (sizeof...(args) * 10)); // Rough estimate
    
    size_t arg_index = 0;
    std::array<std::string, sizeof...(args)> arg_strings = {
        to_string_impl(args)...
    };
    
    for (size_t i = 0; i < fmt.size(); ) {
        if (i + 1 < fmt.size() && fmt[i] == '{' && fmt[i + 1] == '}') {
            if (arg_index >= sizeof...(args)) {
                throw std::invalid_argument("Too few arguments");
            }
            result += arg_strings[arg_index++];
            i += 2;
        } else {
            result += fmt[i++];
        }
    }
    
    if (arg_index < sizeof...(args)) {
        throw std::invalid_argument("Too many arguments");
    }
    
    return result;
}

int main() {
    try {
        std::cout << simple_format("Hello {}, you are {} years old!\n", "Alice", 30);
        std::cout << format_fold("Pi = {}, e = {}\n", 3.14159, 2.71828);
    } catch (const std::exception& e) {
        std::cerr << "Format error: " << e.what() << std::endl;
    }
    return 0;
}
```

### Wind Data Analysis Enhancement
```cpp
#include <regex>
#include <iostream>
#include <fstream>
#include <map>
#include <vector>
#include <algorithm>
#include <numeric>
#include <iomanip>

class HurricaneAnalyzer {
public:
    struct StormData {
        int year;
        std::vector<int> wind_measurements;
        double saffir_simpson_days = 0.0;
    };
    
private:
    std::regex header_regex{R"((\d{4})\d{6})"};
    std::regex data_regex{R"(.{18}[*SEWLD].{7}\s+(\d+).{13}\s+(\d+).{13}\s+(\d+).{13}\s+(\d+).*)"};
    
    int knots_to_saffir_simpson(int knots) const {
        if (knots < 64) return 0;
        if (knots < 83) return 1;
        if (knots < 96) return 2;
        if (knots < 114) return 3;
        if (knots < 136) return 4;
        return 5;
    }
    
public:
    std::map<int, StormData> analyze_file(const std::string& filename) {
        std::map<int, StormData> storm_data;
        std::ifstream file(filename);
        std::string line;
        StormData* current_storm = nullptr;
        
        while (std::getline(file, line)) {
            std::smatch match;
            
            if (std::regex_search(line, match, header_regex)) {
                int year = std::stoi(match[1].str());
                current_storm = &storm_data[year];
                current_storm->year = year;
            } 
            else if (current_storm && std::regex_search(line, match, data_regex)) {
                for (int i = 1; i <= 4; ++i) {
                    int knots = std::stoi(match[i].str());
                    current_storm->wind_measurements.push_back(knots);
                    current_storm->saffir_simpson_days += knots_to_saffir_simpson(knots) / 4.0;
                }
            }
        }
        
        return storm_data;
    }
    
    void generate_report(const std::map<int, StormData>& data, const std::string& output_file) {
        std::ofstream out(output_file);
        out << "Year,Saffir-Simpson Days,Max Wind Speed,Average Wind Speed\n";
        
        for (const auto& [year, storm] : data) {
            int max_wind = storm.wind_measurements.empty() ? 0 :
                          *std::max_element(storm.wind_measurements.begin(), 
                                          storm.wind_measurements.end());
            
            double avg_wind = storm.wind_measurements.empty() ? 0.0 :
                             std::accumulate(storm.wind_measurements.begin(),
                                           storm.wind_measurements.end(), 0.0) / 
                             storm.wind_measurements.size();
            
            out << year << "," << std::fixed << std::setprecision(2) 
                << storm.saffir_simpson_days << "," << max_wind << "," << avg_wind << "\n";
        }
    }
};

int main() {
    HurricaneAnalyzer analyzer;
    auto data = analyzer.analyze_file("hurdat_atlantic_1851-2011.txt");
    analyzer.generate_report(data, "hurricane_analysis.csv");
    
    std::cout << "Analysis complete. Generated hurricane_analysis.csv\n";
    return 0;
}
```