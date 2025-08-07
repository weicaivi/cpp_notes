# C++ for Advanced Programmers - Comprehensive Summary

## Special Class Members

### Static Class Members

Static members belong to the class itself rather than to any specific instance.

#### Static Data Members
```cpp
class CountedObject {
public:
    CountedObject() { ++objectCount; }
    ~CountedObject() { --objectCount; }
    
    static size_t getObjectCount() { return objectCount; }
    static size_t memoryUsed() { 
        return objectCount * sizeof(CountedObject); 
    }
    
private:
    static size_t objectCount;  // Declaration
};

// Definition (required in .cpp file for complex types)
size_t CountedObject::objectCount = 0;
```

**Key Points:**
- Static data members must be defined outside the class for non-integral types
- Accessed using scope resolution operator: `CountedObject::objectCount`
- Shared among all instances of the class

#### Static Member Functions
```cpp
class MathUtils {
public:
    static double square(double x) { return x * x; }
    static constexpr double PI = 3.14159265359;
    
    // Can access static members but not instance members
    static void printPI() {
        std::cout << "PI = " << PI << std::endl;
        // std::cout << instanceVar;  // ERROR: no access to instance members
    }
    
private:
    double instanceVar = 0.0;
};
```

### Constructors and Destructors

#### Constructor Types and Syntax

```cpp
class Example {
private:
    int value;
    std::string name;
    
public:
    // Default constructor
    Example() : value(0), name("default") {}
    
    // Parameterized constructor
    Example(int v, const std::string& n) : value(v), name(n) {}
    
    // Single-parameter constructor (creates implicit conversion)
    Example(int v) : value(v), name("from_int") {}
    
    // Explicit constructor (prevents implicit conversion)
    explicit Example(double d) : value(static_cast<int>(d)), name("from_double") {}
    
    // Copy constructor
    Example(const Example& other) : value(other.value), name(other.name) {}
    
    // Destructor
    ~Example() { 
        std::cout << "Destroying " << name << std::endl; 
    }
};
```

#### Member Initialization vs Assignment

```cpp
class BetterExample {
private:
    const int id;
    std::string& nameRef;
    
public:
    // CORRECT: Use member initializer list
    BetterExample(int i, std::string& ref) : id(i), nameRef(ref) {}
    
    // WRONG: Cannot assign to const or reference members in body
    // BetterExample(int i, std::string& ref) {
    //     id = i;        // ERROR: const member
    //     nameRef = ref; // ERROR: reference member
    // }
};
```

#### Non-Static Data Member Initializers (NSDMI)

```cpp
class ModernClass {
private:
    int defaultValue = 42;           // NSDMI
    std::string defaultName = "unknown";
    std::vector<int> data{1, 2, 3};  // Uniform initialization
    
public:
    // Compiler-generated default constructor uses NSDMI
    ModernClass() = default;
    
    // Custom constructor can override defaults
    ModernClass(int val, const std::string& name) 
        : defaultValue(val), defaultName(name) {}
};
```

#### Special Constructor Rules

```cpp
class SpecialRules {
public:
    // If ANY constructor is defined, compiler won't generate default constructor
    SpecialRules(int x) {}
    
    // Must explicitly request default constructor if needed
    SpecialRules() = default;
    
    // Disable copy constructor
    SpecialRules(const SpecialRules&) = delete;
    
    // Disable assignment operator
    SpecialRules& operator=(const SpecialRules&) = delete;
};
```

#### Aggregate Initialization

```cpp
struct SimpleAggregate {
    int x;
    double y;
    std::string z;
};

// Various initialization forms
SimpleAggregate a1{1, 2.5, "hello"};
SimpleAggregate a2 = {2, 3.7, "world"};
SimpleAggregate a3{.x = 10, .y = 20.0, .z = "designated"};  // C++20 designated initializers
```

---

## Algorithm Performance Analysis

### Median Calculation Performance Comparison

```cpp
#include <algorithm>
#include <vector>
#include <chrono>

namespace MedianCalculation {
    
// Method 1: Full sort
double median_sort(std::vector<double> v) {
    std::sort(v.begin(), v.end());
    return v[v.size() / 2];
}

// Method 2: Partial sort  
double median_partial_sort(std::vector<double> v) {
    std::partial_sort(v.begin(), v.begin() + v.size()/2 + 1, v.end());
    return v[v.size() / 2];
}

// Method 3: nth_element
double median_nth_element(std::vector<double> v) {
    std::nth_element(v.begin(), v.begin() + v.size()/2, v.end());
    return v[v.size() / 2];
}

}
```

