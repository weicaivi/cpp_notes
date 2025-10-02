# Lecture 12 Summary

## Type Traits

### Concept and Purpose

Type traits are template-based metafunctions in `<type_traits>` that provide compile-time information about types. They serve as the foundation for concepts and enable compile-time polymorphism.

**Metafunction Structure:**
- Return types via nested `type` alias
- Return compile-time values via static `value` member
- Operate entirely at compile time

### Creating Custom Type Traits

**Basic Pointer Trait:**
```cpp
// Default to false
template<class T>
struct is_pointer : public std::false_type {};

// Specialize for pointer types
template<class T>
struct is_pointer<T*> : public std::true_type {};

static_assert(is_pointer<int*>::value);
static_assert(!is_pointer<int>::value);
```

**Transforming Trait (remove_pointer):**
```cpp
template<class T>
struct remove_pointer {
    using type = T;  // Default: return unchanged
};

template<class T>
struct remove_pointer<T*> {
    using type = T;  // For pointers: remove one level
};

static_assert(std::is_same_v<int, remove_pointer<int*>::type>);
```

### Helper Aliases and Variables

**The typename Disambiguation Problem:**
```cpp
// Doesn't compile - ambiguous parsing
template<typename T>
void f(T t) { 
    remove_pointer<T>::type rt;  // ERROR
}

// Fixed with typename keyword
template<typename T>
void f(T t) { 
    typename remove_pointer<T>::type rt;  // OK
}
```

**Modern Helper Aliases:**
```cpp
// Type alias template eliminates typename requirement
template<typename T>
using remove_pointer_t = typename remove_pointer<T>::type;

// Variable template for values
template<typename T>
inline constexpr bool is_pointer_v = is_pointer<T>::value;

// Clean usage
template<typename T>
void f(T t) { 
    remove_pointer_t<T> rt;  // Clean and readable
}

static_assert(is_pointer_v<int*>);  // Cleaner syntax
```

### Standard Library Type Traits Categories

**Type Categories:**
- `is_void`, `is_integral`, `is_floating_point`, `is_pointer`
- `is_lvalue_reference`, `is_rvalue_reference`
- `is_class`, `is_enum`, `is_union`, `is_function`
- `is_trivial`, `is_trivially_copyable`, `is_standard_layout`

**Operation Properties:**
- Construction: `is_constructible`, `is_trivially_constructible`, `is_nothrow_constructible`
- Copy: `is_copy_constructible`, `is_trivially_copy_constructible`, `is_nothrow_copy_constructible`
- Move: `is_move_constructible`, `is_trivially_move_constructible`, `is_nothrow_move_constructible`
- Assignment: `is_copy_assignable`, `is_trivially_copy_assignable`, `is_nothrow_copy_assignable`
- Destruction: `is_destructible`, `is_trivially_destructible`, `is_nothrow_destructible`

**Type Transformations:**
- CV qualifiers: `remove_cv`, `add_const`, `remove_volatile`
- References: `remove_reference`, `add_lvalue_reference`, `add_rvalue_reference`
- Pointers: `remove_pointer`, `add_pointer`
- Others: `decay`, `remove_cvref`, `make_signed`, `make_unsigned`

**Type Relations:**
- `is_same`, `is_base_of`, `is_convertible`, `is_nothrow_convertible`

## Practical Application: Optimized Copy

### Problem Statement

Standard copying (iterator-based) is abstract but can be slow for trivial types. `memcpy` is fast but not abstract. Can we get both?

**The Challenge:**
```cpp
// Abstract but potentially slow for trivial types
template<typename I1, typename I2>
I2 copy(I1 first, I1 last, I2 out) {
    while (first != last) {
        *out = *first;
        ++out; ++first;
    }
    return out;
}

// Fast but not abstract, not type-safe
void* memcpy(void* dest, const void* src, size_t count);
```

### Solution: Compile-Time Dispatch

