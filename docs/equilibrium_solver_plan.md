# Equilibrium Solver Stack & Validation Plan

This document outlines the solver stack, precomputation pipeline, runtime serving model, and validation harness for equilibrium solving (CFR/NFSP variants) for poker cash games and MTTs.

## 1) Solver stack selection

- **Core language**: Rust for safety, modern tooling, and FFI to CUDA (via `cust`/`cudarc`) or C++ kernels. Critical hot paths (traversal, regret/strategy updates, GPU kernels) live in Rust with optional C++ intrinsics for hand evaluation tables.
- **GPU acceleration**: CUDA kernels for batched regret updates, advantage computation, and value backups. Rust orchestrates kernels and manages host/device buffers; kernels can be authored in C++/CUDA or `rust-cuda` where practical.
- **Bindings**: Expose a clean FFI boundary for gRPC/REST servers (Tokio + `tonic`/`axum`). Python bindings (via `pyo3`) remain thin orchestration, not in hot loops.
- **Algorithm variants**: Implement CFR+, DCFR, MCCFR (outcome and external sampling), and NFSP (reinforcement/imitation split). Pluggable policy/value nets (TorchScript via `tch-rs`) for NFSP and abstraction reuse.

## 2) CFR/NFSP validation targets

- **Baseline correctness**: deterministic CFR+ on small limit-Hold'em abstractions to match known exploitability baselines (e.g., OpenSpiel data). Track regret convergence slope and exploitability over iterations.
- **NFSP**: validate mixture policy convergence on toy games (Kuhn/Leduc) and Hold'em abstractions; ensure anticipatory parameter schedules reproduce expected exploitability curves.
- **Performance**: measure iterations/sec and GPU utilization for MCCFR and DCFR; compare to CPU-only runs to validate speedups.

## 3) Precomputed tree ingestion & caching

- **Tree sources**: build/ingest public precomputed trees for standard cash (100bb, 50bb) and MTT stack depths (10–60bb) with common ante structures.
- **Format**: serialize trees and strategy slices in Parquet (columnar, compression, predicate pushdown) for cold storage; partition by game type, stack depth, ante, and street.
- **Local querying**: DuckDB readers for fast columnar scans and sub-tree materialization; store derived tensors (e.g., reach probs, counterfactual values) in Arrow buffers for zero-copy into Rust.
- **Hot cache**: Redis/KeyDB for frequently requested subtrees/strategies keyed by `(game, stack, ante, position, street, bucket)`. Use TTL + LFU; warm caches via background jobs based on request traces.
- **Build pipeline**: ETL step normalizes external trees -> schema, validates node counts/branching, and emits Parquet/metadata. CI step checks schema compatibility and versioning.

## 4) Node-locking & on-demand sims

- **Node-locking**: accept per-node strategy overrides (ranges, action frequencies). Apply deltas before traversal; maintain immutable base tree with overlay of locks for reproducibility.
- **On-demand solving**: if a request misses cache, spin up batched MCCFR/DCFR runs seeded from nearest precomputed abstraction; converge for N iterations or until exploitability target reached.
- **Batching & scheduling**: group compatible requests (same game/stack/street) to maximize GPU occupancy. Scheduler assigns batches to GPU workers; backpressure via bounded queues.
- **Serving layer**: gRPC (with streaming results) and REST endpoints wrap solver requests. Include request metadata (game, stack, antes, ranges, locks, budget) for tracing and cache keys.

## 5) Metrics & abstractions

- **Exploitability**: compute NashConv/exploitability in milli-blinds or chips/hand; track delta vs baseline abstraction per iteration and per-bucket.
- **EV loss**: report EV deltas vs precomputed strategy and vs locked nodes; expose percentile/mean EV loss across buckets.
- **Bucketing**: pluggable abstractions (e.g., EMD board texture buckets, hand strength/shape buckets, positional buckets). Maintain metadata for bucket definitions and mapping tables; store per-bucket regret/strategy tensors to accelerate approximate solves.

## 6) Validation harness

- **Known solutions**: compare exploitability and action frequencies against reference solutions (OpenSpiel, community baselines) for small games and curated Hold'em abstractions.
- **Regression tests**: deterministic seeds for CPU CFR+/DCFR; probabilistic tolerance bands for MCCFR/NFSP. Validate serialization/deserialization of trees and caches.
- **Performance checks**: benchmark iterations/sec, GPU utilization, cache hit rates, and latency percentiles across request types.
- **Spot checks**: random board/position samples with node-locks to ensure overrides are respected; replay solver traces to verify consistency between cached and on-demand paths.

## 7) Next steps (implementation order)

1. Define data schemas for trees/strategies (Parquet + metadata) and build the ETL validator.
2. Implement CPU CFR+ baseline in Rust; add exploitability metric and deterministic tests on toy games.
3. Add MCCFR/DCFR variants with batching abstractions; integrate CUDA kernels for regret/strategy updates.
4. Implement Redis/KeyDB cache and DuckDB-backed cold storage readers; wire cache warmer jobs.
5. Add node-locking overlays and on-demand solver path seeded from precomputed abstractions.
6. Expose gRPC/REST endpoints and collect metrics (exploitability, EV loss, latency, cache hits).
7. Expand NFSP pipeline with TorchScript policy/value nets and add integration tests vs reference curves.