#### Performance Results Analysis

**Debug Build Results (Misleading):**
- `nth_element`: ~2817ms
- `partial_sort`: ~2743ms  
- `sort`: ~3859ms

**Release Build Results (Accurate):**
- `nth_element`: ~165ms **FASTEST**
- `sort`: ~1172ms
- `partial_sort`: ~3206ms **SLOWEST**


1. **Always benchmark in release builds** - Debug builds are worthless for performance analysis
2. **Counter-intuitive results** - `partial_sort` was slowest because sorting half of 10M elements is close to sorting all elements
3. **Algorithm complexity matters:**
   - `nth_element`: O(n) average case
   - `sort`: O(n log n)
   - `partial_sort`: O(n log k) where k ≈ n/2

#### Improved Timer Implementation

```cpp
class PrecisionTimer {
private:
    std::chrono::time_point<std::chrono::high_resolution_clock> start_time;
    
public:
    PrecisionTimer() : start_time(std::chrono::high_resolution_clock::now()) {}
    
    ~PrecisionTimer() {
        auto end_time = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end_time - start_time);
        std::cout << duration.count() << " microseconds" << std::endl;
    }
    
    double elapsed_ms() const {
        auto now = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(now - start_time);
        return duration.count();
    }
};
```

---

## Function Argument Passing

### Pass by Value vs Reference

#### Memory Model Understanding

```cpp
void demonstrate_passing() {
    int original = 42;
    
    // Pass by value - creates copy
    auto by_value = [](int x) {
        x = 999;  // Modifies copy only
        std::cout << "Inside by_value: " << x << std::endl;
    };
    
    // Pass by reference - operates on original
    auto by_reference = [](int& x) {
        x = 999;  // Modifies original
        std::cout << "Inside by_reference: " << x << std::endl;
    };
    
    // Pass by const reference - efficient, safe
    auto by_const_reference = [](const int& x) {
        // x = 999;  // ERROR: cannot modify const
        std::cout << "Inside by_const_reference: " << x << std::endl;
    };
    
    std::cout << "Original: " << original << std::endl;        // 42
    by_value(original);                                        // 999
    std::cout << "After by_value: " << original << std::endl;  // 42
    
    by_reference(original);                                    // 999
    std::cout << "After by_reference: " << original << std::endl; // 999
    
    by_const_reference(original);                              // 999
}
```

#### Best Practices for Parameter Passing

```cpp
class ExpensiveObject {
private:
    std::vector<int> large_data;
    std::string metadata;
    
public:
    ExpensiveObject(size_t size) : large_data(size, 0) {}
    // ... other members
};

class ParameterExamples {
public:
    // BAD: Expensive copy for large objects
    void bad_by_value(ExpensiveObject obj) {
        // obj is a complete copy
    }
    
    // GOOD: Efficient, read-only access
    void good_by_const_ref(const ExpensiveObject& obj) {
        // No copy, cannot modify
    }
    
    // GOOD: When modification is intended
    void good_by_ref(ExpensiveObject& obj) {
        // No copy, can modify original
    }
    
    // GOOD: For small, fundamental types
    void good_by_value(int x, double y, char c) {
        // Copying primitives is cheap
    }
    
    // EXCELLENT: For output streams (cannot be copied)
    void print_object(std::ostream& os, const ExpensiveObject& obj) {
        os << "Object data..." << std::endl;
    }
};
```

#### Modern C++ Parameter Guidelines

```cpp
// Template for automatic best choice
template<typename T>
void smart_function(T&& param) {  // Universal reference
    // Perfect forwarding - covered in advanced topics
    process(std::forward<T>(param));
}

// Specific guidelines:
void guidelines_example() {
    // Small objects (≤ 2-3 pointers): by value
    void process_small(int x, Point2D p);
    
    // Large objects, read-only: by const reference  
    void process_large_readonly(const std::vector<int>& data);
    
    // Large objects, modification needed: by reference
    void process_large_modify(std::vector<int>& data);
    
    // Output parameters: by reference
    bool try_parse(const std::string& input, int& result);
    
    // Optional output: by pointer
    bool try_parse_ptr(const std::string& input, int* result = nullptr);
}
```

---

## Type Conversions

### Implicit Conversions

#### Constructor-Based Conversions

