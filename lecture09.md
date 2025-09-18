# Lecture 9 Summary

## SOLID Principles

### Single Responsibility Principle (SRP)

**Core Idea**: A class should have only one reason to change. Avoid classes that mix multiple concerns.

#### Problem Example
```cpp
// BAD: Mixed concerns
class Camera {
    // Positioning concerns
    void setPosition(double x, double y, double z);
    void rotate(double angle);
    
    // Resolution concerns  
    void setResolution(int width, int height);
    void setColorDepth(int depth);
    
    // File I/O concerns
    void saveToFile(const std::string& filename);
    void loadFromFile(const std::string& filename);
};
```

#### Better Solution
```cpp
// GOOD: Separated concerns
class CameraPosition {
    void setPosition(double x, double y, double z);
    void rotate(double angle);
};

class ImageSettings {
    void setResolution(int width, int height);
    void setColorDepth(int depth);
};

class Camera {
    CameraPosition position_;
    ImageSettings image_settings_;
    // Camera-specific logic only
};
```

#### Function-Level SRP
Use lambdas or functors to break down complex functions:

```cpp
// Option 1: Lambda with captures (recommended for medium complexity)
int processData(const Dataset& data) {
    auto validateInput = [&]() {
        if (data.empty()) throw std::invalid_argument("Empty dataset");
        // validation logic
    };
    
    auto transformData = [&]() {
        // transformation logic
    };
    
    auto generateResults = [&]() {
        // result generation
    };
    
    validateInput();
    transformData();
    return generateResults();
}

// Option 2: Functor (for very complex cases)
struct DataProcessor {
    DataProcessor(const Dataset& data) : data_(data) {}
    
    void validateInput() { /* validation */ }
    void transformData() { /* transformation */ }
    int generateResults() { /* generation */ }
    
    int operator()() {
        validateInput();
        transformData();
        return generateResults();
    }
    
private:
    const Dataset& data_;
    // Intermediate state
};
```

### Open/Closed Principle (OCP)

**Core Idea**: Software entities should be open for extension but closed for modification.

#### Traditional Visitor Pattern
```cpp
// Class designer provides extension point
struct AnimalVisitor {
    virtual ~AnimalVisitor() = default;
    virtual void visit(const Cat&) = 0;
    virtual void visit(const Dog&) = 0;
};

struct Animal {
    virtual ~Animal() = default;
    virtual void accept(const AnimalVisitor& v) const = 0;
};

struct Cat : public Animal {
    void accept(const AnimalVisitor& v) const override {
        v.visit(*this);
    }
};

// Client creates custom behavior without modifying Animal hierarchy
struct LifeSpanVisitor : public AnimalVisitor {
    mutable int lifespan = 0;
    
    void visit(const Cat&) override { lifespan = 15; }
    void visit(const Dog&) override { lifespan = 12; }
};

// Usage
std::unique_ptr<Animal> pet = std::make_unique<Cat>();
LifeSpanVisitor visitor;
pet->accept(visitor);
std::cout << "Lifespan: " << visitor.lifespan << " years\n";
```

#### Modern Variant-Based Extension
```cpp
using Animal = std::variant<Cat, Dog>;

// Users can add new "methods" without modifying the Animal definition
template<class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };

auto lifespan_calculator = overloaded{
    [](const Cat&) { return 15; },
    [](const Dog&) { return 12; }
};

Animal pet = Cat{};
int years = std::visit(lifespan_calculator, pet);
```

### Liskov Substitution Principle (LSP)

**Core Idea**: Subtypes must be substitutable for their base types without altering program correctness.

```cpp
// BAD: Violates LSP
struct Rectangle {
    virtual void setWidth(int w) { width_ = w; }
    virtual void setHeight(int h) { height_ = h; }
    virtual int area() const { return width_ * height_; }
    
protected:
    int width_, height_;
};

struct Square : public Rectangle {
    void setWidth(int w) override { 
        width_ = height_ = w;  // Violates LSP!
    }
    void setHeight(int h) override { 
        width_ = height_ = h;  // Violates LSP!
    }
};

// This function expects Rectangle behavior but breaks with Square
void testRectangle(Rectangle& r) {
    r.setWidth(5);
    r.setHeight(4);
    assert(r.area() == 20);  // Fails for Square!
}
```

