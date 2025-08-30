# Lecture 6 Summary

### Distributed Counter Implementation

Analysis of the three distributed counter versions shows evolution of thread-safe design:

```cpp
// Version 1: Simple mutex (high contention)
// every increment by any thread must acquire the same lock
// If many threads increment at once, they block each other -> poor scalability
class DistributedCounter1 {
    std::shared_mutex mtx;
    long long count = 0;
    
public:
    // takes a unique lock for writing
    void operator++() {
        std::unique_lock lock(mtx);
        ++count;
    }
    
    // takes a shared lock for reading 
    long long get() const {
        std::shared_lock lock(mtx);
        return count;
    }
};

// Version 2: Bucket-based (reduced contention)
// multiple “buckets”, each with its own counter and mutex
// Each thread hashes its thread::id to choose a bucket
// Still some possible collisions if threads hash to the same bucket
// Buckets may be stored contiguously in memory, can suffer false sharing
// which is when multiple threads write to variables on the same cache line,
// causing performance degradation even if the variables are logically independent
class DistributedCounter2 {
    struct Bucket {
        std::shared_mutex mtx;
        size_t count = 0;
    };
    
    static constexpr size_t BUCKET_COUNT = 128;
    std::vector<Bucket> buckets{BUCKET_COUNT};
    
public:
    // Increment only locks one bucket, not the entire counter
    void operator++() {
        size_t index = std::hash<std::thread::id>{}(std::this_thread::get_id()) % BUCKET_COUNT;
        // std::hash is a functor, produces a size_t hash for a thread ID
        // {} constructs a temporary hash object
        // operator()(...) immediately calls the hash object with the thread ID
        std::unique_lock lock(buckets[index].mtx);
        ++buckets[index].count;
    }
    
    // sums all buckets with shared locks
    size_t get() const {
        return std::accumulate(buckets.begin(), buckets.end(), size_t{0},
            [](size_t acc, const Bucket& b) {
                std::shared_lock lock(b.mtx);
                return acc + b.count;
            });
    }
};

// Version 3: Cache-line padding (avoid false sharing)
// Same idea as Version 2: multiple buckets, each with its own mutex
// New addition: Each bucket is aligned to and thu occupies a full cache line (usually 64 bytes)
// Avoids cache-coherence bottlenecks, better scaling under high concurrency
class DistributedCounter3 {
    struct alignas(64) Bucket {  // Align to cache line
        std::shared_mutex mtx;
        size_t count = 0;
        char padding[64 - sizeof(mtx) - sizeof(count)];  // Prevent false sharing
        // char padding[...] ensures that each Bucket occupies exactly one cache line
        // Calculates how many bytes remain in the cache line after mtx and count
        // Fills the remainder with a dummy char array (padding)
    };
    
    static constexpr size_t BUCKET_COUNT = 128;
    std::vector<Bucket> buckets{BUCKET_COUNT};
    
public:
    // Same interface as Version 2
    void operator++() { /* ... */ }
    size_t get() const { /* ... */ }
};
```
#### Performance Considerations

1. **False Sharing Prevention**: Version 3 uses padding to ensure each bucket occupies its own cache line
2. **Lock Granularity**: Distributed approach reduces contention compared to single mutex
3. **Thread-Local Hashing**: Uses thread ID to distribute load across buckets

### Portable Cache-Conscious Programming (C++17)

```cpp
#include <new>

// C++17 hardware interference sizes
// std::hardware_destructive_interference_size is the minimum offset (in bytes) between two objects to avoid false sharing 
// Typically 64 bytes on most CPUs (a cache line size)
struct alignas(std::hardware_destructive_interference_size) cache_aligned_bucket {
// Ensures the whole struct is aligned to the size of a cache line
    std::shared_mutex sm; // a lock for synchronization
    size_t count = 0;
};

// Alternative: ensure objects are on same cache line
// std::hardware_constructive_interference_size is the maximum offset (in bytes) between two objects to encourage them to share a cache line.
// Also usually 64 byte
struct cache_friendly_pair {
    alignas(std::hardware_constructive_interference_size) int frequently_accessed_together[2];
    // Ensures the array is packed into a single cache line (as much as possible)
    // Since both integers are “frequently accessed together,” this maximizes cache efficiency
};
```