```cpp
class SmartInt {
private:
    int value;
    
public:
    // Implicit conversion from int
    SmartInt(int v) : value(v) {}
    
    // Explicit conversion prevents accidents
    explicit SmartInt(double d) : value(static_cast<int>(d)) {}
    
    // Multiple parameters - no implicit conversion
    SmartInt(int v, bool validated) : value(v) {}
    
    int get() const { return value; }
};

void conversion_examples() {
    SmartInt a = 42;        // OK: implicit conversion from int
    SmartInt b(42);         // OK: direct initialization
    SmartInt c{42};         // OK: uniform initialization
    
    // SmartInt d = 3.14;   // ERROR: explicit constructor
    SmartInt e(3.14);       // OK: explicit call
    SmartInt f{3.14};       // OK: uniform initialization
    
    // SmartInt g = {1, true}; // ERROR: multiple parameters
    SmartInt h(1, true);       // OK: direct initialization
    SmartInt i{1, true};       // OK: uniform initialization
}
```

#### Conversion Operators

```cpp
class AdvancedSmartInt {
private:
    int value;
    
public:
    AdvancedSmartInt(int v) : value(v) {}
    
    // Implicit conversion to int
    operator int() const { return value; }
    
    // Explicit conversion to bool
    explicit operator bool() const { return value != 0; }
    
    // Conversion to string
    operator std::string() const { 
        return std::to_string(value); 
    }
};

void advanced_conversions() {
    AdvancedSmartInt smart(42);
    
    int regular = smart;           // OK: implicit conversion to int
    std::string str = smart;       // OK: implicit conversion to string
    
    // bool flag = smart;          // ERROR: explicit conversion required
    bool flag = static_cast<bool>(smart);  // OK: explicit cast
    
    if (smart) {                   // OK: contextual conversion to bool
        std::cout << "Non-zero value" << std::endl;
    }
}
```

### Explicit Conversions

#### Modern C++ Cast Operators

```cpp
class CastingExamples {
public:
    void demonstrate_casts() {
        // static_cast: Safe, compile-time checked conversions
        double d = 3.14159;
        int i = static_cast<int>(d);           // Truncates to 3
        
        void* raw_ptr = &i;
        int* typed_ptr = static_cast<int*>(raw_ptr);  // Safe upcast
        
        // dynamic_cast: Runtime type checking for polymorphic types
        class Base { virtual ~Base() = default; };
        class Derived : public Base {};
        
        Base* base_ptr = new Derived();
        Derived* derived_ptr = dynamic_cast<Derived*>(base_ptr);
        if (derived_ptr) {
            // Safe to use derived_ptr
        }
        
        // reinterpret_cast: Dangerous, bitwise reinterpretation
        uintptr_t address = reinterpret_cast<uintptr_t>(typed_ptr);
        
        // const_cast: Remove/add const (use sparingly)
        const int* const_ptr = &i;
        int* mutable_ptr = const_cast<int*>(const_ptr);
        
        delete base_ptr;
    }
};
```

---

## Operator Overloading

### Comprehensive Operator Overloading Guide

#### Arithmetic Operators

```cpp
class ComplexNumber {
private:
    double real, imag;
    
public:
    ComplexNumber(double r = 0.0, double i = 0.0) : real(r), imag(i) {}
    
    // Member function operators
    ComplexNumber& operator+=(const ComplexNumber& rhs) {
        real += rhs.real;
        imag += rhs.imag;
        return *this;
    }
    
    ComplexNumber& operator-=(const ComplexNumber& rhs) {
        real -= rhs.real;
        imag -= rhs.imag;
        return *this;
    }
    
    // Prefix increment
    ComplexNumber& operator++() {
        ++real;
        return *this;
    }
    
    // Postfix increment
    ComplexNumber operator++(int) {
        ComplexNumber temp(*this);
        ++real;
        return temp;
    }
    
    // Comparison operators
    bool operator==(const ComplexNumber& rhs) const {
        return (real == rhs.real) && (imag == rhs.imag);
    }
    
    bool operator!=(const ComplexNumber& rhs) const {
        return !(*this == rhs);
    }
    
    // Access operators
    double& operator[](size_t index) {
        return (index == 0) ? real : imag;
    }
    
    const double& operator[](size_t index) const {
        return (index == 0) ? real : imag;
    }
    
    // Function call operator (functor)
    double operator()() const {
        return std::sqrt(real * real + imag * imag);  // magnitude
    }
    
    // Getters for non-member functions
    double getReal() const { return real; }
    double getImag() const { return imag; }
};

// Non-member operators (often preferred for symmetry)
ComplexNumber operator+(const ComplexNumber& lhs, const ComplexNumber& rhs) {
    ComplexNumber result = lhs;
    result += rhs;
    return result;
}

ComplexNumber operator-(const ComplexNumber& lhs, const ComplexNumber& rhs) {
    ComplexNumber result = lhs;
    result -= rhs;
    return result;
}

// Stream operators
std::ostream& operator<<(std::ostream& os, const ComplexNumber& c) {
    os << "(" << c.getReal() << " + " << c.getImag() << "i)";
    return os;
}

std::istream& operator>>(std::istream& is, ComplexNumber& c) {
    double r, i;
    char lparen, plus, ioperation, rparen;
    is >> lparen >> r >> plus >> i >> ioperation >> rparen;
    if (lparen == '(' && plus == '+' && ioperation == 'i' && rparen == ')') {
        c = ComplexNumber(r, i);
    }
    return is;
}
```