**Implementation from optimized_copy.h:**
```cpp
namespace mpcs {
    // Generic version (fallback)
    template<typename I1, typename I2>
    I2 optimized_copy(I1 first, I1 last, I2 out) {
        while (first != last) {
            *out = *first;
            ++out; ++first;
        }
        return out;
    }

    // Helper to extract iterator value type
    template<typename T>
    using iter_value_t = typename std::iterator_traits<T>::value_type;

    // Optimized version for contiguous iterators with trivial types
    template<std::contiguous_iterator T, std::contiguous_iterator U>
    requires std::is_trivially_copy_assignable_v<iter_value_t<U>>
             && std::same_as<std::remove_const_t<iter_value_t<T>>, 
                             iter_value_t<U>>
    U optimized_copy(T first, T last, U out) {
        std::memcpy(&*out, &*first, 
                    (last - first) * sizeof(iter_value_t<T>));
        return out + (last - first);
    }
}
```

**How It Works:**
1. Compiler tries to match most specialized version first
2. If types are contiguous iterators AND trivially copyable → uses `memcpy`
3. Otherwise → falls back to element-by-element copy
4. Result: 800% performance improvement for trivial types, no performance loss for complex types

## Simulating Template Virtual Functions

### The Problem

Method templates cannot be virtual because the vtable would need infinite entries (one for each possible instantiation).

```cpp
struct S {
    // ILLEGAL - cannot be virtual
    template<typename T> virtual void f() = 0;
};
```

### The TT Tag Type Solution

**Tag Type Definition:**
```cpp
template<class T>
struct TT {};  // Empty, cheap to construct, always default-constructible
```

**Pattern Implementation:**
```cpp
// Single holder for one type
template<typename T>
struct abstract_creator {
    virtual std::unique_ptr<T> doCreate(TT<T>&&) = 0;
};

// Variadic expansion inherits from all holders
template<typename... Ts>
struct abstract_factory : public abstract_creator<Ts>... {
    template<class U> 
    std::unique_ptr<U> create() {
        abstract_creator<U>& creator = *this;
        return creator.doCreate(TT<U>());
    }
    virtual ~abstract_factory() = default;
};
```

**Key Insights:**
- `TT<T>()` creates empty tag object to dispatch on type
- Virtual dispatch happens on `doCreate` methods
- Template `create` method selects correct virtual method
- Result: template-like interface with virtual behavior

## Design Patterns with Templates

### Abstract Factory Pattern

**Traditional Implementation (Tedious):**
```cpp
class WidgetFactory {
public:
    virtual std::unique_ptr<Window> CreateWindow() = 0;
    virtual std::unique_ptr<Button> CreateButton() = 0;
    virtual std::unique_ptr<ScrollBar> CreateScrollBar() = 0;
    // ... dozens more widget types
};

class MSWindowsWidgetFactory : public WidgetFactory {
public:
    std::unique_ptr<Window> CreateWindow() override {
        return std::make_unique<MSWindow>();
    }
    std::unique_ptr<Button> CreateButton() override {
        return std::make_unique<MSButton>();
    }
    // ... tedious repetition
};
```

