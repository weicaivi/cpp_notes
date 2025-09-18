# Lecture 10 Summary

## I/O Streams Template System

### Character Type Templates

C++ I/O streams are built on templates to support multiple character encodings:

```cpp
// Basic template structure
template<class CharT, class Traits = char_traits<CharT>>
class basic_ostream;

// Standard typedefs
using ostream = basic_ostream<char>;        // 8-bit characters
using wostream = basic_ostream<wchar_t>;    // Wide characters
```

**Important**: `wchar_t` size varies by platform:
- Linux: 32 bits
- Windows: 16 bits

This inconsistency makes `wchar_t` problematic. Prefer `char`, `utf8_t`, or `utf32_t` for Unicode support.

### Stream Architecture

Streams provide formatted I/O while delegating character-level operations to stream buffers:

```cpp
// Stream buffer relationship
class basic_ios {
    basic_streambuf<CharT, Traits>* rdbuf();  // Get/set stream buffer
};

// Every stream has an associated stream buffer
ofstream file("data.txt");  // File stream with filebuf
stringstream ss;            // String stream with stringbuf
```

---

## I/O Manipulators

### Built-in Manipulators

I/O manipulators actively modify streams during insertion/extraction:

```cpp
#include <iostream>
#include <iomanip>

int main() {
    int value = 255;
    
    std::cout << std::hex << value << std::endl;        // ff
    std::cout << std::setw(10) << value << std::endl;   // Padded output
    std::cout << std::setfill('0') << std::setw(8) 
              << std::hex << value << std::endl;        // 000000ff
    
    return 0;
}
```

### Creating Custom Manipulators

#### Simple Function-Based Manipulators

```cpp
// Custom endl implementation
template<typename CharT, typename Traits>
std::basic_ostream<CharT, Traits>& 
custom_endl(std::basic_ostream<CharT, Traits>& os) {
    os << '\n';
    os.flush();
    return os;
}

// How operator<< handles function manipulators
template<typename CharT, typename Traits>
std::basic_ostream<CharT, Traits>& 
operator<<(std::basic_ostream<CharT, Traits>& os,
           std::basic_ostream<CharT, Traits>& (*manip)(std::basic_ostream<CharT, Traits>&)) {
    return manip(os);
}
```

#### Parameterized Manipulators

```cpp
struct setw {
    setw(size_t width) : width_(width) {}
    size_t width_;
};

template<typename CharT, typename Traits>
std::basic_ostream<CharT, Traits>& 
operator<<(std::basic_ostream<CharT, Traits>& os, const setw& sw) {
    os.width(sw.width_);
    return os;
}

// Usage
std::cout << setw(10) << "hello" << std::endl;
```

### Stream State Storage

Use `std::ios_base::xalloc()` to allocate per-stream storage slots:

```cpp
class CustomManipulator {
    static int index_;
public:
    static int get_index() {
        if (index_ == 0) {
            index_ = std::ios_base::xalloc();
        }
        return index_;
    }
    
    static void set_value(std::ios_base& stream, long value) {
        stream.iword(get_index()) = value;
    }
    
    static long get_value(std::ios_base& stream) {
        return stream.iword(get_index());
    }
};

int CustomManipulator::index_ = 0;
```

---

## Stream Buffers and Customization

### Stream Buffer Hierarchy

Stream buffers handle the actual I/O operations:

```cpp
// Custom stream creation pattern
template<class CharT, class Traits = std::char_traits<CharT>>
class basic_custom_stream : public std::basic_ostream<CharT, Traits> {
public:
    basic_custom_stream() : std::basic_ostream<CharT, Traits>(&buffer_) {}
    
private:
    custom_streambuf<CharT, Traits> buffer_;
};

using custom_stream = basic_custom_stream<char>;
```

### Custom Stream Buffer Implementation

```cpp
template<typename CharT, typename Traits = std::char_traits<CharT>>
class debug_streambuf : public std::basic_streambuf<CharT, Traits> {
protected:
    using int_type = typename Traits::int_type;
    
    // Override overflow for unbuffered output
    int_type overflow(int_type ch) override {
        if (ch != Traits::eof()) {
            // Custom output logic here
            std::clog << "[DEBUG] " << static_cast<char>(ch);
            return ch;
        }
        return Traits::eof();
    }
    
    // For buffered output, override sync() and use setp()
    void setup_buffer(CharT* begin, CharT* end) {
        setp(begin, end);  // Set put buffer
    }
};
```

