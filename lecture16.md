# Lecture 16 Summary

## 1. XML Data Binding Overview

XML Data Binding automatically generates C++ classes from XML Schema definitions, enabling type-safe serialization/deserialization.

### Goal: Transform XML Schema to C++ Code

**Input XML Schema:**
```xml
<xs:schema targetNamespace="http://tempuri.org/XMLSchema.xsd">
  <xs:element name="note">
    <xs:complexType>
      <xs:sequence>
        <xs:element name="to" type="xs:string"/>
        <xs:element name="from" type="xs:string"/>
        <xs:element name="heading" type="xs:string"/>
        <xs:element name="body" type="xs:string"/>
      </xs:sequence>
    </xs:complexType>
  </xs:element>
</xs:schema>
```

**Generated C++ Structure:**
```cpp
struct note_type {
    std::string to;
    std::string from;
    std::string heading;
    std::string body;
};

// Auto-generated serialization functions
template<>
note_type fromXML<note_type>(xml::parser& p, std::string name);

template<>
void toXML<note_type>(const note_type& x, xml::serializer& s, std::string name);
```

---

## 2. Versioned Namespace Pattern

### Problem
Library evolution requires supporting multiple versions simultaneously without breaking existing code.

### Solution: Nested Version Namespaces
```cpp
namespace mpcs {
    namespace v1 {
        class formatter { /* version 1 implementation */ };
        class complex_type { /* version 1 */ };
    }
    
    namespace v2 {
        class formatter { /* version 2 with new features */ };
        class complex_type { /* version 2 */ };
    }
    
    // Default to latest stable version
    using namespace v1;
}

// Client code options:

// 1. Use specific version (immune to updates)
using namespace mpcs::v1;

// 2. Use recommended version (gets updates)
using namespace mpcs;
```

---

## 3. Visitor Pattern with Accept

### Adding Virtual Methods Externally

The visitor pattern allows adding polymorphic behavior without modifying existing class hierarchies:

```cpp
// Base visitor for all types in the hierarchy
template<typename T>
struct single_visitor {
    virtual void visit(T const&) const = 0;
};

// Combine visitors using typelists
template<typename... Ts>
struct visitor_helper<typelist<Ts...>> : public single_visitor<Ts>... {
    using single_visitor<Ts>::visit...;  // C++17 using declaration pack expansion
};

// Type hierarchy with accept methods
struct type_base {
    virtual void accept(type_visitor const& tv) const { 
        tv.visit(*this); 
    }
    virtual ~type_base() = default;
    std::string name;
};

struct complex_type : public type_base {
    virtual void accept(type_visitor const& tv) const override { 
        tv.visit(*this); 
    }
    // ... members
};

// Using the visitor
struct name_printer : public type_visitor {
    void visit(type_base const& t) const override {
        std::cout << "Base: " << t.name;
    }
    void visit(complex_type const& ct) const override {
        std::cout << "Complex: " << ct.name;
    }
};
```

---

## 4. Variant-Based Enum Dispatching

### Problem: Manual Switch Statements

Traditional enum handling requires error-prone switch statements that break when new values are added.

### Solution: Tag Types with Variants

```cpp
// Convert enum values to tag types
template<parser::event_type e>
struct event_t {
    static parser::event_type constexpr event = e;
};

// Create variant of all possible tag types (automated)
template<typename T> struct EVHelper;
template<size_t... nums>
struct EVHelper<std::index_sequence<nums...>> {
    using type = std::variant<event_t<parser::event_type(nums)>...>;
};

using event_variant = EVHelper<std::make_index_sequence<parser::eof + 1>>::type;

// Runtime enum to variant conversion
event_holder event_to_variant(parser::event_type e) {
    static std::map<parser::event_type, event_holder> val_to_type = {
        {parser::start_element, event_t<parser::start_element>()},
        {parser::end_element, event_t<parser::end_element>()},
        // ... auto-generated for all values
    };
    return val_to_type[e];
}

// Process events using std::visit instead of switch
struct event_processor {
    template<parser::event_type e>
    void process(event_t<e> const&) {
        // Specialized handling for each event type
    }
    
    void handle_event(parser::event_type e) {
        std::visit([this](auto const& tagged_event) {
            process(tagged_event);
        }, event_to_variant(e));
    }
};
```

---

## 5. Parallel Hierarchies

### Managing Multiple Related Class Hierarchies

The formatter system maintains parallel hierarchies for different output formats:

```cpp
// Abstract formatter hierarchy
template<typename T>
struct formatter : public inheriter<formatter, direct_bases_t<T, all_types>> {
    virtual void generate(generate_args& f, T const&) = 0;
    virtual ~formatter() = default;
};

// Parallel concrete hierarchies
template<typename T> struct struct_formatter;  // Generates structs
template<typename T> struct class_formatter;   // Generates classes

// Specialized formatters inherit from bases
template<>
struct struct_formatter<complex_type> 
    : public struct_formatter_bases<complex_type> {
    void generate(generate_args& f, complex_type const& ct) override {
        generate_struct(f, ct);
        generate_serializers(f, ct);
    }
};

// Factory creates entire parallel hierarchy
using struct_formatter_factory = 
    parallel_concrete_factory<formatter_factory, struct_formatter>;
```

