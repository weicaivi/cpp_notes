# Lecture 15 Summary

## 1. Invariants in Concurrent Programming

### Definition and Purpose

An **invariant** is a condition that must be maintained by every function throughout program execution. Invariants are critical for ensuring program correctness, especially in concurrent environments.

### Common C++ Invariants

#### RAII Invariant
```cpp
// Invariant: All dynamically allocated resources must be owned by RAII objects

// BAD: Violates invariant
void riskyFunction() {
    int* ptr = new int(42);  // Raw ownership
    doSomething();            // Exception here causes leak
    delete ptr;
}

// GOOD: Maintains invariant
void safeFunction() {
    auto ptr = std::make_unique<int>(42);  // RAII ownership
    doSomething();                          // Exception safe
    // Automatic cleanup
}
```

#### Thread-Safety Invariant
```cpp
template<typename T>
class ThreadSafeContainer {
private:
    mutable std::mutex mutex_;
    std::vector<T> data_;
    
    // Invariant: mutex_ must be locked before accessing data_
public:
    void add(const T& item) {
        std::lock_guard<std::mutex> lock(mutex_);  // Maintains invariant
        data_.push_back(item);
    }
    
    size_t size() const {
        std::lock_guard<std::mutex> lock(mutex_);  // Maintains invariant
        return data_.size();
    }
};
```

---

## 2. Lock-Ordering and Deadlock Prevention

### The Deadlock Problem

Deadlocks occur when threads acquire multiple locks in different orders:

```cpp
// DEADLOCK SCENARIO
std::mutex mutexA, mutexB;

// Thread 1
void thread1() {
    std::lock_guard<std::mutex> lockA(mutexA);  // Acquires A
    std::this_thread::sleep_for(10ms);           // Timing window
    std::lock_guard<std::mutex> lockB(mutexB);  // Waits for B (held by Thread 2)
    // Work...
}

// Thread 2
void thread2() {
    std::lock_guard<std::mutex> lockB(mutexB);  // Acquires B
    std::this_thread::sleep_for(10ms);           // Timing window
    std::lock_guard<std::mutex> lockA(mutexA);  // Waits for A (held by Thread 1)
    // Work...
}
```

### Lock-Ordering Invariant Solution

Define a **total order** on all locks and maintain this invariant:
> "Of all locks a thread owns, the one with the highest order was acquired most recently"

```cpp
// Solution: Consistent lock ordering
class BankAccount {
private:
    mutable std::mutex mutex_;
    double balance_;
    const int id_;  // Used for ordering
    
public:
    BankAccount(int id, double initial) : id_(id), balance_(initial) {}
    
    // Transfer maintains lock-ordering invariant
    static void transfer(BankAccount& from, BankAccount& to, double amount) {
        // Always lock lower ID first
        if (from.id_ < to.id_) {
            std::lock_guard<std::mutex> lock1(from.mutex_);
            std::lock_guard<std::mutex> lock2(to.mutex_);
            from.balance_ -= amount;
            to.balance_ += amount;
        } else {
            std::lock_guard<std::mutex> lock1(to.mutex_);
            std::lock_guard<std::mutex> lock2(from.mutex_);
            from.balance_ -= amount;
            to.balance_ += amount;
        }
    }
};
```

### Using std::lock for Multiple Mutexes
```cpp
// Alternative: std::lock acquires multiple locks atomically
void transfer_atomic(BankAccount& from, BankAccount& to, double amount) {
    std::unique_lock<std::mutex> lock1(from.mutex_, std::defer_lock);
    std::unique_lock<std::mutex> lock2(to.mutex_, std::defer_lock);
    
    // Acquires both locks without deadlock
    std::lock(lock1, lock2);
    
    from.balance_ -= amount;
    to.balance_ += amount;
}
```

---

## 3. Templates vs Lock-Ordering Invariants

### The Composability Problem

Templates cannot know what locks their type parameters might acquire, breaking lock-ordering invariants:

```cpp
// Global logger with internal lock
struct Logger {
    void add(const std::string& msg) {
        std::lock_guard<std::recursive_mutex> _(mutex_);
        log_ += msg;
    }
    
    void lock() { mutex_.lock(); }
    void unlock() { mutex_.unlock(); }
    
private:
    std::recursive_mutex mutex_;
    std::string log_;
} globalLogger;

// Type T with invariant checking
class DataType {
public:
    DataType& operator=(const DataType& rhs) {
        if (!check_invariants(rhs)) {
            globalLogger.add("Invariant violation!");  // Acquires logger lock
        }
        // ... assignment logic
        return *this;
    }
    
    bool check_invariants(const DataType&) const {
        return /* validation logic */;
    }
    
    std::string to_string() const { return "DataType{}"; }
};

// Generic thread-safe container
template<typename T>
class ConcurrentStack {
public:
    void push(const T& item) {
        std::lock_guard<std::mutex> lock(mutex_);  // Acquires stack lock first
        data_ = item;  // Calls T::operator=, may acquire logger lock
    }
    
    T get() const {
        std::lock_guard<std::mutex> lock(mutex_);  // Acquires stack lock
        return data_;
    }
    
private:
    mutable std::mutex mutex_;
    T data_;
};
```

