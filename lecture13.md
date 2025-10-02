# Lecture 13 Summary

## Iterators - The C++ Pointer Abstraction

### Core Concept
Iterators are C++'s lightweight abstraction that generalizes pointer arithmetic to work with any container, not just arrays. They provide a uniform interface for traversing different data structures while preventing buffer overruns (which caused HeartBleed).

### Fundamental Iterator Operations

```cpp
// Essential iterator operations
template<typename Iterator>
void demonstrate_iterator_ops(Iterator it) {
    auto value = *it;           // Dereference to get element
    ++it;                       // Advance to next element
    --it;                       // Move to previous (if bidirectional)
    it += 5;                    // Jump forward (if random access)
    
    Iterator it2 = it;          // Copy assignment
    if (it == it2) { }         // Equality comparison
    if (it != it2) { }         // Inequality comparison
}
```

### Iterator Categories Hierarchy

```cpp
// Iterator tag hierarchy (inheritance represents specialization)
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag : public input_iterator_tag {};
struct bidirectional_iterator_tag : public forward_iterator_tag {};
struct random_access_iterator_tag : public bidirectional_iterator_tag {};
struct contiguous_iterator_tag : public random_access_iterator_tag {};  // C++20

// Example: Different containers provide different iterator capabilities
std::forward_list<int>::iterator;     // Forward only
std::list<int>::iterator;              // Bidirectional
std::vector<int>::iterator;            // Random access (contiguous in C++20)
std::istream_iterator<int>;            // Input only
std::ostream_iterator<int>;            // Output only
```

### Why Iterator Categories Matter

```cpp
#include <algorithm>
#include <vector>
#include <list>

// This works - vector has random access iterators
std::vector<int> vec = {3, 1, 4, 1, 5};
std::sort(vec.begin(), vec.end());  // OK!

// This fails - list only has bidirectional iterators
std::list<int> lst = {3, 1, 4, 1, 5};
// std::sort(lst.begin(), lst.end());  // Compilation error!
lst.sort();  // Use list's member function instead
```

---

## Iterator Traits and Type Deduction

### The Problem: Pointers as Iterators
Pointers should work as iterators, but they're not classes and can't have member types:

```cpp
// This works for class iterators
template<typename T>
void process(T it) {
    using value_type = typename T::value_type;  // OK for classes
}

// But fails for pointers
int* ptr;
// process(ptr);  // Error: int* has no member 'value_type'
```

### Solution: iterator_traits Indirection

```cpp
namespace std {
    // Primary template - forwards to iterator's members
    template<typename Iterator>
    struct iterator_traits {
        using value_type = typename Iterator::value_type;
        using difference_type = typename Iterator::difference_type;
        using pointer = typename Iterator::pointer;
        using reference = typename Iterator::reference;
        using iterator_category = typename Iterator::iterator_category;
    };
    
    // Specialization for pointers
    template<typename T>
    struct iterator_traits<T*> {
        using value_type = T;
        using difference_type = ptrdiff_t;
        using pointer = T*;
        using reference = T&;
        using iterator_category = contiguous_iterator_tag;
    };
}

// Now works for both class iterators and pointers
template<typename Iterator>
void process(Iterator it) {
    using value_type = typename std::iterator_traits<Iterator>::value_type;
    value_type val = *it;  // Works for both vector::iterator and int*
}
```

### Creating Custom Iterators

```cpp
template<typename T>
class MyContainer {
public:
    class iterator {
    public:
        // Required member types for iterator_traits
        using value_type = T;
        using difference_type = ptrdiff_t;
        using pointer = T*;
        using reference = T&;
        using iterator_category = std::random_access_iterator_tag;
        
    private:
        T* ptr;
        
    public:
        iterator(T* p) : ptr(p) {}
        
        // Essential operators
        reference operator*() const { return *ptr; }
        pointer operator->() const { return ptr; }
        iterator& operator++() { ++ptr; return *this; }
        iterator operator++(int) { iterator tmp = *this; ++ptr; return tmp; }
        iterator& operator--() { --ptr; return *this; }
        iterator operator--(int) { iterator tmp = *this; --ptr; return tmp; }
        
        // Random access operators
        iterator& operator+=(difference_type n) { ptr += n; return *this; }
        iterator& operator-=(difference_type n) { ptr -= n; return *this; }
        reference operator[](difference_type n) const { return ptr[n]; }
        
        // Comparison operators
        bool operator==(const iterator& other) const { return ptr == other.ptr; }
        bool operator!=(const iterator& other) const { return ptr != other.ptr; }
        bool operator<(const iterator& other) const { return ptr < other.ptr; }
        
        // Arithmetic operators
        iterator operator+(difference_type n) const { return iterator(ptr + n); }
        iterator operator-(difference_type n) const { return iterator(ptr - n); }
        difference_type operator-(const iterator& other) const { return ptr - other.ptr; }
    };
    
    // Container interface
    iterator begin() { return iterator(data); }
    iterator end() { return iterator(data + size); }
    
private:
    T* data;
    size_t size;
};
```

