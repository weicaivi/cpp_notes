# C++ Lecture 14: Advanced Factory Patterns, Memory Models, and C++ Standardization

## Advanced Factory Pattern with Variadics

### Evolution from Basic to Advanced Factory

The lecture presents three levels of factory sophistication:

1. **Basic Factory**: Hardcoded virtual methods for each product
2. **Template-Based Factory**: Uses variadic templates to generate factories
3. **Parameterized Factory**: Adds constructor arguments and template specialization

### Core Components of the Advanced Factory

```cpp
namespace mpcs {
    // Type tag for overload resolution
    template<typename T>
    struct TT {};
    
    // Abstract creator with two variants:
    // 1. No-argument version
    template<typename T>
    struct abstract_creator {
        virtual std::unique_ptr<T> doCreate(TT<T>&&) = 0;
    };
    
    // 2. With-arguments version using function type syntax
    template<typename Result, typename... Args>
    struct abstract_creator<Result(Args...)> {
        virtual std::unique_ptr<Result> doCreate(TT<Result>&&, Args...) = 0;
    };
}
```

### The Abstract Factory Template

```cpp
// Inherits from multiple abstract_creators via pack expansion
template<typename... Ts>
struct abstract_factory : public abstract_creator<Ts>... {
    // Bring all doCreate overloads into scope
    using abstract_creator<Ts>::doCreate...;
    
    // Generic create method with perfect forwarding
    template<class U, typename... Args> 
    std::unique_ptr<U> create(Args&&... args) {
        return doCreate(TT<U>(), std::forward<Args>(args)...);
    }
    
    virtual ~abstract_factory() = default;
};
```

### Concrete Creator Implementation

```cpp
// Basic concrete creator (no arguments)
template<typename AbstractFactory, typename Abstract, typename Concrete>
struct concrete_creator : virtual public AbstractFactory {
    std::unique_ptr<Abstract> doCreate(TT<Abstract>&&) override {
        return std::make_unique<Concrete>();
    }
};

// Advanced concrete creator (with constructor arguments)
template<typename AbstractFactory, typename Result, typename... Args, typename Concrete>
struct concrete_creator<AbstractFactory, Result(Args...), Concrete> 
    : virtual public AbstractFactory {
    std::unique_ptr<Result> doCreate(TT<Result>&&, Args... args) override {
        return std::make_unique<Concrete>(args...);
    }
};
```

### Concrete Factory Assembly

```cpp
// Concrete factory inherits from multiple concrete_creators
template<typename AbstractFactory, typename... ConcreteTypes>
struct concrete_factory;

template<typename... AbstractTypes, typename... ConcreteTypes>
struct concrete_factory<abstract_factory<AbstractTypes...>, ConcreteTypes...>
    : public concrete_creator<abstract_factory<AbstractTypes...>, 
                              AbstractTypes, ConcreteTypes>... {
    static_assert(sizeof...(AbstractTypes) == sizeof...(ConcreteTypes),
                  "Must provide one concrete type for each abstract type");
};
```

### Parameterized Factory Pattern

```cpp
// Allows using template specialization for product families
template<typename AbstractFactory, template<typename> class Concrete>
struct parameterized_factory;

template<template<typename> class Concrete, typename... AbstractTypes>
struct parameterized_factory<abstract_factory<AbstractTypes...>, Concrete>
    : public concrete_factory<abstract_factory<AbstractTypes...>, 
                             Concrete<AbstractTypes>...> {};

// Usage: Define product family templates
template<typename T> struct model;
template<typename T> struct real;

// Specialize for each abstract type
template<>
struct model<locomotive> : public locomotive {
    void blow_horn() override { 
        std::cout << "quiet honk\n"; 
    }
};

template<>
struct real<locomotive> : public locomotive {
    void blow_horn() override { 
        std::cout << "LOUD HONK!\n"; 
    }
};

// Create factories for entire families
using model_factory = parameterized_factory<train_factory, model>;
using real_factory = parameterized_factory<train_factory, real>;
```

### Factory with Constructor Arguments

```cpp
// Define abstract factory with constructor signatures
using flexible_factory = abstract_factory<
    locomotive(Engine),      // Takes Engine parameter
    freight_car(long),       // Takes capacity parameter
    caboose                  // No parameters
>;

// Usage
auto factory = std::make_unique<flexible_model_factory>();
auto loco = factory->create<locomotive>(Engine(800.0));
auto freight = factory->create<freight_car>(1000L);
auto cab = factory->create<caboose>();
```

---

## Memory Ordering and Happens-Before

### The Happens-Before Relationship

The happens-before relationship is fundamental to understanding C++ concurrency:

