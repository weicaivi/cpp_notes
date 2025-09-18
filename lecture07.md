# Lecture 7 Summary

## Memory Management and Smart Pointers

### Smart Pointers and RAII

Raw pointers from `new` violate exception safety. The following code has a memory leak if the second constructor throws:

```cpp
// UNSAFE - potential memory leak
void f() {
    g(new A(), new A());  // If second A() throws, first A is leaked
}
```

#### make_unique and make_shared

These functions create objects safely and return smart pointers:

```cpp
// Safe version using smart pointers
void f() {
    auto a1 = make_unique<A>();
    auto a2 = make_unique<A>();
    g(a1.release(), a2.release());  // Transfer ownership
}

// Even better - modify g() to take smart pointers
void g(unique_ptr<A> p1, unique_ptr<A> p2);

// Now we can call safely
g(make_unique<A>(), make_unique<A>());
```

#### Getting Raw Pointers from Smart Pointers

```cpp
// For non-owning access
auto a = make_unique<A>();
f(a.get());  // f doesn't participate in lifetime management

// To transfer ownership
auto a = make_unique<A>();
g(a.release());  // a no longer owns the object
```

## Atomics and Lock-Free Programming

### C++ Atomics

Atomic types allow thread-safe operations without explicit locks:

```cpp
#include <atomic>

std::atomic<int> counter{0};

// Thread-safe operations
counter++;           // Atomic increment
counter.store(42);   // Atomic store
int val = counter.load();  // Atomic load
```

### Distributed Counter Implementation

#### Version 1: Basic Locking
```cpp
class DistributedCounter {
private:
    long long count;
    mutable std::shared_mutex mtx;
    
public:
    DistributedCounter() : count(0) {}
    
    void operator++() {
        std::unique_lock lock(mtx);
        ++count;
    }
    
    long long get() const {
        std::shared_lock lock(mtx);
        return count;
    }
};
```

#### Version 4: Lock-Free with Padding (Best Performance)
```cpp
class DistributedCounter {
private:
    struct bucket {
        std::atomic<size_t> count{0};
        char padding[256];  // Prevent false sharing
    };
    
    static constexpr size_t buckets = 128;
    std::vector<bucket> counts{buckets};
    
public:
    void operator++() {
        size_t index = std::hash<std::thread::id>()(std::this_thread::get_id()) % buckets;
        counts[index].count++;
    }
    
    size_t get() {
        return std::accumulate(counts.begin(), counts.end(), size_t{0},
            [](auto acc, auto &x) { return acc + x.count.load(); });
    }
};
```

### Lock-Free Stack Implementation
The ABA problem: Suppose thread A reads head = A. While it’s working, thread B pops A and then pushes it back (reusing the same node). Now the head is again A. Thread A sees head == A and assumes nothing changed, but in reality the stack was modified (nodes popped and pushed), which could lead to memory corruption (e.g., freeing an already-freed node)

The lock-free stack uses compare-and-exchange with an operation counter to handle the ABA problem:
1.	Load the current head.
2.	Do some work.
3.	Use compare_exchange to update the head if it hasn’t changed.

```cpp
struct StackHead {
    StackItem *link;     // Pointer to top stack item, nullptr if empty
    unsigned count;      // Operation counter, monotonically increments on every push/pop
};

class Stack {
private:
    std::atomic<StackHead> head;
    // The entire (pointer, count) pair is treated atomically
    // Every modification must update both

public:
    int pop() {
        StackHead expected = head.load();
        StackHead newHead;
        
        while (true) {
            if (expected.link == nullptr) {
                return 0;  // Empty stack
            }
            
            // Candidate new head: skip the first node
            newHead.link = expected.link->next;
            newHead.count = expected.count + 1;

            // Try to swap
            // CAS compares both the pointer and the counter
            // even if the pointer (link) is the same due to ABA, counter will have changed
            // causing CAS to fail. forcing the thread to reload and retry 
            if (head.compare_exchange_weak(expected, newHead)) {
                int value = expected.link->value;
                delete expected.link; // safe to reclaim memory
                return value;
            }
            // else: CAS failed
            // compare_exchange_weak updates expected on failure
            // 'expected' is updated to latest head, retry
        }
    }
    
    void push(int val) {
        StackItem* newItem = new StackItem(val);
        StackHead expected = head.load();
        StackHead newHead;
        
        do {
            newItem->next = expected.link;
            newHead.link = newItem;
            newHead.count = expected.count + 1; // increment counter
        } while (!head.compare_exchange_weak(expected, newHead));
        // if another thread has changed the head in the mean time, expected is updated and loop retries
    }
};
```