#### Better Design
```cpp
// GOOD: Proper abstraction
struct Shape {
    virtual ~Shape() = default;
    virtual int area() const = 0;
};

struct Rectangle : public Shape {
    Rectangle(int w, int h) : width_(w), height_(h) {}
    int area() const override { return width_ * height_; }
    
    void setWidth(int w) { width_ = w; }
    void setHeight(int h) { height_ = h; }
    
private:
    int width_, height_;
};

struct Square : public Shape {
    Square(int side) : side_(side) {}
    int area() const override { return side_ * side_; }
    
    void setSide(int s) { side_ = s; }
    
private:
    int side_;
};
```

### Interface Segregation Principle (ISP)

**Core Idea**: No code should depend on methods it doesn't use.

#### Problem with Fat Interfaces
```cpp
// BAD: Fat interface
struct Job {
    virtual void print() = 0;
    virtual void staple() = 0;
    virtual void scan() = 0;
    virtual void fax() = 0;
};

// Forces implementation of unused methods
struct SimpleDoc : public Job {
    void print() override { /* implement */ }
    void staple() override { throw std::runtime_error("Not supported"); }
    void scan() override { throw std::runtime_error("Not supported"); }
    void fax() override { throw std::runtime_error("Not supported"); }
};
```

#### Solution with Segregated Interfaces
```cpp
// GOOD: Segregated interfaces
struct Printable {
    virtual ~Printable() = default;
    virtual void print() = 0;
};

struct Stapleable {
    virtual ~Stapleable() = default;
    virtual void staple() = 0;
};

struct Scannable {
    virtual ~Scannable() = default;
    virtual void scan() = 0;
};

// Implement only what you need
struct SimpleDoc : public Printable {
    void print() override { /* implement */ }
};

struct MultiFunctionDevice : public Printable, public Stapleable, public Scannable {
    void print() override { /* implement */ }
    void staple() override { /* implement */ }
    void scan() override { /* implement */ }
};

// Functions depend only on what they need
void printDocument(const Printable& doc) {
    doc.print();  // Only depends on printing capability
}
```

#### Concepts-Based ISP (C++20)
```cpp
template<typename T>
concept Printable = requires(T t) {
    t.print();
};

template<typename T>
concept Stapleable = requires(T t) {
    t.staple();
};

// Functions constrain only what they need
void processDocument(Printable auto& doc) {
    doc.print();
    // Can't accidentally call staple() - compile error
}

void finishDocument(Printable auto& doc, Stapleable auto& finisher) {
    doc.print();
    finisher.staple();
}
```

### Dependency Inversion Principle (DIP)

**Core Idea**: Depend on abstractions, not concretions.

#### Problem: Tight Coupling
```cpp
// BAD: Tightly coupled to concrete S3 implementation
class ThumbnailService {
    S3Folder input_folder_;  // Depends on concrete class
    
public:
    ThumbnailService(const std::string& bucket_name) 
        : input_folder_(bucket_name) {}
        
    void generateThumbnails() {
        auto files = input_folder_.listFiles();  // Tied to S3 API
        // process files...
    }
};
```

#### Solution 1: Virtual Base Classes
```cpp
// GOOD: Depends on abstraction
struct FolderInterface {
    virtual ~FolderInterface() = default;
    virtual std::vector<std::string> listFiles() const = 0;
    virtual std::vector<std::byte> readFile(const std::string& name) const = 0;
};

class S3Folder : public FolderInterface {
public:
    S3Folder(const std::string& bucket) : bucket_name_(bucket) {}
    
    std::vector<std::string> listFiles() const override {
        // S3-specific implementation
    }
    
    std::vector<std::byte> readFile(const std::string& name) const override {
        // S3-specific implementation
    }
    
private:
    std::string bucket_name_;
};

class ThumbnailService {
    std::unique_ptr<FolderInterface> folder_;
    
public:
    ThumbnailService(std::unique_ptr<FolderInterface> folder)
        : folder_(std::move(folder)) {}
        
    void generateThumbnails() {
        auto files = folder_->listFiles();  // Works with any folder type
        // process files...
    }
};

// Usage
auto service = ThumbnailService(std::make_unique<S3Folder>("my-bucket"));
```

