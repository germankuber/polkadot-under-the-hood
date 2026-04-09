# Substrate Storage Deep Dive — Source-Code Level Study

## Table of Contents

1. [The Trie: Base-16 Modified Merkle Patricia Trie](#1-the-trie)
2. [Node Encoding and Serialization](#2-node-encoding)
3. [Nibbles and Key Navigation](#3-nibbles)
4. [The Physical Database](#4-physical-database)
5. [Trie Lookup Step by Step](#5-trie-lookup)
6. [Host Functions: The Wasm↔Native Bridge](#6-host-functions)
7. [The Linking: Connecting Wasm Imports to Native Functions](#7-linking)
8. [FRAME Storage Keys: How Pallets Build Keys](#8-frame-storage-keys)
9. [The Overlay: Transactional In-Memory Buffer](#9-overlay)
10. [Applying the Overlay to the Trie](#10-applying-overlay)
11. [Pruning and Archive Nodes](#11-pruning)
12. [State Proofs](#12-state-proofs)
13. [Economic Model of Storage](#13-economic-model)

---

## 1. The Trie

**Crate:** `sp-trie` (`substrate/primitives/trie/src/lib.rs`)

Substrate uses a **Base-16 Modified Merkle Patricia Trie** to store all on-chain state. This is not a regular key-value store — it's a tree structure where every node is hashed, and the hash of the root node (the "state root") goes into the block header. This single hash cryptographically commits to the entire state of the chain.

> **Interactive Simulator:** You can explore how this trie works visually — insert keys, inspect nodes, generate Merkle proofs, and see the KVDB — in the [Merkle Patricia Trie Simulator](https://merklepatriciatree.vercel.app) ([source code](https://github.com/germankuber/merkle-patricia-trie-simulator)).

### Two Layouts: V0 and V1

Substrate defines two trie layouts, both generic over a Hasher `H`:

```rust
pub struct LayoutV0<H>(PhantomData<H>);
pub struct LayoutV1<H>(PhantomData<H>);
```

Both implement the `TrieLayout` trait from the `trie-db` crate. The critical differences:

**LayoutV0:**
```rust
const USE_EXTENSION: bool = false;
const ALLOW_EMPTY: bool = true;
const MAX_INLINE_VALUE: Option<u32> = None;  // all values stored inline
```

**LayoutV1:**
```rust
const USE_EXTENSION: bool = false;
const ALLOW_EMPTY: bool = true;
const MAX_INLINE_VALUE: Option<u32> = Some(TRIE_VALUE_NODE_THRESHOLD); // 33 bytes
```

In V0, all values are stored inline inside trie nodes. In V1, values exceeding `TRIE_VALUE_NODE_THRESHOLD` (33 bytes, defined in `sp-core`) are stored separately and referenced by hash. This is critical for parachains: when a light client or validator requests a state proof, V1 avoids including large values that aren't needed — only their hashes. This significantly reduces PoV (Proof of Validity) size, which has a 5MB limit.

Both layouts set `USE_EXTENSION: false`. Unlike Ethereum's trie which uses extension nodes, Substrate fuses extensions into branch nodes as "partial keys" (nibble sequences). This is more space-efficient.

### Key Functions in sp-trie

**`read_trie_value`** — Read a value from the trie given a DB, root hash, and key. Constructs a `TrieDB` and calls `get(key)`.

**`delta_trie_root`** — The workhorse function. Takes a set of changes (deltas) and applies them to the trie, returning the new root hash. It sorts deltas by key before applying for deterministic trie traversal. The test `generate_storage_root_with_proof_works_independently_from_the_delta_order` verifies this determinism.

**`generate_trie_proof` / `verify_trie_proof`** — Generate and verify Merkle proofs for sets of keys. Supports both inclusion proofs (key exists with value) and non-inclusion proofs (key does not exist).

**`empty_trie_root` / `empty_child_trie_root`** — Compute the hash of an empty trie.

### Child Tries and KeySpacedDB

```rust
pub struct KeySpacedDB<'a, DB: ?Sized, H>(&'a DB, &'a [u8], PhantomData<H>);
```

A child trie is a separate trie that hangs off the main trie. To avoid key collisions in the underlying database, `KeySpacedDB` prepends a keyspace to all key prefixes. When `read_child_trie_value` is called, it wraps the DB with `KeySpacedDB::new(db, keyspace)` and operates normally. The main trie stores the root hash of each child trie under a well-known key prefix (`:child_storage:default:<child_key>`).

Child tries are used by `pallet-contracts` (each smart contract has its own child trie) and were used by crowdloans.

---

## 2. Node Encoding

**File:** `sp-trie/src/node_codec.rs`

### Node Types

Every trie node is serialized as a byte sequence following this general format:

```
[Header] [Partial Key] [Bitmap (branch only)] [Value] [Children (branch only)]
```

The `NodeCodec<H>` struct implements `NodeCodecT` from `trie-db`, providing encoding and decoding of nodes.

### Header Byte Decoding

**File:** `sp-trie/src/node_header.rs`

The first byte of every node determines its type. The two most significant bits provide the primary dispatch:

```
Bits 7-6:
  01 (0x40) → Leaf
  11 (0xC0) → Branch WITH value
  10 (0x80) → Branch WITHOUT value
  00 (0x00) → Special (null, or V1 hashed-value variants)
```

The special `0x00` space is further subdivided:

```
00000000  → Empty/Null (the empty trie)
001xxxxx  → HashedValueLeaf (V1: value stored by hash, 5 bits for nibble count)
0001xxxx  → HashedValueBranch (V1: value stored by hash, 4 bits for nibble count)
00000001  → ESCAPE_COMPACT_HEADER (reserved for future use)
```

Each variant uses progressively more prefix bits, leaving fewer bits for the inline nibble count. The `decode_size` function handles this: remaining bits encode the nibble count, and if maxed out, a `Compact<u32>` overflow follows.

For Leaf/Branch (2-bit prefix): 6 bits remain → up to 63 nibbles inline.
For HashedValueLeaf (3-bit prefix): 5 bits → up to 31 nibbles inline.
For HashedValueBranch (4-bit prefix): 4 bits → up to 15 nibbles inline.

### The decode_plan Function

This is the central parser. It receives raw bytes and produces a `NodePlan` — a zero-copy description of the node using byte ranges rather than copied data.

**For Branch nodes:**
1. Read header → get `nibble_count` and whether it has a value
2. Read partial key bytes: `nibble_count.div_ceil(2)` bytes, with padding check if nibble count is odd
3. Read 2-byte bitmap (little-endian `u16`): 16 bits, one per possible child (nibbles 0x0–0xF)
4. Read value: either `Compact<u32>` length + inline bytes, or fixed-length hash (32 bytes for Blake2)
5. For each set bit in bitmap: read `Compact<u32>` length + child data. If length equals hash length (32), it's a `NodeHandlePlan::Hash`; otherwise it's `NodeHandlePlan::Inline`

**For Leaf nodes:**
1. Read header → get `nibble_count`
2. Read partial key bytes with padding check
3. Read value (inline or hashed)

### The Bitmap

```rust
pub(crate) struct Bitmap(u16);
```

A 16-bit value where each bit indicates the presence of a child at the corresponding nibble position. Encoded as little-endian. Validation ensures a branch always has at least one child (`value == 0` returns error).

### Encoding Functions

`leaf_node` and `branch_node_nibbled` are the mirrors of `decode_plan`.

`branch_node_nibbled` has an interesting implementation detail: the bitmap position is reserved with placeholder zeros, children are written (building the bitmap in parallel), and then the bitmap bytes are overwritten at the reserved position. This is because the bitmap precedes the children in the serialized format, but its content depends on which children exist.

`extension_node` and `branch_node` (without nibble) both call `unreachable!()` — Substrate does not use extension nodes or branches without partial keys.

### Trie Constants

```rust
mod trie_constants {
    pub const LEAF_PREFIX_MASK: u8 = 0b_01 << 6;        // 0x40
    pub const BRANCH_WITHOUT_MASK: u8 = 0b_10 << 6;     // 0x80
    pub const BRANCH_WITH_MASK: u8 = 0b_11 << 6;        // 0xC0
    pub const EMPTY_TRIE: u8 = 0x00;
    pub const ALT_HASHING_LEAF_PREFIX_MASK: u8 = 0b_001 << 5;   // 0x20
    pub const ALT_HASHING_BRANCH_WITH_MASK: u8 = 0b_0001 << 4;  // 0x10
    pub const ESCAPE_COMPACT_HEADER: u8 = 0x01;
}
```

---

## 3. Nibbles

A **nibble** is 4 bits — half a byte. One byte contains two nibbles:

```
Byte:     0xAB
Nibble 0: 0xA  (high 4 bits: 1010)
Nibble 1: 0xB  (low 4 bits:  1011)
```

The trie is **Base-16**: at each level, it branches into up to 16 children (0x0 through 0xF), one per nibble value. This is why the bitmap is exactly 16 bits.

### Why Base-16 and not Base-256?

A Base-256 trie (one child per byte value) would need a 256-bit bitmap (32 bytes) and up to 256 child slots per branch. Most would be empty. Base-16 is a balance between tree depth (2× compared to Base-256) and node width (16 slots with a 2-byte bitmap).

### Partial Key Compression

The Modified Merkle Patricia Trie compresses paths where there's no branching. If keys `0xABC` and `0xABF` share the prefix `AB`, the trie stores:

```
Root → Branch(partial_key=[A,B], bitmap has children C and F)
         ├─ hijo[0xC] → Leaf(value=12)
         └─ hijo[0xF] → Leaf(value=99)
```

The `nibble_count` in node headers represents this compressed partial key length.

### Padding

When `nibble_count` is odd, the partial key bytes have a padding nibble at the start that must be zero:

```
3 nibbles A, B, C → stored in 2 bytes: [0x0A, 0xBC]
                                          ^-- padding nibble (must be 0)
4 nibbles A, B, C, D → stored in 2 bytes: [0xAB, 0xCD] (no padding)
```

The codec validates this:
```rust
if padding && nibble_ops::pad_left(data[input.offset]) != 0 {
    return Err(Error::BadFormat);
}
```

### Storage Keys as Nibbles

A FRAME storage key like `twox128("Balances") ++ twox128("Account") ++ blake2_128_concat(account_id)` produces ~64 bytes = 128 nibbles. The trie navigates these 128 nibbles, compressing paths where there's no branching.

---

## 4. The Physical Database

The trie is a logical structure. What gets persisted in RocksDB (or ParityDB) is a flat key-value store where:

- **Key** = Blake2 hash of the serialized node bytes
- **Value** = the serialized node bytes

### Example: Single Key-Value

Storage: `0xABC → 12`

The trie is one leaf node:
```
leaf_bytes = [0x43, 0xAB, 0xC0, 0x04, 0x0C]
              ^      ^          ^     ^
              |      |          |     value 12
              |      |          Compact(1) = 1 byte of value
              |      partial key: nibbles A,B,C (with padding)
              header: Leaf, 3 nibbles
```

SCALE compact encoding for small values (0-63, single-byte mode): `Compact(n) = n << 2`. The two lowest bits `00` indicate single-byte mode.

```
Compact(0)  → 0 << 2  → 0x00
Compact(1)  → 1 << 2  → 0x04
Compact(2)  → 2 << 2  → 0x08
Compact(3)  → 3 << 2  → 0x0C
Compact(4)  → 4 << 2  → 0x10
Compact(10) → 10 << 2 → 0x28
Compact(63) → 63 << 2 → 0xFC
```

If the two lowest bits were `01`, it would be two-byte mode; `10` = four-byte mode; `11` = big-integer mode.

This gets hashed: `hash_leaf = Blake2(leaf_bytes) = 0x7f3a...`

In RocksDB:
```
┌─────────────────────┬───────────────────────────────┐
│ DB Key              │ DB Value                      │
├─────────────────────┼───────────────────────────────┤
│ 0x7f3a... (32 bytes)│ [0x43, 0xAB, 0xC0, 0x04, 0x0C]│
└─────────────────────┴───────────────────────────────┘
```

The **state root** in the block header is `0x7f3a...`.

### Example: Two Key-Values with Inline Children

Storage: `0xABC → 12`, `0xABF → 30000`

Two leaves:
```
leaf_C = [0x40, 0x04, 0x0C]           → 3 bytes (Leaf, 0 nibbles, value=12)
leaf_F = [0x40, 0x08, 0x30, 0x75]     → 4 bytes (Leaf, 0 nibbles, value=30000 LE)
```

`Compact(2) = 0x08` because `2 << 2 = 8`.

Since both leaves are smaller than 32 bytes (the hash length), they go **inline** inside the branch:

```
branch bytes:
0x82              → header: Branch without value, 2 nibbles partial
0xAB              → partial key: A, B
0x00, 0x90        → bitmap: bits C and F set (little-endian u16)
0x0C              → Compact(3): first child is 3 bytes
0x40, 0x04, 0x0C  → leaf_C inline
0x10              → Compact(4): second child is 4 bytes
0x40, 0x08, 0x30, 0x75  → leaf_F inline
```

`Compact(3) = 0x0C` (3 << 2 = 12), `Compact(4) = 0x10` (4 << 2 = 16).

In RocksDB: **one single entry** containing the branch with both leaves embedded.

### Inline vs. Hash Referenced Children

The rule: if a child's serialized size is less than 32 bytes (the hash length), it's stored inline within the parent. If 32 bytes or more, it's stored separately and the parent holds the 32-byte hash reference.

This makes sense economically: storing a 32-byte hash to reference something smaller than 32 bytes wastes space.

### Example: Branch WITH Value

If we store three key-values: `0xAB → 50`, `0xABC → 12`, `0xABF → 30000`:

```
Root → Branch(partial_key=[A,B], has_value=true, value=50)
         ├─ hijo[0xC] → Leaf(value=12)
         └─ hijo[0xF] → Leaf(value=30000)
```

The branch node has `has_value=true` (header byte `0xC2` instead of `0x82`) and stores the value `50` for key `0xAB`. The children store the values for `0xABC` and `0xABF`. Without the key `0xAB → 50`, the branch would have `has_value=false` — it would exist only as a branching point.

### Nodes Are Never Replaced

When a new block modifies state, new trie nodes are **added** to RocksDB. Old nodes remain. Both the old state root and the new state root are valid and can reconstruct their respective states. Shared nodes (those not affected by changes) are referenced by both roots.

---

## 5. Trie Lookup Step by Step

Given: state root `0xaa11...`, looking up key `0xABC`.

The DB contains the branch from the previous example.

**Step 1:** Go to RocksDB with key `0xaa11...`. Get back the branch bytes:
```
[0x82, 0xAB, 0x00, 0x90, 0x0C, 0x40, 0x04, 0x0C, 0x10, 0x40, 0x08, 0x30, 0x75]
```

Nibbles to consume: `A, B, C`.

**Step 2:** Read byte 0 → `0x82`. Binary: `10_000010`. High bits `10` → Branch without value. Low bits `000010` = 2 → partial key of 2 nibbles.

**Step 3:** Read byte 1 → `0xAB`. Partial key: nibbles `A`, `B`. Compare with pending nibbles `A, B, C`. First 2 match. Consume `A, B`. Remaining: `C`.

**Step 4:** Read bytes 2-3 → `0x00, 0x90`. Bitmap. The bit at position C (12) is set.

**Step 5:** Next nibble is `C`. Present in bitmap? Yes. It's the first set bit, so read the first child.

**Step 6:** Read byte 4 → `0x0C` = Compact(3). First child is 3 bytes.

**Step 7:** Read bytes 5-7 → `0x40, 0x04, 0x0C`. Decode: `0x40` → Leaf, 0 nibbles partial. `0x04` → Compact(1), 1 byte of value. `0x0C` → 12.

**Step 8:** No nibbles remaining. Match. **Result: `Some(12)`**.

### Looking up the Second Key: `0xABF`

**Steps 1-4:** Identical to `0xABC`. After consuming partial key `A,B`, remaining nibble is `F`.

**Step 5:** Next nibble is `F`. Present in bitmap? Yes. But `F` is the **second** child present. Must skip the first child to reach it.

**Step 6:** Read byte 4 → `0x0C` = Compact(3). Skip 3 bytes (the first child). Now at byte 8.

**Step 7:** Read byte 8 → `0x10` = Compact(4). Second child is 4 bytes.

**Step 8:** Read bytes 9-12 → `0x40, 0x08, 0x30, 0x75`. Decode: `0x40` → Leaf, 0 nibbles. `0x08` → Compact(2), 2 bytes of value. `0x30, 0x75` → 30000 in little-endian.

**Step 9:** No nibbles remaining. Match. **Result: `Some(30000)`**.

### Looking up a non-existent key

Looking up `0xABD`: same steps until step 5. The bitmap does not have bit D set → **Result: `None`**.

Looking up `0xAC`: at step 3, partial key is `A,B` but search key has `A,C`. Second nibble doesn't match → **Result: `None`**.

---

## 6. Host Functions: The Wasm↔Native Bridge

**File:** `substrate/primitives/io/src/lib.rs`

The runtime runs inside Wasm. It cannot access RocksDB or the trie directly. Communication happens through **host functions** — functions the runtime calls that the native node executes.

### The `#[runtime_interface]` Macro

```rust
#[runtime_interface]
pub trait Storage {
    fn get(&mut self, key: &[u8]) -> Option<bytes::Bytes> {
        self.storage(key).map(|s| bytes::Bytes::from(s.to_vec()))
    }
    fn set(&mut self, key: &[u8], value: &[u8]) {
        self.set_storage(key.to_vec(), value.to_vec());
    }
    fn root(&mut self, version: StateVersion) -> Vec<u8> {
        self.storage_root(version)
    }
    fn start_transaction(&mut self) { self.storage_start_transaction(); }
    fn rollback_transaction(&mut self) { self.storage_rollback_transaction()...; }
    fn commit_transaction(&mut self) { self.storage_commit_transaction()...; }
}
```

The `#[runtime_interface]` macro generates **two different implementations** depending on the compilation target:

**For native (std):** The actual implementation that calls `self.storage(key)` on the `Externalities` object, which has access to the overlay and trie.

**For Wasm (substrate_runtime):** A stub that makes an external function call:
```rust
extern "C" {
    fn ext_storage_get_version_1(key: u64) -> u64;
}
```

The function name follows the pattern: `ext_` + trait name in snake_case + `_` + function name + `_version_` + version number.

### The `Storage` Trait — Key Functions

- **`get(key)`** — Returns `Option<bytes::Bytes>`. The overlay is checked first; if not found there, the trie is queried.
- **`set(key, value)`** — Writes to the overlay, NOT to the trie.
- **`root(version)`** — Commits all overlay changes to the trie and returns the new state root. `StateVersion` determines V0 vs V1 layout.
- **`start_transaction()` / `rollback_transaction()` / `commit_transaction()`** — Transactional storage support for atomic extrinsic execution.

### DefaultChildStorage

The same interface exists for child tries via the `DefaultChildStorage` trait. Each operation takes an additional `storage_key` parameter identifying the child trie, and internally wraps with `ChildInfo::new_default(storage_key)`.

### SubstrateHostFunctions

All host functions are aggregated in a single type:
```rust
pub type SubstrateHostFunctions = (
    storage::HostFunctions,
    default_child_storage::HostFunctions,
    misc::HostFunctions,
    crypto::HostFunctions,
    hashing::HostFunctions,
    allocator::HostFunctions,
    ...
);
```

Each `HostFunctions` type is generated by the `#[runtime_interface]` macro and contains the list of native functions to register with the Wasm executor.

---

## 7. The Linking

**Files:** `substrate/client/executor/wasmtime/src/imports.rs`, `substrate/client/executor/src/wasm_runtime.rs`

### Creating the Wasm Runtime

```rust
pub fn create_wasm_runtime_with_code<H: HostFunctions>(
    wasm_method: WasmExecutionMethod,
    heap_alloc_strategy: HeapAllocStrategy,
    blob: RuntimeBlob,
    ...
) -> Result<Box<dyn WasmModule>, WasmError>
```

The generic `H: HostFunctions` is `SubstrateHostFunctions` when called from a real node. This function delegates to `sc_executor_wasmtime::create_runtime::<H>(blob, config)`, which compiles the Wasm blob and links host functions.

PolkaVM (RISC-V) support is also present but not yet active in production:
```rust
if let Some(blob) = blob.as_polkavm_blob() {
    return sc_executor_polkavm::create_runtime::<H>(blob);
}
```

### The prepare_imports Function

```rust
pub(crate) fn prepare_imports<H: HostFunctions>(
    linker: &mut Linker<StoreData>,
    module: &Module,
    allow_missing_func_imports: bool,
) -> Result<(), WasmError>
```

This is where the actual linking happens:

**Step 1:** Iterate all Wasm module imports. All must come from module `"env"`:
```rust
for import_ty in module.imports() {
    if import_ty.module() != "env" { return Err(...) }
    pending_func_imports.insert(name, (import_ty, func_ty));
}
```

**Step 2:** Register host functions against pending imports:
```rust
H::register_static(&mut registry)?;
```

This calls `register_static` for each host function:
```rust
fn register_static(&mut self, fn_name: &str, func: impl IntoFunc<...>) {
    if self.pending_func_imports.remove(fn_name).is_some() {
        self.linker.func_wrap("env", fn_name, func)...;
    }
}
```

The matching is by exact function name (e.g., `"ext_storage_get_version_1"`). If the Wasm module imports it and a host function provides it, they get connected via `linker.func_wrap`.

**Step 3:** Handle unresolved imports. If `allow_missing_func_imports` is true, stubs returning errors are created. Otherwise, compilation fails.

### Runtime Cache

```rust
pub struct RuntimeCache {
    runtimes: Mutex<LruMap<VersionedRuntimeId, Arc<VersionedRuntime>>>,
    max_runtime_instances: usize,
}
```

Compiling Wasm is expensive, so runtimes are cached by `code_hash`. A pool of pre-created instances avoids instantiation costs on each block.

### Complete Linking Flow

```
#[runtime_interface] on trait Storage
  → generates storage::HostFunctions with register_static("ext_storage_get_version_1", closure)
  → grouped into SubstrateHostFunctions = (storage::HostFunctions, ...)
  → passed to create_wasm_runtime_with_code::<SubstrateHostFunctions>(blob)
  → calls sc_executor_wasmtime::create_runtime::<H>(blob)
  → calls prepare_imports::<H>(linker, module)
  → H::register_static(&mut registry)
  → registry.register_static("ext_storage_get_version_1", closure)
  → linker.func_wrap("env", "ext_storage_get_version_1", closure)
  → Wasmtime connects Wasm import to native closure
```

---

## 8. FRAME Storage Keys

**Files:** `frame/support/src/storage/types/map.rs`, `frame/support/src/storage/generator/map.rs`

### Key Construction

When a pallet defines:
```rust
#[pallet::storage]
pub type Account<T> = StorageMap<_, Blake2_128Concat, AccountId, AccountInfo>;
```

The `#[pallet::storage]` macro generates a `StorageInstance` implementation:
```rust
struct _GeneratedPrefixForAccount;
impl StorageInstance for _GeneratedPrefixForAccount {
    fn pallet_prefix() -> &'static str { "Balances" }
    const STORAGE_PREFIX: &'static str = "Account";
}
```

The final storage key is constructed by `storage_map_final_key`:
```rust
fn storage_map_final_key<KeyArg>(key: KeyArg) -> Vec<u8> {
    let storage_prefix = storage_prefix(Self::pallet_prefix(), Self::storage_prefix());
    let key_hashed = key.using_encoded(Self::Hasher::hash);
    let mut final_key = Vec::with_capacity(storage_prefix.len() + key_hashed.as_ref().len());
    final_key.extend_from_slice(&storage_prefix);
    final_key.extend_from_slice(key_hashed.as_ref());
    final_key
}
```

The formula: `twox128(pallet_name) ++ twox128(storage_name) ++ Hasher::hash(SCALE_encode(key))`

### Key Length Variability

Storage keys are **not** fixed length. The length depends on the storage type and hasher:

- `StorageValue`: 32 bytes (two twox128 hashes: 16 + 16)
- `StorageMap` with `Blake2_128Concat`: 32 + 16 (hash) + encoded key length
- `StorageMap` with `Twox64Concat`: 32 + 8 (hash) + encoded key length
- `StorageMap` with `Identity`: 32 + encoded key length (no hash at all)
- `StorageDoubleMap`: even longer, concatenating two hashers

### The get/set Path

```rust
// In generator/map.rs
fn get<KeyArg: EncodeLike<K>>(key: KeyArg) -> Self::Query {
    G::from_optional_value_to_query(unhashed::get(Self::storage_map_final_key(key).as_ref()))
}

fn insert<KeyArg: EncodeLike<K>, ValArg: EncodeLike<V>>(key: KeyArg, val: ValArg) {
    unhashed::put(Self::storage_map_final_key(key).as_ref(), &val)
}
```

The `from_optional_value_to_query` call is where the `QueryKind` matters. FRAME provides three query kinds defined in `frame/support/src/storage/types/mod.rs`:

- **`OptionQuery`** — `get()` returns `Option<Value>`. If the key doesn't exist, returns `None`.
- **`ValueQuery`** — `get()` returns `Value` directly. If the key doesn't exist, returns the `OnEmpty` default (e.g., zero for integers).
- **`ResultQuery`** — `get()` returns `Result<Value, Error>`. If the key doesn't exist, returns `Err(OnEmpty::get())`.

```rust
// OptionQuery: None when missing
impl QueryKindTrait<Value, GetDefault> for OptionQuery {
    fn from_optional_value_to_query(v: Option<Value>) -> Option<Value> { v }
}

// ValueQuery: default when missing
impl<Value, OnEmpty: Get<Value>> QueryKindTrait<Value, OnEmpty> for ValueQuery {
    fn from_optional_value_to_query(v: Option<Value>) -> Value {
        v.unwrap_or_else(|| OnEmpty::get())
    }
}
```

`unhashed::get` is the function that calls `sp_io::storage::get`:
```rust
pub fn get<T: Decode>(key: &[u8]) -> Option<T> {
    let raw = sp_io::storage::get(key)?;
    T::decode(&mut &raw[..]).ok()
}
```

### Complete Call Path

```
Pallet: Balances::get(account_id)
  → StorageMap::get(account_id)
    → storage_map_final_key(account_id)
      → twox128("Balances") ++ twox128("Account") ++ blake2_128_concat(account_id.encode())
      → final_key: Vec<u8> (~50-80 bytes)
    → unhashed::get(final_key)
      → sp_io::storage::get(final_key)
        → ext_storage_get_version_1  (Wasm → native crossing)
          → Externalities::storage(final_key)
            → overlay.get(final_key)  if not found →
            → trie.get(final_key)
              → nibble-by-nibble traversal through trie nodes
              → decode headers, bitmaps, partial keys
              → reach leaf → read value
            → SCALE decode → AccountInfo
```

---

## 9. The Overlay

**Files:** `substrate/primitives/state-machine/src/overlayed_changes/mod.rs`, `changeset.rs`

### Purpose

The overlay solves two problems:

1. **Atomicity:** If an extrinsic fails mid-execution, all its storage changes must be reverted. Writing directly to the trie would make this impossible.
2. **Performance:** Each trie write requires recalculating hashes along the path to the root. Accumulating changes and applying once is far cheaper.

### Structure

```rust
pub struct OverlayedChanges<H: Hasher> {
    top: OverlayedChangeSet,
    children: Map<StorageKey, (OverlayedChangeSet, ChildInfo)>,
    offchain: OffchainOverlayedChanges,
    storage_transaction_cache: Option<StorageTransactionCache<H>>,
    ...
}
```

`top` is the main storage overlay. `children` holds one overlay per child trie. They transact in lockstep.

### The OverlayedChangeSet (BTreeMap)

```rust
pub type OverlayedChangeSet = OverlayedMap<StorageKey, StorageEntry>;

pub struct OverlayedMap<K, V> {
    changes: BTreeMap<K, OverlayedEntry<V>>,
    dirty_keys: DirtyKeysSets<K>,  // SmallVec<[Set<K>; 5]>
    ...
}
```

This is a standard library `BTreeMap` — a balanced binary tree in memory. Not a trie, not a database. Each key is the full storage key bytes, each value is an `OverlayedEntry`.

### OverlayedEntry — Transaction History Per Key

```rust
pub struct OverlayedEntry<V> {
    transactions: SmallVec<[InnerValue<V>; 5]>,
}

struct InnerValue<V> {
    value: V,
    extrinsics: Extrinsics,
}

pub enum StorageEntry {
    Set(StorageValue),
    Remove,
    Append { data, current_length, materialized_length, parent_size },
}
```

The `transactions` vector holds one entry per transaction layer that modified this key. The last entry is the current value.

### Three-State Reads

```rust
pub fn storage(&mut self, key: &[u8]) -> Option<Option<&[u8]>>
```

Returns `Option<Option<&[u8]>>` — three possible states:
- `None` → overlay doesn't know about this key → go to trie
- `Some(None)` → key was **deleted** in overlay → return None without hitting trie
- `Some(Some(bytes))` → key has this value in overlay → return it

### Writing to the Overlay

```rust
pub fn set_storage(&mut self, key: StorageKey, val: Option<StorageValue>) {
    self.mark_dirty();  // invalidates storage_transaction_cache
    let extrinsic_index = self.extrinsic_index();
    self.top.set(key, val, extrinsic_index);
}
```

`mark_dirty()` sets `storage_transaction_cache = None`, forcing recalculation of the state root next time it's requested.

### Nested Transactions

**start_transaction:** Pushes an empty set onto `dirty_keys`:
```rust
pub fn start_transaction(&mut self) {
    self.dirty_keys.push(Default::default());
}
```

**Writing a key:** The `set` method on `OverlayedEntry` checks `first_write_in_tx`:
```rust
fn set(&mut self, value: ..., first_write_in_tx: bool, ...) {
    if first_write_in_tx || self.transactions.is_empty() {
        self.transactions.push(InnerValue { value, ... });  // new layer
    } else {
        *self.value_mut() = value;  // overwrite current layer
    }
}
```

`first_write_in_tx` comes from `insert_dirty`:

```rust
fn insert_dirty<K: Ord + Hash>(set: &mut DirtyKeysSets<K>, key: K) -> bool {
    set.last_mut().map(|dk| dk.insert(key)).unwrap_or_default()
}
```

`dk.insert(key)` returns `true` if the key was new to the set, `false` if already present. First time you touch "alice" in this transaction → returns `true` → pushes new layer. Second time → returns `false` → overwrites current layer.

This means: no matter how many times you modify a key within one transaction, there's exactly **one entry** per transaction in the `transactions` vector.

**Rollback:** Pop the dirty keys set, and for each dirty key, pop the last transaction entry:
```rust
// Simplified logic
for key in self.dirty_keys.pop() {
    let overlayed = self.changes.get_mut(&key);
    overlayed.pop_transaction();  // removes last entry
    if overlayed.transactions.is_empty() {
        self.changes.remove(&key);  // key didn't exist before
    }
}
```

**Commit:** Pop the dirty keys set, and for each dirty key, merge the last entry into its predecessor:
```rust
// Simplified logic
for key in self.dirty_keys.pop() {
    let overlayed = self.changes.get_mut(&key);
    if has_predecessor {
        let dropped = overlayed.pop_transaction();
        *overlayed.value_mut() = dropped.value;  // overwrite predecessor
    }
}
```

### Example: Transaction Flow

```
Initial: changes = {}, dirty_keys = []

set("alice", 100)
  changes = { "alice": transactions: [Set(100)] }

start_transaction()
  dirty_keys = [ {} ]

set("alice", 50)
  first_write_in_tx = true → push
  changes = { "alice": transactions: [Set(100), Set(50)] }
  dirty_keys = [ {"alice"} ]

set("alice", 30)
  first_write_in_tx = false → overwrite
  changes = { "alice": transactions: [Set(100), Set(30)] }

set("bob", 200)
  first_write_in_tx = true → push
  changes = { "alice": [Set(100), Set(30)], "bob": [Set(200)] }
  dirty_keys = [ {"alice", "bob"} ]

rollback_transaction()
  pop dirty_keys → {"alice", "bob"}
  "alice": pop → transactions: [Set(100)]  ← back to 100
  "bob": pop → transactions: [] → remove from BTreeMap
  changes = { "alice": transactions: [Set(100)] }
```

### Block Execution Flow

```
Block starts → empty BTreeMap

Extrinsic 1:
  start_transaction()
  storage writes → fill BTreeMap
  success → commit_transaction()

Extrinsic 2:
  start_transaction()
  storage writes → fill BTreeMap
  failure → rollback_transaction() → changes discarded

Extrinsic 3:
  start_transaction()
  storage writes → fill BTreeMap
  success → commit_transaction()

End of block → storage_root() → apply BTreeMap to trie
```

One BTreeMap per block. All extrinsics write to it. At the end, everything is applied to the trie in one batch.

### Detailed Multi-Extrinsic Example

```
Block starts → BTreeMap empty

Extrinsic 1: transfer Alice → Bob
  start_transaction()
    dirty_keys = [ {} ]
  set("alice", 90)
    changes = { "alice": [Set(90)] }
    dirty_keys = [ {"alice"} ]
  set("bob", 110)
    changes = { "alice": [Set(90)], "bob": [Set(110)] }
    dirty_keys = [ {"alice", "bob"} ]
  success → commit_transaction()
    dirty_keys = []
    changes = { "alice": [Set(90)], "bob": [Set(110)] }

Extrinsic 2: transfer Bob → Charlie
  start_transaction()
    dirty_keys = [ {} ]
  set("bob", 80)
    changes = { "alice": [Set(90)], "bob": [Set(110), Set(80)] }
    dirty_keys = [ {"bob"} ]
  set("charlie", 130)
    changes = { ..., "bob": [Set(110), Set(80)], "charlie": [Set(130)] }
    dirty_keys = [ {"bob", "charlie"} ]
  success → commit_transaction()
    merge: "bob" [Set(110), Set(80)] → [Set(80)]
    changes = { "alice": [Set(90)], "bob": [Set(80)], "charlie": [Set(130)] }

Extrinsic 3: transfer Alice → Eve (FAILS)
  start_transaction()
    dirty_keys = [ {} ]
  set("alice", -10)
    changes = { "alice": [Set(90), Set(-10)], "bob": ..., "charlie": ... }
    dirty_keys = [ {"alice"} ]
  set("eve", 110)
    changes = { ..., "eve": [Set(110)] }
    dirty_keys = [ {"alice", "eve"} ]
  FAILS → rollback_transaction()
    pop dirty_keys → {"alice", "eve"}
    "alice": pop → [Set(90)]  ← back to 90
    "eve": pop → [] → removed from BTreeMap
    changes = { "alice": [Set(90)], "bob": [Set(80)], "charlie": [Set(130)] }
    Eve never existed. Alice keeps 90.

End of block → storage_root()
  delta = [("alice", Some(90)), ("bob", Some(80)), ("charlie", Some(130))]
  Apply to trie → new state root
```

### Client/Runtime Transaction Protection

The overlay has an `ExecutionMode` (Client vs Runtime) and `num_client_transactions`. When entering runtime execution (`enter_runtime`), existing transactions are protected — the runtime cannot close them. When exiting (`exit_runtime`), any dangling runtime transactions are automatically rolled back.

---

## 10. Applying the Overlay to the Trie

### storage_root()

```rust
pub fn storage_root<B: Backend<H>>(&mut self, backend: &B, state_version: StateVersion) -> (H::Out, bool) {
    if let Some(cache) = &self.storage_transaction_cache {
        return (cache.transaction_storage_root, true);  // cached
    }
    let delta = self.top.changes_mut()
        .map(|(k, v)| (&k[..], v.value().map(|v| &v[..])));
    let (root, transaction) = backend.full_storage_root(delta, child_delta, state_version);
    self.storage_transaction_cache = Some(StorageTransactionCache {
        transaction, transaction_storage_root: root
    });
    (root, false)
}
```

### full_storage_root

```rust
fn full_storage_root(&self, delta, child_deltas, state_version) -> (H::Out, BackendTransaction<H>) {
    // Step 1: Process child tries
    for (child_info, child_delta) in child_deltas {
        let (child_root, empty, child_txs) =
            self.child_storage_root(child_info, child_delta, state_version);
        txs.consolidate(child_txs);
        child_roots.push((prefixed_key, Some(child_root.encode())));
    }
    // Step 2: Process main trie — chain child roots as additional deltas
    let (root, parent_txs) = self.storage_root(
        delta.chain(child_roots.iter().map(...)),
        state_version,
    );
    txs.consolidate(parent_txs);
    (root, txs)
}
```

Child trie roots are stored as values in the main trie. The `.chain()` appends them to the delta so they're applied in the same pass.

### The delta_trie_root Connection

`backend.storage_root(delta, state_version)` internally calls `delta_trie_root` from `sp-trie`:

1. Opens the existing trie with the previous block's root
2. Applies each delta entry: `Some(value)` → insert, `None` → delete
3. Recalculates hashes of affected nodes
4. Returns the new root hash and a `BackendTransaction` (MemoryDB of new/modified nodes)

### Persistence

The `BackendTransaction` is not written to RocksDB immediately. It's cached in `storage_transaction_cache`. When the block is finalized, `drain_storage_changes()` extracts the transaction, and the client writes the new nodes to RocksDB.

### Complete Block Lifecycle

```
Block N-1 persisted in RocksDB with state root 0xaabb...

Block N:
  1. Empty BTreeMap (overlay)
  2. Execute extrinsics → populate BTreeMap
  3. storage_root():
     - Read trie from block N-1 (root 0xaabb...)
     - Apply BTreeMap delta
     - Calculate new root 0xccdd...
     - Generate new trie nodes (in memory)
  4. Verify 0xccdd... matches block header
  5. drain_storage_changes() → extract new nodes
  6. Write new nodes to RocksDB
  7. Block N persisted with state root 0xccdd...
```

---

## 11. Pruning and Archive Nodes

### The Growth Problem

Each block adds new trie nodes to RocksDB. Old nodes are never replaced (they're referenced by old state roots). After millions of blocks, the database grows without bound.

### Two Node Modes

**Archive node:** Keeps everything forever. Can query state at any historical block. Database size: hundreds of GB to terabytes.

**Pruning node:** Keeps only the last N blocks (default 256). Nodes exclusively belonging to older blocks are deleted.

### Reference Counting

Nodes can be shared between blocks (unchanged state is shared). Deleting all nodes from an old block would break newer blocks that share some of those nodes.

Solution: each node has a reference count tracking how many block states reference it. When a block exits the pruning window, its nodes' refcounts are decremented. Nodes reaching refcount zero are deleted.

```
Node A: refcount = 1 (only block 3) → block 3 pruned → refcount 0 → DELETED
Node B: refcount = 2 (blocks 3, 4) → block 3 pruned → refcount 1 → KEPT
```

The refcount is persisted in the database (not in memory) so it survives node restarts.

### Trade-offs

Pruning nodes cannot serve historical state queries. Archive nodes are needed for block explorers, RPC services, and historical analysis. Validators and collators can run with pruning since they only need recent state.

---

## 12. State Proofs

### Concept

A state proof demonstrates that a key has (or doesn't have) a specific value in the trie, given only the state root hash. The proof consists of the trie nodes along the lookup path.

### Proof Generation

**File:** `trie-db/src/proof/generate.rs`

```rust
pub fn generate_proof<D, L, I, K>(db: &D, root: &TrieHash<L>, keys: I) -> Result<Vec<Vec<u8>>, ...>
```

The function:
1. Sorts and deduplicates keys (for efficient left-to-right traversal)
2. For each key, performs a trie lookup with a `Recorder` attached
3. The recorder captures every node visited during the lookup
4. Uses a stack to track the path and avoid duplicating shared nodes between keys
5. Returns `Vec<Vec<u8>>` — the serialized trie nodes

When multiple keys share a prefix, they share path nodes. The sorted order ensures the shared branch is only included once.

### Proof Verification

The verifier receives the proof and the state root:

1. Hashes each proof node → builds a `MemoryDB` (hashmap of `hash → bytes`)
2. Constructs a `TrieDB` using the MemoryDB and the state root
3. Performs normal trie lookups using this mini in-memory trie
4. If the lookup succeeds and values match → proof valid
5. If any node is missing from the MemoryDB → proof invalid

```rust
pub fn verify_trie_proof<L, I, K, V>(
    root: &TrieHash<L>,
    proof: &[Vec<u8>],
    items: I,  // (key, Option<value>) pairs
) -> Result<(), VerifyError<...>>
```

Items with `Some(value)` verify inclusion. Items with `None` verify non-inclusion.

### Non-Inclusion Proofs

To prove a key doesn't exist, the proof includes nodes along the path where the search fails. For example, if looking up `0xABD` and the branch's bitmap doesn't have bit D set, the proof is just the branch node. The verifier sees the missing bit and confirms non-existence.

### Connection to Parachain PoV

When a collator builds a parachain block, every `storage::get` during execution visits trie nodes. A recorder captures all visited nodes. The collection of these nodes is the PoV (Proof of Validity).

Relay chain validators receive the PoV and:
1. Take the state root from the parachain's last finalized block
2. Build a MemoryDB from the PoV nodes
3. Re-execute all extrinsics
4. Every storage access reads from the PoV's mini-trie
5. If all lookups succeed and the new state root matches → block valid

This is why `LayoutV1` matters: large values are stored by hash, not inline. The PoV doesn't need to include the full value of every neighboring node in the path — just their hashes. This keeps the PoV under the 5MB limit.

---

## 13. Economic Model of Storage

**File:** `substrate/frame/balances/src/lib.rs`

### Two Protection Mechanisms

**Fees (short-term):** Each storage operation has a weight cost translated into transaction fees. Writes are more expensive than reads.

**Storage deposits (long-term):** Users must lock balance proportional to the storage they occupy. The deposit is returned when the data is deleted.

### The Existential Deposit

```rust
#[pallet::constant]
type ExistentialDeposit: Get<Self::Balance>;
```

The minimum balance to keep an account alive. In Polkadot: 1 DOT. If an account's free balance drops below this threshold, the account is "reaped" — deleted from the trie.

### The Reaping Mechanism

The core logic is in `try_mutate_account`:

```rust
let ed = Self::ed();
let maybe_dust = if account.free < ed && account.reserved.is_zero() {
    if account.free.is_zero() {
        None
    } else {
        Some(account.free)  // "dust" — too small to keep
    }
} else {
    *maybe_account = Some(account);  // account survives
    None
};
```

Three outcomes:
- `free >= ed` → account survives, stored in trie
- `free < ed` and `reserved == 0` and `free > 0` → "dust", account reaped, remaining balance handled by `DustRemoval`
- `free == 0` and `reserved == 0` → account empty, reaped

When `maybe_account` becomes `None`, the `try_mutate_exists` call deletes the entry from storage. The account disappears from the trie.

### Provider/Consumer Reference System

```rust
if !did_provide && does_provide {
    frame_system::Pallet::<T>::inc_providers(who);
}
if did_provide && !does_provide {
    frame_system::Pallet::<T>::dec_providers(who)?;
}
```

`does_provide` is `account.free >= ed`. Crossing the ED threshold upward increments the provider ref in `frame_system`. Crossing downward decrements it. When provider refs reach zero (and no consumers exist), `frame_system` removes the account entirely (nonce, all data).

### Transfer Variants

**`transfer_allow_death`** — Uses `Expendable` preservation. If the transfer leaves the sender below ED, the account is reaped:
```
Alice: 1.5 DOT, transfers 1.2 DOT → Alice: 0.3 DOT < 1.0 DOT (ED) → REAPED
```

**`transfer_keep_alive`** — Uses `Preserve` preservation. If the transfer would leave the sender below ED, it fails:
```
Alice: 1.5 DOT, transfers 1.2 DOT → would leave 0.3 < ED → ERROR, transfer rejected
```

### Storage Items in pallet-balances

Each account can have up to 4 entries in the trie:
```rust
pub type Account<T, I> = StorageMap<_, Blake2_128Concat, T::AccountId, AccountData<T::Balance>>;
pub type Holds<T, I> = StorageMap<_, Blake2_128Concat, T::AccountId, BoundedVec<...>>;
pub type Freezes<T, I> = StorageMap<_, Blake2_128Concat, T::AccountId, BoundedVec<...>>;
pub type Locks<T, I> = StorageMap<_, Blake2_128Concat, T::AccountId, WeakBoundedVec<...>>;
```

The existential deposit ensures this storage is "paid for" by requiring minimum balance.

### Genesis Validation

```rust
fn build(&self) {
    for (_, balance) in &self.balances {
        assert!(
            *balance >= <T as Config<I>>::ExistentialDeposit::get(),
            "the balance of any account should always be at least the existential deposit.",
        )
    }
}
```

### Integrity Check

```rust
fn integrity_test() {
    assert!(
        !<T as Config<I>>::ExistentialDeposit::get().is_zero(),
        "The existential deposit must be greater than zero!"
    );
}
```

### Deposit Pattern in Other Pallets

Other pallets follow the same pattern: require a deposit proportional to stored data size. The deposit is locked (not spent) and returned when data is removed. Examples:
- Registering an on-chain identity → deposit per bytes of name, email, etc.
- Creating a proxy → deposit
- Deploying a smart contract → deposit per code size
- Creating a multisig → deposit

This creates economic incentives: users pay for the storage they occupy for as long as they occupy it, and clean up when they no longer need it.