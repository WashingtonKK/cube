# Cube Cloud Deployment Guide

This guide explains how to deploy Cube cloud services and connect them to cube-agent instances running in CVMs.

## Overview

The Cube architecture is split into two parts:
1. **Cloud Services** - SuperMQ services, proxy, audit logs (deployed centrally)
2. **CVM Services** - cube-agent, ollama/vllm (deployed in confidential VMs)

## Prerequisites

- Docker and Docker Compose installed
- Access to `dev.cube.ultraviolet.rs` server
- SSH keys configured
- TDX-enabled server for running CVMs

## 1. Cloud Services Deployment

### Services Included

The cloud profile includes:
- **SuperMQ Services**: nats, auth, users, spicedb
- **Cube Proxy**: API gateway for agent connections
- **Audit Stack**: OpenSearch + Fluent Bit for logging
- **Monitoring**: Jaeger for distributed tracing

### Quick Start

```bash
cd docker

# Option 1: Use the test script (recommended for first deployment)
./test-cloud-deployment.sh

# Option 2: Manual deployment
# Start all cloud services
docker compose -f cloud-compose.yaml --profile cloud up -d

# Check service status
docker compose -f cloud-compose.yaml --profile cloud ps

# View logs
docker compose -f cloud-compose.yaml --profile cloud logs -f
```

The test script (`test-cloud-deployment.sh`) performs the following:
1. Validates Docker Compose configuration
2. Checks for required files
3. Pulls latest images
4. Stops any existing services
5. Starts cloud services
6. Waits for services to become healthy
7. Verifies all critical services are running
8. Runs smoke tests on each service

### Environment Configuration

Key environment variables in `.env`:

```bash
# Cube Proxy Configuration
UV_CUBE_AGENT_URL=https://<cvm-host>:7001  # Point to external agent
UV_CUBE_PROXY_HOST=0.0.0.0
UV_CUBE_PROXY_PORT=3001

# SuperMQ Auth
SMQ_AUTH_GRPC_URL=cube-cloud-auth:7001
SMQ_AUTH_GRPC_TIMEOUT=30s

# Domain
CUBE_DOMAIN=dev.cube.ultraviolet.rs
```

## 2. Agent Connection Configuration

### Agent in CVM Setup

The cube-agent runs inside a TDX CVM and connects to the cloud proxy:

```bash
# In the CVM (via buildroot or Ubuntu cloud-init)
export UV_CUBE_AGENT_LOG_LEVEL=info
export UV_CUBE_AGENT_HOST=0.0.0.0
export UV_CUBE_AGENT_PORT=7001
export UV_CUBE_AGENT_INSTANCE_ID=cube-agent-01
export UV_CUBE_AGENT_TARGET_URL=http://localhost:11434  # ollama/vllm
export UV_CUBE_AGENT_SERVER_CERT=/etc/certs/server.crt
export UV_CUBE_AGENT_SERVER_KEY=/etc/certs/server.key
export UV_CUBE_AGENT_SERVER_CA_CERTS=/etc/certs/ca.crt
export UV_CUBE_AGENT_CA_URL=https://prism.ultraviolet.rs/am-certs
```

### Proxy Configuration

Update `docker/config.json` to route requests to the agent:

```json
{
  "routes": [
    {
      "path": "/v1/chat/completions",
      "target": "$UV_CUBE_AGENT_URL",
      "methods": ["POST"]
    },
    {
      "path": "/v1/models",
      "target": "$UV_CUBE_AGENT_URL",
      "methods": ["GET"]
    }
  ]
}
```

## 3. Attestation Report Integration (Issue #90)

### Overview

The attestation report feature allows the UI to display TDX attestation quotes from cube-agents, proving they run in trusted CVMs.

### Agent Attestation Flow

1. **Agent generates TDX quote** when it detects `/dev/tdx_guest`
2. **Agent sends attestation report** to cloud proxy
3. **Proxy verifies and stores** attestation in database
4. **UI fetches and displays** attestation status

