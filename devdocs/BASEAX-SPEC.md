# The BaseAX specification

BaseAX ("base any x") is a self-describing textual encoding of arbitrary byte
sequences. It wraps the [base-x algorithm](BASE-X-ALGO.md) so that the encoded
string alone determines the alphabet used to produce it: a decoder needs no
out-of-band knowledge of which alphabet the encoder chose.

An encoder picks any alphabet that suits its transport — URL fragments,
filenames, hostile text fields — and the decoder recovers that alphabet from
the message itself. Hence "base any x".

## Conformance

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are to be
interpreted as described in RFC 2119 and RFC 8174.

## How it works

A base-x string is not decodable without its alphabet. BaseAX removes that
dependency with one observation: if the encoder guarantees that **every**
alphabet symbol appears somewhere in the output, then the alphabet is simply
the set of distinct characters in the output.

The base-x payload naturally contains only the symbols the converted number
happens to use, so BaseAX appends the missing ones:

```text
<baseax> := <payload> <separator> <tail>
```

- `<payload>` is the base-x encoding of the input bytes.
- `<separator>` is a repetition of the payload's first character. It marks
  where the payload ends.
- `<tail>` contains every alphabet character that does not occur in the
  payload, each exactly once, in any order.

The union of characters in payload, separator, and tail is exactly the
alphabet. The decoder derives the alphabet by collecting the distinct
characters of the whole string and sorting them.

## Character repertoire

Every character of a BaseAX string MUST be drawn from the 94 printable ASCII
characters excluding space — the regular-expression class `[!-~]`, code points
U+0021 through U+007E inclusive.

This repertoire keeps BaseAX strings safe to embed in line-oriented text: no
whitespace, no control characters, no characters outside ASCII.

## Alphabets

An alphabet is a **set** of at least 2 and at most 94 repertoire characters.

Unlike base-x, where two orderings of the same symbols define two different
encodings, BaseAX defines the alphabet by its character set only. Before use,
an alphabet MUST be put in canonical form: duplicate characters removed and
the remainder sorted in ascending code-point order. Encoders MUST
canonicalize a caller-supplied alphabet string before base-x conversion, and
decoders MUST sort the characters they collect from the input. This is what
makes self-description possible — both sides can reconstruct the same ordered
alphabet from an unordered set.

The canonical alphabet's first (smallest) character is the base-x `leader`:
leading `0x00` input bytes appear as leading occurrences of this character in
the payload.

### Predefined alphabets

`BASECHARS_ALL` — the full 94-character repertoire, for transports that
accept arbitrary printable ASCII:

```text
!"#$%&'()*+,-./0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwxyz{|}~
```

`BASECHARS_URLFRAGSAFE` — 77 characters, a conservative subset of the
characters RFC 3986 permits literally in a URI fragment. It additionally
excludes `$`, `'`, `(`, and `)`, which are fragment-legal but commonly
mangled when URLs are quoted in shells or embedded in prose and markup:

```text
!&*+,-./0123456789:;=?@ABCDEFGHIJKLMNOPQRSTUVWXYZ_abcdefghijklmnopqrstuvwxyz~
```

Both predefined strings are already in canonical form.

## Wire format

For a canonical alphabet of `B` characters and a non-empty input:

```text
+---------------------------+-----------+----------------------------------+
| payload                   | separator | tail                             |
| base-x digits, >= 1 char  | 1 char    | B - U chars, any order           |
+---------------------------+-----------+----------------------------------+
```

where `U` is the number of distinct characters occurring in the payload.

The format maintains four invariants:

1. The payload is non-empty, and the separator equals its first character.
2. The tail never contains the separator character, because that character
   necessarily occurs in the payload. The appended separator is therefore the
   **last** occurrence of the string's first character, which makes the
   framing unambiguous.
3. Each alphabet character absent from the payload occurs exactly once in the
   tail; tail and payload share no characters.
4. The set of distinct characters in the whole string is exactly the
   alphabet.

