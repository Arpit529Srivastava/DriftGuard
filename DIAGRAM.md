# DriftGuard System Architecture & API Flow

## 🏗️ System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              DriftGuard System                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐            │
│  │   Kubernetes    │    │   Git Repo      │    │   DriftGuard    │            │
│  │   API Server    │    │   (Desired      │    │   Controller    │            │
│  │   (Live State)  │◄──►│   State)        │◄──►│   (Core Engine) │            │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘            │
│           │                       │                       │                    │
│           │                       │                       ▼                    │
│           │                       │              ┌─────────────────┐          │
│           │                       │              │   Drift         │          │
│           │                       │              │   Detection     │          │
│           │                       │              │   Engine        │          │
│           │                       │              └─────────────────┘          │
│           │                       │                       │                    │
│           │                       │                       ▼                    │
│           │                       │              ┌─────────────────┐          │
│           │                       │              │   State         │          │
│           │                       │              │   Tracking      │          │
│           │                       │              │   & Resolution  │          │
│           │                       │              └─────────────────┘          │
│           │                       │                       │                    │
│           └───────────────────────┼───────────────────────┼────────────────────┘
│                                   │                       │
│                                   ▼                       ▼
│                        ┌─────────────────┐    ┌─────────────────┐
│                        │   MongoDB       │    │   REST API      │
│                        │   (Enhanced     │    │   Server        │
│                        │   Drift Records)│    │   (HTTP/JSON)   │
│                        └─────────────────┘    └─────────────────┘
│                                   │                       │
│                                   ▼                       ▼
│                        ┌─────────────────┐    ┌─────────────────┐
│                        │   Prometheus    │    │   Health        │
│                        │   Metrics       │    │   Checks        │
│                        └─────────────────┘    └─────────────────┘
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## 🔄 Core Monitoring Flow

```
┌─────────────────┐
│   Start         │
│   DriftGuard    │
└─────────┬───────┘
          │
          ▼
┌─────────────────┐
│   Initialize    │
│   Connections   │
│   (K8s, Git,    │
│    MongoDB)     │
└─────────┬───────┘
          │
          ▼
┌─────────────────┐
│   Periodic      │
│   Analysis      │
│   (Every 30s)   │
└─────────┬───────┘
          │
          ▼
┌─────────────────┐
│   Fetch Live    │
│   K8s State     │
└─────────┬───────┘
          │
          ▼
┌─────────────────┐
│   Fetch Git     │
│   Desired State │
└─────────┬───────┘
          │
          ▼
┌─────────────────┐
│   Compare       │
│   States        │
└─────────┬───────┘
          │
          ▼
┌─────────────────┐
│   Drift         │
│   Detected?     │
└─────────┬───────┘
          │
    ┌─────┴─────┐
    │           │
    ▼           ▼
┌─────────┐ ┌─────────┐
│   No    │ │   Yes   │
│  Drift  │ │  Drift  │
└────┬────┘ └────┬────┘
     │           │
     ▼           ▼
┌─────────┐ ┌─────────┐
│ Update  │ │ Analyze │
│ Record  │ │ Drift   │
│ (No     │ │ Type &  │
│  Drift) │ │ Severity│
└────┬────┘ └────┬────┘
     │           │
     │           ▼
     │    ┌─────────┐
     │    │ Update  │
     │    │ Record  │
     │    │ (Active │
     │    │  Drift) │
     │    └────┬────┘
     │         │
     └─────────┼─────┐
               │     │
               ▼     ▼
        ┌─────────┐ ┌─────────┐
        │ Store   │ │ Log     │
        │ in      │ │ Drift   │
        │ MongoDB │ │ Event   │
        └────┬────┘ └────┬────┘
             │            │
             └────────────┼────┐
                          │    │
                          ▼    ▼
                   ┌─────────┐ ┌─────────┐
                   │ Wait    │ │ Update  │
                   │ Next    │ │ Metrics │
                   │ Cycle   │ │ (Prom)  │
                   └────┬────┘ └────┬────┘
                        │           │
                        └───────────┼──┐
                                    │  │
                                    ▼  ▼
                             ┌─────────┐
                             │ Repeat  │
                             │ Cycle   │
                             └─────────┘
```