#### Solution 2: Templates/Concepts
```cpp
// Concept defining folder requirements
template<typename T>
concept FolderLike = requires(const T& folder, const std::string& name) {
    { folder.listFiles() } -> std::same_as<std::vector<std::string>>;
    { folder.readFile(name) } -> std::same_as<std::vector<std::byte>>;
};

template<FolderLike Folder>
class ThumbnailService {
    Folder folder_;
    
public:
    ThumbnailService(Folder folder) : folder_(std::move(folder)) {}
    
    void generateThumbnails() {
        auto files = folder_.listFiles();
        // process files...
    }
};

// Usage with CTAD
ThumbnailService service{S3Folder{"my-bucket"}};
```

---

## Advanced OO Design Patterns

### Duck-Typed Variants

Combines compile-time efficiency with runtime polymorphism.

```cpp
// Define the variant type
using Shape = std::variant<Circle, Rectangle, Triangle>;

struct Circle {
    double radius;
    double area() const { return M_PI * radius * radius; }
    void draw() const { std::cout << "Drawing circle\n"; }
};

struct Rectangle {
    double width, height;
    double area() const { return width * height; }
    void draw() const { std::cout << "Drawing rectangle\n"; }
};

// Generic operations using std::visit
auto area_calculator = [](const auto& shape) {
    return shape.area();
};

auto drawer = [](const auto& shape) {
    shape.draw();
};

// Usage
std::vector<Shape> shapes = {
    Circle{5.0},
    Rectangle{4.0, 6.0},
    Triangle{3.0, 4.0, 5.0}
};

for (const auto& shape : shapes) {
    std::cout << "Area: " << std::visit(area_calculator, shape) << "\n";
    std::visit(drawer, shape);
}
```

### Advanced Overloaded Pattern
```cpp
template<class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };

// User-extensible behavior without modifying Shape variant
auto shape_processor = overloaded{
    [](const Circle& c) {
        std::cout << "Processing circle with radius " << c.radius << "\n";
        return c.area() * 1.1;  // 10% margin for circles
    },
    [](const Rectangle& r) {
        std::cout << "Processing rectangle " << r.width << "x" << r.height << "\n";
        return r.area() * 1.05; // 5% margin for rectangles
    },
    [](const auto& shape) {
        std::cout << "Processing generic shape\n";
        return shape.area();
    }
};

// Type-safe, extensible, and efficient
for (const auto& shape : shapes) {
    double processed_area = std::visit(shape_processor, shape);
    std::cout << "Processed area: " << processed_area << "\n";
}
```

---

## Type Erasure

Type erasure allows storing different types in a non-template class while preserving their behavior. It bridges the gap between dynamic and static polymorphism. It keeps dynamic polymorphism in a non-intrusive way (i.e. not requiring direct inheritance).

3 key factors in type erasure:
- a public interface that exposes the stable interface. It needs to have
    - a `unique_ptr<Concept>` pointer to the Concept interface, and the concrete object lives inside the templated Model<T>
    - a template constructor that accepts any possible type that satisfies the required operations to construct the model, hence hiding the type afterward
- a Concept that is the pure-virtual interface with virtual functions that all erased types must conform to after wrapping. It's like a base in dynamic polymorphism and a "type-erased vtable".
-  a Model that inherits from the Concept, which stores the concrete type and forwards calls to it. It also handles cloning for copy semantics. 

### Basic Type Erasure Pattern

