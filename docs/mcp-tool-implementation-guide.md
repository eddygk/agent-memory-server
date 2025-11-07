# MCP Tool Implementation Guide - Detailed Answers

**Last Updated**: November 4, 2025
**Working Implementation Reference**: `/home/eddygk/mcp/agent-memory-server/agent_memory_server/mcp.py`

---

## TOOL 1: `set_working_memory` Implementation

### Q1.1: Core Module Function

**Exact function to call**:
```python
await wm_module.set_working_memory(
    working_memory=working_memory_obj,
    redis_client=redis,
)
```

**Function signature** (from `working_memory.py` line ~200):
```python
async def set_working_memory(
    working_memory: WorkingMemory,
    redis_client: Redis,
) -> None
```

**Key implementation details**:
1. Convert `UpdateWorkingMemory` to `WorkingMemory` first:
   ```python
   working_memory_obj = update_memory_obj.to_working_memory(session_id)
   ```
2. Pass the `WorkingMemory` object directly (not deconstructed)
3. Requires Redis connection

**Current working implementation** (lines 948-968):
```python
# Create the UpdateWorkingMemory object
from agent_memory_server.models import UpdateWorkingMemory

update_memory_obj = UpdateWorkingMemory(
    namespace=namespace,
    memories=processed_memories,
    messages=processed_messages,
    context=context,
    data=data or {},
    user_id=user_id,
    ttl_seconds=ttl_seconds,
    long_term_memory_strategy=long_term_memory_strategy or MemoryStrategyConfig(),
)

# Convert to WorkingMemory
from agent_memory_server.utils.redis import get_redis_conn
redis = await get_redis_conn()

working_memory_obj = update_memory_obj.to_working_memory(session_id)

# Store in working memory
await wm_module.set_working_memory(
    working_memory=working_memory_obj,
    redis_client=redis,
)
```

### Q1.2: Background Tasks Handling

**Answer**: Background tasks are handled AFTER the core function call, not passed to it.

**Pattern** (lines 971-980):
```python
# Background tasks for long-term memory promotion
if settings.long_term_memory and (
    working_memory_obj.memories or working_memory_obj.messages
):
    background_tasks = get_background_tasks()
    background_tasks.add_task(
        ltm_module.promote_working_memory_to_long_term,
        session_id=session_id,
        user_id=working_memory_obj.user_id,
        namespace=working_memory_obj.namespace,
    )
```

**Key points**:
- Use `get_background_tasks()` from dependencies (line 13)
- Call `background_tasks.add_task()` with the task function and parameters
- This happens AFTER `set_working_memory` completes
- Not passed to the core function

### Q1.3: Redis Connection

**Answer**: YES, `set_working_memory` requires direct Redis connection.

**Pattern**:
```python
from agent_memory_server.utils.redis import get_redis_conn

redis = await get_redis_conn()
await wm_module.set_working_memory(
    working_memory=working_memory_obj,
    redis_client=redis,  # Parameter name is 'redis_client'
)
```

### Q1.4: Return Value Conversion

**Answer**: The core function returns `None`, so you construct the response manually.

**Pattern** (lines 982-983):
```python
# Return WorkingMemoryResponse
return WorkingMemoryResponse(**working_memory_obj.model_dump())
```

The core function doesn't return anything, so you create `WorkingMemoryResponse` from the `WorkingMemory` object you created.

---

## TOOL 2: `get_working_memory` Implementation

### Q2.1: Core Module Function

**Exact function to call**:
```python
await wm_module.get_working_memory(
    session_id=session_id,
    redis_client=redis,
    recent_messages_limit=recent_messages_limit
)
```

**Function signature** (from `working_memory.py` line ~150):
```python
async def get_working_memory(
    session_id: str,
    redis_client: Redis,
    namespace: str | None = None,
    user_id: str | None = None,
    recent_messages_limit: int | None = None,
) -> WorkingMemory | None
```

