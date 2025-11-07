# FastAPI Dependency Injection Issues in MCP Server

## Executive Summary

The agent-memory MCP server (`mcp.py`) is importing and calling FastAPI endpoint functions from `api.py` that have `Depends()` decorators. These decorators fail silently when called outside of a FastAPI request context, causing:

1. **Silent failures** - No error messages, but operations don't work
2. **Memory creation succeeds but memories never indexed** - Returns `{"status": "ok"}` but nothing happens
3. **Search returns empty results** - Always returns `{"total": 0, "memories": []}`

## Root Cause Analysis

### Problem Pattern

The MCP server currently imports FastAPI endpoint functions directly:

```python
# mcp.py - WRONG APPROACH
from agent_memory_server.api import (
    create_long_term_memory as core_create_long_term_memory,
    delete_long_term_memory as core_delete_long_term_memory,
    get_long_term_memory as core_get_long_term_memory,
    get_working_memory as core_get_working_memory,
    memory_prompt as core_memory_prompt,
    put_working_memory as core_put_working_memory,
    search_long_term_memory as core_search_long_term_memory,
    update_long_term_memory as core_update_long_term_memory,
)
```

These FastAPI functions have `Depends()` decorators that **only work in HTTP request context**:

```python
# api.py - These functions have Depends() that break in MCP context
async def create_long_term_memory(
    payload: CreateMemoryRecordRequest,
    background_tasks: HybridBackgroundTasks,  # <-- This is OK
    current_user: UserInfo = Depends(get_current_user),  # <-- This BREAKS in MCP
):
    ...

async def get_working_memory(
    session_id: str,
    user_id: str | None = None,
    namespace: str | None = None,
    model_name: ModelNameLiteral | None = None,
    context_window_max: int | None = None,
    recent_messages_limit: int | None = None,
    x_client_version: str | None = Header(None, alias="X-Client-Version"),  # <-- BREAKS
    current_user: UserInfo = Depends(get_current_user),  # <-- BREAKS
):
    ...

async def put_working_memory(
    session_id: str,
    memory: UpdateWorkingMemory,
    background_tasks: HybridBackgroundTasks,
    model_name: ModelNameLiteral | None = None,
    context_window_max: int | None = None,
    current_user: UserInfo = Depends(get_current_user),  # <-- BREAKS
):
    ...

async def memory_prompt(
    params: MemoryPromptRequest,
    background_tasks: HybridBackgroundTasks,
    optimize_query: bool = False,
    current_user: UserInfo = Depends(get_current_user),  # <-- BREAKS
) -> MemoryPromptResponse:
    ...
```

## Functions That Need Fixing

Based on the troubleshooting guide, **ALL** MCP tools that call FastAPI endpoints need to be fixed. Here's the status:

### ✅ Already Fixed (by me)

1. **`search_long_term_memory`** - Now calls `ltm_module.search_long_term_memories()` directly
2. **`create_long_term_memories`** - Now calls `ltm_module.index_long_term_memories()` directly
3. **`get_long_term_memory`** - Now calls `ltm_module.get_long_term_memory_by_id()` directly
4. **`edit_long_term_memory`** - Now calls `ltm_module.update_long_term_memory()` directly
5. **`delete_long_term_memories`** - Now calls `ltm_module.delete_long_term_memories()` directly
6. **`get_working_memory`** - Now calls `wm_module.get_working_memory()` directly
7. **`set_working_memory`** - Now calls `wm_module.set_working_memory()` directly

### ❓ Still Need to Fix

8. **`memory_prompt`** - **COMPLEX - needs detailed analysis**

## The `memory_prompt` Complexity

The `memory_prompt` MCP tool is the most complex case. Let me break down what it does:

### Current Implementation (Problematic)

```python
# mcp.py (current - lines 555-717)
@mcp_app.tool()
async def memory_prompt(
    query: str,
    session_id: SessionId | None = None,
    namespace: Namespace | None = None,
    model_name: ModelNameLiteral | None = None,
    context_window_max: int | None = None,
    topics: Topics | None = None,
    entities: Entities | None = None,
    created_at: CreatedAt | None = None,
    last_accessed: LastAccessed | None = None,
    user_id: UserId | None = None,
    memory_type: MemoryType | None = None,
    distance_threshold: float | None = None,
    limit: int = 10,
    offset: int = 0,
    optimize_query: bool = False,
) -> MemoryPromptResponse:
    # ... builds MemoryPromptRequest ...

    # Calls FastAPI endpoint (HAS Depends() - BREAKS!)
    return await core_memory_prompt(
        params=MemoryPromptRequest(query=query, **_params),
        background_tasks=HybridBackgroundTasks(),
        optimize_query=optimize_query,
    )
```

