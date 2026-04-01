# Complete Block Lifecycle in a Substrate Parachain

## Table of Contents

- [Complete Block Lifecycle in a Substrate Parachain](#complete-block-lifecycle-in-a-substrate-parachain)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
- [Part I: Initial Context](#part-i-initial-context)
  - [1. Chain Spec Generation](#1-chain-spec-generation)
  - [2. Initial Verification in Dev Mode](#2-initial-verification-in-dev-mode)
  - [3. Fundamental Concepts: Extrinsics](#3-fundamental-concepts-extrinsics)
  - [4. Dev vs Production: Key Differences](#4-dev-vs-production-key-differences)
- [Part II: The Production Flow — From Collator to Relay Chain](#part-ii-the-production-flow--from-collator-to-relay-chain)
  - [5. The Collator Entry Point](#5-the-collator-entry-point)
  - [6. Block Builder Task: The Main Loop](#6-block-builder-task-the-main-loop)
    - [Why the Parachain Needs Relay Chain Data (Step 8)](#why-the-parachain-needs-relay-chain-data-step-8)
    - [The Inherent Data (Step 9)](#the-inherent-data-step-9)
  - [7. Block Construction: build\_block\_and\_import](#7-block-construction-build_block_and_import)
  - [8. The Proposer: propose\_block and propose\_with](#8-the-proposer-propose_block-and-propose_with)
  - [9. initialize\_block: Runtime Preparation](#9-initialize_block-runtime-preparation)
    - [Block Number Is Set Before Anything Else](#block-number-is-set-before-anything-else)
    - [ExtrinsicInclusionMode](#extrinsicinclusionmode)
  - [10. Applying Inherents: apply\_inherents](#10-applying-inherents-apply_inherents)
    - [10.1. What Are Inherents and Why Do They Exist](#101-what-are-inherents-and-why-do-they-exist)
    - [10.2. The InherentData Mechanism](#102-the-inherentdata-mechanism)
    - [10.3. create\_inherents: From Raw Data to Extrinsics](#103-create_inherents-from-raw-data-to-extrinsics)
    - [10.4. Deep Dive: pallet-timestamp create\_inherent](#104-deep-dive-pallet-timestamp-create_inherent)
    - [10.5. Deep Dive: Timestamp::set Execution](#105-deep-dive-timestampset-execution)
    - [10.6. OnTimestampSet Subscribers](#106-ontimestampset-subscribers)
    - [10.7. Accessing the Timestamp from Other Pallets](#107-accessing-the-timestamp-from-other-pallets)
    - [10.8. Real-World Inherent Use Cases](#108-real-world-inherent-use-cases)
    - [10.9. The apply\_inherents Function](#109-the-apply_inherents-function)
  - [11. Applying Transactions: apply\_extrinsics](#11-applying-transactions-apply_extrinsics)
    - [Soft Deadline vs Hard Deadline](#soft-deadline-vs-hard-deadline)
  - [12. The Transaction Pool and Prioritization](#12-the-transaction-pool-and-prioritization)
    - [How Priority Is Determined](#how-priority-is-determined)
    - [Relevant Files](#relevant-files)
  - [13. BlockBuilder::push — Crossing into the Runtime](#13-blockbuilderpush--crossing-into-the-runtime)
  - [14. Executive::do\_apply\_extrinsic](#14-executivedo_apply_extrinsic)
  - [15. Signature Verification: check](#15-signature-verification-check)
  - [16. Transaction Extensions Pipeline](#16-transaction-extensions-pipeline)
    - [Where They Are Defined](#where-they-are-defined)
    - [Individual Extension Implementations](#individual-extension-implementations)
    - [Relationship Between Signature and Extensions](#relationship-between-signature-and-extensions)
    - [Tuple Implementation](#tuple-implementation)
  - [17. dispatch\_transaction: Validate, Execute, Post-Dispatch](#17-dispatch_transaction-validate-execute-post-dispatch)
    - [Step 1: validate\_and\_prepare](#step-1-validate_and_prepare)
    - [Step 2: call.dispatch(origin)](#step-2-calldispatchorigin)
    - [Step 3: T::post\_dispatch](#step-3-tpost_dispatch)
    - [The Three Branches of apply by Format](#the-three-branches-of-apply-by-format)
  - [18. The Dispatch to the Pallet](#18-the-dispatch-to-the-pallet)
    - [Two-Level Routing](#two-level-routing)
  - [19. Block Finalization: build and finalize\_block](#19-block-finalization-build-and-finalize_block)
  - [20. frame\_system::Pallet::finalize](#20-frame_systempalletfinalize)
  - [21. Storage Root Calculation](#21-storage-root-calculation)
    - [The Host Function](#the-host-function)
    - [OverlayedChanges::storage\_root](#overlayedchangesstorage_root)
  - [22. full\_storage\_root: Hierarchical Trie](#22-full_storage_root-hierarchical-trie)
  - [23. Sealing, Local Import, and Submission to the Relay Chain](#23-sealing-local-import-and-submission-to-the-relay-chain)
  - [24. Collation Task: The Final Step of the Collator](#24-collation-task-the-final-step-of-the-collator)
    - [run\_collation\_task](#run_collation_task)
    - [handle\_collation\_message](#handle_collation_message)
    - [What Happens After SubmitCollation](#what-happens-after-submitcollation)
    - [Connection to the Block Builder Task](#connection-to-the-block-builder-task)
  - [25. Persistence to RocksDB: execute\_and\_import\_block](#25-persistence-to-rocksdb-execute_and_import_block)
  - [26. try\_commit\_operation: The Atomic Write](#26-try_commit_operation-the-atomic-write)
    - [commit\_operation](#commit_operation)
    - [Inside try\_commit\_operation](#inside-try_commit_operation)
    - [Database Columns](#database-columns)
  - [27. The Trie and Pruning](#27-the-trie-and-pruning)
    - [Trie Nodes Are Not Overwritten](#trie-nodes-are-not-overwritten)
    - [Pruning](#pruning)
  - [28. Complete Production Cycle Summary](#28-complete-production-cycle-summary)
- [Part III: Supplementary Concepts](#part-iii-supplementary-concepts)
  - [29. Weight System: The 2D Model](#29-weight-system-the-2d-model)
    - [Why Two Dimensions](#why-two-dimensions)
    - [Weight Structure](#weight-structure)
    - [How Weights Are Determined](#how-weights-are-determined)
    - [CheckWeight Extension](#checkweight-extension)
    - [Block Limits Configuration](#block-limits-configuration)
    - [Weight in Parachains: The PoV Constraint](#weight-in-parachains-the-pov-constraint)
  - [30. Events System](#30-events-system)
    - [What Are Events](#what-are-events)
    - [Depositing Events](#depositing-events)
    - [Event Storage](#event-storage)
    - [The Event Lifecycle](#the-event-lifecycle)
    - [System Events](#system-events)
    - [Querying Events](#querying-events)
  - [31. Header Structure and Digest](#31-header-structure-and-digest)
    - [The Header Fields](#the-header-fields)
    - [Digest Items](#digest-items)
    - [PreRuntime Digest: Aura Slot](#preruntime-digest-aura-slot)
    - [Seal Digest: Block Signature](#seal-digest-block-signature)
    - [Consensus Digest](#consensus-digest)
    - [How Digests Flow Through Block Production](#how-digests-flow-through-block-production)

---

## Introduction

This document traces the complete lifecycle of a block in a Substrate parachain running in **production mode**, from the moment the collator detects it is its turn to produce, through runtime execution and storage root calculation, all the way to persistence in RocksDB and submission as a collation to the relay chain.

The analysis begins with a brief initial context (chain spec, dev mode verification, extrinsic types), then follows the production code path end to end. All concepts — inherents, the timestamp flow, the InherentData mechanism — are explained inline at the exact point in the production flow where they are encountered.

---

# Part I: Initial Context

---

## 1. Chain Spec Generation

The first step to launch a parachain is generating the **chain spec**, the JSON configuration file that defines the blockchain's identity.

The command used:

```bash
chain-spec-builder create -t development \
  --relay-chain paseo \
  --para-id 1000 \
  --runtime ./target/release/wbuild/parachain-template-runtime/parachain_template_runtime.compact.compressed.wasm \
  named-preset development
```

- **`chain-spec-builder create`**: Creates a new chain spec from scratch.
- **`-t development`**: Chain type `development` — single node, no peers needed, ideal for local development.
- **`--relay-chain paseo`**: The relay chain this parachain will connect to is **Paseo** (the testnet that replaced Rococo).
- **`--para-id 1000`**: The assigned parachain ID.
- **`--runtime ...`**: The compiled runtime in compressed WASM format. This blob defines all the parachain's logic (pallets, storage, extrinsics).
- **`named-preset development`**: Uses a **genesis preset** called `development` hardcoded in the runtime (typically defined in `genesis_config_presets`). It configures development accounts (Alice, Bob, etc.), sudo, initial balances, and other genesis state without manual specification.

The result is a JSON file (chain spec) used with `--chain` when launching the collator node.

---

## 2. Initial Verification in Dev Mode

To verify the parachain works correctly, it is initially launched in dev mode:

```bash
polkadot-omni-node --chain ./chain_spec.json --dev
```

In dev mode, the node simulates a standalone collator without needing a real relay chain. Relay chain data is mocked. The startup log confirms:

- **Role: AUTHORITY**: Acting as a collator with block production permissions.
- **Parachain id: 1000**: Launched with the configured para-id.
- **`extrinsics_count: 2`**: Two standard inherents per block (timestamp + parachain system validation data).
- **Blocks produced every ~3 seconds**: With instant finalization.

This verification confirms that the runtime, chain spec, and inherents work correctly before proceeding to the real production flow.

---

## 3. Fundamental Concepts: Extrinsics

In everyday usage, "extrinsic" and "transaction" are used almost interchangeably, but technically **extrinsic** is the broader concept. It is any data that comes from **outside** the runtime and is included in a block.

There are three types:

- **Signed extrinsics**: Classic transactions. Signed by an account, they pay fees. For example: `balances.transfer`, `contracts.call`. This is what most people understand as a "transaction."
- **Unsigned extrinsics**: No signature, no fees, but they pass through the transaction pool. Used for specific things like offchain worker reports or validator heartbeats. The runtime must explicitly validate them with `ValidateUnsigned`.
- **Inherents**: Do not pass through the transaction pool, have no signature, and are inserted directly by the block-producing node. Their purpose and mechanics are explained in depth in [Section 10](#10-applying-inherents-apply_inherents), at the point in the production flow where they are actually applied.

In practice, 99% of the time when someone says "extrinsic" they mean a signed transaction. Substrate uses "extrinsic" as a generic term encompassing everything that enters a block from outside the state.

---

## 4. Dev vs Production: Key Differences

| Aspect | Dev Mode | Production |
|--------|----------|------------|
| Trigger | Local timer | Relay chain blocks |
| Slot derivation | Local clock | Derived from relay chain slot |
| Relay chain data | Mocked | Real validation data, XCMP/DMP messages |
| Block destination | Imported locally only | Sent as collation to relay chain validators |
| Finalization | Instant | Depends on GRANDPA in the relay chain |
| Peers | 0 | Connected to relay chain and other collators |

The runtime doesn't know the difference. It receives data and processes it the same way. From here on, the document follows exclusively the **production flow**.

---

# Part II: The Production Flow — From Collator to Relay Chain

---

## 5. The Collator Entry Point

**File**: `cumulus/client/consensus/aura/src/collators/slot_based/mod.rs`

```rust
pub async fn run<Block, P, R, CBlock, HB, SC, GAP>(
    params: Params<P, R, CBlock, HB, SC, GAP>,
) {
    let block_builder = block_builder_task::run(...);
    let collation = collation_task::run(...);
    futures::future::join(block_builder, collation).await;
}
```

Two tasks running in parallel, connected by an **async Rust channel** (`tokio::sync::mpsc`):

- **`block_builder_task`**: Waits for slots, constructs parachain blocks.
- **`collation_task`**: Receives constructed blocks through the channel and sends them to relay chain validators.

The channel with low capacity serves as backpressure: if the collation task is busy sending one block, the block builder waits before producing another. Both tasks run as concurrent futures with `futures::future::join` inside the same async runtime (tokio) — they are not separate threads.

---

## 6. Block Builder Task: The Main Loop

**File**: `cumulus/client/consensus/aura/src/collators/slot_based/block_builder_task.rs`

```rust
loop {
    // 1. Wait for the next slot
    if slot_timer.wait_until_next_slot().await.is_err() {
        return;
    };

    // 2. Get the best relay chain block
    let Ok(relay_best_hash) = relay_client.best_block_hash().await else {
        continue;
    };

    // 3. Find the correct relay parent with offset
    let Ok(Some(rp_data)) = offset_relay_parent_find_descendants(
        &mut relay_chain_data_cache, relay_best_hash, relay_parent_offset,
    ).await else { continue; };

    // 4. Calculate the parachain slot
    let Some(para_slot) = adjust_para_to_relay_parent_slot(
        rp_data.relay_parent(), relay_chain_slot_duration, para_slot_duration,
    ) else { continue; };

    // 5. Find the best parachain parent
    let Some(parent_search_result) =
        crate::collators::find_parent(relay_parent, para_id, &*para_backend, &relay_client)
            .await
    else { continue; };

    // 6. Determine the assigned core
    let core = match determine_core(
        &mut relay_chain_data_cache, &relay_parent_header,
        para_id, parent_header, relay_parent_offset,
    ).await {
        Ok(Some(cores)) => cores,
        Ok(None) => { continue; },     // No core scheduled
        Err(()) => { continue; },
    };

    // 7. Check if it's our turn to produce in this slot
    let slot_claim = match crate::collators::can_build_upon::<_, _, P>(
        para_slot.slot, relay_slot, para_slot.timestamp,
        parent_hash, included_header_hash, &*para_client, &keystore,
    ).await {
        Some(slot) => slot,
        None => { continue; },         // Not our turn
    };

    // 8. Assemble the validation data
    let validation_data = PersistedValidationData {
        parent_head: parent_header.encode().into(),
        relay_parent_number: *relay_parent_header.number(),
        relay_parent_storage_root: *relay_parent_header.state_root(),
        max_pov_size: *max_pov_size,
    };

    // 9. Create the inherent data
    let (parachain_inherent_data, other_inherent_data) = collator
        .create_inherent_data_with_rp_offset(
            relay_parent, &validation_data, parent_hash,
            slot_claim.timestamp(), Some(rp_data), relay_proof_request, collator_peer_id,
        ).await;

    // 10. Build the block
    let Ok(Some(candidate)) = collator.build_block_and_import(BuildBlockAndImportParams {
        parent_header: &parent_header,
        slot_claim: &slot_claim,
        parachain_inherent_data,
        extra_inherent_data: other_inherent_data,
        proposal_duration: adjusted_authoring_duration,
        max_pov_size: allowed_pov_size,
        ...
    }).await else { continue; };

    // 11. Announce the block to peers
    collator.collator_service().announce_block(new_block_hash, None);

    // 12. Send to the collation task through the channel
    collator_sender.unbounded_send(CollatorMessage {
        relay_parent,
        parent_header: parent_header.clone(),
        parachain_candidate: candidate.into(),
        validation_code_hash,
        core_index: core.core_index(),
        max_pov_size: validation_data.max_pov_size,
    });
}
```

### Why the Parachain Needs Relay Chain Data (Step 8)

The parachain is not an independent chain — it depends on the relay for its security and communication:

- **Verify it's building on the correct state**: `relay_parent_storage_root` lets the runtime verify things against the relay chain state via merkle proofs.
- **Receive messages from other parachains and the relay**: XCMP messages (between parachains) and DMP messages (relay to parachain) come in the relay data.
- **Know the block size limit**: `max_pov_size` is defined by the relay chain. The parachain cannot produce a block larger than this.
- **Validate the parentage chain**: `parent_head` and `relay_parent_number` allow verifying the correct sequence.

### The Inherent Data (Step 9)

Two groups of raw data are generated at this point:

- **`parachain_inherent_data`**: Parachain-specific data — relay chain state proof, DMP/XCMP messages, relay parent header. Will become the `set_validation_data` inherent of `cumulus-pallet-parachain-system`.
- **`other_inherent_data`**: Everything else, primarily the timestamp. Will become the `set_timestamp` inherent of `pallet-timestamp`.

At this point these are raw data in an `InherentData` key-value map. They become actual extrinsics later, when the runtime processes them during `apply_inherents`. The full mechanics of this transformation are explained in [Section 10](#10-applying-inherents-apply_inherents).

---

## 7. Block Construction: build_block_and_import

**File**: `cumulus/client/consensus/aura/src/collators/mod.rs`

```rust
pub async fn build_block_and_import(
    &mut self,
    mut params: BuildBlockAndImportParams<'_, Block, P>,
) -> Result<Option<BuiltBlock<Block>>, Box<dyn Error + Send + 'static>> {
    // 1. Prepare the digest (Aura pre-digest: slot, author)
    let mut digest = params.additional_pre_digest;
    digest.push(params.slot_claim.pre_digest.clone());

    // 2. Initialize the proposer
    let proposer = self.proposer.init(&params.parent_header).await?;

    // 3. Merge inherent data into a single map
    let mut inherent_data_combined = params.extra_inherent_data;
    params.parachain_inherent_data
        .provide_inherent_data(&mut inherent_data_combined).await?;

    // 4. Register ProofSizeExt (records the storage proof for the PoV)
    params.extra_extensions
        .register(ProofSizeExt::new(storage_proof_recorder.clone()));

    // 5. Propose the block (this is where it crosses into the runtime)
    let proposal = proposer.propose(propose_args).await?;

    // 6. Seal the block (sign with the collator's key)
    let sealed_importable = seal::<_, P>(
        proposal.block, proposal.storage_changes,
        &params.slot_claim.author_pub, &self.keystore,
    )?;

    // 7. Import locally into RocksDB
    self.block_import.import_block(sealed_importable).await?;

    // 8. Extract the storage proof (PoV)
    let proof = storage_proof_recorder.drain_storage_proof();

    Ok(Some(BuiltBlock { block, proof, backend_transaction }))
}
```

- **Step 3**: Combines timestamp and parachain data into a single `InherentData` map.
- **Step 4**: `ProofSizeExt` records which parts of storage were read/written during execution. This generates the **PoV (Proof of Validity)** that relay chain validators need to re-execute the block without having the complete state.
- **Step 5**: `propose` internally calls `propose_block` → `propose_with`, where the block is built using Runtime APIs.
- **Step 6**: Signs the block with the collator's key from the keystore. Proves the block was produced by an authorized collator.
- **Step 7**: The block is imported locally **before** being sent to the relay chain.

---

## 8. The Proposer: propose_block and propose_with

**File**: `substrate/client/basic-authorship/src/basic_authorship.rs`

`propose_block` is a wrapper that spawns `propose_with` on a blocking thread:

```rust
pub async fn propose_block(self, args: ProposeArgs<Block>) -> Result<Proposal<Block>> {
    let (tx, rx) = oneshot::channel();
    let spawn_handle = self.spawn_handle.clone();

    spawn_handle.spawn_blocking(
        "basic-authorship-proposer",
        None,
        async move {
            let res = self.propose_with(args).await;
            tx.send(res);
        }.boxed(),
    );

    rx.await?
}
```

It's spawned on a separate thread because building a block is a blocking operation (executes WASM, accesses storage) and shouldn't block the node's async runtime.

The actual logic is in `propose_with`:

```rust
async fn propose_with(self, args: ProposeArgs<Block>) -> Result<Proposal<Block>> {
    let deadline = (self.now)() + max_duration - max_duration / 10;

    // 1. Create BlockBuilder (calls initialize_block here)
    let mut block_builder = BlockBuilderBuilder::new(&*self.client)
        .on_parent_block(self.parent_hash)
        .with_parent_block_number(self.parent_number)
        .with_proof_recorder(storage_proof_recorder)
        .with_inherent_digests(inherent_digests)
        .with_extra_extensions(extra_extensions)
        .build()?;

    // 2. Apply inherents
    self.apply_inherents(&mut block_builder, inherent_data)?;

    // 3. Check if transactions are allowed
    let mode = block_builder.extrinsic_inclusion_mode();
    let end_reason = match mode {
        ExtrinsicInclusionMode::AllExtrinsics => {
            self.apply_extrinsics(&mut block_builder, deadline, block_size_limit).await?
        },
        ExtrinsicInclusionMode::OnlyInherents => EndProposingReason::TransactionForbidden,
    };

    // 4. Finalize the block
    let (block, storage_changes) = block_builder.build()?.into_inner();
    Ok(Proposal { block, storage_changes })
}
```

---

## 9. initialize_block: Runtime Preparation

Creating the `BlockBuilder` (step 1 of `propose_with`) internally calls the Runtime API `Core::initialize_block`. This crosses the node→runtime boundary via the WASM boundary.

**File**: `substrate/frame/executive/src/lib.rs`

```rust
fn initialize_block_impl(
    block_number: &BlockNumberFor<System>,
    parent_hash: &System::Hash,
    digest: &Digest,
) {
    // 1. Reset events from the previous block
    <frame_system::Pallet<System>>::reset_events();

    // 2. Check for runtime upgrade
    let mut weight = Weight::zero();
    if Self::runtime_upgraded() {
        weight = weight.saturating_add(Self::execute_on_runtime_upgrade());
        frame_system::LastRuntimeUpgrade::<System>::put(...);
    }

    // 3. Initialize the system (sets block number, parent hash, digest)
    <frame_system::Pallet<System>>::initialize(block_number, parent_hash, digest);

    // 4. Register base block weight
    weight = System::BlockWeights::get().base_block.saturating_add(weight);
    <frame_system::Pallet<System>>::register_extra_weight_unchecked(
        weight, DispatchClass::Mandatory,
    );

    // 5. Run on_initialize for all pallets
    weight = weight.saturating_add(
        <AllPalletsWithSystem as OnInitializeWithWeightRegistration<System>>
            ::on_initialize_with_weight_registration(*block_number)
    );

    // 6. Mark initialization as finished
    frame_system::Pallet::<System>::note_finished_initialize();

    // 7. Pre-inherents hook
    <System as frame_system::Config>::PreInherents::pre_inherents();
}
```

### Block Number Is Set Before Anything Else

In step 3, `frame_system::Pallet::initialize` writes the block number to storage:

```rust
pub fn initialize(number: &BlockNumberFor<T>, parent_hash: &T::Hash, digest: &generic::Digest) {
    <Number<T>>::put(number);
    <ParentHash<T>>::put(parent_hash);
    <BlockHash<T>>::insert(number.saturating_sub(One::one()), parent_hash);
    <Digest<T>>::put(digest);
}
```

The `number` comes from the header the node constructed before passing it to the runtime. This means the block number is already available in storage when inherents and transactions execute later.

### ExtrinsicInclusionMode

After `initialize_block` returns, the mode is determined:

```rust
fn extrinsic_mode() -> ExtrinsicInclusionMode {
    if <System as frame_system::Config>::MultiBlockMigrator::ongoing() {
        ExtrinsicInclusionMode::OnlyInherents
    } else {
        ExtrinsicInclusionMode::AllExtrinsics
    }
}
```

If a multi-block storage migration is in progress, only inherents are allowed. User transactions are blocked because the storage is in an inconsistent state during migration.

---

## 10. Applying Inherents: apply_inherents

This is where the raw `InherentData` map created in [step 9 of the main loop](#the-inherent-data-step-9) gets transformed into actual extrinsics and applied to the block. To understand this process, it is necessary to first understand what inherents are, how the `InherentData` mechanism works, and how individual pallets like `pallet-timestamp` handle them.

---

### 10.1. What Are Inherents and Why Do They Exist

Inherents are a special type of extrinsic used when **data from the outside world that only the block producer can provide** needs to be included in the block, and the runtime cannot obtain it on its own.

The runtime is isolated inside a WASM sandbox. It has no access to:
- The operating system clock
- The P2P network
- The relay chain
- External APIs
- Hardware / sensors
- The filesystem

The node is the one with access to the real world and passes data to the runtime through inherents. Unlike signed transactions, inherents:
- Have no signature
- Pay no fees
- Do not pass through the transaction pool
- Are inserted directly by the block-producing node

The fundamental tradeoff is that **you trust the block producer blindly**. If the collator lies about the timestamp or any other inherent data, the runtime has no way to know (apart from basic checks like "time must move forward"). That's why inherents are reserved for fundamental infrastructure where there's no alternative.

---

### 10.2. The InherentData Mechanism

`InherentData` is a generic key-value map containing data from **all** inherent providers mixed together. Back in [step 9 of the main loop](#the-inherent-data-step-9), the node-side `InherentDataProvider`s each inserted their data with a unique identifier:

```rust
// The timestamp provider did:
inherent_data.put_data(b"timstap0", &timestamp_value);

// The parachain system provider did:
inherent_data.put_data(b"sprpch0", &validation_data);
```

Everything goes into a single map. The entire map is passed to the runtime at once. Each pallet with `ProvideInherent` then searches for its own key in the map — like a shared mailbox where each pallet looks for the letter with its label.

Nothing forces a pallet to use the `InherentData`. A pallet could implement `create_inherent` ignoring the parameter and always returning an extrinsic. However, in that case `on_initialize` (a hook that runs automatically every block without an extrinsic) would be more appropriate.

---

### 10.3. create_inherents: From Raw Data to Extrinsics

The `create_inherents` call is made inside `apply_inherents` (see [Section 10.9](#109-the-apply_inherents-function)). It calls the Runtime API `BlockBuilder::inherent_extrinsics(data)`, which iterates all pallets that implement `ProvideInherent`. Each pallet:

1. Searches the `InherentData` map for its identifier key.
2. If found, deserializes the value to its expected type.
3. Creates and returns a `Call` — the extrinsic to be included in the block.

In a parachain, this typically produces two extrinsics:
- `Timestamp::set(now)` from `pallet-timestamp`
- `ParachainSystem::set_validation_data(data)` from `cumulus-pallet-parachain-system`

---

### 10.4. Deep Dive: pallet-timestamp create_inherent

**File**: `substrate/frame/timestamp/src/lib.rs`

```rust
#[pallet::inherent]
impl<T: Config> ProvideInherent for Pallet<T> {
    type Call = Call<T>;
    type Error = InherentError;
    const INHERENT_IDENTIFIER: InherentIdentifier = INHERENT_IDENTIFIER;

    fn create_inherent(data: &InherentData) -> Option<Self::Call> {
        let inherent_data = data
            .get_data::<T::Moment>(&INHERENT_IDENTIFIER)
            .expect("Gets the timestamp inherent data")
            .expect("Timestamp inherent data must exist");

        Some(Call::set { now: inherent_data })
    }
}
```

- `data.get_data()`: Searches the inherent data map for key `timstap0` (the `INHERENT_IDENTIFIER` for timestamp). This is the value the node's `InherentDataProvider` put there by reading `SystemTime::now()` from the operating system.
- `Some(Call::set { now: inherent_data })`: Returns a `Call`, the enum representing the pallet's extrinsics. `Call::set` is the variant corresponding to the `set(origin, now)` dispatchable function. This `Call` is the inherent extrinsic that will be applied to the block.

---

### 10.5. Deep Dive: Timestamp::set Execution

When the inherent extrinsic created above is applied (via `block_builder.push`), the following function executes:

**File**: `substrate/frame/timestamp/src/lib.rs`

```rust
pub fn set(origin: OriginFor<T>, #[pallet::compact] now: T::Moment) -> DispatchResult {
    ensure_none(origin)?;
    assert!(!DidUpdate::<T>::exists(), "Timestamp must be updated only once in the block");
    let prev = Now::<T>::get();
    assert!(
        prev.is_zero() || now >= prev + T::MinimumPeriod::get(),
        "Timestamp must increment by at least <MinimumPeriod> between sequential blocks"
    );
    Now::<T>::put(now);
    DidUpdate::<T>::put(true);
    <T::OnTimestampSet as OnTimestampSet<_>>::on_timestamp_set(now);
    Ok(())
}
```

- **`ensure_none(origin)?`**: Verifies the origin is `None` (no signature). Inherents have no signature. If someone tried to send `Timestamp::set` as a signed transaction, it fails here.
- **`assert!(!DidUpdate::<T>::exists())`**: Guarantees it's called only **once per block**. `DidUpdate` is a storage flag.
- **`let prev = Now::<T>::get()`**: Reads the previous block's timestamp from storage.
- **`assert!(prev.is_zero() || now >= prev + T::MinimumPeriod::get())`**: Validates that time moves forward. Prevents a malicious collator from setting a timestamp in the past or too close to the previous one.
- **`Now::<T>::put(now)`**: Writes the new timestamp to storage. From here on, any pallet calling `T::TimeProvider::now()` gets this value.
- **`DidUpdate::<T>::put(true)`**: Sets the flag so it can't be called again this block.
- **`on_timestamp_set(now)`**: Notifies subscribers via the `OnTimestampSet` hook.

`Now` only stores the current value — it is overwritten every block. There is no history. `DidUpdate` is cleared in the pallet's `on_finalize()`, resetting it for the next block.

---

### 10.6. OnTimestampSet Subscribers

The subscribers that typically register for `OnTimestampSet` are consensus pallets:

- **`pallet-babe`**: Uses the timestamp to calculate the current slot.
- **`pallet-aura`**: Same — calculates the slot from the timestamp divided by `SlotDuration`.

The connection is configured in the runtime:

```rust
impl pallet_timestamp::Config for Runtime {
    type OnTimestampSet = Aura; // defines who subscribes
}
```

Multiple pallets can subscribe using a tuple:

```rust
type OnTimestampSet = (Aura, MyCustomPallet);
```

A custom pallet subscribing to this hook could, for example, store a bounded history of timestamps for use in on-chain logic.

---

### 10.7. Accessing the Timestamp from Other Pallets

Any pallet that needs the current timestamp can read it from storage using a generic trait, without coupling to `pallet-timestamp` directly.

**File**: `substrate/frame/support/src/traits/time.rs`

```rust
pub trait Time {
    type Moment;
    fn now() -> Self::Moment;
}
```

This trait is generic from `frame_support`. It doesn't mention `pallet-timestamp` at all. In a custom pallet, a type bound is defined **inside the pallet**:

```rust
#[pallet::config]
pub trait Config: frame_system::Config {
    type TimeProvider: frame_support::traits::Time;
}
```

And in the **runtime**, the dependency is satisfied:

```rust
impl my_pallet::Config for Runtime {
    type TimeProvider = Timestamp;
}
```

This is the classic **dependency inversion** pattern. The pallet defines what it needs; the runtime provides the concrete implementation. In tests, `TimeProvider` can be mocked without instantiating `pallet-timestamp` at all.

`pallet-timestamp` implements the `Time` trait by simply reading from storage:

```rust
impl<T: Config> Time for Pallet<T> {
    type Moment = T::Moment;
    fn now() -> T::Moment {
        Now::<T>::get()
    }
}
```

This reads `Now::<T>`, the same storage item written by `set()`. The complete cycle: node reads OS clock → `InherentDataProvider` packages it → runtime calls `create_inherent()` → generates `Call::set` → executes `set(now)` → `Now::<T>::put(now)` writes to storage → other pallets read via `T::TimeProvider::now()`.

---

### 10.8. Real-World Inherent Use Cases

Beyond the built-in timestamp and parachain validation data, custom inherents could be used for:

- **On-chain price oracle**: The node queries a price API and injects it every block for a DEX or lending platform.
- **Verifiable randomness**: The node generates a VRF with its key each block. BABE in the relay chain does exactly this.
- **Trustless bridge**: A node running an Ethereum light client passes the latest ETH headers as an inherent.
- **High-precision timestamps**: If the chain needs more granular timestamps than the relay chain provides (e.g., for a high-frequency exchange).
- **IoT data**: Nodes in warehouses reading sensors (temperature, location, weight) and injecting them for supply chain tracking.
- **Censorship resistance**: An inherent reporting transactions seen in the mempool but not included in recent blocks.

Creating a custom inherent requires:
1. An `InherentDataProvider` on the node side (implementing the trait).
2. A pallet in the runtime implementing `ProvideInherent`.
3. Registration in both the node's provider chain and the runtime's `construct_runtime!`.

---

### 10.9. The apply_inherents Function

**File**: `substrate/client/basic-authorship/src/basic_authorship.rs`

```rust
fn apply_inherents(
    &self,
    block_builder: &mut sc_block_builder::BlockBuilder<'_, Block, C>,
    inherent_data: InherentData,
) -> Result<(), sp_blockchain::Error> {
    let inherents = block_builder.create_inherents(inherent_data)?;

    for inherent in inherents {
        match block_builder.push(inherent) {
            Err(ApplyExtrinsicFailed(Validity(e))) if e.exhausted_resources() => {
                warn!("Dropping non-mandatory inherent from overweight block.")
            },
            Err(ApplyExtrinsicFailed(Validity(e))) if e.was_mandatory() => {
                error!("Mandatory inherent extrinsic returned error.");
                return Err(ApplyExtrinsicFailed(Validity(e)));
            },
            Err(e) => {
                warn!("Inherent extrinsic returned unexpected error: {}. Dropping.", e);
            },
            Ok(_) => {},
        }
    }
    Ok(())
}
```

- `block_builder.create_inherents(inherent_data)` calls the runtime via Runtime API `BlockBuilder::inherent_extrinsics(data)`. The runtime iterates all pallets with `ProvideInherent` and each one creates its extrinsic (as described in [Section 10.3](#103-create_inherents-from-raw-data-to-extrinsics)).
- Each inherent extrinsic is applied with `push`, which internally calls `apply_extrinsic` in the runtime (described in detail in [Section 13](#13-blockbuilderpush--crossing-into-the-runtime)).
- Three error cases:
  - **Exhausted resources, non-mandatory**: The inherent doesn't fit by weight but isn't required. It's dropped with a warning.
  - **Mandatory failed**: A required inherent (like timestamp or validation data) failed. Block production is **aborted entirely**. A block cannot exist without mandatory inherents.
  - **Other error**: Unexpected error on a non-mandatory inherent. Dropped with a warning.

The mandatory/non-mandatory distinction comes from each pallet's `is_inherent_required()` implementation.

---

## 11. Applying Transactions: apply_extrinsics

**File**: `substrate/client/basic-authorship/src/basic_authorship.rs`

```rust
async fn apply_extrinsics(
    &self,
    block_builder: &mut sc_block_builder::BlockBuilder<'_, Block, C>,
    deadline: time::Instant,
    block_size_limit: Option<usize>,
) -> Result<EndProposingReason, sp_blockchain::Error> {
    let soft_deadline =
        now + time::Duration::from_micros(self.soft_deadline_percent.mul_floor(left_micros));
    let mut skipped = 0;
    let mut pending_iterator =
        self.transaction_pool.ready_at_with_timeout(self.parent_hash, delay).await;

    let end_reason = loop {
        // 1. Get the next transaction from the pool
        let pending_tx = if let Some(pending_tx) = pending_iterator.next() {
            pending_tx
        } else {
            break EndProposingReason::NoMoreTransactions;
        };

        // 2. Check if time is up
        if now > deadline {
            break EndProposingReason::HitDeadline;
        }

        // 3. Check if it fits by size
        if block_size + pending_tx_data.encoded_size() > block_size_limit { ... }

        // 4. Try to apply the transaction
        match block_builder.push(pending_tx_data) {
            Ok(()) => { skipped = 0; },
            Err(exhausted_resources) => {
                if skipped < MAX_SKIPPED_TRANSACTIONS { skipped += 1; continue; }
                else if now < soft_deadline { continue; }
                else { break EndProposingReason::HitBlockWeightLimit; }
            },
            Err(invalid) => { unqueue_invalid.insert(pending_tx_hash, ...); },
        }
    };

    self.transaction_pool.report_invalid(Some(self.parent_hash), unqueue_invalid).await;
    Ok(end_reason)
}
```

The loop exits for four reasons:

- **`NoMoreTransactions`**: The pool is empty.
- **`HitDeadline`**: The `authoring_duration` expired. The collator must finish building before the slot passes.
- **`HitBlockSizeLimit`**: Doesn't fit by size (bytes). In parachains, `block_size_limit` comes from the relay chain's `max_pov_size` (typically 85%).
- **`HitBlockWeightLimit`**: Doesn't fit by weight (computation).

### Soft Deadline vs Hard Deadline

There's a **soft deadline** (50% of the time) that changes behavior when the block is filling up. Before the soft deadline, if a transaction doesn't fit, the system keeps trying with others. After the soft deadline, after `MAX_SKIPPED_TRANSACTIONS` (8) failed attempts, it stops. This optimizes block utilization without wasting too much time.

---

## 12. The Transaction Pool and Prioritization

`ready_at_with_timeout` returns an **iterator** that yields transactions already ordered by priority. It doesn't return the entire mempool at once — it yields them one by one.

### How Priority Is Determined

When a transaction enters the pool, the node calls the runtime via Runtime API `TaggedTransactionQueue::validate_transaction`. The runtime returns:

```rust
pub struct ValidTransaction {
    pub priority: TransactionPriority,  // u64 - how important it is
    pub requires: Vec<TransactionTag>,  // dependencies (e.g., previous nonce)
    pub provides: Vec<TransactionTag>,  // what it provides (e.g., this nonce)
    pub longevity: TransactionLongevity,
    pub propagate: bool,
}
```

- **`priority`**: Calculated in the `TransactionExtension` of `pallet-transaction-payment`, typically based on tip and weight. Higher tip = higher priority.
- **`requires`/`provides`**: Handle nonce ordering. If Alice sends nonce 5 and 6, the pool knows nonce 6 depends on nonce 5 (requires the tag that nonce 5 provides). This ensures they are ordered sequentially even if they arrive out of order.

The pool maintains a dependency graph ordered by priority. The `ready()` iterator yields the highest-priority transactions with all dependencies satisfied first — a topological sort by priority.

The pool itself doesn't decide the priority; it only sorts using what the runtime told it in `validate_transaction`. The runtime owns the prioritization policy.

### Relevant Files

- **Priority calculation**: `substrate/frame/transaction-payment/src/lib.rs`
- **Main pool**: `substrate/client/transaction-pool/src/fork_aware_txpool/`
- **Dependency graph**: `substrate/client/transaction-pool/src/graph/ready.rs`
- **Runtime API**: `substrate/primitives/transaction-pool/src/runtime_api.rs`

---

## 13. BlockBuilder::push — Crossing into the Runtime

**File**: `substrate/client/block-builder/src/lib.rs`

```rust
pub fn push(&mut self, xt: <Block as BlockT>::Extrinsic) -> Result<(), Error> {
    let parent_hash = self.parent_hash;
    let extrinsics = &mut self.extrinsics;
    let version = self.version;

    self.api.execute_in_transaction(|api| {
        let res = if version < 6 {
            api.apply_extrinsic_before_version_6(parent_hash, xt.clone())
        } else {
            api.apply_extrinsic(parent_hash, xt.clone())
        };

        match res {
            Ok(Ok(_)) => {
                extrinsics.push(xt);
                TransactionOutcome::Commit(Ok(()))
            },
            Ok(Err(tx_validity)) => TransactionOutcome::Rollback(Err(
                ApplyExtrinsicFailed::Validity(tx_validity).into(),
            )),
            Err(e) => TransactionOutcome::Rollback(Err(Error::from(e))),
        }
    })
}
```

The key is `execute_in_transaction`. This is **not** a blockchain transaction — it's a **storage transaction** (like a savepoint in a database):

- **Before execution**: Takes a snapshot of the current state.
- **`api.apply_extrinsic`**: Crosses into the runtime via WASM boundary, executes the extrinsic completely.
- **If `Ok(Ok(_))`**: `Commit` — storage changes are kept, the extrinsic is added to the block's vector.
- **If `Ok(Err(...))`**: `Rollback` — **reverts all storage changes** to the snapshot. As if the transaction never happened.
- **If `Err(_)`**: `Rollback` as well.

This is critical for the PoV: a failed extrinsic doesn't leave storage reads recorded in the proof recorder, keeping the PoV size minimal.

This same `push` function is used for both inherents and user transactions. The only difference is that inherents are `Bare` (no signature, no fees) while user transactions are `Signed`.

---

## 14. Executive::do_apply_extrinsic

**File**: `substrate/frame/executive/src/lib.rs`

```rust
fn do_apply_extrinsic(
    uxt: Block::Extrinsic,
    is_inherent: bool,
    check: impl FnOnce(...) -> Result<CheckedOf<...>, TransactionValidityError>,
) -> ApplyExtrinsicResult {
    // 1. Re-decode with depth limit (protection against stack overflow)
    let encoded = uxt.encode();
    let encoded_len = encoded.len();
    let uxt = <Block::Extrinsic as codec::DecodeLimit>::decode_all_with_depth_limit(
        MAX_EXTRINSIC_DEPTH, &mut &encoded[..],
    ).map_err(|_| InvalidTransaction::Call)?;

    // 2. Verify signature
    let xt = check(uxt, &Context::default())?;

    // 3. Get dispatch info (estimated weight and class)
    let dispatch_info = xt.get_dispatch_info();

    // 4. Mark end of inherents if applicable
    if !is_inherent && !<frame_system::Pallet<System>>::inherents_applied() {
        Self::inherents_applied();
    }

    // 5. Register the extrinsic
    <frame_system::Pallet<System>>::note_extrinsic(encoded);

    // 6. Apply — THIS IS WHERE EVERYTHING HAPPENS
    let r = Applyable::apply::<UnsignedValidator>(xt, &dispatch_info, encoded_len)?;

    // 7. Verify mandatory
    if r.is_err() && dispatch_info.class == DispatchClass::Mandatory {
        return Err(InvalidTransaction::BadMandatory.into());
    }

    // 8. Record result
    <frame_system::Pallet<System>>::note_applied_extrinsic(&r, dispatch_info);

    Ok(r.map(|_| ()).map_err(|e| e.error))
}
```

- **Step 1**: Protection against malicious extrinsics with infinitely nested structures that could cause stack overflow.
- **Step 2**: Cryptographic signature verification (`check`). See [Section 15](#15-signature-verification-check).
- **Step 3**: Gets the estimated weight and dispatch class (`Normal`, `Operational`, or `Mandatory`). Inherents are `Mandatory`.
- **Step 4**: The first time a non-inherent extrinsic arrives, it marks the transition from "inherents phase" to "transactions phase." This triggers `PostInherents` and `on_poll` or `MultiBlockMigrator::step`.
- **Step 5**: Registers the encoded extrinsic in storage for later extrinsics root calculation.
- **Step 6**: `Applyable::apply` — charges fees, dispatches the call, refunds excess fees. See [Section 17](#17-dispatch_transaction-validate-execute-post-dispatch).
- **Step 7**: If a mandatory inherent fails, it's fatal — no block can be produced.
- **Step 8**: `note_applied_extrinsic` only records the result (deposits `ExtrinsicSuccess` or `ExtrinsicFailed` event, updates weight counters). It does **not** execute the transaction — that already happened in step 6.

---

## 15. Signature Verification: check

**File**: `substrate/primitives/runtime/src/generic/unchecked_extrinsic.rs`

```rust
fn check(self, lookup: &Lookup) -> Result<Self::Checked, TransactionValidityError> {
    Ok(match self.preamble {
        Preamble::Signed(signed, signature, tx_ext) => {
            let signed = lookup.lookup(signed)?;
            let raw_payload = SignedPayload::new(
                CallAndMaybeEncoded { encoded: self.encoded_call, call: self.function },
                tx_ext,
            )?;
            if !raw_payload.using_encoded(|payload| signature.verify(payload, &signed)) {
                return Err(InvalidTransaction::BadProof.into());
            }
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

Three extrinsic types based on the `Preamble`:

- **`Preamble::Signed`**: User-signed transaction. The signature covers the complete payload: **call + extensions** (nonce, tip, era, chain_id, etc.). This is critical because without the nonce in the signature replay attacks are possible, without chain_id cross-chain replay is possible, without era transactions are immortal, and without tip an attacker could modify priority without invalidating the signature.
- **`Preamble::General`**: Extrinsic with extensions but without a signature. The modern replacement for unsigned extrinsics. Extensions can include their own authorization logic without a classic cryptographic signature.
- **`Preamble::Bare`**: No signature, no extensions. This is the format used by **inherents**. When `Timestamp::set` arrives here, it goes through this branch and exits directly as `ExtrinsicFormat::Bare`. The inherent's validation happens inside the pallet itself via `ensure_none(origin)?`.

The result is a `CheckedExtrinsic` with:
- **`format`**: Who sent it: `Signed(account, extensions)`, `General(extensions)`, or `Bare`.
- **`function`**: The decoded `Call` (which pallet and function to call).

---

## 16. Transaction Extensions Pipeline

### Where They Are Defined

Extensions are configured in the runtime as a tuple:

```rust
pub type TxExtension = (
    CheckNonZeroSender,           // sender is not the zero account
    CheckSpecVersion,             // correct spec version
    CheckTxVersion,               // correct tx version
    CheckGenesis,                 // correct genesis hash
    CheckEra,                     // transaction not expired
    CheckNonce,                   // correct nonce
    CheckWeight,                  // fits in the block by weight
    ChargeTransactionPayment,     // charge fees
);
```

### Individual Extension Implementations

| Extension | File | Purpose |
|-----------|------|---------|
| `CheckNonZeroSender` | `substrate/frame/system/src/extensions/check_non_zero_sender.rs` | Verifies sender ≠ zero account |
| `CheckSpecVersion` | `substrate/frame/system/src/extensions/check_spec_version.rs` | Verifies spec version |
| `CheckTxVersion` | `substrate/frame/system/src/extensions/check_tx_version.rs` | Verifies tx version |
| `CheckGenesis` | `substrate/frame/system/src/extensions/check_genesis.rs` | Verifies genesis hash |
| `CheckEra` | `substrate/frame/system/src/extensions/check_mortality.rs` | Verifies mortal/immortal era |
| `CheckNonce` | `substrate/frame/system/src/extensions/check_nonce.rs` | Verifies and consumes nonce |
| `CheckWeight` | `substrate/frame/system/src/extensions/check_weight.rs` | Verifies block weight |
| `ChargeTransactionPayment` | `substrate/frame/transaction-payment/src/lib.rs` | Calculates and charges fees |

### Relationship Between Signature and Extensions

The signature was already validated in `check` (previous section). The extensions perform **business logic validations** after the signature is already valid. A perfectly valid signature can still have an incorrect nonce, insufficient funds for fees, or an expired era. These are all caught here in the extensions, not in the signature verification. Two separate phases: authentication (signature) then authorization and preconditions (extensions).

### Tuple Implementation

**File**: `substrate/primitives/runtime/src/traits/transaction_extension/mod.rs`

```rust
#[impl_for_tuples(1, 12)]
impl<Call: Dispatchable> TransactionExtension<Call> for Tuple {
    fn validate(
        &self, origin, call, info, len, self_implicit, inherited_implication, source,
    ) -> Result<(ValidTransaction, Self::Val, RuntimeOrigin), TransactionValidityError> {
        let valid = ValidTransaction::default();
        let val = ();

        for_tuples!(#(
            let (item_valid, item_val, origin) = {
                Tuple.validate(origin, call, info, len, item_implicit,
                    &ImplicationParts { ... }, source)?
            };
            let valid = valid.combine_with(item_valid);
            let val = val.push_back(item_val);
        )* );

        Ok((valid, val, origin))
    }
}
```

Iterates over each extension **in order**. Each one:
- Receives the `origin` returned by the previous one (an extension can change the origin for subsequent ones).
- Returns a `ValidTransaction` that is **combined** with previous ones (`combine_with` merges priorities, requires, provides, etc.).
- Returns a `Val` that accumulates into a tuple for later passing to `prepare`.
- If any fails, **the entire chain is cut** and the transaction is invalid.

The same applies to `prepare` and `post_dispatch`: executed sequentially, one by one.

---

## 17. dispatch_transaction: Validate, Execute, Post-Dispatch

**File**: `substrate/primitives/runtime/src/traits/transaction_extension/dispatch_transaction.rs`

```rust
fn dispatch_transaction(
    self,
    origin: <Call as Dispatchable>::RuntimeOrigin,
    call: Call,
    info: &DispatchInfoOf<Call>,
    len: usize,
    extension_version: ExtensionVersion,
) -> Self::Result {
    // 1. Validate and prepare
    let (pre, origin) =
        self.validate_and_prepare(origin, &call, info, len, extension_version)?;

    // 2. Dispatch the call
    let mut res = call.dispatch(origin);

    // 3. Post dispatch
    let pd_res = res.map(|_| ()).map_err(|e| e.error);
    let post_info = match &mut res {
        Ok(info) => info,
        Err(err) => &mut err.post_info,
    };
    post_info.set_extension_weight(info);
    T::post_dispatch(pre, info, post_info, len, &pd_res)?;
    Ok(res)
}
```

### Step 1: validate_and_prepare

```rust
fn validate_and_prepare(self, origin, call, info, len, extension_version) {
    let (_, val, origin) = self.validate_only(
        origin, call, info, len, InBlock, extension_version)?;
    let pre = self.prepare(val, &origin, &call, info, len)?;
    Ok((pre, origin))
}
```

- `validate` runs through all extensions checking preconditions.
- `prepare` runs through the same extensions **mutating state**: `CheckNonce` increments the nonce, `ChargeTransactionPayment` charges fees, `CheckWeight` reserves weight. The `pre` returned by each extension contains data needed for post_dispatch (e.g., how much was charged in fees to know how much to refund).

### Step 2: call.dispatch(origin)

**This is where the call is executed.** The final dispatch to the pallet. The origin comes resolved as `Signed(account)` from the check.

### Step 3: T::post_dispatch

After executing the call, it runs through all extensions in post-dispatch. `ChargeTransactionPayment` compares the estimated weight with the actual weight consumed and **refunds excess fees** to the user if less was used. `set_extension_weight` registers the weight of the extensions themselves.

### The Three Branches of apply by Format

**File**: `substrate/primitives/runtime/src/generic/checked_extrinsic.rs`

```rust
fn apply<I: ValidateUnsigned<Call = Self::Call>>(
    self, info: &DispatchInfoOf<Self::Call>, len: usize,
) -> ApplyExtrinsicResultWithInfo<PostDispatchInfoOf<Self::Call>> {
    match self.format {
        ExtrinsicFormat::Bare => {
            I::pre_dispatch(&self.function)?;
            Extension::bare_validate_and_prepare(&self.function, info, len)?;
            let res = self.function.dispatch(None.into());
            Extension::bare_post_dispatch(info, &mut post_info, len, &pd_res)?;
            Ok(res)
        },
        ExtrinsicFormat::Signed(signer, extension) => extension.dispatch_transaction(
            Some(signer).into(), self.function, info, len, DEFAULT_EXTENSION_VERSION,
        ),
        ExtrinsicFormat::General(extension_version, extension) => extension
            .dispatch_transaction(None.into(), self.function, info, len, extension_version),
    }
}
```

- **Bare** (inherents): Origin `None`, no fees, no signature. Basic pre/post dispatch.
- **Signed**: Origin `Some(signer)`, full extensions pipeline as described above.
- **General**: Origin `None`, extensions pipeline without signature.

---

## 18. The Dispatch to the Pallet

`dispatch` comes from the `Dispatchable` trait. For the runtime, the call is the `RuntimeCall` enum generated by `construct_runtime!`.

### Two-Level Routing

**Level 1 — RuntimeCall** (generated by `construct_runtime!`):

```rust
impl Dispatchable for RuntimeCall {
    fn dispatch(self, origin: RuntimeOrigin) -> DispatchResultWithPostInfo {
        match self {
            RuntimeCall::Balances(call) => call.dispatch(origin),
            RuntimeCall::Timestamp(call) => call.dispatch(origin),
            RuntimeCall::System(call) => call.dispatch(origin),
            // ... each pallet in the runtime
        }
    }
}
```

**Level 2 — Pallet Call** (generated by `#[pallet::call]`):

```rust
impl Dispatchable for Call<T> {
    fn dispatch(self, origin: RuntimeOrigin) -> DispatchResultWithPostInfo {
        match self {
            Call::transfer_allow_death { dest, value } => {
                Self::transfer_allow_death(origin, dest, value)
            },
            // ... each extrinsic in the pallet
        }
    }
}
```

It's simply pattern matching on enums. Everything is resolved statically at compile time. No magic, no reflection, no dynamic function tables.

When a user submits a transaction, the call comes serialized as `(pallet_index, function_index, params)`. It's deserialized to the `RuntimeCall` enum, and the double dispatch routes it: `RuntimeCall::Balances(transfer_allow_death { dest, value })` → Level 1 match → goes to pallet Balances → Level 2 match → goes to function `transfer_allow_death` → executes `Pallet::<T>::transfer_allow_death(origin, dest, value)`.

---

## 19. Block Finalization: build and finalize_block

When the transaction loop ends (any of the four exit conditions from [Section 11](#11-applying-transactions-apply_extrinsics)), `block_builder.build()` is called.

**File**: `substrate/client/block-builder/src/lib.rs`

```rust
pub fn build(self) -> Result<BuiltBlock<Block>, Error> {
    // 1. Finalize the block (crosses into the runtime)
    let header = self.api.finalize_block(self.parent_hash)?;

    // 2. Verify extrinsics root
    debug_assert_eq!(
        header.extrinsics_root().clone(),
        HashingFor::<Block>::ordered_trie_root(
            self.extrinsics.iter().map(Encode::encode).collect(),
            self.api.version(self.parent_hash)?.extrinsics_root_state_version(),
        ),
    );

    // 3. Extract storage changes
    let state = self.call_api_at.state_at(self.parent_hash)?;
    let storage_changes = self.api
        .into_storage_changes(&state, self.parent_hash)
        .map_err(sp_blockchain::Error::StorageChanges)?;

    // 4. Return the built block
    Ok(BuiltBlock {
        block: <Block as BlockT>::new(header, self.extrinsics),
        storage_changes,
    })
}
```

Inside the runtime, `finalize_block` is handled by the `Executive`:

**File**: `substrate/frame/executive/src/lib.rs`

```rust
pub fn finalize_block() -> HeaderFor<System> {
    if !<frame_system::Pallet<System>>::inherents_applied() {
        Self::inherents_applied();
    }

    <frame_system::Pallet<System>>::note_finished_extrinsics();
    <System as frame_system::Config>::PostTransactions::post_transactions();
    Self::on_idle_hook(block_number);
    Self::on_finalize_hook(block_number);
    <frame_system::Pallet<System>>::maybe_apply_pending_code_upgrade();
    <frame_system::Pallet<System>>::finalize()
}
```

- **`note_finished_extrinsics`**: Marks that no more extrinsics will enter.
- **`PostTransactions`**: Hook that runs after all transactions.
- **`on_idle`**: If weight remains in the block, pallets can use it for background work (cleanup, garbage collection, etc.).
- **`on_finalize`**: Runs the `on_finalize` hook for all pallets. Here `pallet-timestamp` clears `DidUpdate` for the next block.
- **`maybe_apply_pending_code_upgrade`**: If there was a `set_code` during the block (runtime upgrade), it's applied here.
- **`finalize()`**: Calculates the state root and returns the final header.

---

## 20. frame_system::Pallet::finalize

**File**: `substrate/frame/system/src/lib.rs`

```rust
pub fn finalize() -> HeaderFor<T> {
    // 1. Log resource usage
    Self::resource_usage_report();

    // 2. Clean up temporary storage
    ExecutionPhase::<T>::kill();
    BlockSize::<T>::kill();
    storage::unhashed::kill(well_known_keys::INTRABLOCK_ENTROPY);
    InherentsApplied::<T>::kill();

    // 3. Read current block data
    let number = <Number<T>>::get();
    let parent_hash = <ParentHash<T>>::get();
    let digest = <Digest<T>>::get();

    // 4. Calculate extrinsics root
    let extrinsics = (0..ExtrinsicCount::<T>::take().unwrap_or_default())
        .map(ExtrinsicData::<T>::take)
        .collect();
    let extrinsics_root =
        extrinsics_data_root::<T::Hashing>(extrinsics, extrinsics_root_state_version);

    // 5. Prune old block hashes
    let block_hash_count = T::BlockHashCount::get();
    let to_remove = number.saturating_sub(block_hash_count).saturating_sub(One::one());
    if !to_remove.is_zero() {
        <BlockHash<T>>::remove(to_remove);
    }

    // 6. Calculate storage root
    let version = T::Version::get().state_version();
    let storage_root = T::Hash::decode(&mut &sp_io::storage::root(version)[..])
        .expect("Node is configured to use the same hash; qed");

    // 7. Assemble and return the final header
    HeaderFor::<T>::new(number, extrinsics_root, storage_root, parent_hash, digest)
}
```

- **Step 2**: Clears all temporary storage (execution phase, block size, intra-block entropy, inherents applied flag). This must not remain in permanent state.
- **Step 4**: Takes all extrinsics registered with `note_extrinsic` during the block, removes them from storage with `take`, and computes a **merkle trie root** over them.
- **Step 5**: Only keeps the last `BlockHashCount` block hashes. Older ones are deleted to prevent storage from growing infinitely.
- **Step 6**: `sp_io::storage::root(version)` — the most important line. A **host function** that crosses into the node to calculate the merkle trie root of all runtime storage.

---

## 21. Storage Root Calculation

### The Host Function

`sp_io::storage::root` is a host function declared in `substrate/primitives/io/src/lib.rs`:

```rust
#[runtime_interface]
pub trait Storage {
    fn root(&mut self, version: StateVersion) -> Vec<u8> {
        self.storage_root(version)
    }
}
```

The runtime WASM calls it but the node executes it. The runtime cannot calculate the trie root directly — the node has access to the full storage backend (RocksDB + overlay of changes).

### OverlayedChanges::storage_root

**File**: `substrate/primitives/state-machine/src/overlayed_changes/mod.rs`

```rust
pub fn storage_root<B: Backend<H>>(
    &mut self,
    backend: &B,
    state_version: StateVersion,
) -> (H::Out, bool)
where H::Out: Ord + Encode,
{
    if let Some(cache) = &self.storage_transaction_cache {
        return (cache.transaction_storage_root, true);
    }

    let delta = self.top.changes_mut()
        .map(|(k, v)| (&k[..], v.value().map(|v| &v[..])));

    let child_delta = self.children.values_mut()
        .map(|v| (&v.1, v.0.changes_mut()
            .map(|(k, v)| (&k[..], v.value().map(|v| &v[..])))));

    let (root, transaction) = backend.full_storage_root(delta, child_delta, state_version);

    self.storage_transaction_cache =
        Some(StorageTransactionCache { transaction, transaction_storage_root: root });

    (root, false)
}
```

- **Cache**: If already computed and no changes since, returns the cached value.
- **Top storage delta**: All changes during the block as `(key, Option<value>)`. `None` means deletion.
- **Child storage deltas**: Same for child tries (used by e.g., smart contracts with their own storage).
- **`backend.full_storage_root`**: Applies the deltas to the existing trie and recalculates the root.

---

## 22. full_storage_root: Hierarchical Trie

**File**: `substrate/primitives/state-machine/src/backend.rs`

```rust
fn full_storage_root<'a>(
    &self,
    delta: impl Iterator<Item = (&'a [u8], Option<&'a [u8]>)>,
    child_deltas: impl Iterator<
        Item = (&'a ChildInfo, impl Iterator<Item = (&'a [u8], Option<&'a [u8]>)>),
    >,
    state_version: StateVersion,
) -> (H::Out, BackendTransaction<H>)
where H::Out: Ord + Encode,
{
    let mut txs = BackendTransaction::with_hasher(RandomState::default());
    let mut child_roots: Vec<_> = Default::default();

    // 1. First the child tries
    for (child_info, child_delta) in child_deltas {
        let (child_root, empty, child_txs) =
            self.child_storage_root(child_info, child_delta, state_version);
        txs.consolidate(child_txs);
        if empty {
            child_roots.push((prefixed_storage_key.into_inner(), None));
        } else {
            child_roots.push((prefixed_storage_key.into_inner(), Some(child_root.encode())));
        }
    }

    // 2. Then the top storage, including child roots as entries
    let (root, parent_txs) = self.storage_root(
        delta.chain(child_roots.iter().map(|(k, v)| (&k[..], v.as_ref().map(|v| &v[..])))),
        state_version,
    );
    txs.consolidate(parent_txs);
    (root, txs)
}
```

A hierarchical design: child tries → their roots become entries in the top trie → the top trie root is the storage root of the block. A single hash summarizing **all** state.

The `BackendTransaction` contains the trie nodes that changed — used later for persisting to RocksDB without recalculating anything.

---

## 23. Sealing, Local Import, and Submission to the Relay Chain

Back in `build_block_and_import`, after `propose_with` returns:

```rust
// 6. Seal the block (sign with collator's key)
let sealed_importable = seal::<_, P>(
    proposal.block, proposal.storage_changes,
    &params.slot_claim.author_pub, &self.keystore,
)?;

// 7. Import locally
self.block_import.import_block(sealed_importable).await?;

// 8. Extract the PoV
let proof = storage_proof_recorder.drain_storage_proof();
```

And back in the main loop (`block_builder_task.rs`), the block is sent through the channel:

```rust
collator_sender.unbounded_send(CollatorMessage {
    relay_parent,
    parachain_candidate: candidate.into(),
    validation_code_hash,
    core_index: core.core_index(),
    max_pov_size: validation_data.max_pov_size,
});
```

The `collation_task` receives it and sends the block + PoV to the relay chain validators.

---

## 24. Collation Task: The Final Step of the Collator

After the block builder task constructs a block, seals it, imports it locally, and sends it through the async channel, the **collation task** picks it up. This is the last piece of the collator's work: it packages the block as a collation (block + compressed PoV) and submits it to the relay chain's Overseer. Once submitted, the collator's job is done.

**File**: `cumulus/client/consensus/aura/src/collators/slot_based/collation_task.rs`

This is still collator code (`cumulus/client/`), not relay chain code.

### run_collation_task

```rust
pub async fn run_collation_task<Block, RClient, CS>(
    Params {
        relay_client,
        collator_key,
        para_id,
        reinitialize,
        collator_service,
        mut collator_receiver,
        mut block_import_handle,
        export_pov,
    }: Params<Block, RClient, CS>,
) {
    let Ok(mut overseer_handle) = relay_client.overseer_handle() else {
        tracing::error!(target: LOG_TARGET, "Failed to get overseer handle.");
        return;
    };

    cumulus_client_collator::initialize_collator_subsystems(
        &mut overseer_handle,
        collator_key,
        para_id,
        reinitialize,
    )
    .await;

    loop {
        futures::select! {
            collator_message = collator_receiver.next() => {
                let Some(message) = collator_message else {
                    return;
                };

                handle_collation_message(
                    message, &collator_service, &mut overseer_handle,
                    relay_client.clone(), export_pov.clone()
                ).await;
            },
            block_import_msg = block_import_handle.next().fuse() => {
                // TODO: Implement me.
                let _ = block_import_msg;
            }
        }
    }
}
```

**Step by Step:**

- **Get the Overseer handle**: The Overseer is the central orchestrator of all subsystems in the Polkadot node. The collator runs a relay chain node internally (light client or full node) to communicate with the relay chain. The `overseer_handle` is the entry point to that world.

- **Initialize collator subsystems**: `initialize_collator_subsystems` registers this collator with the relay chain's collation generation subsystem. It tells the relay chain "I'm a collator for para_id X, here's my key." This only needs to happen once (or again if transitioning to Aura, controlled by the `reinitialize` flag).

- **Main loop**: Waits for messages on the channel from the block builder task. When a `CollatorMessage` arrives (the block we just built), it calls `handle_collation_message`. This runs concurrently with a `block_import_handle` listener (currently unimplemented).

### handle_collation_message

```rust
async fn handle_collation_message<Block: BlockT, RClient: RelayChainInterface + Clone + 'static>(
    message: CollatorMessage<Block>,
    collator_service: &impl CollatorServiceInterface<Block>,
    overseer_handle: &mut OverseerHandle,
    relay_client: RClient,
    export_pov: Option<PathBuf>,
) {
    let CollatorMessage {
        parent_header,
        parachain_candidate,
        validation_code_hash,
        relay_parent,
        core_index,
        max_pov_size,
    } = message;

    let hash = parachain_candidate.block.header().hash();
    let number = *parachain_candidate.block.header().number();

    // 1. Build the collation
    let (collation, block_data) =
        match collator_service.build_collation(&parent_header, hash, parachain_candidate) {
            Some(collation) => collation,
            None => {
                tracing::warn!(target: LOG_TARGET, "Unable to build collation.");
                return;
            },
        };

    // 2. Log PoV size
    if let MaybeCompressedPoV::Compressed(ref pov) = collation.proof_of_validity {
        tracing::info!(
            target: LOG_TARGET,
            "Compressed PoV size: {}kb",
            pov.block_data.0.len() as f64 / 1024f64,
        );
    }

    // 3. Submit to relay chain via Overseer
    overseer_handle
        .send_msg(
            CollationGenerationMessage::SubmitCollation(SubmitCollationParams {
                relay_parent,
                collation,
                parent_head: parent_header.encode().into(),
                validation_code_hash,
                core_index,
                result_sender: None,
                scheduling_parent: None,
            }),
            "SubmitCollation",
        )
        .await;
}
```

**Step by Step:**

1. **`build_collation`**: Takes the parachain candidate (block + storage proof) and packages it into a `Collation`. The collation contains the block header, the extrinsics, and the **compressed PoV** (Proof of Validity). The PoV is the storage proof recorded during block construction by the `ProofSizeExt` — it contains only the parts of storage that were accessed during execution, not the entire state.

2. **Log PoV size**: Logs the compressed PoV size. This is important because the relay chain enforces `max_pov_size`. If the PoV exceeds this limit, validators will reject the collation.

3. **`SubmitCollation`**: This is where the collator's work ends. It sends a `CollationGenerationMessage::SubmitCollation` to the Overseer. The parameters include:
   - **`relay_parent`**: Which relay chain block this collation is built on top of.
   - **`collation`**: The block + compressed PoV.
   - **`parent_head`**: The encoded parent header of the parachain block.
   - **`validation_code_hash`**: Hash of the WASM runtime, so validators know which code to use when re-executing the block for verification.
   - **`core_index`**: Which execution core on the relay chain this parachain is assigned to.

### What Happens After SubmitCollation

Once `send_msg` is called, the message enters the relay chain's subsystem pipeline. This is no longer collator code — it's relay chain validator code in `polkadot/node/`. The validators will:

1. Receive the collation via the collator protocol subsystem.
2. Re-execute the parachain block using only the PoV (without having the complete parachain state).
3. Verify the state root matches.
4. Distribute PoV chunks among validators for availability.
5. If validation passes, include the parachain block in the relay chain.
6. Finalize with GRANDPA.

This is outside the scope of the collator and would be a separate analysis of the relay chain's validation pipeline.

### Connection to the Block Builder Task

This task receives messages from the block builder task through the async channel set up in the collator entry point (`slot_based/mod.rs`):

```
block_builder_task                          collation_task
      |                                          |
      | builds block                             |
      | seals it                                 |
      | imports locally                          |
      | sends CollatorMessage ──── channel ────> |
      |                                          | receives message
      |                                          | build_collation (package block + PoV)
      |                                          | SubmitCollation to Overseer
      |                                          | (collator's job is done)
```

---

## 25. Persistence to RocksDB: execute_and_import_block

`import_block` triggers the persistence pipeline.

**File**: `substrate/client/service/src/client/client.rs`

The key section where storage changes are applied:

```rust
sc_consensus::StorageChanges::Changes(storage_changes) => {
    self.backend.begin_state_operation(&mut operation.op, parent_hash)?;
    let (main_sc, child_sc, offchain_sc, tx, _, tx_index) =
        storage_changes.into_inner();

    if self.config.offchain_indexing_api {
        operation.op.update_offchain_storage(offchain_sc)?;
    }

    operation.op.update_db_storage(tx)?;
    operation.op.update_storage(main_sc.clone(), child_sc.clone())?;
    operation.op.update_transaction_index(tx_index)?;
}
```

- **`update_db_storage(tx)`**: Applies the `BackendTransaction` with the merkle trie nodes that changed.
- **`update_storage(main_sc, child_sc)`**: Applies storage changes as flat key-value pairs for auxiliary lookup columns.
- **`update_transaction_index(tx_index)`**: Updates the transaction index.

Everything accumulates in `operation.op`. The actual write to RocksDB happens inside `lock_import_and_run`:

```rust
self.backend.commit_operation(op)?;
```

---

## 26. try_commit_operation: The Atomic Write

**File**: `substrate/client/db/src/lib.rs`

### commit_operation

```rust
fn commit_operation(&self, operation: Self::BlockImportOperation) -> ClientResult<()> {
    let usage = operation.old_state.usage_info();
    self.state_usage.merge_sm(usage);
    if let Err(e) = self.try_commit_operation(operation) {
        let state_meta_db = StateMetaDb(self.storage.db.clone());
        self.storage.state_db.reset(state_meta_db)?;
        self.blockchain.clear_pinning_cache();
        Err(e)
    } else {
        self.storage.state_db.sync();
        Ok(())
    }
}
```

All-or-nothing: if `try_commit_operation` fails, it resets the state_db to stay consistent with what's on disk. If it succeeds, `sync()` flushes to disk.

### Inside try_commit_operation

A `Transaction<DbHash>` is assembled with everything:

1. **Auxiliary and offchain**: `apply_aux`, `apply_offchain`.
2. **Finalized blocks**: `finalize_block_with_transaction` for each one.
3. **The new block**: Header → `columns::HEADER`, Body → `columns::BODY`, Justifications → `columns::JUSTIFICATIONS`.
4. **State trie nodes**: Via `state_db.insert_block` and `apply_state_commit`:

```rust
fn apply_state_commit(transaction: &mut Transaction<DbHash>, commit: CommitSet<Vec<u8>>) {
    for (key, val) in commit.data.inserted {
        transaction.set_from_vec(columns::STATE, &key, val);
    }
    for key in commit.data.deleted {
        transaction.remove(columns::STATE, &key);
    }
    for (key, val) in commit.meta.inserted {
        transaction.set_from_vec(columns::STATE_META, &key, val);
    }
    for key in commit.meta.deleted {
        transaction.remove(columns::STATE_META, &key);
    }
}
```

5. **Atomic commit**: `self.storage.db.commit(transaction)?;`

A single call that writes everything to RocksDB atomically: header, body, justifications, trie nodes, state meta, lookups, leaves, children, gaps. If anything fails, nothing is written.

### Database Columns

| Column | ID | Contents |
|--------|----|----------|
| STATE | 1 | Merkle trie nodes |
| STATE_META | 2 | state_db metadata |
| KEY_LOOKUP | 3 | hash→lookup key, number→hash |
| HEADER | 4 | Block headers |
| BODY | 5 | Block bodies |
| JUSTIFICATIONS | 6 | Justifications |
| AUX | 8 | Auxiliary data |
| OFFCHAIN | 9 | Offchain storage |
| TRANSACTION | 11 | Indexed transactions |

---

## 27. The Trie and Pruning

### Trie Nodes Are Not Overwritten

Each block **adds** new nodes to the trie; old ones remain. The merkle patricia trie uses nodes referenced by hash. When a value changes, **new nodes** with new hashes are created from the leaf to the root. Previous block's nodes continue existing in the DB.

```
Block 1:
  Root_A -> Node_B -> Leaf_C (Alice balance = 100)

Block 2 (Alice transfers 50):
  Root_D -> Node_E -> Leaf_F (Alice balance = 50)
```

`Root_A`, `Node_B`, `Leaf_C` remain in RocksDB. `Root_D`, `Node_E`, `Leaf_F` were inserted as new entries with different keys (because the hash changed). Nothing was overwritten.

That's why `state_at(hash)` works: it takes the state root from that block's header and navigates the trie from there. Each block has its own root pointing to its own tree of nodes.

### Pruning

This structure grows infinitely without intervention. The `state_db` tracks which nodes belong to which block. When a block is canonicalized and old enough (according to the `PruningMode`), it marks nodes that are no longer reachable from any live block for deletion:

```rust
let commit = self.storage.state_db.canonicalize_block(&hash)?;
apply_state_commit(transaction, commit);
```

This commit includes the `deleted` nodes that nobody needs anymore.

Pruning modes:
- **`PruningMode::ArchiveAll`**: Nothing is ever deleted. State from all blocks is available.
- **`PruningMode::blocks_pruning(256)`**: Only the last 256 blocks of state are kept.

Attempting `state_at(old_block)` on a node with pruning fails with "State already discarded."

---

## 28. Complete Production Cycle Summary

```
 1. Collator waits for next slot (slot_based/block_builder_task.rs)
 2. Gets best relay chain block hash
 3. Finds the correct relay parent with offset
 4. Calculates parachain slot from relay chain
 5. Finds best parachain parent block
 6. Determines assigned core on the relay chain
 7. Checks if it's our turn to produce (can_build_upon with keystore)
 8. Assembles PersistedValidationData from relay chain
 9. Creates inherent data (timestamp + parachain validation data)
10. Calls build_block_and_import:
    a. Merges inherent data into single map
    b. Registers ProofSizeExt for PoV recording
    c. Calls proposer.propose() which spawns propose_with on blocking thread:
       i.   BlockBuilder::new() → calls initialize_block in the runtime:
            - Resets events
            - Checks for runtime upgrade
            - Sets block number, parent hash, digest
            - Runs on_initialize for all pallets
       ii.  apply_inherents:
            - create_inherents → each pallet with ProvideInherent creates its Call
            - push each inherent → apply_extrinsic in runtime → Timestamp::set,
              ParachainSystem::set_validation_data
       iii. apply_extrinsics (loop pulling from transaction pool):
            - For each transaction:
              · check: verify cryptographic signature
              · validate extensions: nonce, fees, era, spec version, genesis, weight
              · prepare: charge fees, consume nonce, reserve weight
              · dispatch: two-level routing → RuntimeCall → Pallet Call → execute function
              · post_dispatch: refund excess fees
            - Loop exits on: empty pool, deadline, size limit, or weight limit
       iv.  block_builder.build() → finalize_block in runtime:
            - note_finished_extrinsics
            - PostTransactions hook
            - on_idle (if weight remaining)
            - on_finalize for all pallets (timestamp clears DidUpdate)
            - maybe_apply_pending_code_upgrade
            - frame_system::finalize():
              · Clean temporary storage
              · Calculate extrinsics root (merkle trie of all extrinsics)
              · Calculate storage root (host function → merkle patricia trie of all storage)
              · Return final header
    d. Seals the block (signs with collator's key from keystore)
    e. Imports locally (persist to RocksDB atomically via commit_operation):
       - Header, body, justifications → respective DB columns
       - Trie nodes → STATE column via state_db.insert_block
       - All in one atomic Transaction
    f. Extracts the storage proof (PoV)
11. Announces new block to peers
12. Sends CollatorMessage through channel to collation_task
13. collation_task sends block + PoV to relay chain validators
```

**What follows (not covered in this document)**: The relay chain validators receive the collation, re-execute the block using only the PoV (without having the complete state), verify the state root matches, and if validation passes, include it in the relay chain where it is finalized with GRANDPA.

---

# Part III: Supplementary Concepts

---

## 29. Weight System: The 2D Model

### Why Two Dimensions

Originally, Substrate used a single-dimension weight representing computation time (`ref_time`). However, for parachains, there's a second critical constraint: the **Proof of Validity (PoV) size**. Relay chain validators must download and verify the PoV, which is limited (currently ~5MB on Polkadot).

Every storage read during block execution adds data to the PoV. A transaction might be cheap computationally but read many storage items, bloating the proof. The 2D weight model captures both constraints.

### Weight Structure

**File**: `substrate/primitives/weights/src/weight_v2.rs`

```rust
#[derive(Encode, Decode, MaxEncodedLen, TypeInfo, Eq, PartialEq, Copy, Clone, Debug, Default)]
pub struct Weight {
    #[codec(compact)]
    /// The weight of computational time used based on some reference hardware.
    ref_time: u64,
    #[codec(compact)]
    /// The weight of storage space used by proof of validity.
    proof_size: u64,
}

impl Weight {
    /// Construct [`Weight`] from weight parts, namely reference time and proof size weights.
    pub const fn from_parts(ref_time: u64, proof_size: u64) -> Self {
        Self { ref_time, proof_size }
    }

    /// Returns true if any of `self`'s constituent weights is strictly greater than that of the
    /// `other`'s, otherwise returns false.
    pub const fn any_gt(self, other: Self) -> bool {
        self.ref_time > other.ref_time || self.proof_size > other.proof_size
    }
}
```

- **`ref_time`**: Picoseconds of reference machine execution time. 1 second = 10^12 ref_time.
- **`proof_size`**: Bytes of storage proof. Every storage read adds ~80+ bytes (node hashes in the merkle path).

### How Weights Are Determined

Weights come from **benchmarking**. The `#[pallet::weight]` attribute specifies the weight for each extrinsic:

```rust
#[pallet::call_index(0)]
#[pallet::weight(T::WeightInfo::transfer_allow_death())]
pub fn transfer_allow_death(
    origin: OriginFor<T>,
    dest: AccountIdLookupOf<T>,
    #[pallet::compact] value: T::Balance,
) -> DispatchResult { ... }
```

The `WeightInfo` trait is implemented by auto-generated code from benchmarks:

**File**: `substrate/frame/balances/src/weights.rs` (generated)

```rust
impl<T: frame_system::Config> WeightInfo for SubstrateWeight<T> {
    fn transfer_allow_death() -> Weight {
        Weight::from_parts(52_000_000, 6196)
            .saturating_add(T::DbWeight::get().reads(1))
            .saturating_add(T::DbWeight::get().writes(1))
    }
}
```

- `52_000_000` ref_time (52 microseconds)
- `6196` proof_size base
- Plus 1 read and 1 write from `DbWeight`

`DbWeight` defines the cost of storage operations:

```rust
pub struct RuntimeDbWeight {
    pub read: Weight,   // e.g., Weight { ref_time: 25_000_000, proof_size: 1024 }
    pub write: Weight,  // e.g., Weight { ref_time: 100_000_000, proof_size: 0 }
}
```

### CheckWeight Extension

**File**: `substrate/frame/system/src/extensions/check_weight.rs`

The `CheckWeight` transaction extension validates that the extrinsic fits in the block:

```rust
#[derive(Encode, Decode, Clone, Eq, PartialEq, TypeInfo)]
pub struct CheckWeight<T: Config + Send + Sync>(core::marker::PhantomData<T>);

impl<T: Config + Send + Sync> CheckWeight<T> {
    /// Checks if the current extrinsic does not exceed the maximum weight.
    fn check_extrinsic_weight(
        info: &DispatchInfoOf<T::RuntimeCall>,
        len: usize,
    ) -> Result<(), TransactionValidityError> {
        let max = T::BlockWeights::get().get(info.class).max_extrinsic;
        let total_weight_including_length =
            info.total_weight().saturating_add_proof_size(len as u64);
        match max {
            Some(max) if total_weight_including_length.any_gt(max) => {
                Err(InvalidTransaction::ExhaustsResources.into())
            },
            _ => Ok(()),
        }
    }

    /// Validates and returns the next block length.
    pub fn do_validate(
        info: &DispatchInfoOf<T::RuntimeCall>,
        len: usize,
    ) -> Result<(ValidTransaction, u32), TransactionValidityError> {
        let next_len = Self::check_block_length(info, len)?;
        Self::check_extrinsic_weight(info, len)?;
        Ok((Default::default(), next_len))
    }
}

impl<T: Config + Send + Sync> TransactionExtension<T::RuntimeCall> for CheckWeight<T> {
    const IDENTIFIER: &'static str = "CheckWeight";
    type Val = u32; // next block length

    fn validate(
        &self,
        origin: T::RuntimeOrigin,
        _call: &T::RuntimeCall,
        info: &DispatchInfoOf<T::RuntimeCall>,
        len: usize,
        _self_implicit: Self::Implicit,
        _inherited_implication: &impl Encode,
        _source: TransactionSource,
    ) -> ValidateResult<Self::Val, T::RuntimeCall> {
        let (validity, next_len) = Self::do_validate(info, len)?;
        Ok((validity, next_len, origin))
    }
}
```

The key check is `any_gt`: if **either** `ref_time` or `proof_size` exceeds the limit, the transaction is rejected.

### Block Limits Configuration

Block weights are configured per dispatch class:

**File**: Runtime configuration

```rust
parameter_types! {
    pub const BlockWeights: frame_system::limits::BlockWeights =
        frame_system::limits::BlockWeights::builder()
            .base_block(Weight::from_parts(5_000_000_000, 0))  // Base overhead
            .for_class(DispatchClass::Normal, |weights| {
                weights.max_total = Some(Weight::from_parts(
                    NORMAL_DISPATCH_RATIO * MAXIMUM_BLOCK_WEIGHT.ref_time(),
                    NORMAL_DISPATCH_RATIO * MAX_POV_SIZE,
                ));
            })
            .for_class(DispatchClass::Operational, |weights| {
                weights.max_total = Some(Weight::from_parts(
                    MAXIMUM_BLOCK_WEIGHT.ref_time(),
                    MAX_POV_SIZE,
                ));
            })
            .for_class(DispatchClass::Mandatory, |weights| {
                weights.max_total = None;  // No limit for mandatory (inherents)
            })
            .build_or_panic();
}
```

- **Normal**: User transactions. Typically 75% of block capacity.
- **Operational**: Privileged operations (governance, staking). Can use remaining 25%.
- **Mandatory**: Inherents. No limit — they must succeed or block production fails.

### Weight in Parachains: The PoV Constraint

For parachains, `proof_size` is directly tied to the relay chain's `max_pov_size` limit:

```rust
// In cumulus
pub const MAX_POV_SIZE: u32 = 5 * 1024 * 1024;  // 5 MB

// The block's proof_size budget
let available_proof_size = validation_data.max_pov_size * 0.85;  // 85% safety margin
```

If transactions collectively read enough storage to exceed the PoV limit, subsequent transactions are rejected even if there's `ref_time` remaining. This is why `apply_extrinsics` can exit with `HitBlockWeightLimit` — it checks both dimensions.

---

## 30. Events System

### What Are Events

Events are the runtime's way of communicating what happened during execution to the outside world. Unlike storage (which represents state), events are:

- **Transient**: Only exist during block execution, cleared for the next block
- **Append-only**: Can only be added, never modified
- **For external consumption**: Indexers, UIs, and other off-chain systems read them

### Depositing Events

Pallets emit events using the `deposit_event` function generated by the `#[pallet::event]` macro:

```rust
#[pallet::event]
#[pallet::generate_deposit(pub(super) fn deposit_event)]
pub enum Event<T: Config> {
    /// Transfer succeeded.
    Transfer { from: T::AccountId, to: T::AccountId, amount: T::Balance },
    /// An account was created with some free balance.
    Endowed { account: T::AccountId, free_balance: T::Balance },
}
```

Usage in a dispatchable:

```rust
pub fn transfer(origin: OriginFor<T>, dest: T::AccountId, value: T::Balance) -> DispatchResult {
    let sender = ensure_signed(origin)?;
    // ... transfer logic ...

    Self::deposit_event(Event::Transfer { from: sender, to: dest, amount: value });
    Ok(())
}
```

### Event Storage

**File**: `substrate/frame/system/src/lib.rs`

Events are stored in `frame_system`:

```rust
#[pallet::storage]
#[pallet::unbounded]
pub(super) type Events<T: Config> =
    StorageValue<_, Vec<Box<EventRecord<T::RuntimeEvent, T::Hash>>>, ValueQuery>;

/// Record of an event happening.
#[derive(Encode, Decode, Debug, TypeInfo)]
pub struct EventRecord<E: Parameter + Member, T> {
    /// The phase of the block it happened in.
    pub phase: Phase,
    /// The event itself.
    pub event: E,
    /// The list of the topics this event has.
    pub topics: Vec<T>,
}

pub enum Phase {
    /// Applying an extrinsic.
    ApplyExtrinsic(u32),  // Index of the extrinsic
    /// Finalizing the block.
    Finalization,
    /// Block initialization.
    Initialization,
}
```

The `deposit_event` implementation:

```rust
pub fn deposit_event(event: impl Into<T::RuntimeEvent>) {
    Self::deposit_event_indexed(&[], event.into())
}

pub fn deposit_event_indexed(topics: &[T::Hash], event: T::RuntimeEvent) {
    let phase = ExecutionPhase::<T>::get().unwrap_or(Phase::Finalization);
    let event_record = EventRecord { phase, event, topics: topics.to_vec() };

    Events::<T>::append(event_record);

    // Also store by topic for indexed lookup
    for topic in topics {
        EventTopics::<T>::append(topic, (block_number, event_index));
    }

    EventCount::<T>::mutate(|c| *c = c.saturating_add(1));
}
```

### The Event Lifecycle

1. **Block starts**: `reset_events()` clears `Events<T>` and `EventCount<T>`:

```rust
pub fn reset_events() {
    <Events<T>>::kill();
    EventCount::<T>::kill();
    let _ = <EventTopics<T>>::clear(u32::max_value(), None);
}
```

This happens in `initialize_block` before any execution.

2. **During execution**: Events accumulate in storage as extrinsics execute. The `phase` field records which extrinsic emitted each event.

3. **Block finalization**: Events remain in storage until `finalize()` returns. External systems can query them via RPC (`state_getStorage` on the Events key).

4. **Next block**: `reset_events()` wipes everything. Events are not persisted across blocks.

### System Events

`frame_system` emits special events for every extrinsic:

```rust
#[pallet::event]
pub enum Event<T: Config> {
    /// An extrinsic completed successfully.
    ExtrinsicSuccess { dispatch_info: DispatchInfo },
    /// An extrinsic failed.
    ExtrinsicFailed { dispatch_error: DispatchError, dispatch_info: DispatchInfo },
    /// `:code` was updated.
    CodeUpdated,
    /// A new account was created.
    NewAccount { account: T::AccountId },
    /// An account was reaped.
    KilledAccount { account: T::AccountId },
    /// On on-chain remark happened.
    Remarked { sender: T::AccountId, hash: T::Hash },
    /// An upgrade was authorized.
    UpgradeAuthorized { code_hash: T::Hash, check_version: bool },
}
```

`note_applied_extrinsic` emits `ExtrinsicSuccess` or `ExtrinsicFailed` after each extrinsic:

```rust
pub fn note_applied_extrinsic(r: &DispatchResultWithPostInfo, info: DispatchInfo) {
    Self::deposit_event(match r {
        Ok(_) => Event::ExtrinsicSuccess { dispatch_info: info },
        Err(err) => Event::ExtrinsicFailed { dispatch_error: err.error, dispatch_info: info },
    });
}
```

### Querying Events

**Via RPC** (during block execution or immediately after):

```bash
# Get raw events storage
curl -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"state_getStorage","params":["0x26aa394eea5630e07c48ae0c9558cef780d41e5e16056765bc8461851072c9d7"]}' \
  http://localhost:9933
```

**Via Polkadot.js API**:

```typescript
const events = await api.query.system.events();
events.forEach((record) => {
    const { event, phase } = record;
    console.log(`${event.section}.${event.method}`, phase.toString());
});
```

**Via subscription**:

```typescript
api.query.system.events((events) => {
    events.forEach((record) => { /* handle */ });
});
```

---

## 31. Header Structure and Digest

### The Header Fields

**File**: `substrate/primitives/runtime/src/generic/header.rs`

```rust
/// Abstraction over a block header for a substrate chain.
#[derive(Encode, Decode, PartialEq, Eq, Clone, Debug, TypeInfo)]
pub struct Header<Number: Copy + Into<U256> + TryFrom<U256>, Hash: HashT> {
    /// The parent hash.
    pub parent_hash: Hash::Output,
    /// The block number.
    #[codec(compact)]
    pub number: Number,
    /// The state trie merkle root
    pub state_root: Hash::Output,
    /// The merkle root of the extrinsics.
    pub extrinsics_root: Hash::Output,
    /// A chain-specific digest of data useful for light clients or referencing auxiliary data.
    pub digest: Digest,
}

impl<Number, Hash> Header<Number, Hash> {
    /// Convenience helper for computing the hash of the header.
    pub fn hash(&self) -> Hash::Output {
        Hash::hash_of(self)
    }
}
```

| Field | Size | Description |
|-------|------|-------------|
| `parent_hash` | 32 bytes | Hash of the parent block's header |
| `number` | Compact encoded | Block height (0, 1, 2, ...) |
| `state_root` | 32 bytes | Merkle patricia trie root of all storage |
| `extrinsics_root` | 32 bytes | Merkle trie root of all extrinsics |
| `digest` | Variable | Vector of digest items |

The **block hash** is the hash of the encoded header:

```rust
pub fn hash(&self) -> Hash::Output {
    <Hash as HashT>::hash_of(self)
}
```

### Digest Items

**File**: `substrate/primitives/runtime/src/generic/digest.rs`

```rust
pub struct Digest {
    pub logs: Vec<DigestItem>,
}

pub enum DigestItem {
    /// A pre-runtime digest item.
    PreRuntime(ConsensusEngineId, Vec<u8>),
    /// A consensus digest item.
    Consensus(ConsensusEngineId, Vec<u8>),
    /// A seal digest item.
    Seal(ConsensusEngineId, Vec<u8>),
    /// Runtime environment updated.
    RuntimeEnvironmentUpdated,
    /// Other.
    Other(Vec<u8>),
}

pub type ConsensusEngineId = [u8; 4];
```

- **PreRuntime**: Added before runtime executes. Contains data like the Aura slot.
- **Consensus**: Added during runtime execution. Contains consensus-relevant data.
- **Seal**: Added after runtime execution. Contains the block author's signature.
- **RuntimeEnvironmentUpdated**: Signals runtime code change.

### PreRuntime Digest: Aura Slot

For Aura consensus, the slot number is embedded in the header:

**File**: `substrate/primitives/consensus/aura/src/lib.rs`

```rust
pub const AURA_ENGINE_ID: ConsensusEngineId = *b"aura";

/// The Aura pre-digest.
#[derive(Clone, Debug, Encode, Decode)]
pub struct PreDigest {
    /// The slot number.
    pub slot: Slot,
}
```

Created during block building:

```rust
// In slot_claim construction (block_builder_task.rs)
let pre_digest = DigestItem::PreRuntime(
    AURA_ENGINE_ID,
    PreDigest { slot: para_slot.slot }.encode(),
);
```

This allows validators to verify the block was produced in the correct slot without re-executing.

### Seal Digest: Block Signature

The seal is the collator's signature over the header (minus the seal itself):

**File**: `substrate/client/consensus/aura/src/standalone.rs`

```rust
/// Produce the seal digest item by signing the hash of a block.
///
/// Note that after this is added to a block header, the hash of the block will change.
pub fn seal<Hash, P>(
    header_hash: &Hash,
    public: &P::Public,
    keystore: &KeystorePtr,
) -> Result<sp_runtime::DigestItem, ConsensusError>
where
    Hash: AsRef<[u8]>,
    P: Pair,
    P::Signature: Codec + TryFrom<Vec<u8>>,
    P::Public: AppPublic,
{
    let signature = keystore
        .sign_with(
            <AuthorityId<P> as AppCrypto>::ID,
            <AuthorityId<P> as AppCrypto>::CRYPTO_ID,
            public.as_slice(),
            header_hash.as_ref(),
        )
        .map_err(|e| ConsensusError::CannotSign(format!("{}. Key: {:?}", e, public)))?
        .ok_or_else(|| {
            ConsensusError::CannotSign(format!("Could not find key in keystore. Key: {:?}", public))
        })?;

    let signature = signature
        .as_slice()
        .try_into()
        .map_err(|_| ConsensusError::InvalidSignature(signature, public.to_raw_vec()))?;

    let signature_digest_item =
        <DigestItem as CompatibleDigestItem<P::Signature>>::aura_seal(signature);

    Ok(signature_digest_item)
}
```

The seal is added **after** `finalize_block` returns:

```rust
pub fn seal_block(block: Block, keystore: &KeystorePtr, author: &Public) -> Result<Block> {
    let (mut header, extrinsics) = block.deconstruct();

    // Hash the header WITHOUT the seal
    let header_hash = header.hash();

    // Create and append the seal
    let seal = seal::<_, P>(&header_hash, keystore, author)?;
    header.digest_mut().push(seal);

    Ok(Block::new(header, extrinsics))
}
```

Verification reverses this:

```rust
pub fn verify_seal(header: &Header) -> Result<(), Error> {
    // Pop the seal
    let seal = header.digest().logs().last()
        .ok_or(Error::NoSeal)?;

    // Reconstruct header without seal
    let mut header_without_seal = header.clone();
    header_without_seal.digest_mut().pop();

    // Verify signature
    let header_hash = header_without_seal.hash();
    verify_signature(seal, header_hash)?;

    Ok(())
}
```

### Consensus Digest

Runtime can emit consensus digests to signal changes to the consensus protocol:

```rust
// Example: Authority set change in Aura
pub fn schedule_change(new_authorities: Vec<AuthorityId>) {
    let log = DigestItem::Consensus(
        AURA_ENGINE_ID,
        ConsensusLog::AuthoritiesChange(new_authorities).encode(),
    );
    frame_system::Pallet::<T>::deposit_log(log);
}
```

Common consensus digests:
- **AuthoritiesChange**: New validator set
- **OnDisabled**: A validator is temporarily disabled
- **ForceChange**: Forced authority set change (for emergencies)

### How Digests Flow Through Block Production

```
1. Block builder task creates initial digest:
   └── PreRuntime(AURA, slot)

2. BlockBuilder::new() passes digest to initialize_block:
   └── Runtime stores digest in frame_system::Digest storage

3. During runtime execution:
   └── Pallets may add Consensus items via deposit_log()

4. finalize_block() returns header with digest

5. seal() adds the final item:
   └── Seal(AURA, signature)

Final digest structure:
[
    PreRuntime(AURA, slot_bytes),      // Added by node before runtime
    Consensus(AURA, authorities),       // Optional: added by runtime
    Seal(AURA, signature_bytes),        // Added by node after runtime
]
```

When importing a block, the order is validated:
- PreRuntime items must come first
- Seal must be last
- Multiple PreRuntime items are allowed (multi-engine scenarios)
- Seal is stripped before re-executing for verification