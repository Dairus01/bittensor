# Pull Request: Clarify return ordering and units for `get_revealed_commitment_by_hotkey`

## Problem
The `get_revealed_commitment_by_hotkey` method in `bittensor/core/subtensor.py` contained a TODO comment indicating that the return ordering and units needed clarification and that usage examples were missing. The existing type hint `Optional[tuple[tuple[int, str], ...]]` was vague about what the `int` and `str` represented (e.g., block number vs. commitment message).

## Thought Process
1.  **Code Analysis**: I examined the implementation of `get_revealed_commitment_by_hotkey`. It calls `self.query_module` and then maps the result using `decode_revealed_commitment`.
2.  **Dependency Analysis**: I looked at `bittensor/core/chain_data/utils.py` to find `decode_revealed_commitment`. It returns a tuple `(revealed_block, revealed_commitment)`.
3.  **Verification**: To confirm my understanding, I wrote a unit test that mocks the low-level blockchain query response. This verified that the method indeed returns a tuple of tuples where the first element is the block number (int) and the second is the commitment message (str).
4.  **Documentation Strategy**: I decided to update the docstring to explicitly describe this structure `((reveal_block, commitment_message), ...)` and add a Python example demonstrating a typical return value.
5.  **Testing Strategy**: I added unit tests to `tests/unit_tests/test_subtensor.py` to enforce this contract. This ensures that if the underlying decoding logic changes in the future, the tests will fail, signaling a need to update the documentation.

## Solution
1.  **Updated Docstring**: Modified `bittensor/core/subtensor.py` to remove the TODO. Added a detailed description of the return tuple and a usage example:
    ```python
    Example:
        >>> subtensor.get_revealed_commitment_by_hotkey(netuid=1, hotkey_ss58="5C4hr...")
        ((123, "commitment_string_1"), (150, "commitment_string_2"))
    ```
2.  **Added Unit Tests**: Created `tests/unit_tests/test_subtensor.py` (appended new tests) covering:
    *   `test_get_revealed_commitment_by_hotkey_success`: Verifies correct mapping of block number and message.
    *   `test_get_revealed_commitment_by_hotkey_invalid_address`: Verifies error handling.
    *   `test_get_revealed_commitment_by_hotkey_none`: Verifies handling when no data is found.
