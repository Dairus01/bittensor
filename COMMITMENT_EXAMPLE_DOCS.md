# Documentation Update: Commitment Data Example

## 1. The Problem
The `get_commitment` method in `bittensor/core/subtensor.py` contained a placeholder comment in its docstring:
`# TODO: add a real example of how to handle realistic commitment data, or chop example`

This was insufficient for developers trying to understand how to effectively use the commitment mechanism, particularly regarding data formatting.

## 2. Thought Process
Commitment data on the Bittensor blockchain is stored as a raw string (or bytes). However, "realistic" usage almost always involves structured data, such as:
- Model versioning strings (e.g., "1.0.4")
- IPFS hashes (e.g., "Qm...")
- Timestamps
- Hugging Face repository URLs

To make the example useful, it needed to demonstrate:
1.  **Structuring**: Creating a Python dictionary with relevant fields.
2.  **Serialization**: Converting that dictionary to a string (JSON is the de facto standard) for storage.
3.  **Interaction**: How to pass this string to `set_commitment`.
4.  **Retrieval & Deserialization**: How to get the string back via `get_commitment` and parse it.

## 3. The Solution
I updated the docstring of `subtensor.get_commitment` to include a complete, runnable example.

### The Added Code
```python
Example:
    import bittensor as bt
    import json

    wallet = bt.Wallet()
    subtensor = bt.Subtensor(network="test")
    netuid = 1

    # Realistic commitment data often includes versioning and content hashes
    commitment_data = {
        "version": "1.0.0",
        "model_hash": "QmX...", # IPFS hash or similar
        "timestamp": 1234567890
    }

    # Convert to string for storage
    data_str = json.dumps(commitment_data)

    # Set commitment
    # Note: This requires the wallet to be a registered neuron on the subnet
    subtensor.set_commitment(wallet=wallet, netuid=netuid, data=data_str)

    # Retrieve commitment
    uid = 123
    retrieved_data_str = subtensor.get_commitment(netuid=netuid, uid=uid)

    if retrieved_data_str:
        try:
            # Parse back to dictionary
            retrieved_data = json.loads(retrieved_data_str)
            print(f"Retrieved version: {retrieved_data.get('version')}")
        except json.JSONDecodeError:
            print("Failed to decode commitment data")
```

This ensures developers have a clear pattern to follow for handling metadata on the chain.
