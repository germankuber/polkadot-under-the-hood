# Transaction Pool Deep Dive: From RPC Submission to Block Inclusion

> A step-by-step walkthrough of how a user-submitted transaction travels through the Polkadot SDK node — from the JSON-RPC entry point, through client-side validation, runtime verification, pool insertion with dependency resolution, and finally to block authoring.

## Table of Contents

- [Transaction Pool Deep Dive: From RPC Submission to Block Inclusion](#transaction-pool-deep-dive-from-rpc-submission-to-block-inclusion)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
- [Part I: The RPC Entry Point](#part-i-the-rpc-entry-point)
  - [1. author\_submitExtrinsic: The Door to the Node](#1-author_submitextrinsic-the-door-to-the-node)
  - [2. Decoding the Raw Bytes](#2-decoding-the-raw-bytes)
  - [3. Best Block Hash: The Validation Anchor](#3-best-block-hash-the-validation-anchor)
  - [4. TransactionSource: Where Did This Come From?](#4-transactionsource-where-did-this-come-from)
- [Part II: The Pool Pipeline — Client Side](#part-ii-the-pool-pipeline--client-side)
  - [5. submit\_one → submit\_at → verify](#5-submit_one--submit_at--verify)
  - [6. verify\_one: The Core Validation Orchestrator](#6-verify_one-the-core-validation-orchestrator)
    - [6.1. hash\_and\_length: Identity and Size](#61-hash_and_length-identity-and-size)
    - [6.2. check\_is\_known: Ban and Duplicate Check](#62-check_is_known-ban-and-duplicate-check)
    - [6.3. The Runtime Call: Crossing the Wasm Boundary](#63-the-runtime-call-crossing-the-wasm-boundary)
    - [6.4. Classifying the Result](#64-classifying-the-result)
  - [7. FullChainApi: The Bridge Between Pool and Runtime](#7-fullchainapi-the-bridge-between-pool-and-runtime)
    - [7.1. The Worker Pool Architecture](#71-the-worker-pool-architecture)
    - [7.2. validate\_transaction: The Async Dispatch](#72-validate_transaction-the-async-dispatch)
    - [7.3. validate\_transaction\_blocking: The Actual Runtime Call](#73-validate_transaction_blocking-the-actual-runtime-call)
- [Part III: Runtime Validation — Inside the Wasm](#part-iii-runtime-validation--inside-the-wasm)
  - [8. TaggedTransactionQueue: The Runtime API](#8-taggedtransactionqueue-the-runtime-api)
  - [9. Executive::validate\_transaction](#9-executivevalidate_transaction)
  - [10. UncheckedExtrinsic::check — Signature Verification](#10-uncheckedextrinsiccheck--signature-verification)
    - [10.1. The Preamble: Three Extrinsic Types](#101-the-preamble-three-extrinsic-types)
    - [10.2. Signed Transaction Check](#102-signed-transaction-check)
    - [10.3. The SignedPayload and Hashing](#103-the-signedpayload-and-hashing)
  - [11. CheckedExtrinsic::validate — The Extension Pipeline](#11-checkedextrinsicvalidate--the-extension-pipeline)
  - [12. TxExtension: The Concrete Pipeline](#12-txextension-the-concrete-pipeline)
  - [13. ChargeTransactionPayment: Priority Calculation](#13-chargetransactionpayment-priority-calculation)
    - [13.1. The validate Method](#131-the-validate-method)
    - [13.2. get\_priority: The Ordering Formula](#132-get_priority-the-ordering-formula)
- [Part IV: Back to the Client — Pool Insertion](#part-iv-back-to-the-client--pool-insertion)
  - [14. ValidatedPool::submit](#14-validatedpoolsubmit)
    - [14.1. submit\_one: Handling Each Validation Outcome](#141-submit_one-handling-each-validation-outcome)
    - [14.2. enforce\_limits: Keeping the Pool Bounded](#142-enforce_limits-keeping-the-pool-bounded)
  - [15. BasePool::import — Ready vs Future](#15-basepoolimport--ready-vs-future)
    - [15.1. WaitingTransaction: Computing Missing Tags](#151-waitingtransaction-computing-missing-tags)
    - [15.2. The Decision: Ready or Future](#152-the-decision-ready-or-future)
  - [16. import\_to\_ready: The Cascade Effect](#16-import_to_ready-the-cascade-effect)
    - [16.1. satisfy\_tags: Promoting from Future](#161-satisfy_tags-promoting-from-future)
    - [16.2. Cycle Detection](#162-cycle-detection)
  - [17. ReadyTransactions::import — The Final Ordering](#17-readytransactionsimport--the-final-ordering)
    - [17.1. replace\_previous: Priority-Based Replacement](#171-replace_previous-priority-based-replacement)
    - [17.2. Dependency Linking](#172-dependency-linking)
    - [17.3. The Best Set](#173-the-best-set)
    - [17.4. TransactionRef Ordering](#174-transactionref-ordering)
- [Part V: Block Authoring — Consuming the Pool](#part-v-block-authoring--consuming-the-pool)
  - [18. The BestIterator: Ordered Traversal](#18-the-bestiterator-ordered-traversal)
  - [19. The Complete Circuit: Pool to Block](#19-the-complete-circuit-pool-to-block)
- [Part VI: Architecture — Client vs Runtime](#part-vi-architecture--client-vs-runtime)
  - [20. The Two Worlds: Native Binary vs Wasm](#20-the-two-worlds-native-binary-vs-wasm)
  - [21. How the Runtime Gets Injected into the Client](#21-how-the-runtime-gets-injected-into-the-client)
  - [22. The Tag System: Beyond Nonces](#22-the-tag-system-beyond-nonces)
  - [23. Customizing the Pool Ordering](#23-customizing-the-pool-ordering)
- [Complete Flow Summary](#complete-flow-summary)

---

## Introduction

The transaction pool (often called "mempool" in other blockchain ecosystems) is the component responsible for receiving, validating, storing, and ordering transactions before they are included in a block. In the Polkadot SDK, the pool is entirely **client-side** (runs as native compiled code in the node binary), but it calls into the **runtime** (Wasm) for transaction validation.

This document traces the complete lifecycle of a transaction: from the moment a user submits raw bytes via JSON-RPC, through the validation pipeline that crosses the client/runtime boundary, into the pool's dependency graph where transactions are classified as "ready" or "future", and finally to the block author that consumes them in priority order.

**Key files referenced:**
- `substrate/client/rpc/src/author/mod.rs` — RPC entry point
- `substrate/client/transaction-pool/src/graph/pool.rs` — Pool orchestration and `ChainApi` trait
- `substrate/client/transaction-pool/src/common/api.rs` — `FullChainApi` implementation
- `substrate/client/transaction-pool/src/graph/validated_pool.rs` — Validated pool and submission logic
- `substrate/client/transaction-pool/src/graph/base_pool.rs` — Base pool with ready/future queues
- `substrate/client/transaction-pool/src/graph/future.rs` — Future transactions queue
- `substrate/client/transaction-pool/src/graph/ready.rs` — Ready transactions queue and ordering
- `substrate/frame/executive/src/lib.rs` — Executive module, runtime validation entry
- `substrate/primitives/runtime/src/generic/unchecked_extrinsic.rs` — Extrinsic decoding and signature check
- `substrate/primitives/runtime/src/generic/checked_extrinsic.rs` — Checked extrinsic and extension pipeline
- `substrate/frame/transaction-payment/src/lib.rs` — Fee and priority calculation
- `substrate/frame/system/src/lib.rs` — System pallet, nonce checks, transaction extensions

---

# Part I: The RPC Entry Point

## 1. author_submitExtrinsic: The Door to the Node

Every transaction enters the node through the `author_submitExtrinsic` JSON-RPC method. The trait is defined in the RPC layer:

```rust
pub trait AuthorApi<Hash, BlockHash> {
    #[method(name = "author_submitExtrinsic")]
    async fn submit_extrinsic(&self, extrinsic: Bytes) -> Result<Hash, Error>;
}
```

The implementation in the `Author` struct performs three steps:

```rust
async fn submit_extrinsic(&self, ext: Bytes) -> Result<Hash, Error> {
    let xt = Decode::decode(&mut &ext[..])?;       // 1. Decode bytes
    let best_block = self.client.info().best_hash;  // 2. Get validation anchor
    let hash = self.pool
        .submit_one(best_block, TX_SOURCE, xt)      // 3. Submit to pool
        .await?;
    Ok(hash)
}
```

This is entirely **client-side** code — it runs in the native binary, not in Wasm.

## 2. Decoding the Raw Bytes

```rust
let xt = match Decode::decode(&mut &ext[..]) {
    Ok(xt) => xt,
    Err(err) => return Err(Error::Client(Box::new(err)).into()),
};
```

The type that `Decode::decode` produces is inferred from the context. The `pool.submit_one()` method expects a `TransactionFor<Self>`, which resolves through the type chain:

```
TransactionFor<Pool>
  → <<Pool as TransactionPool>::Block as BlockT>::Extrinsic
    → UncheckedExtrinsic<Address, RuntimeCall, Signature, TxExtension>
```

The concrete `UncheckedExtrinsic` type is defined in the runtime's `lib.rs`:

```rust
pub type UncheckedExtrinsic =
    generic::UncheckedExtrinsic<Address, RuntimeCall, Signature, TxExtension>;
```

And its `Decode` implementation reads:
1. A compact-encoded length prefix
2. A version/type byte (bits 6-7 determine Bare/Signed/General)
3. For Signed: address, signature, extension data
4. The call itself (pallet index + call index + arguments)

## 3. Best Block Hash: The Validation Anchor

```rust
let best_block_hash = self.client.info().best_hash;
```

The pool needs to validate transactions **against a specific state**. Checking the nonce, verifying the account balance for fees, and confirming the signature all require reading from a state trie. The `best_hash` identifies which block's state to use.

Later, when the pool calls the runtime's `validate_transaction`, it passes this hash so the runtime knows: "validate this transaction as if we are building on top of this block."

## 4. TransactionSource: Where Did This Come From?

```rust
const TX_SOURCE: TransactionSource = TransactionSource::External;
```

The pool treats transactions differently based on their origin:

| Source | Meaning | Treatment |
|--------|---------|-----------|
| `External` | From RPC or network gossip | Full validation, may be rejected if pool is full |
| `Local` | Generated by the node itself (e.g., heartbeats, equivocation reports) | Preferential treatment, skips some pool limits |
| `InBlock` | Re-imported from a retracted block during a reorg | Trusted (was already validated), skips some checks |

For `author_submitExtrinsic`, the source is always `External` because the node has no reason to trust an external user's transaction.

---

# Part II: The Pool Pipeline — Client Side

## 5. submit_one → submit_at → verify

The call chain from RPC to validation is:

```rust
// submit_one: convenience wrapper for a single transaction
pub async fn submit_one(&self, at, source, xt) -> Result<...> {
    let res = self
        .submit_at(at, std::iter::once((source, xt)), ValidateTransactionPriority::Submitted)
        .await
        .pop();
    res.expect("One extrinsic passed; one result returned; qed")
}

// submit_at: validates and submits a batch
pub async fn submit_at(&self, at, xts, validation_priority) -> Vec<Result<...>> {
    let validated_transactions =
        self.verify(at, xts, CheckBannedBeforeVerify::Yes, validation_priority).await;
    self.validated_pool.submit(validated_transactions.into_values())
}

// verify: validates all transactions in parallel
async fn verify(&self, at, xts, check, validation_priority) -> IndexMap<Hash, ValidatedTx> {
    let HashAndNumber { number, hash } = *at;
    futures::future::join_all(
        xts.into_iter().map(|(source, xt)| {
            self.verify_one(hash, number, source, xt, check, validation_priority)
        })
    ).await
    .into_iter()
    .collect::<IndexMap<_, _>>()
}
```

Note that `verify` uses `futures::future::join_all` — all transactions in a batch are validated **in parallel**.

## 6. verify_one: The Core Validation Orchestrator

**File:** `substrate/client/transaction-pool/src/graph/pool.rs`

This is where the actual validation logic is orchestrated. It performs several steps in sequence:

```rust
pub(crate) async fn verify_one(
    &self,
    block_hash: <B::Block as BlockT>::Hash,
    block_number: NumberFor<B>,
    source: base::TimedTransactionSource,
    xt: ExtrinsicFor<B>,
    check: CheckBannedBeforeVerify,
    validation_priority: ValidateTransactionPriority,
) -> (ExtrinsicHash<B>, ValidatedTransactionFor<B>) {
```

### 6.1. hash_and_length: Identity and Size

```rust
let (hash, bytes) = self.validated_pool.api().hash_and_length(&xt);
```

This computes two things from the raw encoded extrinsic:

- **Hash**: The Blake2b hash of the encoded bytes. This is the unique identifier of the transaction in the pool — the same hash returned to the user via RPC.
- **Length**: The byte size of the encoded transaction. Used later to enforce pool size limits.

The implementation:

```rust
fn hash_and_length(&self, ex: &graph::RawExtrinsicFor<Self>) -> (graph::ExtrinsicHash<Self>, usize) {
    ex.using_encoded(|x| (<traits::HashingFor<Block> as traits::Hash>::hash(x), x.len()))
}
```

No validation happens here — this is purely identification.

### 6.2. check_is_known: Ban and Duplicate Check

```rust
let ignore_banned = matches!(check, CheckBannedBeforeVerify::No);
if let Err(err) = self.validated_pool.check_is_known(&hash, ignore_banned) {
    return (hash, ValidatedTransaction::Invalid(hash, err));
}
```

Before spending resources on runtime validation, the pool performs two cheap checks:

1. **Is it already in the pool?** If the transaction hash already exists in the ready or future queue, there's no point validating again.
2. **Is it banned?** The pool maintains a time-based ban list (rotator). Transactions get banned when they are pruned (included in a block) or removed as invalid. The default ban duration is 30 minutes.

The `CheckBannedBeforeVerify` parameter controls whether to check the ban list:
- `Yes` — Normal submissions from `submit_at`. Check the ban.
- `No` — Resubmissions from `resubmit_at` (e.g., after a reorg). Skip the ban check because we want to give these transactions another chance.

### 6.3. The Runtime Call: Crossing the Wasm Boundary

```rust
let validation_result = self
    .validated_pool
    .api()
    .validate_transaction(
        block_hash,
        source.clone().into(),
        xt.clone(),
        validation_priority,
    )
    .await;
```

This is the **only point** where the pool crosses from client-side to the runtime (Wasm). Everything before and after this call is native code. The `api()` returns a reference to the `FullChainApi` which dispatches the call to a worker thread (see section 7).

### 6.4. Classifying the Result

```rust
let status = match validation_result {
    Ok(status) => status,
    Err(e) => return (hash, ValidatedTransaction::Invalid(hash, e)),
};

let validity = match status {
    Ok(validity) => {
        if validity.provides.is_empty() {
            ValidatedTransaction::Invalid(hash, error::Error::NoTagsProvided.into())
        } else {
            ValidatedTransaction::valid_at(
                block_number.saturated_into::<u64>(),
                hash, source, xt, bytes, validity,
            )
        }
    },
    Err(TransactionValidityError::Invalid(e)) =>
        ValidatedTransaction::Invalid(hash, error::Error::InvalidTransaction(e).into()),
    Err(TransactionValidityError::Unknown(e)) =>
        ValidatedTransaction::Unknown(hash, error::Error::UnknownTransaction(e).into()),
};
```

Three possible outcomes:

| Result | Meaning | Action |
|--------|---------|--------|
| `Valid` | Runtime approved the transaction | Wrapped with metadata, passed to pool insertion |
| `Invalid` | Definitively invalid (bad signature, stale nonce, insufficient balance) | Banned and rejected |
| `Unknown` | Cannot determine validity now (might be valid later) | Rejected but not banned |

A special case: if `validity.provides.is_empty()`, the transaction is rejected even if valid. Every transaction **must** provide at least one tag to participate in the pool's dependency graph.

## 7. FullChainApi: The Bridge Between Pool and Runtime

**File:** `substrate/client/transaction-pool/src/common/api.rs`

### 7.1. The Worker Pool Architecture

```rust
pub struct FullChainApi<Client, Block> {
    client: Arc<Client>,
    validation_pool_normal: mpsc::Sender<Pin<Box<dyn Future<Output = ()> + Send>>>,
    validation_pool_maintained: mpsc::Sender<Pin<Box<dyn Future<Output = ()> + Send>>>,
    metrics: Option<Arc<ApiMetrics>>,
    // stats...
}
```

The `FullChainApi` maintains **two validation channels**:
- `validation_pool_normal` — For newly submitted transactions (lower priority)
- `validation_pool_maintained` — For revalidation during maintenance (higher priority)

During construction, two worker tasks are spawned:

```rust
spawn_validation_pool_task("transaction-pool-task-0", receiver.clone(), receiver_maintained.clone(), ...);
spawn_validation_pool_task("transaction-pool-task-1", receiver, receiver_maintained, ...);
```

Each worker uses `tokio::select!` to prioritize maintained transactions:

```rust
tokio::select! {
    Some(task) = async { receiver_maintained.lock().await.recv().await } => { task }
    Some(task) = async { receiver_normal.lock().await.recv().await } => { task }
}
```

This means revalidation work (triggered by new block imports) takes precedence over new user submissions.

### 7.2. validate_transaction: The Async Dispatch

```rust
async fn validate_transaction(
    &self, at, source, uxt, validation_priority,
) -> Result<TransactionValidity, Self::Error> {
    let (tx, rx) = oneshot::channel();
    let client = self.client.clone();

    let (stats, validation_pool, prefix) =
        if validation_priority == ValidateTransactionPriority::Maintained {
            (self.validate_transaction_maintained_stats.clone(),
             self.validation_pool_maintained.clone(), ...)
        } else {
            (self.validate_transaction_normal_stats.clone(),
             self.validation_pool_normal.clone(), ...)
        };

    validation_pool.send(async move {
        let res = validate_transaction_blocking(&*client, at, source, uxt);
        let _ = tx.send(res);
    }.boxed()).await?;

    rx.await.unwrap_or(Err(Error::RuntimeApi("Validation was canceled".into())))
}
```

The pattern is: create a oneshot channel, package the validation work as a future, send it to the appropriate worker channel, and await the result. The actual validation is **blocking** (it executes Wasm), which is why it's dispatched to a separate thread rather than run on the async executor.

### 7.3. validate_transaction_blocking: The Actual Runtime Call

```rust
fn validate_transaction_blocking<Client, Block>(
    client: &Client, at: Block::Hash, source: TransactionSource,
    uxt: graph::ExtrinsicFor<FullChainApi<Client, Block>>,
) -> error::Result<TransactionValidity> {
    let runtime_api = client.runtime_api();

    let api_version = runtime_api
        .api_version::<dyn TaggedTransactionQueue<Block>>(at)?
        .ok_or_else(|| Error::RuntimeApi(...))?;

    if api_version >= 3 {
        runtime_api.validate_transaction(at, source, (*uxt).clone(), at)
            .map_err(|e| Error::RuntimeApi(e.to_string()))
    } else {
        // Legacy versions require initialize_block first
        // and have different signatures
    }
}
```

This function:
1. Gets the runtime API interface from the client
2. Checks the API version (for backward compatibility with older runtimes)
3. Calls `runtime_api.validate_transaction()`, which crosses into Wasm

The `client.runtime_api()` call returns an object that can execute runtime APIs. Under the hood, it invokes the Wasm executor (Wasmtime) to run the runtime code against the state trie at block `at`.

---

# Part III: Runtime Validation — Inside the Wasm

## 8. TaggedTransactionQueue: The Runtime API

The runtime API is declared in `substrate/primitives/transaction-pool/src/runtime_api.rs` and implemented in every runtime's `lib.rs` inside the `impl_runtime_apis!` macro:

```rust
impl sp_transaction_pool::runtime_api::TaggedTransactionQueue<Block> for Runtime {
    fn validate_transaction(
        source: TransactionSource,
        tx: <Block as BlockT>::Extrinsic,
        block_hash: <Block as BlockT>::Hash,
    ) -> TransactionValidity {
        Executive::validate_transaction(source, tx, block_hash)
    }
}
```

This is the same function that can be called externally via `state_call("TaggedTransactionQueue_validate_transaction", ...)`. The pool uses it internally, but anyone can call it directly to validate a transaction without submitting it to the pool.

## 9. Executive::validate_transaction

**File:** `substrate/frame/executive/src/lib.rs`

```rust
pub fn validate_transaction(
    source: TransactionSource,
    uxt: Block::Extrinsic,
    block_hash: Block::Hash,
) -> TransactionValidity {
    // 1. Initialize state for the next block
    <frame_system::Pallet<System>>::initialize(
        &(frame_system::Pallet::<System>::block_number() + One::one()),
        &block_hash,
        &Default::default(),
    );

    // 2. Re-encode and decode with depth limit (stack overflow protection)
    let encoded = uxt.encode();
    let uxt = <Block::Extrinsic as codec::DecodeLimit>::decode_all_with_depth_limit(
        MAX_EXTRINSIC_DEPTH, &mut &encoded[..],
    ).map_err(|_| InvalidTransaction::Call)?;

    // 3. Check signature and convert to CheckedExtrinsic
    let xt = uxt.check(&Default::default())?;

    // 4. Get dispatch info (weight, class)
    let dispatch_info = xt.get_dispatch_info();

    // 5. Reject mandatory (inherent) transactions
    if dispatch_info.class == DispatchClass::Mandatory {
        return Err(InvalidTransaction::MandatoryValidation.into());
    }

    // 6. Run the extension validation pipeline
    xt.validate::<UnsignedValidator>(source, &dispatch_info, encoded.len())
}
```

Step 1 is important: the state is initialized as if we're in the **next** block, because the transaction will be included in a future block, not the current one.

Step 5 prevents inherent extrinsics (like timestamps) from entering the pool — they can only be placed by the block author.

## 10. UncheckedExtrinsic::check — Signature Verification

**File:** `substrate/primitives/runtime/src/generic/unchecked_extrinsic.rs`

### 10.1. The Preamble: Three Extrinsic Types

Every extrinsic has a `Preamble` that determines its type:

```rust
pub enum Preamble<Address, Signature, Extension> {
    Bare(ExtrinsicVersion),                       // Inherent (no signature, no extension)
    Signed(Address, Signature, Extension),         // Legacy signed tx (v4)
    General(ExtensionVersion, Extension),          // New-style tx (v5, no hardcoded signature)
}
```

The version byte encodes both the format version and the type:

| Bits 6-7 | Meaning | Version |
|----------|---------|---------|
| `00` | Bare | v4 or v5 |
| `10` | Signed | v4 only |
| `01` | General | v5 only |

### 10.2. Signed Transaction Check

For the most common case (Signed transactions):

```rust
fn check(self, lookup: &Lookup) -> Result<Self::Checked, TransactionValidityError> {
    Ok(match self.preamble {
        Preamble::Signed(signed, signature, tx_ext) => {
            // 1. Resolve address to AccountId
            let signed = lookup.lookup(signed)?;

            // 2. Construct the payload that should have been signed
            let raw_payload = SignedPayload::new(
                CallAndMaybeEncoded { encoded: self.encoded_call, call: self.function },
                tx_ext,
            )?;

            // 3. Verify the signature
            if !raw_payload.using_encoded(|payload| signature.verify(payload, &signed)) {
                return Err(InvalidTransaction::BadProof.into());
            }

            // 4. Return CheckedExtrinsic
            let (function, tx_ext, _) = raw_payload.deconstruct();
            CheckedExtrinsic { format: ExtrinsicFormat::Signed(signed, tx_ext), function }
        },
        Preamble::General(extension_version, tx_ext) => CheckedExtrinsic {
            format: ExtrinsicFormat::General(extension_version, tx_ext),
            function: self.function,
        },
        Preamble::Bare(_) => {
            CheckedExtrinsic { format: ExtrinsicFormat::Bare, function: self.function }
        },
    })
}
```

For General transactions (v5), there is no hardcoded signature verification in `check`. Instead, the `VerifySignature` extension handles authorization in the extension pipeline.

### 10.3. The SignedPayload and Hashing

The `SignedPayload` includes the call, the explicit extension data, and the **implicit** extension data (like genesis hash, spec version — data that's part of the signature but not sent on-chain):

```rust
impl<Call, Extension> Encode for SignedPayload<Call, Extension> {
    fn using_encoded<R, F: FnOnce(&[u8]) -> R>(&self, f: F) -> R {
        self.0.using_encoded(|payload| {
            if payload.len() > 256 {
                f(&blake2_256(payload)[..])  // Hash if > 256 bytes
            } else {
                f(payload)                     // Use raw if ≤ 256 bytes
            }
        })
    }
}
```

If the payload exceeds 256 bytes, it's hashed with Blake2b-256 before signature verification. This prevents extremely large payloads from impacting verification performance.

## 11. CheckedExtrinsic::validate — The Extension Pipeline

**File:** `substrate/primitives/runtime/src/generic/checked_extrinsic.rs`

After `check` succeeds, we have a `CheckedExtrinsic` with a resolved `AccountId` and validated signature. Now the extension pipeline runs:

```rust
fn validate<I: ValidateUnsigned<Call = Self::Call>>(
    &self, source: TransactionSource,
    info: &DispatchInfoOf<Self::Call>, len: usize,
) -> TransactionValidity {
    match self.format {
        ExtrinsicFormat::Bare => {
            let inherent_validation = I::validate_unsigned(source, &self.function)?;
            let legacy_validation = Extension::bare_validate(&self.function, info, len)?;
            Ok(legacy_validation.combine_with(inherent_validation))
        },
        ExtrinsicFormat::Signed(ref signer, ref extension) => {
            let origin = Some(signer.clone()).into();
            extension.validate_only(origin, &self.function, info, len, source, DEFAULT_EXTENSION_VERSION)
                .map(|x| x.0)
        },
        ExtrinsicFormat::General(extension_version, ref extension) => {
            extension.validate_only(None.into(), &self.function, info, len, source, extension_version)
                .map(|x| x.0)
        },
    }
}
```

For Signed transactions, the origin starts as `Some(signer)`. For General transactions, it starts as `None` and extensions are responsible for setting it.

The `validate_only` method iterates through the tuple of extensions, calling each one's `validate` in order. Each extension returns a `ValidTransaction` which are combined (priorities are summed, tags are merged).

## 12. TxExtension: The Concrete Pipeline

The extension pipeline is defined as a type alias in the runtime's `lib.rs`. It is **not** an associated type of any `Config` trait — it's a free type alias wired directly into `UncheckedExtrinsic`:

```rust
pub type TxExtension = (
    frame_system::AuthorizeCall<Runtime>,
    frame_system::CheckNonZeroSender<Runtime>,
    frame_system::CheckSpecVersion<Runtime>,
    frame_system::CheckTxVersion<Runtime>,
    frame_system::CheckGenesis<Runtime>,
    frame_system::CheckMortality<Runtime>,
    frame_system::CheckNonce<Runtime>,
    frame_system::CheckWeight<Runtime>,
    pallet_transaction_payment::ChargeTransactionPayment<Runtime>,
    frame_metadata_hash_extension::CheckMetadataHash<Runtime>,
    frame_system::WeightReclaim<Runtime>,
);

pub type UncheckedExtrinsic =
    generic::UncheckedExtrinsic<Address, RuntimeCall, Signature, TxExtension>;
```

Each extension contributes to the `ValidTransaction` result:

| Extension | Contribution |
|-----------|-------------|
| `CheckNonZeroSender` | Rejects zero-address senders |
| `CheckSpecVersion` | Rejects transactions for wrong runtime version |
| `CheckTxVersion` | Rejects transactions for wrong transaction format version |
| `CheckGenesis` | Rejects transactions for wrong chain (wrong genesis hash) |
| `CheckMortality` | Sets `longevity` based on the era (mortal/immortal) |
| `CheckNonce` | Sets `requires` and `provides` tags based on account nonce |
| `CheckWeight` | Rejects if transaction exceeds block weight limits |
| `ChargeTransactionPayment` | Verifies balance for fees, sets `priority` based on tip |

## 13. ChargeTransactionPayment: Priority Calculation

**File:** `substrate/frame/transaction-payment/src/lib.rs`

### 13.1. The validate Method

```rust
fn validate(
    &self, origin, call, info, len, _, _implication, _source,
) -> Result<(ValidTransaction, Self::Val, RuntimeOrigin), TransactionValidityError> {
    let Ok(who) = frame_system::ensure_signed(origin.clone()) else {
        return Ok((ValidTransaction::default(), Val::NoCharge, origin));
    };
    let fee_with_tip = self.can_withdraw_fee(&who, call, info, len)?;
    let tip = self.0;
    Ok((
        ValidTransaction {
            priority: Self::get_priority(info, len, tip, fee_with_tip),
            ..Default::default()
        },
        Val::Charge { tip: self.0, who, fee_with_tip },
        origin,
    ))
}
```

The extension only sets `priority` — it does not touch `requires`, `provides`, or `longevity`. Those are handled by `CheckNonce` and `CheckMortality` respectively.

### 13.2. get_priority: The Ordering Formula

```rust
pub fn get_priority(info, len, tip, final_fee_with_tip) -> TransactionPriority {
    let max_block_weight = T::BlockWeights::get().max_block;
    let max_block_length = *T::BlockLength::get().max.get(info.class) as u64;

    let bounded_weight = info.total_weight().max(Weight::from_parts(1, 1)).min(max_block_weight);
    let bounded_length = (len as u64).clamp(1, max_block_length);

    // How many such transactions could fit in a block?
    let max_tx_per_block_weight = max_block_weight.checked_div_per_component(&bounded_weight)...;
    let max_tx_per_block_length = max_block_length / bounded_length;
    let max_tx_per_block = max_tx_per_block_length.min(max_tx_per_block_weight);

    // tip + 1 (so zero-tip transactions still get differentiated by size)
    let tip = tip.saturating_add(One::one());
    let scaled_tip = tip * max_tx_per_block;

    match info.class {
        DispatchClass::Normal => scaled_tip,
        DispatchClass::Mandatory => scaled_tip,
        DispatchClass::Operational => {
            let virtual_tip = final_fee_with_tip * OperationalFeeMultiplier;
            scaled_tip + (virtual_tip * max_tx_per_block)
        },
    }
}
```

The logic:
1. Calculate how many transactions of this size/weight could fit in a block (the scarcer resource wins)
2. Scale the tip by that factor — smaller transactions with the same tip get higher priority (they use less block space)
3. Add 1 to the tip so zero-tip transactions are ordered by size (smaller = higher priority)
4. For `Operational` transactions (system operations like equivocation reports), add a "virtual tip" equal to `fee × OperationalFeeMultiplier` (typically 5x) to ensure they get priority during congestion

---

# Part IV: Back to the Client — Pool Insertion

## 14. ValidatedPool::submit

**File:** `substrate/client/transaction-pool/src/graph/validated_pool.rs`

After `verify_one` returns, we're back in client-side code. The validated transactions are passed to `validated_pool.submit()`:

```rust
pub fn submit(
    &self,
    txs: impl IntoIterator<Item = ValidatedTransactionFor<B>>,
) -> Vec<Result<ValidatedPoolSubmitOutcome<B>, B::Error>> {
    let results = txs
        .into_iter()
        .map(|validated_tx| self.submit_one(validated_tx))
        .collect::<Vec<_>>();

    // Only enforce limits if at least one transaction was imported
    let removed = if results.iter().any(|res| res.is_ok()) {
        self.enforce_limits()
    } else {
        Default::default()
    };

    // Mark as ImmediatelyDropped if removed by limits
    results.into_iter().map(|res| match res {
        Ok(outcome) if removed.contains(&outcome.hash) => {
            Err(error::Error::ImmediatelyDropped.into())
        },
        other => other,
    }).collect()
}
```

### 14.1. submit_one: Handling Each Validation Outcome

```rust
fn submit_one(&self, tx: ValidatedTransactionFor<B>) -> Result<...> {
    match tx {
        ValidatedTransaction::Valid(tx) => {
            // Non-propagable transactions on non-validator nodes are rejected
            if !tx.propagate && !(self.is_validator.0)() {
                return Err(error::Error::Unactionable.into());
            }

            // Insert into base pool (this decides ready vs future)
            let imported = self.pool.write().import(tx)?;

            // If it went to ready, notify import notification sinks
            if let base::Imported::Ready { ref hash, .. } = imported {
                // Send hash to all listening sinks (e.g., block author)
            }

            // Fire events (ready, future, usurped, etc.)
            fire_events(&mut *event_dispatcher, &imported);
            Ok(ValidatedPoolSubmitOutcome::new(*imported.hash(), Some(priority)))
        },
        ValidatedTransaction::Invalid(tx_hash, error) => {
            self.rotator.ban(&Instant::now(), std::iter::once(tx_hash));
            Err(error)
        },
        ValidatedTransaction::Unknown(tx_hash, error) => {
            self.event_dispatcher.write().invalid(&tx_hash);
            Err(error)
        },
    }
}
```

### 14.2. enforce_limits: Keeping the Pool Bounded

After successful imports, the pool checks if it exceeds its configured limits:

```rust
fn enforce_limits(&self) -> HashSet<ExtrinsicHash<B>> {
    let status = self.pool.read().status();
    let ready_limit = &self.options.ready;    // default: 8192 count, 20 MB
    let future_limit = &self.options.future;  // default: 512 count, 1 MB

    if ready_limit.is_exceeded(status.ready, status.ready_bytes) ||
       future_limit.is_exceeded(status.future, status.future_bytes)
    {
        let removed = pool.enforce_limits(ready_limit, future_limit);
        self.rotator.ban(&Instant::now(), removed.iter().copied());
        removed
    } else {
        Default::default()
    }
}
```

The default pool limits from `Options::default()`:

```rust
Self {
    ready: base::Limit { count: 8192, total_bytes: 20 * 1024 * 1024 },
    future: base::Limit { count: 512, total_bytes: 1 * 1024 * 1024 },
    reject_future_transactions: false,
    ban_time: Duration::from_secs(60 * 30),  // 30 minutes
}
```

When limits are exceeded, the pool removes the **lowest priority** transactions first (for ready) or the **oldest** transactions first (for future), bans them, and fires drop notifications.

## 15. BasePool::import — Ready vs Future

**File:** `substrate/client/transaction-pool/src/graph/base_pool.rs`

This is where the fundamental decision happens: does the transaction go to the ready queue or the future queue?

### 15.1. WaitingTransaction: Computing Missing Tags

```rust
pub fn import(&mut self, tx: Transaction<Hash, Ex>) -> error::Result<Imported<Hash, Ex>> {
    if self.is_imported(&tx.hash) {
        return Err(error::Error::AlreadyImported(Box::new(tx.hash)));
    }

    let tx = WaitingTransaction::new(tx, self.ready.provided_tags(), &self.recently_pruned);
```

`WaitingTransaction::new` computes the `missing_tags` — it takes the transaction's `requires` list and checks each tag against:
- Tags provided by transactions already in the ready pool (`provided_tags`)
- Tags from recently pruned transactions (`recently_pruned`)

```rust
impl<Hash, Ex> WaitingTransaction<Hash, Ex> {
    pub fn new(
        transaction: Transaction<Hash, Ex>,
        provided: &HashMap<Tag, Hash>,
        recently_pruned: &[HashSet<Tag>],
    ) -> Self {
        let missing_tags = transaction.requires.iter()
            .filter(|tag| {
                let is_provided = provided.contains_key(&**tag) ||
                    recently_pruned.iter().any(|x| x.contains(&**tag));
                !is_provided
            })
            .cloned()
            .collect();

        Self { transaction: Arc::new(transaction), missing_tags, imported_at: Instant::now() }
    }
}
```

### 15.2. The Decision: Ready or Future

```rust
    if !tx.is_ready() {
        if self.reject_future_transactions {
            return Err(error::Error::RejectedFutureTransaction);
        }
        let hash = tx.transaction.hash.clone();
        self.future.import(tx);
        return Ok(Imported::Future { hash });
    }

    self.import_to_ready(tx)
}
```

- If `missing_tags` is empty → `is_ready() == true` → goes to **ready**
- If `missing_tags` is not empty → `is_ready() == false` → goes to **future**

**Concrete example with nonces:**

Account with on-chain nonce = 5:
- Tx nonce 5: `requires: []` (on-chain state covers it) → missing_tags empty → **ready**
- Tx nonce 7: `requires: [tag(nonce:6)]` → no one provides tag(nonce:6) → missing_tags = {tag(nonce:6)} → **future**
- Tx nonce 6 arrives: `requires: [tag(nonce:5)]` (covered) → **ready**, and it `provides: [tag(nonce:6)]` which unlocks tx nonce 7

## 16. import_to_ready: The Cascade Effect

```rust
fn import_to_ready(&mut self, tx: WaitingTransaction<Hash, Ex>) -> error::Result<Imported<Hash, Ex>> {
    let tx_hash = tx.transaction.hash.clone();
    let mut promoted = vec![];
    let mut failed = vec![];
    let mut removed = vec![];

    let mut first = true;
    let mut to_import = vec![tx];

    while let Some(tx) = to_import.pop() {
        // Find transactions in Future that this one unlocks
        to_import.append(&mut self.future.satisfy_tags(&tx.transaction.provides));

        // Import into ready
        match self.ready.import(tx) {
            Ok(mut replaced) => {
                if !first { promoted.push(current_hash); }
                removed.append(&mut replaced);
            },
            Err(error::Error::TooLowPriority { .. }) => { ... },
            Err(error) => { ... },
        }
        first = false;
    }

    Ok(Imported::Ready { hash: tx_hash, promoted, failed, removed })
}
```

The key insight is the **cascade**: when a transaction enters the ready pool and provides new tags, it may unlock transactions in the future pool. Those unlocked transactions are then also imported to ready, potentially unlocking even more. The `while let Some(tx) = to_import.pop()` loop continues until no more transactions can be promoted.

### 16.1. satisfy_tags: Promoting from Future

**File:** `substrate/client/transaction-pool/src/graph/future.rs`

```rust
pub fn satisfy_tags<T: AsRef<Tag>>(
    &mut self,
    tags: impl IntoIterator<Item = T>,
) -> Vec<WaitingTransaction<Hash, Ex>> {
    let mut became_ready = vec![];

    for tag in tags {
        if let Some(hashes) = self.wanted_tags.remove(tag.as_ref()) {
            for hash in hashes {
                let is_ready = {
                    let tx = self.waiting.get_mut(&hash).expect(WAITING_PROOF);
                    tx.satisfy_tag(tag.as_ref());  // Remove from missing_tags
                    tx.is_ready()                    // Check if all missing_tags are empty
                };

                if is_ready {
                    let tx = self.waiting.remove(&hash).expect(WAITING_PROOF);
                    became_ready.push(tx);
                }
            }
        }
    }
    became_ready
}
```

For each provided tag:
1. Look up which future transactions were waiting for it (`wanted_tags`)
2. Remove that tag from their `missing_tags`
3. If a transaction has no more missing tags → it's ready, remove from future and return it

A transaction that requires multiple tags may have one tag satisfied but still remain in future until all tags are satisfied.

### 16.2. Cycle Detection

At the end of `import_to_ready`, there's a cycle detection check:

```rust
if removed.iter().any(|tx| tx.hash == tx_hash) {
    self.ready.remove_subtree(&promoted);
    return Err(error::Error::CycleDetected);
}
```

If the original transaction was removed by a transaction it itself promoted (circular dependency), the entire promoted subgraph is removed and a `CycleDetected` error is returned.

## 17. ReadyTransactions::import — The Final Ordering

**File:** `substrate/client/transaction-pool/src/graph/ready.rs`

```rust
pub fn import(
    &mut self,
    tx: WaitingTransaction<Hash, Ex>,
) -> error::Result<Vec<Arc<Transaction<Hash, Ex>>>> {
```

### 17.1. replace_previous: Priority-Based Replacement

```rust
let (replaced, unlocks) = self.replace_previous(&transaction)?;
```

If the new transaction provides the same tags as existing transactions, their combined priorities are compared. If the new transaction's priority is higher, the old ones are removed (similar to Ethereum's replace-by-fee). If lower, the import fails with `TooLowPriority`.

```rust
fn replace_previous(&mut self, tx) -> error::Result<(Vec<...>, Vec<Hash>)> {
    let replace_hashes = tx.provides.iter()
        .filter_map(|tag| self.provided_tags.get(tag))
        .collect::<HashSet<_>>();

    if replace_hashes.is_empty() { return Ok((vec![], vec![])); }

    let old_priority = replace_hashes.iter()
        .filter_map(|hash| ready.get(hash))
        .fold(0u64, |total, tx| total.saturating_add(tx.transaction.transaction.priority));

    if old_priority >= tx.priority {
        return Err(error::Error::TooLowPriority { old: old_priority, new: tx.priority });
    }

    // Remove old transactions and return them
    Ok((removed, unlocks))
}
```

### 17.2. Dependency Linking

```rust
let mut goes_to_best = true;
for tag in &transaction.requires {
    if let Some(other) = self.provided_tags.get(tag) {
        let tx = ready.get_mut(other).expect(HASH_READY);
        tx.unlocks.push(hash.clone());
        goes_to_best = false;  // Depends on another tx, not a root
    }
}

for tag in &transaction.provides {
    self.provided_tags.insert(tag.clone(), hash.clone());
}
```

For each required tag, the transaction registers itself in the `unlocks` list of the providing transaction. This builds the dependency graph. If the transaction depends on any other transaction in the pool, it doesn't go to `best`.

### 17.3. The Best Set

```rust
if goes_to_best {
    self.best.insert(transaction.clone());
}
```

`best` is a `BTreeSet<TransactionRef>` containing transactions that have **no dependencies** on other pool transactions. These are the "roots" of the dependency graph — the first candidates for block inclusion.

### 17.4. TransactionRef Ordering

```rust
impl<Hash, Ex> Ord for TransactionRef<Hash, Ex> {
    fn cmp(&self, other: &Self) -> cmp::Ordering {
        self.transaction.priority.cmp(&other.transaction.priority)          // Higher priority first
            .then_with(|| other.transaction.valid_till.cmp(&self.transaction.valid_till))  // Lower longevity first
            .then_with(|| other.insertion_id.cmp(&self.insertion_id))       // Older first
    }
}
```

The ordering criteria:
1. **Higher priority first** — Transactions with higher tips/fees come first
2. **Lower longevity first** — Among equal priority, transactions that expire sooner get priority (they'll miss their window otherwise)
3. **Older first** — Among equal everything, transactions that have been waiting longer go first (fairness)

---

# Part V: Block Authoring — Consuming the Pool

## 18. The BestIterator: Ordered Traversal

When a block author needs transactions, it calls `pool.ready()` which returns a `BestIterator`:

```rust
pub fn get(&self) -> BestIterator<Hash, Ex> {
    BestIterator {
        all: self.ready.clone_map(),
        best: self.best.clone(),
        awaiting: Default::default(),
        invalid: Default::default(),
    }
}
```

The iterator starts from the `best` set (root transactions) and expands as it consumes:

```rust
impl<Hash, Ex> Iterator for BestIterator<Hash, Ex> {
    fn next(&mut self) -> Option<Self::Item> {
        loop {
            let best = self.best.iter().next_back()?.clone();  // Highest priority
            let best = self.best.take(&best)?;

            if self.invalid.contains(&best.transaction.hash) { continue; }

            let ready = self.all.get(&best.transaction.hash)?;

            // Unlock dependent transactions
            for hash in &ready.unlocks {
                // Check if all requirements are now satisfied
                // If so, add to best set
            }

            return Some(best.transaction);
        }
    }
}
```

The `best` set grows dynamically: as transactions are consumed and their `provides` tags are "spent", dependent transactions whose requirements are now fully satisfied get promoted into `best`. This ensures the iterator always respects the dependency graph while returning transactions in priority order.

## 19. The Complete Circuit: Pool to Block

In the block proposer (`substrate/client/basic-authorship/src/basic_authorship.rs`), the `apply_extrinsics` method consumes from the pool:

```
transaction_pool.ready()       ← Returns BestIterator
    ↓
loop {
    tx = iterator.next()       ← Takes highest-priority ready tx
    block_builder.push(tx)     ← Calls runtime's apply_extrinsic
    if deadline || block full → break
}
```

The loop stops when:
- No more ready transactions in the pool
- The authoring deadline is reached
- The block is full (weight or size limit)

---

# Part VI: Architecture — Client vs Runtime

## 20. The Two Worlds: Native Binary vs Wasm

The Polkadot SDK node has two execution environments:

**Client side (native binary):**
- Networking (libp2p, gossip)
- Database (RocksDB)
- RPC server (JSON-RPC)
- Transaction pool (all of it: `Pool`, `ValidatedPool`, `BasePool`, `ReadyTransactions`, `FutureTransactions`)
- Consensus (BABE, GRANDPA)
- `FullChainApi`

**Runtime side (Wasm):**
- All pallets (balances, system, staking, etc.)
- `Executive::validate_transaction`
- The `TxExtension` pipeline
- All state transition logic

The pool is entirely client-side. The only point where it crosses into the runtime is `client.runtime_api().validate_transaction()`.

## 21. How the Runtime Gets Injected into the Client

In the node's `service.rs`:

```rust
// 1. Create client with RuntimeApi type
let (client, backend, keystore_container, task_manager) =
    sc_service::new_full_parts::<Block, RuntimeApi, _>(
        config,
        telemetry.as_ref().map(|(_, telemetry)| telemetry.handle()),
        executor,
        Default::default(),
    )?;
let client = Arc::new(client);

// 2. Create transaction pool with that client
let transaction_pool = Arc::from(
    sc_transaction_pool::Builder::new(
        task_manager.spawn_essential_handle(),
        client.clone(),  // ← client injected here
        config.role.is_authority().into(),
    )
    .with_options(config.transaction_pool.clone())
    .with_prometheus(config.prometheus_registry())
    .build(),
);
```

`RuntimeApi` is a **type** (not a value) generated by `impl_runtime_apis!`. It tells the compiler which runtime APIs are available. The actual Wasm code comes embedded in the binary via:

```rust
#[cfg(feature = "std")]
include!(concat!(env!("OUT_DIR"), "/wasm_binary.rs"));
```

This generates a `WASM_BINARY` constant containing the compiled runtime bytes. These bytes are stored in the chain spec (genesis config) and in the on-chain storage under the `:code` key. Since the Wasm lives in storage, it can be **upgraded on-chain** without changing the node binary — that's the power of forkless upgrades.

## 22. The Tag System: Beyond Nonces

The pool's dependency resolution uses **opaque tags** (`Vec<u8>`). The pool doesn't know what they represent — it only knows that transactions `require` and `provide` them.

In practice, different extensions and pallets use tags for different purposes:

**CheckNonce (most common):**
```
requires: [(account_id, nonce - 1)]    // needs previous nonce
provides: [(account_id, nonce)]         // provides this nonce
```

**ValidateUnsigned (e.g., im-online heartbeats):**
```
requires: []
provides: [(unique_heartbeat_tag)]      // prevents duplicate heartbeats
```

**Custom pallets:**
Any pallet implementing `ValidateUnsigned` or a custom `TransactionExtension` can define its own tag scheme. This makes the system flexible enough for account-based (Ethereum-style) or UTXO-based (Bitcoin-style) models.

## 23. Customizing the Pool Ordering

There are two approaches:

**Approach 1: Adjust priority from the runtime (recommended)**

Create a custom `TransactionExtension` that calculates priority with your own logic. Since priorities from all extensions are summed, your extension adds to (or dominates) the default `ChargeTransactionPayment` priority. This requires no client-side changes.

**Approach 2: Implement a custom pool**

The `TransactionPool` trait in `substrate/client/api/src/transaction_pool.rs` defines the interface the node uses. You can implement your own pool with entirely different ordering, dependency resolution, or eviction logic. Wire it in `service.rs` instead of `sc_transaction_pool::FullPool`. This is what some projects do for MEV or UTXO models.

---

# Complete Flow Summary

```
1. User sends bytes via RPC
   → author_submitExtrinsic(bytes)                          [CLIENT]

2. Decode
   → Bytes → UncheckedExtrinsic                             [CLIENT]

3. Pool.submit_one()
   → submit_at() → verify() → verify_one()                  [CLIENT]

4. verify_one()
   a. hash_and_length() → compute hash + size               [CLIENT]
   b. check_is_known() → already in pool? banned?           [CLIENT]
   c. FullChainApi.validate_transaction()
      → worker pool dispatch                                 [CLIENT]
      → validate_transaction_blocking()                      [CLIENT]
        → client.runtime_api().validate_transaction()        [→ WASM]

5. Executive::validate_transaction()                         [RUNTIME]
   a. initialize() → simulate next block state
   b. uxt.check() → verify signature, resolve AccountId
   c. xt.validate() → TxExtension pipeline:
      - AuthorizeCall
      - CheckNonZeroSender
      - CheckSpecVersion / CheckTxVersion / CheckGenesis
      - CheckMortality → sets longevity
      - CheckNonce → sets requires/provides tags
      - CheckWeight → verifies weight
      - ChargeTransactionPayment → verifies balance, sets priority
      - CheckMetadataHash
      - WeightReclaim
   d. Returns ValidTransaction { priority, requires, provides, longevity, propagate }

6. Back to client → verify_one classifies:                   [CLIENT]
   - Ok(validity) → ValidatedTransaction::Valid
   - Err(Invalid) → ValidatedTransaction::Invalid
   - Err(Unknown) → ValidatedTransaction::Unknown

7. validated_pool.submit()                                   [CLIENT]
   → submit_one() per tx:
     - Valid → pool.write().import(tx)
     - Invalid → ban + error
     - Unknown → notify error

8. BasePool.import()                                         [CLIENT]
   → WaitingTransaction::new() computes missing_tags
   → missing_tags empty → import_to_ready()
   → missing_tags not empty → future.import()

9. import_to_ready()                                         [CLIENT]
   → ready.import(tx) → insert into ready pool
   → future.satisfy_tags() → find unlocked txs
   → cascade: unlocked txs also go to ready
   → replace_previous() → higher priority replaces

10. ReadyTransactions.import()                               [CLIENT]
    → link dependencies (unlocks)
    → register provided_tags
    → if no dependencies → insert into BTreeSet "best"
    → ordering: priority > longevity > age

11. enforce_limits()                                         [CLIENT]
    → if pool exceeds limits (8192 ready / 512 future)
    → remove lowest priority, ban them

12. Block author calls pool.ready()                          [CLIENT]
    → BestIterator traverses best set in priority order
    → respects dependency graph
    → block_builder.push(tx) for each
    → until block full or pool empty
```