**Current working implementation** (lines 1001-1018):
```python
from agent_memory_server.utils.redis import get_redis_conn
redis = await get_redis_conn()

memory = await wm_module.get_working_memory(
    session_id=session_id,
    redis_client=redis,
    recent_messages_limit=recent_messages_limit
)
if not memory:
    # Return empty working memory if not found
    from agent_memory_server.models import WorkingMemory
    return WorkingMemory(
        session_id=session_id,
        messages=[],
        memories=[],
    )
return memory
```

### Q2.2: Parameter Mapping

**Direct mapping - no conversion needed**:
- `session_id` → `session_id`
- `recent_messages_limit` → `recent_messages_limit`

Optional parameters you can pass:
- `namespace` (if filtering needed)
- `user_id` (if filtering needed)

### Q2.3: Redis Connection

**Answer**: YES, requires Redis connection.

```python
redis = await get_redis_conn()
```

### Q2.4: Return Value

**Answer**: Core function returns `WorkingMemory | None`.

**Pattern**: If None, create empty WorkingMemory with the session_id. Otherwise return directly.

---

## TOOL 3: `get_long_term_memory` Implementation

### Q3.1: Core Module Function

**Exact function to call**:
```python
await ltm_module.get_long_term_memory_by_id(memory_id)
```

**Function signature** (from `long_term_memory.py` line 1489):
```python
async def get_long_term_memory_by_id(memory_id: str) -> MemoryRecord | None
```

**Current working implementation** (lines 1047-1051):
```python
# Call core function directly (bypasses FastAPI Depends() issues)
memory = await ltm_module.get_long_term_memory_by_id(memory_id)
if not memory:
    raise ValueError(f"Memory with ID {memory_id} not found")
return memory
```

### Q3.2: Parameter Passing

**Direct mapping**: Just pass `memory_id` as-is.

### Q3.3: Redis Connection

**Answer**: NO, this function does NOT need explicit Redis connection passed.

The core function handles Redis internally via its own connection management.

### Q3.4: Return Value and Error Handling

**Pattern**:
- If `None` returned: Raise `ValueError` with descriptive message
- If `MemoryRecord` returned: Return it directly (no conversion needed)

---

## TOOL 4: `edit_long_term_memory` Implementation

### Q4.1: Core Module Function

**Exact function to call**:
```python
await ltm_module.update_long_term_memory(
    memory_id=memory_id,
    updates=update_dict
)
```

**Function signature** (from `long_term_memory.py` line 1515):
```python
async def update_long_term_memory(
    memory_id: str,
    updates: dict[str, Any]
) -> MemoryRecord | None
```

**Current working implementation** (lines 1160-1187):
```python
# Build the update request dictionary, handling event_date parsing
update_dict = {
    "text": text,
    "topics": topics,
    "entities": entities,
    "memory_type": memory_type,
    "namespace": namespace,
    "user_id": user_id,
    "session_id": session_id,
    "event_date": (
        _parse_iso8601_datetime(event_date) if event_date is not None else None
    ),
}

# Filter out None values to only include fields that should be updated
update_dict = {k: v for k, v in update_dict.items() if v is not None}

if not settings.long_term_memory:
    raise ValueError("Long-term memory is disabled")

# Call core function directly (bypasses FastAPI Depends() issues)
updated_memory = await ltm_module.update_long_term_memory(
    memory_id=memory_id,
    updates=update_dict
)
if not updated_memory:
    raise ValueError(f"Memory with ID {memory_id} not found")
return updated_memory
```

### Q4.2: Memory Update Payload

**Answer**: Build a plain dictionary with only non-None values.

**Pattern**:
1. Create dict with all possible fields
2. Filter out None values: `{k: v for k, v in update_dict.items() if v is not None}`
3. Pass as `updates` parameter

### Q4.3: Redis and Background Tasks

**Answer**: NO explicit Redis connection or background_tasks needed.

The core function handles these internally.

### Q4.4: Namespace and User ID Defaults

