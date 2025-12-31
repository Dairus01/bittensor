# Memory Leak Fix in `get_async_subtensor`

## Problem Description

A memory leak was identified in the `bittensor` library when using the `get_async_subtensor` factory function in a loop, specifically when the initialization step fails.

When `get_async_subtensor` is called, it performs two main steps:
1. Instantiates a new `AsyncSubtensor` object.
2. Awaits the `.initialize()` method on this object to establish a connection to the Substrate node.

If the `.initialize()` method raises an exception (e.g., due to a connection error, timeout, or invalid configuration), the exception propagates immediately to the caller. Crucially, the partially initialized `AsyncSubtensor` instance—which may have already allocated resources such as an `AsyncSubstrateInterface` with pending tasks or open (but failing) connections—is lost.

Because the exception interrupts the flow before the instance is returned, the caller never receives the `subtensor` object. Consequently, the caller cannot call `.close()` on it. This leaves the underlying resources (like the `aiohttp` session or background tasks within `async_substrate_interface`) hanging, leading to a memory leak that becomes significant if the operation is retried in a loop.

## Thought Process

1.  **Analysis of the Issue:**
    -   The issue report highlighted a leak when `get_async_subtensor` is called repeatedly.
    -   I examined the code for `get_async_subtensor` in `bittensor/core/async_subtensor.py`.
    -   Code before fix:
        ```python
        sub = AsyncSubtensor(network=network, ...)
        await sub.initialize()  # <--- If this fails, 'sub' is lost
        return sub
        ```
    -   If `sub.initialize()` fails, the `sub` variable goes out of scope, but if `initialize` started async tasks that reference `self`, the object might be kept alive by the event loop or other references, preventing garbage collection. Explicit cleanup via `.close()` is required for `AsyncSubtensor`.

2.  **Reproduction:**
    -   I created a reproduction script that mocked `AsyncSubstrateInterface` to simulate initialization failures.
    -   I observed that when `initialize` raises an exception, the `close` method of the underlying interface was *not* called.
    -   I confirmed that in a loop, this leads to an accumulation of unclosed objects.

3.  **Verification of Synchronous Implementation:**
    -   I checked `bittensor/core/subtensor.py`. The synchronous `Subtensor` class is typically instantiated directly (`Subtensor(...)`). Its initialization happens in `__init__`, and if that fails, the object is generally not created or fully formed. More importantly, there isn't a factory function performing a two-step "create then async init" process that hides the instance on failure. Thus, the issue appeared specific to the async factory pattern.

## Solution

The solution is to wrap the initialization step in a `try...except` block within the `get_async_subtensor` function. This ensures that if initialization fails, we explicitly close the `AsyncSubtensor` instance before allowing the exception to propagate.

**Updated Code:**

```python
async def get_async_subtensor(
    network: Optional[str] = None,
    config: Optional["Config"] = None,
    mock: bool = False,
    log_verbose: bool = False,
) -> "AsyncSubtensor":
    # ... (instantiation)
    sub = AsyncSubtensor(
        network=network, config=config, mock=mock, log_verbose=log_verbose
    )

    # Wrap initialization to ensure cleanup on failure
    try:
        await sub.initialize()
        return sub
    except Exception as e:
        # If initialization fails, close the instance to free resources
        await sub.close()
        raise e
```

By adding this error handling, we guarantee that any `AsyncSubtensor` created by this factory is either successfully initialized and returned, or properly closed and discarded. This prevents the resource leak even when connection errors occur repeatedly.