### Configuration

#### Enable Attestation in Agent

The agent automatically generates attestation reports when running in TDX mode. No additional configuration needed if:
- Running on TDX-enabled hardware
- TDX guest kernel is loaded
- `/dev/tdx_guest` device exists

#### Proxy Attestation Endpoint

Add attestation endpoint to proxy configuration:

```json
{
  "routes": [
    {
      "path": "/attestation/report",
      "target": "$UV_CUBE_AGENT_URL/attestation",
      "methods": ["GET", "POST"],
      "auth_required": true
    }
  ]
}
```

#### UI Integration

The UI should poll the attestation endpoint to display:
- Agent instance ID
- TDX quote (hex-encoded)
- Verification status
- Timestamp
- Platform details (CPU, firmware version)

Example UI component:

```javascript
// Fetch attestation report
const response = await fetch('https://dev.cube.ultraviolet.rs/proxy/attestation/report', {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});

const attestation = await response.json();

// Display in UI
{
  "agent_id": "cube-agent-01",
  "tdx_quote": "0x48656c6c6f...",
  "verified": true,
  "timestamp": "2025-12-08T10:00:00Z",
  "platform": {
    "cpu": "Intel Xeon Gold 6526Y",
    "tdx_version": "1.5"
  }
}
```

## 4. Testing the Deployment

### Automated Testing

Use the provided test script for comprehensive testing:

```bash
cd docker
./test-cloud-deployment.sh
```

### Manual Smoke Tests

```bash
# 1. Check all services are running
docker compose -f cloud-compose.yaml --profile cloud ps

# 2. Test OpenSearch
curl http://localhost:9200/_cluster/health

# 3. Test Traefik dashboard
curl http://localhost:8080/api/http/routers

# 4. Test proxy health (will fail if agent is not connected)
curl http://localhost:8900/health

# 5. Test auth service
curl http://localhost:8189/health

# 6. View audit logs
curl http://localhost:9200/cube-audit-*/_search?pretty

# 7. Check Jaeger UI
open http://localhost:16686
```

### Integration Tests

```bash
# Test agent connection through proxy
curl -X POST https://dev.cube.ultraviolet.rs/proxy/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "model": "llama3",
    "messages": [{"role": "user", "content": "Hello"}]
  }'

# Test attestation report
curl https://dev.cube.ultraviolet.rs/proxy/attestation/report \
  -H "Authorization: Bearer $TOKEN"
```

## 5. Connecting to dev.cube.ultraviolet.rs

### DNS Configuration

Ensure DNS points to your deployment server:

```bash
# Check DNS
dig dev.cube.ultraviolet.rs

# Should resolve to your server IP
```

### TLS Certificate Setup

Using Traefik with Let's Encrypt:

```yaml
# docker/traefik/traefik.yml
certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@ultraviolet.rs
      storage: /letsencrypt/acme.json
      httpChallenge:
        entryPoint: web
```

### Traefik Configuration

Add Traefik service to `cloud-compose.yaml`:

```yaml
services:
  traefik:
    image: traefik:v3.0
    profiles: ["cloud"]
    command:
      - "--api.dashboard=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencrypt.acme.email=admin@ultraviolet.rs"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik/letsencrypt:/letsencrypt
    networks:
      - cube-cloud-network
```

### Proxy Labels

The proxy service already has Traefik labels configured:

```yaml
labels:
  traefik.enable: "true"
  traefik.http.routers.cube-proxy.rule: "Host(`dev.cube.ultraviolet.rs`) && PathPrefix(`/proxy`)"
  traefik.http.routers.cube-proxy.entrypoints: "websecure"
  traefik.http.routers.cube-proxy.tls: "true"
  traefik.http.routers.cube-proxy.tls.certresolver: "letsencrypt"
```

## 6. GitHub Actions Deployment

### Prerequisites