### What the API Function Does (api.py:855-1000+)

The `memory_prompt` API function is approximately **145+ lines** of complex logic:

1. **Validates inputs** - Requires either session or long_term_search
2. **Gets Redis connection** - `redis = await get_redis_conn()`
3. **If session provided**:
   - Calculates effective token limit based on model_name or context_window_max
   - Calls `working_memory.get_working_memory()` to get session history
   - Creates empty WorkingMemory if session doesn't exist
   - Adds summary as SystemMessage if present
   - **Token-based truncation** - Removes oldest messages until under limit
   - Converts messages to MCP message format (SystemMessage, UserMessage)
4. **If long_term_search provided**:
   - Extracts filters from SearchRequest
   - Calls `long_term_memory.search_long_term_memories()` with all filters
   - Converts results to SystemMessage with markdown formatting
   - Updates `last_accessed` timestamp for retrieved memories
5. **Adds the user's query** as final UserMessage
6. **Returns MemoryPromptResponse** with messages array

### Key Challenges

#### Challenge 1: Token-Based Truncation Logic

The API function has complex token calculation logic:

```python
# api.py:894-902
if params.session.model_name or params.session.context_window_max:
    token_limit = _get_effective_token_limit(
        model_name=params.session.model_name,
        context_window_max=params.session.context_window_max,
    )
    effective_token_limit = token_limit
else:
    # No model info provided - use all messages without truncation
    effective_token_limit = None
```

Then later:

```python
# api.py:936-956
if effective_token_limit is not None:
    # Token-based truncation
    if (
        _calculate_messages_token_count(working_mem.messages)
        > effective_token_limit
    ):
        # Keep removing oldest messages until we're under the limit
        recent_messages = working_mem.messages[:]
        while len(recent_messages) > 1:  # Always keep at least 1 message
            recent_messages = recent_messages[1:]  # Remove oldest
            if (
                _calculate_messages_token_count(recent_messages)
                <= effective_token_limit
            ):
                break
    else:
        recent_messages = working_mem.messages
else:
    # No token limit provided - use all messages
    recent_messages = working_mem.messages
```

**Question**: Should MCP tools implement this truncation logic, or should clients handle it?

#### Challenge 2: MCP Message Format Conversion

The API function converts between different message formats:

```python
# api.py:958-988
for msg in recent_messages:
    if msg.role == "user":
        _messages.append(
            UserMessage(
                content=TextContent(type="text", text=msg.content),
            )
        )
    elif msg.role == "assistant":
        _messages.append(
            AssistantMessage(
                content=TextContent(type="text", text=msg.content),
            )
        )
    elif msg.role == "system":
        _messages.append(
            SystemMessage(
                content=TextContent(type="text", text=msg.content),
            )
        )
```

These types (`UserMessage`, `AssistantMessage`, `SystemMessage`, `TextContent`) are imported from:

```python
from mcp.types import (
    AssistantMessage,
    EmbeddedResource,
    ImageContent,
    Resource,
    SystemMessage,
    TextContent,
    UserMessage,
)
```

**Question**: Where are these types defined? Are they part of the MCP protocol spec?

#### Challenge 3: Long-term Memory Retrieval and Formatting

```python
# api.py:991-1023
if params.long_term_search:
    # Extract filter objects from the payload
    filters = params.long_term_search.get_filters()

    logger.debug(f"Long-term search filters: {filters}")

    kwargs = {
        "distance_threshold": params.long_term_search.distance_threshold,
        "limit": params.long_term_search.limit,
        "offset": params.long_term_search.offset,
        "optimize_query": optimize_query,
        **filters,
        "text": params.long_term_search.text or "",
    }

    # Perform semantic search
    search_results = await long_term_memory.search_long_term_memories(**kwargs)

    if search_results.total > 0:
        # Format long-term memories as a system message
        memory_context = "## Relevant memories from long-term storage:\n\n"
        for memory in search_results.memories:
            memory_context += f"- {memory.text}\n"

        _messages.insert(
            0,
            SystemMessage(
                content=TextContent(type="text", text=memory_context),
            ),
        )

        # Update last_accessed for these memories
        background_tasks.add_task(
            long_term_memory.update_last_accessed,
            memory_ids=[m.id for m in search_results.memories],
        )
```

**Good news**: I already fixed `search_long_term_memories()` to work without Depends(), so this part should be straightforward.

