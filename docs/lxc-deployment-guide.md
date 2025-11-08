# Agent Memory Server - Proxmox LXC Deployment Guide

**Target Audience:** Claude Code instances running inside Proxmox LXC containers
**Purpose:** Autonomous deployment of full agent-memory-server stack with all ML capabilities
**Environment:** Proxmox LXC with 8GB+ RAM, 4+ cores, 20GB+ disk space

---

## Prerequisites Check

Before starting, verify your environment meets these requirements:

```bash
# Check available RAM (should be 8GB+)
free -h

# Check CPU cores (should be 4+)
nproc

# Check available disk space (should be 20GB+)
df -h /

# Check network connectivity
ping -c 3 google.com

# Verify you have root or sudo access
sudo whoami
```

**Expected Output:**
- RAM: `Mem: 8.0Gi` or higher
- CPUs: `4` or higher
- Disk: `20G` or higher available on root partition
- Network: Successful ping responses
- User: `root`

**CHECKPOINT 1:** All prerequisites met? If any check fails, resolve before proceeding.

---

## System Preparation

### 1. Update System Packages

```bash
# Update package lists
apt-get update

# Upgrade existing packages (optional but recommended)
apt-get upgrade -y

# Install basic utilities
apt-get install -y \
    curl \
    wget \
    git \
    ca-certificates \
    gnupg \
    lsb-release \
    net-tools \
    htop
```

**Expected Output:** `0 upgraded, 0 newly installed` or similar success message

**CHECKPOINT 2:** System updated successfully?

---

### 2. Install Docker Engine

```bash
# Remove old Docker versions (if any)
apt-get remove -y docker docker-engine docker.io containerd runc || true

# Add Docker's official GPG key
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package lists with Docker repo
apt-get update

# Install Docker Engine
apt-get install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin

# Verify Docker installation
docker --version
docker compose version
```

**Expected Output:**
- `Docker version 24.0.x` or higher
- `Docker Compose version v2.x.x` or higher

**CHECKPOINT 3:** Docker installed and version verified?

---

### 3. Configure Docker for Optimal Performance

```bash
# Start Docker service
systemctl start docker

# Enable Docker to start on boot
systemctl enable docker

# Verify Docker is running
systemctl status docker
```

**Expected Output:** `Active: active (running)`

```bash
# Test Docker with hello-world
docker run hello-world
```

**Expected Output:** `Hello from Docker!` message

**CHECKPOINT 4:** Docker service running and tested?

---

### 4. Configure Firewall Rules

```bash
# Check if UFW is installed and active
ufw status

# If UFW is active, allow required ports:

# Allow Redis (if you plan to expose it - generally NOT recommended)
# ufw allow 6379/tcp

# Allow API server (REST endpoints)
ufw allow 8000/tcp

# Allow MCP server SSE mode (network accessible)
ufw allow 9000/tcp

# Reload firewall
ufw reload || true

# Verify rules
ufw status numbered
```

**Note:** If UFW is not active, skip this step. Proxmox firewall will handle external access.

**CHECKPOINT 5:** Firewall configured (or not needed)?

---

## Repository Setup

### 5. Clone Agent Memory Server Repository

```bash
# Navigate to home directory
cd ~

# Create projects directory
mkdir -p /opt/agent-memory-server

# Clone the repository
cd /opt
git clone https://github.com/eddygk/agent-memory-server.git

# Enter repository directory
cd agent-memory-server

# Verify repository cloned successfully
git status
git log -1 --oneline
```

**Expected Output:**
- `On branch main` or similar
- Latest commit hash and message

**CHECKPOINT 6:** Repository cloned successfully?

---

### 6. Inspect Repository Structure

```bash
# List key files
ls -lh

# Verify critical files exist
test -f docker-compose.yml && echo "✓ docker-compose.yml found"
test -f pyproject.toml && echo "✓ pyproject.toml found"
test -f Dockerfile && echo "✓ Dockerfile found"
test -d agent_memory_server && echo "✓ agent_memory_server/ found"
```

**Expected Output:** All 4 checkmarks

**CHECKPOINT 7:** Repository structure verified?

---

## Environment Configuration

### 7. Create Environment File

