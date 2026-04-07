# SCALE Codec & Runtime Metadata Deep Dive: From Bits to Extrinsic Parsing

> A comprehensive walkthrough of SCALE encoding/decoding in the Polkadot ecosystem — from binary fundamentals and compact encoding, through the runtime metadata type registry, to byte-by-byte parsing of a real signed extrinsic. Based on Joseph's (Papi team lead) PBA lecture and extended with source code analysis of `parity-scale-codec` and `frame-metadata`.

## Table of Contents

- [SCALE Codec \& Runtime Metadata Deep Dive: From Bits to Extrinsic Parsing](#scale-codec--runtime-metadata-deep-dive-from-bits-to-extrinsic-parsing)
  - [Table of Contents](#table-of-contents)
- [Part I: Foundations](#part-i-foundations)
  - [1. Serialization](#1-serialization)
  - [2. Codecs](#2-codecs)
  - [3. Bytes and Hexadecimal](#3-bytes-and-hexadecimal)
    - [Why hexadecimal?](#why-hexadecimal)
    - [Notation conventions](#notation-conventions)
- [Part II: Little-Endian and Free Casting](#part-ii-little-endian-and-free-casting)
  - [4. Endianness](#4-endianness)
    - [Q: Does endianness apply to bits inside a byte?](#q-does-endianness-apply-to-bits-inside-a-byte)
  - [5. Free Casting — The Copy-Free Property](#5-free-casting--the-copy-free-property)
    - [Same value, different types — the pattern](#same-value-different-types--the-pattern)
    - [Q: Why did SCALE choose little-endian?](#q-why-did-scale-choose-little-endian)
    - [Q: Are there benefits to big-endian?](#q-are-there-benefits-to-big-endian)
- [Part III: SCALE Primitives](#part-iii-scale-primitives)
    - [Q: Why not use an existing standard like Protobuf?](#q-why-not-use-an-existing-standard-like-protobuf)
  - [6. Unsigned Integers (U8 to U256)](#6-unsigned-integers-u8-to-u256)
  - [7. Signed Integers and Two's Complement](#7-signed-integers-and-twos-complement)
    - [Q: How do you know if it's a negative number or just a large positive?](#q-how-do-you-know-if-its-a-negative-number-or-just-a-large-positive)
    - [Q: Is two's complement related to endianness?](#q-is-twos-complement-related-to-endianness)
  - [8. Booleans](#8-booleans)
    - [Q: Why not pack multiple booleans into one byte using the first bits as a counter?](#q-why-not-pack-multiple-booleans-into-one-byte-using-the-first-bits-as-a-counter)
- [Part IV: Compact Encoding — SCALE's Signature Primitive](#part-iv-compact-encoding--scales-signature-primitive)
  - [9. The Problem Compact Solves](#9-the-problem-compact-solves)
  - [10. How Compact Works — The Four Modes](#10-how-compact-works--the-four-modes)
    - [Q: What are those 2 flag bits for?](#q-what-are-those-2-flag-bits-for)
  - [11. Mode 0 — Single Byte (flag 00)](#11-mode-0--single-byte-flag-00)
  - [12. Mode 1 — Two Bytes (flag 01)](#12-mode-1--two-bytes-flag-01)
    - [Step-by-step: encoding 64](#step-by-step-encoding-64)
    - [Step-by-step: encoding 65](#step-by-step-encoding-65)
    - [Step-by-step: encoding 100](#step-by-step-encoding-100)
    - [Decoding mode 1 — reverse process](#decoding-mode-1--reverse-process)
  - [13. Mode 2 — Four Bytes (flag 10)](#13-mode-2--four-bytes-flag-10)
  - [14. Mode 3 — Big Integer (flag 11)](#14-mode-3--big-integer-flag-11)
    - [Q: In modes 0, 1, 2 the 6 bits are part of the value. What changes in mode 3?](#q-in-modes-0-1-2-the-6-bits-are-part-of-the-value-what-changes-in-mode-3)
    - [Concrete example: encoding a value that needs 7 bytes](#concrete-example-encoding-a-value-that-needs-7-bytes)
  - [15. The Validity Trap — Not All Permutations Are Valid](#15-the-validity-trap--not-all-permutations-are-valid)
  - [16. When to Use Compact (and When Not To)](#16-when-to-use-compact-and-when-not-to)
    - [Q: Why isn't compact used by default for everything?](#q-why-isnt-compact-used-by-default-for-everything)
  - [17. Compact in Source Code — parity-scale-codec](#17-compact-in-source-code--parity-scale-codec)
    - [Encoding (CompactRef for u32)](#encoding-compactref-for-u32)
    - [Mode 3 for large numbers (u64)](#mode-3-for-large-numbers-u64)
    - [Decoding (Compact for u64)](#decoding-compact-for-u64)
    - [Range validation — the trap in code](#range-validation--the-trap-in-code)
    - [Test cases confirm everything](#test-cases-confirm-everything)
- [Part V: Complex Types](#part-v-complex-types)
  - [18. Enum (Tagged Union)](#18-enum-tagged-union)
    - [Circular definitions](#circular-definitions)
    - [Limit: 256 variants maximum](#limit-256-variants-maximum)
  - [19. Option and Result — Specialized Enums](#19-option-and-result--specialized-enums)
    - [Option](#option)
    - [Result](#result)
  - [20. Struct and Tuple](#20-struct-and-tuple)
    - [Decoding a struct with multi-byte fields — worked example](#decoding-a-struct-with-multi-byte-fields--worked-example)
    - [Struct with vector and compact fields — worked example](#struct-with-vector-and-compact-fields--worked-example)
  - [21. Vector (Dynamic Length)](#21-vector-dynamic-length)
    - [Vector of compacts](#vector-of-compacts)
  - [22. Array (Fixed Length)](#22-array-fixed-length)
  - [23. Opaque](#23-opaque)
  - [24. Types That Are NOT First-Class in SCALE](#24-types-that-are-not-first-class-in-scale)
    - [Strings](#strings)
    - [Maps and Sets](#maps-and-sets)
    - [Q: What is the maximum size of a vector?](#q-what-is-the-maximum-size-of-a-vector)
  - [25. Bit Vectors (bitvec)](#25-bit-vectors-bitvec)
    - [How bitvec is SCALE-encoded](#how-bitvec-is-scale-encoded)
    - [Encoding example: 10 flags](#encoding-example-10-flags)
    - [In the Type Registry](#in-the-type-registry)
    - [Real-world usage: availability bitfields](#real-world-usage-availability-bitfields)
- [Part VI: The derive Macros — Automatic Codec Generation](#part-vi-the-derive-macros--automatic-codec-generation)
  - [26. #\[derive(Encode, Decode)\]](#26-deriveencode-decode)
    - [For structs](#for-structs)
    - [For enums](#for-enums)
  - [27. Codec Attributes](#27-codec-attributes)
  - [28. TypeInfo — Bridging to Metadata](#28-typeinfo--bridging-to-metadata)
- [Part VII: Runtime Metadata](#part-vii-runtime-metadata)
  - [29. What Metadata Solves](#29-what-metadata-solves)
  - [30. The Metadata Blob Structure](#30-the-metadata-blob-structure)
    - [Q: How do you decode the metadata if the metadata is what tells you how to decode things?](#q-how-do-you-decode-the-metadata-if-the-metadata-is-what-tells-you-how-to-decode-things)
    - [Metadata Versions and Runtime APIs](#metadata-versions-and-runtime-apis)
  - [31. The Type Registry — Single Source of Truth](#31-the-type-registry--single-source-of-truth)
    - [The core structures](#the-core-structures)
    - [TypeDef — maps directly to SCALE types](#typedef--maps-directly-to-scale-types)
    - [Concrete example: AccountInfo in the registry](#concrete-example-accountinfo-in-the-registry)
    - [The resolution process — recursive tree walk](#the-resolution-process--recursive-tree-walk)
    - [Variant (enum) in the registry](#variant-enum-in-the-registry)
    - [Sequence (Vec) in the registry](#sequence-vec-in-the-registry)
    - [Q: Where does type\_id\_99 come from? Who assigns it?](#q-where-does-type_id_99-come-from-who-assigns-it)
  - [32. How Pallets Reference the Registry](#32-how-pallets-reference-the-registry)
    - [PalletMetadata structure](#palletmetadata-structure)
    - [Calls — dispatchable functions](#calls--dispatchable-functions)
    - [Events and Errors — same pattern](#events-and-errors--same-pattern)
    - [Constants — pre-encoded values](#constants--pre-encoded-values)
  - [33. Storage — The Richest Part](#33-storage--the-richest-part)
    - [Plain storage — a single global value](#plain-storage--a-single-global-value)
    - [Map storage — one entry per key](#map-storage--one-entry-per-key)
    - [Q: How does the metadata connect to the key-value store?](#q-how-does-the-metadata-connect-to-the-key-value-store)
    - [Building a storage key — step by step](#building-a-storage-key--step-by-step)
    - [Hashers and why they matter](#hashers-and-why-they-matter)
    - [Double maps](#double-maps)
    - [Complete flow: reading a balance](#complete-flow-reading-a-balance)
- [Part VIII: Extrinsic Structure](#part-viii-extrinsic-structure)
  - [34. What Is an Extrinsic](#34-what-is-an-extrinsic)
  - [35. High-Level Layout](#35-high-level-layout)
    - [1. Compact Length](#1-compact-length)
    - [2. Header Byte](#2-header-byte)
    - [3. Address + Signature + Extensions (signed only)](#3-address--signature--extensions-signed-only)
    - [4. Call Data](#4-call-data)
  - [36. Signed Extensions — Extra vs Additional Signed](#36-signed-extensions--extra-vs-additional-signed)
    - [Q: What is additional\_signed? Why doesn't it travel in the extrinsic?](#q-what-is-additional_signed-why-doesnt-it-travel-in-the-extrinsic)
    - [Q: How does the node know which additional\_signed values the sender used?](#q-how-does-the-node-know-which-additional_signed-values-the-sender-used)
    - [Q: Do I sign a hash of the transaction or the whole thing?](#q-do-i-sign-a-hash-of-the-transaction-or-the-whole-thing)
    - [Q: Where in the metadata are the extensions defined?](#q-where-in-the-metadata-are-the-extensions-defined)
    - [Typical signed extensions in Polkadot](#typical-signed-extensions-in-polkadot)
    - [CheckMetadataHash — Trustless Metadata Verification via Merkle Tree](#checkmetadatahash--trustless-metadata-verification-via-merkle-tree)
  - [37. Parsing a Real Extrinsic — Byte by Byte](#37-parsing-a-real-extrinsic--byte-by-byte)
    - [1. Compact Length](#1-compact-length-1)
    - [2. Header Byte](#2-header-byte-1)
    - [3. Address (MultiAddress)](#3-address-multiaddress)
    - [4. Signature (MultiSignature)](#4-signature-multisignature)
    - [5. Signed Extensions (extras)](#5-signed-extensions-extras)
    - [6. Call Data](#6-call-data)
    - [Decoded result](#decoded-result)
- [Part IX: Tools and Implementations](#part-ix-tools-and-implementations)
  - [38. Libraries](#38-libraries)
  - [39. Interactive Tools](#39-interactive-tools)
- [Appendix: Key Concepts Quick Reference](#appendix-key-concepts-quick-reference)

---

# Part I: Foundations

## 1. Serialization

Serialization is the process of converting an in-memory data structure into a format that can be stored or transmitted, and later reconstructed. The most familiar example: `JSON.stringify()` in JavaScript serializes an object into a string, and `JSON.parse()` deserializes it back.

There are two families of serialization:

**Self-describing** formats include metadata about the structure alongside the values. JSON is the classic example — `{"name": "Alice", "age": 30}` tells you the field names, types, and structure just by looking at the data. Easy to debug, but verbose.

**Non-self-describing** formats contain only the raw values. Without knowing the type definitions beforehand, you cannot decode the data. SCALE falls in this category. It's compact and performant, but requires the schema (type definitions) at both encoding and decoding ends.

The trade-off: self-describing is heavier but debuggable; non-self-describing is compact and fast but requires external type information. SCALE chose the latter — optimized for resource-constrained environments like blockchain runtimes.

## 2. Codecs

A codec is the combination of an **encoder** and a **decoder**. The encoder takes a type (a number, a struct, anything) and produces binary data. The decoder takes that binary data and produces the original type.

The key constraint: the **input type of the encoder** is the same as the **output type of the decoder**. If the encoder takes a `u32` and produces bytes, the decoder takes those bytes and produces a `u32`.

In SCALE, everything is thought of in terms of codecs. Every type has its codec. Complex types are codecs that reference other codecs.

## 3. Bytes and Hexadecimal

A **byte** is 8 bits — the smallest addressable unit of memory in virtually all computer architectures. When you access memory, the minimum chunk you get back is one byte.

A **nibble** is half a byte: 4 bits.

### Why hexadecimal?

Hexadecimal (base 16) uses 16 symbols: 0-9 and A-F. Each hex symbol represents exactly one nibble (4 bits). Therefore, 2 hex characters = 1 byte exactly.

For example, `6A` in hex:
- `6` = `0110` (first nibble)
- `A` = `1010` (second nibble, A=10)
- Together: `0110 1010` = one byte

This clean correspondence is why hex is universally used for binary data. Base 64 would compress more, but you'd lose the ability to tell from a single character whether you're looking at one byte or two.

### Notation conventions

- `0x` prefix indicates hexadecimal: `0x6A`
- `0b` prefix indicates binary: `0b01101010`

---

# Part II: Little-Endian and Free Casting

## 4. Endianness

When a value occupies more than one byte (e.g., a `u32` = 4 bytes), the question is: in which order do you place those bytes in memory?

Take the value **15** as a `u32`. In hex it's `0x0000000F` — four bytes: `00`, `00`, `00`, `0F`.

**Big-endian**: most significant byte first → `00 00 00 0F`. Reads naturally left-to-right.

**Little-endian**: least significant byte first → `0F 00 00 00`. Counterintuitive for humans, but has performance advantages.

### Q: Does endianness apply to bits inside a byte?

No. Endianness applies only at the **byte level**. Inside each byte, the bits stay in the same order. That's why the value 15 in little-endian is `0F 00 00 00` and NOT `F0 00 00 00` — the byte `0F` itself is unchanged; only the order of the four bytes is reversed.

Joseph emphasized this point in the lecture: when he first learned about endianness, he assumed it would be a complete mirror (including bits), but it's not. The byte is a black box — endianness only reorders the boxes.

## 5. Free Casting — The Copy-Free Property

This is why SCALE chose little-endian. Take the value **1000** as a `u32`:

- **Little-endian**: `E8 03 00 00`
- **Big-endian**: `00 00 03 E8`

Now cast it to a `u16` (you know 1000 fits in 2 bytes):

- **Little-endian** (`E8 03 00 00`): read the first 2 bytes from the same memory address → `E8 03` → 0x03E8 = 1000. Zero cost, no copying.
- **Big-endian** (`00 00 03 E8`): the useful bytes are at the end. You must copy them to a new location. That costs CPU time.

### Same value, different types — the pattern

Notice how the same value (1000) looks across types in little-endian:

| Type | Encoded |
|------|---------|
| u32 | `E8 03 00 00` |
| u16 | `E8 03` |

The `u16` encoding is literally the first 2 bytes of the `u32` encoding. The extra bytes are just padding at the end. This is the free casting property — you can cast down by simply reading fewer bytes from the same address.

In big-endian, the `u32` is `00 00 03 E8` and the `u16` is `03 E8` — completely different positions. No relationship between them.

### Q: Why did SCALE choose little-endian?

Because **Wasm is little-endian** by default, and Polkadot compiles its runtime to Wasm. The future PVM (PolkaVM, based on RISC-V) will also be little-endian. So little-endian is the natural choice for the ecosystem.

### Q: Are there benefits to big-endian?

Everything is about trade-offs. Ethereum uses big-endian with 256-bit words (which Joseph described as "waste of memory non-stop"). Big-endian can be more natural for network protocols and human readability. But for the Polkadot use case, little-endian's free casting won.

---

# Part III: SCALE Primitives

SCALE stands for **Simple Concatenated Aggregate Little-Endian**. It is:
- **Non-self-describing**: you need the type definitions to decode
- **Little-endian**: for the free casting property
- **Concatenated**: complex types are simply concatenated primitives
- Designed for **high-performance, copy-free encoding/decoding** in resource-constrained environments

### Q: Why not use an existing standard like Protobuf?

This came up during the Q&A. Several reasons:

1. **Protocol independence**: Polkadot is building a protocol. Depending on a third-party encoding format (like Protobuf, owned by Google) means they could change their spec or break backward compatibility at any time, potentially breaking the blockchain. You need to control the protocol yourself.

2. **`no_std` compatibility**: the runtime compiles to Wasm and runs in a `no_std` environment (no standard library). SCALE is designed to work without `std`. Protobuf's Rust implementation has dependencies that require `std`.

3. **Minimality**: SCALE is deliberately tiny and simple. Protobuf is a much larger project with features (like schema evolution, field numbering, wire types) that add complexity unnecessary for SCALE's use case.

4. **Deep integration with Rust derive macros**: SCALE integrates seamlessly with `#[derive(Encode, Decode)]` and auto-generates metadata like `MaxEncodedLen` and `TypeInfo`. This tight coupling with the Rust ecosystem and Frame's macro system would be very hard to replicate with an external format.

5. **Blockchain-specific needs**: features like compact encoding, the type registry for metadata, and the relationship between SCALE and the runtime's self-describing metadata are all designed specifically for the blockchain use case.

## 6. Unsigned Integers (U8 to U256)

Unsigned integers are encoded directly in little-endian with their full byte width:

| Value | Type | Encoded |
|-------|------|---------|
| 20 | u8 | `14` |
| 20 | u16 | `14 00` |
| 256 | u16 | `00 01` |
| 1000 | u32 | `E8 03 00 00` |

The value 256 as u16 can be confusing: `00 01` looks like "1" but in little-endian the `01` in the second byte represents 256 (255 is `FF`, adding 1 overflows to the next byte).

## 7. Signed Integers and Two's Complement

Negative values are represented using **two's complement**. The process:

1. Take the positive value in binary
2. Flip all bits (bitwise NOT)
3. Add 1

Example with one byte (i8), computing -5:
- 5 in binary: `0000 0101`
- Flip all bits: `1111 1010`
- Add 1: `1111 1011`

Shortcut: go right-to-left, keep everything (including the first `1` you find) unchanged, flip the rest.

Properties of two's complement:
- The most significant bit indicates sign: 1 = negative, 0 = positive
- No "minus zero" — all permutations represent unique values
- For an i8: range is -128 to +127
- CPU arithmetic operations work the same as with unsigned values

### Q: How do you know if it's a negative number or just a large positive?

Because the type system tells you. If the type is **signed** (i8, i16, etc.), the MSB indicates the sign. If the type is **unsigned** (u8, u16, etc.), all bits are value. You need to know the type to interpret correctly — which, in SCALE, means you need the metadata.

### Q: Is two's complement related to endianness?

No, they're orthogonal. Two's complement applies to **bits inside individual bytes**. Endianness applies to the **order of bytes**. Inside the byte, there is no endianness. For multi-byte signed values, two's complement determines the bit pattern, and endianness determines byte order. The value -1 is always `FF` regardless of endianness (all bits set). But -256 as i16 would be `00 FF` in little-endian and `FF 00` in big-endian.

Signed integers are rarely used in blockchain.

## 8. Booleans

A boolean uses one full byte: `0x00` = false, `0x01` = true. This "wastes" 7 bits, but the byte is the minimum addressable unit. For situations where you need many boolean flags, SCALE provides bit vectors (covered later).

### Q: Why not pack multiple booleans into one byte using the first bits as a counter?

This was asked during the lecture: "If I have 4 booleans in a row, I waste 28 bits. Why not use the first 3 bits as a count and store up to 5 booleans in a single byte?"

Joseph's answer: this would be "bit boxing" at the codec level, and it's very inefficient in the long run because you'd constantly need to track "which bit was I at last time?" across encoding/decoding. When you genuinely need many boolean flags, you use a **bit mask** — an array or vector of u8 — and interpret it at the application level. You don't need to make this a first-class citizen of the type system.

Victor reinforced this point in the metadata session: the conviction voting in Polkadot governance packs a vote direction and conviction level into a single byte using bit manipulation. The result is that dapp developers see an opaque integer instead of meaningful fields. This makes the metadata less descriptive and the developer experience worse. The lesson: what saves storage space on-chain can cost dearly in understandability off-chain. Trade-offs.

---

# Part IV: Compact Encoding — SCALE's Signature Primitive

## 9. The Problem Compact Solves

Many values in a blockchain runtime can potentially be very large but are usually small. Account counters, vector lengths, balances — they might reach millions but are often single digits. Do you reserve a `u256` (32 bytes) every time, wasting space 99% of the time?

Compact encoding says: use the minimum number of bytes needed for the actual value. If the value is 5, use 1 byte. If it's 1 billion, use 4 bytes. Dynamically.

## 10. How Compact Works — The Four Modes

The **2 least significant bits** of the first byte are flags that tell you how many bytes the value occupies. Since 2 bits give 4 permutations, there are 4 modes:

| Flag (2 LSB) | Mode | Bytes Used | Value Range |
|---------------|------|------------|-------------|
| `00` | 0 | 1 | 0 – 63 |
| `01` | 1 | 2 | 64 – 16,383 |
| `10` | 2 | 4 | 16,384 – 1,073,741,823 |
| `11` | 3 | variable | up to 2^536 |

### Q: What are those 2 flag bits for?

They're a mini-header inside the first byte. Since SCALE is non-self-describing and concatenated (data is packed back-to-back with no separators), when you arrive at a compact value, you need to know where it ends and the next datum begins. The 2 flag bits tell you exactly how many bytes to read. Without them, you'd have no way to know the boundary. It's the price you pay: you "sacrifice" 2 bits of value space for this metadata.

## 11. Mode 0 — Single Byte (flag 00)

The value fits in 6 bits (the remaining bits after the 2-bit flag). Range: 0 to 63.

The encoding is: `value << 2` (shift left 2 to make room for the flag, which is `00`).

| Value | Binary (6 bits + flag) | Hex |
|-------|----------------------|-----|
| 1 | `000001 00` | `0x04` |
| 2 | `000010 00` | `0x08` |
| 3 | `000011 00` | `0x0C` |
| 63 | `111111 00` | `0xFC` |

Notice the pattern: values go 0x04, 0x08, 0x0C — increments of 4, because the 2 flag bits occupy the lowest positions.

## 12. Mode 1 — Two Bytes (flag 01)

The value is spread across 6 bits of the first byte + 8 bits of the second byte = 14 bits total. Range: 64 to 16,383.

Structure:
```
Byte 1: [ 6 bits of value (low part) | 0 1 ]   ← flag is 01
Byte 2: [ 8 bits of value (high part) ]
```

### Step-by-step: encoding 64

64 in binary with 14 bits: `00000100000000`

Split: low 6 bits = `000000`, high 8 bits = `00000001`

Add flag `01` to first byte: `000000 01` → `0x01`

Second byte: `00000001` → `0x01`

Result: **`0x01 0x01`**

### Step-by-step: encoding 65

65 in 14 bits: `00000100000001`

Low 6 bits = `000001`, high 8 bits = `00000001`

First byte with flag: `000001 01` → `0x05`

Second byte: `0x01`

Result: **`0x05 0x01`**

### Step-by-step: encoding 100

100 in 14 bits: `00000001100100`

Low 6 bits = `100100`, high 8 bits = `00000001`

First byte with flag: `100100 01` → `10010001` → `0x91`

Second byte: `0x01`

Result: **`0x91 0x01`**

### Decoding mode 1 — reverse process

Receive `0x91 0x01`:

1. Read first byte: `0x91` = `10010001`
2. Check 2 LSB: `01` → mode 1, read 2 bytes total
3. Remove flag from first byte: `10010001` → `100100` (the 6 value bits)
4. Second byte is all value: `00000001`
5. Concatenate (second byte is high part): `00000001 100100` = `1100100` = **100**

## 13. Mode 2 — Four Bytes (flag 10)

Same mechanism as modes 0 and 1. The value occupies 6 + 8 + 8 + 8 = 30 bits across 4 bytes. Range: up to 2^30 - 1 (1,073,741,823).

Encoding: `(value << 2) | 0b10`, then encode the resulting u32 as little-endian.

## 14. Mode 3 — Big Integer (flag 11)

In modes 0, 1, and 2, the 6 remaining bits of the first byte hold **part of the value**. In mode 3, those 6 bits change role: they hold the **number of additional bytes to read, minus 4**.

### Q: In modes 0, 1, 2 the 6 bits are part of the value. What changes in mode 3?

In mode 3, the number is too large for 4 bytes, so we need a way to express how many bytes are needed. The 6 remaining bits tell us the byte count. We add 4 because if the value fit in 4 bytes, mode 2 would have been used. So the minimum in mode 3 is 5 bytes of value (6 bits = 0, plus 4 = 4... but that case is for u32 overflow).

Maximum: 6 bits can express up to 63. So 63 + 4 = **67 bytes** maximum. 67 × 8 = 536 bits → maximum value is **2^536**.

### Concrete example: encoding a value that needs 7 bytes

Step 1 — the header byte:

We need 7 bytes for the value. Store `7 - 4 = 3` in the 6 bits:

```
6 bits          flag
  ↓               ↓
000011            11
  ↓               ↓
"I need 3+4     "I'm mode 3"
 = 7 bytes"
```

Together: `00001111` → `0x0F`

Step 2 — the 7 value bytes follow directly after.

Step 3 — decoding:

1. Read first byte `0x0F` = `00001111`
2. LSB 2 bits: `11` → mode 3
3. Upper 6 bits: `000011` = 3
4. 3 + 4 = **7 bytes** to read next
5. Those 7 bytes are the value in little-endian
6. The byte after those 7 belongs to the next datum

## 15. The Validity Trap — Not All Permutations Are Valid

Joseph demonstrated this with a trap exercise: the value `0x05` as a standalone compact value is **invalid**. Why?

`0x05` = `00000101`. The 2 LSB are `01` → mode 1 → "read the next byte too." But there is no next byte. The decoder panics.

What about `0x05 0x00` — adding a zero byte? Still invalid. The value it would represent (1) fits in mode 0 (single byte). **SCALE requires the most compact representation possible.** If a value can be expressed in fewer bytes, using more bytes is an error.

This applies to all values under 64 that would have flag `01` — they're invalid in mode 1 because they should be in mode 0. The `parity-scale-codec` implementation enforces this with explicit range checks in the decode path (returning "out of range" errors).

## 16. When to Use Compact (and When Not To)

Compact is ideal for values that **can be large but are usually small**: counters (number of accounts), vector lengths, balances, nonces.

### Q: Why isn't compact used by default for everything?

Three costs:

1. **Slower to encode/decode**: a normal u32 is always 4 bytes — just read them. Compact requires inspecting the flag, deciding the mode, then reading the right number of bytes. More CPU work.

2. **Can be larger for big values**: a u32 is always 4 bytes. But a compact u32 in mode 3 needs 5 bytes (1 header + 4 value). For large values, compact is worse.

3. **Unpredictable size**: fixed types have known sizes, enabling offset calculations and memory optimizations. Compact sizes vary, requiring sequential decoding.

That's why it's opt-in with `#[codec(compact)]` — you decide per-field where the trade-off is worth it.

## 17. Compact in Source Code — parity-scale-codec

File: `src/compact.rs` in the `parity-scale-codec` crate.

### Encoding (CompactRef for u32)

```rust
fn encode_to<W: Output + ?Sized>(&self, dest: &mut W) {
    match self.0 {
        0..=0b0011_1111 => dest.push_byte((*self.0 as u8) << 2),          // Mode 0
        0..=0b0011_1111_1111_1111 =>
            (((*self.0 as u16) << 2) | 0b01).encode_to(dest),             // Mode 1
        0..=0b0011_1111_1111_1111_1111_1111_1111_1111 =>
            ((*self.0 << 2) | 0b10).encode_to(dest),                      // Mode 2
        _ => {
            dest.push_byte(0b11);                                          // Mode 3 header
            self.0.encode_to(dest);                                        // Raw u32 value
        },
    }
}
```

In all modes 0-2: `<< 2` makes room for the flag, then `| flag` sets it. The `.encode_to()` call on u16/u32 produces little-endian bytes automatically.

The match ranges are exact binary boundaries: `0b0011_1111` = 63, `0b0011_1111_1111_1111` = 16383, etc.

### Mode 3 for large numbers (u64)

```rust
_ => {
    let bytes_needed = 8 - self.0.leading_zeros() / 8;
    dest.push_byte(0b11 + ((bytes_needed - 4) << 2) as u8);   // header
    let mut v = *self.0;
    for _ in 0..bytes_needed {
        dest.push_byte(v as u8);   // write low byte
        v >>= 8;                    // shift to next byte
    }
}
```

- `leading_zeros()` counts zero bits from the left; dividing by 8 gives empty bytes
- Header construction: `0b11` (mode 3 flag) + `(bytes_needed - 4) << 2` (byte count in upper 6 bits)
- The loop writes the value byte-by-byte in little-endian

### Decoding (Compact for u64)

```rust
let prefix = input.read_byte()?;
Ok(Compact(match prefix % 4 {           // Extract 2 LSB (same as & 0b11)
    0 => u64::from(prefix) >> 2,         // Mode 0: shift out flag
    1 => { /* read 2 bytes, >> 2, validate range */ },
    2 => { /* read 4 bytes, >> 2, validate range */ },
    3 => match (prefix >> 2) + 4 {       // Mode 3: extract byte count
        bytes_needed => {
            let mut res = 0;
            for i in 0..bytes_needed {
                res |= u64::from(input.read_byte()?) << (i * 8);  // LE reconstruction
            }
            // ... range validation
        },
    },
}))
```

`prefix % 4` extracts the 2 LSB — the mode flag. For mode 3, `(prefix >> 2) + 4` extracts the 6 upper bits and adds the offset.

### Range validation — the trap in code

```rust
1 => {
    let x = u16::decode(...)? >> 2;
    if x > 0b0011_1111 && x <= 255 {  // MUST be > 63
        x as u8
    } else {
        return Err(U8_OUT_OF_RANGE.into());  // Value fits in mode 0 → invalid
    }
}
```

Every mode validates that the decoded value **couldn't have been encoded in a smaller mode**. This is the implementation of the validity trap.

### Test cases confirm everything

```rust
(0u64,  "00"),                          // 0  → mode 0 → 1 byte
(63,    "fc"),                          // 63 → mode 0 → max of mode 0
(64,    "01 01"),                       // 64 → mode 1 → 2 bytes
(16383, "fd ff"),                       // 16383 → mode 1 → max of mode 1
(16384, "02 00 01 00"),                 // 16384 → mode 2 → 4 bytes
(1073741823, "fe ff ff ff"),            // max of mode 2
(1073741824, "03 00 00 00 40"),         // mode 3 → 5 bytes
((1 << 32) - 1, "03 ff ff ff ff"),      // u32::MAX in mode 3
(1 << 32, "07 00 00 00 00 01"),         // needs 6 bytes
```

---

# Part V: Complex Types

Complex types are codecs that reference other codecs. They're pure concatenation — packed back-to-back without separators or delimiters.

## 18. Enum (Tagged Union)

The first byte is the **variant index** (tag). What follows depends on which variant it is.

```rust
enum MyEnum {
    Foo(u16),    // variant 0
    Bar(bool),   // variant 1
    Baz,         // variant 2 (no data)
}
```

| Value | Encoded | Explanation |
|-------|---------|-------------|
| `Foo(1)` | `00 01 00` | tag 0 + u16 LE |
| `Bar(false)` | `01 00` | tag 1 + false |
| `Baz` | `02` | tag 2, nothing follows |

The decoder reads the first byte → knows the variant → knows what type to decode next (or nothing for unit variants).

### Circular definitions

Enums enable circular codec definitions. You can have:

```rust
enum Tree {
    Leaf(u32),
    Node(Vec<Tree>),  // references itself
}
```

This works because `Leaf` breaks the circularity — at some point the recursion terminates.

### Limit: 256 variants maximum

Since the tag is a single byte, an enum can have at most 256 variants (0-255). A workaround mentioned by Joseph: use variant 255 as a flag for a nested enum, though this is non-standard.

## 19. Option and Result — Specialized Enums

### Option

- Variant 0 = `None` (no data follows)
- Variant 1 = `Some(value)`

```
Option<u16> = None      → 0x00
Option<u16> = Some(256) → 0x01 0x00 0x01
                           ↑    ↑────────↑
                          Some  256 in LE
```

Note: `01 00 01` can confuse — the first `0x01` is the "Some" tag, and `00 01` is 256 in little-endian.

### Result

- Variant 0 = `Ok(value)`
- Variant 1 = `Err(value)`

```
Result<bool, u8>:  Ok(true)  → 0x00 0x01
                   Err(42)   → 0x01 0x2A
```

## 20. Struct and Tuple

For SCALE, structs and tuples are **identical at the binary level** — pure concatenation of fields in order. The only difference is semantic: structs have labeled fields, tuples don't. They produce the same bytes.

### Decoding a struct with multi-byte fields — worked example

```rust
struct Transaction {
    nonce: u32,      // 4 bytes
    amount: u64,     // 8 bytes
    fee: u16,        // 2 bytes
    success: bool,   // 1 byte
    tip: Option<u32>,// 1 byte (None) or 5 bytes (Some)
}
```

Encoding `Transaction { nonce: 1000, amount: 100_000, fee: 512, success: true, tip: Some(300) }`:

```
E8 03 00 00  A0 86 01 00 00 00 00 00  00 02  01  01 2C 01 00 00
├──────────┤ ├────────────────────────┤ ├───┤ ├─┤ ├────────────┤
 nonce=1000   amount=100000            fee   ok  tip=Some(300)
              (u64 LE)                 =512  T
```

With `tip: None`, the last part would be just `00` (1 byte) instead of `01 2C 01 00 00` (5 bytes).

### Struct with vector and compact fields — worked example

```rust
struct Block {
    version: u16,
    num_transactions: Compact<u32>,
    transactions: Vec<u64>,
    validator_index: Compact<u32>,
    finalized: bool,
}
```

Encoding `Block { version: 3, num_transactions: Compact(5), transactions: vec![1000, 50000, 200], validator_index: Compact(16384), finalized: true }`:

```
03 00 14 0C E8 03 00 00 00 00 00 00 50 C3 00 00 00 00 00 00 C8 00 00 00 00 00 00 00 02 00 01 00 01
├───┤ ├┤ ├┤ ├──────────────────────┤ ├──────────────────────┤ ├──────────────────────┤ ├─────────┤ ├┤
ver   5  3   tx[0]=1000             tx[1]=50000              tx[2]=200              16384       T
 3       ↑                                                                           compact
      vec len (compact 3 = 0x0C)                                                     mode 2
```

33 bytes total. Without compact for the two fields, it would be 39 bytes.

## 21. Vector (Dynamic Length)

A vector starts with the **length as a compact integer**, followed by the concatenated elements.

| Value | Encoded | Explanation |
|-------|---------|-------------|
| `vec![]` (empty) | `00` | compact 0 |
| `vec![1u8]` | `04 01` | compact 1 + value |
| `vec![1u8, 0]` | `08 01 00` | compact 2 + values |

Remember: 1 in compact = `0x04`, 2 = `0x08`, 3 = `0x0C`.

### Vector of compacts

Vectors can contain any type, including other compacts:

`vec![Compact(1), Compact(64)]` → `08 04 01 01`
- `08` = compact 2 (length)
- `04` = compact 1 (first element)
- `01 01` = compact 64 (second element, mode 1)

You can have vectors of vectors, vectors of structs — arbitrarily nested. Each inner type is decoded by its own codec.

## 22. Array (Fixed Length)

Same as vector but **without the compact length prefix**. The size is known at compile time, so there's nothing to communicate.

`[u8; 3]` with values `[1, 2, 3]` → `01 02 03`

It's essentially a tuple of N identical types.

## 23. Opaque

Sometimes you want to carry binary data without decoding it. Opaque is: a **compact byte count** followed by a **raw blob**.

The key real-world example: the body of a Polkadot block is a `Vec<OpaqueExtrinsic>`. Each extrinsic is a complex SCALE-encoded structure, but the node doesn't need to decode it — the runtime handles that. So the node imports it as opaque: reads the byte count, stores the blob as-is.

```
[compact N] [N bytes of raw data]
     ↑              ↑
"N bytes"    "not my job to decode this"
```

## 24. Types That Are NOT First-Class in SCALE

SCALE is deliberately minimal. Several common data types are NOT first-class primitives — they're represented using simpler types and interpreted at the application level.

### Strings

There is no string type in SCALE. A string is simply a `Vec<u8>` — compact length prefix followed by raw bytes. Your application logic knows that this particular vector of bytes is meant to be interpreted as a UTF-8 string (or whatever encoding). From SCALE's perspective, it's just bytes.

This was a point of discussion in the lecture: someone asked if this makes development harder compared to Solidity where strings are first-class. Joseph's answer: frameworks like Frame, Papi, and other tools handle the string abstraction for you. Developers work with strings normally; the framework translates to `Vec<u8>` when it hits SCALE. The fact that strings aren't a SCALE primitive doesn't mean there can't be a higher-level layer that provides a good developer experience.

### Maps and Sets

A `Map<K, V>` in SCALE is represented as a `Vec<(K, V)>` — a vector of key-value tuples. A `Set<T>` is just a `Vec<T>`. Ordering, uniqueness, and lookup semantics are the application's responsibility. SCALE only cares about serializing the data.

### Q: What is the maximum size of a vector?

The length of a vector is compact-encoded. Since compact can represent values up to 2^536, that's the theoretical maximum number of elements. In practice, you'd run out of memory long before hitting this limit. Note: the length is always compact, never a fixed u8 — a common confusion that would limit vectors to 256 elements.

## 25. Bit Vectors (bitvec)

Joseph mentioned "bit boxing" — using individual bits inside a byte. Booleans waste 7 bits per value. When you need many flags (hundreds or thousands), bitvec packs them efficiently: 1 bit per flag.

### How bitvec is SCALE-encoded

```
[compact length in BITS] [packed bytes]
```

The compact indicates the number of **bits**, not bytes.

### Encoding example: 10 flags

Flags: `true, false, true, true, false, false, true, false, true, true`

Step 1 — compact of bit count: compact(10) = `0x28`

Step 2 — pack bits into bytes using **Lsb0** order (Polkadot's bit ordering). In Lsb0, the first bit goes into the least significant position of the byte:

```
Byte 1 (bits 0-7):
  Bit index:  0  1  2  3  4  5  6  7
  Value:      1  0  1  1  0  0  1  0
  Position:  b0 b1 b2 b3 b4 b5 b6 b7
  Byte = 0b01001101 = 0x4D

Byte 2 (bits 8-9, rest padding):
  Bit index:  8  9
  Value:      1  1   (+ 6 zero padding bits)
  Byte = 0b00000011 = 0x03
```

Result: **`28 4D 03`** — 3 bytes instead of 10 bytes with booleans.

### In the Type Registry

```json
{
    "type_def": {
        "BitSequence": {
            "bit_store_type": "type_id for u8",
            "bit_order_type": "type_id for Lsb0"
        }
    }
}
```

The `bit_order_type` is crucial: `Lsb0` means first bit goes to least significant position; `Msb0` would be the opposite. Polkadot uses `Lsb0`.

### Real-world usage: availability bitfields

In Polkadot, each validator publishes a bitfield indicating which parachain cores they have data available for. With 50 active parachains, each bitfield is ~7 bytes instead of 50 bytes. Across 300 validators per block, that's ~2.1 KB vs ~15 KB — significant savings at block production rate (every 6 seconds).

---

# Part VI: The derive Macros — Automatic Codec Generation

## 26. #[derive(Encode, Decode)]

In practice, you almost never write codecs by hand. The `Encode` and `Decode` derive macros from `parity-scale-codec` generate everything automatically at compile time.

### For structs

```rust
#[derive(Encode, Decode)]
pub struct AccountInfo {
    pub nonce: u32,
    pub consumers: u32,
    pub providers: u32,
    pub sufficients: u32,
    pub data: AccountData,
}
```

The macro generates code equivalent to:

```rust
impl Encode for AccountInfo {
    fn encode_to<W: Output + ?Sized>(&self, dest: &mut W) {
        self.nonce.encode_to(dest);
        self.consumers.encode_to(dest);
        self.providers.encode_to(dest);
        self.sufficients.encode_to(dest);
        self.data.encode_to(dest);       // recursive — AccountData also has Encode
    }
}

impl Decode for AccountInfo {
    fn decode<I: Input>(input: &mut I) -> Result<Self, Error> {
        Ok(AccountInfo {
            nonce: u32::decode(input)?,
            consumers: u32::decode(input)?,
            providers: u32::decode(input)?,
            sufficients: u32::decode(input)?,
            data: AccountData::decode(input)?,
        })
    }
}
```

Pure concatenation in field order — exactly what SCALE specifies.

### For enums

```rust
#[derive(Encode, Decode)]
pub enum MultiAddress {
    Id(AccountId32),
    Index(Compact<u32>),
    Raw(Vec<u8>),
    Address32([u8; 32]),
    Address20([u8; 20]),
}
```

Generates: first byte = variant index, then encode the variant's contents. Each variant index defaults to its position (0, 1, 2...) unless overridden.

## 27. Codec Attributes

**Custom variant index:**
```rust
#[codec(index = 15)]
A,  // uses 15 as tag instead of 0
```

**Compact field:**
```rust
#[codec(compact)]
pub value: u128,  // encoded as Compact<u128> automatically
```

Without the attribute, `u128` always takes 16 bytes. With `#[codec(compact)]`, small values use fewer bytes.

**Skip a field:**
```rust
#[codec(skip)]
pub cached: Option<u64>,  // not serialized; uses Default on decode
```

## 28. TypeInfo — Bridging to Metadata

Typically you see all three derives together:

```rust
#[derive(Encode, Decode, TypeInfo)]
pub struct AccountInfo { ... }
```

- `Encode` → generates the SCALE encoder
- `Decode` → generates the SCALE decoder
- `TypeInfo` → generates type information for the **metadata type registry**

The three work together: `Encode/Decode` for actual serialization; `TypeInfo` so that external clients (Papi, Subxt, PolkadotJS) can discover the type's structure at runtime through metadata.

---

# Part VII: Runtime Metadata

## 29. What Metadata Solves

SCALE is non-self-describing. Given a blob of bytes, you cannot decode it without knowing the types. So how does a client like Papi or Subxt know what types to use?

The runtime exposes its entire type structure through **metadata** — a SCALE-encoded blob that any client can request via the `state_getMetadata` RPC call. It's the runtime describing itself. When the runtime upgrades and types change, clients refresh the metadata and adapt automatically.

## 30. The Metadata Blob Structure

The raw bytes from `state_getMetadata` start with:

```
6d 65 74 61  0f  ...rest...
├──────────┤ ├┤
 "meta"     version 15
 magic #
```

First 4 bytes: `0x6d657461` = "meta" in ASCII — a magic number to verify you're reading metadata. Then the version byte (currently V14 = `0x0e` or V15 = `0x0f`).

The top-level structure (V14):

```rust
pub struct RuntimeMetadataV14 {
    pub types: PortableRegistry,                    // the type registry
    pub pallets: Vec<PalletMetadata<PortableForm>>, // all pallets
    pub extrinsic: ExtrinsicMetadata<PortableForm>, // extrinsic format info
    pub ty: <PortableForm as Form>::Type,            // the Runtime type
}
```

### Q: How do you decode the metadata if the metadata is what tells you how to decode things?

This is the bootstrapping problem — a fish eating its own tail. The metadata is SCALE-encoded, but SCALE is not self-describing. You need type definitions to decode SCALE, and the metadata IS those type definitions.

The solution: the metadata structure is a **well-known type** that library authors hardcode. The Rust definition lives in the `paritytech/frame-metadata` repository (e.g., `frame-metadata/src/v14.rs`, `v15.rs`, `v16.rs`). Every client library (Papi, Subxt, PolkadotJS) has its own hardcoded implementation of the metadata codec.

For Papi specifically, they have their own hardcoded codec for metadata decoding. For Subxt, it's generated from the Rust types. For PolkadotJS, it was manually implemented in TypeScript.

This means: to interact with ANY Substrate chain, the only thing you need to hardcode is how to decode the metadata. Once you have the metadata decoded, everything else (calls, storage, events, types) can be decoded dynamically from the type registry.

### Metadata Versions and Runtime APIs

The metadata has evolved across versions, and the runtime exposes multiple APIs for accessing it:

**`metadata_metadata()`**: the original runtime API. Always returns **V14** for backward compatibility — even if the chain supports newer versions. This was a deliberate decision to avoid breaking existing tooling when V15 was introduced.

**`metadata_metadata_versions()`**: returns which versions the chain supports. For example, a current Polkadot node might return `[14, 15, 4294967295]`. That last number (u32::MAX) is the unstable flag for V16 — the highest bit is set to indicate "this version is still in development."

**`metadata_metadata_at_version(n)`**: returns the metadata in a specific version format. So you can request V14, V15, or V16 (using the unstable flag number).

The version history and what each added:

- **V13 and earlier**: metadata existed but was "mostly useless" — types were represented as strings (like `"AccountId32"`) with no structural definition. You had to know externally what `AccountId32` meant. Complex types like XCM messages were impossible to decode from metadata alone.

- **V14**: the breakthrough. Added the **complete type registry** with full structural definitions for every type. Made the metadata truly self-describing (you could even build a metadata that describes its own encoding, though this isn't done because the blockchain treats metadata as an opaque blob). Added the `metadata_metadata()` runtime API.

- **V15**: added **runtime API definitions** to the metadata (previously, runtime API signatures were also well-known/hardcoded). Added **outer enums** (a single enum containing all calls/events/errors across all pallets). Improved the extrinsic definition with full signed extension information.

- **V16** (unstable at time of writing): added **transaction extensions** (replacing signed extensions), **view functions** (per-pallet read-only functions for dapps), and **deprecation markers** for methods.

## 31. The Type Registry — Single Source of Truth

### The core structures

The registry is an array of types:

```rust
pub struct PortableRegistry {
    pub types: Vec<PortableType>,
}

pub struct PortableType {
    pub id: u32,
    pub ty: Type<PortableForm>,
}

pub struct Type<T: Form = MetaForm> {
    path: Path<T>,              // e.g. "frame_system::AccountInfo"
    type_params: Vec<T::Type>,  // generic parameters
    type_def: TypeDef<T>,       // the actual definition
}
```

Every type gets an ID (its array index). Anything in the metadata that needs to reference a type uses that ID.

### TypeDef — maps directly to SCALE types

```rust
pub enum TypeDef<T: Form = MetaForm> {
    Composite(TypeDefComposite<T>),      // struct/tuple → concatenation
    Variant(TypeDefVariant<T>),          // enum → tag + variant content
    Sequence(TypeDefSequence<T>),        // Vec<T> → compact len + elements
    Array(TypeDefArray<T>),              // [T; N] → N elements, no length prefix
    Tuple(TypeDefTuple<T>),             // (A, B, C) → concatenation
    Primitive(TypeDefPrimitive),         // u8, bool, etc.
    Compact(TypeDefCompact<T>),          // Compact<T>
    BitSequence(TypeDefBitSequence<T>),  // bitvec
}
```

Each variant maps directly to a SCALE encoding strategy we've already learned.

### Concrete example: AccountInfo in the registry

```
registry[3] → Composite (AccountInfo) {
    fields: [
        { name: "nonce",       type_id: 4 },
        { name: "consumers",   type_id: 4 },
        { name: "providers",   type_id: 4 },
        { name: "sufficients", type_id: 4 },
        { name: "data",        type_id: 5 }
    ]
}

registry[4] → Primitive U32

registry[5] → Composite (AccountData) {
    fields: [
        { name: "free",     type_id: 6 },
        { name: "reserved", type_id: 6 },
        { name: "frozen",   type_id: 6 },
        { name: "flags",    type_id: 7 }
    ]
}

registry[6] → Primitive U128
```

### The resolution process — recursive tree walk

To decode bytes as `AccountInfo`, a client follows this algorithm:

```
resolve(type_id: 3)
  → Composite, 5 fields
    → "nonce": resolve(4) → Primitive U32 → read 4 bytes
    → "consumers": resolve(4) → Primitive U32 → read 4 bytes
    → "providers": resolve(4) → Primitive U32 → read 4 bytes
    → "sufficients": resolve(4) → Primitive U32 → read 4 bytes
    → "data": resolve(5)
      → Composite, 4 fields
        → "free": resolve(6) → Primitive U128 → read 16 bytes
        → "reserved": resolve(6) → read 16 bytes
        → "frozen": resolve(6) → read 16 bytes
        → "flags": resolve(7) → ...
```

Start at a root type_id, resolve its definition, and for each field/variant/element that references another type_id, recurse until you hit primitives.

### Variant (enum) in the registry

`Option<u32>`:

```
registry[10] → Variant {
    variants: [
        { name: "None", index: 0, fields: [] },
        { name: "Some", index: 1, fields: [{ type_id: 4 }] }
    ]
}
```

The decoder reads the tag byte, finds the matching variant by `index`, and resolves its fields recursively.

### Sequence (Vec) in the registry

`Vec<u8>`:

```
registry[12] → Sequence { type_param: 1 }
```

The decoder reads a compact (length), then resolves `type_id: 1` (u8) that many times.

### Q: Where does type_id_99 come from? Who assigns it?

The compiler assigns it automatically. When the runtime compiles, the `#[derive(TypeInfo)]` macro registers each type it encounters and assigns sequential IDs. The type `(AccountId32, AccountId32)` might end up as ID 99 in one runtime version and ID 105 in another.

**IDs are not stable across versions.** They're just array indices internal to that specific metadata blob. That's why clients must refresh metadata whenever the runtime upgrades (i.e., when `spec_version` changes).

## 32. How Pallets Reference the Registry

### PalletMetadata structure

```rust
pub struct PalletMetadata<T: Form = MetaForm> {
    pub name: T::String,                            // "Balances", "System"
    pub storage: Option<PalletStorageMetadata<T>>,
    pub calls: Option<PalletCallMetadata<T>>,
    pub event: Option<PalletEventMetadata<T>>,
    pub constants: Vec<PalletConstantMetadata<T>>,
    pub error: Option<PalletErrorMetadata<T>>,
    pub index: u8,                                  // the pallet index
}
```

The `index` is the first byte you see in an extrinsic identifying which pallet a call belongs to. Each section references the type registry via type IDs.

### Calls — dispatchable functions

```rust
pub struct PalletCallMetadata<T: Form = MetaForm> {
    pub ty: T::Type,  // a single type_id pointing to a Variant in the registry
}
```

That type_id resolves to a `Variant` (enum) where each variant is a dispatchable function:

```
registry[call_type_id] → Variant {
    variants: [
        {
            name: "transfer_allow_death",
            index: 0,
            fields: [
                { name: "dest", type_id: 42 },   // MultiAddress
                { name: "value", type_id: 6 }     // Compact<u128>
            ]
        },
        {
            name: "force_transfer",
            index: 2,
            fields: [
                { name: "source", type_id: 42 },
                { name: "dest", type_id: 42 },
                { name: "value", type_id: 6 }
            ]
        },
    ]
}
```

Note: variant indices are **not necessarily sequential** (0, 2, ...). If a call is removed, the others keep their indices for backward compatibility.

### Events and Errors — same pattern

Both are a single `type_id` pointing to a `Variant` in the registry. For events, each variant is an event type with its data fields. For errors, each variant is an error type (usually with no fields).

### Constants — pre-encoded values

```rust
pub struct PalletConstantMetadata<T: Form = MetaForm> {
    pub name: T::String,         // "ExistentialDeposit"
    pub ty: T::Type,             // type_id → u128
    pub value: Vec<u8>,          // the value, already SCALE-encoded
    pub docs: Vec<T::String>,
}
```

The value comes pre-encoded. You just need the `ty` to know how to decode it.

**The pattern**: calls, events, and errors are all `Variant` enums in the registry. Storage values are types resolved recursively. Constants come pre-encoded with a type_id for decoding. The registry is the single source of truth — everything else is pointers into it.

## 33. Storage — The Richest Part

All blockchain state lives in a key-value store (a Merkle trie). Keys are hashes, values are SCALE-encoded bytes. The metadata tells you how to construct the keys and decode the values.

### Plain storage — a single global value

`TotalIssuance` in pallet Balances: a single u128, no key parameter.

```rust
StorageEntryMetadata {
    name: "TotalIssuance",
    ty: Plain(type_id_6),    // → u128
}
```

Key construction: `Twox128("Balances") ++ Twox128("TotalIssuance")` — 32 bytes, always the same.

### Map storage — one entry per key

`System::Account`: one entry per account. The metadata says:

```rust
StorageEntryMetadata {
    name: "Account",
    modifier: Default,
    ty: Map {
        hashers: [Blake2_128Concat],
        key: type_id_33,     // → AccountId32 = [u8; 32]
        value: type_id_3,    // → AccountInfo
    },
    default: [0, 0, 0, ...],
}
```

### Q: How does the metadata connect to the key-value store?

The metadata is used at **two moments**:

**Moment 1 — constructing the storage key** to query the node. You need to hash things correctly.

**Moment 2 — decoding the value** that the node returns. You need to know the type.

### Building a storage key — step by step

For `System::Account` with a specific account:

```
Twox128("System")                    → 16 bytes (hash of pallet name)
++
Twox128("Account")                   → 16 bytes (hash of entry name)
++
Blake2_128Concat(account_id_bytes)   → 16 bytes hash + 32 bytes raw key
                                     ─────────────
Total:                                 80 bytes
```

How do you know it's `Blake2_128Concat`? From `hashers` in the metadata.
How do you know the key is 32 bytes? From the `key` type_id → resolved in the registry → `[u8; 32]`.

Without metadata, you wouldn't know which hasher to use or how large the key is. Using the wrong hasher produces wrong bytes, and the node returns nothing.

### Hashers and why they matter

```rust
pub enum StorageHasher {
    Blake2_128,          // 16-byte hash, no key recovery
    Blake2_256,          // 32-byte hash, no key recovery
    Blake2_128Concat,    // 16-byte hash + raw key (iterable)
    Twox128,             // 16-byte hash, no key recovery
    Twox256,             // 32-byte hash, no key recovery
    Twox64Concat,        // 8-byte hash + raw key (iterable, faster)
    Identity,            // no hashing, raw key directly
}
```

**Concat hashers** (`Blake2_128Concat`, `Twox64Concat`, `Identity`) append the raw key after the hash. This enables **iteration** over all map entries — you can recover the original key from the storage key. Used when you need to enumerate all entries (e.g., list all accounts).

**Non-concat hashers** only store the hash. You can't iterate. Used for privacy or when enumeration isn't needed.

### Double maps

Some storage items have two keys (e.g., "how much did account A approve for account B to spend?"):

```rust
Map {
    hashers: [Blake2_128Concat, Blake2_128Concat],  // TWO hashers
    key: type_id_99,  // in registry → Tuple(AccountId32, AccountId32)
    value: type_id_6, // u128
}
```

Key construction:
```
Twox128(pallet) ++ Twox128(entry) ++ Blake2_128Concat(account_A) ++ Blake2_128Concat(account_B)
```

The number of hashers tells you how many keys there are, and the `key` type_id resolves to a tuple with the same number of elements.

### Complete flow: reading a balance

```
Your app: "I want the balance of 0xd435...7d"
    │
    ▼
Metadata: "System::Account is a Map, hasher Blake2_128Concat,
           key is type_id_33 (AccountId32), value is type_id_3 (AccountInfo)"
    │
    ▼
Construct storage key:
    Twox128("System") ++ Twox128("Account") ++ Blake2_128Concat(0xd435...7d)
    │
    ▼
RPC: state_getStorage(key) → returns raw bytes
    │
    ▼
Metadata + Registry: "type_id_3 is AccountInfo, resolve recursively"
    │
    ▼
Result: AccountInfo { nonce: 5, providers: 1, data: { free: 100 DOT, ... } }
```

---

# Part VIII: Extrinsic Structure

## 34. What Is an Extrinsic

Any data entering the runtime from outside. Two types:
- **Signed**: a user transaction with signature, sender, and fees
- **Unsigned**: no sender or signature (inherents, offchain workers)

## 35. High-Level Layout

```
[compact length] [header byte] [address + signature + extensions] [call data]
```

### 1. Compact Length

A compact integer: how many bytes follow. This lets the node know where one extrinsic ends and the next begins (blocks contain a vector of extrinsics).

### 2. Header Byte

```
bit 7           bits 6-0
  ↓                ↓
signed flag     format version (currently 4)
```

- Signed extrinsic: `0b10000100` = `0x84`
- Unsigned extrinsic: `0b00000100` = `0x04`

### 3. Address + Signature + Extensions (signed only)

**Address**: typically `MultiAddress`, an enum. Variant 0 (`Id`) is most common: `0x00` + 32 bytes of AccountId.

**Signature**: typically `MultiSignature`, an enum. Variant 1 (`Sr25519`) is most common in Polkadot: `0x01` + 64 bytes of signature.

**Extensions (extras)**: data defined by the runtime's signed extensions, in the exact order specified by the metadata. Typically includes Era, nonce (compact), and tip (compact).

### 4. Call Data

```
[pallet_index] [call_index] [arguments...]
```

- `pallet_index` (1 byte): which pallet, found via `PalletMetadata.index`
- `call_index` (1 byte): which function, found in the pallet's calls `Variant`
- `arguments`: concatenated, decoded via the type registry

## 36. Signed Extensions — Extra vs Additional Signed

Each signed extension has two components:

- **Extra data** (`ty` in metadata): travels in the extrinsic AND is signed
- **Additional signed** (`additional_signed` in metadata): is signed but does NOT travel in the extrinsic

### Q: What is additional_signed? Why doesn't it travel in the extrinsic?

It's an optimization. Certain data must be part of the signed payload (to prevent replay attacks, ensure the right chain/version, etc.) but the node already knows these values from its own state. Sending them would waste bandwidth.

Example: the **genesis hash** identifies which chain the transaction is for. The sender includes it when signing, but doesn't transmit it. The verifying node uses its own genesis hash to reconstruct the signing payload and verify the signature. If the sender signed with Polkadot's genesis hash but the node is on Kusama, the signature won't match → rejected.

### Q: How does the node know which additional_signed values the sender used?

It depends on the value:

**Deterministic values** (only one possibility): genesis hash, spec_version, transaction_version — the node has exactly one of each. No ambiguity.

**Derivable values**: the block hash for `CheckMortality` is derived from the **Era** that DOES travel in the extrinsic. The Era encodes a period and phase; given the current block number, the node calculates which reference block the sender used, looks up its hash, and uses it. If the Era is `Immortal` (`0x00`), the reference hash is the genesis hash.

The design ensures all additional_signed values are either unique or derivable — never ambiguous.

### Q: Do I sign a hash of the transaction or the whole thing?

You sign the full payload: `call_data + extras + additional_signed`. NOT a hash of it.

Exception: if the payload exceeds 256 bytes, it's hashed first (Blake2-256) and you sign the hash. This is an optimization for large transactions.

### Q: Where in the metadata are the extensions defined?

In `ExtrinsicMetadata`, which is part of the top-level runtime metadata:

```rust
pub struct ExtrinsicMetadata {
    pub ty: T::Type,
    pub version: u8,
    pub signed_extensions: Vec<SignedExtensionMetadata>,
}

pub struct SignedExtensionMetadata {
    pub identifier: String,         // "CheckMortality", "CheckNonce", etc.
    pub ty: type_id,                // extra data (travels in extrinsic)
    pub additional_signed: type_id, // signed but not transmitted
}
```

The vector is **ordered** — extensions must be processed in exactly this order. A client reads this vector and knows: which extensions exist, what type each `ty` and `additional_signed` is, and in what order to concatenate them.

### Typical signed extensions in Polkadot

```
signed_extensions: [
  { identifier: "CheckNonZeroSender",        ty: (),    additional_signed: ()           },
  { identifier: "CheckSpecVersion",          ty: (),    additional_signed: u32          },
  { identifier: "CheckTxVersion",            ty: (),    additional_signed: u32          },
  { identifier: "CheckGenesis",              ty: (),    additional_signed: [u8; 32]     },
  { identifier: "CheckMortality",            ty: Era,   additional_signed: [u8; 32]     },
  { identifier: "CheckNonce",               ty: Compact<u32>,  additional_signed: ()    },
  { identifier: "ChargeTransactionPayment", ty: Compact<u128>, additional_signed: ()    },
]
```

If the runtime adds a custom extension tomorrow, it appears in this vector and clients adapt automatically.

### CheckMetadataHash — Trustless Metadata Verification via Merkle Tree

There is an additional opt-in transaction extension not listed above that deserves special attention: **CheckMetadataHash**. This is the mechanism that prevents blind signing and ensures the metadata you used to construct a transaction is the same metadata the chain has.

**The problem**: before this extension existed, when a wallet or signer showed you "you're transferring 100 DOT to Alice," you had to trust that the metadata the wallet used was correct. A malicious RPC node or compromised dapp could provide fake metadata — telling you a field is "amount" when it's actually "tip," or showing wrong decimals so you send 100x more than intended. There was no on-chain verification.

**The solution**: a Merkle tree is constructed from the entire metadata. The root hash of this tree becomes the `additional_signed` value for the `CheckMetadataHash` extension. When you sign a transaction with this extension enabled, you're cryptographically committing to: "I only want this transaction to be valid if the chain's metadata matches exactly what I used to build it."

**How it works**:

1. The client builds a Merkle tree from the metadata
2. The root hash is included in the signing payload as `additional_signed`
3. The node independently computes the same Merkle tree root from its own metadata
4. If the hashes don't match, the signature verification fails → transaction rejected

This is the same pattern as CheckGenesis (sign the genesis hash without transmitting it) — the node already has the metadata, so it can reconstruct the hash independently.

**Why a Merkle tree specifically (not just a hash of the whole metadata)?**

Because of hardware signers like Ledger. The full metadata is ~2 MB — way too large for a Ledger device's memory. But the Ledger needs to read the metadata to show the user what they're signing (field names, types, amounts).

With a Merkle tree, the wallet only sends the **specific types** needed for that transaction plus a **Merkle proof** that those types are part of the full metadata. The Ledger can:
- Verify the proof against the root hash (which is in the signing payload)
- Display the transaction details to the user
- All without needing the full 2 MB metadata in memory

**An ironic shortcoming**: the native token symbol and decimals (e.g., "DOT" with 10 decimals) are NOT first-class fields in the metadata. However, they ARE implicitly included in the Merkle root hash computation (they're provided at compile time). So the hash covers them, but you can't introspectively read them from the metadata. This means a wallet must know the decimals from an external source, which is a known gap that the community has discussed fixing (Gavin Wood himself commented that this should be added as a metadata constant, but as of this writing, nobody has implemented it).

**Note**: CheckMetadataHash is opt-in. Libraries like Papi make it easy to enable. Older wallets like the Polkadot.js extension used a different, inferior approach: they required users to manually "update metadata" when visiting a dapp, essentially trusting the dapp to provide correct metadata. The Merkle tree approach is trustless and strictly superior.

## 37. Parsing a Real Extrinsic — Byte by Byte

A `balances.transfer_allow_death` transaction:

```
49 02 84 00 d4 35 93 c7 15 fd d3 1c 61 14 1a bd
04 a9 9f d6 82 2c 85 58 85 4c cd e3 9a 56 84 e7
a5 6d a2 7d 01 a8 b6 3f 5c 2f 6e 4d 72 5b 38 9e
3b 4d 7c 0e 45 96 1a 8a 9b 7c d1 f3 8e 0a 5c 7a
3b 2f 1d 6e 4a 8c 5b 2d 7f 3e 9a 1c 6d 4b 8a 2e
7d 5f 3c 9b 1a 6e 4d 8c 2b 5f 7a 3d 9e 1c 6b 4a
8d 95 03 14 00 07 00 00 a4 e4 a6 4d ca ba e6 f6
f9 5d e5 2a 81 d4 23 61 92 64 43 e2 6e fe de 9c
7c d9 d6 03 4e 43 c7 61 07 00 b0 d8 63 08
```

### 1. Compact Length

`49 02` → `0x0249` in mode 1 compact → `0x0249 >> 2` = `0x0092` = **146 bytes** of payload follow.

### 2. Header Byte

`84` = `10000100`:
- Bit 7 = 1 → **signed**
- Bits 6-0 = `0000100` = 4 → **version 4**

### 3. Address (MultiAddress)

`00` → variant 0 = `Id(AccountId32)`, followed by 32 bytes:
`d4 35 93 c7 ... a2 7d` → Alice's account on test networks.

### 4. Signature (MultiSignature)

`01` → variant 1 = **Sr25519**, followed by 64 bytes of signature data.

### 5. Signed Extensions (extras)

Processing in metadata-defined order:

**CheckMortality → Era:** `95 03` (2 bytes → mortal transaction, encodes period and phase)

**CheckNonce → Compact<u32>:** `14` → `0x14 >> 2` = **5** (nonce is 5)

**ChargeTransactionPayment → Compact<u128>:** `00` → `0x00 >> 2` = **0** (no tip)

### 6. Call Data

**Pallet index:** `07` → Balances (pallet index 7 in Polkadot)

**Call index:** `00` → `transfer_allow_death` (variant 0)

**Argument 1 — dest (MultiAddress):** `00` (Id variant) + 32 bytes: `a4 e4 a6 4d ... c7 61`

**Argument 2 — value (Compact<u128>):** `07 00 b0 d8 63 08`
- `0x07` = `00000111`, LSB 2 bits = `11` → mode 3
- Upper 6 bits: `000001` = 1 → 1 + 4 = **5 bytes** of value
- Value bytes: `00 b0 d8 63 08` → LE → `0x0863d8b000` = **36,000,000,000** (36 DOT with 10 decimals)

### Decoded result

```
Signed Extrinsic v4:
  Sender:    0xd43593c7...a27d (Alice)
  Signature: Sr25519 (64 bytes)
  Era:       mortal (from 0x9503)
  Nonce:     5
  Tip:       0
  Call:      Balances.transfer_allow_death(
               dest:  0xa4e4a64d...c761,
               value: 36 DOT
             )
```

Everything decoded using only SCALE rules + metadata.

---

# Part IX: Tools and Implementations

## 38. Libraries

- **parity-scale-codec** (Rust) — the reference implementation, used throughout the Polkadot SDK
- **scale-info** (Rust) — type information for the metadata registry
- **@polkadot-api/substrate-bindings** (TypeScript) — used by Papi (Joseph's team)
- **polkadot-js/types** (TypeScript) — used by PolkadotJS
- **scale-codec** implementations exist for Go, Python, Java, C++, and other languages

## 39. Interactive Tools

- **Papi dev console** — when hovering over SCALE-encoded data, it highlights each part and shows what it corresponds to
- **scale.papihow** — a web tool (alpha) for experimenting with SCALE definitions, encoding, and decoding
- **Subscan / Polkadot.js Apps** — block explorers that decode extrinsics and storage using metadata

---

# Appendix: Key Concepts Quick Reference

| Concept | Key Point |
|---------|-----------|
| SCALE | Non-self-describing, little-endian, concatenated codec |
| Why not Protobuf | Protocol independence, `no_std` compat, minimality, Rust integration |
| Endianness | Byte order only; bits inside bytes are unchanged |
| Free casting | LE lets you cast down types by reading fewer bytes from the same address |
| Compact flag | 2 LSB of first byte: 00=1B, 01=2B, 10=4B, 11=variable |
| Compact validity | Values MUST use the smallest mode possible; redundant encodings are invalid |
| Enum tag | First byte = variant index; max 256 variants |
| Vector | Compact length prefix + concatenated elements; max length = max compact (2^536) |
| Array | No length prefix; size known at compile time |
| Strings | NOT first-class; just `Vec<u8>` interpreted at app level |
| Maps/Sets | NOT first-class; Map = `Vec<(K,V)>`, Set = `Vec<T>` |
| Opaque | Compact byte count + raw blob, decoded later by someone else |
| Bitvec | Compact bit count + packed bytes (Lsb0 ordering in Polkadot) |
| Type Registry | Normalized (flat) array of type definitions; everything references by ID |
| TypeDef | Composite, Variant, Sequence, Array, Tuple, Primitive, Compact, BitSequence |
| Type IDs | Not stable across runtime versions; refreshed with each spec_version change |
| Metadata bootstrapping | Metadata codec is a well-known hardcoded type; defined in `frame-metadata` repo |
| Metadata versions | V14 (type registry), V15 (+runtime APIs), V16 (+view functions, tx extensions) |
| Metadata API | `metadata_metadata()` = always V14; `metadata_at_version(n)` for specific version |
| CheckMetadataHash | Merkle tree of metadata; root hash signed as additional_signed; enables trustless verification |
| Merkle proof for Ledger | Send only needed types + proof instead of full 2MB metadata to hardware signers |
| Storage keys | Twox128(pallet) ++ Twox128(entry) ++ hasher(key) |
| Concat hashers | Include raw key after hash → enables iteration |
| Extrinsic header | Bit 7 = signed flag; bits 6-0 = version (currently 4) |
| Extra data | Part of extension that travels in the extrinsic |
| Additional signed | Part of extension that is signed but NOT transmitted |
| Signing payload | call_data + extras + additional_signed (hashed if > 256 bytes) |
| Derive macros | `#[derive(Encode, Decode, TypeInfo)]` generates everything automatically |