#### Challenge 4: Final Query Message

```python
# api.py:1025-1027
_messages.append(
    UserMessage(content=TextContent(type="text", text=params.query))
)
```

Simple enough.

### Helper Functions Used by memory_prompt

The API function calls several helper functions:

1. **`_get_effective_token_limit(model_name, context_window_max)`** (api.py:~250-280)
   - Returns token limit based on model or direct value
   - Needs access to `MODEL_CONFIGS` from config

2. **`_calculate_messages_token_count(messages)`** (api.py:~220-230)
   - Counts tokens in message list
   - Uses tiktoken or similar

3. **`working_memory.get_working_memory()`** (working_memory.py:72)
   - Already have this as `wm_module.get_working_memory()` ✅

4. **`long_term_memory.search_long_term_memories()`** (long_term_memory.py:847)
   - Already have this as `ltm_module.search_long_term_memories()` ✅

5. **`long_term_memory.update_last_accessed()`** (long_term_memory.py:1662)
   - Need to verify this exists and has correct signature

### Proposed Solutions

#### Option 1: Copy Entire Logic to MCP Tool (Current Approach)

**Pros**:
- No dependency on FastAPI endpoints
- Full control over behavior
- Can simplify (skip token truncation if clients handle it)

**Cons**:
- Code duplication (~150 lines)
- Need to maintain two implementations
- Risk of divergence

#### Option 2: Extract Core Logic to Shared Module

