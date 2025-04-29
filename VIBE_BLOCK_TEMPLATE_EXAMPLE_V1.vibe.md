---
BlockID: INPUT_HANDLING_V1
Version: 1.0.0
Status: Definition
Owner: Selfy Core Team
---

# Vibe Block: Input Handling (User Chat Pipeline V1)

## Purpose
<!-- REQUIRED: 1-2 concise sentences explaining the block's core responsibility and reason for existence. What problem does it solve? -->
Performs initial processing of user text input for the User Chat Pipeline. Validates input (non-empty, length), retrieves recent conversation history from memory, stores the current input turn to memory, and packages the validated input and history for the next pipeline stage (Context Engine).

## LLMSummary
<!-- REQUIRED: A *very* brief (max 10 words) summary for high-level architectural views. -->
Validates user input, manages conversation history via memory.

## Interface

### Provides
<!-- REQUIRED: What this block offers to others. List functions, methods, API endpoints, events emitted, data published. Include signatures, key parameters, and return types. -->

- **Class:** `InputProcessor`
  - Description: Encapsulates the logic for handling user input. An instance of this class will likely be created and used by the `Main Pipeline (2.5)`.
  - **Methods:**
    - `__init__(self)`: Constructor. May load configuration values needed for processing.
    - `process_input(self, raw_input: str, user_id: str, session_id: str) -> ProcessedInput`
      - Description: The main method that takes raw user text and session identifiers, performs V1 validation, interacts with the memory system for history and storage, and returns a structured output.
      - Parameters:
        - `raw_input`: (str) The raw text input received from the user.
        - `user_id`: (str) Identifier for the user.
        - `session_id`: (str) Identifier for the current conversation session.
      - Returns: A `ProcessedInput` data structure containing the results.
      - Raises:
        - `InputValidationError` (Custom Exception): If the input fails validation checks (empty, too long).
        - Exceptions originating from `MEMORY_CORE_V1` during history retrieval or storage.

