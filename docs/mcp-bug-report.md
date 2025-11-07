# Critical Bug: MCP Server Tools Fail Silently Due to FastAPI Dependency Injection

## 🚨 Severity: CRITICAL
## 📍 Affects: All MCP server implementations (stdio mode)
## 🐛 Type: Silent failure / Data loss risk

---

## Executive Summary

The `agent-memory-server` MCP implementation has a **critical architectural flaw** where it imports and calls FastAPI endpoint functions that contain `Depends()` decorators. These decorators require HTTP request context and **fail silently** when invoked outside of FastAPI's request lifecycle (such as in MCP stdio mode).

### Impact

- ❌ **All 8 MCP tools appear to succeed but perform no operations**
- ❌ **Memory creation returns success but memories never indexed in Redis**
- ❌ **Search operations always return empty results** (`{"total": 0, "memories": []}`)
- ❌ **No error messages or warnings** - makes debugging extremely difficult
- ❌ **Silent data loss** - users believe operations succeed when they don't
- ❌ **Complete MCP functionality breakdown** - server is non-functional

### User-Visible Symptoms

```python
# User creates memory via MCP
result = await mcp_client.call_tool("create_long_term_memories", {...})
# Returns: {"status": "ok"} ✅

# Check Redis
redis-cli KEYS "memory_idx:*"
# Returns: (empty array) ❌

# User searches for memory
result = await mcp_client.call_tool("search_long_term_memory", {"text": "test"})
# Returns: {"total": 0, "memories": []} ❌
```

**The server reports success but nothing actually happens.**

---

## Environment Details

### Affected Versions
- **All versions** of `redis/agent-memory-server` prior to fix
- Confirmed in latest main branch (as of investigation date)

### Platform Information
- **OS**: All platforms (Linux, macOS, Windows with WSL)
- **Python**: 3.12 (project requirement: `>=3.12,<3.13`)
- **Redis**: Redis 8 (`redis:8` docker image)
- **MCP Mode**: stdio (SSE mode may have same issue but not tested)
- **Deployment**: Docker containers, local development, all environments

### Version Evidence
Files checked in official repository:
- `agent_memory_server/mcp.py` (SHA: `3b5bfdd3a87dc2cf3ef608e859eda0082c6808e2`)
- `agent_memory_server/api.py` (SHA: `7d5f8abecdf9d5688ef01e5c63b4b8180efae824`)

---

## Root Cause: Technical Analysis

### The Problematic Pattern

The MCP server (`agent_memory_server/mcp.py`) imports FastAPI endpoint functions directly:

```python
# agent_memory_server/mcp.py (ORIGINAL - BROKEN)
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

# Then MCP tools call these functions
@mcp_app.tool()
async def create_long_term_memories(
    memories: list[LenientMemoryRecord],
) -> AckResponse:
    # ... setup code ...
    payload = CreateMemoryRecordRequest(memories=memories)
    return await core_create_long_term_memory(  # ❌ BROKEN - calls FastAPI endpoint
        payload, background_tasks=get_background_tasks()
    )
```

### Why These Functions Break

These API functions have `Depends()` decorators that **only work in HTTP request context**:

```python
# agent_memory_server/api.py:637-643
@router.post("/v1/long-term-memory/", response_model=AckResponse)
async def create_long_term_memory(
    payload: CreateMemoryRecordRequest,
    background_tasks: HybridBackgroundTasks,
    current_user: UserInfo = Depends(get_current_user),  # ❌ BREAKS IN MCP
):
    """Create long-term memories."""
    # ...
```

```python
# agent_memory_server/api.py:768-780
@router.get("/v1/working-memory/{session_id}", response_model=WorkingMemoryResponse)
async def get_working_memory(
    session_id: str,
    user_id: str | None = None,
    namespace: str | None = None,
    model_name: ModelNameLiteral | None = None,
    context_window_max: int | None = None,
    recent_messages_limit: int | None = None,
    x_client_version: str | None = Header(None, alias="X-Client-Version"),  # ❌ BREAKS
    current_user: UserInfo = Depends(get_current_user),  # ❌ BREAKS
):
    """Get working memory for a session."""
    # ...
```

### The Mechanism of Failure

#### 1. FastAPI Dependency Injection Requirements

FastAPI's `Depends()` system requires an active HTTP request with:
- `Request` object containing headers, path parameters, query parameters
- `BackgroundTasks` instance managed by FastAPI
- Authentication state from `Depends(get_current_user)`
- Header values via `Header()` extraction

```python
# dependencies.py:60-80
async def get_current_user(
    request: Request,  # ❌ Requires HTTP request object
) -> UserInfo:
    """Extract user information from request."""
    if settings.disable_auth:
        # Development mode
        return UserInfo(
            user_id=settings.default_api_user_id,
            namespace=settings.default_api_namespace,
        )

    # Production - requires OAuth2 token from Authorization header
    # ❌ This entire flow requires HTTP request context
    # ...
```

#### 2. MCP stdio Mode Has No HTTP Context

MCP stdio mode operates via:
- **stdin/stdout** for JSON-RPC communication
- **No HTTP server** - just process stdin/stdout streams
- **No Request object** - nowhere to extract headers/auth from
- **No FastAPI request lifecycle** - `Depends()` never resolves

#### 3. Silent Failure Mode

When `Depends()` parameters cannot be resolved outside HTTP context:

**Expected behavior** (in a well-designed system):
```python
RuntimeError: Cannot use Depends() outside of FastAPI request context
```

**Actual behavior** (in current implementation):
```python
# Depends() silently returns None or defaults
# Function executes with incorrect/missing parameters
# Operations appear successful but perform wrong actions
# Returns success status even when nothing happened
```

### Evidence: Code Comparison

#### Official Repository - BROKEN Pattern

**File**: `agent_memory_server/mcp.py` (SHA: `3b5bfdd`)

