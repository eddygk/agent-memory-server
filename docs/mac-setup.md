# Agent Memory Server: WSL → Mac Migration Guide

**Created:** November 7, 2025
**For:** Eddy's MacBook setup
**Source:** Working WSL Ubuntu implementation from November 3-4, 2024

---

## Executive Summary

This guide documents the complete process to migrate your **perfectly working** agent-memory-server setup from WSL Ubuntu to macOS. The WSL implementation includes critical bug fixes discovered during 4+ hours of debugging in early November 2024.

**⚠️ CRITICAL:** The official repository contains silent bugs that make ALL MCP tools fail. This guide includes the fixes discovered during your WSL setup.

---

## Table of Contents

1. [Timeline & Context](#timeline--context)
2. [Prerequisites for Mac](#prerequisites-for-mac)
3. [Installation Steps](#installation-steps)
4. [Critical Bug Fixes to Apply](#critical-bug-fixes-to-apply)
5. [Environment Configuration](#environment-configuration)
6. [Testing & Verification](#testing--verification)
7. [Mac vs WSL Differences](#mac-vs-wsl-differences)
8. [Troubleshooting](#troubleshooting)

---

## Timeline & Context

### Original Work (WSL Ubuntu)
**Dates:** November 3-4, 2024
**Time Investment:** ~4 hours of debugging and fixes
**Result:** Fully functional MCP server with all critical bugs resolved

### Major Issues Discovered & Fixed

#### 1. **FastAPI Dependency Injection Bug** (CRITICAL - P0)
- **Symptom:** All 8 MCP tools return success but perform NO operations
- **Root Cause:** MCP server imported FastAPI endpoint functions with `Depends()` decorators
- **Impact:**
  - Memory creation returns `{"status": "ok"}` but nothing indexed in Redis
  - Search always returns `{"total": 0, "memories": []}`
  - Silent failure - no error messages
  - Complete MCP functionality breakdown

#### 2. **Docker Container Auto-Restart Issues**
- **Symptom:** Services don't restart after Docker Desktop restart
- **Affected:** redis, mcp-stdio, task-worker containers
- **Fix:** Added `restart: unless-stopped` to docker-compose.yml

#### 3. **Health Check Failures**
- **Symptom:** Container health checks failing
- **Cause:** Missing `procps` package (provides `pgrep` command)
- **Fix:** Added `procps` to Dockerfile system dependencies

#### 4. **MCP SDK Version Incompatibility**
- **Symptom:** `memory_prompt` tool fails with ImportError
- **Cause:** MCP SDK 1.9.4 removed `AssistantMessage` type
- **Fix:** Locked version to `mcp>=1.8.0` in pyproject.toml

---

## Prerequisites for Mac

### Required Software

1. **Homebrew** (Package Manager)
   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```

2. **Docker Desktop for Mac** (NOT Docker Engine)
   ```bash
   brew install --cask docker
   # Then launch Docker Desktop app from Applications
   ```

3. **Python 3.12** (CRITICAL: NOT 3.13 or higher)
   ```bash
   # Install Python 3.12 specifically
   brew install python@3.12

   # Verify version
   python3.12 --version  # Must be 3.12.x
   ```

4. **uv Package Manager**
   ```bash
   pip3.12 install uv
   ```

5. **Git** (Usually pre-installed on Mac)
   ```bash
   git --version
   ```

### System Requirements

- **macOS:** 11.0 (Big Sur) or later
- **RAM:** 8GB minimum (16GB recommended)
- **Disk Space:** 10GB free
- **Docker Desktop:** Running and logged in

---

## Installation Steps

### Step 1: Create Project Directory

```bash
# Create projects directory
mkdir -p ~/projects
cd ~/projects
```

### Step 2: Clone Official Repository

```bash
# Clone the official Redis agent-memory-server
git clone https://github.com/redis/agent-memory-server.git
cd agent-memory-server

# Verify you're on main branch
git branch --show-current
```

### Step 3: Set Up Python Environment

```bash
# Create virtual environment with Python 3.12
python3.12 -m venv .venv

# Activate virtual environment (Mac uses bash/zsh)
source .venv/bin/activate

# Verify correct Python version in venv
python --version  # Should show 3.12.x

# Install uv in the virtual environment
pip install uv

# Install all project dependencies
uv install --all-extras

# This will install:
# - FastAPI, Pydantic, Redis clients
# - OpenAI, Anthropic SDKs
# - MCP SDK
# - Testing frameworks
# - All extras
```

### Step 4: Verify Installation

```bash
# Check installed packages
uv pip list | grep -E "(mcp|fastapi|redis)"

# Expected output includes:
# mcp                  1.8.0
# fastapi              0.115.11+
# redis                5.x.x
```

---

## Critical Bug Fixes to Apply

**⚠️ IMPORTANT:** The official repository has silent bugs. You MUST apply these fixes.

### Fix 1: FastAPI Dependency Injection Bug in mcp.py

**File:** `agent_memory_server/mcp.py`

#### Change 1.1: Update Imports (Lines 8-11)

**BEFORE (BROKEN):**
```python
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

**AFTER (FIXED):**
```python
# Import core modules directly to bypass FastAPI dependency injection
# (don't import from api.py - those functions have Depends() decorators that break in MCP)
from agent_memory_server import long_term_memory as ltm_module
from agent_memory_server import working_memory as wm_module
```

#### Change 1.2: Rewrite All 8 MCP Tools

**⚠️ NOTE:** This is extensive. For complete implementation, reference the detailed documentation files you created:
- `FASTAPI_DEPENDENCY_INJECTION_ISSUES.md` (in your WSL repo)
- `MCP_DEPENDENCY_INJECTION_BUG_GITHUB_ISSUE.md` (in your WSL repo)

**Key Pattern for Each Tool:**

```python
# OLD (BROKEN) - Calls API endpoint with Depends()
@mcp_app.tool()
async def create_long_term_memories(memories: list[LenientMemoryRecord]) -> AckResponse:
    payload = CreateMemoryRecordRequest(memories=memories)
    return await core_create_long_term_memory(  # ❌ BROKEN
        payload, background_tasks=get_background_tasks()
    )

# NEW (FIXED) - Calls core module directly
@mcp_app.tool()
async def create_long_term_memories(memories: list[LenientMemoryRecord]) -> AckResponse:
    """Create long-term memories (MCP version - bypasses FastAPI dependencies)."""
    if not settings.long_term_memory:
        raise ValueError("Long-term memory is disabled")

    # Apply default namespace/user_id
    for mem in memories:
        if mem.namespace is None and settings.default_mcp_namespace:
            mem.namespace = settings.default_mcp_namespace
        if mem.user_id is None and settings.default_mcp_user_id:
            mem.user_id = settings.default_mcp_user_id
        mem.persisted_at = None

    # ✅ Call core function directly (bypasses FastAPI Depends())
    background_tasks = get_background_tasks()
    background_tasks.add_task(
        ltm_module.index_long_term_memories,
        memories=memories,
        deduplicate=True,
    )

    return AckResponse(status="ok")
```

**Tools to Rewrite (all in `mcp.py`):**
1. `create_long_term_memories`
2. `search_long_term_memory`
3. `get_long_term_memory`
4. `edit_long_term_memory`
5. `delete_long_term_memories`
6. `get_working_memory`
7. `set_working_memory`
8. `memory_prompt`

**SHORTCUT:** If you want to transfer the entire fixed `mcp.py` from WSL:
```bash
# On WSL, copy your fixed mcp.py somewhere accessible
# Then on Mac:
scp user@wsl-host:/path/to/mcp.py agent_memory_server/mcp.py
```

### Fix 2: Add procps to Dockerfile

**File:** `Dockerfile`

**Line 20** - Add `procps \` to system dependencies:

**BEFORE:**
```dockerfile
RUN apt-get update && apt-get install -y \
    curl \
    build-essential \
```

**AFTER:**
```dockerfile
RUN apt-get update && apt-get install -y \
    curl \
    procps \
    build-essential \
```

### Fix 3: Add Restart Policies to docker-compose.yml

**File:** `docker-compose.yml`

**Add to ALL services** (redis, api, mcp, mcp-stdio, task-worker):

```yaml
services:
  api:
    # ... existing config ...
    restart: unless-stopped  # ← ADD THIS

  mcp:
    # ... existing config ...
    restart: unless-stopped  # ← ADD THIS

  mcp-stdio:
    # ... existing config ...
    restart: unless-stopped  # ← ADD THIS

  task-worker:
    # ... existing config ...
    restart: unless-stopped  # ← ADD THIS

  redis:
    # ... existing config ...
    restart: unless-stopped  # ← ADD THIS
```

**Also add health checks** to mcp-stdio and task-worker:

```yaml
  mcp-stdio:
    # ... existing config ...
    healthcheck:
      test: ["CMD", "pgrep", "-f", "agent-memory"]
      interval: 30s
      timeout: 10s
      retries: 3

  task-worker:
    # ... existing config ...
    healthcheck:
      test: ["CMD", "pgrep", "-f", "agent-memory"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### Fix 4: Lock MCP SDK Version in pyproject.toml

**File:** `pyproject.toml`

**Line ~25** - Update MCP dependency:

**BEFORE:**
```toml
"mcp>=1.6.0",
```

**AFTER:**
```toml
"mcp>=1.8.0",
```

**Then re-sync dependencies:**
```bash
uv sync --all-extras
```

---

## Environment Configuration

### Create .env File

**⚠️ CRITICAL:** Do NOT copy your WSL `.env` file. The API key in it is exposed in this conversation and should be considered compromised. Get a fresh key.

```bash
# Create .env in project root
cat > .env << 'EOF'
# Redis Connection
# Mac uses standard port (not WSL's custom 16380)
REDIS_URL=redis://localhost:6379

# Server Settings
PORT=8000

# Memory Configuration
LONG_TERM_MEMORY=true
WINDOW_SIZE=12
GENERATION_MODEL=gpt-4o-mini
EMBEDDING_MODEL=text-embedding-3-small
ENABLE_TOPIC_EXTRACTION=true
ENABLE_NER=true

# OpenAI API Key
# ⚠️ GET NEW KEY FROM: https://platform.openai.com/api-keys
OPENAI_API_KEY=YOUR_NEW_KEY_HERE

# Anthropic API Key (if needed)
# ANTHROPIC_API_KEY=your_anthropic_key

# Development Mode (DISABLE AUTHENTICATION - DEVELOPMENT ONLY)
DISABLE_AUTH=true

# MCP Defaults
DEFAULT_MCP_USER_ID=default_user
DEFAULT_MCP_NAMESPACE=default

# Optional: Topic Modeling
TOPIC_MODEL_SOURCE=LLM
TOPIC_MODEL=gpt-4o-mini

# Optional: Logging
LOG_LEVEL=INFO
EOF
```

**Next steps:**
1. Get new OpenAI API key from https://platform.openai.com/api-keys
2. Replace `YOUR_NEW_KEY_HERE` in `.env`
3. Never commit `.env` to git (already in .gitignore)

---

## Testing & Verification

### Start Services

#### Step 1: Launch Docker Desktop

```bash
# Open Docker Desktop app
open -a Docker

# Wait for Docker to fully start (~30 seconds)
# Verify it's running
docker ps
```

#### Step 2: Start Redis First

```bash
# Start only Redis for initial testing
docker-compose up redis -d

# Verify Redis is running and healthy
docker-compose ps redis

# Expected output:
# NAME            STATUS          PORTS
# redis-1         Up (healthy)    0.0.0.0:6379->6379/tcp

# Test Redis connection
docker-compose exec redis redis-cli ping
# Should return: PONG
```

#### Step 3: Rebuild Containers with Fixes

```bash
# Rebuild all containers (picks up Dockerfile changes)
docker-compose build

# This may take 5-10 minutes first time
# Subsequent builds are faster due to caching
```

#### Step 4: Start All Services

```bash
# Start all services
docker-compose up -d

# Check status
docker-compose ps

# All services should show "Up" or "Up (healthy)"
```

#### Step 5: Check Logs

```bash
# View logs from all services
docker-compose logs -f

# Or check specific service
docker-compose logs -f mcp-stdio

# Look for:
# ✅ "MCP server started"
# ✅ "Connected to Redis"
# ❌ Any error messages
```

### Verify MCP Server Functionality

#### Test 1: MCP Server Starts

```bash
# Test MCP stdio mode directly
docker-compose exec mcp-stdio agent-memory mcp --mode stdio

# Press Ctrl+C to stop (it will block waiting for stdin)
```

#### Test 2: Memory Creation Actually Works (CRITICAL)

```bash
# Test direct core function (bypasses MCP - should always work)
docker-compose exec mcp-stdio python3 << 'EOF'
import asyncio
from agent_memory_server import long_term_memory as ltm_module
from agent_memory_server.models import LenientMemoryRecord
from agent_memory_server.dependencies import get_background_tasks

async def test():
    # Create memory using core function
    memories = [LenientMemoryRecord(
        text="Mac migration test memory",
        memory_type="semantic",
        topics=["testing", "migration"]
    )]

    background_tasks = get_background_tasks()
    background_tasks.add_task(
        ltm_module.index_long_term_memories,
        memories=memories,
        deduplicate=True,
    )

    print("✅ Memory creation task scheduled")

    # Wait for indexing
    await asyncio.sleep(3)

    # Search for it
    results = await ltm_module.search_long_term_memories(
        text="Mac migration",
        limit=10
    )

    print(f"✅ Search found {results.total} memories")
    if results.total > 0:
        print(f"   First result: {results.memories[0].text[:50]}...")

asyncio.run(test())
EOF

# Expected output:
# ✅ Memory creation task scheduled
# ✅ Search found 1 memories
#    First result: Mac migration test memory...
```

#### Test 3: Redis Actually Has Data

```bash
# Check Redis for indexed memories
docker-compose exec redis redis-cli KEYS "memory_idx:*"

# Should return array of keys like:
# 1) "memory_idx:01HXE2B..."
# 2) "memory_idx:01HXE2C..."

# NOT empty array!
```

#### Test 4: Health Checks Pass

```bash
# All containers should be healthy
docker-compose ps

# Look for "(healthy)" status on all services
```

### Configure Claude Desktop (Mac)

#### Step 1: Locate Config File

```bash
# Claude Desktop config location on Mac
open ~/Library/Application\ Support/Claude/
```

#### Step 2: Edit claude_desktop_config.json

**File:** `~/Library/Application Support/Claude/claude_desktop_config.json`

**Add agent-memory MCP server:**

```json
{
  "mcpServers": {
    "agent-memory": {
      "command": "docker",
      "args": [
        "exec",
        "-i",
        "agent-memory-server-mcp-stdio-1",
        "agent-memory",
        "mcp"
      ]
    }
  }
}
```

**Note:** The container name `agent-memory-server-mcp-stdio-1` assumes your docker-compose project is named `agent-memory-server`. Verify with:

```bash
docker-compose ps mcp-stdio --format "{{.Name}}"
```

#### Step 3: Restart Claude Desktop

```bash
# Quit Claude Desktop completely
# Then relaunch from Applications
```

#### Step 4: Verify MCP Connection

In Claude Desktop:
1. Look for MCP server indicator in UI
2. Try using an MCP tool (create memory, search, etc.)
3. Check logs: `docker-compose logs -f mcp-stdio`

### End-to-End Validation

#### Test via Claude Desktop

1. **Create Memory:**
   ```
   User: "Create a memory: My favorite color is blue"
   ```

2. **Verify in Redis:**
   ```bash
   docker-compose exec redis redis-cli KEYS "memory_idx:*"
   # Should show new keys (not empty!)
   ```

3. **Search for Memory:**
   ```
   User: "Search for memories about my favorite color"
   ```

4. **Should Return Results:**
   ```
   Expected: {"total": 1, "memories": [{"text": "...favorite color is blue..."}]}
   NOT: {"total": 0, "memories": []}  ← This means bug still present!
   ```

---

## Mac vs WSL Differences

### Docker

| Aspect | WSL Ubuntu | macOS |
|--------|-----------|-------|
| Docker Type | Docker Desktop (WSL integration) | Docker Desktop (HyperKit/VirtioFS) |
| Performance | Native Linux performance | Slight overhead (virtualization) |
| File System | Direct access to Linux FS | Slower file sharing |
| Networking | WSL network | macOS network stack |

### Paths

| Type | WSL Ubuntu | macOS |
|------|-----------|-------|
| Home | `/home/eddygk/` | `/Users/YOUR_USERNAME/` |
| Project | `/home/eddygk/mcp/agent-memory-server` | `/Users/YOUR_USERNAME/projects/agent-memory-server` |
| Docker volumes | `/var/lib/docker/volumes/` | VM-based storage |

### Shell

| Aspect | WSL Ubuntu | macOS |
|--------|-----------|-------|
| Default Shell | bash | zsh (since Catalina) |
| Activation | `source .venv/bin/activate` | `source .venv/bin/activate` (same) |
| Scripts | bash scripts work | bash/zsh both work |

### Redis

| Setting | WSL Ubuntu (Your Setup) | macOS (Recommended) |
|---------|------------------------|---------------------|
| Port | 16380 (custom) | 6379 (standard) |
| URL | `redis://localhost:16380` | `redis://localhost:6379` |
| Why Custom? | Avoid conflicts | No conflicts expected |

### Sudo

| Operation | WSL Ubuntu | macOS |
|-----------|-----------|-------|
| Docker commands | May need sudo setup | No sudo needed |
| Package installation | `sudo apt install` | `brew install` (no sudo) |
| Your setup script | `setup_sudo.sh` needed | NOT needed |

### Python

| Aspect | Both Platforms |
|--------|---------------|
| Version Required | Python 3.12 (NOT 3.13) |
| Virtual Env | `.venv/` directory |
| Package Manager | `uv` (same on both) |

---

## Troubleshooting

### Issue 1: MCP Tools Return Success But Nothing Happens

**Symptoms:**
- `create_long_term_memories` returns `{"status": "ok"}`
- Redis has no keys: `docker-compose exec redis redis-cli KEYS "memory_idx:*"` returns empty
- Search always returns `{"total": 0, "memories": []}`

**Cause:** You didn't apply the FastAPI Depends() fix to `mcp.py`

**Solution:**
1. Verify your `mcp.py` imports from core modules, NOT from `api.py`
2. Check line 8-11 of `agent_memory_server/mcp.py`
3. Should see: `from agent_memory_server import long_term_memory as ltm_module`
4. Should NOT see: `from agent_memory_server.api import`

### Issue 2: Docker Containers Don't Start

**Symptoms:**
- `docker-compose up` fails
- Containers exit immediately
- Health checks failing

**Diagnostics:**
```bash
# Check logs
docker-compose logs api
docker-compose logs mcp-stdio
docker-compose logs redis

# Check container status
docker-compose ps
```

**Common Causes:**

**A. Redis not running:**
```bash
docker-compose up redis -d
docker-compose logs redis
```

**B. Port conflicts:**
```bash
# Check if port 6379 already in use
lsof -i :6379

# If needed, change Redis port in docker-compose.yml and .env
```

**C. Missing environment variables:**
```bash
# Verify .env exists and has OPENAI_API_KEY
cat .env | grep OPENAI_API_KEY
```

### Issue 3: Python Version Issues

**Symptoms:**
- `uv install` fails with dependency errors
- Import errors when running code

**Solution:**
```bash
# Verify Python version
python --version  # Must be 3.12.x

# If wrong version in venv, recreate it
deactivate
rm -rf .venv
python3.12 -m venv .venv
source .venv/bin/activate
uv install --all-extras
```

### Issue 4: Permission Denied Errors

**Symptoms:**
- Can't write to directories
- Docker volume permission errors

**Solution (Mac):**
```bash
# Docker Desktop should handle permissions automatically
# If issues persist, check Docker Desktop settings:
# Preferences → Resources → File Sharing
# Ensure your project directory is allowed
```

### Issue 5: Container Health Checks Failing

**Symptoms:**
- Containers show "unhealthy" status
- `docker-compose ps` shows health: starting or unhealthy

**Solution:**
```bash
# 1. Check if procps was added to Dockerfile
grep procps Dockerfile
# Should show: procps \

# 2. Rebuild containers
docker-compose down
docker-compose build --no-cache
docker-compose up -d

# 3. Check health check commands work
docker-compose exec api curl -f http://localhost:8000/v1/health
docker-compose exec mcp-stdio pgrep -f agent-memory
```

### Issue 6: MCP Server Not Appearing in Claude Desktop

**Symptoms:**
- Claude Desktop doesn't show agent-memory MCP server
- No MCP tools available

**Solution:**
```bash
# 1. Verify container name is correct
docker-compose ps mcp-stdio --format "{{.Name}}"
# Use this exact name in claude_desktop_config.json

# 2. Test MCP server directly
docker exec -i $(docker-compose ps -q mcp-stdio) agent-memory mcp << EOF
{"jsonrpc": "2.0", "method": "initialize", "id": 1}
EOF
# Should return valid JSON response

# 3. Check Claude Desktop logs (Mac)
tail -f ~/Library/Logs/Claude/mcp*.log

# 4. Restart Claude Desktop completely
# Quit and relaunch from Applications
```

### Issue 7: OpenAI API Key Errors

**Symptoms:**
- "Invalid API key" errors
- 401 Unauthorized from OpenAI

**Solution:**
```bash
# 1. Get new API key
# Visit: https://platform.openai.com/api-keys

# 2. Update .env
nano .env
# Replace OPENAI_API_KEY value

# 3. Restart containers
docker-compose restart

# 4. Test API key
docker-compose exec api python3 << 'EOF'
import os
from openai import OpenAI
client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
print("API key valid!" if client.models.list() else "Invalid key")
EOF
```

---

## Quick Reference

### Essential Commands

```bash
# Activate virtual environment
source .venv/bin/activate

# Start all services
docker-compose up -d

# Stop all services
docker-compose down

# View logs
docker-compose logs -f

# Restart a service
docker-compose restart mcp-stdio

# Rebuild containers
docker-compose build

# Check container status
docker-compose ps

# Test Redis connection
docker-compose exec redis redis-cli ping

# Check Redis keys
docker-compose exec redis redis-cli KEYS "memory_idx:*"

# Shell into container
docker-compose exec mcp-stdio bash
```

### Validation Checklist

- [ ] Python 3.12 installed (`python3.12 --version`)
- [ ] Docker Desktop running
- [ ] Project cloned to `~/projects/agent-memory-server`
- [ ] Virtual environment created and activated
- [ ] Dependencies installed (`uv install --all-extras`)
- [ ] Critical bug fixes applied to all 4 files
- [ ] `.env` file created with valid API key
- [ ] Docker containers built successfully
- [ ] Redis container healthy
- [ ] Test memory creation actually indexes in Redis
- [ ] Test memory search returns results (not empty)
- [ ] Claude Desktop config updated
- [ ] MCP server appears in Claude Desktop
- [ ] End-to-end test successful

---

## What NOT to Transfer from WSL

**Do NOT copy these from WSL:**

```
❌ .venv/                    # Python virtual environment (rebuild on Mac)
❌ __pycache__/              # Python bytecode (will regenerate)
❌ .pytest_cache/            # Test cache (will regenerate)
❌ *.pyc files               # Compiled Python (will regenerate)
❌ node_modules/             # If any JS deps (reinstall)
❌ .git/                     # Use fresh clone from official repo
❌ docker volumes/           # Let Docker create fresh
❌ .env file directly        # API key is compromised, create new
❌ .DS_Store (if exists)     # Mac-specific junk
```

**Only transfer these VALUES (not files):**

```
✅ OpenAI API key (get fresh one, WSL key exposed)
✅ Model preferences (gpt-4o-mini, text-embedding-3-small)
✅ Custom settings (DISABLE_AUTH, LONG_TERM_MEMORY, etc.)
✅ Knowledge of which fixes to apply
```

---

## Success Criteria

Your Mac setup is successful when:

1. ✅ **All containers healthy:** `docker-compose ps` shows all services "Up (healthy)"
2. ✅ **Memory creation works:** Test creates memory, Redis has keys
3. ✅ **Memory search works:** Search returns actual results, not empty array
4. ✅ **MCP server accessible:** Claude Desktop shows agent-memory server
5. ✅ **End-to-end test passes:** Create → Search → Edit → Delete all work
6. ✅ **No silent failures:** All operations that return success actually work

---

## Additional Resources

### Documentation Files (from your WSL repo)

- `FASTAPI_DEPENDENCY_INJECTION_ISSUES.md` - Detailed bug analysis
- `MCP_DEPENDENCY_INJECTION_BUG_GITHUB_ISSUE.md` - Complete GitHub issue writeup
- `CLAUDE.md` - Project context and commands

### External Links

- **Official Repo:** https://github.com/redis/agent-memory-server
- **FastAPI Docs:** https://fastapi.tiangolo.com/tutorial/dependencies/
- **MCP Protocol:** https://modelcontextprotocol.io/introduction
- **RedisVL Docs:** https://redis.io/docs/clients/python-redisvl/
- **Docker Desktop for Mac:** https://docs.docker.com/desktop/install/mac-install/

### Getting Help

If you encounter issues:

1. Check troubleshooting section above
2. Review WSL implementation docs (FASTAPI_DEPENDENCY_INJECTION_ISSUES.md)
3. Check Docker logs: `docker-compose logs -f`
4. Verify all 4 fixes were applied correctly
5. Test core functions directly (bypass MCP)

---

## Timeline Estimate

**Fresh Mac setup (assuming prerequisites installed):**

1. Clone repo and setup Python: **15 minutes**
2. Apply bug fixes (4 files): **30 minutes** (if manual) or **5 minutes** (if copying from WSL)
3. Create .env and get API key: **10 minutes**
4. Build Docker containers: **10 minutes** (first time)
5. Test and verify: **20 minutes**

**Total: ~1.5 hours for careful, verified setup**

---

## Final Notes

This setup is based on your working WSL implementation from November 3-4, 2024. All critical bugs have been identified and documented. Following this guide carefully should result in a perfectly working Mac setup.

**Remember:** The official repo has silent bugs. Don't skip the bug fix section!

Good luck with your Mac migration! 🚀