**Problems:**
- Tedious to write
- Error-prone (cut-and-paste bugs)
- Brittle (adding new widget types requires updating all factories)
- Hard to use uniformly (can't write `f.Create<T>()`)

### Template-Based Factory Implementation

**Complete Implementation (from factory.h):**

```cpp
namespace cspp51045 {
    // Tag type for dispatch
    template<typename T>
    struct TT {};

    // Base creator for single type
    template<typename T>
    struct abstract_creator {
        virtual std::unique_ptr<T> doCreate(TT<T>&&) = 0;
    };

    // Abstract factory inherits from all creators
    template<typename... Ts>
    struct abstract_factory : public abstract_creator<Ts>... {
        template<class U> 
        std::unique_ptr<U> create() {
            abstract_creator<U>& creator = *this;
            return creator.doCreate(TT<U>());
        }
        virtual ~abstract_factory() = default;
    };

    // Concrete creator for Abstract->Concrete mapping
    template<typename AbstractFactory, typename Abstract, typename Concrete>
    struct concrete_creator : virtual public AbstractFactory {
        std::unique_ptr<Abstract> doCreate(TT<Abstract>&&) override {
            return std::make_unique<Concrete>();
        }
    };

    // Concrete factory combines all creators
    template<typename AbstractFactory, typename... ConcreteTypes>
    struct concrete_factory;

    template<typename... AbstractTypes, typename... ConcreteTypes>
    struct concrete_factory<abstract_factory<AbstractTypes...>, ConcreteTypes...> 
        : public concrete_creator<abstract_factory<AbstractTypes...>, 
                                  AbstractTypes, ConcreteTypes>... {
    };
}
```

**Usage Example (from factory.cpp):**
```cpp
struct Scrollbar { virtual size_t position() = 0; };
struct Button { virtual void press() = 0; };

struct QtScrollbar : public Scrollbar {
    size_t position() override { return 0; }
};
struct QtButton : public Button {
    void press() override { std::cout << "QtButton pressed\n"; }
};

struct WindowsScrollbar : public Scrollbar {
    size_t position() override { return 0; }
};
struct WindowsButton : public Button {
    void press() override { std::cout << "WindowsButton pressed\n"; }
};

// Define factories with simple aliases
using AbstractWidgetFactory = abstract_factory<Scrollbar, Button>;
using QtWidgetFactory = 
    concrete_factory<AbstractWidgetFactory, QtScrollbar, QtButton>;
using WindowsWidgetFactory = 
    concrete_factory<AbstractWidgetFactory, WindowsScrollbar, WindowsButton>;

// Usage
std::unique_ptr<AbstractWidgetFactory> factory 
    = std::make_unique<WindowsWidgetFactory>();
std::unique_ptr<Button> button = factory->create<Button>();
button->press();  // Outputs: "WindowsButton pressed"
```

### Inheritance Diagram

**Abstract Factory Structure:**
```
abstract_factory<Scrollbar, Button>
    ├── abstract_creator<Scrollbar>
    └── abstract_creator<Button>
```

**Concrete Factory Structure:**
```
concrete_factory<abstract_factory<Scrollbar, Button>, 
                 WindowsScrollbar, WindowsButton>
    ├── concrete_creator<..., Scrollbar, WindowsScrollbar>
    │       ├── (virtual) abstract_factory<Scrollbar, Button>
    │       └── overrides doCreate(TT<Scrollbar>&&)
    └── concrete_creator<..., Button, WindowsButton>
            ├── (virtual) abstract_factory<Scrollbar, Button>
            └── overrides doCreate(TT<Button>&&)
```

Note: `virtual` inheritance prevents diamond problem from multiple inheritance of `abstract_factory`.

### Benefits of Template-Based Approach

1. **Easy to Define:** Single line instead of many method declarations
2. **Type-Safe:** Compiler ensures abstract/concrete type correspondence
3. **Uniform Interface:** `factory->create<T>()` works for all types
4. **Extensible:** Add new widget types by updating type list
5. **No Boilerplate:** Template generates all the repetitive code

## Homework Problems

### HW 12-1: Remove All Pointers
Implement `remove_all_pointers` trait that recursively removes all pointer levels:
```cpp
static_assert(std::is_same_v<int, remove_all_pointers_t<int**>>);
static_assert(std::is_same_v<int, remove_all_pointers_t<int***>>);
```

**Hint:** Use recursion with partial specialization.

### HW 12-2: Reference Traits
Implement `is_reference` and `remove_reference` for both lvalue and rvalue references:
```cpp
static_assert(is_reference_v<int&>);
static_assert(is_reference_v<int&&>);
static_assert(!is_reference_v<int>);
static_assert(std::is_same_v<int, remove_reference_t<int&>>);
static_assert(std::is_same_v<int, remove_reference_t<int&&>>);
```

### HW 12-3: Train Factory
Create factories for train components (Locomotive, FreightCar, Caboose) using provided factory templates.

### HW 12-4: Flexible Factory (Extra Credit)
Extend factory to support constructor arguments:
```cpp
using flexible_train_factory = flexible_abstract_factory
    Locomotive(double /* horsepower */),
    FreightCar(long /* capacity */),
    Caboose
>;
```

### HW 12-5: Parameterized Factory (Extra Credit)
Create generic transformation for concrete types:
```cpp
template<typename T> struct Model { /* ... */ };

using model_train_factory = 
    parameterized_factory<train_factory, Model>;
// Creates Model<Locomotive>, Model<FreightCar>, etc.
```

## Key Takeaways

1. **Type Traits Enable Compile-Time Decisions:** They allow the compiler to select optimal code paths based on type properties
2. **Zero-Cost Abstractions:** Template metaprogramming can achieve both abstraction and performance
3. **Design Patterns as Templates:** Traditional OOP patterns can be automated with templates, reducing boilerplate
4. **Virtual Templates Simulation:** Tag dispatch enables virtual-like behavior for template methods
5. **Variadic Inheritance:** Powerful technique for generating multiple inheritance hierarchies from type lists