1. **Within a thread**: Operations have a sequenced-before relationship based on source order
2. **Between threads**: No inherent happens-before without synchronization
3. **Through synchronization**: Atomics and locks establish happens-before across threads

### The As-If Rule

```cpp
// The compiler/hardware can reorder these:
a = 2;
b = 3;
c = a + b;  // But not this - depends on previous assignments

// The standard says implementations must behave "as if" 
// they executed in order, but can optimize freely when unobservable
```

### Sequential Consistency Example

```cpp
std::atomic<int> a{0}, b{0};

// Thread 1
void thread1() {
    a.store(1);  // A
    b.store(2);  // B
}

// Thread 2  
void thread2() {
    int y = b.load();  // C
    int x = a.load();  // D
}

// If C reads 2, then we have: A → B → C → D
// Therefore D must read 1 (sequential consistency)
```

### Release-Acquire Semantics

```cpp
std::atomic<bool> ready{false};
int data{0};

// Producer thread
void producer() {
    data = 42;  // A: Non-atomic write
    ready.store(true, std::memory_order_release);  // B: Release
}

// Consumer thread
void consumer() {
    while (!ready.load(std::memory_order_acquire)) {  // C: Acquire
        std::this_thread::yield();
    }
    assert(data == 42);  // D: Guaranteed to see A
}

// Release-acquire establishes: A happens-before D
// But only for these two threads - others have no guarantees
```

### Memory Ordering Impact on Performance

```cpp
// Most restrictive (slowest, safest)
x.store(1, std::memory_order_seq_cst);  // Default

// Medium restriction
x.store(1, std::memory_order_release);
y.load(std::memory_order_acquire);

// Least restrictive (fastest, most dangerous)
x.store(1, std::memory_order_relaxed);  // Only atomicity, no ordering
```

---

## The Spaceship Operator (C++20)

### Basic Usage

The spaceship operator `<=>` generates all six comparison operators from a single implementation:

```cpp
#include <compare>

struct Point {
    int x, y;
    
    // Default implementation compares members lexicographically
    auto operator<=>(const Point&) const = default;
};

// Automatically generates: ==, !=, <, <=, >, >=
Point p1{1, 2}, p2{2, 1};
bool less = p1 < p2;        // true (1 < 2 for x)
bool equal = p1 == p2;       // false
bool greater = p1 > p2;      // false
```

### Custom Implementation

```cpp
struct Rectangle {
    int width, height;
    
    // Compare by area
    std::strong_ordering operator<=>(const Rectangle& other) const {
        int area1 = width * height;
        int area2 = other.width * other.height;
        return area1 <=> area2;  // Spaceship on ints
    }
    
    // Must define == separately if custom <=>
    bool operator==(const Rectangle& other) const {
        return width == other.width && height == other.height;
    }
};
```

### Ordering Categories

```cpp
// Strong ordering: Total order with substitutability
struct Version {
    int major, minor, patch;
    std::strong_ordering operator<=>(const Version&) const = default;
};

// Weak ordering: Total order without substitutability  
struct CaseInsensitiveString {
    std::string s;
    std::weak_ordering operator<=>(const CaseInsensitiveString& other) const {
        return strcasecmp(s.c_str(), other.s.c_str()) <=> 0;
    }
};

// Partial ordering: Not all values comparable
struct PartialPoint {
    float x, y;  // NaN values create incomparability
    std::partial_ordering operator<=>(const PartialPoint&) const = default;
};
```

### Complex Example with Partial Ordering

```cpp
struct DominancePoint {
    int x, y;
    
    // Only comparable if one dominates the other
    std::partial_ordering operator<=>(const DominancePoint& p) const {
        if (x < p.x && y < p.y) 
            return std::partial_ordering::less;
        if (x == p.x && y == p.y) 
            return std::partial_ordering::equivalent;
        if (x > p.x && y > p.y) 
            return std::partial_ordering::greater;
        return std::partial_ordering::unordered;  // Incomparable
    }
    
    // Need == for partial ordering
    bool operator==(const DominancePoint& p) const {
        return x == p.x && y == p.y;
    }
};

// Usage
DominancePoint p1{1, 3}, p2{2, 2};
if (auto cmp = p1 <=> p2; cmp < 0) {
    std::cout << "p1 dominates\n";
} else if (cmp > 0) {
    std::cout << "p2 dominates\n";
} else if (cmp == 0) {
    std::cout << "equal\n";
} else {
    std::cout << "incomparable\n";  // This case!
}
```

---

## C++ Standardization Process

### The Process Overview

1. **Initial Idea**: Identify improvement opportunity
2. **Committee Contact**: Check with committee members for prior art
3. **Draft Proposal**: Write paper following committee format
4. **Study Group Review**: Present to relevant study group (SG)
5. **Evolution**: Iterate based on feedback
6. **Working Group**: Move to Library (LWG) or Core (CWG) group
7. **Full Committee Vote**: Final approval by WG21