```cpp
// The "Class" - public interface
class AnyDrawable {
public:
    // Template constructor
    // takes any type T, erases its type, and stores it as Model<T> behind a unique_ptr<Concept>
    // After construction, callers don’t know the concrete type anymore. they just hold AnyDrawable.
    template<typename T>
    AnyDrawable(T&& drawable) 
        : impl_(std::make_unique<Model<std::decay_t<T>>>(std::forward<T>(drawable))) {}
        
    // Copy/move semantics
    // polymorphic objects can’t be copied with a simple =
    // define a virtual clone() in Concept
    // Each Model<T> knows how to copy its own T and return a new Model<T>
    // This ensures deep copy instead of shallow pointer copy
    AnyDrawable(const AnyDrawable& other) 
        : impl_(other.impl_ ? other.impl_->clone() : nullptr) {}
        
    AnyDrawable(AnyDrawable&&) = default;
    AnyDrawable& operator=(const AnyDrawable& other) {
        if (this != &other) {
            impl_ = other.impl_ ? other.impl_->clone() : nullptr;
        }
        return *this;
    }
    AnyDrawable& operator=(AnyDrawable&&) = default;
    
    // Move is straightforward since unique_ptr already handles it

    // Interface methods
    // These methods delegate to the stored Concept object
    void draw() const { 
        if (impl_) impl_->draw(); 
    }
    
    double area() const { 
        return impl_ ? impl_->area() : 0.0; 
    }

private:
    // The "Concept" - internal interface (an "abstract base")
    // Defines the required interface for all erased types
    struct Concept {
        virtual ~Concept() = default;
        virtual void draw() const = 0;
        virtual double area() const = 0;
        virtual std::unique_ptr<Concept> clone() const = 0; // critical for copy semantics
    };
    
    // The "Model" - bridges concrete types to Concept
    // Implements the abstract Concept interface by forwarding calls to the real object
    template<typename T>
    struct Model : Concept {
        T object_; //the concrete object
        
        Model(T object) : object_(std::move(object)) {}
        
        void draw() const override { 
            object_.draw(); 
        }
        
        double area() const override { 
            return object_.area(); 
        }
        
        // clone() knows how to copy a T
        std::unique_ptr<Concept> clone() const override {
            return std::make_unique<Model>(object_);
        }
    };
    
    std::unique_ptr<Concept> impl_;
};
// the user sees only AnyDrawable, but internally each concrete object is wrapped in a Model<T> that conforms to Concept
```

### Using Type-Erased Objects
```cpp
// Different types that satisfy the drawable concept
struct Circle {
    double radius;
    void draw() const { std::cout << "Circle\n"; }
    double area() const { return M_PI * radius * radius; }
};

struct Square {
    double side;
    void draw() const { std::cout << "Square\n"; }
    double area() const { return side * side; }
};

// Store different types in same container
std::vector<AnyDrawable> shapes;
shapes.emplace_back(Circle{5.0});
shapes.emplace_back(Square{4.0});

// Polymorphic behavior without inheritance
for (const auto& shape : shapes) {
    shape.draw();
    std::cout << "Area: " << shape.area() << "\n";
}
```

### Advanced Type Erasure with Multiple Interfaces
```cpp
// Type erasure supporting multiple concepts
// The Concept has multiple methods (process(), cleanup(), status())
// Each concrete type (T) might implement some or all of them
// Model<T> uses `if constexpr (requires {...})` to check at compile-time whether the object supports that method, and only calls it if it exists
class AnyProcessor {
public:
    template<typename T>
    AnyProcessor(T&& processor) 
        // Internally impl_ points to a polymorphic Concept that delegates to the real type
        : impl_(std::make_unique<Model<std::decay_t<T>>>(std::forward<T>(processor))) {}
        
    // Multiple interface methods
    void process() const { impl_->process(); }
    void cleanup() const { impl_->cleanup(); }
    std::string status() const { return impl_->status(); }

private:
    struct Concept {
        virtual ~Concept() = default;
        virtual void process() const = 0;
        virtual void cleanup() const = 0;
        virtual std::string status() const = 0;
    };
    
    template<typename T>
    struct Model : Concept {
        T object_;
        
        Model(T object) : object_(std::move(object)) {}
        
        void process() const override { 
            if constexpr (requires { object_.process(); }) {
                object_.process();
            }
        }
        
        void cleanup() const override { 
            if constexpr (requires { object_.cleanup(); }) {
                object_.cleanup();
            }
        }
        
        std::string status() const override { 
            if constexpr (requires { object_.status(); }) {
                return object_.status();
            }
            return "Unknown";
        }
    };
    
    std::unique_ptr<Concept> impl_;
};
```

---

## Best Practices & Guidelines

### Modern C++ Object Lifecycle