### CTAD with Iterator Traits

```cpp
// How does vector deduce element type from iterator pair?
std::list<int> li = {1, 2, 3};
std::vector vi(li.begin(), li.end());  // Deduces vector<int>

// Through deduction guides using iterator_traits!
template<class InputIt>
vector(InputIt, InputIt) 
    -> vector<typename std::iterator_traits<InputIt>::value_type>;
```

---

## C++20 Ranges - Modern Algorithm Design

### Problems with Traditional STL Algorithms

1. **Iterator pairs are verbose**: Always need `.begin()` and `.end()`
2. **Not composable**: Can't chain algorithms without intermediate containers
3. **Materialization overhead**: Each step creates temporary containers
4. **Wasted computation**: Can't know to stop early
5. **No infinite sequences**: Must process entire range
6. **Poor type inference**: Intermediate containers need explicit types

### Traditional Approach (Pre-C++20)

```cpp
// Painful traditional approach
std::vector<int> data = {9, 3, 2, 8, 5, 6, 7, 4, 1, 10};

// Step 1: Sort (modifies original)
std::sort(data.begin(), data.end());

// Step 2: Filter evens (needs intermediate container)
std::vector<int> filtered;
std::copy_if(data.begin(), data.end(), 
             std::back_inserter(filtered),
             [](int n) { return n % 2 == 0; });

// Step 3: Transform (another intermediate)
std::vector<int> transformed;
std::transform(filtered.begin(), filtered.end(),
               std::back_inserter(transformed),
               [](int n) { return n * 10; });

// Step 4: Take first 3 (yet another intermediate)
std::vector<int> result(transformed.begin(), 
                        transformed.begin() + std::min(3UL, transformed.size()));

// Step 5: Print
std::copy(result.begin(), result.end(),
          std::ostream_iterator<int>(std::cout, " "));
```

### Modern Ranges Approach (C++20/23)

```cpp
#include <ranges>
#include <algorithm>

namespace views = std::ranges::views;
namespace ranges = std::ranges;

std::vector<int> data = {9, 3, 2, 8, 5, 6, 7, 4, 1, 10};

// Elegant pipeline - lazy evaluation, no intermediates!
auto result = data
    | views::sort                              // View adapter
    | views::filter([](int n) { return n % 2 == 0; })
    | views::transform([](int n) { return n * 10; })
    | views::take(3)
    | ranges::to<std::vector>();              // C++23

// Or print directly without materializing
for (int n : data | views::filter([](int n) { return n % 2 == 0; })
                   | views::take(3)) {
    std::cout << n << ' ';
}
```

### Infinite Ranges and Lazy Evaluation

```cpp
#include <ranges>

// Generate infinite sequence of triangular numbers
auto triangular_numbers = std::views::iota(1)  // 1, 2, 3, 4, ...
    | std::views::transform([](int n) { 
          return n * (n + 1) / 2; 
      });

// Take only what we need (lazy - only computes 10 values)
auto first_10 = triangular_numbers 
    | std::views::take(10)
    | std::ranges::to<std::vector>();  // {1, 3, 6, 10, 15, 21, 28, 36, 45, 55}

// Find first triangular number > 1000
auto first_large = std::ranges::find_if(triangular_numbers,
                                        [](int n) { return n > 1000; });
```

### Advanced Range Compositions