### Case Study: const shared_mutex Proposal

The lecture walks through a real proposal development:

**Problem Identification:**
```cpp
// Current requirement - mutex must be mutable
class BankAccount {
    mutable std::shared_mutex mtx_;  // Annoying mutable
    double balance_;
public:
    double getBalance() const {
        std::shared_lock lock(mtx_);  // Modifies mutex
        return balance_;
    }
};

// Proposed - make lock_shared const
class BankAccount {
    std::shared_mutex mtx_;  // No mutable needed!
    double balance_;
public:
    double getBalance() const {
        std::shared_lock lock(mtx_);  // Would be const operation
        return balance_;
    }
};
```

**Committee Feedback Analysis:**

Hans Boehm's response highlighted several considerations:

1. **Performance implications**: Concurrent `lock_shared` calls cause cache coherence traffic
2. **Consistency with atomics**: `atomic<T>::load()` is already const despite potential writes
3. **Linkage issues**: const can affect linkage in unexpected ways
4. **ROM placement**: const mutex might mislead about ROM placement

### Writing Standards Proposals

Using Michael Park's WG21 Markdown format:

```markdown
---
title: Making shared_mutex Operations Const
document: DXXXXR0
date: 2025-04-22
audience: LEWG
author:
  - name: Your Name
    email: your.email@example.com
---

# Abstract

This paper proposes making `shared_mutex::lock_shared()` a const 
member function to eliminate the need for mutable in common use cases.

# Motivation

[Real-world examples and pain points]

# Design Decisions

[Analysis of trade-offs]

# Proposed Wording

[Specific standard text changes]
```

---

## Complete Implementation Examples

### Enhanced Promise/Future with Memory Ordering

```cpp
template<typename T>
class MyPromise {
private:
    struct SharedState {
        std::optional<std::variant<T, std::exception_ptr>> value;
        std::atomic_flag promise_fulfilled = ATOMIC_FLAG_INIT;
        std::mutex mtx;  // For exception safety
        std::condition_variable cv;
    };
    
    std::shared_ptr<SharedState> state_;
    std::atomic<bool> future_retrieved_{false};
    
public:
    MyPromise() : state_(std::make_shared<SharedState>()) {}
    
    void set_value(T val) {
        // Check for double-satisfaction
        {
            std::lock_guard lock(state_->mtx);
            if (state_->value.has_value()) {
                throw std::future_error(
                    std::future_errc::promise_already_satisfied);
            }
            state_->value = std::move(val);
        }
        
        // Release semantics ensure value is visible
        state_->promise_fulfilled.test_and_set(std::memory_order_release);
        state_->promise_fulfilled.notify_one();
        state_->cv.notify_all();
    }
    
    void set_exception(std::exception_ptr e) {
        {
            std::lock_guard lock(state_->mtx);
            if (state_->value.has_value()) {
                throw std::future_error(
                    std::future_errc::promise_already_satisfied);
            }
            state_->value = e;
        }
        
        state_->promise_fulfilled.test_and_set(std::memory_order_release);
        state_->promise_fulfilled.notify_one();
        state_->cv.notify_all();
    }
    
    MyFuture<T> get_future() {
        bool expected = false;
        if (!future_retrieved_.compare_exchange_strong(expected, true)) {
            throw std::future_error(
                std::future_errc::future_already_retrieved);
        }
        return MyFuture<T>(state_);
    }
};

template<typename T>
class MyFuture {
private:
    std::shared_ptr<typename MyPromise<T>::SharedState> state_;
    
public:
    T get() {
        if (!state_) {
            throw std::future_error(std::future_errc::no_state);
        }
        
        // Acquire semantics ensure we see the value
        state_->promise_fulfilled.wait(false, std::memory_order_acquire);
        
        // Move out the value (can only get once)
        auto result = std::move(*state_->value);
        state_.reset();  // Invalidate future
        
        return std::visit(overload{
            [](T&& val) { return std::move(val); },
            [](std::exception_ptr&& e) -> T { 
                std::rethrow_exception(e); 
            }
        }, std::move(result));
    }
    
    bool valid() const noexcept {
        return state_ != nullptr;
    }
    
    void wait() const {
        if (!state_) {
            throw std::future_error(std::future_errc::no_state);
        }
        state_->promise_fulfilled.wait(false, std::memory_order_acquire);
    }
    
    template<class Rep, class Period>
    std::future_status wait_for(
        const std::chrono::duration<Rep, Period>& timeout) const {
        if (!state_) {
            throw std::future_error(std::future_errc::no_state);
        }
        
        std::unique_lock lock(state_->mtx);
        if (state_->cv.wait_for(lock, timeout, 
            [this] { return state_->value.has_value(); })) {
            return std::future_status::ready;
        }
        return std::future_status::timeout;
    }
};
```