#### Rule of Zero/Three/Five
```cpp
// Rule of Zero: Let the compiler handle everything
class GoodResource {
    std::unique_ptr<int[]> data_;
    std::vector<std::string> names_;
    // Compiler-generated special members are correct
};

// Rule of Five: If you need one, implement all
class ManualResource {
public:
    ~ManualResource() { delete[] data_; }
    
    ManualResource(const ManualResource& other) 
        : size_(other.size_), data_(new int[size_]) {
        std::copy(other.data_, other.data_ + size_, data_);
    }
    
    ManualResource& operator=(const ManualResource& other) {
        if (this != &other) {
            ManualResource temp(other);  // Copy-and-swap idiom
            swap(temp);
        }
        return *this;
    }
    
    ManualResource(ManualResource&& other) noexcept 
        : size_(other.size_), data_(other.data_) {
        other.size_ = 0;
        other.data_ = nullptr;
    }
    
    ManualResource& operator=(ManualResource&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            size_ = other.size_;
            data_ = other.data_;
            other.size_ = 0;
            other.data_ = nullptr;
        }
        return *this;
    }
    
private:
    size_t size_;
    int* data_;
    
    void swap(ManualResource& other) noexcept {
        std::swap(size_, other.size_);
        std::swap(data_, other.data_);
    }
};
```

### Header Best Practices
```cpp
#ifndef MYLIB_AWESOME_COMPONENT_H
#define MYLIB_AWESOME_COMPONENT_H

namespace mylib {

// Forward declarations when possible
class DatabaseConnection;
class Logger;

class AwesomeComponent {
public:
    // Use std:: prefix in headers - never "using namespace"
    explicit AwesomeComponent(std::string_view name);
    
    // const correctness
    std::string_view getName() const noexcept;
    void setLogger(std::shared_ptr<Logger> logger);
    
    // Modern C++ features
    template<typename... Args>
    void log(std::format_string<Args...> fmt, Args&&... args) const;
    
private:
    std::string name_;
    std::shared_ptr<Logger> logger_;
};

// Template implementation in header
template<typename... Args>
void AwesomeComponent::log(std::format_string<Args...> fmt, Args&&... args) const {
    if (logger_) {
        logger_->write(std::format(fmt, std::forward<Args>(args)...));
    }
}

} // namespace mylib

#endif // MYLIB_AWESOME_COMPONENT_H
```

### Exception Safety and RAII
```cpp
// Exception-safe operations
class DatabaseTransaction {
public:
    explicit DatabaseTransaction(DatabaseConnection& conn) 
        : conn_(conn), committed_(false) {
        conn_.beginTransaction();
    }
    
    ~DatabaseTransaction() noexcept {
        if (!committed_) {
            try {
                conn_.rollback();
            } catch (...) {
                // Log error but don't throw from destructor
            }
        }
    }
    
    void commit() {
        conn_.commit();
        committed_ = true;
    }
    
    // Non-copyable, movable
    DatabaseTransaction(const DatabaseTransaction&) = delete;
    DatabaseTransaction& operator=(const DatabaseTransaction&) = delete;
    
    DatabaseTransaction(DatabaseTransaction&& other) noexcept 
        : conn_(other.conn_), committed_(other.committed_) {
        other.committed_ = true;  // Prevent rollback in moved-from object
    }
    
private:
    DatabaseConnection& conn_;
    bool committed_;
};

// Usage
void performDatabaseOperations(DatabaseConnection& conn) {
    DatabaseTransaction txn(conn);  // RAII ensures cleanup
    
    // Do work...
    conn.execute("INSERT INTO users...");
    conn.execute("UPDATE accounts...");
    
    txn.commit();  // Explicit success
    // If commit() not called, destructor rolls back automatically
}
```

### Modern Template Constraints
```cpp
// C++20 Concepts
template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

template<typename T>
concept Iterable = requires(T container) {
    container.begin();
    container.end();
};

template<Numeric T>
T gcd(T a, T b) {
    return b == 0 ? a : gcd(b, a % b);
}

// Pre-C++20 SFINAE
template<typename T, 
         typename = std::enable_if_t<std::is_arithmetic_v<T>>>
T clamp(T value, T min_val, T max_val) {
    return std::max(min_val, std::min(value, max_val));
}

// Function parameter concepts (C++20)
void processContainer(Iterable auto& container) {
    for (auto& item : container) {
        // Process item
    }
}
```