```bash
# Navigate to repository root
cd /opt/agent-memory-server

# Create .env file
cat > .env << 'ENVEOF'
# Redis Configuration
REDIS_URL=redis://redis:6379

# LLM Provider - At least one API key required
# Choose OpenAI OR Anthropic (or both)
OPENAI_API_KEY=PLACEHOLDER_OPENAI_KEY
# ANTHROPIC_API_KEY=PLACEHOLDER_ANTHROPIC_KEY

# Model Configuration
GENERATION_MODEL=gpt-4o-mini
EMBEDDING_MODEL=text-embedding-3-small
SLOW_MODEL=gpt-4o
FAST_MODEL=gpt-4o-mini

# Memory Features - Enable ALL for full functionality
LONG_TERM_MEMORY=true
ENABLE_DISCRETE_MEMORY_EXTRACTION=true
INDEX_ALL_MESSAGES_IN_LONG_TERM_MEMORY=false

# Topic Modeling - Use BERTopic for local ML processing
ENABLE_TOPIC_EXTRACTION=true
TOPIC_MODEL_SOURCE=BERTopic
TOPIC_MODEL=MaartenGr/BERTopic_Wikipedia
TOP_K_TOPICS=3

# Named Entity Recognition - Use BERT for local ML processing
ENABLE_NER=true
NER_MODEL=dbmdz/bert-large-cased-finetuned-conll03-english

# RedisVL Configuration
REDISVL_INDEX_NAME=memory_records
REDISVL_DISTANCE_METRIC=COSINE
REDISVL_VECTOR_DIMENSIONS=1536
REDISVL_INDEXING_ALGORITHM=HNSW

# Server Configuration
PORT=8000
MCP_PORT=9000

# Background Tasks
USE_DOCKET=true
DOCKET_NAME=memory-server

# Authentication (disabled for local LXC)
DISABLE_AUTH=true
AUTH_MODE=disabled

# Logging
LOG_LEVEL=INFO

# Working Memory
SUMMARIZATION_THRESHOLD=0.7

# Forgetting/Compaction
FORGETTING_ENABLED=false
COMPACTION_EVERY_MINUTES=10
ENVEOF

# Display the created file (without showing API keys)
echo "✓ .env file created"
grep -E "^[A-Z_]+=.*" .env | grep -v "API_KEY" | head -20
```

**IMPORTANT:** The .env file contains `PLACEHOLDER_OPENAI_KEY`. You MUST replace this with an actual API key.

**Action Required:**
```bash
# Option 1: Edit .env file manually
nano .env
# Replace PLACEHOLDER_OPENAI_KEY with your actual OpenAI key
# Or uncomment and add ANTHROPIC_API_KEY

# Option 2: Set via command line
read -sp "Enter your OpenAI API key: " OPENAI_KEY
sed -i "s/PLACEHOLDER_OPENAI_KEY/$OPENAI_KEY/" .env
```

**Verify API key is set:**
```bash
grep "OPENAI_API_KEY" .env
# Should NOT show PLACEHOLDER
```

**CHECKPOINT 8:** .env file created and API key configured?

---

## Docker Deployment

### 8. Review Docker Compose Configuration

```bash
# Display docker-compose.yml
cat docker-compose.yml | head -50

# Verify services are defined
grep "services:" docker-compose.yml
grep -A 5 "redis:" docker-compose.yml
grep -A 5 "api:" docker-compose.yml
```

**Expected Services:**
- `redis` - Redis vector store
- `api` - FastAPI server
- `task-worker` - Background task processor
- `mcp-server` - MCP server (may or may not be in compose)

**CHECKPOINT 9:** Docker compose file reviewed?

---

### 9. Pull Docker Images

This step downloads all required Docker images. With ML models, this will be ~5-10GB.

```bash
# Pull images (this may take 5-15 minutes)
docker compose pull

# Verify images downloaded
docker images | grep agent-memory
```

**Expected Output:** Multiple images listed with sizes

**CHECKPOINT 10:** Docker images pulled successfully?

---

### 10. Start Services

```bash
# Start all services in detached mode
docker compose up -d

# Wait for services to initialize (30 seconds)
sleep 30

# Check service status
docker compose ps
```

**Expected Output:** All services should show `Up` status or `healthy`

```bash
# View logs to verify startup
docker compose logs --tail=50

# Check for any errors
docker compose logs | grep -i error | tail -20
```

