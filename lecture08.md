# Lecture 8 Summary

## Error Handling and Exceptions

**Key Principle**: Exceptions can be thrown at any point, requiring defensive programming.

### Exception Safety Guidelines

#### 1. Destructor Safety
**Critical Rule**: Never throw exceptions from destructors.

```cpp
// BAD - Can cause std::terminate
struct BadX { 
    ~BadX() { throw "Error"; }  // Dangerous!
};

// GOOD - Catch and handle in destructor
struct GoodX {
    ~GoodX() {
        try {
            risky_operation();
        } catch(...) {
            // Log error but don't rethrow
        }
    }
};
```

When destructors throw during exception unwinding, `std::terminate()` is called immediately.

#### 2. Exception-Safe Interface Design

**std::stack Example**: Why `pop()` returns void instead of the popped value.

```cpp
// Problematic design (not in std::stack)
T stack::pop() {
    T result = top_;
    remove_top();
    return result;  // Exception here loses the element!
}

// Safe design (actual std::stack)
void stack::pop() { remove_top(); }
T& stack::top() { return top_; }

// Usage:
T value = stk.top();  // Get value first
stk.pop();            // Then remove safely
```

#### 3. Vector Growth and Move Semantics

Vectors face a dilemma during reallocation: copy (safe assuming copy constructor doesn't throw but slow) vs move (fast but risky if the move constructor can throw).

`std::string`: Its move constructor is noexcept (since C++11). -> Fast moves.

Custom class with dynamic allocations but not noexcept: Compiler conservatively assumes move might throw. -> Falls back to copy.

Plain old data (int, double, etc.): Moves are trivial, so always noexcept. -> Fast path.

```cpp
// Vector's decision logic
// constexpr: compile time decision, the vector decides once per type T, not at run time
if constexpr (std::is_nothrow_move_constructible_v<T>) {
    // Move elements - fast and safe
    std::uninitialized_move(old_begin, old_end, new_begin);
} else {
    // Copy elements - slow but exception-safe
    std::uninitialized_copy(old_begin, old_end, new_begin);
}
```

### noexcept Specification

#### Basic Usage
```cpp
// Simple noexcept
void safe_function() noexcept {
    // Guaranteed not to throw
}

// Conditional noexcept
template<typename T>
auto safe_square(T&& x) noexcept(std::is_nothrow_move_constructible_v<T>) {
    return x * x;
}
```

#### Advanced noexcept Patterns
want `complex_operation` to be marked `noexcept` if and only if the operations it performs are also `noexcept`
```cpp
// noexcept(noexcept(...)) idiom
template<typename T>
auto complex_operation(T&& x)
    // outer noexcept specifier on the function signature: 
    // Says complex_operation is noexcept if the expression inside evaluates to true at compile time
    // inner noexcept(expr) is an operator, it returns a compile-time bool
    // true if the expression expr is guaranteed not to throw (i.e. calling x.method)
    noexcept(noexcept(x.method()) && std::is_nothrow_move_constructible_v<T>) {
    return x.method();
}
```

**Best Practice**: Mark move constructors and destructors as `noexcept` when possible.

### Modern Error Handling: std::expected

C++23 introduces `std::expected<T, E>` as an alternative to exceptions, combining benefits of error codes and exceptions.

```cpp
#include <expected>

std::expected<int, std::string> divide(int a, int b) {
    if (b == 0) {
        return std::unexpected("Division by zero");
    }
    return a / b;
}

// Usage
auto result = divide(10, 0);
if (result) {
    std::cout << "Result: " << *result << std::endl;
} else {
    std::cout << "Error: " << result.error() << std::endl;
}
```

---

## Polymorphic Containers

### std::pair and std::tuple

#### Basic Usage
```cpp
// Tuple with CTAD (Class Template Argument Deduction)
std::tuple data{42, "hello", 3.14};  // tuple<int, const char*, double>

// Access methods
std::cout << std::get<0>(data);      // 42
std::cout << std::get<int>(data);    // 42 (type-based access)

// Pair usage
std::pair coordinates{10, 20};       // pair<int, int>
std::cout << coordinates.first;      // 10
std::cout << coordinates.second;     // 20
```

#### Structured Bindings (C++17)
```cpp
// Multiple return values
std::tuple<std::string, int, double> get_person_data() {
    return {"Alice", 25, 175.5};
}

// Clean decomposition
auto [name, age, height] = get_person_data();
std::cout << std::format("Name: {}, Age: {}, Height: {}", name, age, height);

// Works with arrays, pairs, tuples, and custom types
int arr[2] = {1, 2};
auto [x, y] = arr;  // x = 1, y = 2
```

### std::optional

Represents values that may or may not exist, eliminating null pointer issues.

```cpp
std::optional<int> find_value(const std::vector<int>& vec, int target) {
    auto it = std::find(vec.begin(), vec.end(), target);
    if (it != vec.end()) {
        return *it;
    }
    return std::nullopt;  // or just {}
}

// Usage patterns
auto result = find_value(numbers, 42);
if (result) {
    std::cout << "Found: " << *result << std::endl;
} else {
    std::cout << "Not found" << std::endl;
}

// Alternative syntax
if (auto value = find_value(numbers, 42); value.has_value()) {
    process(value.value());
}
```

### std::variant

Type-safe unions that can hold one of several types.

```cpp
std::variant<int, std::string, double> data;

data = 42;           // Holds int
data = "hello";      // Now holds string
data = 3.14;         // Now holds double

// Type checking
std::cout << data.index();                              // 2 (double is at index 2)
std::cout << std::holds_alternative<double>(data);      // true
std::cout << std::get<double>(data);                   // 3.14

// Safe access
if (auto* str = std::get_if<std::string>(&data)) {
    std::cout << "String value: " << *str << std::endl;
} else {
    std::cout << "Not a string" << std::endl;
}
```

---

## Object-Oriented Design Patterns

### Templates vs Inheritance Comparison

#### Traditional OO Approach
```cpp
struct Animal {
    virtual std::string name() const = 0;
    virtual std::string eats() const = 0;
    virtual ~Animal() = default;
};

class Cat : public Animal {
public:
    std::string name() const override { return "cat"; }
    std::string eats() const override { return "mice"; }
};

// Usage requires dynamic allocation
std::unique_ptr<Animal> pet = std::make_unique<Cat>();
std::cout << pet->name() << " eats " << pet->eats() << std::endl;
```

#### Concepts-Based Approach (C++20)
```cpp
template<typename T>
concept Animal = requires(const T& a) {
    // describe the requirements
    { a.name() } -> std::convertible_to<std::string>;
    { a.eats() } -> std::convertible_to<std::string>;
};

struct Cat {
    std::string name() const { return "cat"; }
    std::string eats() const { return "mice"; }
};

// Usage with better performance
void describe_animal(Animal auto pet) {
    std::cout << pet.name() << " eats " << pet.eats() << std::endl;
}
// Animal auto pet is shorthand for
// template<Animal T>
// void describe_animal(T pet)

Cat fluffy;
describe_animal(fluffy);  // No virtual dispatch, stack allocation
```

### Duck-Typed Variants: Best of Both Worlds

Combining the performance of templates with runtime polymorphism.

```cpp
// Define variant type
using Animal = std::variant<Cat, Dog, Bird>;

// Create animals
Animal zoo[] = {Cat{}, Dog{}, Bird{}};

// Generic operations using std::visit
auto name_visitor = [](const auto& animal) -> std::string {
    return animal.name();
};

for (const auto& animal : zoo) {
    // std::visit dispatches to the correct branch depenidng on the actual type inside the variant
    std::cout << std::visit(name_visitor, animal) << std::endl;
}
```

#### The Overloaded Pattern
```cpp
// a small helper struct that inherits from multiple lambdas (Ts...) and 
// brings all their operator() into scope via using
// merges multiple lambdas into one callable object with overloaded operator()s
template<class... Ts> 
struct overloaded : Ts... { 
    using Ts::operator()...; 
};
template<class... Ts> 
overloaded(Ts...) -> overloaded<Ts...>;  // Deduction guide

// Usage for different behaviors per type
auto life_span = overloaded {
    // multiple operator() overloads
    [](const Cat&) { return 15; },
    [](const Dog&) { return 12; },
    [](const Bird&) { return 8; }
};

Animal pet = Cat{};
std::cout << "Life span: " << std::visit(life_span, pet) << " years" << std::endl;
```

### Performance and Design Trade-offs

| Approach | Runtime Cost | Compile-time | Extensibility | Type Safety |
|----------|--------------|--------------|---------------|-------------|
| Virtual Functions | High (vtable) | Fast | Good (inheritance) | Strong |
| Templates/Concepts | Low (inlined) | Slow | Limited (recompile) | Strong |
| Variants | Medium (switch) | Medium | Requires modification | Strong |

---


### Custom Promise/Future Implementation
`std::future/std::promise` are just wrappers around a shared state that is heap allocated. The promise side writes into the shared state and the future side reads from it.

The shared state looks like this conceptually.
```cpp
enum class State { empty, val, exc };

template<typename T>
struct SharedState {
    std::mutex mtx;
    std::condition_variable cv;
    State state = State::empty;
    std::optional<T> value;                  // if val
    std::exception_ptr exception;            // if exc
};

```

```cpp
template<typename T>
class MyFuture {
    // each future points to the shares state
    // shared_ptr ensures the state lives until both the producer and consumer are done
    std::shared_ptr<SharedState<T>> state_;
public:
    T get() {
        std::unique_lock lock{state_->mtx};
        state_->cv.wait(lock, [this] {
            // waits on the cv until the state is no longer empty
            // so either producer has set a value, or produce has set an exception
            return state_->state != State::empty; 
        });
        
        switch (state_->state) {
            case State::val: 
                // if a value exists, move it out and return
                return std::move(*state_->value);
            case State::exc:
                // if an exception was stored, rethrow it so the consumer sees the failure
                std::rethrow_exception(state_->exception);
            default:
                throw std::runtime_error{"Invalid future state"};
        }
    }
    
    // Non-blocking check
    bool is_ready() const {
        std::lock_guard lock{state_->mtx};
        return state_->state != State::empty;
    }
};
```

The producer side (MyPromise) would look like:
```cpp
template<typename T>
class MyPromise {
    std::shared_ptr<SharedState<T>> state_;
public:
    MyFuture<T> get_future() { return MyFuture<T>{state_}; }

    void set_value(T v) {
        {
            std::lock_guard lock{state_->mtx};
            state_->value = std::move(v);
            state_->state = State::val;
        }
        state_->cv.notify_all();
    }

    void set_exception(std::exception_ptr e) {
        {
            std::lock_guard lock{state_->mtx};
            state_->exception = e;
            state_->state = State::exc;
        }
        state_->cv.notify_all();
    }
};
```

### Advanced my_async Implementation

properly handles perfect forwarding:

```cpp
namespace mpcs {
    template<typename Func, typename ...Args>
    auto my_async(Func&& f, Args&&... args) {
        using RetType = std::invoke_result_t<std::decay_t<Func>, std::decay_t<Args>...>;
        
        std::packaged_task<RetType(std::decay_t<Args>...)> pt(std::forward<Func>(f));
        auto result = pt.get_future();
        
        std::thread(std::move(pt), std::forward<Args>(args)...).detach();
        return result;
    }
}
```

### Improved Error Handling with std::expected

```cpp
#include <expected>
#include <string>

enum class NetworkError {
    ConnectionFailed,
    Timeout,
    InvalidResponse
};

std::expected<std::string, NetworkError> fetch_data(const std::string& url) {
    // Simulate network operation
    if (url.empty()) {
        return std::unexpected(NetworkError::InvalidResponse);
    }
    
    // Success case
    return "Data from " + url;
}

// Chainable operations
auto process_url(const std::string& url) {
    return fetch_data(url)
        .and_then([](const std::string& data) -> std::expected<int, NetworkError> {
            return static_cast<int>(data.length());
        })
        .or_else([](NetworkError err) -> std::expected<int, NetworkError> {
            return 0;  // Default value on error
        });
}
```

## Key Takeaways

1. **Exception Safety**: Design interfaces that remain consistent even when exceptions occur
2. **noexcept**: Mark functions appropriately to enable optimizations and convey intent
3. **Modern Containers**: Use `std::optional`, `std::variant`, and structured bindings for cleaner code
4. **Design Flexibility**: Consider templates/concepts vs inheritance vs variants based on specific requirements
5. **Performance**: Understand the runtime costs of different polymorphism approaches
