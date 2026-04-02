# Collation Submission Pipeline: From Candidate Receipt to Validator Advertisement

> **Part of [Section 25: Collation Submission Pipeline](./parachain-block-lifecycle.md#25-collation-submission-pipeline-from-candidate-receipt-to-validator-advertisement)** in the Parachain Block Lifecycle document.

---

This section covers the second half of the collator's work, starting after the block has been built, sealed, imported locally, and sent through the async channel to the collation task. It traces the code path from `handle_submit_collation` through candidate receipt construction, erasure root calculation, validator discovery, collation advertisement via P2P, and PoV delivery on request.

All of this code still runs **inside the collator process**. The collator runs an embedded relay chain node with its own Overseer and subsystems. The point where data actually leaves the collator and travels over the network to validators is identified explicitly.

## Table of Contents

- [25.1. The Overseer: Central Message Router](#251-the-overseer-central-message-router)
- [25.2. CollationGenerationSubsystem: Receiving SubmitCollation](#252-collationgenerationsubsystem-receiving-submitcollation)
- [25.3. handle_submit_collation: Preparing the Collation](#253-handle_submit_collation-preparing-the-collation)
- [25.4. construct_and_distribute_receipt: Building the Candidate](#254-construct_and_distribute_receipt-building-the-candidate)
- [25.5. Erasure Coding and the Erasure Root](#255-erasure-coding-and-the-erasure-root)
- [25.6. CollatorProtocolSubsystem: Distributing the Collation](#256-collatorprotocolsubsystem-distributing-the-collation)
- [25.7. Validator Discovery: determine_our_validators](#257-validator-discovery-determine_our_validators)
- [25.8. Advertising to Validators: advertise_collation](#258-advertising-to-validators-advertise_collation)
- [25.9. PoV Delivery: The Request/Response Protocol](#259-pov-delivery-the-requestresponse-protocol)
- [25.10. The Core Index: How the Collator Knows Its Core](#2510-the-core-index-how-the-collator-knows-its-core)
- [25.11. Collator Consensus and Slot Assignment](#2511-collator-consensus-and-slot-assignment)
  - [25.11.1. When Does the Consensus Mechanism Activate?](#25111-when-does-the-consensus-mechanism-activate)
  - [25.11.2. How Aura Slot Assignment Works](#25112-how-aura-slot-assignment-works)
  - [25.11.3. The claim_slot Function: Round-Robin Assignment](#25113-the-claim_slot-function-round-robin-assignment)
  - [25.11.4. Where Are Authorities and Keys Stored?](#25114-where-are-authorities-and-keys-stored)
  - [25.11.5. When Does Competition Actually Happen?](#25115-when-does-competition-actually-happen)
  - [25.11.6. Alternative Consensus Mechanisms](#25116-alternative-consensus-mechanisms)
  - [25.11.7. Why Multiple Collators?](#25117-why-multiple-collators)
- [25.12. Summary: Complete Collator Pipeline](#2512-summary-complete-collator-pipeline)

---

## 25.1. The Overseer: Central Message Router

**File**: `polkadot/node/overseer/src/lib.rs`

The Overseer is the central orchestrator of all subsystems in the Polkadot node. It is defined using the `#[orchestra]` macro, which generates all the wiring automatically:

```rust
#[orchestra(
    gen=AllMessages,
    event=Event,
    signal=OverseerSignal,
    error=SubsystemError,
    message_capacity=2048,
)]
pub struct Overseer<SupportsParachains> {
    #[subsystem(CollationGenerationMessage, sends: [
        RuntimeApiMessage,
        CollatorProtocolMessage,
    ])]
    collation_generation: CollationGeneration,

    #[subsystem(CollatorProtocolMessage, sends: [
        NetworkBridgeTxMessage,
        RuntimeApiMessage,
        CandidateBackingMessage,
        ChainApiMessage,
        ProspectiveParachainsMessage,
    ])]
    collator_protocol: CollatorProtocol,

    // ... many more subsystems (candidate_validation, availability_store, etc.)
}
```

For each subsystem declared with `#[subsystem]`, the macro generates:
- A channel to receive messages of the specified type (e.g., `CollationGenerationMessage`).
- The ability to send the messages listed in `sends`.
- Code that calls `start()` on each subsystem when the Overseer launches.

The Overseer's main loop receives events and routes them:

```rust
async fn run_inner(mut self) -> SubsystemResult<()> {
    loop {
        select! {
            msg = self.events_rx.select_next_some() => {
                match msg {
                    Event::MsgToSubsystem { msg, origin, priority } => {
                        self.route_message(msg.into(), origin).await?;
                    }
                    // ...
                }
            },
        }
    }
}
```

When the collator sends `SubmitCollation` via `overseer_handle.send_msg(...)`, it arrives as `Event::MsgToSubsystem`. The Overseer calls `route_message`, which looks at the message type (`CollationGenerationMessage`) and routes it to the `collation_generation` subsystem.

The collator runs an embedded relay chain node internally, which has its own Overseer with all these subsystems. So the `CollationGenerationSubsystem` and `CollatorProtocolSubsystem` run **inside the collator process**, not on a remote validator.

---

## 25.2. CollationGenerationSubsystem: Receiving SubmitCollation

**File**: `polkadot/node/collation-generation/src/lib.rs`

The subsystem starts via the `start()` method called by the Overseer, which runs the main loop:

```rust
async fn run<Context>(mut self, mut ctx: Context) {
    loop {
        select! {
            incoming = ctx.recv().fuse() => {
                if self.handle_incoming::<Context>(incoming, &mut ctx).await {
                    break;
                }
            },
        }
    }
}
```

The dispatcher `handle_incoming` matches on the message type:

```rust
async fn handle_incoming<Context>(&mut self, incoming, ctx) -> bool {
    match incoming {
        // New relay chain leaf activated (only for legacy CollatorFn mode)
        Ok(FromOrchestra::Signal(OverseerSignal::ActiveLeaves(...))) => {
            self.handle_new_activation(activated, ctx).await;
            false
        },
        // Shutdown signal
        Ok(FromOrchestra::Signal(OverseerSignal::Conclude)) => true,
        // Initialize (legacy CollatorFn mode)
        Ok(FromOrchestra::Communication {
            msg: CollationGenerationMessage::Initialize(config),
        }) => { ... },
        // OUR COLLATION ARRIVES HERE
        Ok(FromOrchestra::Communication {
            msg: CollationGenerationMessage::SubmitCollation(params),
        }) => {
            self.handle_submit_collation(params, ctx).await;
            false
        },
        // Block finalized — ignore
        Ok(FromOrchestra::Signal(OverseerSignal::BlockFinalized(..))) => false,
        // ...
    }
}
```

Two types of messages arrive:

- **Signals** (broadcast from Overseer to all subsystems): `ActiveLeaves`, `Conclude`, `BlockFinalized`.
- **Communication** (directed specifically to this subsystem): `Initialize`, `Reinitialize`, `SubmitCollation`.

The `SubmitCollation` from our collator enters `handle_submit_collation`.

---

## 25.3. handle_submit_collation: Preparing the Collation

**File**: `polkadot/node/collation-generation/src/lib.rs`

```rust
async fn handle_submit_collation<Context>(
    &mut self,
    params: SubmitCollationParams,
    ctx: &mut Context,
) -> Result<()> {
    // 1. Verify subsystem is initialized
    let Some(config) = &self.config else {
        return Err(Error::SubmittedBeforeInit);
    };

    let SubmitCollationParams {
        relay_parent, collation, parent_head,
        validation_code_hash, result_sender, core_index, scheduling_parent,
    } = params;

    // 2. Get validation data from the relay chain
    let mut validation_data = match request_persisted_validation_data(
        relay_parent, config.para_id,
        OccupiedCoreAssumption::TimedOut, ctx.sender(),
    ).await.await?? {
        Some(v) => v,
        None => { return Ok(()); },
    };

    // 3. Replace parent_head with the one the collator used
    validation_data.parent_head = parent_head;

    // 4. Get claim queue
    let claim_queue = request_claim_queue(relay_parent, ctx.sender()).await.await??;

    // 5. Get session index
    let session_index =
        request_session_index_for_child(relay_parent, ctx.sender()).await.await??;

    // 6. Get session info (number of validators)
    let session_info =
        self.session_info_cache.get(relay_parent, session_index, ctx.sender()).await?;

    // 7. Package everything
    let collation = PreparedCollation {
        collation, relay_parent, para_id: config.para_id,
        validation_data, validation_code_hash,
        n_validators: session_info.n_validators,
        core_index, session_index,
    };

    // 8. Construct candidate receipt and distribute
    construct_and_distribute_receipt(
        collation, ctx.sender(), result_sender, &mut self.metrics,
        &transpose_claim_queue(claim_queue), scheduling_parent,
    ).await?;

    Ok(())
}
```

- **Step 2**: Requests `PersistedValidationData` from the relay chain for this parachain at this relay parent. Uses `OccupiedCoreAssumption::TimedOut`, assuming any previous candidate occupying the core has timed out. These requests go to the `RuntimeApiSubsystem`, which executes relay chain runtime APIs.
- **Step 3**: The validation data from the relay chain has the `parent_head` as the relay chain knows it. But the collator may have built on a more recent block not yet included in the relay. So it replaces with the actual `parent_head` used.
- **Step 4**: The claim queue maps cores to parachains. Used to verify this parachain has the right to use the claimed core.
- **Step 6**: The number of validators is needed for **erasure coding** — to know how many chunks to split the PoV into (one per validator).

The `request_*` functions are helpers that send a message to the `RuntimeApiSubsystem` via the Overseer, wait for a response on a oneshot channel, and return the result. Standard request/response pattern between subsystems.

---

## 25.4. construct_and_distribute_receipt: Building the Candidate

**File**: `polkadot/node/collation-generation/src/lib.rs`

```rust
async fn construct_and_distribute_receipt(
    collation: PreparedCollation,
    sender: &mut impl overseer::CollationGenerationSenderTrait,
    result_sender: Option<oneshot::Sender<CollationSecondedSignal>>,
    metrics: &Metrics,
    transposed_claim_queue: &TransposedClaimQueue,
    scheduling_parent: Option<Hash>,
) -> Result<()> {
    let PreparedCollation {
        collation, para_id, relay_parent, validation_data,
        validation_code_hash, n_validators, core_index, session_index,
    } = collation;

    let persisted_validation_data_hash = validation_data.hash();
    let parent_head_data = validation_data.parent_head.clone();
    let parent_head_data_hash = validation_data.parent_head.hash();

    // 1. Compress PoV and verify size
    let pov = {
        let pov = collation.proof_of_validity.into_compressed();
        let encoded_size = pov.encoded_size();
        if encoded_size > validation_data.max_pov_size as usize {
            return Err(Error::POVSizeExceeded(
                encoded_size, validation_data.max_pov_size as usize,
            ));
        }
        pov
    };

    let pov_hash = pov.hash();

    // 2. Calculate erasure root
    let erasure_root = erasure_root(n_validators, validation_data, pov.clone())?;

    // 3. Assemble commitments
    let commitments = CandidateCommitments {
        upward_messages: collation.upward_messages,
        horizontal_messages: collation.horizontal_messages,
        new_validation_code: collation.new_validation_code,
        head_data: collation.head_data,
        processed_downward_messages: collation.processed_downward_messages,
        hrmp_watermark: collation.hrmp_watermark,
    };

    // 4. Build candidate receipt
    let receipt = {
        let descriptor = if let Some(sched_parent) = scheduling_parent {
            CandidateDescriptorV2::new_v3(
                para_id, relay_parent, core_index, session_index,
                persisted_validation_data_hash, pov_hash, erasure_root,
                commitments.head_data.hash(), validation_code_hash, sched_parent,
            )
        } else {
            CandidateDescriptorV2::new(
                para_id, relay_parent, core_index, session_index,
                persisted_validation_data_hash, pov_hash, erasure_root,
                commitments.head_data.hash(), validation_code_hash,
            )
        };

        let ccr = CommittedCandidateReceiptV2 { descriptor, commitments: commitments.clone() };
        ccr.parse_ump_signals(&transposed_claim_queue, scheduling_parent.is_some())
            .map_err(Error::CandidateReceiptCheck)?;
        ccr.to_plain()
    };

    // 5. Send to Collator Protocol for distribution
    sender.send_message(CollatorProtocolMessage::DistributeCollation {
        candidate_receipt: receipt,
        parent_head_data_hash,
        pov,
        parent_head_data,
        result_sender,
        core_index,
    }).await;

    Ok(())
}
```

**Step 1 — Compress PoV and Verify Size**

Compresses the PoV and checks it doesn't exceed `max_pov_size` (defined by the relay chain). If it does, the collation is rejected immediately without reaching validators.

**Step 2 — Calculate Erasure Root**

See [Section 25.5](#255-erasure-coding-and-the-erasure-root) for details.

**Step 3 — Assemble Commitments**

The `CandidateCommitments` are the outputs of the parachain block execution:

- `upward_messages`: Messages from the parachain to the relay chain (UMP).
- `horizontal_messages`: Messages to other parachains (XCMP/HRMP).
- `new_validation_code`: If there was a runtime upgrade, the new WASM code.
- `head_data`: The new parachain block header.
- `processed_downward_messages`: How many DMP messages (relay → parachain) were processed.
- `hrmp_watermark`: Up to which point HRMP messages were processed.

**Step 4 — Build Candidate Receipt**

The `CandidateDescriptor` is the "summary" of the candidate. Contains all hashes needed for validators to verify without having the full data:

- `para_id`: Which parachain.
- `relay_parent`: Which relay block it was built on.
- `core_index`: Which execution core it uses.
- `session_index`: Which session.
- `persisted_validation_data_hash`: Hash of the validation data.
- `pov_hash`: Hash of the PoV.
- `erasure_root`: Merkle root of the erasure coding chunks.
- `head_data.hash()`: Hash of the new parachain header.
- `validation_code_hash`: Hash of the WASM runtime.

Two versions exist: V2 (no `scheduling_parent`, scheduling context equals relay parent) and V3 (explicit `scheduling_parent` for low-latency collation).

**Step 5 — Distribute to Collator Protocol**

Sends `CollatorProtocolMessage::DistributeCollation` to the Collator Protocol Subsystem, which handles network communication with validators. This message goes through the Overseer to the correct subsystem.

---

## 25.5. Erasure Coding and the Erasure Root

**File**: `polkadot/node/collation-generation/src/lib.rs`

```rust
fn erasure_root(
    n_validators: usize,
    persisted_validation: PersistedValidationData,
    pov: PoV,
) -> Result<Hash> {
    let available_data =
        AvailableData { validation_data: persisted_validation, pov: Arc::new(pov) };
    let chunks = polkadot_erasure_coding::obtain_chunks_v1(n_validators, &available_data)?;
    Ok(polkadot_erasure_coding::branches(&chunks).root())
}
```

This function takes the `AvailableData` (validation data + PoV), splits it into **chunks using erasure coding** (one per validator), and calculates the merkle root of those chunks.

Erasure coding allows the PoV to be reconstructed even if only 1/3 of validators have their chunks. This guarantees **availability** without every validator storing the complete PoV.

At this point in the collator, the erasure root is calculated **only to include the hash in the candidate receipt**. The actual distribution of chunks to validators happens later, on the validator side, after the backing group validates the candidate. The collator does not distribute chunks.

---

## 25.6. CollatorProtocolSubsystem: Distributing the Collation

**File**: `polkadot/node/network/collator-protocol/src/collator_side/mod.rs`

When `DistributeCollation` arrives at the Collator Protocol Subsystem, it's handled in `process_msg`:

```rust
match msg {
    DistributeCollation {
        candidate_receipt, parent_head_data_hash: _,
        pov, parent_head_data, result_sender, core_index,
    } => {
        distribute_collation(
            ctx, state, id, candidate_receipt, pov,
            parent_head_data, result_sender, core_index,
        ).await?;
    },
    // ...
}
```

The `distribute_collation` function:

1. **Updates connections with validators**: Ensures P2P connections to the backing group validators.
2. **Verifies the relay parent is in view**: If not, discards.
3. **Verifies core assignment**: Checks the claim queue.
4. **Stores the collation locally**: Saves candidate receipt + PoV + parent head data in local state.
5. **Advertises to interested peers**: For each connected validator whose view includes this relay parent, calls `advertise_collation`.

---

## 25.7. Validator Discovery: determine_our_validators

**File**: `polkadot/node/network/collator-protocol/src/collator_side/mod.rs`

The collator needs to know which validators form its backing group to connect and advertise to them.

```rust
async fn determine_our_validators<Context>(
    ctx: &mut Context,
    runtime: &mut RuntimeInfo,
    core_index: CoreIndex,
    relay_parent: Hash,
) -> Result<GroupValidators> {
    // 1. Get session index
    let session_index = runtime.get_session_index_for_child(ctx.sender(), relay_parent).await?;

    // 2. Get session info (all validator groups, discovery keys)
    let info = &runtime
        .get_session_info_by_index(ctx.sender(), relay_parent, session_index)
        .await?
        .session_info;

    // 3. Get the validator groups
    let groups = &info.validator_groups;
    let num_cores = groups.len();

    // 4. Get rotation info
    let rotation_info = get_group_rotation_info(ctx.sender(), relay_parent).await?;

    // 5. Calculate which group is assigned to our core
    let current_group_index = rotation_info.group_for_core(core_index, num_cores);

    // 6. Get the validators in that group
    let current_validators =
        groups.get(current_group_index).map(|v| v.as_slice()).unwrap_or_default();

    // 7. Map to discovery keys
    let validators = &info.discovery_keys;
    let current_validators =
        current_validators.iter().map(|i| validators[i.0 as usize].clone()).collect();

    Ok(GroupValidators { validators: current_validators })
}
```

- **Step 3**: `validator_groups` is a vector of vectors. Each group is a list of `ValidatorIndex`. For example: group 0 = [validator 5, validator 12, validator 23], group 1 = [validator 1, validator 8, validator 15], etc.
- **Step 4**: `rotation_info` contains how groups rotate between cores. Groups rotate periodically during a session so the same validators don't always validate the same parachain.
- **Step 5**: `group_for_core(core_index, num_cores)` calculates which group is assigned to this core at this moment, taking rotation into account.
- **Step 7**: Converts `ValidatorIndex` to `AuthorityDiscoveryId` — the keys used to find validators on the P2P network. These keys are used with `ConnectToValidators` to establish network connections.

After obtaining these discovery keys, the collator sends `NetworkBridgeTxMessage::ConnectToValidators` to the Network Bridge to establish P2P connections with these specific validators.

---

## 25.8. Advertising to Validators: advertise_collation

**File**: `polkadot/node/network/collator-protocol/src/collator_side/mod.rs`

Once connected, the collator advertises its collation to all validators in the backing group:

```rust
async fn advertise_collation<Context>(
    ctx: &mut Context,
    scheduling_parent: Hash,
    per_scheduling_parent: &mut PerSchedulingParent,
    peer: &PeerId,
    peer_version: CollationVersion,
    peer_ids: &HashMap<PeerId, HashSet<AuthorityDiscoveryId>>,
    advertisement_timeouts: &mut FuturesUnordered<ResetInterestTimeout>,
    metrics: &Metrics,
) {
    for (candidate_hash, collation_and_core) in per_scheduling_parent.collations.iter_mut() {
        // Check if we should advertise to this peer
        let should_advertise =
            validator_group.should_advertise_to(candidate_hash, peer_ids, &peer);

        match should_advertise {
            ShouldAdvertiseTo::Yes => {},
            ShouldAdvertiseTo::NotAuthority | ShouldAdvertiseTo::AlreadyAdvertised => {
                continue;
            },
        }

        // Build the network message
        let message = match peer_version {
            CollationVersion::V3 => {
                CollationProtocols::V3(
                    protocol_v3::CollatorProtocolMessage::AdvertiseCollation {
                        scheduling_parent,
                        candidate_hash: *candidate_hash,
                        parent_head_data_hash: collation.parent_head_data.hash(),
                        candidate_descriptor_version,
                        relay_parent: collation.receipt.descriptor.relay_parent(),
                    },
                )
            },
            // V2 fallback...
        };

        // SEND OVER THE P2P NETWORK
        ctx.send_message(NetworkBridgeTxMessage::SendCollationMessage(
            vec![*peer], message
        )).await;

        // Track that we advertised to this peer
        validator_group.advertised_to_peer(candidate_hash, &peer_ids, peer);
    }
}
```

The `AdvertiseCollation` message is lightweight — it contains only the candidate hash and relay parent, **not the PoV**. The `NetworkBridgeTxMessage::SendCollationMessage` goes to the Network Bridge TX Subsystem, which serializes it and sends it over the P2P network to the validator.

This is the exact point where data leaves the collator process and travels over the network.

The collator tracks which validators have been advertised to (via `advertised_to_peer` with a `BitVec`) to avoid sending duplicate advertisements.

---

## 25.9. PoV Delivery: The Request/Response Protocol

The collation protocol uses a request/response pattern, not a push model:

```
Collator                                          Validator
   |                                                  |
   |-- AdvertiseCollation (lightweight, no PoV) ----->|
   |                                                  |
   |<-- CollationFetchingRequest (requests PoV) ------|
   |                                                  |
   |-- CollationFetchingResponse (receipt+PoV) ------>|
   |                                                  |
   |<-- CollationSeconded (validator backed it) ------|
```

### Receiving the Request

When a validator requests the PoV, it arrives as an incoming request handled by `handle_incoming_request`:

**File**: `polkadot/node/network/collator-protocol/src/collator_side/mod.rs`

```rust
async fn handle_incoming_request<Context>(
    ctx: &mut Context,
    state: &mut State,
    req: std::result::Result<VersionedCollationRequest, incoming::Error>,
) -> Result<()> {
    let req = req?;
    let scheduling_parent = req.scheduling_parent();
    let peer_id = req.peer_id();
    let para_id = req.para_id();

    // Look up the stored collation
    let per_scheduling_parent = state.per_scheduling_parent.get_mut(&scheduling_parent);
    let (receipt, pov, parent_head_data) = /* extract from stored collation */;

    // Send it
    send_collation(state, req, receipt, pov, parent_head_data).await;
}
```

### Sending the Response

```rust
async fn send_collation(
    state: &mut State,
    request: VersionedCollationRequest,
    receipt: CandidateReceipt,
    pov: PoV,
    parent_head_data: HeadData,
) {
    let result = Ok(request_v2::CollationFetchingResponse::CollationWithParentHeadData {
        receipt,
        pov,
        parent_head_data,
    });

    let response = OutgoingResponse {
        result,
        reputation_changes: Vec::new(),
        sent_feedback: Some(tx),
    };

    request.send_outgoing_response(response);
}
```

The collator sends the **complete PoV** (not chunks) to the requesting validator. This is efficient because:
- Not all validators request the PoV — typically only one does.
- The PoV is only sent on demand, not pushed to everyone.
- The validator that receives it will handle erasure coding and chunk distribution later.

### Receiving CollationSeconded

When a validator backs the candidate, it sends `CollationSeconded` back to the collator. This is handled in `handle_incoming_peer_message` and forwarded to the result sender:

```rust
CollationProtocols::V2(V2::CollationSeconded(scheduling_parent, statement)) => {
    let statement = runtime.check_signature(ctx.sender(), scheduling_parent, statement).await?;
    let removed = state.collation_result_senders.remove(&statement.payload().candidate_hash());
    if let Some(sender) = removed {
        let _ = sender.send(CollationSecondedSignal { statement, scheduling_parent });
    }
}
```

This completes the feedback loop — the collator knows its collation was accepted.

---

## 25.10. The Core Index: How the Collator Knows Its Core

The `core_index` originates at the very beginning of the block production flow, in the main loop of `block_builder_task.rs`:

```rust
let core = match determine_core(
    &mut relay_chain_data_cache, &relay_parent_header,
    para_id, parent_header, relay_parent_offset,
).await { ... };
```

`determine_core` queries the relay chain's **claim queue**, which maps cores to parachains: "for my `para_id`, what core am I assigned to at this relay parent?"

From there, the `core_index` travels through the entire pipeline:

```
block_builder_task (determine_core)
    → CollatorMessage { core_index }
        → collation_task
            → SubmitCollation { core_index }
                → handle_submit_collation
                    → construct_and_distribute_receipt { core_index }
                        → DistributeCollation { core_index }
                            → distribute_collation
                                → determine_our_validators(core_index)
                                    → advertise_collation
```

The same `core_index` determined at the start is used throughout: to build the candidate receipt, to find the backing group validators, and to advertise the collation.

---

## 25.11. Collator Consensus and Slot Assignment

A common misconception is that all collators compete to produce every block. In reality, **parachains have their own consensus mechanism** that determines which collator produces in each slot. The most common is **Aura (Authority Round)**, which assigns exactly one collator per slot via round-robin.

### 25.11.1. When Does the Consensus Mechanism Activate?

The parachain's consensus mechanism activates in the `can_build_upon` function we saw in the block builder task:

**File**: `cumulus/client/consensus/aura/src/collators/mod.rs`

```rust
// 7. Check if it's our turn to produce in this slot
let slot_claim = match crate::collators::can_build_upon::<_, _, P>(
    para_slot.slot, relay_slot, para_slot.timestamp,
    parent_hash, included_header_hash, &*para_client, &keystore,
).await {
    Some(slot) => slot,
    None => { continue; },  // Not our turn — skip to next slot
};
```

If `can_build_upon` returns `None`, the collator skips this slot entirely. It doesn't build, doesn't execute, doesn't send anything. Only the collator that gets `Some(slot_claim)` proceeds with block production.

### 25.11.2. How Aura Slot Assignment Works

**File**: `cumulus/client/consensus/aura/src/collators/mod.rs`

```rust
async fn can_build_upon<Block: BlockT, Client, P>(
	para_slot: Slot,
	relay_slot: Slot,
	timestamp: Timestamp,
	parent_hash: Block::Hash,
	included_block: Block::Hash,
	client: &Client,
	keystore: &KeystorePtr,
) -> Option<SlotClaim<P::Public>>
where
	Client: ProvideRuntimeApi<Block>,
	Client::Api: AuraApi<Block, P::Public> + AuraUnincludedSegmentApi<Block> + ApiExt<Block>,
	P: Pair,
	P::Public: Codec,
	P::Signature: Codec,
{
	let mut runtime_api = client.runtime_api();
	runtime_api.set_call_context(sp_core::traits::CallContext::Onchain);

	// 1. Get the list of authorities from the runtime
	let authorities = runtime_api.authorities(parent_hash).ok()?;

	// 2. Check if we are the assigned author for this slot
	let author_pub = aura_internal::claim_slot::<P>(para_slot, &authorities, keystore).await?;

	// 3. If parent is the included block, unincluded segment is empty — can build
	if parent_hash == included_block {
		return Some(SlotClaim::unchecked::<P>(author_pub, para_slot, timestamp));
	}

	// 4. Check API version for slot type
	let api_version = runtime_api
		.api_version::<dyn AuraUnincludedSegmentApi<Block>>(parent_hash)
		.ok()
		.flatten()?;

	let slot = if api_version > 1 { relay_slot } else { para_slot };

	// 5. Ask runtime if there's capacity in the unincluded segment
	runtime_api
		.can_build_upon(parent_hash, included_block, slot)
		.ok()?
		.then(|| SlotClaim::unchecked::<P>(author_pub, para_slot, timestamp))
}
```

**Step 1** gets the authorities (public keys of all authorized collators) from `pallet-aura` in the runtime.

**Step 2** is where Aura consensus happens — `claim_slot` determines if this collator should produce. If the collator's key is not assigned to this slot, `claim_slot` returns `None` and the function exits early.

**Step 3-5** check if there's capacity in the unincluded segment (for async backing scenarios where multiple blocks can be pending inclusion).

### 25.11.3. The claim_slot Function: Round-Robin Assignment

**File**: `substrate/client/consensus/aura/src/standalone.rs`

```rust
pub async fn claim_slot<P: Pair>(
	slot: Slot,
	authorities: &[AuthorityId<P>],
	keystore: &KeystorePtr,
) -> Option<P::Public> {
	let expected_author = slot_author::<P>(slot, authorities);
	expected_author.and_then(|p| {
		if keystore.has_keys(&[(p.to_raw_vec(), sp_application_crypto::key_types::AURA)]) {
			Some(p.clone())
		} else {
			None
		}
	})
}
```

The function first calculates who SHOULD produce (`slot_author`), then checks if the local keystore has the private key for that author.

**File**: `substrate/client/consensus/aura/src/standalone.rs`

```rust
pub fn slot_author<P: Pair>(slot: Slot, authorities: &[AuthorityId<P>]) -> Option<&AuthorityId<P>> {
	if authorities.is_empty() {
		return None;
	}

	let idx = *slot % (authorities.len() as u64);
	assert!(
		idx <= usize::MAX as u64,
		"It is impossible to have a vector with length beyond the address space; qed",
	);

	let current_author = authorities.get(idx as usize).expect(
		"authorities not empty; index constrained to list length;this is a valid index; qed",
	);

	Some(current_author)
}
```

**The formula is `slot % authorities.len()`**. This creates a round-robin:

| Slot | Formula (3 authorities) | Author |
|------|------------------------|--------|
| 0    | 0 % 3 = 0              | Collator A |
| 1    | 1 % 3 = 1              | Collator B |
| 2    | 2 % 3 = 2              | Collator C |
| 3    | 3 % 3 = 0              | Collator A |
| 4    | 4 % 3 = 1              | Collator B |
| ...  | ...                    | ... |

Each slot has exactly **one** assigned author. The collator checks if its keystore contains the private key for that author. If yes → produce. If no → skip.

### 25.11.4. Where Are Authorities and Keys Stored?

Two separate locations:

**1. Authorities (on-chain)**: The list of authorized collator public keys is stored in the runtime:

```
pallet_aura::Authorities<T>
```

This is configured in the genesis (chain spec) and can be updated via:
- `pallet-collator-selection` (staking-based selection)
- `pallet-invulnerables` (governance-appointed collators)
- Direct governance calls

**2. Private Keys (off-chain)**: Each collator stores its private key in the local filesystem:

```
/path-to-node/chains/<chain-id>/keystore/
```

Keys are inserted via `author_insertKey` RPC or loaded from files at startup.

`claim_slot` bridges both: it queries the authorities from the runtime, calculates which one should produce, and checks if the local keystore has the corresponding private key.

### 25.11.5. When Does Competition Actually Happen?

Competition between collators (a "race") only occurs in specific scenarios:

**1. Multiple Nodes with Same Key (Misconfiguration/Backup)**

If two collator nodes have the same Aura key in their keystore, both will pass `claim_slot` for the same slot. Both will build blocks and race to get theirs included first.

```
Collator Node 1 (keystore: Alice) ─┐
                                   ├─> Both claim slot 5 ─> RACE
Collator Node 2 (keystore: Alice) ─┘
```

This is typically a misconfiguration, but sometimes intentional for redundancy (hot backup).

**2. Async Backing with AllowMultipleBlocksPerSlot**

With async backing enabled, a collator may produce multiple blocks in the same slot to fill the unincluded segment:

```rust
// In the runtime configuration
type AllowMultipleBlocksPerSlot = ConstBool<true>;
```

This allows the same collator to produce Velocity + 1 candidates per slot while there's remaining capacity.

**3. It's NOT Proof of Work**

There's no mining, no hash computation race. When multiple collators produce for the same slot, it's simply a **latency race** — the first advertisement to reach a validator and get seconded wins. The others are discarded.

### 25.11.6. Alternative Consensus Mechanisms

A parachain can implement different consensus mechanisms by replacing the slot assignment logic:

**BABE (Blind Assignment for Blockchain Extension)**

Instead of deterministic round-robin, uses a VRF (Verifiable Random Function):

```rust
// Pseudocode
let vrf_output = vrf_sign(slot, secret_key);
if vrf_output < threshold {
    // I can produce in this slot
}
```

Multiple collators might pass the threshold → competition. Or none might pass → empty slot.

**Proof of Stake Selection**

The set of authorities is dynamic based on stake:

1. Collators stake tokens via `pallet-collator-selection`
2. Each session, the top N by stake become the active set
3. Their public keys are written to `pallet_aura::Authorities`
4. Slot assignment still uses Aura round-robin, but among staked collators

```
pallet-collator-selection (who CAN produce) → pallet-aura (who produces WHEN)
```

**Custom Mechanism**

To implement a fully custom consensus:

1. **Runtime**: Replace `pallet-aura` with your own pallet implementing the `AuraApi` runtime API (or a custom API)
2. **Node**: Replace `claim_slot` logic in `can_build_upon` with your custom verification

The rest of the pipeline (block construction, collation submission, validation) remains unchanged.

### 25.11.7. Why Multiple Collators?

With Aura properly configured, each slot has one assigned collator — no competition. So why run multiple collators?

**Liveness**: If the assigned collator for slot N is offline, no block is produced for that slot. Having multiple collators ensures that most slots have an online producer.

**Redundancy**: Running backup nodes with the same key provides failover (though creates the race condition described above).

**Decentralization**: More collators (with different keys) means the parachain doesn't depend on a single operator.

**Not for Security**: Collators don't provide security — they just propose blocks. Security comes from relay chain validators who verify every block. A malicious collator can't include invalid transactions because validators will reject the collation.

---

## 25.12. Summary: Complete Collator Pipeline

Including both the block production (covered in earlier sections) and the collation submission (covered here), the complete collator pipeline is:

```
BLOCK PRODUCTION (sections 5-24):
 1. Wait for slot
 2. Get relay chain data (validation data, relay parent)
 3. Check if it's our turn (can_build_upon)
 4. Create inherent data
 5. Build block (initialize_block → apply_inherents → apply_extrinsics → finalize_block)
 6. Seal with collator key
 7. Import locally to RocksDB
 8. Send through channel to collation_task

COLLATION SUBMISSION (this section):
 9. collation_task sends SubmitCollation to Overseer
10. CollationGenerationSubsystem receives it:
    - Gets validation data, claim queue, session info from relay chain
    - Compresses PoV, verifies size
    - Calculates erasure root (for the candidate receipt hash only)
    - Builds CandidateCommitments and CandidateDescriptor
    - Sends DistributeCollation to CollatorProtocolSubsystem
11. CollatorProtocolSubsystem:
    - Discovers backing group validators via determine_our_validators
    - Connects to them via P2P (ConnectToValidators)
    - Sends AdvertiseCollation (lightweight, no PoV) to all backing group validators
12. When a validator requests the PoV:
    - Sends complete PoV via CollationFetchingResponse
13. Receives CollationSeconded when a validator backs the candidate

After step 13, the collator's job is done. The validator side takes over:
validation, erasure chunk distribution, availability, inclusion in relay chain,
and finalization with GRANDPA.
```