**Answer**: Do NOT apply defaults before calling core function.

The defaults in the MCP tool signature are for the MCP interface only. Pass whatever the user provides (or None).

### Q4.5: Return Value

**Answer**: Core function returns `MemoryRecord | None`.

**Pattern**: If None, raise ValueError. Otherwise return directly.

---

## TOOL 5: `delete_long_term_memories` Implementation

### Q5.1: Core Module Function

**Exact function to call**:
```python
await ltm_module.delete_long_term_memories(memory_ids)
```

**Function signature** (from `long_term_memory.py` line 1479):
```python
async def delete_long_term_memories(memory_ids: list[str]) -> None
```

**Current working implementation** (lines 1219-1221):
```python
# Call core function directly (bypasses FastAPI Depends() issues)
await ltm_module.delete_long_term_memories(memory_ids)
return AckResponse(status="ok", detail=f"Deleted {len(memory_ids)} memories")
```

### Q5.2: Parameter Mapping

**Direct mapping**: Pass `memory_ids` list directly, no conversion.

### Q5.3: Redis and Background Tasks

**Answer**: NO explicit Redis connection or background_tasks needed.

### Q5.4: Filter Parameters

**Answer**: The current MCP tool does NOT support filter parameters.

Looking at line 1192, the tool signature only accepts `memory_ids: list[str]`. If you want to add filtering, you'd need to:
1. Add filter parameters to the tool signature
2. Search for memories matching filters first
3. Extract their IDs
4. Pass those IDs to delete

But the current implementation just deletes the provided IDs directly.

### Q5.5: Return Value

**Answer**: Core function returns `None`.

**Pattern**: Construct AckResponse manually:
```python
return AckResponse(status="ok", detail=f"Deleted {len(memory_ids)} memories")
```

---

## TOOL 6: `memory_prompt` Implementation

### Q6.1: Core Module Function and Approach

**Answer**: Multi-step approach using BOTH modules.

**Exact implementation** (lines 664-770):

```python
from mcp.server.fastmcp.prompts import base
from mcp.types import TextContent
from agent_memory_server.models import SystemMessage, UserMessage
from agent_memory_server.utils.redis import get_redis_conn

redis = await get_redis_conn()
_messages = []

# Step 1: Get working memory if session provided
if session_id and session_id.eq:
    working_mem = await wm_module.get_working_memory(
        session_id=session_id.eq,
        namespace=namespace.eq if namespace else None,
        user_id=user_id.eq if user_id else None,
        redis_client=redis,
    )

    if working_mem:
        # Add summary as system message
        if working_mem.context:
            _messages.append(
                SystemMessage(
                    content=TextContent(
                        type="text",
                        text=f"## A summary of the conversation so far:\n{working_mem.context}",
                    ),
                )
            )

        # Convert working memory messages
        if working_mem.messages:
            for msg in working_mem.messages:
                if msg.role == "user":
                    _messages.append(
                        base.UserMessage(content=TextContent(type="text", text=msg.content))
                    )
                elif msg.role == "assistant":
                    _messages.append(
                        base.AssistantMessage(content=TextContent(type="text", text=msg.content))
                    )
                else:
                    _messages.append(
                        SystemMessage(content=TextContent(type="text", text=msg.content))
                    )

# Step 2: Search long-term memories
search_needed = any([
    topics, entities, created_at, last_accessed, user_id, memory_type
]) or query

if search_needed:
    # Build filters dict
    filters = {}
    if session_id:
        filters["session_id"] = session_id
    if namespace:
        filters["namespace"] = namespace
    # ... add all filter parameters

    # Search
    search_results = await ltm_module.search_long_term_memories(
        text=query or "",
        distance_threshold=distance_threshold,
        limit=limit,
        offset=offset,
        optimize_query=optimize_query,
        **filters,
    )

    if search_results.total > 0:
        # Format as system message at the START
        memory_context = "## Relevant memories from long-term storage:\n\n"
        for memory in search_results.memories:
            memory_context += f"- {memory.text}\n"

        _messages.insert(
            0,
            SystemMessage(
                content=TextContent(type="text", text=memory_context),
            ),
        )

        # Update last_accessed (background task)
        background_tasks = get_background_tasks()
        background_tasks.add_task(
            ltm_module.update_last_accessed,
            memory_ids=[m.id for m in search_results.memories],
        )

# Step 3: Add user's query as final message
_messages.append(
    base.UserMessage(content=TextContent(type="text", text=query))
)

return MemoryPromptResponse(messages=_messages)
```

