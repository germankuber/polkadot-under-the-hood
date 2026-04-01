# XCM Message Execution: The on_idle Processing Pipeline

## Table of Contents

- [XCM Message Execution: The on\_idle Processing Pipeline](#xcm-message-execution-the-on_idle-processing-pipeline)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [1. on\_idle\_hook: The Executive Triggers Processing](#1-on_idle_hook-the-executive-triggers-processing)
  - [2. pallet-message-queue::on\_idle](#2-pallet-message-queueon_idle)
  - [3. service\_queues\_impl: Round-Robin Across Origins](#3-service_queues_impl-round-robin-across-origins)
  - [4. service\_queue: Processing One Queue](#4-service_queue-processing-one-queue)
  - [5. service\_page: Processing One Page](#5-service_page-processing-one-page)
  - [6. service\_page\_item: Processing One Message](#6-service_page_item-processing-one-message)
  - [7. process\_message\_payload: Storage Transaction Wrapper](#7-process_message_payload-storage-transaction-wrapper)
  - [8. process\_message: Decoding and Executing XCM](#8-process_message-decoding-and-executing-xcm)
  - [9. XcmExecutor::execute: The XCM Virtual Machine](#9-xcmexecutorexecute-the-xcm-virtual-machine)
  - [10. process: The Instruction Loop](#10-process-the-instruction-loop)
  - [11. process\_instruction: Where Each Instruction Executes](#11-process_instruction-where-each-instruction-executes)
    - [Asset Transfers](#asset-transfers)
    - [Reserve Transfers](#reserve-transfers)
    - [Teleports](#teleports)
    - [Runtime Call Execution](#runtime-call-execution)
    - [Fee Payment](#fee-payment)
    - [Origin and Control Flow](#origin-and-control-flow)
    - [HRMP Channel Management](#hrmp-channel-management)
    - [Queries and Reporting](#queries-and-reporting)
  - [12. How Bytes Become Instructions: SCALE Decoding](#12-how-bytes-become-instructions-scale-decoding)
  - [13. Complete Execution Chain](#13-complete-execution-chain)

---

## Introduction

This document covers how XCM messages that were enqueued during `set_validation_data` (see the "Parachain Inherent Data" document) actually get executed. The execution happens during `on_idle` in `finalize_block`, when `pallet-message-queue` processes its queues using the remaining block weight.

This corresponds to **Section 19.1** of the main block lifecycle document, expanding the `on_idle` step inside `finalize_block`.

---

## 1. on_idle_hook: The Executive Triggers Processing

**File**: `substrate/frame/executive/src/lib.rs`

During `finalize_block`, the Executive calls `on_idle_hook`:

```rust
fn on_idle_hook(block_number: NumberFor<Block>) {
    // Don't process if multi-block migration is ongoing
    if <System as frame_system::Config>::MultiBlockMigrator::ongoing() {
        return;
    }

    // Calculate remaining weight
    let weight = <frame_system::Pallet<System>>::block_weight();
    let max_weight = <System::BlockWeights as frame_support::traits::Get<_>>::get().max_block;
    let remaining_weight = max_weight.saturating_sub(weight.total());

    // Call on_idle for all pallets with remaining weight as budget
    if remaining_weight.all_gt(Weight::zero()) {
        let used_weight = <AllPalletsWithSystem as OnIdle<BlockNumberFor<System>>>::on_idle(
            block_number,
            remaining_weight,
        );
        <frame_system::Pallet<System>>::register_extra_weight_unchecked(
            used_weight,
            DispatchClass::Mandatory,
        );
    }
}
```

The Executive calculates how much weight remains after all inherents and transactions have been applied, then passes that as a budget to `on_idle` for **all pallets**. `pallet-message-queue` is one of those pallets and uses this budget to process enqueued XCM messages.

---

## 2. pallet-message-queue::on_idle

**File**: `substrate/frame/message-queue/src/lib.rs`

```rust
fn on_idle(_n: BlockNumberFor<T>, remaining_weight: Weight) -> Weight {
    if let Some(weight_limit) = T::IdleMaxServiceWeight::get() {
        Self::service_queues_impl(
            weight_limit.min(remaining_weight),
            ServiceQueuesContext::OnIdle,
        )
    } else {
        Weight::zero()
    }
}
```

Takes the minimum of `IdleMaxServiceWeight` (a configured cap) and the actual remaining weight. If no cap is configured, does nothing. Otherwise delegates to `service_queues_impl`.

---

## 3. service_queues_impl: Round-Robin Across Origins

**File**: `substrate/frame/message-queue/src/lib.rs`

```rust
fn service_queues_impl(weight_limit: Weight, context: ServiceQueuesContext) -> Weight {
    let mut weight = WeightMeter::with_limit(weight_limit);

    // Maximum weight for processing a single message
    let overweight_limit = Self::max_message_weight(weight_limit).unwrap_or_else(|| {
        Weight::zero()
    });

    // Get the first queue in the ready ring
    let mut next = match Self::bump_service_head(&mut weight) {
        Some(h) => h,
        None => return weight.consumed(),  // No queues to process
    };

    // Track if we're making progress
    let mut last_no_progress = None;

    loop {
        // Process one queue
        let (progressed, n) =
            Self::service_queue(next.clone(), &mut weight, overweight_limit);

        next = match n {
            Some(n) => {
                if !progressed {
                    if last_no_progress == Some(n.clone()) {
                        break;  // Full circle without progress, stop
                    }
                    if last_no_progress.is_none() {
                        last_no_progress = Some(next.clone())
                    }
                    n
                } else {
                    last_no_progress = None;
                    n
                }
            },
            None => break,  // No more queues
        }
    }
    weight.consumed()
}
```

The **ready ring** is a circular linked list of origins that have pending messages. Each origin represents a queue: one for DMP (relay chain → parachain), one for each HRMP channel (parachain A → this parachain, parachain B → this parachain, etc.).

The loop works in **round-robin**: processes one queue, moves to the next, processes that one, and so on. This ensures fair processing across all message sources. The loop stops when:

- The weight budget runs out.
- A full circle is completed without any queue making progress.
- No more queues remain in the ready ring.

---

## 4. service_queue: Processing One Queue

**File**: `substrate/frame/message-queue/src/lib.rs`

```rust
fn service_queue(
    origin: MessageOriginOf<T>,
    weight: &mut WeightMeter,
    overweight_limit: Weight,
) -> (bool, Option<MessageOriginOf<T>>) {
    let mut book_state = BookStateFor::<T>::get(&origin);
    let mut total_processed = 0;

    // Skip if queue is paused (e.g., channel suspended)
    if T::QueuePausedQuery::is_paused(&origin) {
        let next_ready = book_state.ready_neighbours.as_ref().map(|x| x.next.clone());
        return (false, next_ready);
    }

    // Loop through pages
    while book_state.end > book_state.begin {
        let (processed, status) =
            Self::service_page(&origin, &mut book_state, weight, overweight_limit);
        total_processed.saturating_accrue(processed);
        match status {
            Bailed | NoProgress => break,  // Out of weight or stuck
            NoMore => (),                   // Page done, go to next
        };
        book_state.begin.saturating_inc();
    }

    // If queue is empty, remove from ready ring
    let next_ready = book_state.ready_neighbours.as_ref().map(|x| x.next.clone());
    if book_state.begin >= book_state.end {
        if let Some(neighbours) = book_state.ready_neighbours.take() {
            Self::ready_ring_unknit(&origin, neighbours);
        }
    }
    BookStateFor::<T>::insert(&origin, &book_state);

    (total_processed > 0, next_ready)
}
```

Processes one origin's queue page by page. If the queue is **paused** (from a `ChannelSignal::Suspend`), it skips it entirely. If the queue is emptied, it's removed from the ready ring so future rounds don't visit it.

---

## 5. service_page: Processing One Page

**File**: `substrate/frame/message-queue/src/lib.rs`

```rust
fn service_page(
    origin: &MessageOriginOf<T>,
    book_state: &mut BookStateOf<T>,
    weight: &mut WeightMeter,
    overweight_limit: Weight,
) -> (u32, PageExecutionStatus) {
    let page_index = book_state.begin;
    let mut page = match Pages::<T>::get(origin, page_index) {
        Some(p) => p,
        None => return (0, NoMore),
    };

    let mut total_processed = 0;

    // Execute as many messages as possible
    let status = loop {
        match Self::service_page_item(
            origin, page_index, book_state, &mut page, weight, overweight_limit,
        ) {
            Bailed => break PageExecutionStatus::Bailed,
            NoItem => break PageExecutionStatus::NoMore,
            NoProgress => break PageExecutionStatus::NoProgress,
            Executed(true) => total_processed.saturating_inc(),
            Executed(false) => (),
        }
    };

    // If page is complete, remove from storage
    if page.is_complete() {
        Pages::<T>::remove(origin, page_index);
        book_state.count.saturating_dec();
    } else {
        Pages::<T>::insert(origin, page_index, page);
    }

    (total_processed, status)
}
```

Loops through messages in a page, calling `service_page_item` for each one. If the page is fully processed, it's deleted from storage. If not (ran out of weight), it's saved back with the progress marker so the next block picks up where it left off.

---

## 6. service_page_item: Processing One Message

**File**: `substrate/frame/message-queue/src/lib.rs`

```rust
pub(crate) fn service_page_item(
    origin: &MessageOriginOf<T>,
    page_index: PageIndex,
    book_state: &mut BookStateOf<T>,
    page: &mut PageOf<T>,
    weight: &mut WeightMeter,
    overweight_limit: Weight,
) -> ItemExecutionStatus {
    if page.is_complete() {
        return ItemExecutionStatus::NoItem;
    }
    if weight.try_consume(T::WeightInfo::service_page_item()).is_err() {
        return ItemExecutionStatus::Bailed;
    }

    // Peek at the first message in the page
    let payload = &match page.peek_first() {
        Some(m) => m,
        None => return ItemExecutionStatus::NoItem,
    }[..];

    // Process the message
    let res = Self::process_message_payload(
        origin.clone(), page_index, page.first_index,
        payload, weight, overweight_limit,
    );

    let is_processed = match res {
        InsufficientWeight => return ItemExecutionStatus::Bailed,
        Unprocessable { permanent: false } => return ItemExecutionStatus::NoProgress,
        Processed | Unprocessable { permanent: true } | StackLimitReached => true,
        Overweight => false,
    };

    if is_processed {
        book_state.message_count.saturating_dec();
        book_state.size.saturating_reduce(payload_len as u64);
    }
    page.skip_first(is_processed);
    ItemExecutionStatus::Executed(is_processed)
}
```

Takes one message from the page and sends it to `process_message_payload`. Based on the result:

- **`Processed`**: Message executed successfully. Remove from page, decrement counters.
- **`InsufficientWeight`**: Not enough weight. Stop processing (try next block).
- **`Overweight`**: Message is too heavy for any block. Stays in queue permanently.
- **`Unprocessable { permanent: true }`**: Corrupt or unsupported message. Discard it.
- **`Unprocessable { permanent: false }`**: Temporary failure, retry later.

---

## 7. process_message_payload: Storage Transaction Wrapper

**File**: `substrate/frame/message-queue/src/lib.rs`

```rust
fn process_message_payload(
    origin: MessageOriginOf<T>,
    page_index: PageIndex,
    message_index: T::Size,
    message: &[u8],
    meter: &mut WeightMeter,
    overweight_limit: Weight,
) -> MessageExecutionStatus {
    let mut id = sp_io::hashing::blake2_256(message);

    let transaction =
        storage::with_transaction(|| -> TransactionOutcome<Result<_, DispatchError>> {
            let res =
                T::MessageProcessor::process_message(message, origin.clone(), meter, &mut id);
            match &res {
                Ok(_) => TransactionOutcome::Commit(Ok(res)),
                Err(_) => TransactionOutcome::Rollback(Ok(res)),
            }
        });

    match transaction {
        Ok(Ok(success)) => {
            Self::deposit_event(Event::<T>::Processed { id, origin, weight_used, success });
            MessageExecutionStatus::Processed
        },
        Ok(Err(Overweight(w))) if w.any_gt(overweight_limit) => {
            // Permanently overweight
            MessageExecutionStatus::Overweight
        },
        Ok(Err(Overweight(_))) => {
            // Temporarily out of weight
            MessageExecutionStatus::InsufficientWeight
        },
        Ok(Err(BadFormat | Corrupt | Unsupported)) => {
            // Invalid message, drop permanently
            MessageExecutionStatus::Unprocessable { permanent: true }
        },
        // ...
    }
}
```

Wraps the execution in `storage::with_transaction` — the same pattern we saw in `BlockBuilder::push`:
- **If `Ok`**: `Commit` — storage changes are kept.
- **If `Err`**: `Rollback` — all storage changes are reverted. The message failed but no state was corrupted.

`T::MessageProcessor::process_message` is where the actual XCM processing happens.

---

## 8. process_message: Decoding and Executing XCM

**File**: `polkadot/xcm/xcm-builder/src/process_xcm_message.rs`

```rust
fn process_message(
    message: &[u8],
    origin: Self::Origin,
    meter: &mut WeightMeter,
    id: &mut XcmHash,
) -> Result<bool, ProcessMessageError> {
    // 1. Decode bytes into VersionedXcm
    let versioned_message = VersionedXcm::<Call>::decode_all_with_depth_limit(
        MAX_XCM_DECODE_DEPTH,
        &mut &message[..],
    ).map_err(|e| ProcessMessageError::Corrupt)?;

    // 2. Convert to current XCM version
    let message = Xcm::<Call>::try_from(versioned_message)
        .map_err(|_| ProcessMessageError::Unsupported)?;

    // 3. Prepare execution (validate and calculate weight)
    let pre = XcmExecutor::prepare(message, Weight::MAX)
        .map_err(|_| ProcessMessageError::Unsupported)?;

    // 4. Check weight budget
    let required = pre.weight_of();
    if !meter.can_consume(required) {
        return Err(ProcessMessageError::Overweight(required));
    }

    // 5. Execute
    let (consumed, result) = match XcmExecutor::execute(
        origin.into(), pre, id, Weight::zero()
    ) {
        Outcome::Complete { used } => (used, Ok(true)),
        Outcome::Incomplete { used, error } => (used, Ok(false)),
        Outcome::Error(error) => (required, Err(error)),
    };

    meter.consume(consumed);
    result
}
```

- **Step 1**: Decodes the raw bytes into `VersionedXcm`. The bytes are SCALE-encoded: first byte is the XCM version discriminant, then the encoded instruction vector.
- **Step 2**: Converts from the versioned format to the current XCM version (`Xcm<Call>`).
- **Step 3**: `prepare` pre-validates the instructions and calculates the total weight needed without executing.
- **Step 4**: Checks if there's enough weight in the budget. If not, returns `Overweight` and the message will be retried next block.
- **Step 5**: `XcmExecutor::execute` runs the instructions. Three outcomes:
  - **Complete**: All instructions executed successfully.
  - **Incomplete**: Some instructions executed, then one failed. Partial execution.
  - **Error**: Fatal error. Consumes all estimated weight as worst case.

---

## 9. XcmExecutor::execute: The XCM Virtual Machine

**File**: `polkadot/xcm/xcm-executor/src/lib.rs`

```rust
fn execute(
    origin: impl Into<Location>,
    WeighedMessage(xcm_weight, mut message): WeighedMessage<Config::RuntimeCall>,
    id: &mut XcmHash,
    weight_credit: Weight,
) -> Outcome {
    let origin = origin.into();
    let mut properties = Properties { weight_credit, message_id: None };

    // 1. Barrier check — is this message allowed to execute?
    if let Err(e) = Config::Barrier::should_execute(
        &origin, message.inner_mut(), xcm_weight, &mut properties,
    ) {
        return Outcome::Incomplete {
            used: xcm_weight,
            error: InstructionError { index: 0, error: XcmError::Barrier },
        };
    }

    // 2. Create the VM instance
    let mut vm = Self::new(origin, *id);
    vm.message_weight = xcm_weight;

    // 3. Execute instructions, handling errors and appendix
    while !message.0.is_empty() {
        let result = vm.process(message);
        message = match result {
            Err(error) => {
                vm.total_surplus.saturating_accrue(error.weight);
                vm.error = Some((error.index, error.xcm_error));
                // On error: run error handler, then appendix
                vm.take_error_handler().or_else(|| vm.take_appendix())
            },
            Ok(()) => {
                vm.drop_error_handler();
                // On success: run appendix
                vm.take_appendix()
            },
        }
    }

    // 4. Post-process and return outcome
    vm.post_process(xcm_weight)
}
```

- **Step 1 — Barrier**: The configurable `Barrier` decides if this message has permission to execute. Typical checks: does the origin have permission? Did it pay for execution (`BuyExecution` / `PayFees`)? Is it from a trusted location? If rejected, nothing executes.
- **Step 2**: Creates the XcmExecutor as a virtual machine with an origin, a holding register (temporary asset storage), and weight tracking.
- **Step 3**: The main loop. Calls `vm.process(message)` which executes the instruction list. If it succeeds, runs the appendix (cleanup instructions). If it fails, runs the error handler first, then the appendix. This allows XCM programs to have try/catch-like error handling.

---

## 10. process: The Instruction Loop

**File**: `polkadot/xcm/xcm-executor/src/lib.rs`

```rust
fn process(&mut self, xcm: Xcm<Config::RuntimeCall>) -> Result<(), ExecutorError> {
    let mut result = Ok(());
    for (i, mut instr) in xcm.0.into_iter().enumerate() {
        match &mut result {
            r @ Ok(()) => {
                let inst_res = recursion_count::using_once(&mut 1, || {
                    recursion_count::with(|count| {
                        if *count > RECURSION_LIMIT {
                            return None;
                        }
                        *count = count.saturating_add(1);
                        Some(())
                    })
                    .flatten()
                    .ok_or(XcmError::ExceedsStackLimit)?;
                    defer! {
                        recursion_count::with(|count| {
                            *count = count.saturating_sub(1);
                        });
                    }
                    self.process_instruction(instr)
                });
                if let Err(error) = inst_res {
                    *r = Err(ExecutorError {
                        index: i as u32,
                        xcm_error: error,
                        weight: Weight::zero(),
                    });
                }
            },
            Err(ref mut error) => {
                // After an error, just accumulate weight of remaining instructions
                if let Ok(x) = Config::Weigher::instr_weight(&mut instr) {
                    error.weight.saturating_accrue(x)
                }
            },
        }
    }
    result
}
```

Iterates over every instruction in the XCM message:

- **While `Ok`**: Calls `self.process_instruction(instr)` for each instruction. If it fails, records the error with the instruction index.
- **After an error**: Remaining instructions are **not executed**. Only their weight is accumulated to report how much weight was "saved" (surplus).
- **Recursion protection**: A counter prevents infinite recursion (e.g., a `Transact` that sends another XCM message that does another `Transact`). If `RECURSION_LIMIT` is exceeded, returns `ExceedsStackLimit`.

---

## 11. process_instruction: Where Each Instruction Executes

**File**: `polkadot/xcm/xcm-executor/src/lib.rs`

`process_instruction` is a large match on the `Instruction` enum. Each XCM instruction has its own handler. The instructions grouped by category:

### Asset Transfers

| Instruction | What It Does |
|-------------|-------------|
| `WithdrawAsset(assets)` | Takes assets from the origin's account and places them in the **holding register** (a temporary register in the VM). Uses `Config::AssetTransactor::withdraw_asset`. |
| `DepositAsset { assets, beneficiary }` | Takes assets from the holding register and deposits them into the beneficiary's account. Uses `Config::AssetTransactor::deposit_asset`. |
| `TransferAsset { assets, beneficiary }` | Withdraw + Deposit in a single step. Transfers directly from origin to beneficiary. |

### Reserve Transfers

| Instruction | What It Does |
|-------------|-------------|
| `ReserveAssetDeposited(assets)` | The origin chain says "these assets are backed by my reserve." Verifies the origin is a trusted reserve location (`Config::IsReserve`), then mints the assets into holding. |
| `DepositReserveAsset { assets, dest, xcm }` | Deposits assets into the destination chain's sovereign account and sends an XCM message to that chain to notify it. |
| `InitiateReserveWithdraw { assets, reserve, xcm }` | Burns assets locally and sends a message to the reserve chain to release them. |

### Teleports

| Instruction | What It Does |
|-------------|-------------|
| `ReceiveTeleportedAsset(assets)` | Verifies the origin is a trusted teleporter (`Config::IsTeleporter`), checks in the assets, and mints them into holding. |
| `InitiateTeleport { assets, dest, xcm }` | Burns assets locally and sends a message to the destination chain to mint them there. |

### Runtime Call Execution

| Instruction | What It Does |
|-------------|-------------|
| `Transact { origin_kind, call }` | Decodes a runtime call from the payload, filters it with `SafeCallFilter`, converts the XCM origin to a local dispatch origin via `Config::OriginConverter`, then calls `Config::CallDispatcher::dispatch(call, origin)`. This can execute **any** runtime extrinsic: governance, staking, balances, etc. |

### Fee Payment

| Instruction | What It Does |
|-------------|-------------|
| `BuyExecution { fees, weight_limit }` | Takes assets from holding and buys execution weight via `self.trader.buy_weight()`. Required by the `AllowTopLevelPaidExecutionFrom` barrier. |
| `PayFees { asset }` | Modern version of BuyExecution. Pays for the entire message weight at once. |
| `RefundSurplus` | Refunds unused weight back to the holding register. |

### Origin and Control Flow

| Instruction | What It Does |
|-------------|-------------|
| `ClearOrigin` | Clears the origin. Used after depositing assets for security. |
| `DescendOrigin(who)` | Changes the origin to a child location. |
| `AliasOrigin(target)` | Changes the origin to a different location (with permission check via `Config::Aliasers`). |
| `SetErrorHandler(handler)` | Sets instructions to execute if a subsequent instruction fails. |
| `SetAppendix(appendix)` | Sets instructions that always execute at the end (cleanup). |
| `Trap(code)` | Always errors with the given code. Useful for testing. |

### HRMP Channel Management

| Instruction | What It Does |
|-------------|-------------|
| `HrmpNewChannelOpenRequest { sender, ... }` | Processes a request to open an HRMP channel. |
| `HrmpChannelAccepted { recipient }` | Processes acceptance of an HRMP channel. |
| `HrmpChannelClosing { initiator, sender, recipient }` | Processes closing of an HRMP channel. |

### Queries and Reporting

| Instruction | What It Does |
|-------------|-------------|
| `QueryResponse { query_id, response, ... }` | Handles a response to a previous query. |
| `ReportError(response_info)` | Reports the current error status back to the querier. |
| `ReportHolding { response_info, assets }` | Reports the current holding register contents. |

Each instruction ultimately calls runtime-configured traits (`AssetTransactor`, `IsReserve`, `IsTeleporter`, `CallDispatcher`, `AssetLocker`, `AssetExchanger`, etc.) that perform the actual state mutations on the parachain.

---

## 12. How Bytes Become Instructions: SCALE Decoding

The raw message bytes are decoded using **SCALE codec** (Substrate's binary encoding). When `#[derive(Encode, Decode)]` is placed on an enum, the macro automatically generates serialization/deserialization code.

The first bytes are the **variant discriminant** of the enum. `VersionedXcm` is an enum:

```rust
enum VersionedXcm {
    V2(Xcm),   // discriminant 2
    V3(Xcm),   // discriminant 3
    V4(Xcm),   // discriminant 4
    V5(Xcm),   // discriminant 5
}
```

`Xcm` is a `Vec<Instruction>`, and each `Instruction` is also an enum with its own discriminant. The bytes decode like this:

```
[5, 2, 0, ...assets_bytes..., 12, ...fees_bytes..., ...]
 │  │  │                       │
 │  │  │                       └── Instruction 2: BuyExecution (discriminant 12)
 │  │  └── Instruction 1: WithdrawAsset (discriminant 0)
 │  └── 2 instructions in the Vec
 └── VersionedXcm::V5 (discriminant 5)
```

`decode_all_with_depth_limit` reads byte by byte: first the version discriminant, then the vector length, then for each instruction its discriminant and its fields. The `with_depth_limit` adds a recursion depth counter to protect against malicious payloads with infinitely nested structures.

No hand-written parsing code exists. Everything is generated by the `#[derive(Decode)]` macro at compile time.

---

## 13. Complete Execution Chain

```
finalize_block (Executive)
  │
  └── on_idle_hook
        │
        ├── Calculate remaining_weight = max_block - weight_used
        │
        └── pallet-message-queue::on_idle(remaining_weight)
              │
              └── service_queues_impl
                    │
                    ├── bump_service_head → get first origin from ready ring
                    │
                    └── loop (round-robin across origins):
                          │
                          └── service_queue(origin)
                                │
                                ├── Check if queue is paused → skip if so
                                │
                                └── loop (pages):
                                      │
                                      └── service_page(origin, page_index)
                                            │
                                            └── loop (messages):
                                                  │
                                                  └── service_page_item
                                                        │
                                                        └── process_message_payload
                                                              │
                                                              ├── storage::with_transaction (savepoint)
                                                              │
                                                              └── T::MessageProcessor::process_message
                                                                    │
                                                                    ├── decode bytes → VersionedXcm
                                                                    ├── convert → Xcm<Call>
                                                                    ├── XcmExecutor::prepare (calculate weight)
                                                                    ├── check weight budget
                                                                    │
                                                                    └── XcmExecutor::execute
                                                                          │
                                                                          ├── Barrier::should_execute (permission check)
                                                                          ├── create VM (origin, holding register)
                                                                          │
                                                                          └── vm.process(message)
                                                                                │
                                                                                └── for each instruction:
                                                                                      │
                                                                                      └── process_instruction
                                                                                            │
                                                                                            ├── WithdrawAsset → AssetTransactor::withdraw
                                                                                            ├── DepositAsset → AssetTransactor::deposit
                                                                                            ├── TransferAsset → AssetTransactor::transfer
                                                                                            ├── ReserveAssetDeposited → verify + mint
                                                                                            ├── Transact → decode call → dispatch
                                                                                            ├── BuyExecution → trader.buy_weight
                                                                                            ├── ClearOrigin → clear
                                                                                            └── ... (30+ instruction types)
```