# Polkadot Under the Hood

<p align="center">
  <img src="./assets/polkadot-logo.jpg" alt="Polkadot Logo" width="200">
</p>

Deep dives into Polkadot/Substrate internals with detailed code analysis and line-by-line explanations.

## Contents

| Document | Description |
|----------|-------------|
| [Parachain Block Lifecycle](./parachain-block-lifecycle.md) | Complete lifecycle of a block in a Substrate parachain — from collator slot claim through runtime execution, storage root calculation, and persistence to RocksDB |
| [Parachain Inherent Data and XCM Processing](./parachain-inherent-data-and-xcm-processing.md) | How parachains receive data from the relay chain — fetching DMP/HRMP messages, state proof verification, and message queue processing |
| [XCM Message Execution Pipeline](./xcm-message-execution-pipeline.md) | How XCM messages execute during on_idle — from pallet-message-queue through XcmExecutor instruction processing |

## About

This repository documents the internal workings of Polkadot SDK components by tracing code paths end-to-end. Each document includes:

- Step-by-step flow explanations
- Code snippets from the actual `polkadot-sdk` repository
- File paths and line references
- Diagrams where helpful

## References

- [Polkadot SDK Repository](https://github.com/paritytech/polkadot-sdk)
- [Polkadot Wiki](https://wiki.polkadot.network/)
- [Substrate Documentation](https://docs.substrate.io/)

## License

MIT