#### Smart Pointer Implementation Example

```cpp
template<typename T>
class SimpleUniquePtr {
private:
    T* ptr;
    
public:
    explicit SimpleUniquePtr(T* p = nullptr) : ptr(p) {}
    
    ~SimpleUniquePtr() { delete ptr; }
    
    // Disable copy
    SimpleUniquePtr(const SimpleUniquePtr&) = delete;
    SimpleUniquePtr& operator=(const SimpleUniquePtr&) = delete;
    
    // Enable move (simplified)
    SimpleUniquePtr(SimpleUniquePtr&& other) noexcept : ptr(other.ptr) {
        other.ptr = nullptr;
    }
    
    SimpleUniquePtr& operator=(SimpleUniquePtr&& other) noexcept {
        if (this != &other) {
            delete ptr;
            ptr = other.ptr;
            other.ptr = nullptr;
        }
        return *this;
    }
    
    // Dereference operators
    T& operator*() const { return *ptr; }
    T* operator->() const { return ptr; }
    
    // Boolean conversion
    explicit operator bool() const { return ptr != nullptr; }
    
    // Release ownership
    T* release() {
        T* temp = ptr;
        ptr = nullptr;
        return temp;
    }
    
    // Reset pointer
    void reset(T* p = nullptr) {
        delete ptr;
        ptr = p;
    }
    
    T* get() const { return ptr; }
};
```

### I/O Manipulator Implementation

```cpp
// Understanding how endl works
namespace CustomManipulators {
    
std::ostream& custom_endl(std::ostream& os) {
    os << '\n';
    os.flush();
    return os;
}

std::ostream& tab(std::ostream& os) {
    os << '\t';
    return os;
}

// Parameterized manipulator
class SetWidth {
private:
    int width;
public:
    explicit SetWidth(int w) : width(w) {}
    
    friend std::ostream& operator<<(std::ostream& os, const SetWidth& sw) {
        os.width(sw.width);
        return os;
    }
};

SetWidth setw(int width) {
    return SetWidth(width);
}

}

// Usage
void manipulator_demo() {
    using namespace CustomManipulators;
    
    std::cout << "Hello" << custom_endl << "World" << tab << "!" << std::endl;
    std::cout << setw(10) << "Right" << setw(10) << "Aligned" << std::endl;
}
```

---

## Object Lifecycle and Storage Duration

### Storage Duration Categories

#### Automatic Storage Duration

```cpp
class AutomaticExample {
private:
    std::string name;
    
public:
    AutomaticExample(const std::string& n) : name(n) {
        std::cout << "Creating " << name << std::endl;
    }
    
    ~AutomaticExample() {
        std::cout << "Destroying " << name << std::endl;
    }
};

void demonstrate_automatic() {
    std::cout << "Entering function" << std::endl;
    
    AutomaticExample obj1("first");
    {
        AutomaticExample obj2("second");
        AutomaticExample obj3("third");
        // obj3 destroyed here
        // obj2 destroyed here
    }
    
    AutomaticExample obj4("fourth");
    // obj4 destroyed here
    // obj1 destroyed here
    
    std::cout << "Exiting function" << std::endl;
}
// Output:
// Entering function
// Creating first
// Creating second  
// Creating third
// Destroying third
// Destroying second
// Creating fourth
// Destroying fourth
// Destroying first
// Exiting function
```

#### Static Storage Duration