**Expected Output:** Minimal or no errors. Some warnings are normal during initialization.

**CHECKPOINT 11:** All services started successfully?

---

### 11. Wait for ML Models to Download

BERTopic and BERT NER models will download on first run. This takes 5-15 minutes.

```bash
# Monitor API container logs for model downloads
docker compose logs -f api

# Look for messages like:
# "Loading BERTopic model..."
# "Loading NER model..."
# "Model loaded successfully"

# Press Ctrl+C to stop following logs once you see models loaded
```

**Alternative:** Check docker stats to see when model downloads complete (network activity drops to near zero)

```bash
docker stats --no-stream
```

**CHECKPOINT 12:** ML models downloaded and loaded?

---

### 12. Verify Redis Connection

```bash
# Test Redis connection
docker exec agent-memory-server-redis-1 redis-cli ping

# Expected Output: PONG

# Check Redis info
docker exec agent-memory-server-redis-1 redis-cli INFO server | grep redis_version
```

**Expected Output:** `redis_version:8.x.x`

**CHECKPOINT 13:** Redis operational?

---

### 13. Verify API Server

```bash
# Get LXC IP address
LXC_IP=$(hostname -I | awk '{print $1}')
echo "LXC IP: $LXC_IP"

# Test API health endpoint
curl -s http://localhost:8000/v1/health | jq .

# Expected Output:
# {
#   "status": "healthy",
#   "redis_connected": true,
#   ...
# }
```

**If `jq` is not installed:**
```bash
apt-get install -y jq
```

**CHECKPOINT 14:** API server responding?

---

## MCP Server Setup (SSE Mode)

The MCP server needs to run in SSE mode to be accessible over the network.

### 14. Start MCP Server in SSE Mode

**Option A: Run in Docker (Recommended)**

```bash
# Stop existing docker compose (we'll modify it)
docker compose down

# Add MCP server to docker-compose.yml
cat >> docker-compose.yml << 'MCCEOF'

  mcp-server:
    build: .
    command: agent-memory mcp --mode sse --port 9000 --host 0.0.0.0
    ports:
      - "9000:9000"
    env_file:
      - .env
    environment:
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
      - api
    restart: unless-stopped
MCCEOF

# Restart all services
docker compose up -d

# Verify MCP server is running
docker compose ps | grep mcp-server
```

**Expected Output:** `mcp-server` showing `Up` status

**Option B: Run as Standalone Container**

```bash
# Run MCP server in separate container
docker run -d \
  --name agent-memory-mcp \
  --network agent-memory-server_default \
  -p 9000:9000 \
  --env-file .env \
  -e REDIS_URL=redis://redis:6379 \
  agent-memory-server-api:latest \
  agent-memory mcp --mode sse --port 9000 --host 0.0.0.0

# Verify container running
docker ps | grep agent-memory-mcp
```

**CHECKPOINT 15:** MCP server container running?

---

### 15. Test MCP SSE Endpoint

```bash
# Get LXC IP
LXC_IP=$(hostname -I | awk '{print $1}')
echo "LXC IP: $LXC_IP"

# Test SSE endpoint (should return SSE stream headers)
curl -v http://localhost:9000/sse 2>&1 | head -20

# Test from external connection (if possible)
# curl -v http://$LXC_IP:9000/sse
```

**Expected Output:**
- HTTP/1.1 200 OK
- `Content-Type: text/event-stream`
- Connection stays open (SSE stream)

Press Ctrl+C to stop the curl request.

**CHECKPOINT 16:** MCP SSE endpoint accessible?

---

## Health Checks & Verification

### 16. Comprehensive System Check