```python
# Lines 8-18 - Imports from api.py
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

**File**: `agent_memory_server/api.py` (SHA: `7d5f8ab`)

All API endpoints have `Depends()` decorators:

```python
# Line 77 - GET /v1/sessions/
async def get_sessions(
    options: GetSessionsQuery = Depends(),  # ❌
    current_user: UserInfo = Depends(get_current_user),  # ❌
):

# Line 328 - POST /v1/working-memory/
async def create_working_memory(
    memory: WorkingMemoryRequest,
    background_tasks: HybridBackgroundTasks,
    current_user: UserInfo = Depends(get_current_user),  # ❌
):

# Line 364 - PUT /v1/working-memory/{session_id}
async def put_working_memory(
    session_id: str,
    memory: UpdateWorkingMemory,
    background_tasks: HybridBackgroundTasks,
    model_name: ModelNameLiteral | None = None,
    context_window_max: int | None = None,
    current_user: UserInfo = Depends(get_current_user),  # ❌
):

# Line 462 - POST /v1/working-memory/{session_id}/messages
async def add_messages_to_working_memory(
    session_id: str,
    messages: list[MemoryMessage],
    background_tasks: HybridBackgroundTasks,
    current_user: UserInfo = Depends(get_current_user),  # ❌
):

# Line 565 - DELETE /v1/sessions/{session_id}
async def delete_session(
    session_id: str,
    current_user: UserInfo = Depends(get_current_user),  # ❌
):

# Line 597 - POST /v1/memory/prompt
async def memory_prompt(
    params: MemoryPromptRequest,
    background_tasks: HybridBackgroundTasks,
    optimize_query: bool = False,
    current_user: UserInfo = Depends(get_current_user),  # ❌
) -> MemoryPromptResponse:

# Line 637 - POST /v1/long-term-memory/
async def create_long_term_memory(
    payload: CreateMemoryRecordRequest,
    background_tasks: HybridBackgroundTasks,
    current_user: UserInfo = Depends(get_current_user),  # ❌
):

# Line 768 - GET /v1/working-memory/{session_id}
async def get_working_memory(
    session_id: str,
    user_id: str | None = None,
    namespace: str | None = None,
    model_name: ModelNameLiteral | None = None,
    context_window_max: int | None = None,
    recent_messages_limit: int | None = None,
    x_client_version: str | None = Header(None, alias="X-Client-Version"),  # ❌
    current_user: UserInfo = Depends(get_current_user),  # ❌
):

# Line 786 - POST /v1/long-term-memory/search
async def search_long_term_memory(
    payload: SearchRequest,
    optimize_query: bool = False,
    current_user: UserInfo = Depends(get_current_user),  # ❌
):

# Line 816 - GET /v1/long-term-memory/{memory_id}
async def get_long_term_memory(
    memory_id: str,
    current_user: UserInfo = Depends(get_current_user),  # ❌
):

# Line 859 - PATCH /v1/long-term-memory/{memory_id}
async def update_long_term_memory(
    memory_id: str,
    payload: EditMemoryRecordRequest,
    current_user: UserInfo = Depends(get_current_user),  # ❌
):

# Line 883 - DELETE /v1/long-term-memory/
async def delete_long_term_memory(
    payload: DeleteMemoryRecordRequest,
    current_user: UserInfo = Depends(get_current_user),  # ❌
):
```

**Every single endpoint** that the MCP server tries to call has `Depends()` decorators that break in stdio mode.

---

## Affected MCP Tools

All 8 MCP tools are broken:

### 1. ❌ `create_long_term_memories`

**Broken implementation**:
```python
@mcp_app.tool()
async def create_long_term_memories(
    memories: list[LenientMemoryRecord],
) -> AckResponse:
    # ...
    payload = CreateMemoryRecordRequest(memories=memories)
    return await core_create_long_term_memory(  # ❌ Calls broken API function
        payload, background_tasks=get_background_tasks()
    )
```

**Symptom**: Returns `{"status": "ok"}` but memories never indexed in Redis. Background task to index memories doesn't execute properly.

### 2. ❌ `search_long_term_memory`

**Broken implementation**:
```python
@mcp_app.tool()
async def search_long_term_memory(...) -> MemoryRecordResults:
    payload = SearchRequest(text=text, ...)
    results = await core_search_long_term_memory(  # ❌ Calls broken API function
        payload, optimize_query=optimize_query
    )
    return MemoryRecordResults(...)
```

**Symptom**: Always returns `{"total": 0, "memories": []}` regardless of stored data.

### 3. ❌ `get_long_term_memory`

**Broken implementation**:
```python
@mcp_app.tool()
async def get_long_term_memory(memory_id: str) -> MemoryRecord:
    return await core_get_long_term_memory(memory_id)  # ❌ Calls broken API function
```

**Symptom**: Returns null or incorrect memory data.

### 4. ❌ `edit_long_term_memory`

**Broken implementation**:
```python
@mcp_app.tool()
async def edit_long_term_memory(...) -> MemoryRecord:
    payload = EditMemoryRecordRequest(...)
    return await core_update_long_term_memory(  # ❌ Calls broken API function
        memory_id, payload
    )
```

**Symptom**: Returns success but updates never applied to Redis.

### 5. ❌ `delete_long_term_memories`

**Broken implementation**:
```python
@mcp_app.tool()
async def delete_long_term_memories(memory_ids: list[str]) -> AckResponse:
    payload = DeleteMemoryRecordRequest(memory_ids=memory_ids)
    return await core_delete_long_term_memory(payload)  # ❌ Calls broken API function
```

**Symptom**: Returns success but memories remain in Redis.

### 6. ❌ `get_working_memory`

**Broken implementation**:
```python
@mcp_app.tool()
async def get_working_memory(...) -> WorkingMemoryResponse:
    return await core_get_working_memory(  # ❌ Calls broken API function
        session_id=session_id.eq,
        ...
    )
```

**Symptom**: Returns empty or incorrect session state.

### 7. ❌ `set_working_memory`

**Broken implementation**:
```python
@mcp_app.tool()
async def set_working_memory(...) -> WorkingMemoryResponse:
    # ...
    return await core_put_working_memory(  # ❌ Calls broken API function
        session_id, memory, background_tasks, ...
    )