```cpp
class StaticExample {
private:
    std::string identifier;
    static int instance_count;
    
public:
    StaticExample(const std::string& id) : identifier(id) {
        ++instance_count;
        std::cout << "Creating static " << identifier 
                  << " (count: " << instance_count << ")" << std::endl;
    }
    
    ~StaticExample() {
        std::cout << "Destroying static " << identifier << std::endl;
    }
    
    static int getInstanceCount() { return instance_count; }
};

int StaticExample::instance_count = 0;

// Global objects - constructed before main()
StaticExample global_obj("global");

void function_with_static() {
    static StaticExample func_static("function_static");
    std::cout << "Inside function_with_static" << std::endl;
}

class ClassWithStatic {
public:
    static StaticExample class_static;
};

StaticExample ClassWithStatic::class_static("class_static");

int main() {
    std::cout << "Entering main" << std::endl;
    
    static StaticExample main_static("main_static");
    
    function_with_static();  // func_static created on first call
    function_with_static();  // func_static already exists
    
    std::cout << "Total static objects: " << StaticExample::getInstanceCount() << std::endl;
    std::cout << "Exiting main" << std::endl;
    
    return 0;
}
// Static objects destroyed in reverse order after main() returns
```

#### Dynamic Storage Duration

```cpp
class DynamicExample {
private:
    std::vector<int> data;
    std::string name;
    
public:
    DynamicExample(const std::string& n, size_t size) 
        : name(n), data(size, 0) {
        std::cout << "Creating dynamic " << name 
                  << " with " << size << " elements" << std::endl;
    }
    
    ~DynamicExample() {
        std::cout << "Destroying dynamic " << name << std::endl;
    }
    
    void process() {
        std::cout << "Processing " << name << std::endl;
    }
};

void demonstrate_dynamic() {
    // Modern C++ - use smart pointers
    {
        auto ptr1 = std::make_unique<DynamicExample>("unique_ptr", 1000);
        ptr1->process();
        
        // Transfer ownership
        auto ptr2 = std::move(ptr1);  // ptr1 is now nullptr
        if (ptr2) {
            ptr2->process();
        }
        
        // ptr2 automatically destroyed at end of scope
    }
    
    // Shared ownership
    {
        auto shared1 = std::make_shared<DynamicExample>("shared_ptr", 2000);
        {
            auto shared2 = shared1;  // Both point to same object
            std::cout << "Reference count: " << shared1.use_count() << std::endl;
            // shared2 destroyed here, but object survives
        }
        shared1->process();
        // Object destroyed when shared1 goes out of scope
    }
}
```

### RAII (Resource Acquisition Is Initialization)

```cpp
class FileManager {
private:
    std::FILE* file;
    std::string filename;
    
public:
    explicit FileManager(const std::string& fname) : filename(fname) {
        file = std::fopen(fname.c_str(), "r");
        if (!file) {
            throw std::runtime_error("Failed to open file: " + filename);
        }
        std::cout << "Opened file: " << filename << std::endl;
    }
    
    ~FileManager() {
        if (file) {
            std::fclose(file);
            std::cout << "Closed file: " << filename << std::endl;
        }
    }
    
    // Disable copy
    FileManager(const FileManager&) = delete;
    FileManager& operator=(const FileManager&) = delete;
    
    // Enable move
    FileManager(FileManager&& other) noexcept 
        : file(other.file), filename(std::move(other.filename)) {
        other.file = nullptr;
    }
    
    FileManager& operator=(FileManager&& other) noexcept {
        if (this != &other) {
            if (file) std::fclose(file);
            file = other.file;
            filename = std::move(other.filename);
            other.file = nullptr;
        }
        return *this;
    }
    
    std::FILE* get() const { return file; }
    bool is_open() const { return file != nullptr; }
};

void demonstrate_raii() {
    try {
        FileManager fm("example.txt");
        
        // File operations here
        if (fm.is_open()) {
            // Process file
        }
        
        // File automatically closed when fm goes out of scope
        // Even if an exception is thrown!
        
    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << std::endl;
    }
}
```

---

## Practical Examples and Code Analysis

### Improved Pascal's Triangle Implementation

