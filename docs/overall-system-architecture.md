## System Architecture Overview

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Interface Layer                      │
├─────────────────────────────────────────────────────────────────┤
│  Web UI (React)          │  REST API Clients  │  CLI Tool        │
│  - Upload Interface      │  - Python SDK      │  - Batch Jobs    │
│  - Job Dashboard         │  - cURL/Postman    │  - Automation    │
│  - Result Viewer         │                    │                  │
└──────────────┬───────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API Gateway Layer                           │
├─────────────────────────────────────────────────────────────────┤
│  FastAPI Application (Python 3.11+)                              │
│  - Authentication/Authorization (JWT)                            │
│  - Rate Limiting (per user/API key)                              │
│  - Request Validation & File Upload                              │
│  - WebSocket Server (real-time updates)                          │
│  - OpenAPI/Swagger Documentation                                 │
└──────────────┬───────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Orchestration Layer                           │
├─────────────────────────────────────────────────────────────────┤
│  Job Queue Manager                                               │
│  - Celery Distributed Task Queue                                 │
│  - Redis Message Broker & Result Backend                         │
│  - Task Priority & Routing                                       │
│  - Dead Letter Queue for Failed Tasks                            │
│  - Flower Monitoring Dashboard                                   │
└──────────────┬───────────────────────────────────────────────────┘
               │
     ┌─────────┴─────────┬──────────────┬──────────────┐
     ▼                   ▼              ▼              ▼
┌──────────┐    ┌──────────────┐  ┌──────────┐  ┌──────────────┐
│Extraction│    │   Routing    │  │ Analysis │  │  Reporting   │
│  Workers │    │   Workers    │  │ Workers  │  │   Workers    │
└──────────┘    └──────────────┘  └──────────┘  └──────────────┘
```

## Component Architecture

### 1. Storage Layer

```
┌─────────────────────────────────────────────────────────────────┐
│                        Storage Layer                             │
├─────────────────────────────────────────────────────────────────┤
│  PostgreSQL Database                                             │
│  ├─ jobs (id, status, user_id, created_at, metadata)            │
│  ├─ binaries (hash, filename, type, size, extraction_source)    │
│  ├─ analysis_results (binary_id, agent_id, decompiled_code)     │
│  ├─ vulnerabilities (binary_id, cve_id, severity, description)  │
│  └─ users (id, email, api_key_hash, role)                       │
│                                                                  │
│  Object Storage (MinIO/S3)                                       │
│  ├─ uploads/ (raw packages: .msi, .rpm, .zip)                   │
│  ├─ extracted/ (binaries: .exe, .dll, .so, .elf)                │
│  ├─ decompiled/ (source code artifacts)                          │
│  └─ reports/ (generated markdown/PDF reports)                    │
│                                                                  │
│  Redis Cache                                                     │
│  ├─ Job status cache (15min TTL)                                │
│  ├─ Binary hash → analysis result mapping                        │
│  └─ Rate limit counters                                          │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Extraction Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    Extraction Engine                             │
├─────────────────────────────────────────────────────────────────┤
│  File Type Detector                                              │
│  ├─ Magic number analysis (python-magic)                         │
│  ├─ PE header parsing (pefile)                                   │
│  ├─ ELF parsing (pyelftools)                                     │
│  └─ Package format detection                                     │
│                                                                  │
│  Format-Specific Extractors                                      │
│  ├─ MSI Extractor (msiexec /a)                                   │
│  ├─ RPM Extractor (rpm2cpio + cpio)                              │
│  ├─ DEB Extractor (ar + tar)                                     │
│  ├─ ZIP/7z Extractor (py7zr)                                     │
│  ├─ Docker Image Extractor (docker save + tar)                   │
│  └─ EXE Self-Extractor (monitor temp folder)                     │
│                                                                  │
│  Post-Extraction Processing                                      │
│  ├─ Recursive extraction (max depth: 5)                          │
│  ├─ Binary filtering (extensions, MIME types)                    │
│  ├─ Hash calculation (SHA256)                                    │
│  ├─ Deduplication check (skip if hash exists)                    │
│  └─ Metadata collection (size, timestamps, nested path)          │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Smart Routing Engine