## 🌐 API Endpoints & Data Flow

### Health & Monitoring APIs

```
┌─────────────────────────────────────────────────────────────────┐
│                    Health & Monitoring APIs                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GET /health                                                    │
│  ├── Response: {"status":"healthy","message":"Service is running"} │
│  └── Purpose: Basic health check for load balancers            │
│                                                                 │
│  GET /ready                                                     │
│  ├── Response: {"status":"ready","message":"All components ready"} │
│  └── Purpose: Kubernetes readiness probe                       │
│                                                                 │
│  GET /metrics                                                   │
│  ├── Response: Prometheus metrics format                       │
│  └── Purpose: Metrics collection for monitoring                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Drift Detection APIs

```
┌─────────────────────────────────────────────────────────────────┐
│                    Drift Detection APIs                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GET /api/v1/drift-records                                      │
│  ├── Query Params: namespace, drift_detected                    │
│  ├── Response: {"drift_records":[...],"count":N}               │
│  └── Purpose: List all drift records with filtering            │
│                                                                 │
│  GET /api/v1/drift-records/:resourceId                         │
│  ├── Params: resourceId (format: Kind:namespace:name)          │
│  ├── Response: Single drift record object                      │
│  └── Purpose: Get specific resource drift details              │
│                                                                 │
│  GET /api/v1/drift-records/active                              │
│  ├── Response: {"drift_records":[...],"count":N}               │
│  └── Purpose: Get currently active drift records               │
│                                                                 │
│  GET /api/v1/drift-records/resolved                            │
│  ├── Response: {"drift_records":[...],"count":N}               │
│  └── Purpose: Get resolved drift records                       │
│                                                                 │
│  GET /api/v1/statistics                                         │
│  ├── Response: {"statistics":{...}}                            │
│  └── Purpose: Get comprehensive drift statistics               │
│                                                                 │
│  POST /api/v1/analyze                                           │
│  ├── Response: {"message":"Analysis triggered","status":"started"} │
│  └── Purpose: Trigger manual drift analysis                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 📊 Data Flow Diagrams

### Drift Detection Flow

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Client    │    │   DriftGuard│    │   MongoDB   │
│   Request   │    │   API       │    │   Database  │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
       │ GET /api/v1/     │                  │
       │ drift-records    │                  │
       │─────────────────►│                  │
       │                  │                  │
       │                  │ Query drift      │
       │                  │ records          │
       │                  │─────────────────►│
       │                  │                  │
       │                  │ Return records   │
       │                  │◄─────────────────│
       │                  │                  │
       │ Return JSON      │                  │
       │ response         │                  │
       │◄─────────────────│                  │
       │                  │                  │
```

### Drift Analysis Flow

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Client    │    │   DriftGuard│    │ Kubernetes  │    │   Git Repo  │
│   Request   │    │   Controller│    │   API       │    │             │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │                  │
       │ POST /api/v1/    │                  │                  │
       │ analyze          │                  │                  │
       │─────────────────►│                  │                  │
       │                  │                  │                  │
       │                  │ Get live state   │                  │
       │                  │─────────────────►│                  │
       │                  │                  │                  │
       │                  │ Return live      │                  │
       │                  │ state            │                  │
       │                  │◄─────────────────│                  │
       │                  │                  │                  │
       │                  │ Get desired      │                  │
       │                  │ state            │                  │
       │                  │────────────────────────────────────►│
       │                  │                  │                  │
       │                  │ Return desired   │                  │
       │                  │ state            │                  │
       │                  │◄────────────────────────────────────│
       │                  │                  │                  │
       │                  │ Compare states   │                  │
       │                  │ & detect drift   │                  │
       │                  │                  │                  │
       │                  │ Store results    │                  │
       │                  │ in MongoDB       │                  │
       │                  │                  │                  │
       │ Return success   │                  │                  │
       │ response         │                  │                  │
       │◄─────────────────│                  │                  │
       │                  │                  │                  │
```