Two consequences are worth noting. Every BaseAX string contains at least
`B + 1` characters (each alphabet character at least once, plus the
separator's duplicate). And two encodings of the same bytes with the same
alphabet may differ in tail order, so applications MUST NOT compare BaseAX
strings to decide whether the underlying data is equal — compare decoded
bytes instead.

Tail order carries no information. Encoders MAY emit the tail in any order,
including a randomized one. Where deterministic output matters (tests,
content-addressed storage, string comparison), encoders SHOULD emit the tail
in ascending code-point order; call this the canonical encoding.

## Encoding

1. Canonicalize the alphabet; reject it unless it contains 2 through 94
   distinct repertoire characters.
2. Return the empty string for an empty byte sequence (see "Empty input").
3. Encode the bytes with base-x over the canonical alphabet; this is the
   payload.
4. Append the payload's first character as the separator.
5. Append each alphabet character that does not occur in the payload, once,
   in any order.

```text
function baseax_encode(input, base_chars):
    alphabet = sort(distinct characters of base_chars)
    if length(alphabet) < 2 or length(alphabet) > 94:
        fail with "invalid alphabet"
    if any character of alphabet is outside [!-~]:
        fail with "invalid alphabet"

    if input is empty:
        return ""

    payload   = basex_encode(input, alphabet)      // see BASE-X-ALGO.md
    separator = payload[0]
    tail      = characters of alphabet not occurring in payload, any order

    return payload + separator + tail
```

## Decoding

1. Return an empty byte sequence for an empty string (see "Empty input").
2. Read the first character; find its **last** occurrence in the string. That
   occurrence is the separator. If the first character never reappears, the
   input is not a BaseAX string.
3. Split: everything before the separator is the payload; everything after it
   is the tail.
4. Derive the alphabet: the distinct characters of the entire input, sorted
   in ascending code-point order.
5. Decode the payload with base-x over the derived alphabet.

```text
function baseax_decode(input):
    if input is empty:
        return empty byte sequence

    separator = input[0]
    sep_index = index of the last occurrence of separator in input
    if sep_index == 0:
        fail with "invalid baseax string"

    payload  = input[0 .. sep_index - 1]           // inclusive bounds
    tail     = input[sep_index + 1 .. end]
    alphabet = sort(distinct characters of input)

    return basex_decode(payload, alphabet)
```

The framing can equivalently be expressed as the greedy regular expression
`(.)(.*)\1(.*)`, anchored to the whole input: group 1 plus group 2 is the
payload, and group 3 is the tail. Greedy matching of `(.*)` is what selects
the last occurrence of the first character. Implementations using this form
MUST anchor the match to the full string; an unanchored match can skip a
corrupted prefix and silently decode garbage.

Because the alphabet is derived from the input itself, base-x's
invalid-character error cannot occur: every payload character is in the
derived alphabet by construction. A malformed or corrupted string therefore
tends to decode to garbage rather than fail. Decoders SHOULD apply strict
validation and reject the input if any check fails:

- The derived alphabet has at least 2 characters.
- Every character of the input is in `[!-~]`.
- No tail character occurs in the payload, and no character occurs twice in
  the tail (invariant 3).

Applications that expect a particular alphabet SHOULD additionally verify
that the derived alphabet equals (or is an acceptable subset of) the expected
one.

## Empty input

The empty byte sequence encodes to the empty string, and the empty string
decodes to the empty byte sequence. This mirrors base-x's own convention. An
empty message carries no alphabet, which is harmless: there is nothing to
decode.

(See "Reference implementation notes" — the reference encoder currently
mishandles this case.)

## Worked examples

Tails are shown in canonical ascending order; any tail permutation encodes
the same bytes.

Alphabet `01` (B = 2), input `0x05` (binary `101`):

```text
payload    101
separator  1
tail       (empty — both symbols occur in the payload)
encoded    1011
```

Alphabet `0123456789abcdef` (B = 16), input `0x48 0x69` ("Hi"):

```text
payload    4869
separator  4
tail       012357abcdef
encoded    48694012357abcdef
```

The same alphabet, input `0x00 0x48 0x69` — the leading zero byte becomes a
leading `0`, the alphabet's leader:

```text
payload    04869
separator  0
tail       12357abcdef
encoded    04869012357abcdef
```

The same alphabet, input `0x00` — the whole payload is one leader symbol, so
the separator repeats it:

```text
payload    0
separator  0
tail       123456789abcdef
encoded    00123456789abcdef
```

`BASECHARS_URLFRAGSAFE` (B = 77), input `Hello, World!` as UTF-8:

```text
payload    +i9;DVqIJ~TQ&0:=_
separator  +
encoded    +i9;DVqIJ~TQ&0:=_+!*,-./12345678?@ABCEFGHKLMNOPRSUWXYZabcdefghjklmnoprstuvwxyz
```

All 17 payload characters are distinct, so the tail supplies the remaining
60 alphabet characters and the total length is 78 = 17 + 1 + 60.

## Size and complexity

The payload has the base-x length: roughly `n * 8 / log2(B)` characters for
`n` input bytes — about 1.22× the byte count for B = 94, 1.28× for B = 77,
2× for B = 16. Framing adds `1 + (B - U)` characters, where `U` is the
number of distinct payload characters. For long inputs the payload usually
covers the alphabet and the overhead approaches a single character; for short
inputs the tail dominates, since output length never drops below `B + 1`.
Encoding one byte under `BASECHARS_ALL` costs about 96 characters. When
inputs are tiny and a large alphabet is not required, a smaller alphabet
yields shorter output.

Time and auxiliary space are inherited from base-x — `O(n * d)` schoolbook
radix conversion — plus `O(L + B)` for framing, where `L` is the string
length. The optimizations described in BASE-X-ALGO.md apply unchanged to the
payload.

## Relationship to base-x

- BaseAX inherits base-x's leading-zero rule and is therefore **not** a
  standards-compatible base16, base32, or base64, even at those alphabet
  sizes.
- base-x treats alphabet order as significant; BaseAX deliberately does not.
  Canonicalization is what lets the decoder reconstruct the alphabet from an
  unordered character set.
- A BaseAX string is not a valid input to a plain base-x decoder (the
  separator and tail would be decoded as digits), and vice versa.

## Layered profiles (informative)

BaseAX encodes bytes. Common layerings, as used by the reference codebase:

- **Text**: encode the string as UTF-8, then BaseAX-encode the bytes; reverse
  on decode.
- **baseax-z**: compress with raw DEFLATE (RFC 1951, `deflate-raw`) before
  encoding; inflate after decoding.
- **baseax-z-c**: compress, then encrypt, then encode; decode, then decrypt,
  then inflate. Compression MUST precede encryption — ciphertext does not
  compress.

Each layer transforms bytes before the BaseAX step or after its inverse; the
wire format defined here is unaffected. The originating application
(AccuDiff) uses baseax-z-c with `BASECHARS_URLFRAGSAFE` to carry deflated,
encrypted application state in URI fragments.

## Security considerations

BaseAX provides no integrity or authenticity. Any string that satisfies the
framing rule decodes to *some* byte sequence, and because the alphabet is
derived from the (possibly corrupted) input, corruption is more likely to
yield garbage output than a decode error. Applications MUST validate decoded
data, and SHOULD use an authenticated layer (for example an AEAD in a
baseax-z-c-style profile) when tampering matters.

A randomized tail is a covert channel: with `B - U` unused characters an
encoder can smuggle up to `log2((B - U)!)` bits in the permutation, and a
non-cryptographic shuffle can leak PRNG state. Contexts where this matters
SHOULD require canonical (sorted) tails.

## Reference implementation notes (informative)

The reference implementation lives in the AccuDiff codebase:
`src/base/baseax.ts`, with the layered profiles in `src/base/baseax-z.ts`
and `src/base/baseax-z-c.ts`, and tests under `__tests__/`. It builds on the
`base-x` npm package. Divergences from this specification, verified against
that code:

- **Empty input (erratum).** `baseaxEncode` on an empty buffer produces the
  literal string `undefined` followed by the tail (JavaScript string
  concatenation of an out-of-range index), and `baseaxDecode("")` throws.
  This specification defines empty ↔ empty instead.
- **Lenient framing.** The decoder applies `/(.)(.*)\1(.*)/` unanchored. An
  input whose first character never repeats can match at a later offset,
  silently ignoring a corrupted prefix while still deriving the alphabet
  from the entire input — for example, `xabca` decodes to `0x00 0x06` rather
  than failing. Strict decoders reject such inputs.
- **No strict validation.** The reference decoder performs none of the
  RECOMMENDED checks (repertoire membership, minimum alphabet size, tail
  disjointness).
- **Tail shuffle.** Encoding shuffles the tail by default via
  `sort(() => Math.random() - 0.5)` — a biased, non-cryptographic shuffle.
  This conforms (tail order is insignificant) but is not a canonical
  encoding and is unsuitable where the covert-channel consideration above
  applies.

## References

- [BASE-X-ALGO.md](BASE-X-ALGO.md) — the underlying radix-conversion
  algorithm, including the leading-zero convention.
- `base-x` — <https://github.com/cryptocoinjs/base-x>.
- RFC 1951 — DEFLATE, used by the baseax-z profile.
- RFC 3986 — URI syntax, background for `BASECHARS_URLFRAGSAFE`.
- RFC 2119 / RFC 8174 — requirement keywords.