```cpp
#include <vector>
#include <iostream>
#include <iomanip>
#include <algorithm>
#include <sstream>

class PascalsTriangle {
public:
    using Row = std::vector<long long>;  // Use long long to handle larger values
    
private:
    std::vector<Row> rows;
    size_t max_element_width;
    
    void calculate_element_width() {
        if (rows.empty()) {
            max_element_width = 1;
            return;
        }
        
        auto max_element = *std::max_element(rows.back().begin(), rows.back().end());
        max_element_width = std::to_string(max_element).length();
    }
    
    void add_row() {
        if (rows.empty()) {
            rows.push_back({1});
            return;
        }
        
        const Row& previous_row = rows.back();
        Row new_row;
        new_row.reserve(previous_row.size() + 1);
        
        new_row.push_back(1);  // First element is always 1
        
        // Calculate middle elements
        for (size_t i = 1; i < previous_row.size(); ++i) {
            new_row.push_back(previous_row[i-1] + previous_row[i]);
        }
        
        new_row.push_back(1);  // Last element is always 1
        rows.push_back(std::move(new_row));
    }
    
    std::string format_element(long long value) const {
        std::ostringstream oss;
        oss << std::setw(max_element_width) << value;
        return oss.str();
    }
    
public:
    explicit PascalsTriangle(size_t num_rows) {
        rows.reserve(num_rows);
        
        for (size_t i = 0; i < num_rows; ++i) {
            add_row();
        }
        
        calculate_element_width();
    }
    
    // Get specific row (0-indexed)
    const Row& get_row(size_t row_index) const {
        if (row_index >= rows.size()) {
            throw std::out_of_range("Row index out of range");
        }
        return rows[row_index];
    }
    
    // Get specific element (0-indexed)
    long long get_element(size_t row, size_t col) const {
        const auto& target_row = get_row(row);
        if (col >= target_row.size()) {
            throw std::out_of_range("Column index out of range");
        }
        return target_row[col];
    }
    
    size_t num_rows() const { return rows.size(); }
    
    // Print specific row
    void print_row(std::ostream& os, size_t row_index) const {
        const Row& row = get_row(row_index);
        
        for (size_t i = 0; i < row.size(); ++i) {
            if (i > 0) os << " ";
            os << format_element(row[i]);
        }
    }
    
    // Print entire triangle
    friend std::ostream& operator<<(std::ostream& os, const PascalsTriangle& triangle) {
        const size_t num_rows = triangle.rows.size();
        const size_t spacing = triangle.max_element_width + 1;
        
        for (size_t i = 0; i < num_rows; ++i) {
            // Calculate leading spaces for centering
            size_t leading_spaces = (num_rows - i - 1) * spacing / 2;
            os << std::string(leading_spaces, ' ');
            
            triangle.print_row(os, i);
            os << '\n';
        }
        return os;
    }
};

// Usage example
void pascal_demo() {
    try {
        PascalsTriangle triangle(10);
        std::cout << triangle << std::endl;
        
        // Access specific elements
        std::cout << "Element at (5,2): " << triangle.get_element(5, 2) << std::endl;
        
    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << std::endl;
    }
}
```

### Enhanced STL Algorithm Examples

