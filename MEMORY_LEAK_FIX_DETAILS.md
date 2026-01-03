# Memory Leak Fix Details

## Issue Analysis
The reported memory leak occurred when `get_async_subtensor` was called in a loop.
Originally suspected to be due to `initialize()` failures not cleaning up resources, further investigation revealed a deeper issue in the underlying `async_substrate_interface` library.

### Root Cause
The `async_substrate_interface` library uses a custom `@cached_fetcher` decorator to cache results of certain method calls (like `get_block_hash`, `get_runtime_for_version`).
The implementation of this decorator (`_CachedFetcherMethod`) stored a strong reference to the `AsyncSubstrateInterface` instance in a dictionary (`self._instances`), which is attached to the class itself (as a descriptor).

Because `_CachedFetcherMethod` lives on the class, and it held strong references to every `AsyncSubstrateInterface` instance ever created (keyed by the instance itself), these instances were never garbage collected. This caused a massive memory leak where every connection object persisted for the lifetime of the process.

### The Fix
We implemented a runtime patch (monkeypatch) for `async_substrate_interface` within `bittensor`.

1.  **Weak Reference Storage:** We patched `_CachedFetcherMethod` to use `weakref.WeakKeyDictionary` for `self._instances` instead of a standard `dict`. This allows the dictionary entry to be removed when the `AsyncSubstrateInterface` instance is no longer referenced elsewhere.
2.  **Breaking Reference Cycles:** Using `WeakKeyDictionary` alone was insufficient because the values in the dictionary (the `CachedFetcher` instances) held strong references back to the key (the `AsyncSubstrateInterface` instance) via bound methods. We introduced `WeakMethodCallable`, a wrapper that holds a weak reference to the bound method, breaking the reference cycle.

### Implementation
- **Patch File:** `bittensor/utils/async_substrate_interface_patch.py` contains the patched classes and logic.
- **Entry Point:** `bittensor/__init__.py` applies this patch immediately upon import, ensuring all usages of `AsyncSubstrateInterface` within `bittensor` (and user code importing `bittensor`) are patched.

### Verification
- A reproduction script confirmed the leak (100 instances retained after 100 iterations).
- The patch was verified to reduce the retained instances to 1 (the last loop variable), eliminating the leak.
- Existing unit tests passed, ensuring no regression in functionality.

## Recommendations
This patch is a temporary fix within `bittensor`. The issue should be reported to the `async-substrate-interface` repository for a permanent upstream fix. Once fixed upstream and the dependency version bumped, this patch can be removed.