---

## Modern Text Formatting with std::format

### Motivation: Problems with Traditional Approaches

#### iostream Issues
```cpp
// Verbose and error-prone
std::ostringstream msg;
msg << "Operation " << operation 
    << " failed with error code " << error_code;
std::string result = msg.str();
```

#### printf Issues
```cpp
// Type-unsafe and non-extensible
printf("%s\n", 1, 2);  // Compiles but crashes at runtime!
```

### std::format Solution

```cpp
#include <format>

// Type-safe and extensible
std::string msg = std::format("Operation {} failed with error code {}", 
                              operation, error_code);

// Automatic type deduction
std::cout << std::format("id {} - name {}", 5, "Mike");
```

### Implementing format with Variadics

```cpp
// Base case - no more arguments
std::string simple_format(std::string_view fmt) {
    return std::string(fmt);
}

// Helper to convert any type to string
template<typename T>
std::string make_string(const T& t) {
    std::ostringstream oss;
    oss << t;
    return oss.str();
}

// Recursive variadic implementation
template<typename T, typename... Ts>
std::string simple_format(std::string_view fmt, const T& t, const Ts&... ts) {
    auto braces = fmt.find("{}");
    if (braces == std::string_view::npos) {
        throw std::invalid_argument("Not enough format specifiers");
    }
    
    return std::string(fmt.substr(0, braces)) + 
           make_string(t) + 
           simple_format(fmt.substr(braces + 2), ts...);
}
```

### Custom Formatters

#### Inheriting from Existing Formatter

```cpp
struct Temperature { 
    double degrees{}; 
    bool fahrenheit{true}; 
};

template<>
struct std::formatter<Temperature> : std::formatter<std::string_view> {
    template<typename FormatContext>
    auto format(const Temperature& t, FormatContext& ctx) {
        std::string temp_str = std::format("{} degrees {}", 
                                          t.degrees, 
                                          t.fahrenheit ? "F" : "C");
        return std::formatter<std::string_view>::format(temp_str, ctx);
    }
};

// Usage with inherited format specifiers
Temperature temp{25, false};
auto s1 = std::format("{}", temp);        // "25 degrees C"
auto s2 = std::format("{:~^15}", temp);   // "~~25 degrees C~~"
```

#### Custom Parser Implementation

```cpp
template<>
struct std::formatter<Temperature> {
    bool force_celsius = true;
    
    template<typename ParseContext>
    auto parse(ParseContext& ctx) {
        auto pos = ctx.begin();
        while (pos != ctx.end() && *pos != '}') {
            if (*pos == 'F') {
                force_celsius = false;
            }
            ++pos;
        }
        return pos;
    }
    
    template<typename FormatContext>
    auto format(const Temperature& t, FormatContext& ctx) {
        double display_temp = force_celsius ? 
            (t.fahrenheit ? (t.degrees - 32) * 5/9 : t.degrees) :
            (t.fahrenheit ? t.degrees : t.degrees * 9/5 + 32);
            
        char unit = force_celsius ? 'C' : 'F';
        
        return std::format_to(ctx.out(), "{:.1f} degrees {}", display_temp, unit);
    }
};

// Usage with custom format specifiers
Temperature temp{77, true};  // 77°F
auto s1 = std::format("{}", temp);    // "25.0 degrees C" (converted)
auto s2 = std::format("{:F}", temp);  // "77.0 degrees F" (original)
```

---

## Regular Expressions Theory

### Formal Language Hierarchy (Chomsky Hierarchy)