### Deadlock Scenario
```cpp
ConcurrentStack<DataType> stack;

// Thread 1
void thread1() {
    stack.push(DataType{});  
    // Order: stack.mutex_ → globalLogger.mutex_ (if invariant fails)
}

// Thread 2
void thread2() {
    std::lock_guard<Logger> logLock(globalLogger);  // Lock logger first
    globalLogger.add(stack.get().to_string());      
    // Order: globalLogger.mutex_ → stack.mutex_
    // DEADLOCK!
}
```

---

## 4. Transactional Memory

### Concept and Motivation

Transactional Memory (TM) provides a **compositional** alternative to locks that avoids deadlock by design. Threads execute code speculatively and "commit" changes atomically.

### Key Properties

1. **Atomicity**: All operations appear to execute instantaneously
2. **Isolation**: No intermediate states visible to other threads
3. **Composability**: Nested transactions work correctly

### Transactional Memory Example

```cpp
// Current lock-based approach (prone to deadlock)
class Account {
    std::mutex mutex_;
    double balance_;
public:
    void withdraw(double amount) {
        std::lock_guard<std::mutex> lock(mutex_);
        if (balance_ >= amount)
            balance_ -= amount;
    }
};

// Proposed transactional approach (no deadlock possible)
class TransactionalAccount {
    double balance_;
public:
    void withdraw(double amount) {
        synchronized {  // All code in block executes atomically
            if (balance_ >= amount)
                balance_ -= amount;
        }
    }
    
    // Complex operations compose naturally
    static void transfer(TransactionalAccount& from, 
                         TransactionalAccount& to, 
                         double amount) {
        synchronized {
            from.withdraw(amount);
            to.deposit(amount);
            // Entire transfer is atomic, no partial states visible
        }
    }
};
```

### Rewriting the Problematic Example

```cpp
// With transactional memory - no deadlock possible
struct TransactionalLogger {
    void add(const std::string& s) {
        synchronized { log_ += s; }  // Atomic append
    }
private:
    std::string log_;
};

template<typename T>
class TransactionalStack {
    void set(const T& obj) {
        synchronized { item_ = obj; }  // Atomic assignment
    }
    
    T get() const {
        synchronized { return item_; }  // Atomic read
    }
private:
    T item_;
};

// Thread 1 & 2 can execute in any order without deadlock
// All synchronized blocks act as if protected by single global lock
// But implementation optimizes for actual parallelism
```

### Implementation Strategies

#### Software Transactional Memory (STM)
- Compiler analyzes code for conflict detection
- Maintains read/write sets per transaction
- Validates and commits atomically

#### Hardware Transactional Memory (HTM)
- CPU support (Intel TSX, IBM BlueGene)
- Cache coherence protocol tracks conflicts
- Automatic rollback on conflicts

### C++ Transactional Memory Status

1. **2014**: Ambitious Technical Specification proposed
2. **2021**: Simplified "TM-Lite" proposal (P1875R2)
3. **Current**: Experimental implementations available

```cpp
// Proposed C++ syntax (not yet standard)
int shared_counter = 0;

void increment() {
    atomic_do {  // Proposed keyword
        shared_counter++;
        if (shared_counter > MAX)
            atomic_cancel;  // Rollback transaction
    }
}
```

---

## 5. Heterogeneous Computing

### Modern Reality

Most computational power now resides in specialized processors:
- **GPUs**: Massive parallelism for graphics/ML
- **DSPs**: Signal processing
- **FPGAs**: Reconfigurable hardware
- **TPUs**: Tensor operations for AI

### C++ Heterogeneous Support

