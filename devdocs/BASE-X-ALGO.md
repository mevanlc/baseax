# The base-x algorithm

base-x converts an arbitrary sequence of bytes to and from digits drawn from a
caller-supplied alphabet. It treats the bytes as a big-endian base-256 number
and changes that number's radix to the size of the alphabet.

Ordinary integer conversion would discard leading zero bytes. base-x preserves
them with an additional rule: each leading zero byte is represented by one
leading occurrence of the alphabet's zero symbol.

## Terms and preconditions

Let:

- `alphabet` be an ordered sequence of distinct symbols;
- `B` be the number of symbols in `alphabet`;
- `alphabet[i]` be the symbol representing digit value `i`; and
- `leader = alphabet[0]` be the zero symbol.

The order of the alphabet is significant. Two alphabets containing the same
symbols in different orders define different encodings.

The algorithm requires at least two distinct symbols. This implementation also
uses an 8-bit reverse lookup table: alphabet symbols must have character codes
from 0 through 255, and the alphabet may contain at most 254 symbols. The value
255 is reserved in the table to mean "not in the alphabet."

## Numeric model

For input bytes `b[0], b[1], ..., b[n-1]`, the represented integer is:

```text
b[0] * 256^(n-1) + b[1] * 256^(n-2) + ... + b[n-1]
```

Encoding expresses this integer using base-`B` digits. Decoding performs the
inverse conversion. The conversion itself cannot retain leading zero bytes,
because adding zero-valued digits to the front does not change an integer.
Counting and restoring those bytes separately makes the mapping reversible.

## Encoding

1. Return an empty string for an empty byte sequence.
2. Count and skip the leading zero bytes.
3. Convert the remaining big-endian base-256 digits to base `B`.
4. Prefix the result with one `leader` for each skipped zero byte.
5. Replace every base-`B` digit with the symbol at that index in `alphabet`.

The following pseudocode stores temporary base-`B` digits least-significant
first. This makes it easy to extend the number as carries are produced.

```text
function encode(input, alphabet):
    if input is empty:
        return ""

    B = length(alphabet)
    leader = alphabet[0]

    zero_count = number of consecutive 0 bytes at the start of input
    digits = empty list  // base-B digits, least-significant first

    for each byte in input after the leading zero bytes:
        // Incorporate the next base-256 digit:
        //     value = value * 256 + byte
        carry = byte

        for i from 0 through length(digits) - 1:
            accumulator = digits[i] * 256 + carry
            digits[i] = accumulator modulo B
            carry = floor(accumulator / B)

        while carry is not 0:
            append (carry modulo B) to digits
            carry = floor(carry / B)

    output = leader repeated zero_count times

    for each digit in digits, from last to first:
        append alphabet[digit] to output

    return output
```

The implementation avoids repeated growth at the front of an array by
preallocating a conservative upper bound and keeping the active digits aligned
to its right. For `n` nonzero-leading input bytes, the bound is:

```text
floor(n * log(256) / log(B) + 1)
```

## Decoding

Decoding first maps each symbol back to its digit value. A reverse lookup table
constructed from the alphabet makes this lookup constant-time.

1. Return an empty byte sequence for an empty string.
2. Count and skip leading `leader` symbols. Each represents a complete zero
   byte, not merely a zero-valued base-`B` digit.
3. Reject the input if any remaining symbol is absent from the alphabet.
4. Convert the remaining big-endian base-`B` digits to base 256.
5. Prefix the converted bytes with one zero byte for each skipped `leader`.

```text
function decode(input, alphabet):
    if input is empty:
        return empty byte sequence

    B = length(alphabet)
    leader = alphabet[0]
    value_of = map each alphabet symbol to its index

    zero_count = number of consecutive leader symbols at the start of input
    bytes = empty list  // base-256 digits, least-significant first

    for each symbol in input after the leading leaders:
        if symbol is not in value_of:
            fail with "invalid character"

        // Incorporate the next base-B digit:
        //     value = value * B + value_of[symbol]
        carry = value_of[symbol]

        for i from 0 through length(bytes) - 1:
            accumulator = bytes[i] * B + carry
            bytes[i] = accumulator modulo 256
            carry = floor(accumulator / 256)

        while carry is not 0:
            append (carry modulo 256) to bytes
            carry = floor(carry / 256)

    output = zero_count zero bytes

    for each byte in bytes, from last to first:
        append byte to output

    return output
```

