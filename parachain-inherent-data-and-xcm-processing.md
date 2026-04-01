# Parachain Inherent Data: From Relay Chain to Message Processing

## Table of Contents

- [Parachain Inherent Data: From Relay Chain to Message Processing](#parachain-inherent-data-from-relay-chain-to-message-processing)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
- [Part I: The Collator Fetches Data from the Relay Chain](#part-i-the-collator-fetches-data-from-the-relay-chain)
  - [1. Entry Point: create\_inherent\_data\_with\_rp\_offset](#1-entry-point-create_inherent_data_with_rp_offset)
  - [2. ParachainInherentDataProvider::create\_at](#2-parachaininherentdataprovidercreate_at)
  - [3. collect\_relay\_storage\_proof: Building the Merkle Proof](#3-collect_relay_storage_proof-building-the-merkle-proof)
  - [4. get\_static\_relay\_storage\_keys: What Keys Are Needed](#4-get_static_relay_storage_keys-what-keys-are-needed)
    - [Fixed Keys (Always Needed)](#fixed-keys-always-needed)
    - [Dynamic Keys (Per HRMP Channel)](#dynamic-keys-per-hrmp-channel)
    - [Optional Keys](#optional-keys)
  - [5. How the Collator Accesses the Relay Chain](#5-how-the-collator-accesses-the-relay-chain)
- [Part II: The Runtime Processes the Inherent](#part-ii-the-runtime-processes-the-inherent)
  - [6. set\_validation\_data: The Parachain System Inherent](#6-set_validation_data-the-parachain-system-inherent)
  - [7. Relay Chain State Proof Verification](#7-relay-chain-state-proof-verification)
  - [8. Runtime Upgrade Handling](#8-runtime-upgrade-handling)
  - [9. Storage Updates](#9-storage-updates)
  - [10. Processing DMP Messages: enqueue\_inbound\_downward\_messages](#10-processing-dmp-messages-enqueue_inbound_downward_messages)
    - [The Message Queue Chain (MQC)](#the-message-queue-chain-mqc)
    - [Full Messages vs Hashed Messages](#full-messages-vs-hashed-messages)
    - [Step 6 — Enqueueing](#step-6--enqueueing)
    - [Step 7 — Integrity Verification](#step-7--integrity-verification)
  - [11. Processing HRMP Messages: enqueue\_inbound\_horizontal\_messages](#11-processing-hrmp-messages-enqueue_inbound_horizontal_messages)
    - [Key Differences from DMP](#key-differences-from-dmp)
    - [Step 7 — Integrity Verification](#step-7--integrity-verification-1)
    - [Step 8 — Processing via XcmpMessageHandler](#step-8--processing-via-xcmpmessagehandler)
    - [Step 9 — HRMP Watermark](#step-9--hrmp-watermark)
  - [12. handle\_xcmp\_messages: Decoding and Enqueueing](#12-handle_xcmp_messages-decoding-and-enqueueing)
    - [Three Message Formats](#three-message-formats)
  - [13. The Message Queue: Where Messages Wait](#13-the-message-queue-where-messages-wait)
    - [Structure](#structure)
  - [14. When Do Messages Actually Execute?](#14-when-do-messages-actually-execute)
- [Part III: The Complete Message Flow](#part-iii-the-complete-message-flow)
  - [15. DMP vs HRMP: Key Differences](#15-dmp-vs-hrmp-key-differences)
  - [16. Complete Flow Diagram](#16-complete-flow-diagram)

---

## Introduction

This document covers how a parachain receives and processes messages from the relay chain and other parachains. It traces two connected flows:

1. **Node side (collator)**: How the collator fetches DMP messages, HRMP messages, and the relay chain state proof from its embedded relay chain node.
2. **Runtime side**: How `set_validation_data` verifies the data, enqueues the messages, and how they eventually get executed.

This is the "input" side of cross-chain communication — messages coming **into** the parachain. The "output" side (messages going out via UMP and HRMP) is handled during `on_finalize` and packaged in the `CandidateCommitments`.

---

# Part I: The Collator Fetches Data from the Relay Chain

---

## 1. Entry Point: create_inherent_data_with_rp_offset

**File**: `cumulus/client/consensus/aura/src/collators/mod.rs`

This function is called in step 9 of the block builder task's main loop, right before building the block:

```rust
pub async fn create_inherent_data_with_rp_offset(
    &self,
    relay_parent: PHash,
    validation_data: &PersistedValidationData,
    parent_hash: Block::Hash,
    timestamp: impl Into<Option<Timestamp>>,
    relay_parent_descendants: Option<RelayParentData>,
    relay_proof_request: RelayProofRequest,
    collator_peer_id: PeerId,
) -> Result<(ParachainInherentData, InherentData), Box<dyn Error + Send + Sync + 'static>> {
    // 1. Create the parachain inherent data (relay chain data)
    let paras_inherent_data = ParachainInherentDataProvider::create_at(
        relay_parent,
        &self.relay_client,
        validation_data,
        self.para_id,
        relay_parent_descendants
            .map(RelayParentData::into_inherent_descendant_list)
            .unwrap_or_default(),
        relay_proof_request,
        collator_peer_id,
    ).await;

    // 2. Create other inherent data (timestamp, etc.)
    let mut other_inherent_data = self
        .create_inherent_data_providers
        .create_inherent_data_providers(parent_hash, ())
        .await?
        .create_inherent_data()
        .await?;

    // 3. Replace timestamp if collator provides one
    if let Some(timestamp) = timestamp.into() {
        other_inherent_data.replace_data(sp_timestamp::INHERENT_IDENTIFIER, &timestamp);
    }

    Ok((paras_inherent_data, other_inherent_data))
}
```

This produces two separate outputs:

- **`paras_inherent_data`** (`ParachainInherentData`): Everything from the relay chain — state proof, DMP messages, HRMP messages, validation data. Becomes the `set_validation_data` inherent.
- **`other_inherent_data`** (`InherentData`): The timestamp and any other inherent providers. Becomes the `Timestamp::set` inherent.

---

## 2. ParachainInherentDataProvider::create_at

**File**: `cumulus/client/parachain-inherent/src/lib.rs`

This is where the collator actually **downloads data from the relay chain**:

```rust
pub async fn create_at(
    relay_parent: PHash,
    relay_chain_interface: &impl RelayChainInterface,
    validation_data: &PersistedValidationData,
    para_id: ParaId,
    relay_parent_descendants: Vec<RelayHeader>,
    relay_proof_request: RelayProofRequest,
    collator_peer_id: PeerId,
) -> Option<ParachainInherentData> {
    // 1. Collect the relay chain state proof
    let relay_chain_state = collect_relay_storage_proof(
        relay_chain_interface,
        para_id,
        relay_parent,
        !relay_parent_descendants.is_empty(),
        include_next_authorities,
        relay_proof_request,
    ).await?;

    // 2. Download DMP messages
    let downward_messages = relay_chain_interface
        .retrieve_dmq_contents(para_id, relay_parent)
        .await
        .ok()?;

    // 3. Download HRMP messages
    let horizontal_messages = relay_chain_interface
        .retrieve_all_inbound_hrmp_channel_contents(para_id, relay_parent)
        .await
        .ok()?;

    // 4. Package everything
    Some(ParachainInherentData {
        downward_messages,
        horizontal_messages,
        validation_data: validation_data.clone(),
        relay_chain_state,
        relay_parent_descendants,
        collator_peer_id,
    })
}
```

Three pieces of data are fetched:

**Step 1 — Relay chain state proof**: A merkle proof of specific storage keys from the relay chain. Not the entire state — just a proof for the keys the parachain needs. See [Section 3](#3-collect_relay_storage_proof-building-the-merkle-proof).

**Step 2 — DMP messages** (`retrieve_dmq_contents`): All pending messages in the **Downward Message Queue** for this parachain. These are messages from the relay chain (or governance) directed to this parachain.

**Step 3 — HRMP messages** (`retrieve_all_inbound_hrmp_channel_contents`): All pending messages in the **HRMP inbound channels** for this parachain. These are messages from other parachains, routed through the relay chain.

---

## 3. collect_relay_storage_proof: Building the Merkle Proof

**File**: `cumulus/client/parachain-inherent/src/lib.rs`

```rust
async fn collect_relay_storage_proof(
    relay_chain_interface: &impl RelayChainInterface,
    para_id: ParaId,
    relay_parent: PHash,
    include_authorities: bool,
    include_next_authorities: bool,
    relay_proof_request: RelayProofRequest,
) -> Option<StorageProof> {
    // 1. Get static keys (always needed)
    let mut all_top_keys = get_static_relay_storage_keys(
        relay_chain_interface, para_id, relay_parent,
        include_authorities, include_next_authorities,
    ).await?;

    // 2. Add dynamic keys requested by the collator
    let RelayProofRequest { keys } = relay_proof_request;
    let mut child_keys: HashMap<Vec<u8>, HashSet<Vec<u8>>> = HashMap::new();
    for key in keys {
        match key {
            RelayStorageKey::Top(k) => { all_top_keys.insert(k); },
            RelayStorageKey::Child { storage_key, key } => {
                child_keys.entry(storage_key).or_default().insert(key);
            },
        }
    }

    // 3. Generate proof for top-level storage
    let mut all_proofs = Vec::new();
    let top_keys_vec: Vec<Vec<u8>> = all_top_keys.into_iter().collect();
    match relay_chain_interface.prove_read(relay_parent, &top_keys_vec).await {
        Ok(top_proof) => { all_proofs.push(top_proof); },
        Err(e) => { return None; },
    }

    // 4. Generate proofs for child tries
    for (storage_key, data_keys) in child_keys {
        let child_info = ChildInfo::new_default(&storage_key);
        let data_keys_vec: Vec<Vec<u8>> = data_keys.into_iter().collect();
        match relay_chain_interface
            .prove_child_read(relay_parent, &child_info, &data_keys_vec)
            .await
        {
            Ok(child_proof) => { all_proofs.push(child_proof); },
            Err(e) => { /* log error */ },
        }
    }

    // 5. Merge all proofs into one
    Some(StorageProof::merge(all_proofs))
}
```

- **Step 1**: Gets the static keys that are always needed. See [Section 4](#4-get_static_relay_storage_keys-what-keys-are-needed).
- **Step 2**: Adds any additional keys the collator requested (top keys and child trie keys).
- **Step 3**: `prove_read` asks the relay chain node: "give me a merkle proof that demonstrates the values of these keys at block `relay_parent`". The relay chain returns a `StorageProof` containing only the trie nodes needed to verify those keys against the `relay_parent_storage_root`. It's not the entire state — it's a minimal proof.
- **Step 4**: Same for child tries (e.g., contract storage or parachain-specific storage on the relay chain).
- **Step 5**: Merges all proofs (top + child tries) into a single `StorageProof`. This is the `relay_chain_state` that travels inside the `ParachainInherentData`.

---

## 4. get_static_relay_storage_keys: What Keys Are Needed

**File**: `cumulus/client/parachain-inherent/src/lib.rs`

```rust
async fn get_static_relay_storage_keys(
    relay_chain_interface: &impl RelayChainInterface,
    para_id: ParaId,
    relay_parent: PHash,
    include_authorities: bool,
    include_next_authorities: bool,
) -> Option<HashSet<Vec<u8>>> {
    // First, fetch the channel indexes to know which channels exist
    let ingress_channels = relay_chain_interface
        .get_storage_by_key(relay_parent,
            &relay_well_known_keys::hrmp_ingress_channel_index(para_id))
        .await.ok()?;
    let egress_channels = relay_chain_interface
        .get_storage_by_key(relay_parent,
            &relay_well_known_keys::hrmp_egress_channel_index(para_id))
        .await.ok()?;

    // Fixed keys - always needed
    let mut relevant_keys: HashSet<Vec<u8>> = HashSet::from([
        relay_well_known_keys::CURRENT_BLOCK_RANDOMNESS.to_vec(),
        relay_well_known_keys::ONE_EPOCH_AGO_RANDOMNESS.to_vec(),
        relay_well_known_keys::TWO_EPOCHS_AGO_RANDOMNESS.to_vec(),
        relay_well_known_keys::CURRENT_SLOT.to_vec(),
        relay_well_known_keys::ACTIVE_CONFIG.to_vec(),
        relay_well_known_keys::dmq_mqc_head(para_id),
        relay_well_known_keys::relay_dispatch_queue_size(para_id),
        relay_well_known_keys::relay_dispatch_queue_remaining_capacity(para_id).key,
        relay_well_known_keys::hrmp_ingress_channel_index(para_id),
        relay_well_known_keys::hrmp_egress_channel_index(para_id),
        relay_well_known_keys::upgrade_go_ahead_signal(para_id),
        relay_well_known_keys::upgrade_restriction_signal(para_id),
        relay_well_known_keys::para_head(para_id),
    ]);

    // Dynamic keys - per HRMP channel
    relevant_keys.extend(ingress_channels.into_iter().map(|sender| {
        relay_well_known_keys::hrmp_channels(HrmpChannelId { sender, recipient: para_id })
    }));
    relevant_keys.extend(egress_channels.into_iter().map(|recipient| {
        relay_well_known_keys::hrmp_channels(HrmpChannelId { sender: para_id, recipient })
    }));

    // Optional keys - authorities
    if include_authorities {
        relevant_keys.insert(relay_well_known_keys::AUTHORITIES.to_vec());
    }
    if include_next_authorities {
        relevant_keys.insert(relay_well_known_keys::NEXT_AUTHORITIES.to_vec());
    }

    Some(relevant_keys)
}
```

### Fixed Keys (Always Needed)

| Key | Purpose |
|-----|---------|
| `CURRENT_BLOCK_RANDOMNESS` | Current epoch randomness for pallets needing verifiable randomness |
| `ONE_EPOCH_AGO_RANDOMNESS` | Previous epoch randomness |
| `TWO_EPOCHS_AGO_RANDOMNESS` | Two epochs ago randomness |
| `CURRENT_SLOT` | Current relay chain slot |
| `ACTIVE_CONFIG` | Host configuration (message limits, max PoV size, etc.) |
| `dmq_mqc_head(para_id)` | DMQ Message Queue Chain head — for verifying DMP message integrity |
| `relay_dispatch_queue_size(para_id)` | UMP queue size (legacy) |
| `relay_dispatch_queue_remaining_capacity(para_id)` | How much space remains for UMP messages |
| `hrmp_ingress_channel_index(para_id)` | List of parachains with inbound HRMP channels to us |
| `hrmp_egress_channel_index(para_id)` | List of parachains with outbound HRMP channels from us |
| `upgrade_go_ahead_signal(para_id)` | Whether relay chain approves a pending runtime upgrade |
| `upgrade_restriction_signal(para_id)` | Whether relay chain prohibits upgrades right now |
| `para_head(para_id)` | Our parachain's head data as known by the relay chain |

### Dynamic Keys (Per HRMP Channel)

For each inbound channel (other para → us) and outbound channel (us → other para), the channel details are requested: capacity, max size, message count, MQC head. These are dynamic because they depend on which channels this parachain has open.

### Optional Keys

- `AUTHORITIES`: Current BABE authorities. Only when needed.
- `NEXT_AUTHORITIES`: Next epoch authorities. Only when an epoch change is detected in the relay parent descendants.

---

## 5. How the Collator Accesses the Relay Chain

The `relay_chain_interface` used in all these functions is **not a remote HTTP call**. It's a local interface to the relay chain node that the collator runs internally.

The collator runs a **full node of the relay chain** inside the same process. It has the complete relay chain database in RocksDB locally, synchronizes blocks, and participates in the relay chain's P2P network (but doesn't validate — it has no validator keys).

When the collator starts, you see it synchronizing two chains: the parachain and the relay chain. That's why collators use significant disk space and memory — they run two nodes simultaneously.

There are two modes:

- **In-process** (most common in production): The collator runs a full relay chain node in the same process. `retrieve_dmq_contents`, `prove_read`, etc. are direct reads from local RocksDB. No network latency.
- **RPC provider**: The collator connects to an external relay chain node via WebSocket RPC. Uses less resources but introduces latency and dependency on an external node.

The `RelayChainInterface` trait abstracts both modes — the collator code doesn't know or care which one is being used.

---

# Part II: The Runtime Processes the Inherent

---

## 6. set_validation_data: The Parachain System Inherent

**File**: `cumulus/pallets/parachain-system/src/lib.rs`

When the inherent is applied during block construction, this function executes inside the runtime:

```rust
pub fn set_validation_data(
    origin: OriginFor<T>,
    data: BasicParachainInherentData,
    inbound_messages_data: InboundMessagesData,
) -> DispatchResult {
    // 1. Basic checks
    ensure_none(origin)?;
    assert!(
        !<ValidationData<T>>::exists(),
        "ValidationData must be updated only once in a block",
    );

    // 2. Unpack the data
    let BasicParachainInherentData {
        validation_data: vfp,
        relay_chain_state,
        relay_parent_descendants,
        collator_peer_id,
    } = data;

    // 3. Verify relay block number advances
    T::CheckAssociatedRelayNumber::check_associated_relay_number(
        vfp.relay_parent_number,
        LastRelayChainBlockNumber::<T>::get(),
    );

    // 4. Verify the state proof
    let relay_state_proof = RelayChainStateProof::new(
        T::SelfParaId::get(),
        vfp.relay_parent_storage_root,
        relay_chain_state.clone(),
    ).expect("Invalid relay chain state proof");

    // 5. Verify relay parent descendants (for low-latency)
    // 6. Consensus hook and unincluded segment
    // 7. Process runtime upgrade signals
    // 8. Read host config and messaging state from proof
    // 9. Store everything in storage
    // 10. Process DMP messages
    // 11. Process HRMP messages

    Ok(())
}
```

Like `Timestamp::set`, this is an inherent: no signature (`ensure_none`), runs once per block, and is mandatory.

---

## 7. Relay Chain State Proof Verification

```rust
let relay_state_proof = RelayChainStateProof::new(
    T::SelfParaId::get(),
    vfp.relay_parent_storage_root,
    relay_chain_state.clone(),
).expect("Invalid relay chain state proof");
```

This is where the merkle proof is **cryptographically verified**. The `relay_parent_storage_root` comes from the `PersistedValidationData` (which is attested by the relay chain validators). The `relay_chain_state` is the proof the collator assembled in `collect_relay_storage_proof`.

If the collator altered any data in the proof, it won't match the root and the runtime **panics**. This is what makes the parachain trustless with respect to the collator for relay chain data — the collator could lie, but the proof would expose it.

After verification, the proof is used to read various values:

```rust
let upgrade_go_ahead_signal = relay_state_proof
    .read_upgrade_go_ahead_signal()
    .expect("Invalid upgrade go ahead signal");

let host_config = relay_state_proof
    .read_abridged_host_configuration()
    .expect("Invalid host configuration in relay chain state proof");

let relevant_messaging_state = relay_state_proof
    .read_messaging_state_snapshot(&host_config)
    .expect("Invalid messaging state in relay chain state proof");
```

Each `read_*` call extracts a value from the verified proof. These reads are guaranteed to be authentic because the proof was verified against the relay parent storage root.

---

## 8. Runtime Upgrade Handling

```rust
match upgrade_go_ahead_signal {
    Some(relay_chain::UpgradeGoAhead::GoAhead) => {
        assert!(
            <PendingValidationCode<T>>::exists(),
            "No new validation function found in storage, GoAhead signal is not expected",
        );
        let validation_code = <PendingValidationCode<T>>::take();
        frame_system::Pallet::<T>::update_code_in_storage(&validation_code);
        <T::OnSystemEvent as OnSystemEvent>::on_validation_code_applied();
        Self::deposit_event(Event::ValidationFunctionApplied {
            relay_chain_block_num: vfp.relay_parent_number,
        });
    },
    Some(relay_chain::UpgradeGoAhead::Abort) => {
        <PendingValidationCode<T>>::kill();
        Self::deposit_event(Event::ValidationFunctionDiscarded);
    },
    None => {},
}
```

The relay chain controls when a parachain can upgrade its runtime:

- **GoAhead**: The relay chain authorized the upgrade. The pending WASM code is applied via `update_code_in_storage`. Depending on the system version, this either writes directly to `:code` (immediate, legacy) or to `:pending_code` with a 2-block delay (current).
- **Abort**: The relay chain rejected the upgrade. The pending code is discarded.
- **None**: No upgrade signal. Business as usual.

This coordination is necessary because the relay chain validators need to know which WASM to use when re-executing parachain blocks for verification.

---

## 9. Storage Updates

```rust
<ValidationData<T>>::put(&vfp);
<RelayStateProof<T>>::put(relay_chain_state);
<RelevantMessagingState<T>>::put(relevant_messaging_state.clone());
<HostConfiguration<T>>::put(host_config);
```

All the verified data is stored so other pallets can access it during the block:

- **ValidationData**: The `PersistedValidationData` (relay parent number, storage root, max PoV size, parent head).
- **RelayStateProof**: The raw proof, in case any pallet needs to read additional keys.
- **RelevantMessagingState**: Channel capacities, queue sizes, MQC heads — used during `on_finalize` for sending outbound messages.
- **HostConfiguration**: Relay chain limits (max message sizes, max candidates, etc.).

---

## 10. Processing DMP Messages: enqueue_inbound_downward_messages

```rust
total_weight.saturating_accrue(Self::enqueue_inbound_downward_messages(
    relevant_messaging_state.dmq_mqc_head,
    inbound_messages_data.downward_messages,
));
```

**File**: `cumulus/pallets/parachain-system/src/lib.rs`

```rust
fn enqueue_inbound_downward_messages(
    expected_dmq_mqc_head: relay_chain::Hash,
    downward_messages: AbridgedInboundDownwardMessages,
) -> Weight {
    // 1. Verify enough messages were included
    downward_messages.check_enough_messages_included_basic("DMQ");

    // 2. Get current MQC head
    let mut dmq_head = <LastDmqMqcHead<T>>::get();

    // 3. Separate full messages and hashed messages
    let (messages, hashed_messages) = downward_messages.messages();
    let message_count = messages.len() as u32;

    if let Some(last_msg) = messages.last() {
        Self::deposit_event(Event::DownwardMessagesReceived { count: message_count });

        // 4. Update MQC head with full messages
        for msg in messages {
            dmq_head.extend_downward(msg);
        }
        <LastDmqMqcHead<T>>::put(&dmq_head);

        // 5. Update MQC head with hashed messages
        for msg in hashed_messages {
            dmq_head.extend_with_hashed_msg(msg);
        }

        // 6. Enqueue messages for processing
        T::DmpQueue::handle_messages(downward_messages.bounded_msgs_iter());
    }

    // 7. Verify integrity — THE CRITICAL CHECK
    assert_eq!(dmq_head.head(), expected_dmq_mqc_head, "DMQ head mismatch");

    ProcessedDownwardMessages::<T>::put(message_count);
    weight_used
}
```

### The Message Queue Chain (MQC)

The MQC is a hash chain: each message is hashed together with the previous hash, forming a chain: `hash(prevHash + msg)`. This allows verifying that no message was altered, omitted, or improperly added.

### Full Messages vs Hashed Messages

Messages come in two forms:
- **Full messages**: With complete payload. Can be executed.
- **Hashed messages**: Only the hash, no payload. Included to maintain the MQC chain without inflating the PoV with large messages.

Both are used to update the MQC head, but only full messages are enqueued for execution.

### Step 6 — Enqueueing

```rust
T::DmpQueue::handle_messages(downward_messages.bounded_msgs_iter());
```

`T::DmpQueue` is typically `pallet-message-queue`. The messages are **enqueued**, not executed immediately. They are stored as pages in the message queue's storage.

### Step 7 — Integrity Verification

```rust
assert_eq!(dmq_head.head(), expected_dmq_mqc_head, "DMQ head mismatch");
```

`expected_dmq_mqc_head` comes from the relay state proof (cryptographically verified). If the collator altered, omitted, or added any DMP message, the accumulated hash won't match and the block **panics**. Validators re-executing the block would get the same panic and reject the candidate.

This means the collator **cannot censor DMP messages**. If the relay chain says there are pending messages, they must be in the inherent.

---

## 11. Processing HRMP Messages: enqueue_inbound_horizontal_messages

```rust
total_weight.saturating_accrue(Self::enqueue_inbound_horizontal_messages(
    &relevant_messaging_state.ingress_channels,
    inbound_messages_data.horizontal_messages,
    vfp.relay_parent_number,
));
```

**File**: `cumulus/pallets/parachain-system/src/lib.rs`

```rust
fn enqueue_inbound_horizontal_messages(
    ingress_channels: &[(ParaId, AbridgedHrmpChannel)],
    horizontal_messages: AbridgedInboundHrmpMessages,
    relay_parent_number: relay_chain::BlockNumber,
) -> Weight {
    // 1. Get current MQC heads (one per channel)
    let mut mqc_heads = <LastHrmpMqcHeads<T>>::get();

    // 2. Separate full and hashed messages
    let (messages, hashed_messages) = horizontal_messages.messages();

    // 3. Verify advancement rule
    // 4. Prune closed channels
    Self::prune_closed_mqc_heads(ingress_channels, &mut mqc_heads);

    // 5. Process full messages
    for (sender, msg) in messages {
        Self::check_hrmp_message_metadata(
            ingress_channels, &mut prev_msg_metadata, (msg.sent_at, *sender),
        );
        mqc_heads.entry(*sender).or_default().extend_hrmp(msg);
    }
    LastHrmpMqcHeads::<T>::put(&mqc_heads);

    // 6. Process hashed messages
    for (sender, msg) in hashed_messages {
        Self::check_hrmp_message_metadata(
            ingress_channels, &mut prev_msg_metadata, (msg.sent_at, *sender),
        );
        mqc_heads.entry(*sender).or_default().extend_with_hashed_msg(msg);
    }

    // 7. Verify integrity of ALL channels
    Self::check_hrmp_mcq_heads(ingress_channels, &mut mqc_heads);

    // 8. Execute via XCMP handler
    let max_weight =
        <ReservedXcmpWeightOverride<T>>::get().unwrap_or_else(T::ReservedXcmpWeight::get);
    let weight_used = T::XcmpMessageHandler::handle_xcmp_messages(
        horizontal_messages.flat_msgs_iter(),
        max_weight,
    );

    // 9. Update watermark
    HrmpWatermark::<T>::put(last_processed_block);

    weight_used
}
```

### Key Differences from DMP

- **Multiple channels**: HRMP has one MQC head per sender parachain, stored in `LastHrmpMqcHeads` as a `BTreeMap<ParaId, MessageQueueChain>`.
- **Message ordering**: Messages must be ordered by `(sent_at, sender)`. `check_hrmp_message_metadata` verifies this and panics on violation.
- **Channel existence**: Each message's sender must have an open channel. `get_ingress_channel_or_panic` verifies this.
- **Closed channels**: `prune_closed_mqc_heads` removes MQC heads for channels that no longer exist.

### Step 7 — Integrity Verification

```rust
Self::check_hrmp_mcq_heads(ingress_channels, &mut mqc_heads);
```

Same principle as DMP but per channel: for each ingress channel, the accumulated MQC head must match what the relay chain reports. If any channel's messages were tampered with, it panics.

### Step 8 — Processing via XcmpMessageHandler

```rust
let weight_used = T::XcmpMessageHandler::handle_xcmp_messages(
    horizontal_messages.flat_msgs_iter(),
    max_weight,
);
```

`T::XcmpMessageHandler` is typically `pallet-xcmp-queue`. Unlike DMP where messages are enqueued for later, HRMP messages are processed here with a weight budget. Messages that don't fit in the budget are enqueued for future blocks.

### Step 9 — HRMP Watermark

```rust
HrmpWatermark::<T>::put(last_processed_block);
```

The watermark tells the relay chain "I processed all HRMP messages up to this relay block number." The relay chain uses this to discard processed messages from its queues. This value is included in the `CandidateCommitments`.

---

## 12. handle_xcmp_messages: Decoding and Enqueueing

**File**: `cumulus/pallets/xcmp-queue/src/lib.rs`

```rust
fn handle_xcmp_messages<'a, I: Iterator<Item = (ParaId, RelayBlockNumber, &'a [u8])>>(
    iter: I,
    max_weight: Weight,
) -> Weight {
    let mut meter = WeightMeter::with_limit(max_weight);

    for (sender, _sent_at, mut data) in iter {
        let format = XcmpMessageFormat::decode(&mut data);

        match format {
            // Control signals: suspend/resume channel
            XcmpMessageFormat::Signals => {
                while !data.is_empty() {
                    match ChannelSignal::decode(&mut data) {
                        Ok(ChannelSignal::Suspend) => Self::suspend_channel(sender),
                        Ok(ChannelSignal::Resume) => Self::resume_channel(sender),
                        Err(_) => break,
                    }
                }
            },

            // XCM messages: decode and enqueue
            XcmpMessageFormat::ConcatenatedVersionedXcm |
            XcmpMessageFormat::ConcatenatedOpaqueVersionedXcm => {
                while can_process_next_batch {
                    let batch = Self::take_first_concatenated_xcms(
                        &mut data, encoding, XCM_BATCH_SIZE, &mut meter,
                    );
                    if batch.is_empty() { break; }
                    Self::enqueue_xcmp_messages(
                        sender, &batch, is_first_sender_batch, &mut meter,
                    );
                }
            },

            // Legacy format: dropped
            XcmpMessageFormat::ConcatenatedEncodedBlob => {
                continue;
            },
        }
    }
    meter.consumed()
}
```

### Three Message Formats

**Signals**: Channel control flow messages. `Suspend` tells the sender to stop sending (the receiving chain is overwhelmed). `Resume` lifts the suspension. These are processed immediately, not enqueued.

**ConcatenatedVersionedXcm / ConcatenatedOpaqueVersionedXcm**: Real XCM messages. A single HRMP message can contain **multiple XCM messages concatenated**. They are decoded in batches and **enqueued** in `pallet-message-queue` via `enqueue_xcmp_messages`. Not executed immediately.

**ConcatenatedEncodedBlob**: Legacy format. Dropped.

---

## 13. The Message Queue: Where Messages Wait

**File**: `substrate/frame/message-queue/src/lib.rs`

Both DMP and HRMP messages end up in `pallet-message-queue`. The storage uses a **page-based system**:

```rust
fn do_enqueue_messages<'a>(
    origin: &MessageOriginOf<T>,
    messages: impl Iterator<Item = BoundedSlice<'a, u8, MaxMessageLenOf<T>>>,
) {
    let mut book_state = BookStateFor::<T>::get(origin);
    let mut maybe_page = None;

    for message in messages {
        // Try to append to current page
        if let Some(mut page) = maybe_page {
            maybe_page = match page.try_append_message::<T>(message) {
                Ok(_) => Some(page),
                Err(_) => {
                    Pages::<T>::insert(origin, book_state.end - 1, page);
                    None
                },
            }
        }
        // If doesn't fit, create new page
        if maybe_page.is_none() {
            book_state.end.saturating_inc();
            book_state.count.saturating_inc();
            maybe_page = Some(Page::from_message::<T>(message));
        }
        book_state.message_count.saturating_inc();
        book_state.size.saturating_accrue(message.len() as u64);
    }

    // Add to the ready ring if not already there
    if book_state.ready_neighbours.is_none() {
        book_state.ready_neighbours = Some(Self::ready_ring_knit(origin));
    }

    BookStateFor::<T>::insert(origin, book_state);
}
```

### Structure

- **Pages**: Messages are grouped into pages. Each page holds multiple messages.
- **BookState**: Each origin (DMP, HRMP from para X, etc.) has its own book tracking pages: `begin`, `end`, `count`, `message_count`, `size`.
- **Ready Ring**: A circular linked list of origins that have pending messages. Used for round-robin processing.

At this point, messages are just stored in pages. Not executed yet.

---

## 14. When Do Messages Actually Execute?

Messages execute **later in the same block** (if there's weight available) or in **future blocks**. The execution happens in `on_idle`, which runs during `finalize_block`:

```
finalize_block:
    note_finished_extrinsics
    → PostTransactions
    → on_idle          ← HERE: pallet-message-queue processes its queue
    → on_finalize
    → finalize
```

During `on_idle`, `pallet-message-queue`:

1. Iterates the **ready ring** (round-robin across origins).
2. For each origin, takes messages from its pages.
3. Processes each message with `T::MessageProcessor` (the **XcmExecutor**).
4. The XcmExecutor decodes the XCM instructions and executes them (token transfers, teleports, reserve transfers, governance calls, etc.).
5. Stops when the weight budget runs out.

Messages that don't fit in the current block's weight budget remain in the queue and are processed in the next block's `on_idle`.

---

# Part III: The Complete Message Flow

---

## 15. DMP vs HRMP: Key Differences

| Aspect | DMP | HRMP |
|--------|-----|------|
| Direction | Relay chain → parachain | Other parachain → this parachain |
| Route | Direct (relay has a queue per parachain) | Via relay chain (relay-routed) |
| Channels | Single queue (DMQ) | One channel per sender parachain |
| MQC heads | One | One per channel |
| Processing in set_validation_data | Enqueued via `T::DmpQueue` | Processed via `T::XcmpMessageHandler` with weight budget |
| Actual execution | In `on_idle` via message queue | In `on_idle` via message queue |
| Censorship protection | MQC head check | MQC head check per channel |
| Watermark | Not applicable | HRMP watermark updated |

---

## 16. Complete Flow Diagram

```
═══════════════════════════════════════════════════════════════════
                    NODE SIDE (COLLATOR)
═══════════════════════════════════════════════════════════════════

  Relay chain full node (embedded in collator)
       │
       ├── get_static_relay_storage_keys()
       │       → defines which relay chain storage keys are needed
       │
       ├── collect_relay_storage_proof()
       │       → prove_read() generates merkle proof for those keys
       │
       ├── retrieve_dmq_contents()
       │       → downloads pending DMP messages from relay chain
       │
       └── retrieve_all_inbound_hrmp_channel_contents()
               → downloads pending HRMP messages from all channels

       All packaged into ParachainInherentData
               │
               ▼

═══════════════════════════════════════════════════════════════════
                    RUNTIME SIDE (WASM)
═══════════════════════════════════════════════════════════════════

  set_validation_data (inherent execution)
       │
       ├── 1. RelayChainStateProof::new()
       │       → verifies merkle proof against relay_parent_storage_root
       │       → if collator altered any data, PANICS
       │
       ├── 2. Read from verified proof:
       │       → host config, messaging state, upgrade signals
       │
       ├── 3. Process runtime upgrade (GoAhead/Abort)
       │
       ├── 4. Store validation data, config, messaging state
       │
       ├── 5. enqueue_inbound_downward_messages (DMP)
       │       │
       │       ├── Update MQC head hash chain
       │       ├── assert MQC == relay chain's MQC (anti-censorship)
       │       └── T::DmpQueue::handle_messages()
       │               └── pallet-message-queue::enqueue_message()
       │                       └── stored in Pages (not executed yet)
       │
       └── 6. enqueue_inbound_horizontal_messages (HRMP)
               │
               ├── Update MQC head per channel
               ├── Verify message ordering (sent_at, sender)
               ├── Verify sender has open channel
               ├── assert MQC per channel == relay chain's MQC
               └── T::XcmpMessageHandler::handle_xcmp_messages()
                       ├── Signals → suspend/resume channel
                       └── XCM → enqueue_xcmp_messages()
                               └── pallet-message-queue (stored, not executed)

  ... later in the same block ...

  finalize_block → on_idle
       │
       └── pallet-message-queue service loop
               │
               ├── Round-robin across origins (DMP, HRMP channels)
               ├── Take messages from pages
               ├── T::MessageProcessor (XcmExecutor)
               │       └── Decode and execute XCM instructions
               │               (transfers, teleports, governance, etc.)
               └── Stop when weight budget exhausted
                   (remaining messages processed in future blocks)
```