### Cache-Conscious Programming Guidelines

1. **Separate Independent Data**: Keep independently accessed data on different cache lines
2. **Collocate Related Data**: Keep frequently co-accessed data on same cache lines
3. **Read-Only vs Mutable Separation**: Separate immutable data from frequently modified data
4. **Lock-Data Relationships**:
   - Frequently contended locks: separate from protected data
   - Rarely contended locks: collocate with protected data
5. **Power-of-Two Size Considerations**: Large power-of-two sized objects may cause direct-mapped cache conflicts


    Modern CPUs fetch data into cache lines (usually 64 bytes wide). These lines are stored in a cache hierarchy (L1, L2, L3). Each cache line maps to a slot in the cache based on its address. For direct-mapped caches (or set-associative caches with limited associativity), the cache index is computed like this:

        cache_index = (address / cache_line_size) % num_cache_sets

    That means multiple memory addresses may compete for the same cache line if their addresses differ by a multiple of (cache_line_size × num_cache_sets).


    If an object’s size is a power of two (e.g., 64, 128, 256 bytes), then successive objects in an array will start at addresses that are also spaced by a power of two. Because the cache index calculation itself uses modulo with a power of two. That means all those objects may map to the same cache set (or a small subset of sets), even though there are plenty of other sets available.


    Suppose:

        - Cache line size = 64 bytes
        - Number of sets = 128 (so total cache = 64 × 128 = 8 KB)
        - We have objects of size = 128 bytes (a power of two)

    Now, object addresses in an array look like:

        Object 0: base + 0
        Object 1: base + 128
        Object 2: base + 256
        Object 3: base + 384

    If you compute cache indexes:

        index(obj0) = (0 / 64) % 128 = 0 % 128 = 0
        index(obj1) = (128 / 64) % 128 = 2 % 128 = 2
        index(obj2) = (256 / 64) % 128 = 4 % 128 = 4

    So far okay — they spread across sets.

    But if the cache has fewer sets, say 8 sets:

        index(obj0) = 0 % 8 = 0
        index(obj1) = 2 % 8 = 2
        index(obj2) = 4 % 8 = 4
        index(obj3) = 6 % 8 = 6
        index(obj4) = 8 % 8 = 0  <-- conflict with obj0

    Now we’re cycling back and evicting earlier lines prematurely -> cache thrashing.


    The bigger the object, the fewer objects fit per cache line, and the more likely their stride aligns poorly with cache set boundaries. If the stride is a power of two and the cache indexing is modulo a power of two, aliasing happens systematically, not randomly. This means instead of spreading evenly across all cache sets, they collide in a repeating, predictable pattern, wasting much of the cache capacity

    Mitigation Strategies
        
        - Pad object sizes to avoid exact powers of two. Example: instead of 128 bytes, make it 136 bytes.
        - Introduce prime-number strides. Some allocators intentionally offset or randomize object placement to reduce conflicts.
        - Use cache-friendly grouping. For instance, storing hot fields together in a smaller structure rather than lumping everything into a large power-of-two block.
        - Rely on higher associativity caches. Modern CPUs are often 8-way or 16-way associative, which greatly reduces but doesn’t entirely eliminate this effect.

    In summary, when your data structure size = a power of two, it can align “too neatly” with the cache’s power-of-two indexing function. That creates systematic cache conflicts, leading to more evictions and worse performance, even if the cache isn’t full.


## Class Template Argument Deduction (CTAD)

### Function Template Argument Deduction Review
```cpp
template<typename T>
auto square(T x) { return x * x; }

// Usage
auto result1 = square(7);        // T deduced as int
auto result2 = square<double>(7); // T explicitly specified as double
```

### CTAD Mechanism
```cpp
template<class MutexType>
struct lock_guard {
    lock_guard(MutexType& m) : m(m) { m.lock(); }
    ~lock_guard() { m.unlock(); }
    MutexType& m;
};

std::mutex mtx;
lock_guard l(mtx);  // MutexType deduced as std::mutex

// Also enables:
std::vector v = {1, 2, 3};  // vector<int> deduced
```

## Move Semantics

### Conceptual Foundation

**Move vs Copy**: Moving transfers ownership/resources from source to destination, potentially invalidating the source object.