---

## 6. Advanced Factory Implementation

### Sophisticated Abstract Factory with Variadic Templates

```cpp
// Type-to-type mapping for disambiguation
template<typename T>
struct Type2Type {
    typedef T type;
};

// Signature adaptation for perfect forwarding
template<typename T> 
struct adapt_signature {
    using type = std::add_const_t<T>&;
};

template<typename T> 
struct adapt_signature<T&> {
    using type = T;
};

template<typename T>
struct adapt_signature<T&&> {
    using type = T;
};

// Abstract creator base
template<typename T>
struct abstract_creator {
    virtual std::unique_ptr<T> doCreate(Type2Type<T>&&) const = 0;
};

// Specialized for function signatures
template<typename Result, typename... Args>
struct abstract_creator<Result*(Args...)> {
    virtual std::unique_ptr<Result> doCreate(
        Type2Type<Result>&&, 
        typename adapt_signature<Args>::type...
    ) const = 0;
};

// Abstract factory combining multiple creators
template<typename... Ts>
struct abstract_factory<typelist<Ts...>> 
    : public abstract_creator<Ts>... {
    
    template<class U, typename... Args> 
    std::unique_ptr<U> create(Args&&... args) const {
        // Select correct abstract_creator base
        abstract_creator<typename Signature<U, Ts...>::type> const& creator = *this;
        return creator.doCreate(Type2Type<U>(), std::forward<Args>(args)...);
    }
};

// Concrete factory implementation
template<typename AbstractFactory, typename... ConcreteTypes>
struct concrete_factory;

template<typename... AbstractTypes, typename... ConcreteTypes>
struct concrete_factory<
    abstract_factory<typelist<AbstractTypes...>>, 
    typelist<ConcreteTypes...>
> : public concrete_creator<
        abstract_factory<typelist<AbstractTypes...>>, 
        AbstractTypes, 
        ConcreteTypes
    >... {
};
```

### C++17 Improvement: Variadic Using
```cpp
// Flexible factory using C++17 variadic using declarations
template<typename... Ts>
struct flexible_abstract_factory<typelist<Ts...>> 
    : public abstract_creator<Ts>... {
    
    using abstract_creator<Ts>::doCreate...;  // No need for Signature metaprogramming!
    
    template<class U, typename... Args> 
    std::unique_ptr<U> create(Args&&... args) {
        return doCreate(Type2Type<U>(), args...);
    }
};
```

---

## 7. IndentStream for Output Formatting

### Custom Stream Buffer for Automatic Indentation

```cpp
class IndentStreamBuf : public std::streambuf {
public:
    IndentStreamBuf(std::ostream& stream)
        : wrappedStreambuf(stream.rdbuf())
        , isLineStart(true)
        , myIndent(0) {}
    
    virtual int overflow(int outputVal) override {
        if (outputVal == traits_type::eof())
            return traits_type::eof();
            
        if (outputVal == '\n') {
            isLineStart = true;
        } else if (isLineStart) {
            // Insert indentation at line start
            for (size_t i = 0; i < myIndent; ++i) {
                wrappedStreambuf->sputc(' ');
            }
            isLineStart = false;
        }
        
        wrappedStreambuf->sputc(static_cast<char>(outputVal));
        return outputVal;
    }
    
    size_t myIndent;
    
private:
    std::streambuf* wrappedStreambuf;
    bool isLineStart;
};

// Stream manipulators
std::ostream& indent(std::ostream& ostr) {
    if (auto* buf = dynamic_cast<IndentStreamBuf*>(ostr.rdbuf())) {
        buf->myIndent += 4;
    }
    return ostr;
}

std::ostream& unindent(std::ostream& ostr) {
    if (auto* buf = dynamic_cast<IndentStreamBuf*>(ostr.rdbuf())) {
        buf->myIndent -= 4;
    }
    return ostr;
}

// Usage
IndentStream ios(std::cout);
ios << "class Foo {\n" << indent;
ios << "int x;\n";
ios << "int y;\n" << unindent;
ios << "};\n";
```

---

## 8. Complete XSD to C++ Generator Architecture

### System Flow

```
XML Schema → Parser → Type Model → Formatter → C++ Code
```

### Core Components

#### 1. XML Schema Parser (`xmlbind.h`)
```cpp
global_scope inhaleSchema(std::istream& is) {
    global_scope global;
    scopeStack.push_back(std::ref(static_cast<scope&>(global)));
    
    xml::parser p(is, "schema");
    event_processor ep(p);
    eventProcessors.push(std::make_unique<event_processor>(p));
    
    for (parser::event_type e : p) {
        eventProcessors.top()->process(e);
        p.attribute_map();
    }
    
    return global;
}
```