```cpp
// Zip multiple ranges together
auto indices = std::views::iota(0);
auto values = std::vector{10, 20, 30, 40};
auto pairs = std::views::zip(indices, values);  // [(0,10), (1,20), (2,30), (3,40)]

// Create a lookup map from zipped ranges
auto triangular_lookup = std::views::iota(1, 11)
    | std::views::transform([](int n) { 
          return std::pair{n, n * (n + 1) / 2}; 
      })
    | std::ranges::to<std::map>();  // map: 1->1, 2->3, 3->6, ...

// Complex pipeline with multiple data sources
auto process_data = [](auto& customers, auto& orders) {
    return std::views::zip(customers, orders)
        | std::views::filter([](const auto& pair) {
              auto& [customer, order] = pair;
              return order.amount > 100.0;
          })
        | std::views::transform([](const auto& pair) {
              auto& [customer, order] = pair;
              return customer.name + ": $" + std::to_string(order.amount);
          })
        | std::views::take(10)
        | std::ranges::to<std::vector<std::string>>();
};
```

### Custom Range Adaptors

```cpp
// Create custom range adaptor for sliding window
template<std::ranges::forward_range R>
class sliding_window_view {
    R base_;
    std::size_t window_size_;
    
public:
    sliding_window_view(R base, std::size_t n) 
        : base_(std::move(base)), window_size_(n) {}
    
    class iterator {
        using base_iterator = std::ranges::iterator_t<R>;
        base_iterator current_;
        base_iterator end_;
        std::size_t window_size_;
        
    public:
        auto operator*() const {
            return std::ranges::subrange(current_, 
                std::next(current_, std::min(window_size_, 
                         std::distance(current_, end_))));
        }
        // ... other iterator operations
    };
    
    auto begin() { return iterator{std::ranges::begin(base_), 
                                   std::ranges::end(base_), 
                                   window_size_}; }
    auto end() { /* ... */ }
};

// Usage
auto data = std::vector{1, 2, 3, 4, 5};
auto windows = sliding_window_view(data, 3);
// Produces: [1,2,3], [2,3,4], [3,4,5]
```

---

## Memory Models and Concurrency

### C++ Memory Model Basics

The C++ memory model defines how threads interact through memory. Key concepts:

1. **Sequential Consistency**: Default, most restrictive, easiest to reason about
2. **Acquire-Release**: Lighter synchronization for producer-consumer patterns
3. **Relaxed Ordering**: No synchronization, only atomicity

### Atomic Operations and Memory Ordering

```cpp
#include <atomic>
#include <thread>

// Sequential consistency (default)
std::atomic<int> x{0}, y{0};
std::atomic<int> r1{0}, r2{0};

void thread1() {
    x.store(1);                           // Sequentially consistent
    r1.store(y.load());
}

void thread2() {
    y.store(1);
    r2.store(x.load());
}
// Guarantee: Cannot have r1 == 0 && r2 == 0

// Acquire-Release semantics
std::atomic<bool> ready{false};
int data{0};

void producer() {
    data = 42;                            // Non-atomic write
    ready.store(true, std::memory_order_release);  // Release fence
}

void consumer() {
    while (!ready.load(std::memory_order_acquire)) {  // Acquire fence
        std::this_thread::yield();
    }
    assert(data == 42);  // Guaranteed to see producer's write
}

// Relaxed ordering (careful - no synchronization!)
std::atomic<int> counter{0};

void increment() {
    counter.fetch_add(1, std::memory_order_relaxed);  // Just atomicity
}
```

### Advanced Promise/Future Implementation