#### Type 3: Regular Languages
- Recognized by finite state automata
- Can express patterns like `(tom|dick|harry)(, (tom|dick|harry))* (and (tom|dick|harry))?`
- **Limitation**: Cannot count (e.g., can't match `a^n b^n`)

```cpp
// Regular expression operators
// * : zero or more
// + : one or more  
// ? : optional
// | : alternation
// [] : character class

std::string pattern = R"((tom|dick|harry)(, (tom|dick|harry))*(and (tom|dick|harry))?)";
```

#### Type 2: Context-Free Languages
- Recognized by pushdown automata (finite state + stack)
- Can count one thing: `S → ab | aSb` matches `a^n b^n`
- **Limitation**: Cannot count two things simultaneously

#### Beyond Context-Free
Many practical regex engines support features that exceed regular languages:
- Backreferences: `^(.*)\1$` (matches repeated strings)
- Lookahead/lookbehind assertions
- Capture groups

### Finite State Automata

#### Nondeterministic Finite Automaton (NFA)
```
State transitions for [tdh][oa][mr]
Start(1) --[tdh]--> (2) --[oa]--> (3) --[mr]--> Accept(4)

Multiple transitions possible: backtracking required
```

#### Deterministic Finite Automaton (DFA)
```
All possibilities tracked simultaneously:
State(1) --[tdh]--> State(2,3,5) --[oa]--> State(4,6,8) --[mr]--> Accept
```

**Trade-off**: DFAs are faster (no backtracking) but use more memory (more states).

---

## Practical Regular Expressions in C++

### Basic Usage

```cpp
#include <regex>
#include <string>
#include <iostream>

int main() {
    std::string text = "How now, brown cow";
    
    // regex_match: entire string must match
    std::regex ow_pattern("ow");
    std::regex full_pattern("H.*w");
    
    std::cout << std::boolalpha;
    std::cout << std::regex_match(text, ow_pattern);     // false
    std::cout << std::regex_match(text, full_pattern);   // true
    
    // regex_search: find substring
    std::cout << std::regex_search(text, ow_pattern);    // true
    
    return 0;
}
```

### Capture Groups

```cpp
std::string email = "user@example.com";
std::regex email_pattern(R"((.*)@(.*))");
std::smatch results;

if (std::regex_search(email, results, email_pattern)) {
    std::cout << "Full match: " << results[0] << std::endl;  // user@example.com
    std::cout << "Username: " << results[1] << std::endl;    // user
    std::cout << "Domain: " << results[2] << std::endl;      // example.com
}

// Non-capturing groups: (?:pattern)
std::regex non_capture(R"((?:user|admin)@(.*))");  // Only domain captured
```

### Advanced Features

```cpp
// Repeat counts
std::regex four_digits(R"(\d{4})");           // Exactly 4 digits
std::regex three_to_five(R"(\d{3,5})");       // 3 to 5 digits

// Character classes
std::regex word_chars(R"(\w+)");              // Alphanumeric
std::regex whitespace(R"(\s+)");              // Whitespace
std::regex digits(R"(\d+)");                  // Digits

// Raw strings avoid escaping
std::regex pattern(R"(\d{4}-\d{2}-\d{2})");   // Date pattern: YYYY-MM-DD
```

### Performance Considerations

#### Catastrophic Backtracking
```cpp
// BAD: Exponential backtracking on failure
std::regex bad_pattern(R"((x+x+)+y)");

// GOOD: Rewritten to avoid backtracking
std::regex good_pattern(R"(xx+y)");
```

#### Optimization Guidelines
1. **Limit alternation**: `ab(cd|ef)` better than `(abcd|abef)`
2. **Use non-capturing groups** when capture isn't needed: `(?:pattern)`
3. **Anchor patterns** when possible: `^pattern$`
4. **Test failure cases** - often slower than success cases

### String Searching Alternatives

For literal string search without patterns:

```cpp
#include <algorithm>
#include <functional>

// C++17 searchers
std::string text = "The quick brown fox";
std::string target = "quick";

// Boyer-Moore algorithm
auto searcher = std::boyer_moore_searcher(target.begin(), target.end());
auto result = std::search(text.begin(), text.end(), searcher);

if (result != text.end()) {
    std::cout << "Found at position: " << std::distance(text.begin(), result);
}
```

---

## Compile-Time Regular Expressions (CTRE)

### Performance Problem

Standard C++ regex performance is poor due to:
1. **Runtime compilation**: NFA construction happens at runtime
2. **Locale overhead**: Character classification for each character
3. **Interpretation**: NFA execution is interpreted, not compiled

### CTRE Solution

Hana Dusíková's Compile-Time Regular Expressions library moves regex compilation to compile-time:

```cpp
#include <ctre.hpp>

// Compile-time regex compilation
constexpr auto date_pattern = ctre::match<R"((\d{4})/(\d{2})/(\d{2}))">;

std::optional<ctre::match_results> extract_date(std::string_view input) {
    if (auto match = date_pattern(input)) {
        return match;
    }
    return std::nullopt;
}

// Usage
if (auto result = extract_date("2025/03/25")) {
    std::cout << "Year: " << result->get<1>().str() << std::endl;   // 2025
    std::cout << "Month: " << result->get<2>().str() << std::endl;  // 03
    std::cout << "Day: " << result->get<3>().str() << std::endl;    // 25
}
```

### Performance Comparison

CTRE achieves dramatic performance improvements:
- **100x faster** than std::regex for simple patterns
- **Near rust/perl performance** levels
- **Compile-time error checking** for invalid patterns

```cpp
// Template-based regex representation
template<char... chars>
struct regex_pattern {
    // Compile-time NFA construction
    static constexpr auto nfa = build_nfa<chars...>();
    
    // Compile-time optimized matcher generation
    template<typename Iterator>
    static bool match(Iterator begin, Iterator end) {
        return execute_nfa<nfa>(begin, end);
    }
};
```

---

## constexpr and Compile-Time Programming

### constexpr Functions

Functions marked `constexpr` can execute at compile-time when given compile-time constant arguments:

```cpp
constexpr int factorial(int n) {
    return (n <= 1) ? 1 : n * factorial(n - 1);
}

// Computed at compile-time
constexpr int fact5 = factorial(5);  // 120

// Template parameter usage
template<int N>
struct factorial_array {
    static constexpr int value = factorial(N);
};
```

### constexpr Classes

Classes can be used at compile-time if their constructors and relevant methods are `constexpr`:

```cpp
class ConstexprString {
    const char* data_;
    size_t size_;
    
public:
    constexpr ConstexprString(const char* str) 
        : data_(str), size_(length(str)) {}
    
    constexpr size_t size() const { return size_; }
    constexpr char operator[](size_t i) const { return data_[i]; }
    
private:
    constexpr size_t length(const char* str) const {
        return *str ? 1 + length(str + 1) : 0;
    }
};

// Compile-time string processing
constexpr ConstexprString greeting("Hello");
static_assert(greeting.size() == 5);
static_assert(greeting[0] == 'H');
```

### Template Metaprogramming for CTRE

CTRE uses advanced template techniques to represent regex patterns as types:

```cpp
// Simplified CTRE concepts
template<char C>
struct character_literal {
    template<typename Iterator>
    static constexpr bool match(Iterator& it, Iterator end) {
        return it != end && *it++ == C;
    }
};

template<typename... Patterns>
struct sequence {
    template<typename Iterator>
    static constexpr bool match(Iterator& it, Iterator end) {
        return (Patterns::match(it, end) && ...);  // C++17 fold expression
    }
};

template<typename Pattern>
struct optional_pattern {
    template<typename Iterator>
    static constexpr bool match(Iterator& it, Iterator end) {
        Pattern::match(it, end);  // Ignore result
        return true;
    }
};

// Usage: Pattern for "abc?"
using pattern = sequence<
    character_literal<'a'>,
    character_literal<'b'>,
    optional_pattern<character_literal<'c'>>
>;
```

### Finite Automata as Template Instantiations

```cpp
template<int StateCount, int AcceptState>
struct finite_automaton {
    template<int From, int To, char Input>
    struct transition {
        static constexpr int from = From;
        static constexpr int to = To;
        static constexpr char input = Input;
    };
    
    // State machine execution at compile-time
    template<typename... Transitions>
    struct state_machine {
        template<typename Iterator>
        static constexpr bool execute(Iterator begin, Iterator end) {
            int current_state = 0;
            for (auto it = begin; it != end; ++it) {
                current_state = next_state<Transitions...>(current_state, *it);
                if (current_state == -1) return false;  // No transition
            }
            return current_state == AcceptState;
        }
    };
};
```

### Real-World CTRE Example

```cpp
#include <ctre.hpp>
#include <string_view>

// Email validation with CTRE
constexpr auto email_regex = ctre::match<R"([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})">;

constexpr bool is_valid_email(std::string_view email) {
    return static_cast<bool>(email_regex(email));
}

// Compile-time validation
static_assert(is_valid_email("user@example.com"));
static_assert(!is_valid_email("invalid.email"));

// Runtime usage with compile-time optimized code
int main() {
    std::string user_input;
    std::getline(std::cin, user_input);
    
    if (email_regex(user_input)) {
        std::cout << "Valid email address!" << std::endl;
    } else {
        std::cout << "Invalid email format." << std::endl;
    }
    
    return 0;
}
```