### Complete ostream_joiner Implementation

```cpp
template<typename CharT = char, 
         typename Traits = std::char_traits<CharT>>
class ostream_joiner {
public:
    // Iterator traits
    using iterator_category = std::output_iterator_tag;
    using value_type = void;
    using difference_type = void;
    using pointer = void;
    using reference = void;
    using char_type = CharT;
    using traits_type = Traits;
    using ostream_type = std::basic_ostream<CharT, Traits>;
    
private:
    // Internal proxy for handling assignment
    class assignment_proxy {
        ostream_joiner* parent_;
    public:
        explicit assignment_proxy(ostream_joiner* p) : parent_(p) {}
        
        template<typename T>
        assignment_proxy& operator=(const T& value) {
            if (!parent_->first_) {
                *parent_->stream_ << parent_->delimiter_;
            }
            *parent_->stream_ << value;
            parent_->first_ = false;
            return *this;
        }
    };
    
    ostream_type* stream_;
    std::basic_string<CharT, Traits> delimiter_;
    bool first_;
    
public:
    // Constructor taking string_view for efficiency
    ostream_joiner(ostream_type& stream, 
                   std::basic_string_view<CharT, Traits> delim)
        : stream_(&stream), delimiter_(delim), first_(true) {}
    
    // Iterator operations
    assignment_proxy operator*() { return assignment_proxy(this); }
    ostream_joiner& operator++() { return *this; }
    ostream_joiner& operator++(int) { return *this; }
};

// Deduction guides
template<typename CharT, typename Traits>
ostream_joiner(std::basic_ostream<CharT, Traits>&, const CharT*)
    -> ostream_joiner<CharT, Traits>;

template<typename CharT, typename Traits>
ostream_joiner(std::basic_ostream<CharT, Traits>&, 
               std::basic_string_view<CharT, Traits>)
    -> ostream_joiner<CharT, Traits>;

// Helper function
template<typename CharT, typename Traits = std::char_traits<CharT>>
auto make_ostream_joiner(std::basic_ostream<CharT, Traits>& os,
                         std::basic_string_view<CharT, Traits> delim) {
    return ostream_joiner<CharT, Traits>(os, delim);
}

// Extension: operator<< for containers
template<typename Container,
         typename = std::enable_if_t<
             !std::is_same_v<Container, std::string> &&
             !std::is_same_v<Container, std::string_view>>>
auto operator<<(std::ostream& os, const Container& c) 
    -> decltype(std::begin(c), std::end(c), os) {
    os << "[";
    std::copy(std::begin(c), std::end(c),
              ostream_joiner(os, ", "));
    os << "]";
    return os;
}
```

### List Sorting Considerations

```cpp
// Why std::sort doesn't work with std::list
template<typename RandomIt>
void sort(RandomIt first, RandomIt last);  // Requires random access

// std::list only provides bidirectional iterators
std::list<int> lst = {3, 1, 4, 1, 5};
// std::sort(lst.begin(), lst.end());  // ERROR: wrong iterator category

// Solution 1: Use list's member function (merge sort)
lst.sort();  // O(n log n) time, O(1) space

// Solution 2: Copy to vector if random access is needed
std::vector<int> vec(lst.begin(), lst.end());
std::sort(vec.begin(), vec.end());
lst.assign(vec.begin(), vec.end());

// Solution 3: Custom merge sort for bidirectional iterators
template<typename BidirIt>
void merge_sort(BidirIt first, BidirIt last) {
    auto n = std::distance(first, last);
    if (n <= 1) return;
    
    auto mid = std::next(first, n / 2);
    merge_sort(first, mid);
    merge_sort(mid, last);
    std::inplace_merge(first, mid, last);
}
```

## Key Takeaways

1. **Factory patterns** can be generalized using variadic templates and type tags
2. **Memory ordering** requires understanding happens-before relationships and synchronization
3. **The spaceship operator** simplifies comparison implementation with three ordering types
4. **C++ standardization** is an open process where good ideas can improve the language
5. **Lock-free programming** requires careful attention to memory ordering semantics

## Common Pitfalls and Best Practices

- **Factory Design**: Use virtual inheritance to avoid diamond problems
- **Memory Ordering**: Default to sequential consistency unless performance critical
- **Spaceship Operator**: Remember to define `operator==` separately for custom implementations
- **Promise/Future**: Always check for double-satisfaction and invalid states
- **Iterator Categories**: Match algorithm requirements to iterator capabilities