```cpp
template<typename T>
class MyPromise {
private:
    struct SharedState {
        std::optional<std::variant<T, std::exception_ptr>> value;
        std::atomic_flag promise_fulfilled;
        std::mutex mutex;
        std::condition_variable cv;
    };
    
    std::shared_ptr<SharedState> state_;
    
public:
    MyPromise() : state_(std::make_shared<SharedState>()) {}
    
    void set_value(T val) {
        {
            std::lock_guard lock(state_->mutex);
            if (state_->value) {
                throw std::future_error(std::future_errc::promise_already_satisfied);
            }
            state_->value = std::move(val);
        }
        state_->promise_fulfilled.test_and_set();
        state_->promise_fulfilled.notify_one();
        state_->cv.notify_all();
    }
    
    void set_exception(std::exception_ptr e) {
        {
            std::lock_guard lock(state_->mutex);
            if (state_->value) {
                throw std::future_error(std::future_errc::promise_already_satisfied);
            }
            state_->value = e;
        }
        state_->promise_fulfilled.test_and_set();
        state_->promise_fulfilled.notify_one();
        state_->cv.notify_all();
    }
    
    MyFuture<T> get_future();
};

template<typename T>
class MyFuture {
private:
    std::shared_ptr<typename MyPromise<T>::SharedState> state_;
    
public:
    T get() {
        // Use atomic flag for efficient waiting (C++20)
        state_->promise_fulfilled.wait(false);
        
        return std::visit(overload{
            [](T& val) { return std::move(val); },
            [](std::exception_ptr& e) -> T { std::rethrow_exception(e); }
        }, *state_->value);
    }
    
    bool is_ready() const {
        return state_->promise_fulfilled.test();
    }
    
    template<typename Rep, typename Period>
    std::future_status wait_for(const std::chrono::duration<Rep, Period>& timeout) {
        std::unique_lock lock(state_->mutex);
        if (state_->cv.wait_for(lock, timeout, [this] { return state_->value.has_value(); })) {
            return std::future_status::ready;
        }
        return std::future_status::timeout;
    }
};
```

---

## Lock Management and RAII

### Evolution of Lock Management

```cpp
// BAD: Manual lock management (error-prone)
std::mutex mtx;
void bad_example() {
    mtx.lock();
    // If this throws, mutex is never unlocked!
    potentially_throwing_operation();
    mtx.unlock();  // May never execute
}

// BETTER: Basic RAII with lock_guard
void better_example() {
    std::lock_guard lock(mtx);  // CTAD deduces std::lock_guard<std::mutex>
    potentially_throwing_operation();
    // Destructor automatically unlocks
}

// BEST: Flexible RAII with unique_lock
void best_example() {
    std::unique_lock lock(mtx);
    potentially_throwing_operation();
    
    // Can manually unlock/relock if needed
    lock.unlock();
    do_something_without_lock();
    lock.lock();
    do_something_with_lock();
}
```

### Advanced Lock Patterns

```cpp
// Avoid deadlock with scoped_lock (C++17)
class BankAccount {
    mutable std::mutex mtx_;
    double balance_;
    
public:
    void transfer_to(BankAccount& other, double amount) {
        // Locks both mutexes without deadlock!
        std::scoped_lock lock(mtx_, other.mtx_);
        
        if (balance_ >= amount) {
            balance_ -= amount;
            other.balance_ += amount;
        }
    }
    
    double get_balance() const {
        std::lock_guard lock(mtx_);
        return balance_;
    }
};

// Reader-Writer locks with shared_mutex
class Cache {
    mutable std::shared_mutex mtx_;
    std::map<int, std::string> data_;
    
public:
    std::string read(int key) const {
        std::shared_lock lock(mtx_);  // Multiple readers OK
        auto it = data_.find(key);
        return it != data_.end() ? it->second : "";
    }
    
    void write(int key, std::string value) {
        std::unique_lock lock(mtx_);  // Exclusive writer access
        data_[key] = std::move(value);
    }
    
    void update(int key, std::function<void(std::string&)> updater) {
        std::unique_lock lock(mtx_);
        updater(data_[key]);
    }
};

// Defer lock pattern
void process_with_timeout() {
    std::timed_mutex mtx;
    std::unique_lock lock(mtx, std::defer_lock);  // Don't lock yet
    
    if (lock.try_lock_for(std::chrono::milliseconds(100))) {
        // Got the lock within timeout
        do_work();
    } else {
        // Timeout - handle gracefully
        handle_timeout();
    }
}
```

### Lock-Free Programming

