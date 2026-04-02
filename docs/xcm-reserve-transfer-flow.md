# XCM Reserve Transfer: Complete Flow from Extrinsic to Token Deposit

> **Deep dive into how a reserve transfer moves tokens between two parachains using the XCM VM, from the user's extrinsic call through local execution, message transport, and remote execution.**

## Table of Contents

- [XCM Reserve Transfer: Complete Flow from Extrinsic to Token Deposit](#xcm-reserve-transfer-complete-flow-from-extrinsic-to-token-deposit)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
- [Part I: The Origin Chain — From Extrinsic to Outbound Message](#part-i-the-origin-chain--from-extrinsic-to-outbound-message)
  - [1. Entry Point: do\_reserve\_transfer\_assets](#1-entry-point-do_reserve_transfer_assets)
    - [Parameter Conversion](#parameter-conversion)
    - [Filters and Validations](#filters-and-validations)
    - [Transfer Type Determination](#transfer-type-determination)
  - [2. The XCM Executor: prepare\_and\_execute](#2-the-xcm-executor-prepare_and_execute)
  - [3. WithdrawAsset: Taking Tokens from the User](#3-withdrawasset-taking-tokens-from-the-user)
    - [The Withdraw Loop](#the-withdraw-loop)
    - [Which Pallet Handles the Withdraw?](#which-pallet-handles-the-withdraw)
    - [The Matcher: How It Decides Which Adapter Handles the Asset](#the-matcher-how-it-decides-which-adapter-handles-the-asset)
    - [How Does It Know If the Asset Is the Native Token (e.g. DOT)?](#how-does-it-know-if-the-asset-is-the-native-token-eg-dot)
    - [The Holding Register](#the-holding-register)
  - [4. DepositReserveAsset: Locking Tokens and Building the Remote Message](#4-depositreserveasset-locking-tokens-and-building-the-remote-message)
    - [Taking Assets from Holding](#taking-assets-from-holding)
    - [Delivery Fee Calculation](#delivery-fee-calculation)
    - [Depositing into the Sovereign Account](#depositing-into-the-sovereign-account)
    - [The Retry Mechanism in deposit\_assets\_with\_retry](#the-retry-mechanism-in-deposit_assets_with_retry)
    - [Building and Sending the Remote Message](#building-and-sending-the-remote-message)
  - [5. Asset Location and Reanchoring](#5-asset-location-and-reanchoring)
    - [What Is an Asset Location?](#what-is-an-asset-location)
    - [Why Reanchoring Is Needed](#why-reanchoring-is-needed)
    - [The Reanchoring Code](#the-reanchoring-code)
  - [6. The send Function: Dispatching the Remote Message](#6-the-send-function-dispatching-the-remote-message)
    - [validate\_send: The Router Gateway](#validate_send-the-router-gateway)
  - [7. execute\_xcm\_transfer: The Two-Phase Execution](#7-execute_xcm_transfer-the-two-phase-execution)
- [Part II: Message Transport — From Storage to the Destination Chain](#part-ii-message-transport--from-storage-to-the-destination-chain)
  - [8. Why Three Transport Mechanisms?](#8-why-three-transport-mechanisms)
  - [9. XCMP Delivery: XcmpQueue::deliver](#9-xcmp-delivery-xcmpqueuedeliver)
  - [10. send\_fragment: Writing to Outbound Storage](#10-send_fragment-writing-to-outbound-storage)
    - [Page-Based Storage](#page-based-storage)
    - [Dynamic Fee Adjustment](#dynamic-fee-adjustment)
  - [11. The Message Journey Through the Relay Chain](#11-the-message-journey-through-the-relay-chain)
- [Part III: The Destination Chain — Receiving and Executing the Message](#part-iii-the-destination-chain--receiving-and-executing-the-message)
  - [12. ReserveAssetDeposited: Minting Backed Tokens](#12-reserveassetdeposited-minting-backed-tokens)
    - [Trust Verification](#trust-verification)
    - [Minting the Tokens](#minting-the-tokens)
    - [The Mint Implementation](#the-mint-implementation)
  - [13. ClearOrigin: Security Cleanup](#13-clearorigin-security-cleanup)
  - [14. BuyExecution: Paying for Execution on the Destination](#14-buyexecution-paying-for-execution-on-the-destination)
  - [15. DepositAsset: Crediting the Beneficiary](#15-depositasset-crediting-the-beneficiary)
- [Part IV: Reserve Transfer vs Teleport](#part-iv-reserve-transfer-vs-teleport)
  - [16. Key Differences](#16-key-differences)
  - [17. When to Use Each](#17-when-to-use-each)
- [Part V: Walkthrough — A Token Native to Parachain A Travels to Parachain B](#part-v-walkthrough--a-token-native-to-parachain-a-travels-to-parachain-b)
  - [18. Setting the Scene](#18-setting-the-scene)
  - [19. Step by Step: Origin Side (Parachain A)](#19-step-by-step-origin-side-parachain-a)
  - [20. Step by Step: Transport](#20-step-by-step-transport)
  - [21. Step by Step: Destination Side (Parachain B)](#21-step-by-step-destination-side-parachain-b)
  - [22. What Connects the Two Sides?](#22-what-connects-the-two-sides)
- [Part VI: The Complete Flow](#part-vi-the-complete-flow)
  - [23. End-to-End Summary](#23-end-to-end-summary)
  - [24. Complete Flow Diagram](#24-complete-flow-diagram)

---

## Introduction

This document traces the complete lifecycle of an XCM reserve transfer between two parachains. A user on **Parachain A** wants to send a token (native to Parachain A) to a beneficiary on **Parachain B**. Parachain A is the **reserve** — the chain where the token was created and where the real supply lives.

The flow involves three major phases:

1. **Origin chain execution** (Parachain A): The user's extrinsic triggers XCM instructions that withdraw tokens from the user, deposit them into Parachain B's sovereign account, and build a remote message.
2. **Message transport**: The XCM message is enqueued in Parachain A's outbound storage, routed through the relay chain, and delivered to Parachain B's inbound queue.
3. **Destination chain execution** (Parachain B): The message is executed in Parachain B's XCM VM, which mints backed tokens and deposits them into the beneficiary's account.

Throughout this document, we follow the actual code in the `polkadot-sdk` repository, tracing every function call from the extrinsic entry point to the final deposit.

---

# Part I: The Origin Chain — From Extrinsic to Outbound Message

---

## 1. Entry Point: do_reserve_transfer_assets

**File**: `polkadot/xcm/pallet-xcm/src/lib.rs`

When a user calls the `reserve_transfer_assets` extrinsic, it delegates to `do_reserve_transfer_assets`. This is the function that validates everything, builds the XCM messages, and triggers execution:

```rust
fn do_reserve_transfer_assets(
    origin: OriginFor<T>,
    dest: Box<VersionedLocation>,
    beneficiary: Box<VersionedLocation>,
    assets: Box<VersionedAssets>,
    fee_asset_item: u32,
    weight_limit: WeightLimit,
) -> DispatchResult {
    let origin_location = T::ExecuteXcmOrigin::ensure_origin(origin)?;
    let dest = (*dest).try_into().map_err(|()| Error::<T>::BadVersion)?;
    let beneficiary: Location = (*beneficiary).try_into().map_err(|()| Error::<T>::BadVersion)?;
    let assets: Assets = (*assets).try_into().map_err(|()| Error::<T>::BadVersion)?;

    ensure!(assets.len() <= MAX_ASSETS_FOR_TRANSFER, Error::<T>::TooManyAssets);
    let value = (origin_location, assets.into_inner());
    ensure!(T::XcmReserveTransferFilter::contains(&value), Error::<T>::Filtered);
    let (origin, assets) = value;
    let fee_asset_item = fee_asset_item as usize;
    let fees = assets.get(fee_asset_item as usize).ok_or(Error::<T>::Empty)?.clone();

    let (fees_transfer_type, assets_transfer_type) =
        Self::find_fee_and_assets_transfer_types(&assets, fee_asset_item, &dest)?;
    ensure!(assets_transfer_type != TransferType::Teleport, Error::<T>::Filtered);
    ensure!(assets_transfer_type == fees_transfer_type, Error::<T>::TooManyReserves);

    let (local_xcm, remote_xcm) = Self::build_xcm_transfer_type(
        origin.clone(),
        dest.clone(),
        Either::Left(beneficiary),
        assets,
        assets_transfer_type,
        FeesHandling::Batched { fees },
        weight_limit,
    )?;
    Self::execute_xcm_transfer(origin, dest, local_xcm, remote_xcm)
}
```

This function has three major phases: parameter conversion, validation, and message construction.

### Parameter Conversion

```rust
let origin_location = T::ExecuteXcmOrigin::ensure_origin(origin)?;
let dest = (*dest).try_into().map_err(|()| Error::<T>::BadVersion)?;
let beneficiary: Location = (*beneficiary).try_into().map_err(|()| Error::<T>::BadVersion)?;
let assets: Assets = (*assets).try_into().map_err(|()| Error::<T>::BadVersion)?;
```

The first line converts the Substrate `origin` (who signed the transaction) into an XCM `Location`. For example, if a user signs from their account on Parachain A, this produces something like `Location { parents: 0, interior: AccountId32 { id: user_account } }`.

The three `try_into()` calls convert `Versioned*` types (which exist for compatibility between XCM v2, v3, v4) into the concrete current types. If the user submits a message with an incompatible old version, this fails with `BadVersion`.

### Filters and Validations

```rust
ensure!(assets.len() <= MAX_ASSETS_FOR_TRANSFER, Error::<T>::TooManyAssets);
let value = (origin_location, assets.into_inner());
ensure!(T::XcmReserveTransferFilter::contains(&value), Error::<T>::Filtered);
```

The first check limits how many different assets can be sent in a single transfer. The second check is a runtime-configurable filter that determines **which assets this chain allows to be reserve-transferred**. If the chain doesn't permit reserve transfers of a particular asset, this is where it stops.

### Transfer Type Determination

```rust
let (fees_transfer_type, assets_transfer_type) =
    Self::find_fee_and_assets_transfer_types(&assets, fee_asset_item, &dest)?;
ensure!(assets_transfer_type != TransferType::Teleport, Error::<T>::Filtered);
ensure!(assets_transfer_type == fees_transfer_type, Error::<T>::TooManyReserves);
```

`find_fee_and_assets_transfer_types` determines whether the assets should go via **reserve transfer**, **teleport**, or **local transfer**, based on where each asset's reserve is relative to the destination.

Two critical validations follow: this is a `reserve_transfer_assets` call, so teleportable assets are rejected. And fees and assets must share the same reserve — you can't pay fees with a token whose reserve is Chain A and send assets whose reserve is Chain B.

Finally, `build_xcm_transfer_type` constructs **two XCM messages**: one for local execution (withdraw + deposit into sovereign account) and one for remote execution (notify the destination chain). These are passed to `execute_xcm_transfer`.

---

## 2. The XCM Executor: prepare_and_execute

**File**: `polkadot/xcm/xcm-executor/src/lib.rs`

The local XCM message is executed through the XCM executor's `prepare_and_execute`:

```rust
fn prepare_and_execute(
    origin: impl Into<Location>,
    message: Xcm<Call>,
    id: &mut XcmHash,
    weight_limit: Weight,
    weight_credit: Weight,
) -> Outcome {
    let pre = match Self::prepare(message, weight_limit) {
        Ok(x) => x,
        Err(error) => return Outcome::Error(error),
    };
    Self::execute(origin, pre, id, weight_credit)
}
```

`prepare` weighs the message — calculates how much execution weight it will consume. `execute` then runs it instruction by instruction via `process_instruction`. This is a trait defined on `ExecuteXcm`, and the concrete implementation lives in `XcmExecutor<Config>` in `xcm-executor/src/lib.rs`.

For a reserve transfer, the local message typically contains two instructions: `WithdrawAsset` and `DepositReserveAsset`.

---

## 3. WithdrawAsset: Taking Tokens from the User

**File**: `polkadot/xcm/xcm-executor/src/lib.rs` (inside `process_instruction`)

The first instruction withdraws tokens from the user's account:

```rust
WithdrawAsset(assets) => {
    let origin = self.origin_ref().ok_or(XcmError::BadOrigin)?;
    self.ensure_can_subsume_assets(assets.len())?;
    let mut total_surplus = Weight::zero();
    let mut withdrawn = AssetsInHolding::new();
    Config::TransactionalProcessor::process(|| {
        for asset in assets.inner() {
            let (credit, surplus) = Config::AssetTransactor::withdraw_asset_with_surplus(
                asset,
                origin,
                Some(&self.context),
            )?;
            withdrawn.subsume_assets(credit);
            total_surplus.saturating_accrue(surplus);
        }
        Ok(())
    })
    .and_then(|_| {
        self.holding.subsume_assets(withdrawn);
        self.total_surplus.saturating_accrue(total_surplus);
        Ok(())
    })
},
```

### The Withdraw Loop

The instruction iterates over each asset in the message. For each one, it calls `withdraw_asset_with_surplus`, which debits the user's account in the corresponding pallet's storage. If the transfer includes both DOT and USDT, the loop runs twice — once for each asset.

The `credit` returned is a representation of what was withdrawn. The `surplus` is weight that was overestimated — almost always zero unless special metering is involved.

After the loop, all withdrawn assets are moved into `self.holding` — the VM's temporary buffer.

### Which Pallet Handles the Withdraw?

`Config::AssetTransactor` is not a single type — it's an **associated type** configured in the runtime. Typically it's a tuple of adapters:

```rust
type AssetTransactor = (
    FungibleAdapter<Balances, IsConcreteFungible<NativeLocation>, ...>,
    FungiblesAdapter<Assets, ConvertedConcreteId<...>, ...>,
);
```

When `withdraw_asset_with_surplus` is called, the tuple tries each adapter in order. The first adapter that matches the asset handles the withdrawal. `FungibleAdapter` handles the native token (e.g., DOT) via `pallet-balances`. `FungiblesAdapter` handles multi-asset tokens (e.g., USDT) via `pallet-assets`.

The `withdraw_asset_with_surplus` function itself is a thin wrapper:

```rust
fn withdraw_asset_with_surplus(
    what: &Asset,
    who: &Location,
    maybe_context: Option<&XcmContext>,
) -> Result<(AssetsInHolding, Weight), XcmError> {
    Self::withdraw_asset(what, who, maybe_context).map(|assets| (assets, Weight::zero()))
}
```

It delegates to `withdraw_asset`, which in the case of `FungibleAdapter` does `Balances::withdraw(origin_account, amount)`, and in the case of `FungiblesAdapter` does `Assets::burn_from(asset_id, amount, origin_account)`. Both directly modify the on-chain storage.

### The Matcher: How It Decides Which Adapter Handles the Asset

**File**: `polkadot/xcm/xcm-builder/src/fungibles_adapter.rs`

Each adapter has a `Matcher` that inspects the XCM `Asset` and determines if it belongs to that adapter:

```rust
fn matches_fungibles(a: &Asset) -> result::Result<(AssetId, Balance), MatchError> {
    let (amount, id) = match (&a.fun, &a.id) {
        (Fungible(ref amount), AssetId(ref id)) => (amount, id),
        _ => return Err(MatchError::AssetNotHandled),
    };
    let what = ConvertAssetId::convert(id).ok_or(MatchError::AssetIdConversionFailed)?;
    let amount =
        ConvertBalance::convert(amount).ok_or(MatchError::AmountToBalanceConversionFailed)?;
    Ok((what, amount))
}
```

First, it checks if the asset is fungible. If it's an NFT, it returns `AssetNotHandled` and the tuple moves to the next adapter.

Then `ConvertAssetId::convert(id)` attempts to convert the XCM `Location` (e.g., `PalletInstance(50)/GeneralIndex(1984)`) into the numeric `AssetId` that `pallet-assets` uses internally (e.g., `1984`). If the Location doesn't match the expected pattern, it returns `None`, which becomes `AssetIdConversionFailed`, and the tuple tries the next adapter.

Finally, `ConvertBalance::convert` converts the XCM `u128` amount to the pallet's `Balance` type.

### How Does It Know If the Asset Is the Native Token (e.g. DOT)?

The native token uses a different adapter — `FungibleAdapter` (singular, not plural). Its matcher is much simpler: it checks if the asset's Location is `Here` (on the relay chain) or `Parent` (from a parachain's perspective). The native token doesn't have a numeric ID — it lives in `pallet-balances`, which only handles one token.

So in the tuple, `FungibleAdapter` matches first for the native token, and `FungiblesAdapter` matches for everything else.

### The Holding Register

After all withdrawals complete:

```rust
self.holding.subsume_assets(withdrawn);
```

The `holding` is an `AssetsInHolding` — a temporary in-memory buffer inside the XCM VM. It is **not** on-chain storage. It's a `Vec<Asset>` that keeps track of both the `AssetId` (the `Location`) and the `Fungible(amount)` for each asset. If 10 DOT and 500 USDT were withdrawn, the holding contains two entries with their respective Locations and amounts.

The tokens are now "in transit" — removed from the user's account but not yet deposited anywhere. The next instruction takes them from here.

---

## 4. DepositReserveAsset: Locking Tokens and Building the Remote Message

**File**: `polkadot/xcm/xcm-executor/src/lib.rs` (inside `process_instruction`)

This is the core instruction that makes a reserve transfer work. It takes tokens from the holding register, deposits them into the destination chain's sovereign account, and constructs + sends the remote XCM message:

```rust
DepositReserveAsset { assets, dest, xcm } => {
    self.transactional_process(|self_ref| {
        let mut assets = self_ref.holding.saturating_take(assets);

        let maybe_delivery_fee_from_assets = if self_ref.fees.is_empty()
            && !self_ref.fees_mode.jit_withdraw
        {
            self_ref.take_delivery_fee_from_assets(
                &mut assets, &dest, FeeReason::DepositReserveAsset, &xcm,
            )?
        } else {
            None
        };

        let mut message = Vec::with_capacity(xcm.len() + 2);
        Self::do_reserve_deposit_assets(
            assets,
            &dest,
            &mut message,
            Some(&self_ref.context),
        )?;
        message.push(ClearOrigin);
        message.extend(xcm.0.into_iter());

        if let Some(delivery_fee) = maybe_delivery_fee_from_assets {
            self_ref.holding.subsume_assets(delivery_fee);
        }
        self_ref.send(dest, Xcm(message), FeeReason::DepositReserveAsset)?;
        Ok(())
    })
},
```

### Taking Assets from Holding

```rust
let mut assets = self_ref.holding.saturating_take(assets);
```

The `assets` parameter is a filter — it specifies which assets to take from the holding. `saturating_take` removes them from the holding and returns them. Whatever is taken is no longer in the holding register.

### Delivery Fee Calculation

**File**: `polkadot/xcm/xcm-executor/src/lib.rs`

If fees haven't been paid through another mechanism, the delivery fee is deducted from the assets being transferred:

```rust
fn take_delivery_fee_from_assets(
    &self,
    assets: &mut AssetsInHolding,
    destination: &Location,
    reason: FeeReason,
    xcm: &Xcm<()>,
) -> Result<Option<AssetsInHolding>, XcmError> {
    let to_weigh_reanchored = Self::reanchored_assets(&assets, destination);
    let remote_instruction = match reason {
        FeeReason::DepositReserveAsset => ReserveAssetDeposited(to_weigh_reanchored),
        FeeReason::InitiateReserveWithdraw => WithdrawAsset(to_weigh_reanchored),
        FeeReason::InitiateTeleport => ReceiveTeleportedAsset(to_weigh_reanchored),
        _ => return Err(XcmError::NotHoldingFees),
    };

    let mut message_to_weigh = Vec::with_capacity(xcm.len() + 2);
    message_to_weigh.push(remote_instruction);
    message_to_weigh.push(ClearOrigin);
    message_to_weigh.extend(xcm.0.clone().into_iter());

    let (_, fee) =
        validate_send::<Config::XcmSender>(destination.clone(), Xcm(message_to_weigh))?;

    let maybe_delivery_fee = fee.get(0).map(|asset_needed_for_fees| {
        let asset_to_pay_for_fees =
            self.calculate_asset_for_delivery_fees(asset_needed_for_fees.clone());
        let delivery_fee = assets.saturating_take(asset_to_pay_for_fees.into());
        delivery_fee
    });
    Ok(maybe_delivery_fee)
}
```

The function builds a dummy message identical to what will actually be sent, then calls `validate_send` to ask the router: "how much would it cost to send this?" The router returns the fee, and that amount is deducted from the assets being transferred.

This means the fee is not paid separately — it comes out of the transfer itself. If a user sends 10 DOT, the delivery fee (say 0.01 DOT) is subtracted, and only 9.99 DOT get deposited into the sovereign account.

### Depositing into the Sovereign Account

`do_reserve_deposit_assets` deposits the tokens on-chain into the sovereign account of the destination chain, and pushes `ReserveAssetDeposited(reanchored_assets)` into the remote message.

The actual deposit uses `deposit_assets_with_retry`:

```rust
fn deposit_assets_with_retry(
    mut to_deposit: AssetsInHolding,
    beneficiary: &Location,
    context: Option<&XcmContext>,
) -> Result<Weight, XcmError> {
    let mut total_surplus = Weight::zero();
    let mut failed_deposits = AssetsInHolding::new();
    let assets: Vec<Asset> = to_deposit.assets_iter().collect();
    for asset in assets {
        let what = to_deposit.try_take(asset.into()).map_err(|_| XcmError::AssetNotFound)?;
        match Config::AssetTransactor::deposit_asset_with_surplus(what, &beneficiary, context) {
            Ok(surplus) => {
                total_surplus.saturating_accrue(surplus);
            },
            Err((unspent, _)) => {
                failed_deposits.subsume_assets(unspent);
            },
        }
    }
    // ... retry logic ...
}
```

### The Retry Mechanism in deposit_assets_with_retry

The function attempts to deposit each asset individually. If any deposit fails, it doesn't abort immediately — it marks the asset for retry. After the first pass, it retries all failed deposits:

```rust
let assets: Vec<Asset> = failed_deposits.assets_iter().collect();
for asset in assets {
    let what = failed_deposits.try_take(asset.into()).map_err(|_| XcmError::AssetNotFound)?;
    match Config::AssetTransactor::deposit_asset_with_surplus(what, &beneficiary, context) {
        Ok(surplus) => {
            total_surplus.saturating_accrue(surplus);
        },
        Err((_, error)) => {
            if !matches!(
                error,
                XcmError::FailedToTransactAsset(string)
                    if *string == *<&'static str>::from(sp_runtime::TokenError::BelowMinimum)
            ) {
                return Err(error);
            }
        },
    };
}
```

Why retry? There can be order dependencies. For example, a custom token might require the account to already exist, and the account is only created when the native token is deposited (paying the existential deposit). If the native token comes second in the list, the first token fails, but after depositing the native token the account exists and the retry succeeds.

On the second pass, `BelowMinimum` errors (dust amounts too small to be worth depositing) are silently ignored. Any other error causes a hard failure.

The `beneficiary` in this context is the **sovereign account** of Parachain B within Parachain A. The `deposit_asset_with_surplus` call uses the same adapter tuple — `FungibleAdapter` deposits via `Balances::deposit`, and `FungiblesAdapter` deposits via `Assets::mint`. The tokens are now locked on-chain in a special account that Parachain B controls.

### Building and Sending the Remote Message

After the deposit, the remote message is assembled:

```rust
let mut message = Vec::with_capacity(xcm.len() + 2);
Self::do_reserve_deposit_assets(assets, &dest, &mut message, Some(&self_ref.context))?;
message.push(ClearOrigin);
message.extend(xcm.0.into_iter());
self_ref.send(dest, Xcm(message), FeeReason::DepositReserveAsset)?;
```

`do_reserve_deposit_assets` pushes `ReserveAssetDeposited(reanchored_assets)` into the message. Then `ClearOrigin` is added, followed by the custom instructions from the original transfer (typically `BuyExecution` and `DepositAsset`).

The final message that travels to Parachain B looks like:

1. `ReserveAssetDeposited(assets)` — "tokens have been locked for you"
2. `ClearOrigin` — clear the sender's authority
3. `BuyExecution { fees, weight_limit }` — pay for execution on the destination
4. `DepositAsset { beneficiary }` — credit the final recipient

---

## 5. Asset Location and Reanchoring

### What Is an Asset Location?

Every asset in XCM is identified by a `Location` — a relative path that describes where the asset lives. For example, a token created in `pallet-assets` (at position 50 in the runtime) with ID 10 on Parachain A:

- From **Parachain A itself**: `Location { parents: 0, interior: PalletInstance(50)/GeneralIndex(10) }` — "here, pallet 50, asset 10"
- From **Parachain B**: `Location { parents: 1, interior: Parachain(A)/PalletInstance(50)/GeneralIndex(10) }` — "go up to the relay, then to Parachain A, pallet 50, asset 10"

The `PalletInstance(50)` comes from the position of `pallet-assets` in the `construct_runtime!` macro — it's hardcoded in the runtime, not stored in storage. The `GeneralIndex(10)` is the asset ID within that pallet — this is stored in `pallet-assets`' storage when the asset is created.

These Locations are **relative** to the chain looking at them, like filesystem paths. The same asset has different Locations depending on who's describing it.

### Why Reanchoring Is Needed

When Parachain A builds the remote message for Parachain B, the asset Locations need to be adjusted to Parachain B's perspective. Parachain B can't interpret a Location relative to Parachain A — it needs Locations relative to itself.

The user originally specified the asset as `PalletInstance(50)/GeneralIndex(10)` (relative to Parachain A, where the extrinsic was called). But the remote message will execute on Parachain B, which needs to see `parents: 1, Parachain(A)/PalletInstance(50)/GeneralIndex(10)` — "the asset lives on Parachain A".

### The Reanchoring Code

**File**: `polkadot/xcm/xcm-executor/src/lib.rs`

```rust
pub fn reanchored_assets(&self, target: &Location, context: &InteriorLocation) -> Assets {
    let mut assets: Vec<Asset> = self
        .fungible
        .iter()
        .filter_map(|(id, accounting)| match id.clone().reanchored(target, context) {
            Ok(new_id) => Some(Asset::from((new_id, Fungible(accounting.amount())))),
            Err(()) => None,
        })
        .chain(self.non_fungible.iter().filter_map(|(class, inst)| {
            match class.clone().reanchored(target, context) {
                Ok(new_class) => Some(Asset::from((new_class, *inst))),
                Err(()) => None,
            }
        }))
        .collect();
    assets.sort();
    assets.into()
}
```

For each asset, `id.clone().reanchored(target, context)` recalculates the Location. Internally, this does two things:

1. **`prepend(context)`**: Converts the relative Location to an **absolute** (universal) Location by prepending the current chain's position in the ecosystem.
2. **`simplify(target)`**: Converts the absolute Location to a **relative** Location from the target's perspective, removing redundant path segments.

For our example (Parachain A sending to Parachain B):

- **Original** (relative to Parachain A): `parents: 0, interior: PalletInstance(50)/GeneralIndex(10)`
- **After prepend** (absolute): `GlobalConsensus(Polkadot)/Parachain(A)/PalletInstance(50)/GeneralIndex(10)`
- **After simplify for Parachain B** (relative to destination): `parents: 1, interior: Parachain(A)/PalletInstance(50)/GeneralIndex(10)`

The amount stays the same. Only the Location changes. The origin chain doesn't need to "know" anything about the destination's internal structure — it simply recalculates the path based on the relative positions of the two chains.

---

## 6. The send Function: Dispatching the Remote Message

**File**: `polkadot/xcm/xcm-executor/src/lib.rs`

After the remote message is assembled, the VM's `send` function dispatches it:

```rust
fn send(
    &mut self,
    dest: Location,
    msg: Xcm<()>,
    reason: FeeReason,
) -> Result<XcmHash, XcmError> {
    let mut msg = msg;
    if !matches!(msg.last(), Some(SetTopic(_))) {
        let topic_id = self.context.topic_or_message_id();
        msg.0.push(SetTopic(topic_id.into()));
    }

    let (ticket, fee) = validate_send::<Config::XcmSender>(dest.clone(), msg)?;
    self.take_fee(fee, reason)?;
    match Config::XcmSender::deliver(ticket) {
        Ok(message_id) => {
            Config::XcmEventEmitter::emit_sent_event(
                self.original_origin.clone(), dest, None, message_id,
            );
            Ok(message_id)
        },
        Err(error) => {
            Config::XcmEventEmitter::emit_send_failure_event(
                self.original_origin.clone(), dest, error.clone(),
                self.context.topic_or_message_id(),
            );
            Err(error.into())
        },
    }
}
```

First, if the message doesn't already have a `SetTopic` at the end, one is appended. This acts as a correlation ID for tracking the message across chains.

Then `validate_send` asks the router whether it can deliver the message and what it costs. `take_fee` pays the delivery fee from the holding register.

Finally, `Config::XcmSender::deliver(ticket)` hands the message to the transport layer. The `XcmSender` is configured in the runtime and varies depending on the route — for parachain-to-parachain communication, it's typically `XcmpQueue`.

### validate_send: The Router Gateway

`validate_send` is a generic wrapper function that delegates to the configured sender:

```rust
pub fn validate_send<Sender: SendXcm>(dest: Location, msg: Xcm<()>) -> SendResult {
    Sender::validate(dest, msg)
}
```

It does nothing on its own — it calls `validate` on whichever `SendXcm` implementation is configured as `Config::XcmSender` in the runtime. The concrete implementation depends on the route:

| Route | Implementation | File |
|-------|---------------|------|
| Parachain → Relay | `impl SendXcm` in `ParachainSystem` | `cumulus/pallets/parachain-system/src/lib.rs` |
| Relay → Parachain | `impl SendXcm` in DMP sender | `polkadot/runtime/parachains/src/dmp.rs` |
| Parachain → Parachain | `impl SendXcm` in `XcmpQueue` | `cumulus/pallets/xcmp-queue/src/lib.rs` |

Each implementation does two things: it checks that the route exists (e.g., an HRMP channel is open), and it calculates the delivery price. The price is typically based on the message size in bytes and the weight of the instructions.

In production, the pricing uses implementations like `ExponentialPrice` or `FixedRateOfFungible` found in `cumulus/primitives/utility/src/lib.rs`, which charge per byte of the encoded message. In test environments, simpler implementations are used — for example, one that charges a `base_fee` multiplied by the number of `parents` in the asset Location:

```rust
fn price_for_delivery(_: Self::Id, msg: &Xcm<()>) -> Assets {
    let base_fee: super::Balance = 1_000_000;
    let parents = msg.iter().find_map(|xcm| match xcm {
        ReserveAssetDeposited(assets) => {
            let AssetId(location) = &assets.inner().first().unwrap().id;
            Some(location.parents)
        },
        _ => None,
    });
    let amount = base_fee
        .saturating_add(base_fee.saturating_mul(parents.unwrap_or(0) as super::Balance));
    (A::get(), amount).into()
}
```

The `validate` call returns a `ticket` (the message ready for delivery) and the `fee` (what it costs). The ticket is then passed to `deliver` to actually enqueue the message in storage.

---

## 7. execute_xcm_transfer: The Two-Phase Execution

**File**: `polkadot/xcm/pallet-xcm/src/lib.rs`

Back in `pallet-xcm`, `execute_xcm_transfer` orchestrates both phases:

```rust
fn execute_xcm_transfer(
    origin: Location,
    dest: Location,
    mut local_xcm: Xcm<<T as Config>::RuntimeCall>,
    remote_xcm: Option<Xcm<()>>,
) -> DispatchResult {
    let weight = T::Weigher::weight(&mut local_xcm, Weight::MAX).map_err(|_| {
        Error::<T>::UnweighableMessage
    })?;
    let mut hash = local_xcm.using_encoded(sp_io::hashing::blake2_256);
    let outcome = T::XcmExecutor::prepare_and_execute(
        origin.clone(), local_xcm, &mut hash, weight, weight,
    );
    Self::deposit_event(Event::Attempted { outcome: outcome.clone() });
    outcome.clone().ensure_complete().map_err(|error| {
        Error::<T>::LocalExecutionIncompleteWithError {
            index: error.index,
            error: error.error.into(),
        }
    })?;

    if let Some(remote_xcm) = remote_xcm {
        let (ticket, price) =
            validate_send::<T::XcmRouter>(dest.clone(), remote_xcm.clone())?;
        if origin != Here.into_location() {
            Self::charge_fees(origin.clone(), price.clone())?;
        }
        let message_id = T::XcmRouter::deliver(ticket)?;
        let e = Event::Sent {
            origin, destination: dest, message: remote_xcm, message_id,
        };
        Self::deposit_event(e);
    }
    Ok(())
}
```

**Phase 1 — Local execution**: `prepare_and_execute` runs the local XCM message (`WithdrawAsset` + `DepositReserveAsset`). This withdraws tokens from the user and deposits them into the sovereign account. If the local XCM includes a `send` (as part of `DepositReserveAsset`), the message is also dispatched from within the VM.

**Phase 2 — Remote message delivery** (if applicable): If `build_xcm_transfer_type` returned a separate `remote_xcm`, it's validated and delivered here. `validate_send` checks routing and calculates cost, `charge_fees` pays for delivery (unless the origin is the relay chain itself), and `deliver` hands the message to the transport layer.

After `execute_xcm_transfer` completes, the tokens are locked in the sovereign account and the remote message is enqueued for transport.

---

# Part II: Message Transport — From Storage to the Destination Chain

---

## 8. Why Three Transport Mechanisms?

The transport mechanism depends on the relationship between sender and receiver:

| Route | Mechanism | Direction |
|-------|-----------|-----------|
| Parachain → Relay chain | UMP (Upward Message Passing) | "Up" to the relay |
| Relay chain → Parachain | DMP (Downward Message Passing) | "Down" from the relay |
| Parachain → Parachain | HRMP/XCMP | "Horizontal" via the relay |

Each has different infrastructure because the physical path is different. UMP uses the parachain's validation data. DMP uses a queue in the relay's state per parachain. HRMP uses channels that must be explicitly opened between parachains, with queues stored on the relay chain.

For a parachain-to-parachain reserve transfer, the message travels via HRMP/XCMP.

---

## 9. XCMP Delivery: XcmpQueue::deliver

**File**: `cumulus/pallets/xcmp-queue/src/lib.rs`

When `Config::XcmSender::deliver` is called for a parachain-to-parachain message:

```rust
fn deliver(
    (recipient, xcm): (ParaId, VersionedXcm<()>),
) -> Result<XcmHash, SendError> {
    let hash = xcm.using_encoded(sp_io::hashing::blake2_256);
    let mut encoding = XcmEncoding::Simple;
    let mut all_channels = <OutboundXcmpStatus<T>>::get();
    if let Some(channel_details) = Self::try_get_outbound_channel(&mut all_channels, recipient) {
        if channel_details.flags.has_concatenated_opaque_versioned_xcm_support() {
            encoding = XcmEncoding::Double;
        }
    }
    let result = match encoding {
        XcmEncoding::Simple => {
            Self::send_fragment(recipient, XcmpMessageFormat::ConcatenatedVersionedXcm, xcm)
        },
        XcmEncoding::Double => Self::send_fragment(
            recipient,
            XcmpMessageFormat::ConcatenatedOpaqueVersionedXcm,
            xcm.encode(),
        ),
    };
    match result {
        Ok(_) => {
            Self::deposit_event(Event::XcmpMessageSent { message_hash: hash });
            Ok(hash)
        },
        Err(e) => Err(SendError::Transport(e.into())),
    }
}
```

The message is hashed for identification, then a format negotiation happens: if the destination parachain supports the newer opaque encoding (`Double`), that's used. Otherwise, the simple format is used. This is a compatibility mechanism between different parachain versions.

The actual storage write happens in `send_fragment`.

---

## 10. send_fragment: Writing to Outbound Storage

**File**: `cumulus/pallets/xcmp-queue/src/lib.rs`

```rust
fn send_fragment<Fragment: Encode>(
    recipient: ParaId,
    format: XcmpMessageFormat,
    fragment: Fragment,
) -> Result<u32, MessageSendError> {
    let mut encoded_fragment = fragment.encode();
    let channel_info =
        T::ChannelInfo::get_channel_info(recipient).ok_or(MessageSendError::NoChannel)?;
    let max_message_size = channel_info.max_message_size.min(T::MaxPageSize::get()) as usize;

    if size_to_check > max_message_size {
        return Err(MessageSendError::TooBig);
    }

    let mut all_channels = <OutboundXcmpStatus<T>>::get();
    let channel_details =
        Self::try_get_or_insert_outbound_channel(&mut all_channels, recipient)
            .ok_or(MessageSendError::TooManyChannels)?;

    // Try to fit in an existing page
    let mut existing_page = None;
    'existing_page_check: {
        if channel_details.last_index > channel_details.first_index {
            let page = OutboundXcmpMessages::<T>::get(recipient, channel_details.last_index - 1);
            if XcmpMessageFormat::decode(&mut &page[..]) != Ok(format) {
                break 'existing_page_check;
            }
            if page.len() + encoded_fragment.len() > max_message_size {
                break 'existing_page_check;
            }
            existing_page = Some(page.into_inner());
        }
    }

    let mut current_page = existing_page.unwrap_or_else(|| {
        channel_details.last_index += 1;
        format.encode()
    });
    current_page.append(&mut encoded_fragment);

    <OutboundXcmpMessages<T>>::insert(recipient, channel_details.last_index - 1, current_page);
    <OutboundXcmpStatus<T>>::put(all_channels);

    // Dynamic fee adjustment
    let threshold = channel_info.max_total_size / delivery_fee_constants::THRESHOLD_FACTOR;
    if total_size > threshold {
        Self::increase_fee_factor(recipient, encoded_fragment_len as u128);
    }
    Ok(page_count)
}
```

### Page-Based Storage

The function first verifies that an HRMP channel exists with the recipient and that the message fits within the channel's size limits.

Messages are stored in **pages**. The function tries to append the new message to the last existing page if there's room and the format matches. If it doesn't fit, a new page is created. Multiple XCM messages to the same destination can share a single page — this is an optimization to avoid per-message overhead.

The storage layout is `OutboundXcmpMessages: Map<(ParaId, page_index), Vec<u8>>`. This is the **parachain origin's storage**, not the relay chain's.

### Dynamic Fee Adjustment

```rust
let threshold = channel_info.max_total_size / delivery_fee_constants::THRESHOLD_FACTOR;
if total_size > threshold {
    Self::increase_fee_factor(recipient, encoded_fragment_len as u128);
}
```

If the outbound queue is filling up past a threshold, the delivery fee factor increases. This is an anti-spam mechanism — if a parachain is saturating a channel, it becomes more expensive to send additional messages. Similar in principle to EIP-1559's dynamic base fee.

---

## 11. The Message Journey Through the Relay Chain

After `send_fragment`, the message sits in the **origin parachain's storage** as `OutboundXcmpMessages`. The journey continues:

1. The **collator** of Parachain A builds a block that includes these outbound messages.
2. The collator presents the block to the relay chain as a **parachain candidate**.
3. The relay chain's **validators** validate the block and see the outbound HRMP messages.
4. The relay chain places these messages in the **inbound HRMP queue** of Parachain B — this is stored in the relay chain's state.
5. Parachain B's **collator**, when building its next block, fetches these inbound messages (via `retrieve_all_inbound_hrmp_channel_contents`) and includes them in the `ParachainInherentData`.
6. Parachain B's runtime processes them in `set_validation_data` → `enqueue_inbound_horizontal_messages` → `handle_xcmp_messages`, eventually executing them in `on_idle` via `pallet-message-queue`.

The message passes through **three storages**:

| Storage | Location | Key |
|---------|----------|-----|
| `OutboundXcmpMessages` | Parachain A | `(recipient_para_id, page_index)` |
| HRMP channel queue | Relay chain | `(sender_para_id, recipient_para_id)` |
| `pallet-message-queue` pages | Parachain B | `(origin, page_index)` |

---

# Part III: The Destination Chain — Receiving and Executing the Message

---

When the XCM message reaches Parachain B and is dequeued for execution by `pallet-message-queue`, the XCM VM processes each instruction in order.

## 12. ReserveAssetDeposited: Minting Backed Tokens

**File**: `polkadot/xcm/xcm-executor/src/lib.rs` (inside `process_instruction`)

```rust
ReserveAssetDeposited(assets) => {
    self.ensure_can_subsume_assets(assets.len())?;
    Config::TransactionalProcessor::process(|| {
        let origin = self.origin_ref().ok_or(XcmError::BadOrigin)?;
        let mut minted_assets = AssetsInHolding::new();
        for asset in assets.inner() {
            ensure!(
                Config::IsReserve::contains(asset, origin),
                XcmError::UntrustedReserveLocation
            );
            Config::AssetTransactor::mint_asset(asset, &self.context)
                .map(|minted| minted_assets.subsume_assets(minted))?;
        }
        self.holding.subsume_assets(minted_assets);
        Ok(())
    })
},
```

### Trust Verification

```rust
ensure!(
    Config::IsReserve::contains(asset, origin),
    XcmError::UntrustedReserveLocation
);
```

This is the critical security check. `IsReserve` is a runtime-configurable filter. The `origin` is the chain that sent the message (Parachain A). For each asset, the destination asks: "do I trust this chain as the legitimate reserve for this asset?"

If Parachain B hasn't configured Parachain A as a valid reserve for this particular asset, the instruction fails with `UntrustedReserveLocation` and no tokens are minted. A random chain sending fake `ReserveAssetDeposited` messages is rejected here.

### Minting the Tokens

```rust
Config::AssetTransactor::mint_asset(asset, &self.context)
    .map(|minted| minted_assets.subsume_assets(minted))?;
```

For each trusted asset, `mint_asset` creates new tokens on Parachain B. These are **not** tokens taken from any account — they are minted from nothing. They represent the tokens locked in the sovereign account on Parachain A.

After minting, the tokens go into the holding register, just like `WithdrawAsset` on the origin chain.

### The Mint Implementation

**File**: `polkadot/xcm/xcm-builder/src/fungibles_adapter.rs`

```rust
fn mint_asset(what: &Asset, context: &XcmContext) -> Result<AssetsInHolding, XcmError> {
    let (asset_id, amount) = Matcher::matches_fungibles(what)?;
    let credit = Assets::issue(asset_id, amount);
    Ok(AssetsInHolding::new_from_fungible_credit(what.id.clone(), Box::new(credit)))
}
```

The same `matches_fungibles` matcher is used. It takes the reanchored Location (e.g., `parents: 1, Parachain(A)/PalletInstance(50)/GeneralIndex(10)`) and converts it to a local asset ID. This requires that Parachain B has **pre-configured** a mapping: a local asset must already exist that maps to this Location.

`Assets::issue(asset_id, amount)` is a call to `pallet-assets` that increases the total supply of the local asset. There is no on-chain link between this minted token and the locked token on Parachain A. The relationship is purely by convention — same Location identifies the same asset across chains, and the security guarantee comes from `IsReserve` + the XCM protocol ensuring the amounts match.

If Parachain B hasn't registered a local asset that maps to the incoming Location, `matches_fungibles` fails with `AssetIdConversionFailed` and the mint doesn't happen.

---

## 13. ClearOrigin: Security Cleanup

```rust
ClearOrigin => {
    self.context.origin = None;
},
```

Sets the origin to `None`. After this, `self.origin_ref()` returns `None`.

This is a security measure. The origin up to this point was the chain that sent the message (Parachain A). `ReserveAssetDeposited` needed it to verify the reserve. But the instructions that follow (`BuyExecution`, `DepositAsset`) should not execute with the authority of the sending chain. Without `ClearOrigin`, a malicious message could include additional instructions that act on behalf of the sender.

---

## 14. BuyExecution: Paying for Execution on the Destination

**File**: `polkadot/xcm/xcm-executor/src/lib.rs` (inside `process_instruction`)

```rust
BuyExecution { fees, weight_limit } => {
    let Some(weight) = Option::<Weight>::from(weight_limit) else { return Ok(()) };
    self.asset_used_in_buy_execution = Some(fees.id.clone());

    self.transactional_process(|self_ref| {
        let max_fee =
            self_ref.holding.try_take(fees.clone().into()).map_err(|e| {
                XcmError::NotHoldingFees
            })?;
        let unspent = self_ref.trader.buy_weight(weight, max_fee, &self_ref.context)
            .map_err(|(unspent, e)| {
                self_ref.holding.subsume_assets(unspent);
                e
            })?;
        self_ref.holding.subsume_assets(unspent);
        Ok(())
    })
},
```

If `weight_limit` is `Unlimited`, execution is free — something else already authorized it. In the normal case, there's a concrete weight limit.

The function records which asset is being used for fees (for later reference if delivery fees are needed). Then it takes the fee asset from the holding register using `try_take`. If the holding doesn't have enough of the specified asset, it fails with `NotHoldingFees`.

`buy_weight` is the core fee calculation: it converts tokens into execution weight. The `trader` is a runtime-configurable component that defines the exchange rate between tokens and weight. If the token offered isn't sufficient for the requested weight, it fails and the tokens are returned to holding.

Any overpayment (`unspent`) goes back into the holding register. If 0.01 DOT was offered but only 0.005 DOT was needed, the remaining 0.005 DOT stays in holding and will be available for the beneficiary in the next instruction.

---

## 15. DepositAsset: Crediting the Beneficiary

**File**: `polkadot/xcm/xcm-executor/src/lib.rs` (inside `process_instruction`)

```rust
DepositAsset { assets, beneficiary } => {
    self.transactional_process(|self_ref| {
        let deposited = self_ref.holding.saturating_take(assets);
        let surplus = Self::deposit_assets_with_retry(
            deposited,
            &beneficiary,
            Some(&self_ref.context),
        )?;
        self_ref.total_surplus.saturating_accrue(surplus);
        Ok(())
    })
},
```

This is the final step. It takes whatever remains in the holding (the minted tokens minus execution fees) and deposits them into the beneficiary's account using `deposit_assets_with_retry` — the same function used on the origin chain, but now the beneficiary is the **user's account** on Parachain B, not a sovereign account.

`deposit_asset_with_surplus` uses the adapter tuple to deposit into the right pallet. The user now has tokens in their account on Parachain B.

---

# Part IV: Reserve Transfer vs Teleport

---

## 16. Key Differences

A **teleport** destroys tokens on the origin and creates them on the destination. There is no sovereign account and no reserve backing.

| Aspect | Reserve Transfer | Teleport |
|--------|-----------------|----------|
| Origin instruction | `DepositReserveAsset` — deposits in sovereign account | `InitiateTeleport` — burns tokens |
| Destination instruction | `ReserveAssetDeposited` — mints backed by reserve | `ReceiveTeleportedAsset` — mints without external backing |
| Token lifecycle | Locked on origin, representation minted on destination | Destroyed on origin, created on destination |
| Trust requirement | `IsReserve` — destination trusts origin as reserve | `IsTeleporter` — destination trusts origin burned the tokens |
| Trust level | Moderate — backed by locked tokens | Maximum — no backing, pure trust |
| Use case | Cross-chain transfers between independent chains | Transfers between tightly coupled chains (relay ↔ Asset Hub) |

The trust verification on the destination changes accordingly:

```rust
// Reserve transfer
ensure!(Config::IsReserve::contains(asset, origin), XcmError::UntrustedReserveLocation);

// Teleport
ensure!(Config::IsTeleporter::contains(asset, origin), XcmError::UntrustedTeleportLocation);
```

## 17. When to Use Each

**Reserve transfers** are used when chains don't fully trust each other. The tokens remain locked on the reserve chain, providing a guarantee. If the destination chain has a bug, the tokens on the reserve are unaffected.

**Teleports** require complete mutual trust. If the origin chain claims to have burned tokens but didn't, the destination mints tokens from nothing. This is why teleports are only used between chains that share governance — for example, the Polkadot relay chain and Asset Hub, which are both governed by DOT holders.

---

# Part V: Walkthrough — A Token Native to Parachain A Travels to Parachain B

---

This section walks through the reserve transfer step by step in plain language, clarifying who does what at each stage.

## 18. Setting the Scene

The token was created on Parachain A — it's native to Parachain A. This means Parachain A is the **reserve**: the chain where the real supply lives. Parachain B doesn't have this token natively; it will hold a representation backed by tokens locked on Parachain A.

There are only two actors: Parachain A (origin, reserve) and Parachain B (destination). No third chain like Asset Hub is involved.

## 19. Step by Step: Origin Side (Parachain A)

**Step 1** — The user on Parachain A calls `reserve_transfer_assets` with:

- **dest**: Parachain B
- **beneficiary**: the recipient's account on Parachain B
- **asset**: the token, with a local Location like `PalletInstance(50)/GeneralIndex(10)` — this points to pallet-assets position 50, asset ID 10, on this chain

**Step 2** — `WithdrawAsset` executes. The asset transactor debits the user's account in pallet-assets (or pallet-balances for native tokens). The tokens move from the user's on-chain account into the holding register — a temporary in-memory buffer inside the XCM VM. The tokens are no longer in the user's account but haven't arrived anywhere yet.

**Step 3** — `DepositReserveAsset` executes. It takes the tokens from the holding register and deposits them into the **sovereign account of Parachain B** within Parachain A. This is a real account on Parachain A that Parachain B controls. The tokens are now locked: they're on-chain, in Parachain A's storage, but the user no longer owns them.

**Step 4** — In the same instruction, the remote message is built. Before including the asset in the message, the Location is **reanchored** to Parachain B's perspective. The original Location was `PalletInstance(50)/GeneralIndex(10)` (relative to Parachain A). After reanchoring, it becomes `parents: 1, Parachain(A)/PalletInstance(50)/GeneralIndex(10)` — "go up to the relay, then to Parachain A, pallet 50, asset 10". This is how Parachain B needs to see the asset Location.

The origin chain doesn't "know" Parachain B's internal structure. The reanchoring is a pure path calculation based on the relative positions of the two chains. The `PalletInstance(50)` and `GeneralIndex(10)` were in the Location from the start — they come from the user's original extrinsic. The reanchor only adds the routing prefix (`parents: 1, Parachain(A)/...`) so the destination can find its way back to the asset's origin.

**Step 5** — The complete message is assembled:

1. `ReserveAssetDeposited` with the reanchored asset — tells the destination "tokens are locked for you"
2. `ClearOrigin` — clears the sender's authority
3. `BuyExecution` — pay for execution on the destination
4. `DepositAsset` — credit the beneficiary

**Step 6** — The message is enqueued in `OutboundXcmpMessages` on Parachain A's storage. It sits there as bytes indexed by `(recipient_para_id, page_index)`, waiting for the collator to include it in the next block.

## 20. Step by Step: Transport

**Step 7** — Parachain A's collator builds a block that includes the outbound messages. The relay chain validators validate the block, see the HRMP messages, and route them into Parachain B's inbound queue on the relay chain.

**Step 8** — Parachain B's collator, when building its next block, fetches the inbound messages from the relay chain and includes them in the `ParachainInherentData`. The runtime processes them through `set_validation_data` → `enqueue_inbound_horizontal_messages` → `handle_xcmp_messages`, eventually storing them in `pallet-message-queue` for execution.

## 21. Step by Step: Destination Side (Parachain B)

**Step 9** — During `on_idle`, `pallet-message-queue` dequeues the XCM message and hands it to the XCM executor.

**Step 10** — `ReserveAssetDeposited` executes. First, it checks `IsReserve`: does Parachain B trust Parachain A as a legitimate reserve for this asset? This is configured in Parachain B's runtime — if Parachain A isn't listed, the message is rejected with `UntrustedReserveLocation`.

If trusted, it mints tokens. The `matches_fungibles` matcher takes the reanchored Location (`parents: 1, Parachain(A)/PalletInstance(50)/GeneralIndex(10)`) and converts it to a **local asset ID** on Parachain B. This requires that Parachain B has **pre-registered** a local asset that maps to this Location. If no such mapping exists, `ConvertAssetId::convert` returns `None` and the mint fails.

If the mapping exists, `pallet-assets::issue(local_asset_id, amount)` creates new tokens from nothing. These tokens go into the holding register.

**Step 11** — `ClearOrigin` executes — sets the origin to `None` for security.

**Step 12** — `BuyExecution` executes — takes a portion of the minted tokens from holding to pay for execution weight on Parachain B.

**Step 13** — `DepositAsset` executes — takes whatever remains in holding (minted tokens minus fees) and deposits them into the beneficiary's account on Parachain B.

The beneficiary now holds tokens on Parachain B, backed by the equivalent amount locked in Parachain B's sovereign account on Parachain A.

## 22. What Connects the Two Sides?

There is **no on-chain link** between the locked tokens on Parachain A and the minted tokens on Parachain B. No pointer, no reference, no proof stored in state. The relationship is maintained by:

- **The same Asset Location** — both sides use the same Location to identify the token, adjusted by reanchoring
- **Matching amounts** — whatever was locked on A equals what was minted on B
- **Trust in the protocol** — `IsReserve` says "I trust that Parachain A actually locked the tokens before sending this message"
- **Relay chain validation** — the relay chain validates Parachain A's block, ensuring the outbound message is legitimate

If Parachain A had a bug and sent a `ReserveAssetDeposited` without actually locking tokens, Parachain B would mint unbacked tokens. The security comes from running audited code and the relay chain's validation, not from cryptographic proofs of the locking.

---

# Part VI: The Complete Flow

---

## 23. End-to-End Summary

A user on Parachain A sends a native token to a beneficiary on Parachain B:

**Origin chain (Parachain A):**

1. User calls `reserve_transfer_assets` extrinsic with destination, beneficiary, asset, and fee parameters.
2. `do_reserve_transfer_assets` validates parameters, checks filters, determines transfer type, and builds local + remote XCM messages.
3. `WithdrawAsset` executes: debits the user's account via the asset transactor (pallet-balances or pallet-assets) and places tokens in the holding register.
4. `DepositReserveAsset` executes: takes tokens from holding, deducts delivery fees, deposits into Parachain B's sovereign account, reanchors asset Locations, builds the remote message, and sends it via `XcmpQueue`.
5. `send_fragment` writes the message bytes to `OutboundXcmpMessages` in Parachain A's storage.

**Transport:**

6. Parachain A's collator includes the outbound messages in its block.
7. Relay chain validators validate the block and route the HRMP messages to Parachain B's inbound queue.
8. Parachain B's collator fetches the inbound messages and includes them in the `ParachainInherentData`.

**Destination chain (Parachain B):**

9. `set_validation_data` processes the inbound HRMP messages, enqueuing them in `pallet-message-queue`.
10. During `on_idle`, the message queue dequeues the XCM message and hands it to the XCM executor.
11. `ReserveAssetDeposited` executes: verifies Parachain A is a trusted reserve via `IsReserve`, mints tokens via `pallet-assets::issue`, places them in holding.
12. `ClearOrigin` executes: clears the sender's authority for security.
13. `BuyExecution` executes: takes a portion from holding to pay for execution weight.
14. `DepositAsset` executes: deposits the remaining tokens from holding into the beneficiary's account.

The beneficiary now holds tokens on Parachain B, backed by the equivalent amount locked in Parachain B's sovereign account on Parachain A.

---

## 24. Complete Flow Diagram

```
═══════════════════════════════════════════════════════════════════
                PARACHAIN A (ORIGIN / RESERVE)
═══════════════════════════════════════════════════════════════════

  User calls reserve_transfer_assets(dest, beneficiary, assets)
       │
       ├── do_reserve_transfer_assets
       │       ├── Convert & validate parameters
       │       ├── Check XcmReserveTransferFilter
       │       ├── find_fee_and_assets_transfer_types
       │       └── build_xcm_transfer_type → (local_xcm, remote_xcm)
       │
       └── execute_xcm_transfer
               │
               ├── Phase 1: XcmExecutor::prepare_and_execute(local_xcm)
               │       │
               │       ├── WithdrawAsset
               │       │       ├── AssetTransactor::withdraw_asset
               │       │       │       ├── FungibleAdapter → pallet-balances (native token)
               │       │       │       └── FungiblesAdapter → pallet-assets (other tokens)
               │       │       └── Tokens move: user account → holding register
               │       │
               │       └── DepositReserveAsset
               │               ├── holding.saturating_take(assets)
               │               ├── take_delivery_fee_from_assets
               │               │       ├── Build dummy message
               │               │       ├── validate_send → get delivery cost
               │               │       └── Deduct fee from assets
               │               ├── do_reserve_deposit_assets
               │               │       ├── reanchored_assets → adjust Locations
               │               │       ├── deposit_assets_with_retry
               │               │       │       └── Tokens move: holding → sovereign account
               │               │       └── Push ReserveAssetDeposited to message
               │               ├── Push ClearOrigin to message
               │               ├── Append BuyExecution + DepositAsset
               │               └── send(dest, message)
               │                       ├── validate_send → ticket + fee
               │                       ├── take_fee
               │                       └── XcmSender::deliver(ticket)
               │
               └── Phase 2: Message enqueued
                       └── XcmpQueue::deliver
                               └── send_fragment
                                       └── OutboundXcmpMessages::insert(
                                               recipient, page_index, bytes)

═══════════════════════════════════════════════════════════════════
                    RELAY CHAIN (TRANSPORT)
═══════════════════════════════════════════════════════════════════

  Validators validate Parachain A's block
       │
       └── Route HRMP messages
               └── Place in Parachain B's inbound queue

═══════════════════════════════════════════════════════════════════
                PARACHAIN B (DESTINATION)
═══════════════════════════════════════════════════════════════════

  Collator fetches inbound HRMP messages
       │
       └── set_validation_data
               └── enqueue_inbound_horizontal_messages
                       └── handle_xcmp_messages
                               └── pallet-message-queue::enqueue

  ... later in the same block (on_idle) ...

  pallet-message-queue::service_queues
       │
       └── XcmExecutor processes message:
               │
               ├── ReserveAssetDeposited(assets)
               │       ├── IsReserve::contains(asset, origin) → trust check
               │       ├── AssetTransactor::mint_asset
               │       │       ├── matches_fungibles → convert Location to local ID
               │       │       └── pallet-assets::issue(asset_id, amount)
               │       └── Minted tokens → holding register
               │
               ├── ClearOrigin
               │       └── context.origin = None
               │
               ├── BuyExecution { fees, weight_limit }
               │       ├── holding.try_take(fees)
               │       ├── trader.buy_weight(weight, max_fee)
               │       └── Unspent fees → back to holding
               │
               └── DepositAsset { beneficiary }
                       ├── holding.saturating_take(assets)
                       └── deposit_assets_with_retry
                               └── Tokens move: holding → beneficiary account

  ✓ Beneficiary now holds tokens on Parachain B
    backed by locked tokens in sovereign account on Parachain A
```