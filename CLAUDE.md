# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Redis Agent Memory Server is a memory layer for AI agents that provides:
- **Dual Interface**: REST API and Model Context Protocol (MCP) server
- **Two-Tier Memory**: Working memory (session-scoped) + Long-term memory (persistent)
- **Configurable Strategies**: Customizable memory extraction (discrete, summary, preferences, custom)
- **Pluggable Backends**: Vector store factory system supporting multiple databases
- **Semantic Search**: Vector-based search with metadata filtering using RedisVL

## Python Version

**CRITICAL**: This project requires Python 3.12 (not 3.13 or higher). The `pyproject.toml` specifies `requires-python = ">=3.12,<3.13"`.

## Redis Version

This project uses **Redis 8** (`redis:8` docker image). Do not use Redis Stack or earlier versions.

## Essential Commands

### Initial Setup

```bash
pip install uv                # Install uv package manager (once)
uv venv                       # Create virtual environment (once)
uv install --all-extras       # Install all dependencies
uv sync --all-extras          # Sync dependencies after changes
```

### Activate Virtual Environment

**REQUIRED** before running any commands:

```bash
source .venv/bin/activate
```

### Running Tests

**CRITICAL**: All tests must pass before committing (100% pass rate required).

```bash
# Run all tests
uv run pytest

# Run tests including those requiring API keys
uv run pytest --run-api-tests

# Run specific test file
uv run pytest tests/test_working_memory.py

# Run single test function
uv run pytest tests/test_working_memory.py::test_add_message

# Run tests in parallel (faster)
uv run pytest -n auto

# Run with verbose output
uv run pytest -v

# Run with coverage
uv run pytest --cov

# Run only unit tests
uv run pytest tests/unit/

# Run only integration tests
uv run pytest tests/integration/
```

### Linting and Formatting

```bash
uv run ruff check             # Run linter
uv run ruff format            # Format code
uv run ruff check --fix       # Auto-fix linting issues
```

### Pre-commit Hooks

This project uses `pre-commit` for automated checks:

```bash
uv run pre-commit install                # Install hooks (once)
uv run pre-commit run --all-files        # Run all hooks manually
```

Hooks include: ruff (linter + formatter), trailing whitespace, YAML validation, typos checker.

### Managing Dependencies

```bash
uv add <package>              # Add dependency to pyproject.toml
uv remove <package>           # Remove dependency
uv sync --all-extras          # Sync lockfile after changes
```

### Running Servers

```bash
# API Server (development mode with --no-worker)
uv run agent-memory api --no-worker

# API Server (production mode with separate worker)
uv run agent-memory api

# API Server with options
uv run agent-memory api --host 0.0.0.0 --port 8000 --reload

# MCP Server (stdio mode - for Claude Desktop)
uv run agent-memory mcp

# MCP Server (SSE mode - for development/testing)
uv run agent-memory mcp --mode sse --port 9000

# MCP Server (development mode with --no-worker)
uv run agent-memory mcp --mode sse --port 9000 --no-worker
```

### Background Task Worker

```bash
# Start task worker (for production deployment)
uv run agent-memory task-worker

# Start task worker with custom concurrency
uv run agent-memory task-worker --concurrency 10
```

### Database Operations

```bash
# Rebuild Redis search index
uv run agent-memory rebuild-index

# Run memory migrations
uv run agent-memory migrate-memories

# Schedule a specific task
uv run agent-memory schedule-task "agent_memory_server.long_term_memory.compact_long_term_memories"
```

### Docker Commands

```bash
# Start full stack (Redis + API + MCP + Worker)
docker-compose up

# Start only Redis
docker-compose up redis

# Stop all services
docker-compose down

# Rebuild and start
docker-compose up --build
```

## Critical Architectural Patterns

### 1. Dual Interface Design

The system exposes two interfaces that share the same core logic:

- **REST API** (`api.py`): HTTP endpoints for traditional web applications
- **MCP Server** (`mcp.py`): Model Context Protocol for AI agent integration

Both use shared modules: `working_memory.py`, `long_term_memory.py`, `extraction.py`, etc.

### 2. Two-Tier Memory System

```
Working Memory (Session-scoped)  →  Long-term Memory (Persistent)
    ↓                                      ↓
- Messages                          - Semantic search
- Structured memories               - Topic modeling
- Summary of conversation           - Entity recognition
- Metadata                          - Deduplication
```

**Key Flow:**
1. Messages stored in working memory (session-scoped)
2. Structured memories extracted from messages
3. Memories promoted to long-term storage (persistent)
4. Background tasks enrich memories (embeddings, topics, entities)
5. Semantic search retrieves relevant memories

### 3. Memory Extraction Strategies

**IMPORTANT**: The system supports configurable memory extraction strategies via `MemoryStrategyConfig`:

- **discrete** (default): Extract individual facts and preferences
- **summary**: Create conversation summaries
- **preferences**: Focus on user preferences/characteristics
- **custom**: Use domain-specific prompts with security validation