### Motivation

#### Performance Benefits
```cpp
// Tree copying requires duplicating entire structure
std::unique_ptr<TreeNode> copyTree(const std::unique_ptr<TreeNode>& source);

// Tree moving only transfers root ownership
std::unique_ptr<TreeNode> moveTree(std::unique_ptr<TreeNode>&& source) {
    return std::move(source);  // O(1) operation
}
```

#### Semantic Requirements
```cpp
// unique_ptr cannot be copied (ownership semantics)
class unique_ptr {
    unique_ptr(const unique_ptr&) = delete;
    unique_ptr& operator=(const unique_ptr&) = delete;
    
    // But can be moved
    unique_ptr(unique_ptr&&) noexcept;
    unique_ptr& operator=(unique_ptr&&) noexcept;
};
```

### Rvalue References

```cpp
template<class T> 
void swap(T& a, T& b) {
    T tmp = std::move(a);  // Move from a
    a = std::move(b);      // Move from b  
    b = std::move(tmp);    // Move from tmp
}
```

### Move-Enabled Containers

```cpp
template<typename T> 
class vector {
public:
    void push_back(const T& t);  // Copy version
    void push_back(T&& t);       // Move version
};

// Usage
std::vector<std::unique_ptr<btree>> trees;
for(int i = 0; i < 10; i++) {
    trees.push_back(std::make_unique<btree>());  // Move temporary
}
```

### Implementing Move Semantics

```cpp
class MovableResource {
    std::unique_ptr<int[]> data;
    size_t size;

public:
    // Move constructor
    MovableResource(MovableResource&& other) noexcept 
        : data(std::move(other.data)), size(other.size) {
        other.size = 0;  // Reset source
    }
    
    // Move assignment
    MovableResource& operator=(MovableResource&& other) noexcept {
        if (this != &other) {
            data = std::move(other.data);
            size = other.size;
            other.size = 0;
        }
        return *this;
    }
    
    // Rule of Five: also need copy constructor, copy assignment, destructor
    MovableResource(const MovableResource&);
    MovableResource& operator=(const MovableResource&);
    ~MovableResource();
};
```

### Perfect Forwarding

```cpp
template<typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

// Usage preserves value categories
struct Widget {
    Widget(int& lvalue_ref, const int& const_ref, int&& rvalue_ref);
};

int x = 42;
const int y = 24;

auto w = make_unique<Widget>(x, y, 100);  // Perfectly forwards all argument types
```

## Low-Level Systems Programming

### Pointer Fundamentals

```cpp
// Basic pointer operations
int* ptr = new int(42);
int value = *ptr;           // Dereference
int* copy = ptr;           // Pointer copy (shallow)
delete ptr;                // Manual cleanup

// Pointer arithmetic
int arr[10];
int* p = arr;
*(p + 5) = 100;           // Equivalent to arr[5] = 100
```

### nullptr vs Traditional Approaches

```cpp
// Modern C++ (correct)
void func(char* str);
void func(int num);

func(nullptr);            // Unambiguously calls func(char*)

// Legacy C++ (problematic)
func(0);                  // Ambiguous: could call either overload
func(NULL);               // Macro-based, still potentially ambiguous
```

### String Handling

#### C-Style Strings
```cpp
const char* c_str = "Hello";  // char const[6]
std::cout << c_str[1];        // Prints 'e'
```

#### string_view for Efficient Text Processing
```cpp
#include <string_view>

// Efficient: no copying, works with any string type
void process_text(std::string_view text) {
    // Process without copying underlying data
}

// Usage examples
std::string s = "Hello";
process_text(s);              // Works with std::string
process_text("World");        // Works with string literals  
process_text({"foobar", 3}); // Works with pointer + length
```

#### Raw String Literals
```cpp
// Multi-line strings without escaping
std::string json = R"({
    "name": "John",
    "path": "C:\Users\John\Documents",
    "regex": "\d+\.\d+"
})";

// Custom delimiters for content containing )"
std::string code = R"CODE(
    std::string msg = R"(embedded "quotes")";
)CODE";
```

### Function Pointers and std::function