### Thread Safety and Concurrency
```cpp
// Thread-safe class with proper synchronization
class ThreadSafeCounter {
public:
    void increment() {
        std::scoped_lock lock(mutex_);
        ++count_;
    }
    
    void add(int value) {
        std::scoped_lock lock(mutex_);
        count_ += value;
    }
    
    int get() const {
        std::shared_lock lock(mutex_);  // Reader lock
        return count_;
    }
    
    // Atomic operations for simple cases
    void incrementAtomic() {
        atomic_count_.fetch_add(1, std::memory_order_relaxed);
    }
    
private:
    mutable std::shared_mutex mutex_;  // Allows const methods to lock
    int count_ = 0;
    
    std::atomic<int> atomic_count_{0};
};

// Lock-free programming with atomics
class LockFreeFlag {
    std::atomic_flag flag_ = ATOMIC_FLAG_INIT;
    
public:
    void set() {
        flag_.test_and_set(std::memory_order_acquire);
        flag_.notify_one();
    }
    
    void wait() {
        flag_.wait(false, std::memory_order_acquire);
    }
    
    void clear() {
        flag_.clear(std::memory_order_release);
    }
};
```

---

## Code Examples

### Enhanced Promise/Future Implementation

```cpp
#include <memory>
#include <atomic>
#include <optional>
#include <variant>
#include <exception>

template<class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };

namespace mpcs {

template<typename T>
class MyFuture;

template<typename T>
struct SharedState {
    std::optional<std::variant<T, std::exception_ptr>> result;
    std::atomic<bool> ready{false};
    
    // For waiting/notification
    mutable std::mutex mutex;
    mutable std::condition_variable cv;
};

template<typename T>
class MyFuture {
public:
    MyFuture(const MyFuture&) = delete;
    MyFuture& operator=(const MyFuture&) = delete;
    
    MyFuture(MyFuture&&) = default;
    MyFuture& operator=(MyFuture&&) = default;
    
    T get() {
        // Wait for result using condition variable (more portable than atomic wait)
        std::unique_lock lock(state_->mutex);
        state_->cv.wait(lock, [this] { return state_->ready.load(); });
        
        // Extract result using overloaded visitor
        return std::visit(overloaded{
            [](T&& value) -> T { return std::move(value); },
            [](std::exception_ptr ep) -> T { std::rethrow_exception(ep); }
        }, std::move(*state_->result));
    }
    
    // Non-blocking check
    bool is_ready() const noexcept {
        return state_->ready.load();
    }
    
    // Wait with timeout
    template<typename Rep, typename Period>
    std::future_status wait_for(const std::chrono::duration<Rep, Period>& timeout) const {
        std::unique_lock lock(state_->mutex);
        if (state_->cv.wait_for(lock, timeout, [this] { return state_->ready.load(); })) {
            return std::future_status::ready;
        }
        return std::future_status::timeout;
    }

private:
    friend class MyPromise<T>;
    
    explicit MyFuture(std::shared_ptr<SharedState<T>> state) 
        : state_(std::move(state)) {}
    
    std::shared_ptr<SharedState<T>> state_;
};

template<typename T>
class MyPromise {
public:
    MyPromise() : state_(std::make_shared<SharedState<T>>()) {}
    
    // Non-copyable but movable
    MyPromise(const MyPromise&) = delete;
    MyPromise& operator=(const MyPromise&) = delete;
    MyPromise(MyPromise&&) = default;
    MyPromise& operator=(MyPromise&&) = default;
    
    void set_value(T value) {
        set_result(std::move(value));
    }
    
    void set_exception(std::exception_ptr ep) {
        set_result(ep);
    }
    
    MyFuture<T> get_future() {
        return MyFuture<T>(state_);
    }

private:
    template<typename U>
    void set_result(U&& result) {
        {
            std::lock_guard lock(state_->mutex);
            if (state_->ready.load()) {
                throw std::future_error(std::future_errc::promise_already_satisfied);
            }
            state_->result = std::forward<U>(result);
            state_->ready.store(true);
        }
        state_->cv.notify_all();
    }
    
    std::shared_ptr<SharedState<T>> state_;
};

} // namespace mpcs
```