Set up these GitHub Secrets:
- `DEPLOY_SSH_KEY`: SSH private key for deployment server
- `DEPLOY_USER`: SSH username (e.g., `ubuntu`)
- `CUBE_AGENT_URL`: URL of the cube-agent in CVM (optional override)

### Manual Deployment

Trigger deployment manually:

1. Go to Actions tab in GitHub
2. Select "Deploy Cube Cloud Services"
3. Click "Run workflow"
4. Choose environment (dev/staging/production)
5. Click "Run workflow"

### Automatic Deployment

Pushes to `main` branch automatically deploy to dev environment when these files change:
- `docker/cloud-compose.yaml`
- `docker/supermq-compose.yaml`
- `docker/.env`

## 7. Monitoring and Troubleshooting

### View Logs

```bash
# All services
docker compose -f cloud-compose.yaml --profile cloud logs -f

# Specific service
docker compose -f cloud-compose.yaml --profile cloud logs -f cube-proxy

# Audit logs in OpenSearch
curl http://localhost:9200/cube-audit-*/_search?pretty
```

### Check Service Health

```bash
# Service status
docker compose -f cloud-compose.yaml --profile cloud ps

# Inspect specific service
docker inspect cube-cloud-proxy

# Resource usage
docker stats
```

### Common Issues

#### Proxy can't connect to agent
- Check `UV_CUBE_AGENT_URL` points to correct CVM host
- Verify agent is running: `ssh cvm-host 'ps aux | grep cube-agent'`
- Check network connectivity: `curl -k https://cvm-host:7001/health`

#### Attestation reports not showing
- Verify agent is running in TDX mode: `ls /dev/tdx_guest`
- Check agent logs: `journalctl -u cube-agent | grep attestation`
- Ensure CA URL is reachable: `curl https://prism.ultraviolet.rs/am-certs`

#### Services not starting
- Check logs: `docker compose logs <service-name>`
- Verify environment variables in `.env`
- Ensure ports are not in use: `netstat -tulpn | grep <port>`

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    dev.cube.ultraviolet.rs                   │
│                         (Cloud)                              │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐ │
│  │ Traefik  │  │   Cube   │  │ SuperMQ  │  │ OpenSearch │ │
│  │  (TLS)   │─▶│  Proxy   │─▶│ Services │  │  + Fluent  │ │
│  └──────────┘  └──────────┘  └──────────┘  └────────────┘ │
│       │             │              │                         │
│       │             │              │                         │
│       │             ▼              │                         │
│       │        ┌─────────┐        │                         │
│       │        │ Jaeger  │        │                         │
│       │        └─────────┘        │                         │
└───────┼────────────┼──────────────┼─────────────────────────┘
        │            │              │
        │            │              │
        │      HTTPS │              │ gRPC
        │    (mTLS)  │              │
        │            │              │
        ▼            ▼              ▼
┌─────────────────────────────────────────────────────────────┐
│                    TDX Confidential VM                       │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌────────────┐              ┌────────────┐                 │
│  │ Cube-Agent │─────────────▶│   Ollama   │                 │
│  │  (TDX)     │              │   /vLLM    │                 │
│  │            │              │            │                 │
│  │ - Attest   │              │ - Models   │                 │
│  │ - TLS      │              │ - Inference│                 │
│  │ - /dev/    │              │            │                 │
│  │   tdx_guest│              │            │                 │
│  └────────────┘              └────────────┘                 │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

## Summary

1. ✅ **Cloud services** run on `dev.cube.ultraviolet.rs`
2. ✅ **Cube-agent** runs in TDX CVM with AI models
3. ✅ **Proxy** connects cloud to agent via mTLS
4. ✅ **Attestation reports** flow from agent to UI
5. ✅ **Audit logs** captured in OpenSearch
6. ✅ **Traefik** handles TLS termination and routing

For questions or issues, please open an issue at: https://github.com/ultravioletrs/cube/issues