```cpp
// SYCL Example - Single-source heterogeneous programming
#include <CL/sycl.hpp>

void matrix_multiply_gpu() {
    sycl::queue q{sycl::gpu_selector{}};  // Select GPU
    
    constexpr size_t N = 1024;
    std::vector<float> A(N * N), B(N * N), C(N * N);
    
    {
        sycl::buffer<float> bufA(A.data(), sycl::range<2>(N, N));
        sycl::buffer<float> bufB(B.data(), sycl::range<2>(N, N));
        sycl::buffer<float> bufC(C.data(), sycl::range<2>(N, N));
        
        q.submit([&](sycl::handler& h) {
            auto a = bufA.get_access<sycl::access::mode::read>(h);
            auto b = bufB.get_access<sycl::access::mode::read>(h);
            auto c = bufC.get_access<sycl::access::mode::write>(h);
            
            h.parallel_for(sycl::range<2>(N, N), [=](sycl::id<2> idx) {
                int row = idx[0];
                int col = idx[1];
                float sum = 0.0f;
                for (int k = 0; k < N; ++k) {
                    sum += a[row][k] * b[k][col];
                }
                c[row][col] = sum;
            });
        });
    }  // Synchronization happens here
}
```

### Memory Models in Heterogeneous Systems

Different devices have different memory hierarchies:

```cpp
// CUDA-style memory hierarchy
__global__ void kernel(float* global_mem) {
    __shared__ float shared_mem[256];  // Block-shared memory
    
    int tid = threadIdx.x;
    int gid = blockIdx.x * blockDim.x + tid;
    
    // Coalesced global memory access
    shared_mem[tid] = global_mem[gid];
    __syncthreads();  // Block synchronization
    
    // Work with fast shared memory
    // ...
}
```

---

## 6. CMake Build System

### Overview

CMake is a **meta-build system** that generates platform-specific build files:

```
CMakeLists.txt → CMake → Makefiles/Ninja/VS Solutions → Compiler → Executable
```

### Basic CMake Structure

```cmake
# CMakeLists.txt - Minimum example
cmake_minimum_required(VERSION 3.20)
project(MyProject VERSION 1.0.0 LANGUAGES CXX)

# Set C++ standard
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Add executable
add_executable(myapp 
    src/main.cpp
    src/utils.cpp
)

# Include directories
target_include_directories(myapp PRIVATE 
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)

# Link libraries
find_package(Threads REQUIRED)
target_link_libraries(myapp PRIVATE 
    Threads::Threads
)
```

### Advanced CMake Features

#### Target-Based Design
```cmake
# Modern CMake uses targets with properties
add_library(mylib STATIC
    src/lib.cpp
)

# Public headers propagate to consumers
target_include_directories(mylib 
    PUBLIC  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
            $<INSTALL_INTERFACE:include>
    PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src
)

# Dependencies propagate automatically
add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE mylib)  # Gets include dirs automatically
```

#### Configuration and Installation
```cmake
# Configure file with version info
configure_file(
    "${PROJECT_SOURCE_DIR}/config.h.in"
    "${PROJECT_BINARY_DIR}/config.h"
)

# Installation rules
install(TARGETS mylib
    EXPORT MyLibTargets
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin
)

install(DIRECTORY include/
    DESTINATION include
)

# Package configuration for find_package()
install(EXPORT MyLibTargets
    FILE MyLibTargets.cmake
    NAMESPACE MyLib::
    DESTINATION lib/cmake/MyLib
)
```

#### Testing Integration
```cmake
# Enable testing
enable_testing()
include(CTest)

# Add test executable
add_executable(test_suite 
    tests/test_main.cpp
    tests/test_utils.cpp
)

target_link_libraries(test_suite PRIVATE mylib)

# Register tests
add_test(NAME unit_tests COMMAND test_suite)

# Custom test with properties
add_test(NAME integration_test COMMAND app --test)
set_tests_properties(integration_test PROPERTIES
    TIMEOUT 30
    ENVIRONMENT "TEST_MODE=1"
)
```

### CMake Best Practices

1. **Use Modern CMake** (3.12+)
   ```cmake
   # BAD: Old style with directory properties
   include_directories(${CMAKE_SOURCE_DIR}/include)
   add_definitions(-DDEBUG)
   
   # GOOD: Target-based properties
   target_include_directories(myapp PRIVATE include)
   target_compile_definitions(myapp PRIVATE DEBUG)
   ```

2. **Generator Expressions**
   ```cmake
   target_compile_options(myapp PRIVATE
       $<$<CONFIG:Debug>:-g -O0>
       $<$<CONFIG:Release>:-O3>
       $<$<CXX_COMPILER_ID:GNU>:-Wall -Wextra>
       $<$<CXX_COMPILER_ID:MSVC>:/W4>
   )
   ```

3. **Find or Fetch Dependencies**
   ```cmake
   include(FetchContent)
   
   FetchContent_Declare(
       googletest
       GIT_REPOSITORY https://github.com/google/googletest.git
       GIT_TAG        release-1.12.1
   )
   
   FetchContent_MakeAvailable(googletest)
   
   # Now use it
   target_link_libraries(test_suite PRIVATE gtest_main)
   ```