## 🔍 Drift Detection Logic

### Resource State Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                    Drift Detection Logic                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Fetch Live State from Kubernetes API                       │
│     ├── Deployments, Services, ConfigMaps, Secrets             │
│     └── Filter by configured namespaces and resource types     │
│                                                                 │
│  2. Fetch Desired State from Git Repository                    │
│     ├── Parse YAML manifests from specified branch             │
│     └── Extract resource definitions                           │
│                                                                 │
│  3. Compare States for Each Resource                           │
│     ├── Generate hash of live state                            │
│     ├── Generate hash of desired state                         │
│     └── Compare hashes for equality                            │
│                                                                 │
│  4. If Hashes Differ - Deep Analysis                           │
│     ├── Field-by-field comparison                              │
│     ├── Classify drift type (Scaling, VersionChange, etc.)     │
│     ├── Assign severity level (Low, Medium, High)              │
│     └── Generate detailed drift report                         │
│                                                                 │
│  5. Update Drift Record                                        │
│     ├── Set drift_status: "active" or "resolved"               │
│     ├── Update timestamps (first_detected, resolved_at)        │
│     ├── Store drift details and resolution message             │
│     └── Update last_known_good_hash                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Drift Classification Matrix

```
┌─────────────────────────────────────────────────────────────────┐
│                    Drift Classification                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Drift Type          │ Severity │ Examples                     │
│  ────────────────────┼──────────┼────────────────────────────── │
│  Scaling             │ Medium   │ Replica count changes        │
│  VersionChange       │ High     │ Container image updates      │
│  ResourceChange      │ Medium   │ CPU/memory modifications     │
│  ConfigurationChange │ Medium   │ Environment variables        │
│  LabelChange         │ Low      │ Label additions/removals     │
│  AnnotationChange    │ Low      │ Annotation modifications     │
│  PortChange          │ Medium   │ Port configuration changes   │
│  VolumeChange        │ Medium   │ Volume mount modifications   │
│  SecretChange        │ High     │ Secret reference changes     │
│  ConfigMapChange     │ Medium   │ ConfigMap reference changes  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 📈 State Transition Flow

```
┌─────────────────┐
│   Initial       │
│   State         │
│   (No Drift)    │
└─────────┬───────┘
          │
          ▼
┌─────────────────┐
│   Monitor       │
│   Resource      │
└─────────┬───────┘
          │
          ▼
┌─────────────────┐
│   Drift         │
│   Detected?     │
└─────────┬───────┘
          │
    ┌─────┴─────┐
    │           │
    ▼           ▼
┌─────────┐ ┌─────────┐
│   No    │ │   Yes   │
│  Drift  │ │  Drift  │
└────┬────┘ └────┬────┘
     │           │
     ▼           ▼
┌─────────┐ ┌─────────┐
│ Update  │ │ Set     │
│ Status: │ │ Status: │
│ "none"  │ │ "active"│
└────┬────┘ └────┬────┘
     │           │
     │           ▼
     │    ┌─────────┐
     │    │ Log     │
     │    │ Drift   │
     │    │ Event   │
     │    └────┬────┘
     │         │
     └─────────┼─────┐
               │     │
               ▼     ▼
        ┌─────────┐ ┌─────────┐
        │ Continue│ │ Monitor │
        │ Monitor │ │ for     │
        │         │ │ Resolution │
        └────┬────┘ └────┬────┘
             │            │
             │            ▼
             │    ┌─────────┐
             │    │ Drift   │
             │    │ Resolved│
             │    │ ?       │
             │    └────┬────┘
             │         │
             │    ┌─────┴─────┐
             │    │           │
             │    ▼           ▼
             │ ┌─────────┐ ┌─────────┐
             │ │   No    │ │   Yes   │
             │ │ Continue│ │ Set     │
             │ │ Monitor │ │ Status: │
             │ │         │ │ "resolved" │
             │ └────┬────┘ └────┬────┘
             │      │           │
             │      │           ▼
             │      │    ┌─────────┐
             │      │    │ Log     │
             │      │    │ Resolution │
             │      │    │ Event   │
             │      │    └────┬────┘
             │      │         │
             └──────┼─────────┼─────┐
                    │         │     │
                    ▼         ▼     ▼
             ┌─────────┐ ┌─────────┐ ┌─────────┐
             │ Return  │ │ Update  │ │ Continue│
             │ to      │ │ Record  │ │ Monitor │
             │ Normal  │ │ with    │ │         │
             │ State   │ │ Resolution │         │
             └─────────┘ └─────────┘ └─────────┘