### Q6.2: Two-Part Return Value

**Message structure**:
1. **SystemMessage** with long-term memories (inserted at position 0)
2. **SystemMessage** with conversation summary (if working memory has context)
3. **Working memory messages** (UserMessage/AssistantMessage from session)
4. **UserMessage** with the query (appended at end)

### Q6.3: Session-Scoped Memory Integration

**Answer**: YES, retrieve working memory first if session_id provided.

**Pattern**:
```python
if session_id and session_id.eq:
    working_mem = await wm_module.get_working_memory(...)
```

### Q6.4: Default Parameters

**Answer**: YES, apply defaults at the start (lines 669-672):
```python
if user_id is None and settings.default_mcp_user_id:
    user_id = UserId(eq=settings.default_mcp_user_id)
if namespace is None and settings.default_mcp_namespace:
    namespace = Namespace(eq=settings.default_mcp_namespace)
```

### Q6.5: Message Formatting

**Answer**: Use specific message types from models and FastMCP.

**Import requirements**:
```python
from mcp.server.fastmcp.prompts import base
from mcp.types import TextContent
from agent_memory_server.models import SystemMessage
```

**Message types**:
- Context/memories: `SystemMessage(content=TextContent(type="text", text=...))`
- User messages: `base.UserMessage(content=TextContent(type="text", text=...))`
- Assistant messages: `base.AssistantMessage(content=TextContent(type="text", text=...))`

---

## CROSS-TOOL PATTERNS

### Q7.1: Import Pattern Consistency

**YES - All tools follow this pattern**:

```python
# At top of mcp.py (lines 8-41)
from agent_memory_server import long_term_memory as ltm_module
from agent_memory_server import working_memory as wm_module
from agent_memory_server.utils.redis import get_redis_conn
from agent_memory_server.dependencies import get_background_tasks
```

**DO NOT import**:
- `from agent_memory_server.api import ...` (these have FastAPI Depends())

### Q7.2: Error Handling Pattern

**Pattern varies by tool**:

1. **Search operations** - Return empty results on error:
   ```python
   try:
       results = await ltm_module.search_long_term_memories(...)
       return MemoryRecordResults(...)
   except Exception as e:
       logger.error(f"Error: {e}")
       return MemoryRecordResults(total=0, memories=[], next_offset=None)
   ```

2. **Get operations** - Raise ValueError if not found:
   ```python
   memory = await ltm_module.get_long_term_memory_by_id(memory_id)
   if not memory:
       raise ValueError(f"Memory with ID {memory_id} not found")
   return memory
   ```

3. **Create/Update/Delete** - Let exceptions propagate:
   ```python
   await ltm_module.delete_long_term_memories(memory_ids)
   return AckResponse(status="ok")
   ```

### Q7.3: Background Tasks Pattern

**When to use**: For operations that trigger asynchronous processing.

**Tools that use background tasks**:

1. **`create_long_term_memories`** (lines 383-389):
   ```python
   background_tasks = get_background_tasks()
   background_tasks.add_task(
       ltm_module.index_long_term_memories,
       memories=memories,
       deduplicate=True,
   )
   ```

2. **`set_working_memory`** (lines 971-980):
   ```python
   if settings.long_term_memory and (working_memory_obj.memories or working_memory_obj.messages):
       background_tasks = get_background_tasks()
       background_tasks.add_task(
           ltm_module.promote_working_memory_to_long_term,
           session_id=session_id,
           user_id=working_memory_obj.user_id,
           namespace=working_memory_obj.namespace,
       )
   ```