```
┌─────────────────────────────────────────────────────────────────┐
│                      Routing Engine                              │
├─────────────────────────────────────────────────────────────────┤
│  Binary Classification Pipeline                                  │
│  ├─ Platform Detection (Windows PE/Linux ELF/macOS Mach-O)      │
│  ├─ Architecture Detection (x86/x64/ARM/MIPS)                    │
│  ├─ .NET Detection (CIL header check)                            │
│  ├─ Packer Detection (UPX/Themida/VMProtect signatures)          │
│  └─ Language Inference (strings analysis, imports)               │
│                                                                  │
│  MCP Server Selection Rules                                      │
│  ├─ if .NET assembly → DecompilerServer OR mcp-ilspy-server     │
│  ├─ if .NET + Unity → DecompilerServer (Unity support)          │
│  ├─ if Windows PE → DnSpy-MCPserver-Extension                    │
│  ├─ if Linux/macOS native → binary-mcp (Ghidra)                 │
│  ├─ if Android APK → binary-mcp (JADX integration)              │
│  └─ if packed binary → unpack first, then re-route               │
│                                                                  │
│  Load Balancing & Failover                                       │
│  ├─ Round-robin across MCP server replicas                       │
│  ├─ Health check polling (every 30s)                             │
│  ├─ Circuit breaker pattern (fail fast after 3 errors)           │
│  └─ Fallback routing (Ghidra → IDA Pro on timeout)              │
└─────────────────────────────────────────────────────────────────┘
```

### 4. MCP Server Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                   MCP Server Cluster                             │
├─────────────────────────────────────────────────────────────────┤
│  .NET Analysis Tier                                              │
│  ├─ DecompilerServer (port 3002, .NET 9.0 runtime)              │
│  │   └─ Capabilities: DLL decompilation, Unity assemblies        │
│  ├─ mcp-ilspy-server (stdio transport)                           │
│  │   └─ Capabilities: CIL disassembly, C# decompilation          │
│  └─ DnSpy-MCPserver-Extension (port 3001, SSE transport)         │
│      └─ Capabilities: Interactive debugging, patching            │
│                                                                  │
│  Native Binary Analysis Tier                                     │
│  ├─ binary-mcp (Docker container)                                │
│  │   ├─ Ghidra headless analyzer                                 │
│  │   ├─ ILSpyCmd wrapper                                         │
│  │   └─ x64dbg integration                                       │
│  └─ TriageMCP (port 3003)                                        │
│      ├─ YARA rule engine                                         │
│      ├─ Detect It Easy (DIE) integration                         │
│      └─ Initial triage and classification                        │
│                                                                  │
│  Supporting MCP Servers                                          │
│  ├─ filesystem-mcp (report generation)                           │
│  ├─ github-mcp (CVE/vulnerability lookup)                        │
│  └─ postgres-mcp (direct DB queries)                             │
└─────────────────────────────────────────────────────────────────┘
```

### 5. AI Agent Framework

```
┌─────────────────────────────────────────────────────────────────┐
│                    AI Agent Abstraction Layer                    │
├─────────────────────────────────────────────────────────────────┤
│  BaseAgent Interface                                             │
│  ├─ analyze(binary: Binary, context: dict) → AnalysisResult     │
│  ├─ explain(code: str, question: str) → str                      │
│  └─ get_capabilities() → list[str]                               │
│                                                                  │
│  Agent Implementations                                           │
│  ├─ ClaudeAgent                                                  │
│  │   ├─ Model: Claude 3.5 Sonnet via Anthropic API              │
│  │   ├─ MCP Support: Native (via Claude Desktop protocol)       │
│  │   ├─ Context Window: 200K tokens                              │
│  │   └─ Cost: $3/$15 per MTok (input/output)                    │
│  │                                                               │
│  ├─ OpenCodeInterpreterAgent                                     │
│  │   ├─ Model: CodeLlama/DeepSeek-Coder via local inference     │
│  │   ├─ MCP Support: Via HTTP proxy bridge                       │
│  │   ├─ Context Window: 16K-32K tokens                           │
│  │   └─ Cost: Free (self-hosted)                                │
│  │                                                               │
│  └─ Custom Agent Plugin System                                   │
│      ├─ Plugin discovery (agents/ directory)                     │
│      ├─ Dynamic loading (importlib)                              │
│      └─ Configuration via YAML                                   │
│                                                                  │
│  Agent Selection Strategy                                        │
│  ├─ Priority: user preference > cost > accuracy > speed          │
│  ├─ Fallback chain: Claude → OpenCodeInterpreter → Manual       │
│  └─ Cost optimization: batch similar queries to reduce API calls │
└─────────────────────────────────────────────────────────────────┘
```

### 6. Analysis Workers Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Celery Worker Pools                         │
├─────────────────────────────────────────────────────────────────┤
│  Worker Types (separate queues)                                  │
│  ├─ extraction_queue (CPU-bound, 4 workers)                      │
│  │   └─ Tasks: unpack archives, extract binaries                 │
│  │                                                               │
│  ├─ routing_queue (IO-bound, 8 workers)                          │
│  │   └─ Tasks: file type detection, MCP server selection         │
│  │                                                               │
│  ├─ analysis_queue (GPU-optional, 2 workers)                     │
│  │   └─ Tasks: invoke MCP servers, AI agent orchestration        │
│  │                                                               │
│  └─ reporting_queue (IO-bound, 2 workers)                        │
│      └─ Tasks: generate reports, PDF export, notifications       │
│                                                                  │
│  Worker Configuration                                            │
│  ├─ Concurrency: gevent (IO-bound) / prefork (CPU-bound)        │
│  ├─ Auto-scaling: scale between 2-10 workers based on queue      │
│  ├─ Task timeout: 30min (for large binaries)                     │
│  ├─ Retry policy: exponential backoff, max 3 retries             │
│  └─ Resource limits: 4GB RAM per worker, 2 CPU cores             │
│                                                                  │
│  Task Chain Example                                              │
│  upload → extract → deduplicate → route → analyze → store →     │
│  generate_report → notify_user                                   │
└─────────────────────────────────────────────────────────────────┘
```

