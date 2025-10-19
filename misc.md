C++ implementation

- GNU: gcc/libstdc++, widely used in Unix-based systems
- LLVM: clang/libc++, widely used in MacOS (gcc on Mac is in fact Apple-Clang, a changed version of Clang)
    `stdlib=libc++`
- Microsoft: msvc/MS-STL; msvc is not open source. wildly used in Windows
    usually microsoft implements libraries faster, while msvc updates slower (slow language feature update)

`std::cmp_xxx` defined in `<utility>` can safely compare unsigned and signed integers (excluding bool/characters)
    - `std::cmp_greater_equal(std::string::npos, 0)` will return `false`
    - `std::in_range<T>(x)` can be used to check whether a value is represetable by interger type `T`
  

Any arithmetic operation involving types smaller than `int` (like `char`, `short`, or `unsigned short`) promotes them to `int` before the operation.
- If all values of that type fit in an `int` (true for `16-bit unsigned short` on most platforms), the promotion goes to `int`.
- Otherwise, promotion goes to `unsigned int`.

```cpp
int main() {
    unsigned short s1 = 0xFF00, s2 = 0x100, s3 = 0xFFFF; // decimal 65280, 256, 65535
    // might expect 16-bit wraparound (i.e., 0xFF00 + 0x100 = 0) 
    // bc carry-out from the most significant bit is discarded for unsigned (0x0001 for signed)
    // s1 + s2 is promoted to int 65280 + 256 = 65536
    // s3 is also promoted to int (65535)
    if (s1 + s2 > s3) std::cout << "Unexpected!\n"; // this gets printed
    
    unsigned short i1 = 0xFFFF0000, i2 = 0x10000, i3 = 0xFFFFFFFF;
    // When assigning a large constant to an unsigned short, the value is truncated (modulo 2^16)
    // i1 = 0xFFFF0000 take the low 16 bits -> 0x0000 = 0
    // i2 = 0x10000 low 16 bits -> 0x0000 = 0
    // i3 = 0xFFFFFFFF low 16 bits -> 0xFFFF = 65535
    if (i1 + i2 > i3) std::cout << "Unexpected too!\n"; // this won't be printed

    return 0;
}
```

Fixed-width & “least/fast” integer types

- `std::int_leastN_t / std::uint_leastN_t`
    Guarantee at least N bits. They are the smallest available types with ≥ N bits.
    Example: if a weird target has no 8-bit type, int_least8_t could indeed be 16-bit.
- `std::int_fastN_t / std::uint_fastN_t`
    Guarantee at least N bits, but choose the fastest type on the target. Commonly `int_fast8_t` is often just `int` on many platforms.
- `std::intmax_t / std::uintmax_t`
    The largest standard integer types available. Often `long long/unsigned long long`, but could be something larger if the implementation has it.

They’re typedefs (aliases) to some underlying integer types chosen by the implementation. Often they map to int, long, or long long, but that’s not guaranteed or standardized beyond the bit-width/ordering promises.


On most platforms, `std::uint8_t` is a typedef to` unsigned char` (only if there’s an exact 8-bit type). Streaming a `uint8_t*` (i.e., `unsigned char*`) to `std::cout` selects the C-string overload, so it will try to print bytes until a `'\0'`, not the pointer value. To print the address, cast to `const void*`:
```cpp
std::uint8_t* p = ...;
std::cout << static_cast<const void*>(p) << '\n';
```

If you actually want to print the numeric byte value of a `uint8_t`, cast to an int-sized type first:
```cpp
std::uint8_t b = 200;
std::cout << static_cast<unsigned>(b) << '\n';  // prints 200, not a char
```

For raw memory/manipulation semantics (not integers), consider `std::byte` instead of `std::uint8_t`.

If you need exact widths in interfaces or on-disk formats, prefer the fixed-width types (`std::int32_t`, etc.). For general arithmetic where exact width isn’t critical, plain int is often simplest and fastest.

If you truly want 16-bit wraparound semantics in an expression, cast after the operation or use a fixed-width type and a final cast:
```cpp
std::uint16_t a = 0xFF00, b = 0x0100;
auto sum = static_cast<std::uint16_t>(a + b); // wraps to 0
```

Bit manipulation of unsigned integers using <bit> in c++20

`std::has_single_bit` checks if the number is a power of two (only one bit set).
Defined for unsigned integers. Passing zero always yields false.
Has to cast to an unsigned before calling if the number if not already unsigned.
```cpp
#include <bit>
#include <iostream>

int main() {
    std::cout << std::has_single_bit(8u) << "\n";  // true (1000b)
    std::cout << std::has_single_bit(10u) << "\n"; // false (1010b)
}

// equivalent to the classic trick
(x != 0) && ((x & (x - 1)) == 0)
```

`std::bit_ceil` 
    If the input is already a power of two (has_single_bit(x) true) -> return x.
    Else, return the next power of two above x
    Implementation-wise, usually done by finding the position of the highest set bit and then shifting 1 one step higher
    e.g. Input = 5 (0101b) highest bit is at position 2 (value = 4). Next power of two = 1 << 3 = 8 (shift the bits of 1 three positions to the left)
    If the next power of two doesn’t fit in the type (e.g., `std::bit_ceil(UINT_MAX))`, the behavior is undefined. The standard leaves it up to the implementation.
```cpp
#include <bit>
#include <iostream>

int main() {
    std::cout << std::bit_ceil(5u) << "\n";   // 8
    std::cout << std::bit_ceil(8u) << "\n";   // 8
    std::cout << std::bit_ceil(0u) << "\n";   // 1 (!)
}
```

`std::bit_floor` Largest power of two <= x 
```cpp
std::bit_floor(10u) // 8
std::bit_floor(8u)  // 8
```

`std::bit_width(x)` Index of the most significant bit + 1. (i.e., the number of bits needed to represent x)
```cpp
std::bit_width(8u)  // 4 (1000b)
std::bit_width(9u)  // 4 (1001b)
std::bit_width(0u)  // 0
```

`std::rotl / std::rotr` Rotate left / right by a given shift.
```cpp
/**
0b0001u in binary is ...00000001
the u suffix specifies it is unsigned. The number of bits N depends on the underlying type (e.g., unsigned int)
to perform a left shift by 2 positions
The bits are shifted to the left by the specified number of positions
The bits that "fall off" the left end are reinserted at the right end
...00000001  -> ...00000100, the 1 doesn't reach the leftmost bit
*/
std::rotl(0b0001u, 2) // 0b0100
```

`std::countl_zero, std::countl_one, std::countr_zero, std::countr_one` count leading/trailing zeros/ones

`std::popcount(x)` count the number of 1 bits (population count).