**Pros**:
- DRY (Don't Repeat Yourself)
- Single source of truth
- Both API and MCP use same logic

**Cons**:
- Requires refactoring `api.py`
- More architectural changes
- Risk of breaking existing API

#### Option 3: Minimal MCP Implementation

Skip complex features that MCP clients might not need:

```python
@mcp_app.tool()
async def memory_prompt(
    query: str,
    session_id: SessionId | None = None,
    namespace: Namespace | None = None,
    # ... other params ...
) -> MemoryPromptResponse:
    """Simplified version - clients handle token management"""
    from agent_memory_server.utils.redis import get_redis_conn
    redis = await get_redis_conn()

    _messages = []

    # 1. Get working memory if session provided (no token truncation)
    if session_id:
        _session_id = session_id.eq if session_id and session_id.eq else None
        if _session_id:
            working_mem = await wm_module.get_working_memory(
                session_id=_session_id,
                redis_client=redis,
            )
            if working_mem and working_mem.messages:
                # Convert all messages without truncation
                for msg in working_mem.messages:
                    # Convert to MCP message format
                    ...

    # 2. Get long-term memories if search provided
    if long_term_search:
        search_results = await ltm_module.search_long_term_memories(...)
        # Format as system message
        ...

    # 3. Add query
    _messages.append(UserMessage(content=TextContent(type="text", text=query)))

    return MemoryPromptResponse(messages=_messages)
```

**Pros**:
- Simpler, less code duplication
- MCP clients can handle token management themselves
- Still provides core functionality

**Cons**:
- Different behavior from API version
- Might not meet all use cases

## Specific Questions for Other Claude Instance

### Question 1: Token Truncation Strategy

The API version has sophisticated token-based truncation logic. Should the MCP version:

A. **Implement the same logic** (copy ~50 lines of token calculation code)?
B. **Skip truncation** and let MCP clients handle it?
C. **Implement simplified truncation** (e.g., last N messages only)?

### Question 2: Message Format Conversion

The MCP types (`UserMessage`, `AssistantMessage`, `SystemMessage`, `TextContent`) are imported from `mcp.types`.

- Are these standard MCP protocol types?
- Should I use them directly or convert to a simpler format?
- The current code seems to work with them - is this the right approach?

### Question 3: Helper Function Access

The API uses helper functions like:
- `_get_effective_token_limit()`
- `_calculate_messages_token_count()`

These are defined in `api.py` with leading underscores (private). Should I:

A. **Import them from api.py** (but they're private)?
B. **Copy them to mcp.py** (code duplication)?
C. **Extract to shared utils module** (requires refactoring)?

### Question 4: Background Tasks for update_last_accessed

The API schedules a background task to update `last_accessed` timestamps:

```python
background_tasks.add_task(
    long_term_memory.update_last_accessed,
    memory_ids=[m.id for m in search_results.memories],
)
```

Should the MCP version:

A. **Do the same** (use `get_background_tasks()` and schedule task)?
B. **Skip it** (not critical for MCP clients)?
C. **Call synchronously** (might slow down response)?

### Question 5: WorkingMemory Creation on Empty Session

The API creates an empty `WorkingMemory` object if session doesn't exist:

```python
# api.py:913-923
if not working_mem:
    working_mem = WorkingMemory(
        session_id=params.session.session_id,
        namespace=params.session.namespace,
        user_id=params.session.user_id,
        messages=[],
        memories=[],
    )
```

Should MCP version do the same, or return an error if session doesn't exist?

## Current Implementation Status

### What I've Fixed So Far

```python
# mcp.py - Added these imports
from agent_memory_server import long_term_memory as ltm_module
from agent_memory_server import working_memory as wm_module
```

### Fixed Functions

1. ✅ `search_long_term_memory` - Calls `ltm_module.search_long_term_memories()` directly
2. ✅ `create_long_term_memories` - Calls `ltm_module.index_long_term_memories()` via background tasks
3. ✅ `get_long_term_memory` - Calls `ltm_module.get_long_term_memory_by_id()` directly
4. ✅ `edit_long_term_memory` - Calls `ltm_module.update_long_term_memory()` directly
5. ✅ `delete_long_term_memories` - Calls `ltm_module.delete_long_term_memories()` directly
6. ✅ `get_working_memory` - Calls `wm_module.get_working_memory()` directly
7. ✅ `set_working_memory` - Calls `wm_module.set_working_memory()` and schedules background promotion

### Still Calling FastAPI Endpoints (BROKEN)

8. ❌ `memory_prompt` - Still calls `core_memory_prompt()` from api.py (lines 713-717)

## Code References

### Files Modified

- `/home/eddygk/mcp/agent-memory-server/agent_memory_server/mcp.py`
  - Added imports: lines 19-20
  - Fixed `create_long_term_memories`: lines 379-398
  - Fixed `search_long_term_memory`: lines 511-547
  - Fixed `get_working_memory`: lines 921-938
  - Fixed `set_working_memory`: lines 881-930
  - Fixed `get_long_term_memory`: lines 949-956
  - Fixed `edit_long_term_memory`: lines 1082-1092
  - Fixed `delete_long_term_memories`: lines 1121-1126
  - **STILL BROKEN** `memory_prompt`: lines 555-717 (calls `core_memory_prompt`)

### Key API File Locations

- `api.py:855-1000+` - The `memory_prompt()` function I need to replicate
- `api.py:~250-280` - `_get_effective_token_limit()` helper
- `api.py:~220-230` - `_calculate_messages_token_count()` helper
- `working_memory.py:72` - `get_working_memory()` core function (already using)
- `long_term_memory.py:847` - `search_long_term_memories()` core function (already using)
- `long_term_memory.py:1662` - `update_last_accessed()` function (need to verify)

## Testing Required

Once `memory_prompt` is fixed, we need to verify:

1. **Memory creation works** - Test `create_long_term_memories` returns success AND memory appears in Redis
2. **Memory search works** - Test `search_long_term_memory` returns actual results (not empty)
3. **Memory prompt works** - Test `memory_prompt` retrieves and formats context correctly
4. **Background tasks execute** - Verify indexing and promotion tasks actually run
5. **Working memory works** - Test `set_working_memory` and `get_working_memory` flow

### Diagnostic Commands

From the troubleshooting guide:

```bash
# Test if memory actually stored
docker-compose exec redis redis-cli KEYS "memory_idx:*"

# Test search function directly
docker-compose exec mcp-stdio python3 << 'EOF'
import asyncio
from agent_memory_server import long_term_memory

async def test():
    results = await long_term_memory.search_long_term_memories(
        text="test query",
        limit=10
    )
    print(f"Results: {results.total} memories found")

asyncio.run(test())
EOF
```

## Next Steps

1. **Get guidance on memory_prompt implementation** - Which option (1, 2, or 3)?
2. **Implement the fix** based on guidance
3. **Rebuild Docker containers** - `docker-compose up -d --build mcp-stdio`
4. **Run end-to-end test** - Create memory, search for it, verify it's found
5. **Test with actual Claude Desktop** - User's Windows setup

## Environment Context

- **OS**: Windows 11 with WSL2 (Ubuntu)
- **Docker**: Docker Desktop with WSL integration
- **Redis**: Port 16380 (custom port)
- **Python**: 3.12 (required, not 3.13)
- **MCP Mode**: stdio (not SSE)
- **Container**: `agent-memory-server-mcp-stdio-1`

## References

- Troubleshooting guide provided by user (comprehensive 400+ line document)
- Previous Claude instance successfully fixed this on macOS
- User has working MacBook config using Docker exec approach
