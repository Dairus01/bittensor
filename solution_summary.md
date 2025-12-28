# Solution Summary: Sending Extrinsic Flow Example

## Problem Description
The documentation for the `sign_and_send_extrinsic` method in `bittensor/core/async_subtensor.py` was missing a clear example. A TODO comment (`# TODO: Full clear example of sending extrinsic flow`) highlighted the need for demonstrating the full flow of composing, signing, and sending an extrinsic.

Without this example, users might find it difficult to understand how to correctly prepare a `GenericCall` object and use the `sign_and_send_extrinsic` method, especially in an asynchronous context.

## Thought Process
1.  **Analyze the Codebase**: I examined `bittensor/core/async_subtensor.py` to understand the `sign_and_send_extrinsic` method signature and its dependency on `GenericCall`.
2.  **Verify `compose_call` Availability**: I confirmed that `AsyncSubtensor` provides a helper method `compose_call` (lines 5841+) which simplifies the creation of `GenericCall` objects using on-chain metadata.
3.  **Construct the Example**:
    *   **Setup**: Initialize `AsyncSubtensor` and `Wallet`.
    *   **Compose**: Use `await subtensor.compose_call(...)` to create the call object. This is a crucial step as it handles metadata and parameter validation.
    *   **Send**: Pass the composed call and the wallet to `sign_and_send_extrinsic`.
    *   **Handle Response**: Demonstrate how to check for success using `response.is_success` and handle potential errors.
4.  **Verification**: I ensured that the example code aligns with the actual method signatures and usage patterns within the `bittensor` library.

## Solution
I added a comprehensive example to the docstring of `sign_and_send_extrinsic` in `bittensor/core/async_subtensor.py`.

### Added Example Code:
```python
        Example:
            import asyncio
            import bittensor as bt

            async def main():
                # Setup
                wallet = bt.Wallet()
                subtensor = await bt.AsyncSubtensor(network="test").initialize()

                # Compose the call
                # For example, a transfer call
                call = await subtensor.compose_call(
                    call_module='Balances',
                    call_function='transfer_keep_alive',
                    call_params={
                        'dest': '5Dest...',
                        'value': 1000000
                    }
                )

                # Sign and send
                response = await subtensor.sign_and_send_extrinsic(
                    call=call,
                    wallet=wallet,
                    sign_with='coldkey', # signing with coldkey
                    wait_for_inclusion=True,
                    wait_for_finalization=True
                )

                if response.is_success:
                    print("Transaction success:", response.extrinsic_hash)
                else:
                    print("Transaction failed:", response.error_message)

            asyncio.run(main())
```

This addition directly addresses the TODO and provides users with a copy-pasteable template for interacting with the blockchain via extrinsics.