Strategies are configured in `WorkingMemory.long_term_memory_strategy` field. See `memory_strategies.py` and `docs/memory-extraction-strategies.md`.

### 4. Vector Store Factory System

**CRITICAL**: The system uses a pluggable vector store factory pattern:

- Configure via `VECTORSTORE_FACTORY` environment variable (Python dotted path)
- Default: `"agent_memory_server.vectorstore_factory.create_redis_vectorstore"`
- Custom factories must return `VectorStore` or `VectorStoreAdapter`
- Signature: `(embeddings: Embeddings) -> Union[VectorStore, VectorStoreAdapter]`

This allows swapping Redis for Chroma, Pinecone, or custom backends. See `vectorstore_factory.py` and `vectorstore_adapter.py`.

### 5. RedisVL Integration (Required)

**CRITICAL**: Always use RedisVL query types for search operations:

```python
# ✅ CORRECT - Use RedisVL queries
from redisvl.query import VectorQuery, FilterQuery

query = VectorQuery(
    vector=embedding,
    vector_field_name="vector",
    return_fields=["text", "metadata"]
)

# ❌ AVOID - Direct redis-py client searches
# redis.ft().search(...)  # Don't do this
```

This is a project requirement enforced throughout the codebase.

### 6. Async-First Design

- All core operations are async (`async def`)
- Background task processing via Docket (Redis-based queue)
- Async Redis connections throughout
- Use `asyncio.run()` for CLI commands that need async

### 7. Worker vs No-Worker Modes

**Development Mode (`--no-worker`)**:
- Background tasks run immediately in the same process
- Simpler for local development and testing
- Default for Docker development image

**Production Mode (no flag)**:
- Background tasks queued for separate worker process
- API server and task-worker run as separate containers
- Better scalability and fault isolation

## Authentication & Security

### Development vs Production

**DEVELOPMENT** (local testing only):
```bash
export DISABLE_AUTH=true
```

**PRODUCTION** (required):
```bash
export DISABLE_AUTH=false
export OAUTH2_ISSUER_URL=https://your-auth-provider.com
export OAUTH2_AUDIENCE=your-api-audience
```

**CRITICAL**: Never set `DISABLE_AUTH=true` in production environments.

### Authentication Modes

- `disabled`: No authentication (development only)
- `token`: API key/token authentication
- `oauth2`: OAuth2/JWT with JWKS validation

Supported providers: Auth0, AWS Cognito, Okta, Azure AD

### Custom Prompt Security

When using custom memory extraction strategies, prompts are validated for security:
- No system prompt injection attempts
- No credential/key extraction attempts
- Sandboxed execution via `prompt_security.py`

## Environment Configuration

### Required Variables

```bash
# Redis (required)
REDIS_URL=redis://localhost:6379

# LLM Provider (at least one required)
OPENAI_API_KEY=your-key
# OR
ANTHROPIC_API_KEY=your-key
```

### Common Configuration

```bash
# Models
GENERATION_MODEL=gpt-4o-mini           # For text generation
EMBEDDING_MODEL=text-embedding-3-small  # For vector embeddings
SLOW_MODEL=gpt-4o                       # For complex tasks
FAST_MODEL=gpt-4o-mini                  # For quick tasks

# Memory Features
LONG_TERM_MEMORY=true
ENABLE_DISCRETE_MEMORY_EXTRACTION=true
ENABLE_TOPIC_EXTRACTION=true
ENABLE_NER=true                         # Named Entity Recognition

# Topic Modeling
TOPIC_MODEL_SOURCE=LLM                  # or "BERTopic"
TOPIC_MODEL=gpt-4o-mini

# Vector Store
VECTORSTORE_FACTORY=agent_memory_server.vectorstore_factory.create_redis_vectorstore

# Background Tasks
USE_DOCKET=true
DOCKET_NAME=memory-server

# Logging
LOG_LEVEL=INFO                          # DEBUG, INFO, WARNING, ERROR
```

### Advanced Configuration

```bash
# RedisVL Settings
REDISVL_INDEX_NAME=memory_records
REDISVL_DISTANCE_METRIC=COSINE
REDISVL_VECTOR_DIMENSIONS=1536
REDISVL_INDEXING_ALGORITHM=HNSW

# Working Memory
SUMMARIZATION_THRESHOLD=0.7             # Fraction of context window

# Forgetting/Compaction
FORGETTING_ENABLED=false
FORGETTING_MAX_AGE_DAYS=90
COMPACTION_EVERY_MINUTES=10
```

## Project Structure