```cpp
// Lock-free stack using atomic operations
template<typename T>
class LockFreeStack {
private:
    struct Node {
        T data;
        std::atomic<Node*> next;
        Node(T val) : data(std::move(val)), next(nullptr) {}
    };
    
    std::atomic<Node*> head_{nullptr};
    
public:
    void push(T val) {
        Node* new_node = new Node(std::move(val));
        Node* old_head = head_.load(std::memory_order_relaxed);
        
        do {
            new_node->next.store(old_head, std::memory_order_relaxed);
        } while (!head_.compare_exchange_weak(old_head, new_node,
                                              std::memory_order_release,
                                              std::memory_order_relaxed));
    }
    
    std::optional<T> pop() {
        Node* old_head = head_.load(std::memory_order_relaxed);
        
        while (old_head && 
               !head_.compare_exchange_weak(old_head, 
                                           old_head->next.load(std::memory_order_relaxed),
                                           std::memory_order_acquire,
                                           std::memory_order_relaxed)) {
            // Retry on failure
        }
        
        if (old_head) {
            T result = std::move(old_head->data);
            delete old_head;
            return result;
        }
        return std::nullopt;
    }
    
    ~LockFreeStack() {
        while (pop()) { }  // Clean up remaining nodes
    }
};
```

---

## Complete Code Examples

### Factory Pattern with Variadic Templates

The lecture's factory implementation demonstrates advanced template metaprogramming:

```cpp
#include <memory>
#include <tuple>

namespace advanced {

// Tag type for type deduction
template<typename T>
struct TypeTag {};

// Abstract creator interface
template<typename T>
struct AbstractCreator {
    virtual std::unique_ptr<T> doCreate(TypeTag<T>&&) = 0;
    virtual ~AbstractCreator() = default;
};

// Abstract factory inheriting from multiple creators
template<typename... Products>
struct AbstractFactory : public AbstractCreator<Products>... {
    template<typename Product>
    std::unique_ptr<Product> create() {
        AbstractCreator<Product>& creator = *this;
        return creator.doCreate(TypeTag<Product>{});
    }
};

// Concrete creator for specific product mapping
template<typename Factory, typename AbstractProduct, typename ConcreteProduct>
struct ConcreteCreator : virtual public Factory {
    std::unique_ptr<AbstractProduct> doCreate(TypeTag<AbstractProduct>&&) override {
        return std::make_unique<ConcreteProduct>();
    }
};

// Concrete factory implementation
template<typename Factory, typename... ConcreteProducts>
struct ConcreteFactory;

template<typename... AbstractProducts, typename... ConcreteProducts>
struct ConcreteFactory<AbstractFactory<AbstractProducts...>, ConcreteProducts...>
    : public ConcreteCreator<AbstractFactory<AbstractProducts...>, 
                            AbstractProducts, ConcreteProducts>... {
    static_assert(sizeof...(AbstractProducts) == sizeof...(ConcreteProducts),
                  "Must provide concrete type for each abstract type");
};

// Example usage
struct Engine { 
    double horsepower; 
    Engine(double hp = 100) : horsepower(hp) {}
};

struct Vehicle { 
    virtual void start() = 0; 
    virtual ~Vehicle() = default;
};

struct Component { 
    virtual void activate() = 0; 
    virtual ~Component() = default;
};

struct Car : Vehicle { 
    void start() override { std::cout << "Car starting\n"; }
};

struct Radio : Component { 
    void activate() override { std::cout << "Radio playing\n"; }
};

using VehicleFactory = AbstractFactory<Vehicle, Component>;
using CarFactory = ConcreteFactory<VehicleFactory, Car, Radio>;

void demo() {
    std::unique_ptr<VehicleFactory> factory = std::make_unique<CarFactory>();
    auto vehicle = factory->create<Vehicle>();
    auto component = factory->create<Component>();
    
    vehicle->start();      // "Car starting"
    component->activate(); // "Radio playing"
}

}  // namespace advanced
```

### Advanced Type Traits Implementation

```cpp
namespace traits {

// Remove all pointers recursively
template<typename T>
struct remove_all_pointers {
    using type = T;
};

template<typename T>
struct remove_all_pointers<T*> : remove_all_pointers<T> {};

template<typename T>
using remove_all_pointers_t = typename remove_all_pointers<T>::type;

// Custom is_reference implementation
template<typename T>
struct is_reference : std::false_type {};

template<typename T>
struct is_reference<T&> : std::true_type {};

template<typename T>
struct is_reference<T&&> : std::true_type {};

template<typename T>
inline constexpr bool is_reference_v = is_reference<T>::value;

// Advanced: detect member function
template<typename T, typename = void>
struct has_size : std::false_type {};

template<typename T>
struct has_size<T, std::void_t<decltype(std::declval<T>().size())>> 
    : std::true_type {};

template<typename T>
inline constexpr bool has_size_v = has_size<T>::value;

// Conditional type selection
template<bool Condition, typename TrueType, typename FalseType>
struct conditional {
    using type = TrueType;
};

template<typename TrueType, typename FalseType>
struct conditional<false, TrueType, FalseType> {
    using type = FalseType;
};

template<bool Condition, typename TrueType, typename FalseType>
using conditional_t = typename conditional<Condition, TrueType, FalseType>::type;

// Example usage with SFINAE
template<typename Container>
auto get_size(const Container& c) -> std::enable_if_t<has_size_v<Container>, size_t> {
    return c.size();
}

template<typename Array, size_t N>
size_t get_size(const Array(&)[N]) {
    return N;
}

}  // namespace traits
```

