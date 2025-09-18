# Lecture 1 Summary

## Essential Resources

### Web Resources
- **Language Standard**: https://eel.is/c++draft/
- **ISO C++**: http://isocpp.org/
- **Core Guidelines**: http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines
- **Standards Committee**: http://www.open-std.org/jtc1/sc22/wg21/
- **Online Compilers**: 
  - https://wandbox.org
  - https://godbolt.org (Compiler Explorer)
- **Community**: StackOverflow C++, cppreference.com, r/cpp, http://includecpp.org

### Recommended Books
- **Bjarne Stroustrup**: 
  - *A Tour of C++* (3rd edition) - C++ for programmers
  - *The C++ Programming Language* (4th edition) - reference (covers through C++14)
  - *Programming: Principles and Practices* - intro to programming
- **Nico Josuttis**: *C++20, The Complete Guide*
- **Anthony Williams**: *C++ Concurrency in Action* (2nd edition, free upgrade to 3rd at Manning)

## C++ Language Characteristics

### Core Properties
1. **Compiled Language** (Ahead-of-Time compilation)
2. **Multiparadigm** (not just object-oriented)
3. **Lightweight Abstractions**
4. **Low-level + High-level Support**
5. **Statically Type-safe with Type Inference**
6. **RAII, Exceptions, and Expected**

### The "Lightweight Abstraction" Philosophy
- **Problem**: Traditional tradeoff between abstraction (good for programmers) and performance (good for computers)
- **C++ Solution**: Powerful abstractions with zero performance penalty through compile-time generation
- **Result**: Best of both worlds - clear, maintainable code that runs as fast as hand-optimized C

### Example: Object Copying

#### C Approach (Low-level, Brittle)
```c
// Simple object - just memory copy
HTMLPage a, b;
memcpy(&b, &a, sizeof(HTMLPage)); // Fast but not abstract

// Compound object - manual deep copy required
HTMLPage a = /* ... */;
HTMLPage b;
// Must manually implement deep copying logic
deep_copy(&b, &a); // Pollutes code with implementation details
```

#### C++ Approach (Abstract + Efficient)
```cpp
class HTMLPage {
public:
    // Copy constructor defines proper copying behavior
    HTMLPage(const HTMLPage& other) { 
        // Deep copy implementation
        // Compiler handles the complexity
    }
    /* ... */
};

HTMLPage a = /* ... */;
HTMLPage b = a; // Abstract: simple assignment
                // Lightweight: compiler generates optimal code
```

### Example: Container Copying

#### Java Approach (Abstract but Inefficient)
```java
Collection<Employee> copy = new HashSet<Employee>(org.size());
Iterator<Employee> iterator = org.iterator();
while(iterator.hasNext()) {
    copy.add(iterator.next().clone()); // One object at a time
}
```

#### C++ std::copy (Abstract AND Efficient)
```cpp
vector<char> v;
// ... populate v
copy(v.begin(), v.end(), arr); // Abstract interface

// Compiler automatically generates:
// - memcpy for contiguous primitive types (800% faster)
// - Proper iteration for complex types
// - Optimal code for each specific case
```

## Type System

### Static Type Safety
```cpp
// C++ enforces type safety at compile time
string a = "foo"s;
a = 7; // Compile error - type mismatch

// Compare with JavaScript:
// var a = "foo";
// a = 7; // Legal but potentially problematic
// console.log(a.length()); // Runtime error
```

### Type Inference with `auto`
```cpp
// Get type safety without verbosity
auto a = "foo"s;        // Deduced as string
auto numbers = {1,2,3}; // Deduced as initializer_list<int>
a = 7; // Still a compile error - type doesn't change
```


## Development Environment Setup

### Build System: CMake
```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.5)
set(CMAKE_CXX_STANDARD 20)
project(HelloWorld)
add_executable(hello_world hello.cpp)
```

Build commands:
```bash
cmake -S . -B ./build
cmake --build ./build
```

## Basic C++ Syntax

### Hello World
```cpp
#include <iostream>

int main() {
    std::cout << "Hello, world!\n";
    return 0;
}
```

### Personalized Greeting (C++20)
```cpp
#include <iostream>
#include <format>
using namespace std;

int main() {
    string name;
    cout << "What's your name? ";
    cin >> name;
    cout << format("Hello, {}!\n", name);
    return 0;
}
```


### Variables and Type Inference
```cpp
// Explicit typing
int i = 5;
i = 7;          // OK - same type
// i = "hello"; // Compile error - wrong type

// Type inference
auto j = 3;           // j is int
auto name = "Mike"s;  // name is string
auto pi = 3.14159;    // pi is double
```

### Function Definitions and Overloading
```cpp
// Traditional function
int square(int n) {
    return n * n;
}

// Overloaded function (same name, different types)
double square(double n) {
    return n * n;
}

// C++20 abbreviated function template
auto square(auto n) {
    return n * n;
}

// Pre-C++20 template syntax
template<typename T>
T square(T n) {
    return n * n;
}

int main() {
    cout << square(2) + square(3.1416);  // Calls appropriate overload
    return 0;
}
```

### Containers: Vector Template
```cpp
#include <vector>
#include <iostream>
using namespace std;

int main() {
    // Explicit type specification
    vector<int> v1 = {1, 2, 3};
    
    // Type deduction (C++17)
    vector v2 = {1, 2, 3};  // Deduced as vector<int>
    
    // Adding elements
    v2.push_back(4);
    
    // Range-based for loop
    for (auto element : v2) {
        cout << element << ' ';  // Prints: 1 2 3 4
    }
    
    return 0;
}
```

## Code Examples

### Vector Demo
```cpp
#include <vector>
#include <iostream>
using namespace std;

// generic square function
auto square(auto x) { 
    return x * x; 
}

int main() {
    vector v = {1, 2, 3, 4};  // C++17 CTAD
    v.push_back(5);
    
    cout << "Original: ";
    for (const auto& elem : v) {  // const reference for efficiency
        cout << elem << " ";
    }
    cout << "\nSquared: ";
    
    for (const auto& elem : v) {
        cout << square(elem) << " ";
    }
    cout << "\n";
    
    return 0;
}
```

### Frame Program
```cpp
#include <iostream>
#include <string>
#include <format>
using namespace std;

int main() {
    cout << "What's your name? ";
    string name;
    cin >> name;
    
    const string greeting = format("Hello, {}!", name);
    const int pad = 5;
    
    // Calculate dimensions
    const int rows = pad * 2 + 3;
    const size_t cols = greeting.size() + pad * 2 + 2;
    
    // Create border strings for efficiency
    const string stars(cols, '*');
    const string spaces(greeting.size(), ' ');
    const string mixed = format("* {} *", spaces);
    
    cout << "\n" << stars << "\n"
         << mixed << "\n"
         << format("* {} *\n", greeting)
         << mixed << "\n"
         << stars << "\n";
    
    return 0;
}
```

## Key Concepts and Best Practices

### The Preprocessor
- `#include` and `#define` are text manipulation commands
- **Avoid** preprocessor for convenience - causes problems:
  - Ignores namespaces
  - Bloats compilation units
  - Complicates refactoring
- Modern C++ provides better alternatives (modules, constexpr, etc.)

### Why C++ is Different
- **Not** "better C" or "worse Java"
- Superficial similarities are misleading
- Good C/Java code ≠ Good C++ code
- Requires learning C++-specific idioms and patterns

