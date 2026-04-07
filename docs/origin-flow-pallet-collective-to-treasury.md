---
layout:
  width: default
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
  tags:
    visible: true
---

# Origin Flow: pallet-collective to pallet-treasury

## Origin Flow Deep Dive: From pallet-collective to pallet-treasury

> A step-by-step walkthrough of how a custom origin is born inside `pallet-collective`, transforms into a runtime-level origin, travels through the dispatch system, and gets validated by the destination pallet. More importantly: a deep exploration of the **architectural pattern** that underpins all of FRAME — pallet-local types wrapped into runtime-global enums via auto-generated `From` impls, with trait-based indirection at every boundary.

### Table of Contents

* [Origin Flow Deep Dive: From pallet-collective to pallet-treasury](origin-flow-pallet-collective-to-treasury.md#origin-flow-deep-dive-from-pallet-collective-to-pallet-treasury)
  * [Table of Contents](origin-flow-pallet-collective-to-treasury.md#table-of-contents)
  * [Introduction](origin-flow-pallet-collective-to-treasury.md#introduction)
* [Part I: The Universal Pattern](origin-flow-pallet-collective-to-treasury.md#part-i-the-universal-pattern)
  * [1. The Core Architecture: Local Types → Global Enums](origin-flow-pallet-collective-to-treasury.md#1-the-core-architecture-local-types--global-enums)
  * [2. From as the Exit Door: How Pallet Types Escape](origin-flow-pallet-collective-to-treasury.md#2-from-as-the-exit-door-how-pallet-types-escape)
  * [3. The Constraint Funnel: Trait → Config → Runtime](origin-flow-pallet-collective-to-treasury.md#3-the-constraint-funnel-trait--config--runtime)
  * [4. Compiler Inference: How Generic Types Resolve Without Explicit Passing](origin-flow-pallet-collective-to-treasury.md#4-compiler-inference-how-generic-types-resolve-without-explicit-passing)
* [Part II: Origin Definition and Registration](origin-flow-pallet-collective-to-treasury.md#part-ii-origin-definition-and-registration)
  * [5. Defining the Custom Origin in pallet-collective](origin-flow-pallet-collective-to-treasury.md#5-defining-the-custom-origin-in-pallet-collective)
  * [6. Registering the Origin with #\[pallet::origin\]](origin-flow-pallet-collective-to-treasury.md#6-registering-the-origin-with-palletorigin)
  * [7. construct\_runtime! Generates the RuntimeOrigin Enum](origin-flow-pallet-collective-to-treasury.md#7-construct_runtime-generates-the-runtimeorigin-enum)
  * [8. Automatic From Generation for Each Variant](origin-flow-pallet-collective-to-treasury.md#8-automatic-from-generation-for-each-variant)
* [Part III: Wiring Origins and Calls in Config](origin-flow-pallet-collective-to-treasury.md#part-iii-wiring-origins-and-calls-in-config)
  * [9. Wiring RuntimeOrigin in the Pallet Config](origin-flow-pallet-collective-to-treasury.md#9-wiring-runtimeorigin-in-the-pallet-config)
  * [10. The Dispatchable Trait: How dispatch Knows Its Origin Type](origin-flow-pallet-collective-to-treasury.md#10-the-dispatchable-trait-how-dispatch-knows-its-origin-type)
  * [11. The Proposal: A Runtime Call Stored On-Chain](origin-flow-pallet-collective-to-treasury.md#11-the-proposal-a-runtime-call-stored-on-chain)
* [Part IV: The Dispatch Journey](origin-flow-pallet-collective-to-treasury.md#part-iv-the-dispatch-journey)
  * [12. Voting Ends: Constructing the Origin](origin-flow-pallet-collective-to-treasury.md#12-voting-ends-constructing-the-origin)
  * [13. The .into() Call: From Local Origin to Global Origin](origin-flow-pallet-collective-to-treasury.md#13-the-into-call-from-local-origin-to-global-origin)
  * [14. proposal.dispatch(origin): Entering the Runtime Dispatch](origin-flow-pallet-collective-to-treasury.md#14-proposaldispatchorigin-entering-the-runtime-dispatch)
  * [15. Runtime Dispatch: The Call Router](origin-flow-pallet-collective-to-treasury.md#15-runtime-dispatch-the-call-router)
  * [16. Pallet-Level Dispatch: Reaching the Target Function](origin-flow-pallet-collective-to-treasury.md#16-pallet-level-dispatch-reaching-the-target-function)
* [Part V: Origin Validation](origin-flow-pallet-collective-to-treasury.md#part-v-origin-validation)
  * [17. All Pallet Functions Receive RuntimeOrigin, Never a Local Origin](origin-flow-pallet-collective-to-treasury.md#17-all-pallet-functions-receive-runtimeorigin-never-a-local-origin)
  * [18. EnsureOrigin: A Validator, Not an Origin](origin-flow-pallet-collective-to-treasury.md#18-ensureorigin-a-validator-not-an-origin)
  * [19. OriginTrait and PalletsOrigin: Opening the RuntimeOrigin Envelope](origin-flow-pallet-collective-to-treasury.md#19-origintrait-and-palletsorigin-opening-the-runtimeorigin-envelope)
  * [20. EitherOfDiverse: Composing Multiple Validators](origin-flow-pallet-collective-to-treasury.md#20-eitherofdiverse-composing-multiple-validators)
  * [21. EnsureRoot: Checking for Root Origin](origin-flow-pallet-collective-to-treasury.md#21-ensureroot-checking-for-root-origin)
  * [22. EnsureProportionMoreThan: Checking Collective Approval](origin-flow-pallet-collective-to-treasury.md#22-ensureproportionmorethan-checking-collective-approval)
* [Part VI: Reuse and Extension Patterns](origin-flow-pallet-collective-to-treasury.md#part-vi-reuse-and-extension-patterns)
  * [23. The Instance Parameter: Supporting Multiple Collectives](origin-flow-pallet-collective-to-treasury.md#23-the-instance-parameter-supporting-multiple-collectives)
  * [24. DefaultVote: Configurable Abstention Strategy](origin-flow-pallet-collective-to-treasury.md#24-defaultvote-configurable-abstention-strategy)
  * [25. Where to Place Custom Trait Implementations](origin-flow-pallet-collective-to-treasury.md#25-where-to-place-custom-trait-implementations)
* [Part VII: Summary](origin-flow-pallet-collective-to-treasury.md#part-vii-summary)
  * [26. Complete Flow Diagram](origin-flow-pallet-collective-to-treasury.md#26-complete-flow-diagram)
  * [27. Key Architectural Principles](origin-flow-pallet-collective-to-treasury.md#27-key-architectural-principles)

***

### Introduction

In FRAME-based runtimes, pallets are fully decoupled. They don't import each other or call each other directly. Instead, they communicate through the **origin system**: one pallet produces an origin that encodes "who authorized this action," and another pallet validates that origin to decide whether the action is permitted.

This document traces the complete lifecycle of an origin produced by `pallet-collective` (a governance collective like a Council) as it travels through the runtime dispatch system and gets validated by `pallet-treasury` (which manages a chain's on-chain funds).

But the specific pallets are secondary. The real subject is **the architectural pattern itself**: how FRAME uses Rust's type system to encapsulate pallet-local types inside runtime-global enums, with trait-based indirection at every boundary. This same pattern appears everywhere — origins, events, calls, errors — and understanding it once unlocks understanding of the entire framework.

***

## Part I: The Universal Pattern

### 1. The Core Architecture: Local Types → Global Enums

The most important concept in FRAME is this: **every pallet defines its own local types, and the runtime wraps them all into global enums.**

This happens for three core type families:

| Pallet defines | Macro attribute     | Runtime generates    | Purpose                           |
| -------------- | ------------------- | -------------------- | --------------------------------- |
| `RawOrigin`    | `#[pallet::origin]` | `RuntimeOrigin` enum | Who authorized an action          |
| `Event`        | `#[pallet::event]`  | `RuntimeEvent` enum  | What happened in a block          |
| `Call`         | `#[pallet::call]`   | `RuntimeCall` enum   | What dispatchable functions exist |

The mechanism is identical for all three:

1. The pallet defines its local type (e.g., `enum Event { Proposed, Voted, ... }`).
2. A macro attribute registers it (e.g., `#[pallet::event]`).
3. `construct_runtime!` scans all pallets and builds the global enum with one variant per pallet.
4. `From` impls are auto-generated so pallet-local types can be converted into the global enum.

For example, when `pallet-collective` emits an event:

```rust
Self::deposit_event(Event::Approved { proposal_hash });
```

Internally, `Event::Approved` is converted via `From` to `RuntimeEvent::Council(Event::Approved { ... })` and stored in the block's event log. The same `From`-based wrapping pattern that origins use.

This is not a coincidence — it's the **foundational design pattern** of the entire framework. Once you understand it for origins, you understand it for everything.

***

### 2. From as the Exit Door: How Pallet Types Escape

A pallet's `RawOrigin`, `Event`, or `Call` is just a Rust type defined inside the pallet's module. By itself, **it's an isolated type that nothing outside the pallet can understand**.

The `From` implementation generated by `construct_runtime!` is the **exit door**. It's what allows a pallet-local type to leave the pallet and enter the runtime layer:

```
Without From:
  RawOrigin::Members(4, 5)  →  stuck inside pallet-collective, invisible to treasury

With From:
  RawOrigin::Members(4, 5)  →  .into()  →  RuntimeOrigin::Council(Members(4, 5))
                                              ↑ now the entire runtime understands it
```

The `From` constraint appears in pallet Configs to guarantee this exit door exists:

```rust
type RuntimeOrigin: From<RawOrigin<Self::AccountId, I>>;
```

This says: "whatever type the runtime assigns as `RuntimeOrigin` must be able to receive and wrap my `RawOrigin`." If that `From` didn't exist, the pallet could create origins but never send them anywhere — they'd be trapped.

The reverse direction (`TryInto`) is the **entry door** — it allows validators to reach back into the `RuntimeOrigin` and extract the pallet-specific origin for inspection.

***

### 3. The Constraint Funnel: Trait → Config → Runtime

A recurring source of confusion in FRAME code is: "this trait is so generic, how does it end up working with a specific type?" The answer is the **constraint funnel** — each layer adds restrictions, narrowing the type step by step.

**Example: How `Dispatchable::RuntimeOrigin` becomes `RuntimeOrigin`**

Layer 1 — The trait is maximally generic:

> `substrate/primitives/runtime/src/traits/mod.rs`

```rust
pub trait Dispatchable {
    type RuntimeOrigin: Debug;  // any type with Debug
    fn dispatch(self, origin: Self::RuntimeOrigin) -> ...;
}
```

Layer 2 — `frame_system::Config` adds the first real constraints:

> `substrate/frame/system/src/lib.rs`

```rust
#[pallet::config(with_default, frame_system_config)]
#[pallet::disable_frame_system_supertrait_check]
pub trait Config: 'static + Eq + Clone {
    // ... other associated types ...

    /// The `RuntimeOrigin` type used by dispatchable calls.
    #[pallet::no_default_bounds]
    type RuntimeOrigin: Into<Result<RawOrigin<Self::AccountId>, Self::RuntimeOrigin>>
        + From<RawOrigin<Self::AccountId>>
        + Clone
        + OriginTrait<Call = Self::RuntimeCall, AccountId = Self::AccountId>
        + AsTransactionAuthorizedOrigin;
}
```

This is where `RuntimeOrigin` stops being "any type with Debug" and gains its first concrete shape:

* `From<RawOrigin<Self::AccountId>>` — `RuntimeOrigin` must be **constructable from** a `frame_system::RawOrigin`. Direction: `RawOrigin → RuntimeOrigin`. This is what allows `RuntimeOrigin::from(RawOrigin::Root)` or `RuntimeOrigin::from(RawOrigin::Signed(account))`.
* `Into<Result<RawOrigin<Self::AccountId>, Self::RuntimeOrigin>>` — the reverse: `RuntimeOrigin` must be **unwrappable back into** a `RawOrigin` (returning `Err(self)` if it's not a system origin). This is what `EnsureOrigin` implementations use to inspect what's inside.
* `OriginTrait` — provides the standard interface (`caller()`, `add_filter()`, etc.).

Every pallet that declares `Config: frame_system::Config` inherits these bounds automatically.

Layer 3 — The pallet Config adds its own constraint on top:

> `substrate/frame/collective/src/lib.rs`

```rust
pub trait Config<I: 'static = ()>: frame_system::Config {
    type RuntimeOrigin: From<RawOrigin<Self::AccountId, I>>;
    type Proposal: Dispatchable<RuntimeOrigin = <Self as Config<I>>::RuntimeOrigin>;
}
```

Note that this `RuntimeOrigin` is a **different associated type** than `frame_system::Config::RuntimeOrigin` — they live in different traits (`<T as frame_system::Config>::RuntimeOrigin` vs `<T as pallet_collective::Config<I>>::RuntimeOrigin`). The name is the same by convention, not by collision. In the runtime, both are assigned to the same concrete `RuntimeOrigin` enum.

The `From<RawOrigin<Self::AccountId, I>>` here refers to `pallet_collective::RawOrigin`, not `frame_system::RawOrigin` — they are different types. The collective's `RawOrigin` has variants like `Members(MemberCount, MemberCount)` representing a council vote quorum. This bound ensures `RuntimeOrigin` can be constructed from a collective vote result, just like Layer 2 ensures it can be constructed from `Root`/`Signed`/`None`.

Layer 4 — The runtime concretizes everything:

> `substrate/bin/node/runtime/src/lib.rs`

```rust
impl pallet_collective::Config<CouncilCollective> for Runtime {
    type RuntimeOrigin = RuntimeOrigin;  // the concrete enum
    type Proposal = RuntimeCall;          // the concrete call enum
}
```

Where does this `RuntimeOrigin` type come from? It is **not hand-written** — it is auto-generated by the `#[frame_support::runtime]` macro (line 2606 in the same file):

```rust
#[frame_support::runtime]
mod runtime {
    #[runtime::runtime]
    #[runtime::derive(
        RuntimeCall,
        RuntimeEvent,
        RuntimeError,
        RuntimeOrigin,   // ← generated here
        // ...
    )]
    pub struct Runtime;

    #[runtime::pallet_index(0)]
    pub type System = frame_system::Pallet<Runtime>;
    // ...
}
```

The macro reads all the pallets listed in this block and generates a `RuntimeOrigin` enum with one variant per pallet that defines a custom origin. It also generates the `From` impls for each variant — this is how a single `RuntimeOrigin` satisfies both `From<frame_system::RawOrigin>` and `From<pallet_collective::RawOrigin>` at the same time.

The funnel:

```
Dispatchable trait          →  any type with Debug
    ↓
frame_system::Config        →  must convert to/from RawOrigin, implement OriginTrait
    ↓
pallet_collective::Config   →  must also support From<collective::RawOrigin>
    ↓
Runtime assignment          →  it's RuntimeOrigin, the auto-generated enum
```

Each layer narrows without breaking the genericity of the layers above. The trait stays reusable, `frame_system` sets the foundational bounds, the pallet Config stays pallet-generic, and only the runtime pins everything to concrete types.

***

### 4. Compiler Inference: How Generic Types Resolve Without Explicit Passing

A common question when reading FRAME code: "where does this generic type get specified? I don't see it being passed anywhere."

Let's trace a real example end-to-end: how `EnsureRoot` ends up knowing that `O = RuntimeOrigin` without anyone telling it.

#### Step 1 — The trait `EnsureOrigin` defines the contract with a generic `OuterOrigin`

> `substrate/frame/support/src/traits/dispatch.rs:33`

```rust
pub trait EnsureOrigin<OuterOrigin> {
    type Success;

    fn ensure_origin(o: OuterOrigin) -> Result<Self::Success, BadOrigin> {
        Self::try_origin(o).map_err(|_| BadOrigin)
    }

    fn try_origin(o: OuterOrigin) -> Result<Self::Success, OuterOrigin>;
}
```

Anyone implementing this trait must say: "given an origin of type `OuterOrigin`, I can verify if it's valid and return a `Success` or reject it." The key point: `OuterOrigin` is a **generic parameter on the trait**, not a concrete type yet.

#### Step 2 — `EnsureRoot` is a struct that only has `AccountId`

> `substrate/frame/system/src/lib.rs:1277`

```rust
pub struct EnsureRoot<AccountId>(PhantomData<AccountId>);
```

Just `AccountId`. No parameter for the origin type. You can't pass `O` when creating this struct.

#### Step 3 — The impl adds `O` as a generic that is NOT on the struct

> `substrate/frame/system/src/lib.rs:1278`

```rust
impl<O: OriginTrait, AccountId> EnsureOrigin<O> for EnsureRoot<AccountId> {
    type Success = ();
    fn try_origin(o: O) -> Result<Self::Success, O> {
        match o.as_system_ref() {
            Some(RawOrigin::Root) => Ok(()),
            _ => Err(o),
        }
    }
}
```

This says: "I work for **any** `O` that implements `OriginTrait`." But `O` is not part of the struct — nobody passes it when writing `EnsureRoot<AccountId>`. So where does it come from?

#### Step 4 — In the runtime, `EnsureRoot` is used inside `EitherOfDiverse`, without passing `O`

> `substrate/bin/node/runtime/src/lib.rs:1282`

```rust
type EnsureRootOrHalfCouncil = EitherOfDiverse<
    EnsureRoot<AccountId>,                                                     // ← no O here
    pallet_collective::EnsureProportionMoreThan<AccountId, CouncilCollective, 1, 2>,  // ← no O here either
>;
```

Nobody wrote `EnsureRoot<AccountId, RuntimeOrigin>`. Just `EnsureRoot<AccountId>`.

#### Step 5 — `EitherOfDiverse` also has a hidden generic `OuterOrigin` in its impl

> `substrate/frame/support/src/traits/dispatch.rs:358-371`

```rust
pub struct EitherOfDiverse<L, R>(PhantomData<(L, R)>);

impl<OuterOrigin, L: EnsureOrigin<OuterOrigin>, R: EnsureOrigin<OuterOrigin>>
    EnsureOrigin<OuterOrigin> for EitherOfDiverse<L, R>
{
    type Success = Either<L::Success, R::Success>;
    fn try_origin(o: OuterOrigin) -> Result<Self::Success, OuterOrigin> {
        L::try_origin(o)
            .map_or_else(|o| R::try_origin(o).map(Either::Right), |o| Ok(Either::Left(o)))
    }
}
```

`OuterOrigin` is also not on the struct. But this impl says: for me to work, both `L` and `R` must implement `EnsureOrigin<OuterOrigin>`. Whatever `OuterOrigin` turns out to be, it propagates inward to `L` and `R`.

#### Step 6 — The constraint in `pallet_treasury::Config` is what gives everything a value

> `substrate/frame/treasury/src/lib.rs:222`

```rust
/// Origin from which rejections must come.
type RejectOrigin: EnsureOrigin<Self::RuntimeOrigin>;
```

And the runtime assigns:

> `substrate/bin/node/runtime/src/lib.rs:1316`

```rust
impl pallet_treasury::Config for Runtime {
    type RejectOrigin = EnsureRootOrHalfCouncil;
}
```

#### Step 7 — The compiler resolves from outside in

Here's the chain of inference:

1. **`RejectOrigin`** must implement `EnsureOrigin<Self::RuntimeOrigin>`. Since `Self` is `Runtime`, this means `EnsureOrigin<RuntimeOrigin>`.
2. **`EnsureRootOrHalfCouncil`** is `EitherOfDiverse<EnsureRoot<AccountId>, EnsureProportionMoreThan<...>>`. The compiler goes to the impl of `EitherOfDiverse` and matches: `OuterOrigin = RuntimeOrigin`.
3. But the impl requires `L: EnsureOrigin<OuterOrigin>` and `R: EnsureOrigin<OuterOrigin>`. So now the compiler needs:
   * `EnsureRoot<AccountId>: EnsureOrigin<RuntimeOrigin>`
   * `EnsureProportionMoreThan<AccountId, CouncilCollective, 1, 2>: EnsureOrigin<RuntimeOrigin>`
4. **For `EnsureRoot`**: the compiler goes to its impl, matches **`O = RuntimeOrigin`**, and checks: does `RuntimeOrigin: OriginTrait`? Yes — the `#[frame_support::runtime]` macro generated that impl. Compiles.
5. **For `EnsureProportionMoreThan`**: same process, **`O = RuntimeOrigin`**. Does `RuntimeOrigin: OriginTrait + From<RawOrigin<AccountId, CouncilCollective>>`? Yes. Compiles.

```
pallet_treasury::Config
    "RejectOrigin must be EnsureOrigin<RuntimeOrigin>"
        ↓
EitherOfDiverse impl
    OuterOrigin = RuntimeOrigin
    "L and R must be EnsureOrigin<RuntimeOrigin>"
        ↓                       ↓
EnsureRoot impl          EnsureProportionMoreThan impl
    O = RuntimeOrigin        O = RuntimeOrigin
    ✓ OriginTrait            ✓ OriginTrait + From<RawOrigin>
```

Nobody wrote `EnsureRoot<AccountId, RuntimeOrigin>`. The `RuntimeOrigin` propagated on its own, from the Config constraint, through `EitherOfDiverse`, all the way down to `EnsureRoot` and `EnsureProportionMoreThan`.

**Rule of thumb**: if a generic type appears on the `impl` but not on the `struct`, the compiler infers it from the context where the struct is used. You won't find it passed anywhere in the code — it's resolved at compile time through trait bounds.

***

## Part II: Origin Definition and Registration

### 5. Defining the Custom Origin in pallet-collective

**File**: `substrate/frame/collective/src/lib.rs`

The pallet defines its own origin type as a Rust enum:

```rust
pub enum RawOrigin<AccountId, I> {
    /// Approved by `MemberCount` out of `MemberCount` total members.
    Members(MemberCount, MemberCount),
    /// Dispatched by a single member of the collective.
    Member(AccountId),
    /// Phantom for the instance type.
    _Phantom(PhantomData<I>),
}
```

Two meaningful variants:

* **`Members(n, m)`** — "This action was approved by `n` members out of `m` total." This is the collective's group signature. It carries the approval ratio so that downstream validators can enforce proportion-based rules (e.g., "more than half must have approved").
* **`Member(AccountId)`** — "This action was dispatched by a single member." Used when a proposal has `threshold < 2` and executes immediately without voting.

The `I` type parameter enables **instancing**: the same pallet can be used multiple times in a runtime (e.g., Council as `Instance1`, TechnicalCommittee as `Instance2`). Each instance gets its own storage, events, and origin types. `Instance1`, `Instance2`, etc. are empty marker structs that exist solely to make Rust treat each instance as a different type.

At this point, `RawOrigin` is just a type definition. It doesn't do anything yet — it's a data structure that will carry information about collective decisions.

***

### 6. Registering the Origin with #\[pallet::origin]

**File**: `substrate/frame/collective/src/lib.rs`

```rust
#[pallet::origin]
pub type Origin<T, I = ()> = RawOrigin<<T as frame_system::Config>::AccountId, I>;
```

This macro attribute does one thing: it **registers** this type as the pallet's custom origin. It tells the FRAME macro system: "this pallet produces its own origins, and here's the type."

By itself, it generates no code. It's a marker — an opt-in signal. When `construct_runtime!` later scans all pallets, it checks which ones have `#[pallet::origin]` and includes them in the global `RuntimeOrigin` enum. Pallets without `#[pallet::origin]` (like `pallet-treasury` or `pallet-balances`) don't get a variant — they use the standard system origins (`Signed`, `Root`, `None`) and nothing more.

The next section shows exactly what the macro generates from this marker.

***

### 7. construct\_runtime! Generates the RuntimeOrigin Enum

**File**: Runtime crate (e.g., `runtime/src/lib.rs`)

When the runtime developer declares their pallets:

```rust
construct_runtime!(
    pub enum Runtime {
        System: frame_system,                               // ← has #[pallet::origin]
        Council: pallet_collective::<Instance1>,            // ← has #[pallet::origin]
        TechnicalCommittee: pallet_collective::<Instance2>, // ← has #[pallet::origin]
        Treasury: pallet_treasury,                          // ← NO #[pallet::origin]
        Balances: pallet_balances,                          // ← NO #[pallet::origin]
        // ...
    }
);
```

The macro scans each pallet. It finds that `frame_system` and `pallet_collective` (both instances) have `#[pallet::origin]`. It then auto-generates the global origin enum — **only pallets with `#[pallet::origin]` get a variant**:

```rust
// Auto-generated by construct_runtime!
pub enum RuntimeOrigin {
    System(frame_system::RawOrigin<AccountId>),                             // ✓ has #[pallet::origin]
    Council(pallet_collective::RawOrigin<AccountId, Instance1>),            // ✓ has #[pallet::origin]
    TechnicalCommittee(pallet_collective::RawOrigin<AccountId, Instance2>), // ✓ has #[pallet::origin]
    // Treasury  → NOT here, no #[pallet::origin]
    // Balances  → NOT here, no #[pallet::origin]
}
```

The macro generates the same kind of enum for calls and events:

```rust
// Also auto-generated
pub enum RuntimeCall {
    System(frame_system::Call<Runtime>),
    Council(pallet_collective::Call<Runtime, Instance1>),
    Treasury(pallet_treasury::Call<Runtime>),
    // ... one variant per pallet
}

pub enum RuntimeEvent {
    System(frame_system::Event<Runtime>),
    Council(pallet_collective::Event<Runtime, Instance1>),
    Treasury(pallet_treasury::Event<Runtime>),
    // ... one variant per pallet
}
```

All three follow the exact same pattern: pallet-local type wrapped in a variant of the global enum.

***

### 8. Automatic From Generation for Each Variant

The macro also generates `From` implementations for each variant, enabling conversion from pallet-local types to the global enum. For origins:

```rust
// Auto-generated for frame_system
impl From<frame_system::RawOrigin<AccountId>> for RuntimeOrigin {
    fn from(o: frame_system::RawOrigin<AccountId>) -> Self {
        RuntimeOrigin::System(o)
    }
}

// Auto-generated for Council (Instance1)
impl From<pallet_collective::RawOrigin<AccountId, Instance1>> for RuntimeOrigin {
    fn from(o: pallet_collective::RawOrigin<AccountId, Instance1>) -> Self {
        RuntimeOrigin::Council(o)
    }
}

// Auto-generated for TechnicalCommittee (Instance2)
impl From<pallet_collective::RawOrigin<AccountId, Instance2>> for RuntimeOrigin {
    fn from(o: pallet_collective::RawOrigin<AccountId, Instance2>) -> Self {
        RuntimeOrigin::TechnicalCommittee(o)
    }
}
```

These `From` impls are the **exit door** (see [Section 2](origin-flow-pallet-collective-to-treasury.md#2-from-as-the-exit-door-how-pallet-types-escape)). Without them, a pallet's `RawOrigin` would be trapped inside the pallet.

The corresponding `TryInto` impls are also generated, enabling the reverse: extracting a pallet-specific origin from the global `RuntimeOrigin`. This is the **entry door** used by validators like `EnsureProportionMoreThan` to inspect origins.

The same `From`/`TryInto` generation happens for `RuntimeEvent` and `RuntimeCall` — same pattern, same macro, same purpose.

***

## Part III: Wiring Origins and Calls in Config

### 9. Wiring RuntimeOrigin in the Pallet Config

**File**: Runtime crate

In the pallet's `Config` trait, the `RuntimeOrigin` associated type establishes the connection between pallet and runtime:

```rust
pub trait Config<I: 'static = ()>: frame_system::Config {
    /// Must be convertible FROM the pallet's RawOrigin.
    type RuntimeOrigin: From<RawOrigin<Self::AccountId, I>>;
    // ...
}
```

The `From<RawOrigin<...>>` constraint guarantees the exit door exists: whatever type the runtime assigns must accept the pallet's `RawOrigin` via `.into()`.

The runtime satisfies this:

```rust
impl pallet_collective::Config<CouncilCollective> for Runtime {
    type RuntimeOrigin = RuntimeOrigin;  // the auto-generated enum
    // ...
}
```

The compiler checks: "Does `RuntimeOrigin` implement `From<pallet_collective::RawOrigin<AccountId, Instance1>>`?" Yes — the macro generated it. Compiles.

Note that **`Self::RuntimeOrigin` ultimately comes from `frame_system::Config`**, since every pallet inherits from it:

```rust
pub trait Config<I: 'static = ()>: frame_system::Config {
//                                  ^^^^^^^^^^^^^^^^^^^^
```

And `frame_system::Config` defines:

```rust
type RuntimeOrigin: Into<Result<RawOrigin<Self::AccountId>, Self::RuntimeOrigin>>
    + From<RawOrigin<Self::AccountId>>
    + Clone
    + OriginTrait<Call = Self::RuntimeCall, AccountId = Self::AccountId>;
```

So the `RuntimeOrigin` type is shared across **all pallets** in the runtime — they all inherit it from `frame_system`. This is how the treasury and the collective end up working with the same origin type without knowing about each other.

***

### 10. The Dispatchable Trait: How dispatch Knows Its Origin Type

The `Dispatchable` trait is defined in `sp_runtime`:

```rust
pub trait Dispatchable {
    type RuntimeOrigin: Debug;
    type Config;
    type Info;
    type PostInfo;
    fn dispatch(self, origin: Self::RuntimeOrigin) -> DispatchResultWithInfo<Self::PostInfo>;
}
```

At the trait level, `RuntimeOrigin` has a single constraint: `Debug`. It could be anything.

The pallet's Config narrows it. In `pallet_collective`:

> `substrate/frame/collective/src/lib.rs:333`

```rust
pub trait Config<I: 'static = ()>: frame_system::Config {
    type RuntimeOrigin: From<RawOrigin<Self::AccountId, I>>;

    /// The runtime call dispatch type.
    type Proposal: Parameter
        + Dispatchable<
            RuntimeOrigin = <Self as Config<I>>::RuntimeOrigin,
            //              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
            //  "the origin type that Proposal uses to dispatch
            //   must be the SAME as this pallet's RuntimeOrigin"
            PostInfo = PostDispatchInfo,
        > + From<frame_system::Call<Self>>
        + GetDispatchInfo;
}
```

This connects the dispatch system to the origin system: when the collective calls `proposal.dispatch(origin)`, the compiler guarantees that `origin` is the right type because both `Proposal` and the pallet agree on the same `RuntimeOrigin`.

The runtime concretizes both:

> `substrate/bin/node/runtime/src/lib.rs:1193`

```rust
impl pallet_collective::Config<CouncilCollective> for Runtime {
    type RuntimeOrigin = RuntimeOrigin;  // the auto-generated enum
    type Proposal = RuntimeCall;          // the auto-generated call enum
}
```

`RuntimeCall` implements `Dispatchable` with `type RuntimeOrigin = RuntimeOrigin` (auto-generated by `construct_runtime!`). Now when the pallet calls `proposal.dispatch(origin)`, the compiler knows:

* `proposal` is a `RuntimeCall`
* `dispatch` expects a `RuntimeOrigin`
* `origin` (from `.into()`) is a `RuntimeOrigin`

Everything type-checks. The entire chain — from trait definition to dispatch call — is verified at compile time with zero runtime overhead.

***

### 11. The Proposal: A Runtime Call Stored On-Chain

When a member proposes a motion, the dispatchable call is stored in the pallet's storage:

**File**: `substrate/frame/collective/src/lib.rs`

```rust
// In do_propose_proposed:
<ProposalOf<T, I>>::insert(proposal_hash, proposal);
```

The `proposal` is a `RuntimeCall` — for example, `RuntimeCall::Treasury(pallet_treasury::Call::reject_proposal { proposal_id: 42 })`. It's SCALE-encoded and stored under its hash.

The proposal sits in storage for the duration of the voting period. It contains no origin information — the origin is constructed only at execution time, based on the voting results. The call (what to do) and the origin (who authorized it) are **separate things that only come together at dispatch time**.

When it's time to execute, the proposal is read back:

```rust
// In do_close, before calling do_approve_proposal:
let (proposal, len) = Self::validate_and_get_proposal(
    &proposal_hash,
    length_bound,
    proposal_weight_bound,
)?;
```

***

## Part IV: The Dispatch Journey

### 12. Voting Ends: Constructing the Origin

**File**: `substrate/frame/collective/src/lib.rs`

When `do_close` determines that a proposal has been approved (enough votes), it calls `do_approve_proposal`:

```rust
fn do_approve_proposal(
    seats: MemberCount,
    yes_votes: MemberCount,
    proposal_hash: T::Hash,
    proposal: <T as Config<I>>::Proposal,
) -> (Weight, u32) {
    Self::deposit_event(Event::Approved { proposal_hash });

    let dispatch_weight = proposal.get_dispatch_info().call_weight;
    let origin = RawOrigin::Members(yes_votes, seats).into();
    let result = proposal.dispatch(origin);
    // ...
}
```

This is where the origin is born. `RawOrigin::Members(yes_votes, seats)` captures the voting result: how many members approved and the total number of seats.

***

### 13. The .into() Call: From Local Origin to Global Origin

```rust
let origin = RawOrigin::Members(yes_votes, seats).into();
```

The `.into()` call uses the `From` implementation generated in Section 8. If this is the Council (Instance1), it performs:

```
RawOrigin::Members(4, 5)
    → RuntimeOrigin::Council(RawOrigin::Members(4, 5))
```

Before the `.into()`, the origin is a pallet-local type that only `pallet-collective` understands. After the `.into()`, it's a `RuntimeOrigin` that the entire runtime can route and inspect.

This is the moment the origin **exits the pallet** and enters the runtime layer. The `From` impl is the exit door that makes this possible.

***

### 14. proposal.dispatch(origin): Entering the Runtime Dispatch

```rust
let result = proposal.dispatch(origin);
```

`proposal` is a `RuntimeCall` (e.g., `RuntimeCall::Treasury(Call::reject_proposal { proposal_id: 42 })`).

As explained in Section 10, `dispatch` expects `Self::RuntimeOrigin`, which has been resolved to `RuntimeOrigin`. So it receives the `RuntimeOrigin::Council(Members(4, 5))` we just constructed.

At this point, **two separate things come together**: the call (what to do) and the origin (who authorized it). The call was stored in storage for days. The origin was just constructed from the voting results. They were separate until this exact line.

***

### 15. Runtime Dispatch: The Call Router

The `Dispatchable` impl for `RuntimeCall` is auto-generated by `construct_runtime!`:

```rust
// Auto-generated
impl Dispatchable for RuntimeCall {
    type RuntimeOrigin = RuntimeOrigin;

    fn dispatch(self, origin: RuntimeOrigin) -> DispatchResultWithPostInfo {
        match self {
            RuntimeCall::System(call) => call.dispatch(origin),
            RuntimeCall::Balances(call) => call.dispatch(origin),
            RuntimeCall::Treasury(call) => call.dispatch(origin),
            RuntimeCall::Council(call) => call.dispatch(origin),
            // ... one arm per pallet
        }
    }
}
```

The runtime dispatch is a **pure router**. It matches on the `RuntimeCall` variant to determine which pallet should handle the call, then forwards both the call and the origin to that pallet. It does not inspect, modify, or validate the origin — that's the destination pallet's job.

In our example, `RuntimeCall::Treasury(call)` matches, and `call` is the pallet-level `pallet_treasury::Call::reject_proposal { proposal_id: 42 }`. The origin is forwarded unchanged: `RuntimeOrigin::Council(Members(4, 5))`.

***

### 16. Pallet-Level Dispatch: Reaching the Target Function

The `#[pallet::call]` macro generates a `Dispatchable` impl for each pallet's `Call` enum:

```rust
// Auto-generated by #[pallet::call] for pallet_treasury
impl Dispatchable for pallet_treasury::Call<T> {
    fn dispatch(self, origin: RuntimeOrigin) -> DispatchResultWithPostInfo {
        match self {
            Call::reject_proposal { proposal_id } => {
                Pallet::<T>::reject_proposal(origin, proposal_id)
            },
            Call::approve_proposal { proposal_id } => {
                Pallet::<T>::approve_proposal(origin, proposal_id)
            },
            // ...
        }
    }
}
```

Two levels of dispatch have occurred:

```
RuntimeCall::Treasury(Call::reject_proposal { proposal_id: 42 })
    → RuntimeCall dispatch: matches Treasury, forwards to pallet
    → pallet_treasury::Call dispatch: matches reject_proposal, calls the function
    → Pallet::<T>::reject_proposal(origin, 42)
```

***

## Part V: Origin Validation

### 17. All Pallet Functions Receive RuntimeOrigin, Never a Local Origin

Every dispatchable function in every pallet receives the origin as `OriginFor<T>`:

```rust
pub fn reject_proposal(origin: OriginFor<T>, proposal_id: u32) -> ... {
```

`OriginFor<T>` is a type alias defined in `frame_system`:

```rust
pub type OriginFor<T> = <T as frame_system::Config>::RuntimeOrigin;
```

Which in the runtime resolves to the global `RuntimeOrigin` enum. **No pallet function ever receives a pallet-local origin directly.** The origin always arrives as the global `RuntimeOrigin`, and it's the function's responsibility to validate it using an `EnsureOrigin` validator.

This is a hard rule: the dispatch system always passes the runtime-level origin. Pallet-local origins only exist during creation (inside the producing pallet) and during validation (inside an `EnsureOrigin` impl). In between, they travel wrapped inside `RuntimeOrigin`.

***

### 18. EnsureOrigin: A Validator, Not an Origin

A critical distinction: **`EnsureOrigin` is not an origin. It's a validator — a filter — that inspects an origin and says yes or no.**

In the treasury's Config:

> `substrate/frame/treasury/src/lib.rs:222`

```rust
pub trait Config: frame_system::Config {
    /// Origin from which rejections must come.
    type RejectOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    // ...
}
```

`RejectOrigin` is not "the origin needed to reject proposals." It's "the rule that decides whether a given origin is allowed to reject proposals." It receives an origin and returns `Ok` or `Err`.

The `EnsureOrigin` trait:

```rust
pub trait EnsureOrigin<OuterOrigin> {
    type Success;

    /// The core method. Try to validate the origin.
    fn try_origin(o: OuterOrigin) -> Result<Self::Success, OuterOrigin>;

    /// Convenience wrapper: converts Err to BadOrigin.
    fn ensure_origin(o: OuterOrigin) -> Result<Self::Success, BadOrigin> {
        Self::try_origin(o).map_err(|_| BadOrigin)
    }
}
```

* **`try_origin`**: Receives an origin. Returns `Ok(Success)` if valid, or `Err(origin)` if not. The key detail: on failure, it **returns the origin back** so another validator can try it (used by `EitherOfDiverse`).
* **`ensure_origin`**: Same thing but discards the origin on failure, returning just `BadOrigin`.
* **`type Success`**: What information is extracted on success. Can be `()` (nothing, just passed), `AccountId` (who signed), `(MemberCount, MemberCount)` (approval ratio), etc.

The treasury uses it as a guard at the top of its functions:

```rust
pub fn reject_proposal(origin: OriginFor<T>, proposal_id: u32) -> ... {
    T::RejectOrigin::ensure_origin(origin)?;
    // Only reaches here if the origin passed validation
}
```

***

### 19. OriginTrait and PalletsOrigin: Opening the RuntimeOrigin Envelope

To validate an origin, validators need to look **inside** the `RuntimeOrigin` enum. Two key pieces enable this:

**`OriginTrait`** — A trait implemented by `RuntimeOrigin` (auto-generated by `construct_runtime!`). It provides methods to inspect the origin:

```rust
pub trait OriginTrait {
    type PalletsOrigin;
    type AccountId;

    /// Try to extract the frame_system::RawOrigin (Root, Signed, None)
    fn as_system_ref(&self) -> Option<&RawOrigin<Self::AccountId>>;

    /// Get the inner PalletsOrigin
    fn caller(&self) -> &Self::PalletsOrigin;

    /// Check if this is Root
    fn is_root(&self) -> bool;
    // ...
}
```

**`PalletsOrigin`** (also called `OriginCaller`) — The inner enum that `RuntimeOrigin` wraps. It's the actual data:

```rust
// RuntimeOrigin wraps this:
pub enum OriginCaller {
    System(frame_system::RawOrigin<AccountId>),
    Council(pallet_collective::RawOrigin<AccountId, Instance1>),
    TechnicalCommittee(pallet_collective::RawOrigin<AccountId, Instance2>),
}
```

The relationship:

```
RuntimeOrigin
  └── contains a PalletsOrigin (OriginCaller)
        └── which is an enum with one variant per pallet that defines origins
```

Different validators use different methods:

* **`EnsureRoot`** uses `o.as_system_ref()` — a shortcut that directly checks if the origin is `System(Root)`.
* **`EnsureProportionMoreThan`** uses `o.caller().try_into()` — gets the `PalletsOrigin`, then attempts to convert it to a specific pallet's `RawOrigin`.

***

### 20. EitherOfDiverse: Composing Multiple Validators

**File**: Runtime crate

The runtime assigns a composed validator to the treasury:

```rust
impl pallet_treasury::Config for Runtime {
    type RejectOrigin = EitherOfDiverse<
        EnsureRoot<AccountId>,
        pallet_collective::EnsureProportionMoreThan<AccountId, CouncilCollective, 1, 2>,
    >;
    // ...
}
```

`EitherOfDiverse` is defined in `frame_support`:

```rust
pub struct EitherOfDiverse<L, R>(PhantomData<(L, R)>);

impl<OuterOrigin, L: EnsureOrigin<OuterOrigin>, R: EnsureOrigin<OuterOrigin>>
    EnsureOrigin<OuterOrigin> for EitherOfDiverse<L, R>
{
    fn try_origin(o: OuterOrigin) -> Result<Self::Success, OuterOrigin> {
        L::try_origin(o).or_else(|o| R::try_origin(o))
    }
}
```

It's a logical **OR**: try the left validator first. If it fails (returns `Err` with the origin), try the right one. If either passes, the whole thing passes.

In our case: "Either Root **or** more than half the Council."

Note the `OuterOrigin` parameter on the `impl`: it's not specified explicitly anywhere. As explained in Section 4, the compiler infers it as `RuntimeOrigin` from the constraint `type RejectOrigin: EnsureOrigin<Self::RuntimeOrigin>`.

Also note why `try_origin` returns `Err(origin)` instead of `Err(BadOrigin)`: **so the origin can be passed to the next validator**. If `EnsureRoot` consumed the origin on failure, `EitherOfDiverse` couldn't forward it to `EnsureProportionMoreThan`.

***

### 21. EnsureRoot: Checking for Root Origin

**File**: `substrate/frame/system/src/lib.rs`

```rust
pub struct EnsureRoot<AccountId>(PhantomData<AccountId>);

impl<O: OriginTrait, AccountId> EnsureOrigin<O> for EnsureRoot<AccountId> {
    type Success = ();
    fn try_origin(o: O) -> Result<Self::Success, O> {
        match o.as_system_ref() {
            Some(RawOrigin::Root) => Ok(()),
            _ => Err(o),
        }
    }
}
```

Uses `as_system_ref()` (from `OriginTrait`, see Section 19) to extract the `frame_system::RawOrigin` from inside the `RuntimeOrigin`. If it's `Root`, return `Ok(())`. Otherwise return `Err(o)`.

The bound `O: OriginTrait` is the only constraint. `EnsureRoot` doesn't need `From` or `TryInto` — it uses the `OriginTrait` method, which is the simplest way to inspect system-level origins.

In our flow, the origin is `RuntimeOrigin::Council(Members(4, 5))`, which is not Root. So `EnsureRoot` returns `Err`, and `EitherOfDiverse` proceeds to the second validator.

***

### 22. EnsureProportionMoreThan: Checking Collective Approval

**File**: `substrate/frame/collective/src/lib.rs`

```rust
pub struct EnsureProportionMoreThan<AccountId, I: 'static, const N: u32, const D: u32>(
    PhantomData<(AccountId, I)>,
);

impl<O: OriginTrait + From<RawOrigin<AccountId, I>>, AccountId, I, const N: u32, const D: u32>
    EnsureOrigin<O> for EnsureProportionMoreThan<AccountId, I, N, D>
where
    for<'a> &'a O::PalletsOrigin: TryInto<&'a RawOrigin<AccountId, I>>,
{
    type Success = ();
    fn try_origin(o: O) -> Result<Self::Success, O> {
        match o.caller().try_into() {
            Ok(RawOrigin::Members(n, m)) if n * D > N * m => return Ok(()),
            _ => (),
        }
        Err(o)
    }
}
```

**Struct parameters:**

| Parameter   | Purpose                    | Example Value         |
| ----------- | -------------------------- | --------------------- |
| `AccountId` | Account type, generic      | `AccountId`           |
| `I`         | Collective instance marker | `Instance1` (Council) |
| `const N`   | Proportion numerator       | `1`                   |
| `const D`   | Proportion denominator     | `2`                   |

`N` and `D` are **const generics** — values, not types, fixed at compile time. They represent the minimum proportion required: `N/D`. With `N=1, D=2`, the rule is "more than 1/2".

**The `where` clause:**

```rust
where
    for<'a> &'a O::PalletsOrigin: TryInto<&'a RawOrigin<AccountId, I>>,
```

This says: "I must be able to take a reference to the `PalletsOrigin` (the inner enum of `RuntimeOrigin`, see Section 19) and try to convert it into a reference to this collective's `RawOrigin`."

This conversion only succeeds if the origin came from the correct collective instance. A Council origin (`Instance1`) converts to `RawOrigin<_, Instance1>` but fails for `RawOrigin<_, Instance2>`.

The `for<'a>` is Rust's higher-ranked trait bound syntax: "this must work for any lifetime." It's needed because `o.caller()` returns a reference, and the `TryInto` must work regardless of the reference's lifetime.

**The validation logic:**

1. **`o.caller()`** — calls the `OriginTrait::caller()` method to extract the `PalletsOrigin` from the `RuntimeOrigin`.
2. **`.try_into()`** — attempts to convert the `PalletsOrigin` reference into a `RawOrigin<AccountId, I>` reference. This is the **entry door** back into the pallet-local type. If the origin came from a different collective instance, this fails.
3. **`n * D > N * m`** — cross-multiplication to avoid floating point division. With `N=1, D=2`: checks `n * 2 > 1 * m`, i.e., "more than half."

With our example origin `Members(4, 5)` and `N=1, D=2`:

```
4 * 2 > 1 * 5
8 > 5  ✓  passes
```

The origin is valid. `ensure_origin` returns `Ok(())`, and the treasury function proceeds to execute the proposal rejection.

Note that `EnsureProportionMoreThan` is defined **inside `pallet-collective`**, the same pallet that produces the origins. The pallet provides both the "exit door" (the `RawOrigin` and its `From` impl) and the "entry door" (the `EnsureOrigin` validators that know how to read those origins). Other pallets like treasury use these validators without knowing anything about the collective's internals.

***

## Part VI: Reuse and Extension Patterns

### 23. The Instance Parameter: Supporting Multiple Collectives

The `I` parameter in `RawOrigin<AccountId, I>` is what prevents a TechnicalCommittee origin from being accepted where a Council origin is required.

Even though both collectives use the same pallet code, their origins are **different Rust types**:

```rust
pallet_collective::RawOrigin<AccountId, Instance1>  // Council
pallet_collective::RawOrigin<AccountId, Instance2>  // TechnicalCommittee
```

The runtime declares the instances:

```rust
type CouncilCollective = pallet_collective::Instance1;
type TechnicalCollective = pallet_collective::Instance2;

#[runtime::pallet_index(14)]
pub type Council = pallet_collective::Pallet<Runtime, Instance1>;

#[runtime::pallet_index(15)]
pub type TechnicalCommittee = pallet_collective::Pallet<Runtime, Instance2>;
```

`Instance1` and `Instance2` are empty structs — pure type-level markers with no data:

```rust
pub struct Instance1;
pub struct Instance2;
```

But because they're different types, everything downstream becomes distinct:

* **Storage**: `Members<T, Instance1>` and `Members<T, Instance2>` have different storage key prefixes. Council members and TechnicalCommittee members never mix.
* **Origins**: `RawOrigin<AccountId, Instance1>` and `RawOrigin<AccountId, Instance2>` are different types. The `TryInto` generated for one doesn't match the other.
* **Validators**: `EnsureProportionMoreThan<AccountId, Instance1, 1, 2>` only accepts Council origins. TechnicalCommittee origins fail the `try_into()`.

Without the instance parameter, you'd need to write separate pallets for Council and TechnicalCommittee — identical code duplicated. With `I`, you write the pallet once and instantiate it as many times as needed.

***

### 24. DefaultVote: Configurable Abstention Strategy

**File**: `substrate/frame/collective/src/lib.rs`

When the voting period ends and `do_close` is called, members who didn't vote are **abstentions**. The `DefaultVote` trait determines how these abstentions are counted:

```rust
pub trait DefaultVote {
    fn default_vote(
        prime_vote: Option<bool>,
        yes_votes: MemberCount,
        no_votes: MemberCount,
        len: MemberCount,
    ) -> bool;
}
```

Two built-in strategies:

**`PrimeDefaultVote`** — Abstentions follow the prime member's vote:

```rust
fn default_vote(prime_vote: Option<bool>, ..) -> bool {
    prime_vote.unwrap_or(false)
}
```

**`MoreThanMajorityThenPrimeDefaultVote`** — If ayes already form a majority, abstentions count as aye. Otherwise, fall back to the prime:

```rust
fn default_vote(prime_vote: Option<bool>, yes_votes, _, len) -> bool {
    let more_than_majority = yes_votes * 2 > len;
    more_than_majority || prime_vote.unwrap_or(false)
}
```

The strategy is selected in the runtime Config:

```rust
impl pallet_collective::Config<CouncilCollective> for Runtime {
    type DefaultVote = pallet_collective::PrimeDefaultVote;
    // ...
}
```

This is defined as a trait (not a hardcoded function) **specifically so that runtime developers can implement their own custom strategy**:

```rust
pub struct MySuperMajorityVote;
impl DefaultVote for MySuperMajorityVote {
    fn default_vote(_: Option<bool>, yes_votes: MemberCount, _: MemberCount, len: MemberCount) -> bool {
        yes_votes * 3 > len * 2  // 2/3 supermajority
    }
}
```

The same "define trait in pallet, implement outside" pattern used by `EnsureOrigin`, `Currency`, and every other pluggable component in FRAME.

**Important**: `DefaultVote` is only evaluated in `do_close` when the voting period has expired. If a proposal reaches the threshold before the deadline, these strategies are never called — abstentions are irrelevant.

***

### 25. Where to Place Custom Trait Implementations

When you implement a custom `DefaultVote`, `EnsureOrigin`, or any other trait from a pallet, the placement depends on scope:

**Simple, single-runtime use (90% of cases)**: Place it directly in the runtime configuration file, near the `impl Config`:

```
runtime/
  src/
    lib.rs          // or configs/collective.rs in modular runtimes
    // custom struct + impl goes right before the impl Config block
```

This is what you see in real-world runtimes like Polkadot and Kusama — custom types defined right alongside the Config they're used in.

**Shared across multiple runtimes** (e.g., relay chain and parachains sharing the same governance rules): Extract to a dedicated crate:

```
primitives/
  governance/
    src/lib.rs      // custom DefaultVote, custom EnsureOrigin, etc.
```

Import from each runtime that needs it.

**General rule**: Don't overengineer. If it's a 10-line struct used by one runtime, put it next to the `impl Config`. Extract to a crate only when actual reuse demands it.

***

## Part VII: Summary

### 26. Complete Flow Diagram

```
PALLET-COLLECTIVE (Voting completes)
  │
  └── do_approve_proposal(seats=5, yes_votes=4, proposal_hash, proposal)
        │
        ├── 1. Create local origin
        │     RawOrigin::Members(4, 5)
        │     [pallet-local type, invisible to the rest of the runtime]
        │
        ├── 2. Convert to runtime origin via .into() [the "exit door"]
        │     → RuntimeOrigin::Council(RawOrigin::Members(4, 5))
        │     [now wrapped in the global enum, visible to all pallets]
        │
        └── 3. proposal.dispatch(origin)
              │
              │   proposal = RuntimeCall::Treasury(Call::reject_proposal { id: 42 })
              │   origin   = RuntimeOrigin::Council(Members(4, 5))
              │   [call + origin come together for the first time]
              │
RUNTIME DISPATCH (Auto-generated by construct_runtime!)
              │
              └── match RuntimeCall::Treasury(call) => call.dispatch(origin)
                    │
                    │   [Pure router: forwards call + origin unchanged to pallet]
                    │
PALLET-TREASURY DISPATCH (Auto-generated by #[pallet::call])
                    │
                    └── match Call::reject_proposal { id } =>
                          Pallet::reject_proposal(origin, 42)
                            │
                            │   [origin is OriginFor<T> = RuntimeOrigin, always global]
                            │
                            ├── 4. Validate origin
                            │     T::RejectOrigin::ensure_origin(origin)?
                            │     [RejectOrigin is a VALIDATOR, not an origin]
                            │
                            │   RejectOrigin = EitherOfDiverse<
                            │       EnsureRoot,
                            │       EnsureProportionMoreThan<_, CouncilCollective, 1, 2>
                            │   >
                            │   [Logical OR: try first, if fails try second]
                            │
ENSUREROOT (frame_system)
                            │
                            ├── 5a. Try EnsureRoot first
                            │     o.as_system_ref()
                            │       → looks for System(Root) inside RuntimeOrigin
                            │       → not Root → Err(origin)
                            │     [returns origin back so next validator can try]
                            │
ENSUREPROPORTIONMORETHAN (pallet-collective)
                            │
                            ├── 5b. Try EnsureProportionMoreThan
                            │     o.caller()
                            │       → extracts PalletsOrigin from RuntimeOrigin
                            │     .try_into()
                            │       → converts PalletsOrigin to RawOrigin<_, Instance1>
                            │       → Ok(RawOrigin::Members(4, 5)) [the "entry door"]
                            │     Check: 4 * 2 > 1 * 5 → 8 > 5 → ✓
                            │     → Ok(())
                            │
PALLET-TREASURY (Execution)
                            │
                            └── 6. Origin valid → execute reject_proposal logic
```

***

### 27. Key Architectural Principles

| Principle                                     | How It's Achieved                                                                                                                                          |
| --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Pallets are decoupled**                     | No pallet imports another. They communicate through traits and the type system.                                                                            |
| **The runtime is the compositor**             | `construct_runtime!` generates the glue: `RuntimeOrigin`, `RuntimeCall`, `RuntimeEvent`, `From` impls, `Dispatchable` impls.                               |
| **One pattern rules all**                     | Origins, events, and calls all use the same mechanism: pallet-local type → `From` → global enum variant.                                                   |
| **`From` is the exit door**                   | Without it, pallet types are trapped. `From` lets them escape to the runtime level. `TryInto` lets validators reach back in.                               |
| **The constraint funnel**                     | Traits are maximally generic. Configs add bounds. Runtimes concretize. Each layer narrows without breaking the layers above.                               |
| **Compiler inference eliminates boilerplate** | Generic type parameters on `impl` blocks are inferred from context, not passed explicitly.                                                                 |
| **Origins are data, not permissions**         | `RawOrigin::Members(4, 5)` is just a fact: "4 of 5 approved." Whether that's sufficient is decided by the validator.                                       |
| **Validators are pluggable**                  | `EnsureOrigin` is a trait. The runtime chooses which validators to use for each action. Custom validators can be implemented outside any pallet.           |
| **Everything resolves at compile time**       | Generic type parameters, trait bounds, and `From`/`TryInto` are all resolved by the Rust compiler. Zero runtime overhead. No vtables, no dynamic dispatch. |
| **Instances enable reuse**                    | The `I` parameter gives each pallet instance its own storage, origins, and events without code duplication.                                                |