### Output Iterator Implementation (HW Solution)

```cpp
template<typename CharT = char, typename Traits = std::char_traits<CharT>>
class ostream_joiner {
public:
    using iterator_category = std::output_iterator_tag;
    using value_type = void;
    using difference_type = void;
    using pointer = void;
    using reference = void;
    using char_type = CharT;
    using traits_type = Traits;
    using ostream_type = std::basic_ostream<CharT, Traits>;
    
private:
    ostream_type* stream_;
    const CharT* delimiter_;
    bool first_;
    
    // Proxy class for operator=
    class proxy {
        ostream_joiner* parent_;
    public:
        explicit proxy(ostream_joiner* p) : parent_(p) {}
        
        template<typename T>
        proxy& operator=(const T& value) {
            if (!parent_->first_) {
                *parent_->stream_ << parent_->delimiter_;
            }
            *parent_->stream_ << value;
            parent_->first_ = false;
            return *this;
        }
    };
    
public:
    ostream_joiner(ostream_type& stream, const CharT* delim)
        : stream_(&stream), delimiter_(delim), first_(true) {}
    
    proxy operator*() { return proxy(this); }
    ostream_joiner& operator++() { return *this; }
    ostream_joiner& operator++(int) { return *this; }
};

// Deduction guide for CTAD
template<typename CharT, typename Traits>
ostream_joiner(std::basic_ostream<CharT, Traits>&, const CharT*) 
    -> ostream_joiner<CharT, Traits>;

// Helper function for easy creation
template<typename CharT, typename Traits = std::char_traits<CharT>>
auto make_ostream_joiner(std::basic_ostream<CharT, Traits>& stream, 
                         const CharT* delimiter) {
    return ostream_joiner<CharT, Traits>(stream, delimiter);
}

// Bonus: operator<< for containers
template<typename Container>
std::enable_if_t<traits::has_size_v<Container>, std::ostream&>
operator<<(std::ostream& os, const Container& container) {
    std::copy(std::begin(container), std::end(container),
              ostream_joiner(os, ", "));
    return os;
}

// Demo
void demo_joiner() {
    std::vector v = {1, 2, 3, 4, 5};
    
    // Traditional: trailing delimiter
    std::copy(v.begin(), v.end(), 
              std::ostream_iterator<int>(std::cout, ", "));
    std::cout << "\n";  // Output: 1, 2, 3, 4, 5, 
    
    // Better: no trailing delimiter
    std::copy(v.begin(), v.end(),
              ostream_joiner(std::cout, ", "));
    std::cout << "\n";  // Output: 1, 2, 3, 4, 5
    
    // With operator<<
    std::cout << v << "\n";  // Output: 1, 2, 3, 4, 5
}
```

## Key Takeaways

1. **Iterators** provide a uniform interface for traversing containers while maintaining type safety through traits
2. **Ranges** revolutionize algorithm composition with lazy evaluation and pipeline syntax
3. **Memory ordering** allows fine-tuned control over thread synchronization for performance
4. **RAII lock management** ensures exception safety and prevents deadlocks
5. **Template metaprogramming** enables compile-time polymorphism without runtime overhead

## Common Pitfalls to Avoid

- Don't assume all iterators support all operations (check iterator category)
- Avoid creating intermediate containers when ranges can provide views
- Never use manual lock/unlock - always use RAII wrappers
- Be cautious with relaxed memory ordering - sequential consistency is safest
- Remember that iterator_traits is necessary for pointer compatibility