## Time and Chrono Library

### std::chrono Components

- **Clocks**: Sources of time information (`system_clock`, `high_resolution_clock`)
- **Durations**: Time intervals with type safety
- **Time Points**: Specific moments in time

```cpp
#include <chrono>
using namespace std::chrono;

// Duration examples
auto duration1 = seconds(30);
auto duration2 = milliseconds(500);
auto duration3 = duration1 + duration2;  // Type-safe addition

// Time point examples
auto now = system_clock::now();
auto later = now + hours(1);
auto elapsed = later - now;
```

### User-Defined Literals

C++14 introduced convenient time literals:

```cpp
using namespace std::chrono_literals;

// Much more readable
std::this_thread::sleep_for(30s);     // 30 seconds
std::this_thread::sleep_for(500ms);   // 500 milliseconds
std::this_thread::sleep_for(2h + 30min);  // 2.5 hours

// Custom temperature literals example
struct Temp { double degrees_K; };

constexpr Temp operator"" _K(long double d) { return Temp{static_cast<double>(d)}; }
constexpr Temp operator"" _C(long double d) { return Temp{d + 273.15}; }
constexpr Temp operator"" _F(long double d) { return Temp{(d - 32)*5/9 + 273.15}; }

static_assert(32.0_F.degrees_K == 273.15);  // Freezing point
```

## Inter-Thread Communication

### Condition Variables

Condition variables enable producer-consumer patterns and event-driven synchronization:

```cpp
#include <condition_variable>
#include <queue>
#include <mutex>

class ProducerConsumer {
private:
    std::queue<int> data_queue; // shared resource
    std::mutex mtx; // protects access to the queue
    std::condition_variable cv; // synchronization so consumers sleep until there's work
    
public:
    // producer thread generates data and puts them into a queue 
    void produce(int item) {
        {
            std::lock_guard lock(mtx);
            data_queue.push(item);
        } // lock is released automatically here
        cv.notify_one();  // Wake up one waiting consumer
    }
    
    int consume() {
        std::unique_lock lock(mtx);
        cv.wait(lock, [this]{ return !data_queue.empty(); });
        // cv.wait(lock, predicate)
        // puts the thread to sleep and releases the lock until signaled
        // wakes up only if the predicate is true (triggered by producer calling cv.notify_one)

        int item = data_queue.front();
        data_queue.pop();
        return item;
        // lock is released on return
    }
};
```

### Futures and Promises

#### std::async
Runs functions asynchronously and returns results via futures:

```cpp
#include <future>

int calculate_answer() {
    std::this_thread::sleep_for(1s);
    return 42;
}

int main() {
    // Start calculation in background
    auto future_result = std::async(std::launch::async, calculate_answer);
    
    // Do other work
    std::cout << "Doing other work...\n";
    
    // Get result (blocks if not ready)
    int answer = future_result.get();
    std::cout << "Answer: " << answer << std::endl;
}
```

#### Promise-Future Pairs
Enable one-way communication between threads:

```cpp
void producer_thread(std::promise<int> p) {
    // Simulate work
    std::this_thread::sleep_for(1s);
    p.set_value(42);  // Send value to consumer
}

int main() {
    std::promise<int> p; // creates a promise
    std::future<int> f = p.get_future(); // obtain the future tied to this promise
    
    std::thread t(producer_thread, std::move(p)); // launch a thread that runs producer_thread
    // must std::move(p) because promises are non-copyable (each one is unique — only one owner) 
    
    // Wait for result, blocks the main thread until set_value() is called
    // f.get() here is both synchronization (wait until ready) and communication (retrieve the result)
    int result = f.get(); 
    std::cout << "Received: " << result << std::endl;
    
    t.join(); // Wait for the producer thread to finish before exiting
}
```

#### Packaged Tasks
Wrap callable objects to get future results:

```cpp
int multiply(int a, int b) { return a * b; }

int main() {
    std::packaged_task<int(int, int)> task(multiply);
    std::future<int> result = task.get_future();
    
    std::thread t(std::move(task), 6, 7);
    
    std::cout << "Result: " << result.get() << std::endl;  // 42
    t.join();
}
```

### Parallel Accumulate Implementation

```cpp
template<typename Iterator, typename T>
T parallel_accumulate(Iterator first, Iterator last, T init) {
    const unsigned long length = std::distance(first, last);
    if (!length) return init;
    
    const unsigned long min_per_thread = 25;
    const unsigned long max_threads = (length + min_per_thread - 1) / min_per_thread;
    const unsigned long hardware_threads = std::thread::hardware_concurrency();
    const unsigned long num_threads = std::min(
        hardware_threads != 0 ? hardware_threads : 2, max_threads);
    
    const unsigned long block_size = length / num_threads;
    
    std::vector<std::future<T>> futures(num_threads - 1);
    Iterator block_start = first;
    
    // Launch async tasks for all but the last block
    for (unsigned long i = 0; i < (num_threads - 1); ++i) {
        Iterator block_end = block_start;
        std::advance(block_end, block_size);
        
        futures[i] = std::async(std::launch::async,
            std::accumulate<Iterator, T>, block_start, block_end, T{});
        
        block_start = block_end;
    }
    
    // Process the final block in current thread
    T last_result = std::accumulate(block_start, last, init);
    
    // Collect results from other threads
    T result = last_result;
    for (auto& f : futures) {
        result += f.get();
    }
    
    return result;
}
```

## Error Handling with std::expected

`std::expected<T, E>` as a modern alternative to exceptions for error handling.

### Basic Usage

```cpp
#include <expected>  // C++23

std::expected<double, std::string> divide(double a, double b) {
    if (b == 0.0) {
        return std::unexpected("Division by zero");
    }
    return a / b;
}

int main() {
    auto result = divide(10.0, 2.0);
    
    if (result) {
        std::cout << "Result: " << *result << std::endl;  // 5.0
    } else {
        std::cout << "Error: " << result.error() << std::endl;
    }
    
    // Or use value() which throws if no value
    try {
        double val = divide(10.0, 0.0).value();
    } catch (const std::string& error) {
        std::cout << "Caught: " << error << std::endl;
    }
}
```

### Advantages of std::expected

1. **Type Safety**: Errors are part of the type system
2. **Performance**: No stack unwinding overhead
3. **Flexibility**: Can choose local or centralized error handling
4. **Composability**: Can be chained and transformed

```cpp
// Chaining operations
auto safe_calculation(double x) -> std::expected<double, std::string> {
    return divide(x, 2.0)
        .and_then([](double result) { return divide(result, 3.0); })
        .and_then([](double result) { return divide(result, 4.0); });
}
```

## Key Takeaways

1. **Memory Management**: Use smart pointers (`unique_ptr`, `shared_ptr`) instead of raw pointers from `new`
2. **Concurrency**: Atomics provide lock-free programming but require careful design
3. **Time Handling**: `std::chrono` provides type-safe time operations with user-defined literals
4. **Thread Communication**: Use condition variables, futures, and promises for coordination
5. **Error Handling**: Consider `std::expected` as an alternative to exceptions for better local error handling
6. **Performance**: Lock-free data structures can improve performance but introduce complexity