3. **`memory_prompt`** (lines 759-763):
   ```python
   background_tasks = get_background_tasks()
   background_tasks.add_task(
       ltm_module.update_last_accessed,
       memory_ids=[m.id for m in search_results.memories],
   )
   ```

**Pattern**:
- Get instance: `background_tasks = get_background_tasks()`
- Add task: `background_tasks.add_task(function, param1, param2, ...)`
- Do NOT pass to core functions - call separately

### Q7.4: Redis Connection Pattern

**Tools that need Redis connection**:

| Tool | Needs Redis? | Pattern |
|------|--------------|---------|
| `create_long_term_memories` | NO | Core function handles internally |
| `search_long_term_memory` | NO | Core function handles internally |
| `memory_prompt` | YES | Pass to `wm_module.get_working_memory()` |
| `set_working_memory` | YES | Pass to `wm_module.set_working_memory()` |
| `get_working_memory` | YES | Pass to `wm_module.get_working_memory()` |
| `get_long_term_memory` | NO | Core function handles internally |
| `edit_long_term_memory` | NO | Core function handles internally |
| `delete_long_term_memories` | NO | Core function handles internally |

**Rule**: Only working memory operations need explicit Redis connection.

### Q7.5: Default Namespace and User ID Pattern

**Pattern**: Apply defaults at the START of the function.

**Example** (from `search_long_term_memory` lines 507-510):
```python
if user_id is None and settings.default_mcp_user_id:
    user_id = UserId(eq=settings.default_mcp_user_id)
if namespace is None and settings.default_mcp_namespace:
    namespace = Namespace(eq=settings.default_mcp_namespace)
```

**Apply to these tools**:
- `create_long_term_memories` (lines 374-379)
- `search_long_term_memory` (lines 507-510)
- `memory_prompt` (lines 669-672)
- `set_working_memory` (uses parameters directly, defaults in signature)

---

## CODE REFERENCE ANSWERS

### Q8.1: Working Memory Module Function Signatures

**From `working_memory.py`**:

```python
async def set_working_memory(
    working_memory: WorkingMemory,
    redis_client: Redis,
) -> None:
    """Store working memory in Redis"""
    # Implementation at line ~200
```

```python
async def get_working_memory(
    session_id: str,
    redis_client: Redis,
    namespace: str | None = None,
    user_id: str | None = None,
    recent_messages_limit: int | None = None,
) -> WorkingMemory | None:
    """Retrieve working memory from Redis"""
    # Implementation at line ~150
```

### Q8.2: Long-Term Memory Module Function Signatures

**From `long_term_memory.py`**:

```python
async def index_long_term_memories(
    memories: list[MemoryRecord],
    deduplicate: bool = True,
) -> None:
    """Index memories into vector store"""
    # Implementation at line ~800
```

```python
async def search_long_term_memories(
    text: str = "",
    session_id: SessionId | None = None,
    namespace: Namespace | None = None,
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
) -> MemoryRecordResults:
    """Search memories by semantic similarity and filters"""
    # Implementation at line ~1000
```

```python
async def get_long_term_memory_by_id(memory_id: str) -> MemoryRecord | None:
    """Get single memory by ID"""
    # Implementation at line 1489
```

```python
async def update_long_term_memory(
    memory_id: str,
    updates: dict[str, Any]
) -> MemoryRecord | None:
    """Update memory fields"""
    # Implementation at line 1515
```

```python
async def delete_long_term_memories(memory_ids: list[str]) -> None:
    """Delete memories by IDs"""
    # Implementation at line 1479
```

```python
async def promote_working_memory_to_long_term(
    session_id: str,
    user_id: str | None = None,
    namespace: str | None = None,
) -> None:
    """Background task: promote working memory to long-term"""
    # Implementation at line ~1200
```