```
agent_memory_server/
├── main.py                    # FastAPI application factory
├── api.py                     # REST API endpoints
├── mcp.py                     # MCP server implementation
├── config.py                  # Settings and configuration
├── auth.py                    # OAuth2/JWT authentication
├── models.py                  # Pydantic data models
├── working_memory.py          # Session-scoped memory
├── long_term_memory.py        # Persistent memory with search
├── memory_strategies.py       # Configurable extraction strategies
├── vectorstore_factory.py     # Vector store factory pattern
├── vectorstore_adapter.py     # Adapter for vector stores
├── extraction.py              # Topic/entity extraction
├── summarization.py           # Conversation summarization
├── messages.py                # Message handling
├── filters.py                 # Search filtering
├── llms.py                    # LLM provider clients
├── docket_tasks.py            # Background task definitions
├── migrations.py              # Database migrations
├── cli.py                     # Command-line interface
├── dependencies.py            # FastAPI dependencies
├── healthcheck.py             # Health check endpoint
├── logging.py                 # Structured logging
├── prompt_security.py         # Custom prompt validation
└── utils/
    ├── redis.py               # Redis connection management
    ├── keys.py                # Redis key patterns
    ├── api_keys.py            # API key utilities
    ├── recency.py             # Recency scoring
    └── redis_query.py         # RedisVL query builders
```

## Testing Architecture

The project uses `pytest` with `testcontainers` for Redis integration tests.

### Test Organization

- `tests/unit/`: Pure unit tests (no external dependencies)
- `tests/integration/`: Integration tests (require Redis via testcontainers)
- `tests/`: Mixed integration tests (most tests are here)

### Test Fixtures (conftest.py)

Key fixtures available in tests:

- `redis_conn`: Redis connection
- `test_client`: FastAPI TestClient
- `working_memory`: WorkingMemory instance
- `memory_data`: Sample memory data

### Running Specific Tests

```bash
# By marker (if defined)
uv run pytest -m unit

# By directory
uv run pytest tests/unit/

# By pattern
uv run pytest -k "test_search"

# Single file
uv run pytest tests/test_working_memory.py

# Single test
uv run pytest tests/test_working_memory.py::test_add_message -v
```

## Development Workflow

1. **Initial Setup**: `pip install uv && uv venv && uv install --all-extras`
2. **Start Redis**: `docker-compose up redis`
3. **Configure Environment**: Set `DISABLE_AUTH=true` and API keys in `.env`
4. **Activate venv**: `source .venv/bin/activate`
5. **Run Server**: `uv run agent-memory api --no-worker`
6. **Make Changes**: Edit code
7. **Run Tests**: `uv run pytest` (must pass 100%)
8. **Run Linting**: `uv run pre-commit run --all-files`
9. **Commit**: Git commit (pre-commit hooks will run automatically)

## API Reference

### REST API Endpoints

- `POST /v1/working-memory/` - Create working memory session
- `GET /v1/working-memory/{id}` - Get working memory
- `POST /v1/working-memory/{id}/messages` - Add messages
- `POST /v1/long-term-memory/` - Create long-term memories
- `POST /v1/long-term-memory/search` - Search memories
- `POST /v1/memory/prompt` - Hydrate prompt with relevant memories
- `GET /v1/health` - Health check

### MCP Server Tools

- `create_long_term_memories` - Store persistent memories
- `search_long_term_memory` - Semantic search with filtering
- `memory_prompt` - Hydrate queries with context
- `set_working_memory` - Manage session memory
- `get_working_memory` - Retrieve session state

## Common Development Tasks

### Adding a New Memory Strategy

1. Create class in `memory_strategies.py` inheriting from `BaseMemoryStrategy`
2. Implement `extract_memories()` method
3. Register in `STRATEGY_REGISTRY` dict
4. Add tests in `tests/test_memory_strategies.py`
5. Update docs in `docs/memory-extraction-strategies.md`

### Adding a New Vector Store Backend

1. Create factory function: `(embeddings: Embeddings) -> VectorStore|VectorStoreAdapter`
2. Set `VECTORSTORE_FACTORY` to your function path
3. If custom adapter needed, inherit from `VectorStoreAdapter` in `vectorstore_adapter.py`
4. Add tests in `tests/integration/test_vectorstore_factory_integration.py`

### Adding a New LLM Provider

1. Add model config to `MODEL_CONFIGS` dict in `config.py`
2. Add provider client in `llms.py` (update `get_model_client()`)
3. Add tests for the new provider
4. Update documentation

## Release Process

### Releasing Server (Docker Images)

1. Update version in `agent_memory_server/__init__.py`
2. Commit and push to main
3. Go to GitHub Actions → "Release Docker Images"
4. Click "Run workflow"
5. Optionally check "Push latest tag"

Images published to:
- Docker Hub: `redislabs/agent-memory-server:<version>`
- GHCR: `ghcr.io/redis/agent-memory-server:<version>`

### Releasing Client (PyPI)

1. Merge PR to main
2. Tag commit with format `client/vX.Y.Z` (production) or `client/vX.Y.Z-test` (test PyPI)
3. Push tag: `git push origin client/vX.Y.Z`

## Documentation

- API docs: http://localhost:8000/docs (when server running)
- OpenAPI spec: http://localhost:8000/openapi.json
- Full docs: https://redis.github.io/agent-memory-server/
- Examples: `examples/` directory