```cpp
#include <algorithm>
#include <numeric>
#include <vector>
#include <iterator>
#include <functional>
#include <cmath>

class STLAlgorithmExamples {
public:
    // Vector length calculation using different approaches
    static void demonstrate_vector_length() {
        std::vector<double> vec{3.0, 4.0, 5.0};
        
        // Method 1: Manual transformation + accumulation
        std::vector<double> squared(vec.size());
        std::transform(vec.begin(), vec.end(), squared.begin(),
                      [](double x) { return x * x; });
        
        double length1 = std::sqrt(std::accumulate(squared.begin(), squared.end(), 0.0));
        
        // Method 2: Using back_inserter (safer)
        std::vector<double> squared2;
        std::transform(vec.begin(), vec.end(), std::back_inserter(squared2),
                      [](double x) { return x * x; });
        
        double length2 = std::sqrt(std::accumulate(squared2.begin(), squared2.end(), 0.0));
        
        // Method 3: Using inner_product (most elegant)
        double length3 = std::sqrt(std::inner_product(vec.begin(), vec.end(), 
                                                     vec.begin(), 0.0));
        
        // Method 4: Using accumulate with custom operation
        double length4 = std::sqrt(std::accumulate(vec.begin(), vec.end(), 0.0,
                                                  [](double sum, double x) {
                                                      return sum + x * x;
                                                  }));
        
        // Method 5: Modern approach with std::transform_reduce (C++17)
        double length5 = std::sqrt(std::transform_reduce(vec.begin(), vec.end(), 0.0,
                                                        std::plus<>(),
                                                        [](double x) { return x * x; }));
        
        std::cout << "Vector: ";
        std::copy(vec.begin(), vec.end(), std::ostream_iterator<double>(std::cout, " "));
        std::cout << "\nLength (method 1): " << length1;
        std::cout << "\nLength (method 2): " << length2;
        std::cout << "\nLength (method 3): " << length3;
        std::cout << "\nLength (method 4): " << length4;
        std::cout << "\nLength (method 5): " << length5 << std::endl;
    }
    
    // Advanced algorithm combinations
    static void demonstrate_advanced_algorithms() {
        std::vector<int> data{5, 2, 8, 1, 9, 3, 7, 4, 6};
        
        // Find all elements greater than 5
        std::vector<int> filtered;
        std::copy_if(data.begin(), data.end(), std::back_inserter(filtered),
                    [](int x) { return x > 5; });
        
        // Transform and filter in one step
        std::vector<int> transformed_filtered;
        std::transform_if(data.begin(), data.end(), std::back_inserter(transformed_filtered),
                         [](int x) { return x * 2; },  // transform
                         [](int x) { return x % 2 == 0; }); // filter condition
        
        // Partition data
        auto partition_point = std::partition(data.begin(), data.end(),
                                            [](int x) { return x % 2 == 0; });
        
        std::cout << "Original: ";
        std::copy(data.begin(), data.end(), std::ostream_iterator<int>(std::cout, " "));
        
        std::cout << "\nFiltered (>5): ";
        std::copy(filtered.begin(), filtered.end(), std::ostream_iterator<int>(std::cout, " "));
        
        std::cout << "\nPartitioned (even first): ";
        std::copy(data.begin(), data.end(), std::ostream_iterator<int>(std::cout, " "));
        std::cout << std::endl;
    }
};

// Custom transform_if implementation (not in standard library)
template<typename InputIt, typename OutputIt, typename UnaryOp, typename Predicate>
OutputIt transform_if(InputIt first, InputIt last, OutputIt d_first,
                      UnaryOp unary_op, Predicate pred) {
    for (; first != last; ++first) {
        if (pred(*first)) {
            *d_first = unary_op(*first);
            ++d_first;
        }
    }
    return d_first;
}
```

### Function Object (Functor) Examples

```cpp
// Homework solution: Nth_Power functor
class Nth_Power {
private:
    int exponent;
    
public:
    explicit Nth_Power(int exp) : exponent(exp) {}
    
    // Function call operator
    template<typename T>
    auto operator()(T base) const -> decltype(std::pow(base, exponent)) {
        return std::pow(base, exponent);
    }
    
    // Alternative implementation for integer bases
    long long operator()(int base) const {
        long long result = 1;
        for (int i = 0; i < exponent; ++i) {
            result *= base;
        }
        return result;
    }
};

// Advanced functor with state
class Accumulator {
private:
    mutable double sum = 0.0;  // mutable allows modification in const function
    
public:
    double operator()(double value) const {
        sum += value;
        return sum;
    }
    
    double get_sum() const { return sum; }
    void reset() { sum = 0.0; }
};

// Generic predicate functor
template<typename T, typename Comparator = std::less<T>>
class InRange {
private:
    T min_val, max_val;
    Comparator comp;
    
public:
    InRange(T min_v, T max_v, Comparator c = Comparator{}) 
        : min_val(min_v), max_val(max_v), comp(c) {}
    
    bool operator()(const T& value) const {
        return !comp(value, min_val) && comp(value, max_val);
    }
};

void functor_examples() {
    // Nth_Power usage
    std::vector<int> numbers{1, 2, 3, 4, 5};
    Nth_Power cube{3};
    
    std::cout << "7 cubed: " << cube(7) << std::endl;  // 343
    
    std::cout << "First five cubes: ";
    std::transform(numbers.begin(), numbers.end(),
                  std::ostream_iterator<long long>(std::cout, " "),
                  cube);
    std::cout << std::endl;
    
    // Accumulator usage
    std::vector<double> values{1.5, 2.5, 3.5, 4.5};
    Accumulator acc;
    
    std::for_each(values.begin(), values.end(), acc);
    std::cout << "Sum: " << acc.get_sum() << std::endl;
    
    // InRange usage
    std::vector<int> data{1, 5, 10, 15, 20, 25, 30};
    InRange<int> in_range(10, 25);
    
    auto count = std::count_if(data.begin(), data.end(), in_range);
    std::cout << "Numbers in range [10, 25): " << count << std::endl;
}
```

### Memory Management Best Practices