The implementation likewise preallocates a conservative decoding buffer. For
`m` symbols after the leading leaders, the bound is:

```text
floor(m * log(B) / log(256) + 1)
```

The public `decode` operation reports invalid input as an error. The
`decodeUnsafe` variant performs the same algorithm but returns no result when it
encounters a symbol outside the alphabet.

## Leading zero example

With the base-16 alphabet `0123456789abcdef`, decoding `000f` does **not** use
the usual two-hex-digits-per-byte convention. The three leading `0` symbols
produce three zero bytes, and the remaining `f` represents the integer 15:

```text
encoded symbols:  0    0    0    f
decoded bytes:   00   00   00   0f
```

This is why base-x is not a standards-compatible implementation of base16,
base32, or base64. It uses those alphabet sizes for integer radix conversion,
not the fixed-width bit grouping defined by those formats.

## Complexity

If `n` is the number of input bytes and `d` is the number of output digits,
encoding takes `O(n * d)` time and `O(d)` auxiliary space. If `m` is the number
of encoded symbols and `k` is the number of decoded bytes, decoding takes
`O(m * k)` time and `O(k)` auxiliary space. In the usual fixed-base case, both
time bounds are quadratic in the input length and both space bounds are linear.

## Optimization Opportunities

The current algorithm uses one array element per base-`B` digit when encoding
and one array element per byte when decoding. Its arithmetic is the schoolbook
form of arbitrary-precision radix conversion: each new input unit propagates a
carry through the digits accumulated so far. Several faster implementations
could preserve the exact same base-x wire format.

### Power-of-two bases

When `B = 2^q`, multiplication and division by `B` can be replaced with direct
bit grouping. For example, an alphabet of 64 symbols has six-bit digits. An
encoder can count the leading zero bytes, find the significant bit length of
the remaining integer, and split those bits into `q`-bit groups aligned from
the least-significant end. Decoding joins the groups in the opposite direction.

This reduces both operations to `O(n)` time. It must still preserve base-x's
special handling of leading zero bytes and its variable-width first digit. A
base64-sized specialization would therefore produce base-x output more quickly;
it would not produce standard base64.

### Wider limbs and digit chunks

For any base, an implementation can store several base-`B` digits in each
machine-word-sized limb. Choose the largest `p` for which `B^p` fits safely in a
limb, and perform arithmetic in base `B^p`. Input bytes can likewise be consumed
in chunks representing powers of 256. This reduces loop iterations and carry
operations, although the schoolbook form remains quadratic for fixed `B`.

The limb width must leave enough room for intermediate products and carries.
Languages without checked or sufficiently wide integer operations may need
explicit overflow handling.

### Large-integer conversion algorithms

For large inputs, divide-and-conquer radix conversion or a well-optimized
arbitrary-precision integer library can improve on digit-at-a-time schoolbook
conversion. Such an implementation can import the byte sequence as one
big-endian integer and repeatedly split it by large powers of `B`, or combine
chunks recursively. Whether this is faster in practice depends on input size,
allocation costs, and the host language's large-integer implementation.

### SIMD and table-driven operations

SIMD is especially applicable to power-of-two bit packing. It can also speed up
symbol validation and the independent final mappings between digit values and
alphabet symbols. The generic radix-conversion loop is harder to vectorize
because each position's carry depends on the preceding arithmetic. Wider scalar
limbs or a different large-integer algorithm will usually provide the first
substantial improvement there.

All optimized paths must retain the same alphabet ordering, empty-input result,
leading-zero convention, and invalid-symbol behavior. They should be tested
against the generic algorithm over boundary sizes, every supported base, long
runs of leading zeroes, and randomized inputs.