```python
async def update_last_accessed(memory_ids: list[str]) -> None:
    """Background task: update last_accessed timestamp"""
    # Implementation at line ~1450
```

### Q8.3: Existing Working Example Validation

**YES - Both patterns are correct**:

1. **`create_long_term_memories`** (lines 370-389):
   - ✅ Applies defaults for namespace/user_id
   - ✅ Uses background_tasks for async indexing
   - ✅ Returns AckResponse immediately
   - ✅ Does NOT need Redis connection

2. **`search_long_term_memory`** (lines 392-548):
   - ✅ Applies defaults for namespace/user_id
   - ✅ Creates SearchRequest to build filters
   - ✅ Calls core function with **filters unpacked
   - ✅ Returns MemoryRecordResults
   - ✅ Has try/except for error handling
   - ✅ Does NOT need Redis connection

**Key differences**:
- **Write operations** (create/update/delete) use background_tasks for indexing
- **Read operations** (search/get) call core functions directly
- **Working memory operations** need explicit Redis connection
- **Long-term memory operations** do NOT need explicit Redis connection

---

## TESTING AND VERIFICATION

### Q9.1: Testing Pattern

**After applying each fix, verify with**:

```python
# Test create
create_long_term_memories(memories=[{
    "text": "Test memory",
    "topics": ["test"],
}])

# Verify in Redis
docker exec agent-memory-server-redis-1 redis-cli KEYS "memory:*"
# Should see: memory:{ulid}

# Verify in vector index
docker exec agent-memory-server-redis-1 redis-cli FT.SEARCH memory_records "*" LIMIT 0 10
# Should see the memory in results

# Test search
search_long_term_memory(text="test memory", limit=5)
# Should return the created memory
```

### Q9.2: Symptom Verification

**Before fix (Bug 1)**:
- ✅ Returns `{"status": "ok"}`
- ❌ No Redis keys created
- ❌ Search returns empty results
- ❌ Logs show "Successfully created" but no actual persistence

**After fix**:
- ✅ Returns `{"status": "ok"}`
- ✅ Redis keys appear: `KEYS "memory:*"` shows results
- ✅ Vector index has data: `FT.SEARCH memory_records "*"` returns records
- ✅ Search returns created memories
- ✅ Logs show actual Redis operations

**Redis key patterns to check**:
```bash
# Long-term memories (hash)
memory:{ulid}

# Working memory (hash)
working_memory:{session_id}

# Vector index (automatic)
memory_idx:*
```

---

## QUICK REFERENCE CHECKLIST

When fixing each tool:

- [ ] Import `ltm_module` and/or `wm_module` at top
- [ ] Import `get_redis_conn` if working memory operation
- [ ] Import `get_background_tasks` if write operation
- [ ] Apply namespace/user_id defaults at function start
- [ ] Call core module function directly (no FastAPI wrapper)
- [ ] Pass Redis connection for working memory ops
- [ ] Use background_tasks for async operations (create/update)
- [ ] Handle None returns appropriately (raise or return empty)
- [ ] Return correct response type (AckResponse, MemoryRecord, etc.)
- [ ] Test with Redis key verification

---

## SUMMARY

**Critical points for MacBook setup**:

1. **Never call FastAPI endpoint functions** - they have `Depends()` decorators that fail in MCP
2. **Working memory = needs Redis**, Long-term memory = doesn't
3. **Background tasks = separate call**, not passed to core functions
4. **Apply defaults first**, before calling core functions
5. **Filter None values** when building update dictionaries
6. **Error handling varies** by operation type (search vs get vs write)

**File paths for reference**:
- MCP tools: `agent_memory_server/mcp.py`
- Working memory: `agent_memory_server/working_memory.py`
- Long-term memory: `agent_memory_server/long_term_memory.py`
- Models: `agent_memory_server/models.py`

**All 9 tools are now working** in the Linux setup. Follow these patterns exactly for MacBook implementation.