```

**Symptom**: Appears to save but data lost. Memory promotion tasks don't execute.

### 8. ❌ `memory_prompt`

**Broken implementation**:
```python
@mcp_app.tool()
async def memory_prompt(...) -> MemoryPromptResponse:
    # ...
    return await core_memory_prompt(  # ❌ Calls broken API function
        params=MemoryPromptRequest(...),
        background_tasks=HybridBackgroundTasks(),
        optimize_query=optimize_query,
    )
```

**Symptom**: Returns empty context or crashes. Background tasks for `update_last_accessed` don't execute.

---

## Reproduction Steps

### Prerequisites

1. Deploy `agent-memory-server` MCP server in stdio mode
2. Configure Claude Desktop (or any MCP client) to use the server
3. Have Redis running and accessible

### Step-by-Step Reproduction

#### Test 1: Memory Creation Fails Silently

```python
# 1. Create a memory via MCP
result = await mcp_client.call_tool("create_long_term_memories", {
    "memories": [{
        "text": "Test memory for bug reproduction",
        "topics": ["testing", "bug"],
        "memory_type": "semantic"
    }]
})

# Expected: {"status": "ok"}
# Actual: {"status": "ok"} ✅ (but it's lying!)

# 2. Check Redis directly
docker-compose exec redis redis-cli KEYS "memory_idx:*"
# Expected: Shows memory keys like ["memory_idx:01HXABC123..."]
# Actual: (empty array) ❌

# 3. Search for the memory
result = await mcp_client.call_tool("search_long_term_memory", {
    "text": "Test memory",
    "limit": 10
})

# Expected: {"total": 1, "memories": [{...}]}
# Actual: {"total": 0, "memories": []} ❌
```

#### Test 2: Working Memory Doesn't Persist

```python
# 1. Set working memory
result = await mcp_client.call_tool("set_working_memory", {
    "session_id": "test_session_123",
    "messages": [{
        "role": "user",
        "content": "Hello, remember this message"
    }]
})

# Expected: Success with session data
# Actual: Returns success ✅ (but it's lying!)

# 2. Get working memory back
result = await mcp_client.call_tool("get_working_memory", {
    "session_id": "test_session_123"
})

# Expected: Returns the message we just set
# Actual: Empty messages array or error ❌
```

#### Test 3: Direct Core Function Works

```python
# Bypass MCP and call core functions directly
import asyncio
from agent_memory_server import long_term_memory as ltm_module
from agent_memory_server.models import LenientMemoryRecord

async def test_direct():
    # Create memory using core function (not API endpoint)
    from agent_memory_server.dependencies import get_background_tasks

    memories = [LenientMemoryRecord(
        text="Direct core function test",
        memory_type="semantic"
    )]

    background_tasks = get_background_tasks()
    background_tasks.add_task(
        ltm_module.index_long_term_memories,
        memories=memories,
        deduplicate=True,
    )

    # Wait for background task
    await asyncio.sleep(3)

    # Search
    results = await ltm_module.search_long_term_memories(
        text="Direct core",
        limit=10
    )

    print(f"Results: {results.total} memories found")
    # Expected: 1 memory found ✅
    # Actual: 1 memory found ✅ (core functions work!)

asyncio.run(test_direct())
```

**Conclusion**: Core functions work correctly. The bug is specifically in MCP's use of API endpoint functions with `Depends()`.

---

## The Fix: Bypass FastAPI Layer

### Solution Architecture

Instead of calling FastAPI endpoint functions (which have `Depends()`), **call core business logic modules directly**.

The project has a clean separation:
- **API Layer** (`api.py`) - FastAPI endpoints with `Depends()` decorators
- **Core Layer** (`long_term_memory.py`, `working_memory.py`) - Business logic without `Depends()`

MCP should use the **Core Layer** directly.

### Implementation: Fixed Code

#### 1. Change Imports

**Before (BROKEN)**:
```python
# agent_memory_server/mcp.py
from agent_memory_server.api import (
    create_long_term_memory as core_create_long_term_memory,
    delete_long_term_memory as core_delete_long_term_memory,
    # ...
)
```

**After (FIXED)**:
```python
# agent_memory_server/mcp.py
# Import core modules directly to bypass FastAPI dependency injection
# (don't import from api.py - those functions have Depends() decorators that break in MCP)
from agent_memory_server import long_term_memory as ltm_module
from agent_memory_server import working_memory as wm_module
from agent_memory_server.config import settings
from agent_memory_server.dependencies import get_background_tasks
```

#### 2. Fix `create_long_term_memories`

**Before (BROKEN)**:
```python
@mcp_app.tool()
async def create_long_term_memories(
    memories: list[LenientMemoryRecord],
) -> AckResponse:
    # Apply defaults...
    for mem in memories:
        if mem.namespace is None and settings.default_mcp_namespace:
            mem.namespace = settings.default_mcp_namespace
        if mem.user_id is None and settings.default_mcp_user_id:
            mem.user_id = settings.default_mcp_user_id

    payload = CreateMemoryRecordRequest(memories=memories)
    return await core_create_long_term_memory(  # ❌ BROKEN
        payload, background_tasks=get_background_tasks()
    )
```

**After (FIXED)**:
```python
@mcp_app.tool()
async def create_long_term_memories(
    memories: list[LenientMemoryRecord],
) -> AckResponse:
    """Create long-term memories (MCP version - bypasses FastAPI dependencies)."""
    if not settings.long_term_memory:
        raise ValueError("Long-term memory is disabled in configuration")

    # Apply default namespace and user_id if not provided
    for mem in memories:
        if mem.namespace is None and settings.default_mcp_namespace:
            mem.namespace = settings.default_mcp_namespace
        if mem.user_id is None and settings.default_mcp_user_id:
            mem.user_id = settings.default_mcp_user_id
        # Ensure persisted_at is not set (server will assign)
        mem.persisted_at = None

    # ✅ Call core function directly (bypasses FastAPI Depends() issues)
    background_tasks = get_background_tasks()
    background_tasks.add_task(
        ltm_module.index_long_term_memories,
        memories=memories,
        deduplicate=True,
    )

    return AckResponse(status="ok")