#### Function Pointers
```cpp
// Basic function pointer
// old C-style way of working with functions as variables
int add(int a, int b) { return a + b; }
int (*operation)(int, int) = &add;
// `int (*operation)(int, int)` declares a pointer to a function taking two ints and returning an int
// `= &add;` assigns the address of the function add to the pointer.
int result = operation(3, 4);  // result = 7

// Function pointer with runtime selection
double mean(const std::vector<double>&) { /* ... */ }
double median(const std::vector<double>&) { /* ... */ }

std::string choice;
std::cin >> choice;
double (*calculator)(const std::vector<double>&) = 
    (choice == "mean") ? mean : median;
// calculator is a pointer to a function with signature double(const std::vector<double>&)
```

#### std::function for Type Erasure
Normally, a lambda, a function pointer, and a functor all have different types. `std::function` “erases” the specific type information, storing a wrapper internally. At runtime, it just knows “I can call this with `(const std::vector<double>&)` and I’ll get back a double.”
```cpp
#include <functional>

// std::function<R(Args...)> is a type-erased wrapper for any callable object that can be invoked with the given signature
std::function<double(const std::vector<double>&)> processor;

processor = mean;                                    // Function
processor = [](const auto& v) { return v.front(); }; // Lambda
processor = Calculator{};                            // Functor (an object with operator() like Calculator{})

// Member function usage
struct Data { int getValue() const; };
std::function<int(const Data&)> getter = &Data::getValue; // &Data::getValue is a pointer to member function, which std::function<int(const Data&)> can wrap it neatly
Data d;
int value = getter(d); // under the hood it calls d.getValue()
```

### Memory Management

#### Understanding new/delete
```cpp
// operator new(sizeof(T)) allocate memory
// T constructor construct object in memory
T* ptr = new T(args);

// T destructor destroy object
// 2. operator delete(ptr) deallocate memory
delete ptr;
```

#### Custom Memory Management
```cpp
// Placement new: construct object at specific location
void* buffer = std::malloc(sizeof(MyClass));
MyClass* obj = new(buffer) MyClass(args);  // Construct at buffer
obj->~MyClass();                           // Manual destructor call
std::free(buffer);                         // Release memory

// No-throw allocation
MyClass* ptr = new(std::nothrow) MyClass;
if (!ptr) {
    // Handle allocation failure without exception
}
```

#### Exception Safety with RAII
```cpp
// Problematic: potential memory leak
void unsafe_function() {
    process(new Resource(), new Resource());  // Exception between allocations = leak
}

// Safe: RAII ensures cleanup
void safe_function() {
    auto r1 = std::make_unique<Resource>();
    auto r2 = std::make_unique<Resource>();
    process(std::move(r1), std::move(r2));
}
```

### Advanced Pointer Types

#### Member Pointers
```cpp
struct Point {
    int x, y;
    void move(int dx, int dy);
};

// Pointer to member variable
int Point::* coord_ptr = &Point::x; // `int Point::*` means: “a pointer to an int member inside Point”
Point p{10, 20};
p.*coord_ptr = 30;  // Sets p.x = 30
// take object p, follow the pointer-to-member coord_ptr, assign 30

// Pointer to member function  
void (Point::*move_ptr)(int, int) = &Point::move; // a pointer to a Point member function taking (int, int) and returning void.
(p.*move_ptr)(5, 5);  // Calls p.move(5, 5)

// Runtime member selection
std::vector<Point> points;
int Point::* selected_coord = user_wants_x ? &Point::x : &Point::y;
for (auto& point : points) {
    std::cout << point.*selected_coord << std::endl;
}
```

## Best Practices Summary

### Cache-Conscious Design
1. Align independent data to separate cache lines
2. Group frequently co-accessed data
3. Separate read-only from frequently modified data
4. Consider lock placement relative to protected data
5. Use C++17 hardware interference sizes for portable code

### Modern C++ Memory Management
1. Prefer `std::make_unique`/`std::make_shared` over raw `new`
2. Use RAII for automatic resource management
3. Understand move semantics for efficient resource transfer
4. Apply the Rule of Five when managing resources
5. Use `std::string_view` for efficient string processing

### Concurrent Programming
1. Minimize shared mutable state
2. Consider lock-free alternatives where appropriate
3. Design for cache-friendly data access patterns
4. Profile on target architectures
5. Understand the performance implications of synchronization primitives