#### 2. Type Model Hierarchy
```cpp
// Base type with visitor support
struct type_base {
    virtual void accept(type_visitor const& tv) const;
    std::string name;
};

// Complex types with nested members
struct complex_type : virtual public type_base, public scope {
    std::map<std::string, std::unique_ptr<type_base>> memberTypes;
    std::vector<std::unique_ptr<data_member>> dataMembers;
    std::string containingElementName;
    bool anonymous;
};

// Data members with cardinality
struct data_member {
    std::string name;
    type_base& type;
};

struct optional_data_member : public data_member {
    // Generates std::optional<T>
};

struct multiple_data_member : public data_member {
    // Generates std::vector<T>
};
```

#### 3. Code Generation (`struct_formatter.h`)
```cpp
template<>
struct struct_formatter<complex_type> {
    void generate(generate_args& f, complex_type const& ct) override {
        generate_struct_definition(f, ct);
        generate_serializer(f, ct);
        generate_deserializer(f, ct);
    }
    
private:
    void generate_serializer(generate_args& f, complex_type const& ct) {
        f.os << "template<>\n";
        f.os << "void toXML<" << ct.name << ">(";
        f.os << ct.name << " const& x, xml::serializer& s, std::string name) {\n";
        f.os << indent;
        
        if (!ct.anonymous)
            f.os << "if(name.empty()) name = \"" << ct.containingElementName << "\";\n";
        
        f.os << "s.start_element(name);\n";
        
        for (auto& dm : ct.dataMembers) {
            generate_member_serializer(f, *dm);
        }
        
        f.os << "s.end_element();\n";
        f.os << unindent << "}\n";
    }
};
```

---

## 9. Code Analysis and Improvements

### Issues in Original Code

1. **Missing Error Handling in `findType`**
```cpp
// Original (incomplete)
type_base& findType(string name) {
    for (auto it = scopeStack.rbegin(); it != scopeStack.rend(); it++) {
        auto lookup = it->get().memberTypes.find(name);
        if (lookup != it->get().memberTypes.end())
            return *lookup->second;
    }
    // Todo: Handle error - MISSING!
}

// Improved version
type_base& findType(const std::string& name) {
    for (auto it = scopeStack.rbegin(); it != scopeStack.rend(); ++it) {
        if (auto lookup = it->get().memberTypes.find(name); 
            lookup != it->get().memberTypes.end()) {
            return *lookup->second;
        }
    }
    throw std::runtime_error("Type '" + name + "' not found in any scope");
}
```

2. **Improved Output Stream Joiner**
```cpp
// Enhanced ostream_joiner with better iterator traits
struct ostream_joiner {
    using iterator_category = std::output_iterator_tag;
    using value_type = void;
    using difference_type = std::ptrdiff_t;
    using pointer = void;
    using reference = void;
    
    ostream_joiner(std::ostream& os, std::string_view delimiter) 
        : impl_(os, delimiter) {}
    
    template<typename T>
    ostream_joiner& operator=(const T& value) {
        impl_.write(value);
        return *this;
    }
    
    ostream_joiner& operator*() { return *this; }
    ostream_joiner& operator++() { return *this; }
    ostream_joiner& operator++(int) { return *this; }
    
private:
    struct impl {
        impl(std::ostream& os, std::string_view del) 
            : os(&os), delimiter(del), first(true) {}
        
        template<typename T>
        void write(const T& value) {
            if (!first) *os << delimiter;
            first = false;
            *os << value;
        }
        
        std::ostream* os;
        std::string delimiter;
        bool first;
    } impl_;
};
```

3. **Complete Built-in Type Support**
```cpp
// Add more XML Schema built-in types
struct double_type : public builtin_type {
    double_type() : builtin_type("double", "xs:double") {}
    virtual void accept(type_visitor const& tv) const override { 
        tv.visit(*this); 
    }
};

struct date_type : public builtin_type {
    date_type() : builtin_type("std::chrono::year_month_day", "xs:date") {}
    virtual void accept(type_visitor const& tv) const override { 
        tv.visit(*this); 
    }
};

// Formatter specializations
template<>
struct struct_formatter<double_type> : public struct_formatter_bases<double_type> {
    virtual std::string xToChars() const override { 
        return "std::to_string(x)"; 
    }
    virtual std::string charsToX() const override { 
        return "std::stod(x)"; 
    }
};
```

### Best Practices Demonstrated

1. **Method Length**: No method exceeds 5 lines - extreme decomposition for clarity
2. **Template Metaprogramming**: Heavy use of typelists and SFINAE for compile-time polymorphism
3. **Virtual Inheritance**: Used to handle diamond inheritance in parallel hierarchies
4. **Perfect Forwarding**: Preserves value categories through factory methods
5. **Heterogeneous Lookup**: Uses `std::less<>` for efficient string_view lookups

## Key Takeaways

1. **Versioned namespaces** enable smooth library evolution
2. **Visitor pattern** adds polymorphic behavior without modifying existing hierarchies
3. **Variant-based dispatching** eliminates error-prone switch statements
4. **Parallel hierarchies** with factories enable pluggable code generation strategies
5. **Custom stream buffers** provide transparent output formatting
6. **Extreme decomposition** (5-line methods) improves maintainability and testability
