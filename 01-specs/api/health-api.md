# Health Check API

## Overview

Health check endpoints for monitoring the NestJS API service and its dependencies.

**Base URL:** `http://localhost:4000`

---

## Endpoints

### 1. GET /health

**Description:** Comprehensive health check for all system components.

**Response:**

```json
{
  "status": "healthy" | "degraded",
  "timestamp": "2026-02-11T10:30:00.000Z",
  "uptime": 123.456,
  "checks": {
    "database": {
      "status": "up" | "down",
      "message": "Error message (if down)"
    },
    "ragService": {
      "status": "up" | "down",
      "message": "HTTP 500 | Connection refused (if down)"
    }
  }
}
```

**Status Codes:**
- `200 OK` - Always returns 200, check `status` field for health state

---

### 2. GET /health/live

**Description:** Liveness probe for Kubernetes/container orchestration.

**Response:**

```json
{
  "status": "ok",
  "timestamp": "2026-02-11T10:30:00.000Z"
}
```

**Status Codes:**
- `200 OK` - Service is alive

---

### 3. GET /health/ready

**Description:** Readiness probe - checks if service can accept traffic.

**Response (Ready):**

```json
{
  "status": "ready",
  "timestamp": "2026-02-11T10:30:00.000Z"
}
```

**Response (Not Ready):**

```json
{
  "status": "not_ready",
  "timestamp": "2026-02-11T10:30:00.000Z"
}
```

**Status Codes:**
- `200 OK` - Always returns 200, check `status` field

---

## Usage Examples

### Check Overall Health

```bash
curl http://localhost:4000/health
```

### Kubernetes Probes

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 4000
  initialDelaySeconds: 10
  periodSeconds: 30

readinessProbe:
  httpGet:
    path: /health/ready
    port: 4000
  initialDelaySeconds: 5
  periodSeconds: 10
```

---

## Implementation Details

### Checked Components

1. **Database (PostgreSQL)**
   - Executes `SELECT 1` query
   - Timeout: Prisma default
   - Status: `up` | `down`

2. **RAG Service (FastAPI)**
   - Calls `GET {fastapi.baseUrl}/health`
   - Timeout: 3 seconds
   - Status: `up` | `down`

### Configuration

RAG service URL is read from `ConfigService`:

```typescript
this.config.get<string>('fastapi.baseUrl', 'http://localhost:8000')
```

Default: `http://localhost:8000`

---

## Security Headers

The API uses `helmet` middleware to set secure HTTP headers:

- `X-DNS-Prefetch-Control`
- `X-Frame-Options`
- `Strict-Transport-Security`
- `X-Content-Type-Options`
- `X-Permitted-Cross-Domain-Policies`
- `Referrer-Policy`
- `Content-Security-Policy`

Applied in `main.ts`:

```typescript
import helmet from 'helmet';
app.use(helmet());
```
