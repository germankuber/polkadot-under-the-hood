# Runtime WASM Injection Deep Dive: From Compilation to Execution

> A step-by-step walkthrough of how the Polkadot SDK runtime WASM is compiled, embedded in the chain specification, written to on-chain storage under the `:code` key, compiled by Wasmtime, cached in an LRU, and executed on demand through the Runtime API interface.

## Table of Contents

- [Runtime WASM Injection Deep Dive: From Compilation to Execution](#runtime-wasm-injection-deep-dive-from-compilation-to-execution)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
- [Part I: The Entry Point — From main.rs to service.rs](#part-i-the-entry-point--from-mainrs-to-servicers)
  - [1. main.rs: The Binary Entry Point](#1-mainrs-the-binary-entry-point)
  - [2. command.rs: CLI Parsing and Runner](#2-commandrs-cli-parsing-and-runner)
  - [3. load\_spec: Loading the Chain Specification](#3-load_spec-loading-the-chain-specification)
  - [4. Chain Spec Behavior: New Chain vs Existing Chain vs Restart](#4-chain-spec-behavior-new-chain-vs-existing-chain-vs-restart)
- [Part II: The Chain Specification — Where the WASM Comes From](#part-ii-the-chain-specification--where-the-wasm-comes-from)
  - [5. WASM\_BINARY: Compile-Time Embedding](#5-wasm_binary-compile-time-embedding)
  - [6. chain\_spec.rs: Embedding WASM in the Chain Spec](#6-chain_specrs-embedding-wasm-in-the-chain-spec)
  - [7. Chain Spec as JSON: The Portable Format](#7-chain-spec-as-json-the-portable-format)
- [Part III: Service Construction — Building the Node](#part-iii-service-construction--building-the-node)
  - [8. new\_partial: The Assembly Point](#8-new_partial-the-assembly-point)
  - [9. Type Aliases: How RuntimeApi Flows Through the System](#9-type-aliases-how-runtimeapi-flows-through-the-system)
  - [10. new\_wasm\_executor: Creating the WASM Engine](#10-new_wasm_executor-creating-the-wasm-engine)
  - [11. new\_full\_parts: Creating the Client](#11-new_full_parts-creating-the-client)
  - [12. new\_full\_parts\_with\_genesis\_builder: The Core Function](#12-new_full_parts_with_genesis_builder-the-core-function)
- [Part IV: Genesis Block Construction — Writing WASM to Storage](#part-iv-genesis-block-construction--writing-wasm-to-storage)
  - [13. GenesisBlockBuilder: From Chain Spec to Storage](#13-genesisblockbuilder-from-chain-spec-to-storage)
  - [14. build\_genesis\_block: The First WASM Execution](#14-build_genesis_block-the-first-wasm-execution)
  - [15. resolve\_state\_version\_from\_wasm: Reading Runtime Version](#15-resolve_state_version_from_wasm-reading-runtime-version)
  - [16. Client::new — Committing Genesis to the Database](#16-clientnew--committing-genesis-to-the-database)
- [Part V: The Client and RuntimeApi — Connecting the Pieces](#part-v-the-client-and-runtimeapi--connecting-the-pieces)
  - [17. The Client Struct: PhantomData and the RA Type](#17-the-client-struct-phantomdata-and-the-ra-type)
  - [18. ProvideRuntimeApi: Where Everything Connects](#18-provideruntimeapi-where-everything-connects)
  - [19. CallApiAt: From API Object to Executor](#19-callapiat-from-api-object-to-executor)
  - [20. Runtime API Instances: Lightweight and Disposable](#20-runtime-api-instances-lightweight-and-disposable)
- [Part VI: WASM Execution and Caching](#part-vi-wasm-execution-and-caching)
  - [21. RuntimeCache: The LRU Cache](#21-runtimecache-the-lru-cache)
  - [22. VersionedRuntime: The Compiled Runtime](#22-versionedruntime-the-compiled-runtime)
  - [23. create\_versioned\_wasm\_runtime: The Compilation Pipeline](#23-create_versioned_wasm_runtime-the-compilation-pipeline)
  - [24. Instance Pool: Reusing Compiled Code](#24-instance-pool-reusing-compiled-code)
  - [25. Runtime Upgrades: What Happens When :code Changes](#25-runtime-upgrades-what-happens-when-code-changes)
- [Part VII: Architecture — Runtime APIs vs Dispatch Functions](#part-vii-architecture--runtime-apis-vs-dispatch-functions)
  - [26. Runtime APIs: The Client-to-Runtime Interface](#26-runtime-apis-the-client-to-runtime-interface)
  - [27. Dispatch Functions: The User-to-Runtime Interface](#27-dispatch-functions-the-user-to-runtime-interface)
  - [28. Same Entry Path, Different Behavior](#28-same-entry-path-different-behavior)
  - [29. Creating Custom Runtime APIs](#29-creating-custom-runtime-apis)
- [Part VIII: Well-Known Keys and Storage Layout](#part-viii-well-known-keys-and-storage-layout)
  - [30. Well-Known Keys: The Special Storage Entries](#30-well-known-keys-the-special-storage-entries)
  - [31. Pallet Keys vs Well-Known Keys](#31-pallet-keys-vs-well-known-keys)
- [Complete Flow Summary](#complete-flow-summary)

---

## Introduction

When a Polkadot SDK node starts, it must load the runtime WASM code that defines the blockchain's logic, compile it to native code, and make it available for execution. This document traces the complete path from binary startup through service construction, genesis block creation, WASM caching, and runtime API execution.

Understanding this flow is essential for grasping how the Polkadot SDK achieves its key innovation: **forkless runtime upgrades**. Because the runtime is stored as WASM bytes in the on-chain state (not hardcoded in the binary), it can be upgraded through governance without requiring node operators to update their software.

**Key files referenced:**
- `templates/minimal/node/src/main.rs` — Binary entry point
- `templates/minimal/node/src/command.rs` — CLI parsing and runner
- `templates/minimal/node/src/chain_spec.rs` — Chain specification with embedded WASM
- `templates/minimal/node/src/service.rs` — Node service construction
- `substrate/client/service/src/lib.rs` — `new_full_parts`, `new_full_parts_with_genesis_builder`
- `substrate/client/service/src/client/client.rs` — `Client::new`, `ProvideRuntimeApi`
- `substrate/client/chain-spec/src/genesis_block.rs` — `GenesisBlockBuilder`, `build_genesis_block`
- `substrate/client/executor/src/wasm_runtime.rs` — `RuntimeCache`, `VersionedRuntime`, WASM compilation

---

# Part I: The Entry Point — From main.rs to service.rs

## 1. main.rs: The Binary Entry Point

The node binary starts in `main.rs`, which simply delegates to the command module:

```rust
fn main() -> polkadot_sdk::sc_cli::Result<()> {
    command::run()
}
```

No logic here — everything happens in `command::run()`.

## 2. command.rs: CLI Parsing and Runner

`command::run()` parses CLI arguments and decides what to do. The most important branch is when no subcommand is provided — the normal node startup:

```rust
pub fn run() -> sc_cli::Result<()> {
    let cli = Cli::from_args();

    match &cli.subcommand {
        Some(Subcommand::BuildSpec(cmd)) => { ... },
        Some(Subcommand::ExportBlocks(cmd)) => { ... },
        Some(Subcommand::PurgeChain(cmd)) => { ... },
        // ... other subcommands ...

        None => {
            let runner = cli.create_runner(&cli.run)?;
            runner.run_node_until_exit(|config| async move {
                service::new_full::<sc_network::NetworkWorker<_, _>>(config, cli.consensus)
                    .map_err(sc_cli::Error::Service)
            })
        },
    }
}
```

Three things happen in the `None` branch:

1. **`cli.create_runner()`** — Parses the CLI arguments and creates a `Configuration` struct containing the database path, network config, telemetry, and crucially the chain spec. During this step, `load_spec()` is called.
2. **`runner.run_node_until_exit()`** — Creates the tokio runtime and runs the async closure.
3. **`service::new_full()`** — Constructs the entire node: client, pool, network, consensus.

## 3. load_spec: Loading the Chain Specification

The `SubstrateCli` implementation defines how chain specs are loaded:

```rust
impl SubstrateCli for Cli {
    fn load_spec(&self, id: &str) -> Result<Box<dyn sc_service::ChainSpec>, String> {
        Ok(match id {
            "dev" => Box::new(chain_spec::development_chain_spec()?),
            path => {
                Box::new(chain_spec::ChainSpec::from_json_file(std::path::PathBuf::from(path))?)
            },
        })
    }
}
```

Two paths:
- `"dev"` — Generates the chain spec in memory using `development_chain_spec()`, which includes the compiled WASM blob directly from the runtime crate.
- Any other string — Treats it as a file path and parses a JSON chain spec from disk.

At this point, the chain spec is **only parsed into memory**. No WASM is compiled, no database is written, no runtime is loaded.

## 4. Chain Spec Behavior: New Chain vs Existing Chain vs Restart

The chain spec is always loaded when the node starts, but it's used differently depending on the state of the database:

**Case 1: New node, new blockchain (genesis)**
- The chain spec provides everything: WASM blob, initial balances, configuration.
- The genesis block builder writes all this to the database as block 0.
- The WASM goes into storage under the key `:code`.

**Case 2: New node, existing blockchain (sync)**
- The chain spec is used to identify which chain to connect to (the genesis hash serves as the network identifier).
- The database is empty, so the node syncs blocks from peers.
- The WASM that actually gets used comes from the synced state, not the chain spec. There may have been runtime upgrades since genesis.

**Case 3: Node restart (database already populated)**
- The chain spec is loaded to verify it matches the existing database (same chain).
- The database already has all the state, including the current `:code`.
- No genesis construction happens.

---

# Part II: The Chain Specification — Where the WASM Comes From

## 5. WASM_BINARY: Compile-Time Embedding

The WASM binary is produced during compilation of the runtime crate. In the runtime's `lib.rs`:

```rust
#[cfg(feature = "std")]
include!(concat!(env!("OUT_DIR"), "/wasm_binary.rs"));
```

This `include!` macro pulls in a file generated by the build process (`build.rs`) that contains:

```rust
pub const WASM_BINARY: Option<&[u8]> = Some(include_bytes!("...path to compiled wasm..."));
```

The build process:
1. Compiles the runtime crate to WebAssembly target (`wasm32-unknown-unknown`)
2. Runs `wasm-opt` for optimization
3. Compresses the blob with zstd
4. Embeds the compressed bytes as a Rust constant

The `#[cfg(feature = "std")]` gate means this constant only exists in native builds (not inside the WASM runtime itself — that would be recursive).

## 6. chain_spec.rs: Embedding WASM in the Chain Spec

The chain spec constructor takes `WASM_BINARY` and passes it as the first argument to the builder:

```rust
use minimal_template_runtime::WASM_BINARY;

pub fn development_chain_spec() -> Result<ChainSpec, String> {
    Ok(ChainSpec::builder(
            WASM_BINARY.expect("Development wasm not available"),  // ← WASM bytes here
            Default::default(),
        )
        .with_name("Development")
        .with_id("dev")
        .with_chain_type(ChainType::Development)
        .with_genesis_config_preset_name(sp_genesis_builder::DEV_RUNTIME_PRESET)
        .with_properties(props())
        .build())
}
```

The builder stores the WASM bytes. When `build_storage()` is called later (during genesis construction), these bytes are placed in the storage map under the key `:code`.

## 7. Chain Spec as JSON: The Portable Format

For production or sharing between nodes, the chain spec can be exported to JSON:

```bash
# 1. Compile the runtime to WASM
cargo build --release
# The WASM ends up at:
# target/release/wbuild/<runtime-crate>/<runtime>.compact.compressed.wasm

# 2. Generate the chain spec JSON
./target/release/minimal-template-node build-spec --chain dev > chain-spec.json

# 3. (Optional) Convert to raw format (hashes the storage keys)
./target/release/minimal-template-node build-spec --chain chain-spec.json --raw > chain-spec-raw.json
```

The raw JSON contains the WASM as a hex-encoded blob:

```json
{
  "name": "Development",
  "id": "dev",
  "chainType": "Development",
  "genesis": {
    "raw": {
      "top": {
        "0x3a636f6465": "0x52bc1760..."
      }
    }
  }
}
```

`0x3a636f6465` is the hex encoding of `":code"`. The value is the compressed WASM blob in hex. When another node starts with `--chain chain-spec-raw.json`, it reads this JSON, extracts the WASM, and follows the same genesis construction path.

---

# Part III: Service Construction — Building the Node

## 8. new_partial: The Assembly Point

**File:** `templates/minimal/node/src/service.rs`

`new_partial` creates the fundamental components of the node. The first thing to notice is the imports at the top of the file:

```rust
use minimal_template_runtime::{interface::OpaqueBlock as Block, RuntimeApi};
```

Two types are imported from the runtime crate:
- `Block` — The block type (header + body structure)
- `RuntimeApi` — The type generated by `impl_runtime_apis!` that defines which runtime APIs are available

These are **types**, not values. They carry no data — they exist purely for compile-time type safety.

## 9. Type Aliases: How RuntimeApi Flows Through the System

```rust
type HostFunctions = sp_io::SubstrateHostFunctions;

pub(crate) type FullClient =
    sc_service::TFullClient<Block, RuntimeApi, WasmExecutor<HostFunctions>>;
```

`TFullClient` expands to:

```rust
pub type TFullClient<TBl, TRtApi, TExec> =
    Client<TFullBackend<TBl>, TFullCallExecutor<TBl, TExec>, TBl, TRtApi>;
```

Which is:

```rust
Client<Backend<Block>, LocalCallExecutor<Block, Backend<Block>, WasmExecutor<HostFunctions>>, Block, RuntimeApi>
```

`RuntimeApi` is the fourth type parameter. It will end up as `PhantomData<RA>` inside the `Client` struct — it occupies zero bytes in memory but tells the compiler what runtime API functions this client can call.

## 10. new_wasm_executor: Creating the WASM Engine

```rust
let executor = sc_service::new_wasm_executor(&config.executor);
```

**File:** `substrate/client/service/src/lib.rs`

```rust
pub fn new_wasm_executor<H: HostFunctions>(config: &ExecutorConfiguration) -> WasmExecutor<H> {
    let strategy = config
        .default_heap_pages
        .map_or(DEFAULT_HEAP_ALLOC_STRATEGY, |p| HeapAllocStrategy::Static { extra_pages: p as _ });
    WasmExecutor::<H>::builder()
        .with_execution_method(config.wasm_method)
        .with_onchain_heap_alloc_strategy(strategy)
        .with_offchain_heap_alloc_strategy(strategy)
        .with_max_runtime_instances(config.max_runtime_instances)
        .with_runtime_cache_size(config.runtime_cache_size)
        .build()
}
```

This creates the `WasmExecutor` — the component that knows how to compile and execute WASM using Wasmtime. It's configured with:
- The execution method (Wasmtime compiled, which is the only option now)
- Heap allocation strategy (how much memory to give the WASM runtime)
- Maximum runtime instances (for the instance pool)
- Runtime cache size (how many compiled runtimes to keep in the LRU)

At this point, no WASM has been loaded or compiled — the executor is just ready to work when asked.

## 11. new_full_parts: Creating the Client

```rust
let (client, backend, keystore_container, task_manager) =
    sc_service::new_full_parts::<Block, RuntimeApi, _>(
        config,
        telemetry.as_ref().map(|(_, telemetry)| telemetry.handle()),
        executor,
        Default::default(),
    )?;
let client = Arc::new(client);
```

`new_full_parts` delegates through a chain:

```
new_full_parts()
  → new_full_parts_record_import()
    → creates Backend (RocksDB)
    → creates GenesisBlockBuilder from chain_spec
    → new_full_parts_with_genesis_builder()
```

## 12. new_full_parts_with_genesis_builder: The Core Function

**File:** `substrate/client/service/src/lib.rs`

```rust
pub fn new_full_parts_with_genesis_builder<TBl, TRtApi, TExec, TBuildGenesisBlock>(
    config: &Configuration,
    telemetry: Option<TelemetryHandle>,
    executor: TExec,
    backend: Arc<TFullBackend<TBl>>,
    genesis_block_builder: TBuildGenesisBlock,
    enable_import_proof_recording: bool,
) -> Result<TFullParts<TBl, TRtApi, TExec>, Error>
```

Note that `TRtApi` (the `RuntimeApi` type) appears as a generic parameter but is **never used directly in the function body**. It only flows through to the return type `TFullParts`, which includes `TFullClient<TBl, TRtApi, TExec>`. The type is carried through purely for compile-time safety.

The function:
1. Creates the keystore
2. Creates the task manager
3. Handles code substitutes (for wasm runtime overrides)
4. Calls `new_client()` to create the actual client, passing the `genesis_block_builder`

```rust
let client = new_client(
    backend.clone(),
    executor,
    genesis_block_builder,
    fork_blocks,
    bad_blocks,
    extensions,
    Box::new(task_manager.spawn_handle()),
    config.prometheus_config.as_ref().map(|config| config.registry.clone()),
    telemetry,
    ClientConfig { ... },
)?;
```

---

# Part IV: Genesis Block Construction — Writing WASM to Storage

## 13. GenesisBlockBuilder: From Chain Spec to Storage

**File:** `substrate/client/chain-spec/src/genesis_block.rs`

Before `new_full_parts_with_genesis_builder` is called, the genesis block builder is created in `new_full_parts_record_import`:

```rust
let genesis_block_builder = GenesisBlockBuilder::new(
    config.chain_spec.as_storage_builder(),  // ← chain spec as a storage provider
    !config.no_genesis(),
    backend.clone(),
    executor.clone(),
)?;
```

The constructor calls `build_storage()` on the chain spec to generate the full genesis storage:

```rust
impl<Block: BlockT, B: Backend<Block>, E: RuntimeVersionOf> GenesisBlockBuilder<Block, B, E> {
    pub fn new(
        build_genesis_storage: &dyn BuildStorage,
        commit_genesis_state: bool,
        backend: Arc<B>,
        executor: E,
    ) -> sp_blockchain::Result<Self> {
        let genesis_storage =
            build_genesis_storage.build_storage().map_err(sp_blockchain::Error::Storage)?;
        Self::new_with_storage(genesis_storage, commit_genesis_state, backend, executor)
    }
}
```

`build_storage()` returns a `Storage` struct — essentially a `HashMap<Vec<u8>, Vec<u8>>` with all the key-value pairs for the genesis state. This includes:
- `:code` → WASM bytes (from the chain spec)
- `:heappages` → heap page configuration
- All pallet storage: account balances, sudo key, configuration, etc.

## 14. build_genesis_block: The First WASM Execution

```rust
impl<Block: BlockT, B: Backend<Block>, E: RuntimeVersionOf> BuildGenesisBlock<Block>
    for GenesisBlockBuilder<Block, B, E>
{
    fn build_genesis_block(self) -> sp_blockchain::Result<(Block, Self::BlockImportOperation)> {
        let Self { genesis_storage, commit_genesis_state, backend, executor, _phantom } = self;

        // 1. Execute WASM just to read the state version
        let genesis_state_version =
            resolve_state_version_from_wasm::<_, HashingFor<Block>>(&genesis_storage, &executor)?;

        // 2. Write ALL genesis storage to the backend
        let mut op = backend.begin_operation()?;
        let state_root =
            op.set_genesis_state(genesis_storage, commit_genesis_state, genesis_state_version)?;

        // 3. Construct the genesis block with the computed state root
        let genesis_block = construct_genesis_block::<Block>(state_root, genesis_state_version);

        Ok((genesis_block, op))
    }
}
```

Step 1 is critical: before writing the genesis state, we need to know which state trie version (V0 or V1) the runtime expects. This requires **executing the WASM** — the very first WASM execution in the entire lifecycle of the node.

Step 2 writes everything to the database backend. The WASM bytes from the chain spec become the value under the `:code` key in the state trie.

Step 3 constructs block 0 (the genesis block) with the computed state root hash.

## 15. resolve_state_version_from_wasm: Reading Runtime Version

**File:** `substrate/client/chain-spec/src/genesis_block.rs`

```rust
pub fn resolve_state_version_from_wasm<E, H>(
    storage: &Storage,
    executor: &E,
) -> sp_blockchain::Result<StateVersion>
where
    E: RuntimeVersionOf,
    H: HashT,
{
    if let Some(wasm) = storage.top.get(well_known_keys::CODE) {
        let mut ext = sp_state_machine::BasicExternalities::new_empty();

        let code_fetcher = sp_core::traits::WrappedRuntimeCode(wasm.as_slice().into());
        let runtime_code = sp_core::traits::RuntimeCode {
            code_fetcher: &code_fetcher,
            heap_pages: None,
            hash: <H as HashT>::hash(wasm).encode(),
        };
        let runtime_version = RuntimeVersionOf::runtime_version(executor, &mut ext, &runtime_code)
            .map_err(|e| sp_blockchain::Error::VersionInvalid(e.to_string()))?;
        Ok(runtime_version.state_version())
    } else {
        Err(sp_blockchain::Error::VersionInvalid(
            "Runtime missing from initial storage, could not read state version.".to_string(),
        ))
    }
}
```

Step by step:

1. **`storage.top.get(well_known_keys::CODE)`** — Looks up the WASM bytes under the key `":code"` in the genesis storage. If not present, the node cannot start.

2. **`BasicExternalities::new_empty()`** — Creates a minimal externalities environment. The WASM runtime needs host functions available to execute, but since we only want to read the version, an empty environment suffices.

3. **`RuntimeCode { code_fetcher, heap_pages, hash }`** — Packages the WASM bytes for the executor. The hash is used for cache lookups — if the same WASM was compiled before, the executor can reuse the cached compilation.

4. **`RuntimeVersionOf::runtime_version(executor, &mut ext, &runtime_code)`** — Executes the WASM to call `Core_version()`. This returns the `RuntimeVersion` struct which includes the `state_version` field.

5. **`runtime_version.state_version()`** — Extracts whether the runtime uses state trie V0 or V1, which determines how the genesis trie is constructed.

## 16. Client::new — Committing Genesis to the Database

**File:** `substrate/client/service/src/client/client.rs`

```rust
pub fn new<G>(
    backend: Arc<B>,
    executor: E,
    spawn_handle: Box<dyn SpawnNamed>,
    genesis_block_builder: G,
    fork_blocks: ForkBlocks<Block>,
    bad_blocks: BadBlocks<Block>,
    prometheus_registry: Option<Registry>,
    telemetry: Option<TelemetryHandle>,
    config: ClientConfig<Block>,
) -> sp_blockchain::Result<Self>
where
    G: BuildGenesisBlock<Block, BlockImportOperation = ...>,
{
    let info = backend.blockchain().info();
    if info.finalized_state.is_none() {
        let (genesis_block, mut op) = genesis_block_builder.build_genesis_block()?;
        info!(
            "🔨 Initializing Genesis block/state (state: {}, header-hash: {})",
            genesis_block.header().state_root(),
            genesis_block.header().hash()
        );
        let block_state = if info.best_hash == Default::default() {
            NewBlockState::Final
        } else {
            NewBlockState::Normal
        };
        let (header, body) = genesis_block.deconstruct();
        op.set_block_data(header, Some(body), None, None, block_state, true)?;
        backend.commit_operation(op)?;
    }
    // ... create notification workers, return Client struct ...
}
```

The check `info.finalized_state.is_none()` determines whether the database is empty. If it is:
1. Calls `genesis_block_builder.build_genesis_block()` — which we just traced above
2. Sets the genesis block data (header, body)
3. Commits the operation to the backend (RocksDB)

After `backend.commit_operation(op)`, the WASM bytes are persisted in RocksDB under the `:code` key. They will survive node restarts.

If the database is already populated (restart scenario), this entire block is skipped.

---

# Part V: The Client and RuntimeApi — Connecting the Pieces

## 17. The Client Struct: PhantomData and the RA Type

```rust
pub struct Client<B, E, Block, RA>
where
    Block: BlockT,
{
    backend: Arc<B>,
    executor: E,
    storage_notifications: StorageNotifications<Block>,
    import_notification_sinks: NotificationSinks<BlockImportNotification<Block>>,
    finality_notification_sinks: NotificationSinks<FinalityNotification<Block>>,
    importing_block: RwLock<Option<Block::Hash>>,
    block_rules: BlockRules<Block>,
    config: ClientConfig<Block>,
    telemetry: Option<TelemetryHandle>,
    unpin_worker_sender: TracingUnboundedSender<UnpinWorkerMessage<Block>>,
    code_provider: CodeProvider<Block, B, E>,
    _phantom: PhantomData<RA>,   // ← RuntimeApi lives here
}
```

`RA` (the `RuntimeApi` type) is stored as `PhantomData` — it occupies zero bytes in memory. The type exists purely to satisfy the Rust compiler's type system. It tells the compiler: "this client knows about these runtime APIs" without storing any actual data.

## 18. ProvideRuntimeApi: Where Everything Connects

```rust
impl<B, E, Block, RA> ProvideRuntimeApi<Block> for Client<B, E, Block, RA>
where
    B: backend::Backend<Block>,
    E: CallExecutor<Block, Backend = B> + Send + Sync,
    Block: BlockT,
    RA: ConstructRuntimeApi<Block, Self> + Send + Sync,
{
    type Api = <RA as ConstructRuntimeApi<Block, Self>>::RuntimeApi;

    fn runtime_api(&self) -> ApiRef<'_, Self::Api> {
        RA::construct_runtime_api(self)
    }
}
```

When anyone calls `client.runtime_api()`:

1. `RA::construct_runtime_api(self)` is called — `RA` is the `RuntimeApi` type generated by `impl_runtime_apis!`
2. It creates a lightweight API object that holds a reference to the client
3. This object implements all the runtime API traits (`TaggedTransactionQueue`, `Core`, `BlockBuilder`, etc.)
4. When you call a method on it (like `.validate_transaction()`), it uses `CallApiAt` to execute the WASM

## 19. CallApiAt: From API Object to Executor

```rust
impl<B, E, Block, RA> CallApiAt<Block> for Client<B, E, Block, RA>
where
    B: backend::Backend<Block>,
    E: CallExecutor<Block, Backend = B> + Send + Sync,
    Block: BlockT,
    RA: Send + Sync,
{
    type StateBackend = B::State;

    fn call_api_at(&self, params: CallApiAtParams<Block>) -> Result<Vec<u8>, sp_api::ApiError> {
        self.executor
            .contextual_call(
                params.at,
                params.function,
                &params.arguments,
                params.overlayed_changes,
                params.recorder,
                params.call_context,
                params.extensions,
            )
            .map_err(Into::into)
    }
}
```

This is the final bridge: the API object calls `call_api_at`, which delegates to the executor's `contextual_call`. The executor reads `:code` from the state at the given block hash, looks up the compiled runtime in the cache (or compiles it), and executes the requested function.

## 20. Runtime API Instances: Lightweight and Disposable

Each call to `client.runtime_api()` creates a **new** API object. This is intentional and cheap — the object is essentially a wrapper that holds a reference to the client and an empty `OverlayedChanges` (a scratch pad for state changes during execution).

```
client.runtime_api()             → creates new API object (cheap)
  .validate_transaction(...)     → executes WASM
                                    → first time: compiles WASM (expensive)
                                    → subsequent: uses cache (cheap)

client.runtime_api()             → creates ANOTHER API object (cheap)
  .validate_transaction(...)     → uses cached WASM (cheap)
```

A new object is created each time because the `OverlayedChanges` from one call must not contaminate the next. For `validate_transaction`, changes are always discarded (validation is read-only). For `execute_block`, changes are committed to storage.

---

# Part VI: WASM Execution and Caching

## 21. RuntimeCache: The LRU Cache

**File:** `substrate/client/executor/src/wasm_runtime.rs`

```rust
pub struct RuntimeCache {
    runtimes: Mutex<LruMap<VersionedRuntimeId, Arc<VersionedRuntime>>>,
    max_runtime_instances: usize,
    cache_path: Option<PathBuf>,
}
```

The `RuntimeCache` is an **LRU (Least Recently Used)** cache. When it's full and a new entry needs to be inserted, the entry that hasn't been used for the longest time is evicted.

The cache key is:

```rust
struct VersionedRuntimeId {
    code_hash: Vec<u8>,
    wasm_method: WasmExecutionMethod,
    heap_alloc_strategy: HeapAllocStrategy,
}
```

The `code_hash` is the hash of the WASM bytes stored under `:code`. If the runtime is upgraded on-chain, the new WASM has a different hash, resulting in a cache miss and a new compilation.

The cache lookup and compilation flow:

```rust
pub fn with_instance<'c, H, R, F>(&self, runtime_code, ext, wasm_method, ..., f) -> Result<...> {
    let versioned_runtime_id = VersionedRuntimeId {
        code_hash: code_hash.clone(),
        heap_alloc_strategy,
        wasm_method,
    };

    let mut runtimes = self.runtimes.lock();
    let versioned_runtime = if let Some(vr) = runtimes.get(&versioned_runtime_id) {
        vr.clone()                                           // CACHE HIT
    } else {
        let code = runtime_code.fetch_runtime_code()?;       // Read :code from storage
        let result = create_versioned_wasm_runtime::<H>(...); // Compile with Wasmtime
        runtimes.insert(versioned_runtime_id, result.clone()); // Store in cache
        result                                                // CACHE MISS → compiled
    };

    drop(runtimes);  // Release lock before executing
    Ok(versioned_runtime.with_instance(ext, f))
}
```

## 22. VersionedRuntime: The Compiled Runtime

```rust
struct VersionedRuntime {
    module: Box<dyn WasmModule>,
    version: Option<RuntimeVersion>,
    instances: Vec<Mutex<Option<Box<dyn WasmInstance>>>>,
}
```

Each entry in the cache is a `VersionedRuntime` containing:
- **`module`** — The WASM compiled to native code by Wasmtime. This is the expensive part to produce.
- **`version`** — The cached `RuntimeVersion` (spec name, version numbers, etc.), so we don't need to execute `Core_version` every time.
- **`instances`** — A pool of reusable WASM instances. Each instance has its own memory and can execute functions independently.

## 23. create_versioned_wasm_runtime: The Compilation Pipeline

```rust
fn create_versioned_wasm_runtime<H>(
    code: &[u8],
    ext: &mut dyn Externalities,
    wasm_method: WasmExecutionMethod,
    heap_alloc_strategy: HeapAllocStrategy,
    allow_missing_func_imports: bool,
    max_instances: usize,
    cache_path: Option<&Path>,
) -> Result<VersionedRuntime, WasmError>
where
    H: HostFunctions,
{
    // 1. Decompress the WASM blob
    let blob = RuntimeBlob::uncompress_if_needed(code)?;

    // 2. Try to read the version from custom WASM sections (fast path)
    let mut version = read_embedded_version(&blob)?;

    // 3. Compile with Wasmtime
    let runtime = create_wasm_runtime_with_code::<H>(
        wasm_method, heap_alloc_strategy, blob, allow_missing_func_imports, cache_path,
    )?;

    // 4. If no embedded version, call Core_version (slow path)
    if version.is_none() {
        let version_result = runtime.new_instance()?.call("Core_version".into(), &[]);
        if let Ok(version_buf) = version_result {
            version = Some(decode_version(&version_buf)?);
        }
    }

    // 5. Pre-allocate the instance pool
    let mut instances = Vec::with_capacity(max_instances);
    instances.resize_with(max_instances, || Mutex::new(None));

    Ok(VersionedRuntime { module: runtime, version, instances })
}
```

Step 2 is an optimization: modern runtimes embed their version in custom WASM sections (`runtime_version` and `runtime_apis`). Reading these sections is much faster than executing `Core_version`, because it doesn't require instantiating the WASM module.

Step 3 is the expensive part: Wasmtime JIT-compiles the WASM to native machine code. This can take hundreds of milliseconds for a full runtime.

## 24. Instance Pool: Reusing Compiled Code

Each `VersionedRuntime` maintains a pool of instances. When a runtime API call is made:

```rust
impl VersionedRuntime {
    fn with_instance<R, F>(&self, ext: &mut dyn Externalities, f: F) -> Result<R, Error> {
        // Try to find a free instance in the pool
        let instance = self.instances.iter()
            .find_map(|(index, i)| i.try_lock().map(|i| (index, i)));

        match instance {
            Some((index, mut locked)) => {
                // Reuse existing instance or create a new one from the module
                let (mut instance, new_inst) = locked.take()
                    .map(|r| Ok((r, false)))
                    .unwrap_or_else(|| self.module.new_instance().map(|i| (i, true)))?;

                let result = f(&*self.module, &mut *instance, self.version.as_ref(), ext);

                if result.is_ok() {
                    *locked = Some(instance);  // Return to pool for reuse
                }
                // On error, the instance is dropped (not returned to pool)

                result
            },
            None => {
                // All pool slots are in use — create a temporary instance
                let mut instance = self.module.new_instance()?;
                f(&*self.module, &mut *instance, self.version.as_ref(), ext)
            },
        }
    }
}
```

Creating an instance from a compiled module is much cheaper than compiling the module itself. The instance gets its own linear memory and execution stack, but the compiled code is shared.

On success, the instance is returned to the pool. On error, it's dropped — a fresh instance will be created next time to avoid reusing a potentially corrupted state.

## 25. Runtime Upgrades: What Happens When :code Changes

A runtime upgrade is a regular dispatch call (`system.set_code(new_wasm_bytes)`) that writes new bytes to `:code` in storage. From the executor's perspective, nothing special happens:

1. The new WASM bytes are written to `:code` by the runtime upgrade extrinsic
2. The block containing the upgrade is committed to the database
3. The next call to `client.runtime_api()` against a block **after** the upgrade will:
   - Read `:code` from the new state
   - Compute its hash
   - Look up the hash in the RuntimeCache LRU
   - **Cache miss** — the hash is different from the previous runtime
   - Compile the new WASM with Wasmtime
   - Cache it under the new hash
   - Execute

The old compiled runtime stays in the LRU cache until it's evicted by newer entries. Queries against old blocks (before the upgrade) will still use the old runtime, since the state at those blocks has the old `:code`. This is how archival queries work correctly even across runtime upgrades.

There is no explicit cache invalidation. The natural behavior of the hash-based cache key handles everything:

```
Before upgrade:  :code has hash H1 → LRU has entry for H1
After upgrade:   :code has hash H2 → LRU misses on H2 → compiles → caches H2
Old block query: :code at old block still has H1 → LRU hits on H1
Eventually:      H1 evicted from LRU by lack of use
```

---

# Part VII: Architecture — Runtime APIs vs Dispatch Functions

## 26. Runtime APIs: The Client-to-Runtime Interface

Runtime APIs are functions that the **node** (client side) calls into the runtime (WASM). They are declared with `decl_runtime_apis!` in primitive crates and implemented with `impl_runtime_apis!` in the runtime's `lib.rs`.

```rust
// Declaration (in substrate/primitives/transaction-pool/src/runtime_api.rs)
sp_api::decl_runtime_apis! {
    pub trait TaggedTransactionQueue {
        fn validate_transaction(
            source: TransactionSource,
            tx: <Block as BlockT>::Extrinsic,
            block_hash: <Block as BlockT>::Hash,
        ) -> TransactionValidity;
    }
}

// Implementation (in the runtime's lib.rs, inside impl_runtime_apis!)
impl sp_transaction_pool::runtime_api::TaggedTransactionQueue<Block> for Runtime {
    fn validate_transaction(source, tx, block_hash) -> TransactionValidity {
        Executive::validate_transaction(source, tx, block_hash)
    }
}
```

Runtime APIs:
- Have no `origin` — there's no user authorizing the call
- Don't charge fees — they're internal node operations
- Don't require consensus — each node can call them independently
- Are not included in blocks — they're off-chain queries/operations

Standard runtime APIs that come with Substrate:

| API | Purpose |
|-----|---------|
| `Core` | `version()`, `execute_block()`, `initialize_block()` |
| `TaggedTransactionQueue` | `validate_transaction()` |
| `BlockBuilder` | `apply_extrinsic()`, `finalize_block()`, `inherent_extrinsics()` |
| `Metadata` | `metadata()`, `metadata_versions()` |
| `AccountNonceApi` | `account_nonce()` |
| `TransactionPaymentApi` | `query_info()`, `query_fee_details()` |

## 27. Dispatch Functions: The User-to-Runtime Interface

Dispatch functions are defined inside pallets with `#[pallet::call]`:

```rust
#[pallet::call]
impl<T: Config> Pallet<T> {
    pub fn transfer(origin: OriginFor<T>, dest: T::AccountId, amount: u128) -> DispatchResult {
        let who = ensure_signed(origin)?;
        // modifies storage...
    }
}
```

`construct_runtime!` gathers all pallet calls into a single `RuntimeCall` enum that can be dispatched. Dispatch functions:
- Have an `origin` — someone must authorize the action
- Charge fees — to prevent spam
- Require consensus — all nodes must agree on the result
- Are included in blocks — as extrinsics

## 28. Same Entry Path, Different Behavior

Both runtime APIs and dispatch functions enter the runtime through the same path:

```
client.runtime_api()
  → ProvideRuntimeApi::runtime_api()
    → RA::construct_runtime_api(self)
      → api.some_function()
        → CallApiAt::call_api_at()
          → executor.contextual_call()
            → RuntimeCache::with_instance()
              → instance.call("FunctionName", &params)
                → Wasmtime executes native compiled code
```

The string passed to `instance.call()` determines which function runs inside the WASM:

```
instance.call("TaggedTransactionQueue_validate_transaction", &params)  → no dispatch
instance.call("Core_execute_block", &params)                           → dispatches all txs
instance.call("BlockBuilder_apply_extrinsic", &params)                 → dispatches one tx
instance.call("Core_version", &params)                                 → no dispatch
```

From the executor's perspective, they're all the same: "call this function with these bytes, give me the result." What each function does inside the WASM is up to the runtime.

The key distinction:
- **Needs consensus** (all nodes must agree, state changes permanently) → dispatch function in a pallet
- **Internal tool or query** (each node calls independently, may or may not persist changes) → runtime API

## 29. Creating Custom Runtime APIs

Developers can create their own runtime APIs. The declaration goes in a **separate crate** (not inside a pallet), typically in a primitives crate:

```rust
// my-pallet-runtime-api/src/lib.rs
sp_api::decl_runtime_apis! {
    pub trait MyCustomApi {
        fn calculate_something(input: u64) -> u64;
        fn get_special_data(account: AccountId) -> Option<MyData>;
    }
}
```

The implementation goes in the runtime's `lib.rs`:

```rust
impl_runtime_apis! {
    impl my_pallet_runtime_api::MyCustomApi<Block> for Runtime {
        fn calculate_something(input: u64) -> u64 {
            // access state, perform calculations, etc.
        }
    }
}
```

The API can then be called from the client side — typically from a custom RPC handler:

```rust
fn calculate_something(&self, input: u64) -> RpcResult<u64> {
    let api = self.client.runtime_api();
    let best = self.client.info().best_hash;
    api.calculate_something(best, input).map_err(|e| ...)
}
```

Or directly via `state_call` without a custom RPC:

```
state_call("MyCustomApi_calculate_something", encoded_params)
```

---

# Part VIII: Well-Known Keys and Storage Layout

## 30. Well-Known Keys: The Special Storage Entries

The runtime WASM is stored under a well-known key — a special storage key defined in the Substrate primitives:

```rust
// substrate/primitives/core/src/storage.rs
pub mod well_known_keys {
    pub const CODE: &[u8] = b":code";
    pub const HEAP_PAGES: &[u8] = b":heappages";
    pub const EXTRINSIC_INDEX: &[u8] = b":extrinsic_index";
}
```

These keys are stored **unhashed** in the state trie — the literal bytes `":code"` are used as the key. This is done with `storage::unhashed::put_raw()`:

```rust
storage::unhashed::put_raw(well_known_keys::CODE, &new_wasm_bytes);
```

The `:` prefix ensures these keys can never collide with pallet storage keys, because pallet key hashes never start with `:`.

## 31. Pallet Keys vs Well-Known Keys

Regular pallet storage uses a hashed key scheme:

```
key = twox128(pallet_name) ++ twox128(storage_name) ++ hasher(storage_key)
```

For example, the `Account` storage in `frame_system`:

```
twox128("System") ++ twox128("Account") ++ blake2_128_concat(account_id)
```

Both pallet keys and well-known keys live in the same state trie, but are constructed differently:

```
State trie:
  ":code"                                                    → WASM bytes (unhashed)
  ":heappages"                                               → heap config (unhashed)
  twox128("System") ++ twox128("Account") ++ blake2(Alice)   → AccountInfo (hashed)
  twox128("Balances") ++ twox128("TotalIssuance")            → u128 (hashed)
```

RocksDB doesn't care whether a key is hashed or not — it stores raw bytes. The distinction only matters to the runtime and client code that reads and writes storage.

---

# Complete Flow Summary

```
COMPILE TIME:
  cargo build
    → compiles runtime crate to WASM target
    → generates WASM_BINARY constant (compressed bytes)

NODE STARTUP:
  main.rs → command::run()
    → Cli::from_args() parses CLI
    → load_spec("dev") → development_chain_spec()
      → ChainSpec::builder(WASM_BINARY, ...).build()
      → WASM bytes embedded in chain spec (in memory)

  cli.create_runner()
    → creates Configuration { chain_spec, ... }

  runner.run_node_until_exit()
    → service::new_full(config, consensus)
      → service::new_partial(&config)

SERVICE CONSTRUCTION:
  new_partial():
    1. executor = new_wasm_executor()              → creates WasmExecutor
    2. new_full_parts::<Block, RuntimeApi, _>()
       → new_full_parts_record_import()
         → backend = new_db_backend()              → opens RocksDB
         → genesis_block_builder = GenesisBlockBuilder::new(
               chain_spec.as_storage_builder()     → extracts genesis storage
           )
         → new_full_parts_with_genesis_builder()
           → new_client(backend, executor, genesis_block_builder, ...)

CLIENT CREATION:
  Client::new():
    if database is empty (finalized_state.is_none()):
      → genesis_block_builder.build_genesis_block()
        → build_storage()                          → HashMap { ":code" → WASM, ... }
        → resolve_state_version_from_wasm()        → FIRST WASM EXECUTION (Core_version)
        → op.set_genesis_state(storage)            → writes all keys to trie
        → construct_genesis_block(state_root)      → creates block 0
      → backend.commit_operation(op)               → persists to RocksDB
      → ":code" → WASM bytes now in storage
    else:
      → skip (database already has state)

RUNTIME EXECUTION (on demand):
  client.runtime_api()                             → RA::construct_runtime_api(self)
    .validate_transaction(at, ...)                 → CallApiAt::call_api_at()
      → executor.contextual_call()                 → LocalCallExecutor
        → RuntimeCache::with_instance()
          → build key: VersionedRuntimeId { code_hash, method, heap }
          → LRU lookup:
            HIT  → reuse compiled module, take instance from pool
            MISS → fetch :code from storage
                 → uncompress_if_needed()
                 → read_embedded_version() (custom WASM sections)
                 → create_wasm_runtime_with_code() (Wasmtime JIT compile)
                 → store in LRU cache
          → instance.call("FunctionName", &params) → Wasmtime executes

RUNTIME UPGRADE:
  system.set_code(new_wasm) dispatched in a block
    → new bytes written to :code in storage
    → block committed to database
    → next runtime_api() call against new state:
      → :code has new bytes → new hash → LRU MISS → recompile → cache
    → queries against old blocks still use old runtime (old :code, old hash, LRU HIT)
```