```bash
# Create verification script
cat > /opt/verify-deployment.sh << 'VERIFYEOF'
#!/bin/bash

echo "=== Agent Memory Server Deployment Verification ==="
echo

echo "1. Docker Services:"
docker compose ps
echo

echo "2. Redis Status:"
docker exec agent-memory-server-redis-1 redis-cli ping
echo

echo "3. API Health:"
curl -s http://localhost:8000/v1/health | jq -r '.status // "FAILED"'
echo

echo "4. MCP SSE Endpoint:"
timeout 2 curl -s http://localhost:9000/sse > /dev/null 2>&1 && echo "✓ SSE endpoint responding" || echo "✗ SSE endpoint not responding"
echo

echo "5. Disk Usage:"
df -h / | tail -1
echo

echo "6. Memory Usage:"
free -h | grep Mem
echo

echo "7. LXC IP Address:"
hostname -I | awk '{print $1}'
echo

echo "8. Container Resource Usage:"
docker stats --no-stream
echo

echo "=== Verification Complete ==="
VERIFYEOF

chmod +x /opt/verify-deployment.sh

# Run verification
/opt/verify-deployment.sh
```

**Review the output:** All checks should pass.

**CHECKPOINT 17:** All verification checks passed?

---

## Systemd Service Setup (Optional but Recommended)

This ensures the agent-memory-server starts automatically after LXC reboots.

### 17. Create Systemd Service

```bash
# Create systemd service file
cat > /etc/systemd/system/agent-memory-server.service << 'SYSTEMDEOF'
[Unit]
Description=Agent Memory Server Docker Compose
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/agent-memory-server
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target
SYSTEMDEOF

# Reload systemd
systemctl daemon-reload

# Enable service
systemctl enable agent-memory-server.service

# Test service
systemctl start agent-memory-server.service

# Check status
systemctl status agent-memory-server.service
```

**Expected Output:** `Active: active (exited)`

**Verify auto-start:**
```bash
systemctl is-enabled agent-memory-server.service
# Expected: enabled
```

**CHECKPOINT 18:** Systemd service configured and enabled?

---

## Network Configuration

### 18. Document Network Access

```bash
# Get LXC IP address
LXC_IP=$(hostname -I | awk '{print $1}')

# Create network info file
cat > /opt/NETWORK_INFO.txt << EOF
=== Agent Memory Server Network Configuration ===

LXC IP Address: $LXC_IP

Service Endpoints:
- Redis:          redis://$LXC_IP:6379 (internal only)
- API Server:     http://$LXC_IP:8000
- API Health:     http://$LXC_IP:8000/v1/health
- API Docs:       http://$LXC_IP:8000/docs
- MCP SSE:        http://$LXC_IP:9000/sse

Client Configuration (for Claude Desktop):

Windows/macOS Config:
{
  "mcpServers": {
    "agent-memory-server": {
      "transport": {
        "type": "sse",
        "url": "http://$LXC_IP:9000/sse"
      }
    }
  }
}

Generated: $(date)
EOF

# Display network info
cat /opt/NETWORK_INFO.txt
```

**IMPORTANT:** Save the LXC IP address - you'll need it to configure clients.

**CHECKPOINT 19:** Network configuration documented?

---

### 19. Test External Connectivity

From your client machine (WSL/Windows), test connectivity:

```bash
# From client machine (not LXC), run:
# curl -v http://<LXC_IP>:9000/sse

# Example:
# curl -v http://192.168.1.100:9000/sse
```

**Expected Output:** SSE connection established

**If connection fails:**
- Check Proxmox firewall rules
- Verify ports 8000 and 9000 are allowed
- Check LXC network configuration (bridge mode recommended)

**CHECKPOINT 20:** External connectivity verified?

---

## Performance Optimization

### 20. Configure Docker Resource Limits (Optional)

For LXC with 8GB+ RAM, you can allocate resources explicitly:

```bash
# Edit docker-compose.yml to add resource limits
# This is optional for powerful LXC

# Example: Add to api service:
#   deploy:
#     resources:
#       limits:
#         cpus: '3'
#         memory: 6G
#       reservations:
#         cpus: '2'
#         memory: 4G
```

**CHECKPOINT 21:** Resource limits configured (if needed)?

---

## Monitoring and Maintenance

### 21. Create Monitoring Scripts

```bash
# Create log viewer script
cat > /opt/view-logs.sh << 'LOGSEOF'
#!/bin/bash
echo "=== Agent Memory Server Logs ==="
docker compose -f /opt/agent-memory-server/docker-compose.yml logs --tail=100 "$@"
LOGSEOF

chmod +x /opt/view-logs.sh

# Create restart script
cat > /opt/restart-services.sh << 'RESTARTEOF'
#!/bin/bash
echo "Restarting Agent Memory Server..."
cd /opt/agent-memory-server
docker compose down
docker compose up -d
echo "Services restarted"
/opt/verify-deployment.sh
RESTARTEOF

chmod +x /opt/restart-services.sh

# Test log viewer
/opt/view-logs.sh api
```