```

## 🚀 User Interaction Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    User Interaction Flow                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Setup & Installation                                        │
│     ├── Install prerequisites (Go, MongoDB, kubectl)           │
│     ├── Configure DriftGuard (config.yaml)                     │
│     ├── Start MongoDB and DriftGuard                           │
│     └── Verify system health (/health endpoint)                │
│                                                                 │
│  2. Initial Deployment                                          │
│     ├── Create test Git repository with K8s manifests          │
│     ├── Deploy resources to Kubernetes cluster                 │
│     ├── Trigger initial drift analysis                         │
│     └── Verify no drift detected                               │
│                                                                 │
│  3. Create Test Drift                                           │
│     ├── Modify live resources (scale, change image, etc.)      │
│     ├── Trigger drift analysis                                 │
│     ├── Check active drift records                             │
│     └── Verify drift detection and classification              │
│                                                                 │
│  4. Resolve Drift                                               │
│     ├── Revert changes or apply Git manifests                  │
│     ├── Trigger drift analysis                                 │
│     ├── Check resolved drift records                           │
│     └── Verify resolution detection                            │
│                                                                 │
│  5. Monitor & Analyze                                           │
│     ├── Check statistics endpoint                              │
│     ├── Monitor Prometheus metrics                             │
│     ├── Set up alerts and dashboards                           │
│     └── Review drift patterns and trends                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 🔧 Configuration Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    Configuration Flow                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Server Configuration                                        │
│     ├── Port: 8080                                             │
│     ├── Timeouts: read/write/idle                              │
│     └── Logging levels                                         │
│                                                                 │
│  2. Database Configuration                                      │
│     ├── MongoDB host and port                                  │
│     ├── Database name                                          │
│     └── Connection timeout                                     │
│                                                                 │
│  3. Kubernetes Configuration                                    │
│     ├── Kubeconfig path                                        │
│     ├── Context selection                                       │
│     ├── Namespaces to monitor                                  │
│     ├── Resource types to watch                                │
│     └── System namespace filtering                             │
│                                                                 │
│  4. Git Configuration                                           │
│     ├── Repository URL                                         │
│     ├── Default branch                                         │
│     ├── Clone and pull timeouts                                │
│     └── Authentication (if needed)                             │
│                                                                 │
│  5. Drift Detection Configuration                               │
│     ├── Analysis interval                                       │
│     ├── Periodic detection enable/disable                      │
│     └── Drift classification rules                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 📊 Metrics & Monitoring

```
┌─────────────────────────────────────────────────────────────────┐
│                    Metrics & Monitoring                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Prometheus Metrics                                             │
│  ├── driftguard_drift_events_total                             │
│  ├── driftguard_resources_watched                              │
│  ├── driftguard_drift_score                                    │
│  ├── driftguard_sync_duration_seconds                          │
│  └── driftguard_api_request_duration_seconds                   │
│                                                                 │
│  Health Checks                                                  │
│  ├── /health - Basic service health                            │
│  ├── /ready - Kubernetes readiness probe                       │
│  └── /metrics - Prometheus metrics endpoint                    │
│                                                                 │
│  Logging                                                        │
│  ├── Structured JSON logging                                   │
│  ├── Emoji indicators for events                               │
│  ├── Drift detection events                                    │
│  ├── Resolution events                                         │
│  └── System health events                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