### Docker Integration

```dockerfile
# Dockerfile for C++ development with CMake
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y \
    cmake \
    ninja-build \
    g++ \
    git \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /workspace

# Build commands
# docker build -t cpp-dev .
# docker run -it -v $(pwd):/workspace cpp-dev
```

### Building with CMake

```bash
# Out-of-source build (recommended)
cmake -S . -B build              # Generate build system
cmake --build build              # Build the project
cmake --build build --target test # Run tests
cmake --install build --prefix /usr/local # Install

# With specific generator
cmake -S . -B build -G Ninja
cmake --build build --parallel 8

# Debug vs Release builds
cmake -S . -B build/Debug -DCMAKE_BUILD_TYPE=Debug
cmake -S . -B build/Release -DCMAKE_BUILD_TYPE=Release
```

## Enhanced Code Examples

### Complete Lock-Free Queue Using Atomics
```cpp
template<typename T>
class LockFreeQueue {
private:
    struct Node {
        std::atomic<T*> data;
        std::atomic<Node*> next;
        
        Node() : data(nullptr), next(nullptr) {}
    };
    
    std::atomic<Node*> head;
    std::atomic<Node*> tail;
    
public:
    LockFreeQueue() {
        Node* dummy = new Node;
        head.store(dummy);
        tail.store(dummy);
    }
    
    void push(T item) {
        Node* newNode = new Node;
        T* data = new T(std::move(item));
        newNode->data.store(data);
        
        Node* prevTail = tail.exchange(newNode);
        prevTail->next.store(newNode);
    }
    
    std::optional<T> pop() {
        Node* head_node = head.load();
        Node* next = head_node->next.load();
        
        if (next == nullptr) {
            return std::nullopt;
        }
        
        T* data = next->data.exchange(nullptr);
        head.store(next);
        delete head_node;
        
        if (data) {
            T value = std::move(*data);
            delete data;
            return value;
        }
        
        return std::nullopt;
    }
};
```

### Hybrid Transactional Memory Simulation
```cpp
// Simulating TM behavior with current C++
template<typename Func>
class Transaction {
private:
    struct WriteSet {
        void* addr;
        std::vector<char> old_value;
        std::vector<char> new_value;
    };
    
    std::vector<WriteSet> writes;
    std::mutex& global_lock;
    
public:
    explicit Transaction(std::mutex& lock) : global_lock(lock) {}
    
    template<typename T>
    void write(T& var, T value) {
        writes.push_back({
            &var,
            serialize(var),
            serialize(value)
        });
    }
    
    bool commit() {
        std::lock_guard<std::mutex> lock(global_lock);
        
        // Validate reads
        for (const auto& w : writes) {
            if (!validate(w)) {
                rollback();
                return false;
            }
        }
        
        // Apply writes
        for (const auto& w : writes) {
            apply(w);
        }
        
        return true;
    }
    
private:
    void rollback() { /* Restore old values */ }
    bool validate(const WriteSet&) { /* Check conflicts */ return true; }
    void apply(const WriteSet&) { /* Apply new values */ }
    
    template<typename T>
    std::vector<char> serialize(const T& obj) {
        std::vector<char> buffer(sizeof(T));
        std::memcpy(buffer.data(), &obj, sizeof(T));
        return buffer;
    }
};

// Usage
std::mutex tm_lock;

void transfer_with_tm(Account& from, Account& to, double amount) {
    bool success = false;
    
    while (!success) {
        Transaction tx(tm_lock);
        
        // Speculative execution
        double from_balance = from.get_balance();
        double to_balance = to.get_balance();
        
        if (from_balance >= amount) {
            tx.write(from.balance, from_balance - amount);
            tx.write(to.balance, to_balance + amount);
            
            success = tx.commit();
        } else {
            break;  // Insufficient funds
        }
        
        if (!success) {
            std::this_thread::yield();  // Retry after conflict
        }
    }
}
```

## Key Takeaways

1. **Invariants** are fundamental to writing correct concurrent programs - maintain them religiously
2. **Lock-ordering** prevents deadlocks but breaks down with templates and modular design
3. **Transactional Memory** offers composable concurrency without deadlock risks
4. **Heterogeneous computing** is the present and future - C++ is adapting with SYCL, executors, and more
5. **CMake** provides portable, maintainable build configuration using modern target-based design

## References and Further Reading

- Gottschlich & Boehm: "Generic Programming Needs Transactional Memory" (2013)
- ISO C++ Committee: "Transactional Memory Lite" (P1875R2, 2021)
- "Modern CMake for C++" by Rafał Świdziński
- SYCL 2020 Specification for heterogeneous computing