```cpp
class ModernMemoryManagement {
public:
    // RAII wrapper for C-style resources
    template<typename T, typename Deleter>
    class RAIIWrapper {
    private:
        T* resource;
        Deleter deleter;
        
    public:
        explicit RAIIWrapper(T* r, Deleter d) : resource(r), deleter(d) {}
        
        ~RAIIWrapper() {
            if (resource) {
                deleter(resource);
            }
        }
        
        // Non-copyable
        RAIIWrapper(const RAIIWrapper&) = delete;
        RAIIWrapper& operator=(const RAIIWrapper&) = delete;
        
        // Movable
        RAIIWrapper(RAIIWrapper&& other) noexcept 
            : resource(other.resource), deleter(std::move(other.deleter)) {
            other.resource = nullptr;
        }
        
        RAIIWrapper& operator=(RAIIWrapper&& other) noexcept {
            if (this != &other) {
                if (resource) deleter(resource);
                resource = other.resource;
                deleter = std::move(other.deleter);
                other.resource = nullptr;
            }
            return *this;
        }
        
        T* get() const { return resource; }
        T* operator->() const { return resource; }
        T& operator*() const { return *resource; }
        
        explicit operator bool() const { return resource != nullptr; }
    };
    
    // Factory function for RAII wrapper
    template<typename T, typename Deleter>
    static auto make_raii(T* resource, Deleter deleter) {
        return RAIIWrapper<T, Deleter>(resource, deleter);
    }
    
    static void demonstrate_memory_patterns() {
        // Pattern 1: Unique ownership
        {
            auto unique_data = std::make_unique<std::vector<int>>(1000, 42);
            // Process data...
            // Automatically cleaned up
        }
        
        // Pattern 2: Shared ownership
        {
            auto shared_data = std::make_shared<std::string>("shared resource");
            std::vector<std::shared_ptr<std::string>> holders;
            
            for (int i = 0; i < 5; ++i) {
                holders.push_back(shared_data);
            }
            
            std::cout << "Reference count: " << shared_data.use_count() << std::endl;
            // Resource cleaned up when last shared_ptr is destroyed
        }
        
        // Pattern 3: Custom deleter
        {
            auto file_wrapper = make_raii(std::fopen("temp.txt", "w"),
                                         [](std::FILE* f) { 
                                             if (f) std::fclose(f); 
                                         });
            
            if (file_wrapper) {
                std::fprintf(file_wrapper.get(), "Hello, RAII!\n");
            }
            // File automatically closed
        }
        
        // Pattern 4: Container of smart pointers
        {
            std::vector<std::unique_ptr<int>> int_ptrs;
            
            for (int i = 0; i < 10; ++i) {
                int_ptrs.push_back(std::make_unique<int>(i * i));
            }
            
            // Process container...
            // All managed objects automatically cleaned up
        }
    }
};
```

---

## Key Takeaways and Best Practices

### Performance Guidelines

1. **Always benchmark in release builds** - Debug builds provide misleading performance data
2. **Choose algorithms based on complexity**, not intuition:
   - `std::nth_element`: O(n) for finding kth element
   - `std::partial_sort`: O(n log k) 
   - `std::sort`: O(n log n)
3. **Profile before optimizing** - Expert developers are poor at predicting performance

### Parameter Passing Rules

1. **Small objects (≤ 16 bytes)**: Pass by value
2. **Large objects, read-only**: Pass by const reference
3. **Large objects, modification needed**: Pass by non-const reference
4. **Output parameters**: Use references or pointers
5. **Optional parameters**: Use pointers or std::optional

### Constructor Design Principles

1. **Use member initializer lists** for efficiency and correctness
2. **Prefer NSDMI** for default values when possible
3. **Mark single-parameter constructors explicit** unless conversion is intended
4. **Follow Rule of Zero/Three/Five** for resource management

### Modern C++ Patterns

1. **Prefer RAII** over manual resource management
2. **Use smart pointers** instead of raw pointers for ownership
3. **Embrace move semantics** for expensive objects
4. **Use range-based algorithms** with lambdas for clarity
5. **Apply const-correctness** throughout your design

### Common Pitfalls to Avoid

1. **Forgetting to enable optimizations** when measuring performance
2. **Using pass-by-value for large objects** unnecessarily
3. **Implementing copy constructors incorrectly** for classes with resources
4. **Not following the Rule of Three/Five** consistently
5. **Overusing explicit type conversions** instead of designing clear interfaces