## Deployment Architecture

### Docker Compose Stack

```yaml
services:
  # API Layer
  fastapi:
    image: easyreversing/api:latest
    networks: [dmz, application]
    depends_on: [postgres, redis]
    environment:
      - DATABASE_URL=postgresql://postgres:5432/easyreversing
      - REDIS_URL=redis://redis:6379/0
    
  # Worker Pools
  celery-extraction:
    image: easyreversing/worker:latest
    command: celery -A tasks worker -Q extraction_queue
    networks: [application, data]
    volumes:
      - shared-storage:/data
    
  celery-analysis:
    image: easyreversing/worker:latest
    command: celery -A tasks worker -Q analysis_queue
    networks: [application, mcp]
    
  # MCP Servers
  decompiler-server:
    image: pardeike/decompiler-server:latest
    networks: [mcp]
    
  binary-mcp:
    image: easyreversing/binary-mcp:latest
    networks: [mcp]
    volumes:
      - ghidra-projects:/ghidra
    
  # Data Layer
  postgres:
    image: postgres:16
    networks: [data]
    volumes:
      - postgres-data:/var/lib/postgresql/data
    
  redis:
    image: redis:7-alpine
    networks: [application]
    
  minio:
    image: minio/minio:latest
    networks: [data]
    volumes:
      - object-storage:/data

networks:
  dmz:
  application:
  mcp:
  data:

volumes:
  shared-storage:
  postgres-data:
  object-storage:
  ghidra-projects:
```

## Scalability & Performance

### Horizontal Scaling Strategy
- **API tier**: 3+ FastAPI replicas behind load balancer
- **Worker tier**: Auto-scale Celery workers (2-20 based on queue depth)
- **MCP servers**: 2+ replicas per server type with health checks
- **Database**: PostgreSQL primary with read replicas for reporting queries

### Performance Targets
- **Upload processing**: <5s for packages up to 100MB
- **Extraction**: <30s for typical MSI/RPM packages
- **Decompilation**: <2min for .NET assemblies up to 50MB
- **Report generation**: <10s for standard markdown reports
- **API response time**: p95 < 200ms for non-upload endpoints