- **Data Structure:** `ProcessedInput` (dataclass, defined likely within this block's implementation or a shared pipeline types module)
  - Description: Standardized output structure for this block.
  - Fields:
    - `validated_input`: (str) The user input text after V1 validation and preprocessing (whitespace trimming).
    - `conversation_history`: (List[MemoryItem]) The recent conversation history fetched from `MEMORY_CORE_V1` (structure defined in `MEMORY_INTERFACE_V1`).
    - `input_metadata`: (Dict[str, Any]) Dictionary containing key context identifiers passed into `process_input`, e.g., `{'user_id': str, 'session_id': str}`.

- **Custom Exception:** `InputValidationError` (inherits from `Exception`)
  - Description: Raised when input fails validation rules. Carries an informative message.

### Consumes
<!-- REQUIRED (even if empty): What this block *needs* from others. List functions/methods called, APIs consumed, events subscribed to, required data structures. -->

- **Vibe Block:** `CONFIG_SYSTEM_V1`
  - Description: Reads configuration settings for input handling rules.
  - **Expected Configuration Keys (Examples via `config_get`):**
    - `pipeline.input_handling.max_input_length` (int, Default: 2048): Maximum allowed characters for user input.
    - `pipeline.input_handling.history.max_turns` (int, Default: 10): Number of recent conversation turns (`last_n`) to fetch from memory.
- **Vibe Block:** `LOGGING_SYSTEM_V1`
  - Description: Logs input processing steps, validation results (success/failure), memory interactions, and errors. Uses `get_logger(__name__)`.
- **Vibe Block:** `MEMORY_CORE_V1` (via `UnifiedMemorySystem` interface from `MEMORY_INTERFACE_V1`)
  - Description: Interacts with the memory system.
  - **Methods Called:**
    - `get_recent_conversation(session_id: str, last_n: int)`: To retrieve history.
    - `add_memory(item: MemoryItem)`: To store the current validated user input turn.
- **Vibe Block:** `MEMORY_INTERFACE_V1`
  - Description: Uses the `MemoryItem` data structure definition for creating and consuming conversation turns.

## Dependencies
<!-- REQUIRED: List the BlockIDs of other Vibe Blocks this one directly interacts with (based on the 'Consumes' section). -->
- `CONFIG_SYSTEM_V1`
- `LOGGING_SYSTEM_V1`
- `MEMORY_CORE_V1`
- `MEMORY_INTERFACE_V1`

## State Management
<!-- Optional: Describe how this block manages state. Is it stateless? Does it use a database, cache, in-memory store? -->
Largely stateless itself. Relies on `MEMORY_CORE_V1` for persistent conversation state. May hold configuration values loaded during initialization.

## Key Abstractions & Patterns
<!-- Optional: Internal design notes. What major classes, patterns (e.g., Strategy, Factory), or concepts are used *inside* the block that might be useful context? -->
- **Pipeline Stage:** Acts as the first functional stage in the User Chat Pipeline.
- **Data Transfer Object:** Uses `ProcessedInput` dataclass for standardized output.
- **Validation Logic:** Implements basic input validation rules.
- **Memory Interaction Facade:** Encapsulates the specific memory calls needed for conversation turn management (get history, add current).

## Constraints & Assumptions
<!-- Optional: Important limitations, performance characteristics, security considerations, environmental assumptions. -->
- **Initialization Order:** Assumes `CONFIG_SYSTEM_V1`, `LOGGING_SYSTEM_V1`, and `MEMORY_CORE_V1` (and its sub-components) have been successfully initialized before an `InputProcessor` instance is created or used.
- **V1 Scope:**
    - Validation is limited to non-empty and max length checks. No complex format or content validation.
    - Preprocessing is limited to whitespace trimming. No case normalization, command handling, or advanced sanitization.
    - Context Enrichment is limited to adding basic metadata (`timestamp`, `user_id`, `session_id`, `role`) to the `MemoryItem` before storage. No entity/intent extraction.
- **Error Handling:** Relies on raising `InputValidationError` for failed checks. Assumes callers (the main pipeline) will handle this exception appropriately. Errors from memory operations are expected to propagate.

## Usage Notes & Examples
<!-- Optional: How should other blocks interact with this one? Any initialization steps? Code snippets? -->
```python
# --- Conceptual Usage (within Main Pipeline - Block 2.5) ---
from selfy_agent.selfy_dev.user_pipeline.input_handling import InputProcessor, ProcessedInput, InputValidationError
from selfy_agent.selfy_dev.global_modules.logging import get_logger
# Assume memory_system, config_system etc are initialized and available

logger = get_logger('main_pipeline')

try:
    # Instantiate the processor (likely once during pipeline setup)
    input_processor = InputProcessor()

    # --- Inside the main loop processing user input ---
    raw_user_text = get_input_from_user_interface() # Function to get raw text
    current_user_id = "user_abc" # Get from session/auth
    current_session_id = "session_xyz" # Get from session

    try:
        processed_data: ProcessedInput = input_processor.process_input(
            raw_input=raw_user_text,
            user_id=current_user_id,
            session_id=current_session_id
        )

        logger.info("Input processed successfully.")
        # Pass processed_data to the next stage (Context Engine - Block 2.2)
        # context_engine.process(processed_data)

    except InputValidationError as ive:
        logger.warning(f"Input validation failed: {ive}")
        # Send feedback to user: "Input cannot be empty / is too long."
        send_error_to_user_interface(str(ive))
    except Exception as e:
        # Handle errors from memory system or other unexpected issues
        logger.error(f"Error during input processing stage: {e}", exc_info=True)
        send_error_to_user_interface("Sorry, there was an internal error processing your input.")

except Exception as setup_error:
    logger.critical(f"Failed to initialize InputProcessor: {setup_error}", exc_info=True)
    # Handle critical pipeline setup failure

# --- Example Configuration (in config.yaml or similar) ---
# pipeline:
#   input_handling:
#     max_input_length: 4096 # Allow longer inputs
#     history:
#       max_turns: 20 # Fetch more history