**CHECKPOINT 22:** Monitoring scripts created?

---

## Troubleshooting Guide

### Common Issues and Solutions

#### Issue 1: Docker containers not starting

```bash
# Check Docker service
systemctl status docker

# Restart Docker
systemctl restart docker

# Check logs
journalctl -u docker -n 50
```

#### Issue 2: Redis connection failures

```bash
# Check Redis container
docker logs agent-memory-server-redis-1

# Test Redis directly
docker exec agent-memory-server-redis-1 redis-cli ping

# Restart Redis
docker compose restart redis
```

#### Issue 3: ML models not loading

```bash
# Check disk space
df -h

# Check API logs for model download
docker compose logs api | grep -i "model\|download\|loading"

# Manually trigger model download
docker exec -it agent-memory-server-api-1 python -c "
from agent_memory_server.extraction import get_topic_model, get_ner_model
print('Loading NER model...')
get_ner_model()
print('Loading Topic model...')
get_topic_model()
print('Models loaded successfully')
"
```

#### Issue 4: MCP SSE endpoint not accessible

```bash
# Check if MCP server is running
docker compose ps | grep mcp

# Check MCP logs
docker compose logs mcp-server

# Verify port binding
netstat -tulpn | grep 9000

# Test locally
curl -v http://localhost:9000/sse
```

#### Issue 5: High memory usage

```bash
# Check container stats
docker stats

# If memory is critical, restart services
/opt/restart-services.sh

# Consider reducing concurrent workers
# Edit .env and set lower values for:
# TASK_WORKER_CONCURRENCY=5
```

#### Issue 6: Slow performance

```bash
# Check CPU usage
htop

# Check disk I/O
iostat -x 1 5

# Check Redis performance
docker exec agent-memory-server-redis-1 redis-cli --latency

# Consider adding more CPU cores to LXC
```

---

## Deployment Complete!

### Final Verification Checklist

Run this final check:

```bash
# Execute comprehensive verification
/opt/verify-deployment.sh

# Expected results:
# ✓ All Docker containers running
# ✓ Redis responding to PING
# ✓ API health check returns "healthy"
# ✓ MCP SSE endpoint accessible
# ✓ Sufficient disk space
# ✓ Memory usage < 80%
# ✓ LXC IP address displayed
```

### Summary of What's Running

- **Redis** (port 6379): Vector store for memory records
- **API Server** (port 8000): REST API for memory operations
- **Task Worker**: Background processing for memory extraction
- **MCP Server** (port 9000): Model Context Protocol server (SSE mode)
- **ML Models**: BERTopic + BERT NER loaded and ready

### Next Steps

1. **Document the LXC IP address** from `/opt/NETWORK_INFO.txt`
2. **Configure client machines** to connect to `http://<LXC_IP>:9000/sse`
3. **Test memory operations** via Claude Desktop
4. **Monitor logs** regularly using `/opt/view-logs.sh`

### Useful Commands

```bash
# View all logs
/opt/view-logs.sh

# View specific service logs
/opt/view-logs.sh api
/opt/view-logs.sh redis
/opt/view-logs.sh mcp-server

# Restart all services
/opt/restart-services.sh

# Verify deployment
/opt/verify-deployment.sh

# Check resource usage
docker stats

# Access Redis CLI
docker exec -it agent-memory-server-redis-1 redis-cli

# Access API container shell
docker exec -it agent-memory-server-api-1 bash
```

---

## Deployment Status: SUCCESS ✓

Your Agent Memory Server is now fully operational on Proxmox LXC with:
- ✓ Full ML capabilities (BERTopic, BERT NER)
- ✓ Network-accessible MCP server (SSE mode)
- ✓ Centralized Redis vector store
- ✓ Auto-start on boot (systemd)
- ✓ Monitoring and maintenance scripts

**LXC IP:** Check `/opt/NETWORK_INFO.txt`
**MCP Endpoint:** `http://<LXC_IP>:9000/sse`

---

**End of Deployment Guide**

Last Updated: 2025-01-08