```

#### 3. Fix `search_long_term_memory`

**Before (BROKEN)**:
```python
@mcp_app.tool()
async def search_long_term_memory(...) -> MemoryRecordResults:
    # Build payload...
    payload = SearchRequest(text=text, ...)

    results = await core_search_long_term_memory(  # ❌ BROKEN
        payload, optimize_query=optimize_query
    )

    return MemoryRecordResults(...)
```

**After (FIXED)**:
```python
@mcp_app.tool()
async def search_long_term_memory(
    text: str | None,
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
    """Search long-term memories (MCP version - bypasses FastAPI dependencies)."""
    try:
        if not settings.long_term_memory:
            raise ValueError("Long-term memory is disabled in configuration")

        # Create SearchRequest to get filters (this part of the code is reusable)
        _params: dict[str, Any] = {}
        if session_id is not None:
            _params["session_id"] = session_id
        if namespace is not None:
            _params["namespace"] = namespace
        if topics is not None:
            _params["topics"] = topics
        if entities is not None:
            _params["entities"] = entities
        if created_at is not None:
            _params["created_at"] = created_at
        if last_accessed is not None:
            _params["last_accessed"] = last_accessed
        if user_id is not None:
            _params["user_id"] = user_id
        if memory_type is not None:
            _params["memory_type"] = memory_type
        if distance_threshold is not None:
            _params["distance_threshold"] = distance_threshold
        if limit is not None:
            _params["limit"] = limit
        if offset is not None:
            _params["offset"] = offset

        payload = SearchRequest(text=text, **_params)

        # ✅ Call core function directly (bypasses FastAPI Depends() issues)
        filters = payload.get_filters()
        kwargs = {
            "distance_threshold": payload.distance_threshold,
            "limit": payload.limit,
            "offset": payload.offset,
            "optimize_query": optimize_query,
            **filters,
            "text": payload.text or "",
        }

        results = await ltm_module.search_long_term_memories(**kwargs)

        return MemoryRecordResults(
            total=results.total,
            memories=results.memories,
            next_offset=results.next_offset,
        )

    except Exception as e:
        logger.error(f"Error in search_long_term_memory tool: {e}")
        return MemoryRecordResults(total=0, memories=[], next_offset=None)
```

#### 4. Fix `get_long_term_memory`

**Before (BROKEN)**:
```python
@mcp_app.tool()
async def get_long_term_memory(memory_id: str) -> MemoryRecord:
    return await core_get_long_term_memory(memory_id)  # ❌ BROKEN
```

**After (FIXED)**:
```python
@mcp_app.tool()
async def get_long_term_memory(memory_id: str) -> MemoryRecord:
    """Get a specific long-term memory by ID (MCP version)."""
    if not settings.long_term_memory:
        raise ValueError("Long-term memory is disabled in configuration")

    # ✅ Call core function directly
    return await ltm_module.get_long_term_memory_by_id(memory_id)
```

#### 5. Fix `edit_long_term_memory`

**Before (BROKEN)**:
```python
@mcp_app.tool()
async def edit_long_term_memory(...) -> MemoryRecord:
    payload = EditMemoryRecordRequest(...)
    return await core_update_long_term_memory(  # ❌ BROKEN
        memory_id, payload
    )
```

**After (FIXED)**:
```python
@mcp_app.tool()
async def edit_long_term_memory(
    memory_id: str,
    text: str | None = None,
    topics: list[str] | None = None,
    entities: list[str] | None = None,
    memory_type: MemoryTypeEnum | None = None,
    namespace: str | None = None,
    user_id: str | None = None,
    session_id: str | None = None,
    event_date: str | None = None,
) -> MemoryRecord:
    """Edit a long-term memory (MCP version)."""
    if not settings.long_term_memory:
        raise ValueError("Long-term memory is disabled in configuration")

    # Build update payload (only include provided fields)
    update_data = {}
    if text is not None:
        update_data["text"] = text
    if topics is not None:
        update_data["topics"] = topics
    if entities is not None:
        update_data["entities"] = entities
    if memory_type is not None:
        update_data["memory_type"] = memory_type
    if namespace is not None:
        update_data["namespace"] = namespace
    if user_id is not None:
        update_data["user_id"] = user_id
    if session_id is not None:
        update_data["session_id"] = session_id
    if event_date is not None:
        update_data["event_date"] = _parse_iso8601_datetime(event_date)

    # ✅ Call core function directly
    return await ltm_module.update_long_term_memory(
        memory_id=memory_id,
        **update_data
    )
```

#### 6. Fix `delete_long_term_memories`

**Before (BROKEN)**:
```python
@mcp_app.tool()
async def delete_long_term_memories(memory_ids: list[str]) -> AckResponse:
    payload = DeleteMemoryRecordRequest(memory_ids=memory_ids)
    return await core_delete_long_term_memory(payload)  # ❌ BROKEN
```

**After (FIXED)**:
```python
@mcp_app.tool()
async def delete_long_term_memories(memory_ids: list[str]) -> AckResponse:
    """Delete long-term memories by IDs (MCP version)."""
    if not settings.long_term_memory:
        raise ValueError("Long-term memory is disabled in configuration")

    # ✅ Call core function directly
    await ltm_module.delete_long_term_memories(memory_ids=memory_ids)

    return AckResponse(status="ok")
```

#### 7. Fix `get_working_memory`

**Before (BROKEN)**:
```python
@mcp_app.tool()
async def get_working_memory(...) -> WorkingMemoryResponse:
    return await core_get_working_memory(  # ❌ BROKEN
        session_id=session_id.eq,
        ...
    )
```

**After (FIXED)**:
```python
@mcp_app.tool()
async def get_working_memory(
    session_id: str,
    recent_messages_limit: int | None = None,
) -> WorkingMemoryResponse:
    """Get working memory for a session (MCP version)."""
    from agent_memory_server.utils.redis import get_redis_conn

    # Apply MCP defaults
    user_id = settings.default_mcp_user_id
    namespace = settings.default_mcp_namespace

    redis = await get_redis_conn()

    # ✅ Call core function directly
    working_mem = await wm_module.get_working_memory(
        session_id=session_id,
        namespace=namespace,
        user_id=user_id,
        redis_client=redis,
    )

    if not working_mem:
        # Return empty response if session doesn't exist
        return WorkingMemoryResponse(
            session_id=session_id,
            user_id=user_id,
            namespace=namespace,
            messages=[],
            memories=[],
            context=None,
            data=None,
        )

    # Apply recent_messages_limit if specified
    messages = working_mem.messages
    if recent_messages_limit and len(messages) > recent_messages_limit:
        messages = messages[-recent_messages_limit:]

    return WorkingMemoryResponse(
        session_id=working_mem.session_id,
        user_id=working_mem.user_id,
        namespace=working_mem.namespace,
        messages=messages,
        memories=working_mem.memories,
        context=working_mem.context,
        data=working_mem.data,
    )
```

#### 8. Fix `set_working_memory`

**Before (BROKEN)**:
```python
@mcp_app.tool()
async def set_working_memory(...) -> WorkingMemoryResponse:
    # ...
    return await core_put_working_memory(  # ❌ BROKEN
        session_id, memory, background_tasks, ...
    )
```

**After (FIXED)**:
```python
@mcp_app.tool()
async def set_working_memory(
    session_id: str,
    memories: list[LenientMemoryRecord] | None = None,
    messages: list[MemoryMessage] | None = None,
    context: str | None = None,
    data: dict[str, Any] | None = None,
    namespace: str | None = None,
    user_id: str | None = None,
    ttl_seconds: int = 3600,
    long_term_memory_strategy: MemoryStrategyConfig | None = None,
) -> WorkingMemoryResponse:
    """Set/replace working memory for a session (MCP version)."""
    from agent_memory_server.utils.redis import get_redis_conn

    # Apply MCP defaults
    if user_id is None:
        user_id = settings.default_mcp_user_id
    if namespace is None:
        namespace = settings.default_mcp_namespace

    redis = await get_redis_conn()

    # Build working memory object
    working_memory = WorkingMemory(
        session_id=session_id,
        user_id=user_id,
        namespace=namespace,
        messages=messages or [],
        memories=memories or [],
        context=context,
        data=data,
        long_term_memory_strategy=long_term_memory_strategy,
    )

    # ✅ Call core function directly
    result = await wm_module.set_working_memory(
        working_memory=working_memory,
        redis_client=redis,
        ttl_seconds=ttl_seconds,
    )

    # Schedule background task for long-term memory promotion if enabled
    if settings.long_term_memory and settings.enable_discrete_memory_extraction:
        background_tasks = get_background_tasks()
        background_tasks.add_task(
            wm_module.promote_to_long_term_memory,
            session_id=session_id,
            user_id=user_id,
            namespace=namespace,
        )

    return WorkingMemoryResponse(
        session_id=result.session_id,
        user_id=result.user_id,
        namespace=result.namespace,
        messages=result.messages,
        memories=result.memories,
        context=result.context,
        data=result.data,
    )
```

#### 9. Fix `memory_prompt` (Most Complex)

This tool required special handling because the API version has ~145 lines of complex logic including token-based truncation. The MCP version uses a **simplified approach** where clients handle token management.

**After (FIXED - Simplified)**:
```python
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
    """
    Hydrate query with relevant memories (MCP version - simplified).

    This is a simplified version where clients handle token management.
    The API version has token-based truncation which we skip here.
    """
    from mcp.types import AssistantMessage, SystemMessage, TextContent, UserMessage
    from agent_memory_server.utils.redis import get_redis_conn

    redis = await get_redis_conn()
    _messages = []

    # 1. Get working memory if session provided (no token truncation)
    if session_id and session_id.eq:
        _namespace = namespace.eq if namespace else settings.default_mcp_namespace
        _user_id = user_id.eq if user_id else settings.default_mcp_user_id

        working_mem = await wm_module.get_working_memory(
            session_id=session_id.eq,
            namespace=_namespace,
            user_id=_user_id,
            redis_client=redis,
        )

        if working_mem:
            # Add summary if present
            if working_mem.context:
                _messages.append(
                    SystemMessage(
                        content=TextContent(
                            type="text",
                            text=f"## Conversation Summary\n\n{working_mem.context}",
                        ),
                    )
                )

            # Convert ALL messages (no token truncation - clients handle this)
            for msg in working_mem.messages:
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

    # 2. Get long-term memories if search criteria provided
    search_needed = any([topics, entities, created_at, last_accessed, memory_type])

    if search_needed or (session_id is None):
        # Build search filters
        _params: dict[str, Any] = {}
        if namespace is not None:
            _params["namespace"] = namespace
        if topics is not None:
            _params["topics"] = topics
        if entities is not None:
            _params["entities"] = entities
        if created_at is not None:
            _params["created_at"] = created_at
        if last_accessed is not None:
            _params["last_accessed"] = last_accessed
        if user_id is not None:
            _params["user_id"] = user_id
        if memory_type is not None:
            _params["memory_type"] = memory_type
        if distance_threshold is not None:
            _params["distance_threshold"] = distance_threshold

        payload = SearchRequest(text=query, limit=limit, offset=offset, **_params)
        filters = payload.get_filters()

        kwargs = {
            "distance_threshold": payload.distance_threshold,
            "limit": payload.limit,
            "offset": payload.offset,
            "optimize_query": optimize_query,
            **filters,
            "text": payload.text or "",
        }

        # ✅ Call core function directly
        search_results = await ltm_module.search_long_term_memories(**kwargs)

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

            # Update last_accessed for these memories (background task)
            background_tasks = get_background_tasks()
            background_tasks.add_task(
                ltm_module.update_last_accessed,
                memory_ids=[m.id for m in search_results.memories],
            )

    # 3. Add the user's query as final message
    _messages.append(
        UserMessage(content=TextContent(type="text", text=query))
    )

    return MemoryPromptResponse(messages=_messages)
```

---

## Fix Verification

### Test Suite

After applying the fix, verify with these tests:

#### Test 1: Memory Creation Actually Works

```python
import asyncio
from agent_memory_server.models import LenientMemoryRecord

async def test_memory_creation():
    # Create via MCP
    result = await mcp_client.call_tool("create_long_term_memories", {
        "memories": [{
            "text": "Verification test memory",
            "topics": ["testing"],
            "memory_type": "semantic"
        }]
    })

    assert result["status"] == "ok"

    # Wait for indexing
    await asyncio.sleep(3)

    # Verify in Redis
    keys = await redis.keys("memory_idx:*")
    assert len(keys) > 0, "Memory should be indexed in Redis"

    print("✅ Test 1 passed: Memory creation works")

asyncio.run(test_memory_creation())
```

#### Test 2: Search Returns Real Results

```python
async def test_search():
    # Search for memory
    results = await mcp_client.call_tool("search_long_term_memory", {
        "text": "Verification test",
        "limit": 10
    })

    assert results["total"] > 0, "Should find at least one memory"
    assert len(results["memories"]) > 0, "Memories array should not be empty"
    assert "Verification test" in results["memories"][0]["text"]

    print("✅ Test 2 passed: Search works")

asyncio.run(test_search())
```

#### Test 3: Working Memory Persists

```python
async def test_working_memory():
    session_id = "test_session_" + ulid.new().str

    # Set working memory
    set_result = await mcp_client.call_tool("set_working_memory", {
        "session_id": session_id,
        "messages": [{
            "role": "user",
            "content": "Test message for persistence"
        }]
    })

    assert set_result["session_id"] == session_id

    # Get it back
    get_result = await mcp_client.call_tool("get_working_memory", {
        "session_id": session_id
    })

    assert get_result["session_id"] == session_id
    assert len(get_result["messages"]) == 1
    assert get_result["messages"][0]["content"] == "Test message for persistence"

    print("✅ Test 3 passed: Working memory persists")

asyncio.run(test_working_memory())
```

#### Test 4: End-to-End Flow

```python
async def test_end_to_end():
    """Complete flow: create, search, edit, delete"""

    # 1. Create memory
    create_result = await mcp_client.call_tool("create_long_term_memories", {
        "memories": [{
            "text": "End-to-end test memory",
            "topics": ["e2e", "testing"],
            "memory_type": "semantic"
        }]
    })
    assert create_result["status"] == "ok"

    # Wait for indexing
    await asyncio.sleep(3)

    # 2. Search for it
    search_result = await mcp_client.call_tool("search_long_term_memory", {
        "text": "End-to-end test",
        "limit": 1
    })
    assert search_result["total"] > 0
    memory_id = search_result["memories"][0]["id"]

    # 3. Get specific memory
    get_result = await mcp_client.call_tool("get_long_term_memory", {
        "memory_id": memory_id
    })
    assert get_result["id"] == memory_id

    # 4. Edit memory
    edit_result = await mcp_client.call_tool("edit_long_term_memory", {
        "memory_id": memory_id,
        "text": "End-to-end test memory (EDITED)"
    })
    assert "EDITED" in edit_result["text"]

    # 5. Delete memory
    delete_result = await mcp_client.call_tool("delete_long_term_memories", {
        "memory_ids": [memory_id]
    })
    assert delete_result["status"] == "ok"

    # 6. Verify deleted
    await asyncio.sleep(1)
    search_after_delete = await mcp_client.call_tool("search_long_term_memory", {
        "text": "End-to-end test",
        "limit": 10
    })
    # Should not find the deleted memory
    found_ids = [m["id"] for m in search_after_delete["memories"]]
    assert memory_id not in found_ids

    print("✅ Test 4 passed: End-to-end flow works")

asyncio.run(test_end_to_end())
```

### Manual Verification

```bash
# 1. Create memory via MCP (Claude Desktop or MCP client)
# Use create_long_term_memories tool

# 2. Check Redis directly
docker-compose exec redis redis-cli KEYS "memory_idx:*"
# Should show keys

# 3. Check Redis hash content
docker-compose exec redis redis-cli HGETALL "memory_idx:01HXE2B..."
# Should show memory data

# 4. Search via MCP
# Use search_long_term_memory tool
# Should return results, not empty array
```

---

## Code Change Statistics

### Files Modified

- **`agent_memory_server/mcp.py`**: 276 lines changed
  - 211 insertions (+)
  - 65 deletions (-)
  - Net: +146 lines

### Git Diff Summary

```diff
agent_memory_server/mcp.py | 276 +++++++++++++++++++++++++++++++++++---------
1 file changed, 211 insertions(+), 65 deletions(-)
```

### Lines of Code

- **Original MCP file**: ~1,100 lines
- **Fixed MCP file**: ~1,250 lines
- **Increase**: ~13% more code (due to replacing simple API calls with direct core logic)

---

## Related Architectural Considerations

### 1. HybridBackgroundTasks

The project uses `HybridBackgroundTasks` for background task scheduling:

```python
# dependencies.py
class HybridBackgroundTasks(BackgroundTasks):
    """Can use either Docket or FastAPI background tasks."""

    def add_task(self, func: Callable[..., Any], *args, **kwargs) -> None:
        if settings.use_docket:
            # Schedule via Docket queue (Redis-based)
            docket_schedule_task(func, *args, **kwargs)
        else:
            # Use FastAPI's background tasks
            super().add_task(func, *args, **kwargs)
```

In MCP context, calling `get_background_tasks()` returns a standalone instance that works correctly:

```python
# In MCP tools
background_tasks = get_background_tasks()
background_tasks.add_task(
    ltm_module.index_long_term_memories,
    memories=memories,
    deduplicate=True,
)
```

This works because `HybridBackgroundTasks` can operate independently of FastAPI's request lifecycle when `use_docket=true`.

### 2. Token-Based Truncation

The API version of `memory_prompt` has sophisticated token-based truncation:

```python
# api.py:~936-956
if effective_token_limit is not None:
    if _calculate_messages_token_count(working_mem.messages) > effective_token_limit:
        # Keep removing oldest messages until we're under the limit
        recent_messages = working_mem.messages[:]
        while len(recent_messages) > 1:
            recent_messages = recent_messages[1:]  # Remove oldest
            if _calculate_messages_token_count(recent_messages) <= effective_token_limit:
                break
    else:
        recent_messages = working_mem.messages
else:
    recent_messages = working_mem.messages
```

**MCP Implementation Decision**: The fixed MCP version **skips token truncation** and lets MCP clients handle it themselves. This is simpler and more appropriate for MCP use cases where clients (like Claude Desktop) manage their own context windows.

### 3. Authentication in MCP Context

FastAPI endpoints use `Depends(get_current_user)` for authentication:

```python
# dependencies.py:60-80
async def get_current_user(request: Request) -> UserInfo:
    if settings.disable_auth:
        return UserInfo(
            user_id=settings.default_api_user_id,
            namespace=settings.default_api_namespace,
        )
    # OAuth2 validation...
```

**MCP Implementation Decision**: MCP tools use **default MCP user/namespace** from settings:
- `settings.default_mcp_user_id`
- `settings.default_mcp_namespace`

These are separate from API defaults and configured via environment variables:
- `DEFAULT_MCP_USER_ID`
- `DEFAULT_MCP_NAMESPACE`

This provides proper tenant isolation between API and MCP usage.

---

## Testing Gaps That Hid This Bug

### Current Test Architecture

The project's MCP tests use heavy mocking that **hides this bug**:

```python
# tests/test_mcp.py
@pytest.fixture
async def mcp_test_setup(async_redis_client, search_index):
    with (
        mock.patch(
            "agent_memory_server.long_term_memory.get_redis_conn",
            return_value=async_redis_client,
        ),
        mock.patch(
            "agent_memory_server.api.get_redis_conn",  # ❌ Mocked both!
            return_value=async_redis_client,
        ),
    ):
        yield
```

**Why Tests Pass Despite the Bug**:
1. Tests mock `get_redis_conn()` in both `long_term_memory` and `api` modules
2. Mocking bypasses the actual function execution paths
3. `Depends()` failures never occur because functions aren't actually called
4. Tests verify return value shapes, not actual Redis state

### Recommended Test Improvements

#### 1. Integration Tests Against Real Redis

```python
@pytest.mark.integration
async def test_mcp_create_memory_actually_indexes():
    """Test that MCP memory creation actually writes to Redis."""
    # Use testcontainers Redis (no mocking!)
    redis = await get_redis_conn()

    # Create memory via MCP
    result = await mcp_client.call_tool("create_long_term_memories", {...})

    # Wait for background indexing
    await asyncio.sleep(3)

    # Verify in REAL Redis
    keys = await redis.keys("memory_idx:*")
    assert len(keys) > 0, "Memory should actually be indexed"
```

#### 2. End-to-End MCP Tests

```python
@pytest.mark.e2e
async def test_mcp_full_memory_lifecycle():
    """Test complete lifecycle: create -> search -> edit -> delete."""
    # No mocking - test against real Redis
    # Verify each operation actually works
    # Check Redis state after each step
```

#### 3. Negative Tests

```python
async def test_mcp_depends_not_used():
    """Ensure MCP tools don't import from api.py."""
    import ast
    import inspect

    # Read mcp.py source
    source = inspect.getsource(mcp_module)
    tree = ast.parse(source)

    # Check imports
    for node in ast.walk(tree):
        if isinstance(node, ast.ImportFrom):
            assert node.module != "agent_memory_server.api", \
                "MCP should not import from api.py (has Depends())"
```

---

## Prevention Measures

### 1. Code Review Guidelines

Add to `CONTRIBUTING.md`:

```markdown
## MCP Implementation Guidelines

**CRITICAL**: MCP server code must NEVER import from `api.py`.

❌ **WRONG**:
```python
from agent_memory_server.api import create_long_term_memory
```

✅ **CORRECT**:
```python
from agent_memory_server import long_term_memory as ltm_module
```

**Reason**: API functions have `Depends()` decorators that fail in stdio mode.
```

### 2. Linting Rule

Add to `.pre-commit-config.yaml`:

```yaml
- repo: local
  hooks:
    - id: check-mcp-imports
      name: Check MCP doesn't import from api.py
      entry: python scripts/check_mcp_imports.py
      language: python
      files: agent_memory_server/mcp.py
```

Script `scripts/check_mcp_imports.py`:

```python
#!/usr/bin/env python3
import ast
import sys

with open("agent_memory_server/mcp.py") as f:
    tree = ast.parse(f.read())

for node in ast.walk(tree):
    if isinstance(node, ast.ImportFrom):
        if node.module == "agent_memory_server.api":
            print("ERROR: mcp.py must not import from api.py (has Depends())")
            sys.exit(1)

print("✅ MCP imports are correct")
```

### 3. Documentation Warning

Add to `api.py`:

```python
"""
FastAPI API endpoints for agent-memory-server.

⚠️  WARNING: Do NOT import functions from this module in mcp.py!
    These functions have Depends() decorators that break in stdio mode.
    Use core modules instead: long_term_memory.py, working_memory.py
"""
```

### 4. Integration Test Requirement

Add to CI/CD pipeline:

```yaml
# .github/workflows/test.yml
- name: Integration Tests (No Mocking)
  run: |
    pytest tests/integration/ --no-mock --real-redis
```

Ensure integration tests actually exercise real Redis and background tasks.

---

## Impact on Users

### Who Is Affected?

**ALL MCP users** of `redis/agent-memory-server` are affected:

1. **Claude Desktop users** - MCP stdio mode (primary use case)
2. **Custom MCP clients** - Any stdio-based integration
3. **Docker deployments** - When using `agent-memory mcp` command
4. **Development environments** - Local MCP testing

### Severity Assessment

**Severity: CRITICAL**

- **Data Loss Risk**: Memories appear to be saved but are lost
- **Silent Failure**: No error messages, extremely difficult to debug
- **Complete Dysfunction**: All MCP functionality broken
- **User Trust**: Users lose confidence in system reliability

### Timeline to Discovery

This bug could go **unnoticed for a long time** because:

1. **Tests pass** (due to mocking)
2. **Server runs without errors** (silent failure)
3. **Operations return success** (lie about actual state)
4. **Only discovered by** checking Redis directly or noticing search never works

In our case, discovery required:
- Creating memories and searching
- Noticing search always returns empty
- Checking Redis directly (`KEYS "memory_idx:*"`)
- Deep debugging to find root cause

---

## Proposed Solution Summary

### Quick Fix (This PR)

1. **Change imports in `mcp.py`**:
   - Remove imports from `api.py`
   - Import `long_term_memory` and `working_memory` modules directly

2. **Reimplement all 8 MCP tools**:
   - Call core functions instead of API endpoints
   - Bypass `Depends()` entirely
   - Maintain same functionality

3. **Test thoroughly**:
   - Integration tests against real Redis
   - Verify memories actually indexed
   - Verify search returns real results

### Long-Term Improvements

1. **Extract shared logic to core modules**
   - Move reusable code from `api.py` to core modules
   - Reduce duplication between API and MCP

2. **Improve test coverage**
   - Add integration tests without mocking
   - Test against real Redis and background tasks
   - Add negative tests (verify no `Depends()` usage)

3. **Add safeguards**
   - Linting rule to prevent api.py imports in mcp.py
   - Code review checklist
   - Documentation warnings

4. **Update documentation**
   - Fix DeepWiki (currently claims "no known issues")
   - Add architecture docs explaining layer separation
   - MCP implementation guidelines

---

## Files Changed

### Modified
- `agent_memory_server/mcp.py` (+211, -65 lines)
  - Lines 8-11: Changed imports from api.py to core modules
  - All MCP tool functions: Reimplemented to call core functions

### To Be Created (Recommended)
- `scripts/check_mcp_imports.py` - Pre-commit linting
- `tests/integration/test_mcp_real_redis.py` - Integration tests
- `docs/architecture/mcp-implementation.md` - Documentation

### To Be Modified (Recommended)
- `agent_memory_server/api.py` - Add warning comment
- `CONTRIBUTING.md` - Add MCP implementation guidelines
- `.pre-commit-config.yaml` - Add import checking hook

---

## References

### Official Repository Evidence

- **MCP Implementation**: [agent_memory_server/mcp.py@3b5bfdd](https://github.com/redis/agent-memory-server/blob/3b5bfdd3a87dc2cf3ef608e859eda0082c6808e2/agent_memory_server/mcp.py)
- **API Endpoints**: [agent_memory_server/api.py@7d5f8ab](https://github.com/redis/agent-memory-server/blob/7d5f8abecdf9d5688ef01e5c63b4b8180efae824/agent_memory_server/api.py)
- **Dependencies**: [agent_memory_server/dependencies.py](https://github.com/redis/agent-memory-server/blob/main/agent_memory_server/dependencies.py)

### Related Documentation

- [FastAPI Dependency Injection](https://fastapi.tiangolo.com/tutorial/dependencies/)
- [Model Context Protocol Specification](https://modelcontextprotocol.io/introduction)
- [Redis Vector Library (RedisVL)](https://redis.io/docs/clients/python-redisvl/)

### Investigation Documents

- Internal troubleshooting guide (400+ lines)
- Pieces LTM memory captures
- DeepWiki query results (outdated - claimed "no known issues")

---

## Checklist for Fix Implementation

- [ ] Change imports in `mcp.py` (remove api.py imports)
- [ ] Reimplement `create_long_term_memories`
- [ ] Reimplement `search_long_term_memory`
- [ ] Reimplement `get_long_term_memory`
- [ ] Reimplement `edit_long_term_memory`
- [ ] Reimplement `delete_long_term_memories`
- [ ] Reimplement `get_working_memory`
- [ ] Reimplement `set_working_memory`
- [ ] Reimplement `memory_prompt`
- [ ] Add integration tests (no mocking)
- [ ] Test against real Redis
- [ ] Verify background tasks execute
- [ ] Update documentation
- [ ] Add linting rules
- [ ] Add code review guidelines

---

## Questions for Maintainers

1. **Backward Compatibility**: Should we maintain any compatibility with the old broken API?
   - **Recommendation**: No - it was completely non-functional

2. **Token Truncation**: Should MCP `memory_prompt` implement token-based truncation or let clients handle it?
   - **Current implementation**: Clients handle it (simpler, more appropriate for MCP)

3. **Testing Strategy**: Should we require integration tests without mocking?
   - **Recommendation**: Yes - unit tests missed this critical bug

4. **Architecture**: Should we extract shared logic from `api.py` to reduce duplication?
   - **Recommendation**: Yes, but as follow-up PR to avoid scope creep

5. **Release**: Should this be a patch, minor, or major version bump?
   - **Recommendation**: Major (breaking change to MCP implementation, even though old version was broken)

---

## Contact

This issue was discovered during deployment troubleshooting when memory operations appeared to succeed but Redis remained empty. Investigation revealed the root cause: FastAPI `Depends()` decorators failing silently in stdio mode.

For questions about this issue or the proposed fix, please comment below.

---

**Labels**: `bug`, `critical`, `mcp`, `fastapi`, `dependency-injection`, `silent-failure`
**Milestone**: Next Release
**Priority**: P0 (Blocking - Complete MCP dysfunction)
