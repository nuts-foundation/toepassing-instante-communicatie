# 🏗️ Healthcare Communication Platform - Deployment Architecture

**Based on the OZO reference implementation, this guide provides the recommended deployment architecture for hackathon participants building healthcare communication bridges.**

---

## Core Deployment Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Load Balancer                              │
│                         (NGINX Ingress Controller)                      │
└─────────────────────────┬───────────────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│   Element Web   │ │  Matrix Synapse │ │   Data Server   │
│   (Frontend)    │ │  (Homeserver)   │ │ (FHIR/Custom)   │
│                 │ │                 │ │                │
│ - Web Client    │ │ - Matrix API    │ │ - Data API      │
│ - E2E Encryption│ │ - Federation    │ │ - Subscriptions │
│ - Voice/Video   │ │ - Room/User Mgmt│ │ - Search        │
└─────────────────┘ └─────────┬───────┘ └─────────┬───────┘
                              │                   │
                              │                   │
          ┌───────────────────┼───────────────────┼───────────────────┐
          │                   │                   │                   │
          ▼                   ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│      ma1sd      │ │  Platform Bridge│ │  ma1sd Bridge   │ │  Your Platform  │
│ Identity Server │ │    Service      │ │   (Auth/3PID)   │ │   Integration   │
│                 │ │                 │ │                 │ │                 │
│ - 3PID Lookup   │ │ - Data ↔ Matrix │ │ - Keycloak/mCSD │ │ - Custom Logic  │
│ - Directory     │ │ - Room Creation │ │ - User Search   │ │ - Webhooks     │
│ - Federation    │ │ - Message Relay │ │ - Authentication│ │ - API Calls    │
└─────────┬───────┘ └─────────┬───────┘ └─────────┬───────┘ └─────────┬───────┘
          │                   │                   │                   │
          └───────────────────┼───────────────────┼───────────────────┘
                              │                   │
                              ▼                   ▼
                    ┌─────────────────┐ ┌─────────────────┐
                    │   PostgreSQL    │ │    Your Data    │
                    │    Database     │ │     Backend     │
                    │                 │ │                 │
                    │ - Synapse Data  │ │ - Keycloak      │
                    │ - Platform Data │ │ - Custom DB     │
                    │ - ma1sd Data    │ │ - mCSD/FHIR     │
                    │ - User Sessions │ │ - Legacy System │
                    └─────────────────┘ └─────────────────┘
```

---

## Simplified Hackathon Deployment Options

### Option 1: Docker Compose (Quickstart)

**Best for:** Initial development, quick prototyping, single-machine demos

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Matrix Homeserver
  synapse:
    image: matrixdotorg/synapse:latest
    ports: 
      - "8008:8008"
    volumes:
      - synapse_data:/data
    environment:
      SYNAPSE_SERVER_NAME: localhost
      SYNAPSE_REPORT_STATS: "no"
    
  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: synapse
      POSTGRES_USER: synapse_user
      POSTGRES_PASSWORD: synapse_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      
  # HAPI FHIR Server
  fhir-server:
    image: hapiproject/hapi:latest
    ports: 
      - "8080:8080"
    environment:
      spring.datasource.url: jdbc:postgresql://postgres:5432/fhir
      spring.datasource.username: synapse_user
      spring.datasource.password: synapse_password
      hapi.fhir.subscription.websocket_enabled: true
      hapi.fhir.subscription.resthook_enabled: true
    depends_on:
      - postgres
    
  # ma1sd Identity Server
  ma1sd:
    image: ma1uta/ma1sd:latest
    ports:
      - "8090:8090"
    volumes:
      - ./ma1sd.yaml:/etc/ma1sd/ma1sd.yaml:ro
    depends_on:
      - postgres
    
  # Your Bridge Service
  your-bridge:
    build: ./bridge
    ports: 
      - "3000:3000"
    environment:
      MATRIX_URL: http://synapse:8008
      FHIR_URL: http://fhir-server:8080/fhir
      MA1SD_URL: http://ma1sd:8090
      DATABASE_URL: postgresql://synapse_user:synapse_password@postgres:5432/bridge
    depends_on:
      - synapse
      - data-server
      - ma1sd

volumes:
  synapse_data:
  postgres_data:
```

**Quick Start Commands:**
```bash
# Clone and setup
git clone <your-bridge-repo>
cd healthcare-matrix-bridge

# Start all services
docker-compose up -d

# Check service health
curl http://localhost:8008/_matrix/client/versions  # Synapse
curl http://localhost:8080/fhir/metadata          # Data Server
curl http://localhost:3000/health                 # Platform Bridge
```

### Option 2: Kubernetes (Production-like)

**Best for:** Scalable demos, production testing, team collaboration

```yaml
# kubernetes-deployment.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: healthcare-matrix

---
# PostgreSQL Database
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: healthcare-matrix
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: "synapse"
        - name: POSTGRES_USER
          value: "synapse_user" 
        - name: POSTGRES_PASSWORD
          value: "synapse_password"
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc

---
# Matrix Synapse
apiVersion: apps/v1
kind: Deployment
metadata:
  name: synapse
  namespace: healthcare-matrix
spec:
  replicas: 1
  selector:
    matchLabels:
      app: synapse
  template:
    metadata:
      labels:
        app: synapse
    spec:
      containers:
      - name: synapse
        image: matrixdotorg/synapse:latest
        ports:
        - containerPort: 8008
        env:
        - name: SYNAPSE_SERVER_NAME
          value: "matrix.your-domain.com"
        - name: SYNAPSE_REPORT_STATS
          value: "no"
        volumeMounts:
        - name: synapse-config
          mountPath: /data
      volumes:
      - name: synapse-config
        persistentVolumeClaim:
          claimName: synapse-pvc

---
# Data Server (FHIR example - adapt for your platform)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-server
  namespace: healthcare-matrix
spec:
  replicas: 1
  selector:
    matchLabels:
      app: data-server
  template:
    metadata:
      labels:
        app: data-server
    spec:
      containers:
      - name: data-server
        image: hapiproject/hapi:latest  # Replace with your platform's image
        ports:
        - containerPort: 8080
        env:
        - name: spring.datasource.url
          value: "jdbc:postgresql://postgres:5432/platform_data"
        - name: spring.datasource.username
          value: "synapse_user"
        - name: spring.datasource.password
          value: "synapse_password"
        - name: hapi.fhir.subscription.resthook_enabled
          value: "true"

---
# Platform Bridge Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: platform-bridge
  namespace: healthcare-matrix
spec:
  replicas: 2
  selector:
    matchLabels:
      app: platform-bridge
  template:
    metadata:
      labels:
        app: platform-bridge
    spec:
      containers:
      - name: bridge
        image: your-platform-bridge:latest
        ports:
        - containerPort: 3000
        env:
        - name: MATRIX_URL
          value: "http://synapse:8008"
        - name: DATA_SERVER_URL
          value: "http://data-server:8080/api"  # Adapt to your platform's API
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10

---
# Services
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: healthcare-matrix
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432

---
apiVersion: v1
kind: Service
metadata:
  name: synapse
  namespace: healthcare-matrix
spec:
  selector:
    app: synapse
  ports:
  - port: 8008
    targetPort: 8008

---
apiVersion: v1
kind: Service
metadata:
  name: fhir-server
  namespace: healthcare-matrix
spec:
  selector:
    app: fhir-server
  ports:
  - port: 8080
    targetPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: bridge-service
  namespace: healthcare-matrix
spec:
  selector:
    app: bridge-service
  ports:
  - port: 3000
    targetPort: 3000
  type: LoadBalancer
```

**Kubernetes Deployment Commands:**
```bash
# Deploy to Kubernetes
kubectl apply -f kubernetes-deployment.yaml

# Check pod status
kubectl get pods -n healthcare-matrix

# Access services
kubectl port-forward -n healthcare-matrix svc/synapse 8008:8008
kubectl port-forward -n healthcare-matrix svc/fhir-server 8080:8080
kubectl port-forward -n healthcare-matrix svc/bridge-service 3000:3000

# Scale bridge service
kubectl scale deployment bridge-service --replicas=3 -n healthcare-matrix
```

---

## Component Responsibilities

### Matrix Synapse (Port 8008)
**Purpose:** Core Matrix homeserver providing communication infrastructure

**Key Responsibilities:**
- Matrix client-server API implementation
- Room and user management
- Message persistence and routing
- Federation with other Matrix homeservers
- Application service integration for bridges
- End-to-end encryption support

**Health Check:** `GET /_matrix/client/versions`

**Configuration:** 
- Application service registration for FHIR bridge
- Database connection to PostgreSQL
- Media upload handling
- Rate limiting and security policies

### HAPI FHIR Server (Port 8080)
**Purpose:** FHIR R4 compliant healthcare data server

**Key Responsibilities:**
- FHIR R4 REST API implementation
- Healthcare resource storage (Patient, Practitioner, CareTeam, etc.)
- FHIR subscriptions (REST-hook webhooks)
- Search and query capabilities
- FHIR validation and conformance checking

**Health Check:** `GET /fhir/metadata`

**Configuration:**
- PostgreSQL database backend
- Subscription webhooks to bridge service
- CORS settings for web clients
- Resource validation rules

### ma1sd Identity Server (Port 8090)
**Purpose:** Matrix identity server for 3PID resolution

**Key Responsibilities:**
- Third-party identifier (3PID) to Matrix ID resolution
- Directory search capabilities
- Identity verification workflows
- Federation support for cross-domain identity lookup

**Health Check:** `GET /_matrix/identity/versions`

**Configuration:**
- REST backend integration for custom identity sources
- Database storage for identity mappings
- Email/phone verification workflows

### Your Bridge Service (Port 3000)
**Purpose:** Custom bridge connecting your platform to Matrix/FHIR

**Key Responsibilities:**
- Bidirectional translation between FHIR and Matrix
- User identity mapping and synchronization
- Room/space creation and management
- Message routing and transformation logic
- Healthcare-specific business logic

**Health Check:** `GET /health`

**Configuration:**
- Matrix application service registration
- FHIR server integration
- Platform-specific user management
- Webhook endpoints for real-time sync

### ma1sd Bridge (Port 3003)
**Purpose:** Identity backend integration for ma1sd

**Key Responsibilities:**
- Backend identity system integration
- 3PID search delegation to your platform
- Authentication delegation
- Platform-specific user lookup and validation

**Health Check:** `GET /health`

**Configuration:**
- Backend system credentials (Keycloak, mCSD, database)
- User attribute mapping
- Authentication flow configuration

---

## Network Flow Patterns

### User Registration Flow
```
User Registration Request
        ↓
Element Web Client
        ↓
Matrix Synapse ──→ ma1sd Identity Server
        ↓                    ↓
Matrix Account      ma1sd Bridge ──→ Your Backend System
   Created                ↓
        ↓            Identity Verified
User can access Matrix rooms
```

**Process Steps:**
1. User attempts to register via Element Web
2. Synapse requests identity verification from ma1sd
3. ma1sd delegates to ma1sd bridge
4. Bridge validates user against your backend (Keycloak/mCSD/database)
5. If valid, Matrix account is created
6. User gains access to appropriate care team spaces

### Message Sending Flow
```
User sends message in Element Web
        ↓
Matrix Synapse (stores message)
        ↓
FHIR Bridge Service (Application Service)
        ↓
HAPI FHIR Server (creates Communication resource)
        ↓
Your Platform (receives FHIR notification)
```

**Process Steps:**
1. Healthcare professional sends message in Matrix room
2. Synapse stores message and forwards to FHIR bridge
3. Bridge extracts room context and user identity
4. Bridge creates FHIR Communication resource
5. Your platform receives FHIR resource for integration

### Care Team Synchronization Flow
```
Care Team Update in Your Platform
        ↓
Your Bridge Service (detects change)
        ↓
HAPI FHIR Server (creates/updates CareTeam)
        ↓
Matrix Synapse (creates/updates Space)
        ↓
Team Members Invited to Matrix Space
```

**Process Steps:**
1. Care team composition changes in your platform
2. Bridge detects change via webhook/polling
3. Bridge updates FHIR CareTeam resource
4. Bridge creates/updates corresponding Matrix space
5. Bridge invites team members with appropriate permissions

---

## Data Persistence Strategy

### PostgreSQL Database Schema

**Primary Databases:**
- `synapse_db`: Matrix homeserver data
  - Events, rooms, users, device keys
  - Message history and media references
  - Application service transactions

- `fhir_db`: HAPI FHIR server data  
  - FHIR resources and search indices
  - Subscription configurations
  - Resource versioning and history

- `ma1sd_db`: Identity server data
  - 3PID to Matrix ID mappings
  - Directory search cache
  - Identity verification sessions

- `bridge_db`: Bridge service data
  - User identity mappings
  - Room to FHIR resource associations
  - Synchronization state and logs

**File Storage Requirements:**
- Matrix media uploads (images, documents, voice messages)
- SSL/TLS certificates for HTTPS
- Application configuration files
- Log files and audit trails

### Backup and Recovery Strategy

**Database Backups:**
```bash
# Automated daily backups
pg_dump synapse_db > synapse_backup_$(date +%Y%m%d).sql
pg_dump fhir_db > fhir_backup_$(date +%Y%m%d).sql
pg_dump ma1sd_db > ma1sd_backup_$(date +%Y%m%d).sql
pg_dump bridge_db > bridge_backup_$(date +%Y%m%d).sql
```

**Media Backups:**
- Regular sync of Matrix media store
- Configuration file versioning
- Log rotation and archival

---

## Security Considerations

### Transport Layer Security

**HTTPS/TLS Configuration:**
- All external traffic encrypted with TLS 1.2+
- Matrix federation requires valid TLS certificates
- Client-server API exclusively over HTTPS
- Internal service communication secured

**Certificate Management:**
- Automated certificate provisioning with cert-manager
- Let's Encrypt integration for public domains
- Certificate rotation and renewal monitoring

### Authentication and Authorization

**Matrix Authentication:**
- OAuth2/OIDC integration via MAS
- Matrix access tokens for API access
- Application service authentication for bridges
- End-to-end encryption key management

**Healthcare Identity Validation:**
- UZI/URA number verification for practitioners
- Email verification for patients and related persons
- Role-based access control (RBAC)
- Multi-factor authentication (MFA) support

### Network Security

**Internal Communication:**
- Service-to-service authentication
- Network segmentation and policies
- Internal DNS resolution
- Service mesh integration (optional)

**External Access:**
- Firewall rules and port restrictions
- DDoS protection and rate limiting
- API gateway for external integrations
- Audit logging for all access

### Data Protection

**Healthcare Data Compliance:**
- GDPR compliance for EU deployments
- HIPAA considerations for US deployments
- Data encryption at rest and in transit
- Audit trails for all data access

**Secret Management:**
- Kubernetes secrets for sensitive data
- External secret management integration
- Regular credential rotation
- Principle of least privilege access

---

## Monitoring and Health Checks

### Service Health Endpoints

**Matrix Synapse:**
```bash
curl http://synapse:8008/_matrix/client/versions
# Response: {"versions": ["r0.6.1", "v1.1", "v1.2", ...]}
```

**HAPI FHIR Server:**
```bash
curl http://fhir-server:8080/fhir/metadata
# Response: FHIR CapabilityStatement resource
```

**ma1sd Identity Server:**
```bash
curl http://ma1sd:8090/_matrix/identity/versions
# Response: {"versions": ["v1", "v2"]}
```

**Bridge Service:**
```bash
curl http://bridge:3000/health
# Response: {"status": "healthy", "services": {...}}
```

### Key Performance Metrics

**Application Metrics:**
- Message throughput (messages/second)
- User registration rate (users/hour)
- FHIR resource creation rate (resources/minute)
- Bridge processing latency (milliseconds)
- Error rates and failure modes

**Infrastructure Metrics:**
- Database connection pool utilization
- Memory and CPU usage per service
- Network latency between services
- Storage utilization and growth
- SSL certificate expiration dates

**Healthcare Specific Metrics:**
- Care team creation/modification rate
- Cross-organization message volume
- User authentication success/failure rates
- Compliance audit trail completeness

### Monitoring Stack Integration

**Prometheus Metrics:**
```yaml
# Example ServiceMonitor for bridge service
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: bridge-metrics
spec:
  selector:
    matchLabels:
      app: bridge-service
  endpoints:
  - port: metrics
    path: /metrics
```

**Grafana Dashboards:**
- Matrix homeserver performance
- FHIR server utilization
- Bridge service health
- Healthcare workflow metrics

**Alerting Rules:**
- Service unavailability alerts
- High error rate notifications
- Resource exhaustion warnings
- Security incident detection

---

## Resource Requirements and Scaling

### Minimum Requirements (Development)

**Hardware:**
- **CPU:** 2 cores minimum, 4 cores recommended
- **RAM:** 4 GB minimum, 8 GB recommended  
- **Storage:** 10 GB for basic setup, 50 GB for extended testing
- **Network:** 100 Mbps for local development

**Software:**
- Docker and Docker Compose
- OR Kubernetes cluster (minikube/kind for local)
- PostgreSQL 12+ or compatible database
- Modern web browser for Element Web

### Recommended Requirements (Demo/Testing)

**Hardware:**
- **CPU:** 4 cores minimum, 8 cores for better performance
- **RAM:** 8 GB minimum, 16 GB recommended
- **Storage:** 50 GB minimum, 100 GB for extended demos
- **Network:** 1 Gbps for multi-user testing

**Infrastructure:**
- Kubernetes cluster with 3+ nodes
- Load balancer for external access
- Persistent storage (block or file)
- SSL/TLS certificate management

### Production Scale (Reference)

**Hardware:**
- **CPU:** 8+ cores for high availability
- **RAM:** 16+ GB with room for growth
- **Storage:** 500+ GB with backup strategy
- **Network:** Multi-gigabit with redundancy

**Architecture:**
- Multi-zone Kubernetes deployment
- Database clustering and replication
- CDN for media distribution
- External monitoring and alerting
- Disaster recovery procedures

### Scaling Strategies

**Horizontal Scaling:**
```yaml
# Scale bridge service replicas
kubectl scale deployment bridge-service --replicas=5

# Scale Synapse with workers
# (requires advanced Synapse configuration)
```

**Vertical Scaling:**
```yaml
# Increase resource limits
resources:
  limits:
    cpu: "2000m"
    memory: "4Gi"
  requests:
    cpu: "1000m" 
    memory: "2Gi"
```

**Database Optimization:**
- Connection pooling configuration
- Read replica setup for FHIR queries
- Database sharding for large deployments
- Query optimization and indexing

---

## Quick Start Deployment Scripts

### Docker Compose Quick Start

```bash
#!/bin/bash
# quick-start-docker.sh

echo "🏥 Healthcare Matrix Bridge - Docker Compose Setup"

# Create project directory
mkdir healthcare-matrix-bridge
cd healthcare-matrix-bridge

# Download docker-compose.yml
curl -o docker-compose.yml https://raw.githubusercontent.com/your-repo/docker-compose.yml

# Download configuration files
mkdir -p config
curl -o config/ma1sd.yaml https://raw.githubusercontent.com/your-repo/ma1sd.yaml
curl -o config/synapse.yaml https://raw.githubusercontent.com/your-repo/synapse.yaml

# Start services
echo "Starting services..."
docker-compose up -d

# Wait for services to start
echo "Waiting for services to initialize..."
sleep 30

# Health checks
echo "🔍 Checking service health..."
echo "Matrix Synapse:" 
curl -s http://localhost:8008/_matrix/client/versions | jq -r '.versions[0]' || echo "❌ Failed"

echo "FHIR Server:"
curl -s http://localhost:8080/fhir/metadata | jq -r '.software.name' || echo "❌ Failed" 

echo "Bridge Service:"
curl -s http://localhost:3000/health | jq -r '.status' || echo "❌ Failed"

echo "✅ Setup complete! Access Element Web at: http://localhost:8080/element"
```

### Kubernetes Quick Start

```bash
#!/bin/bash
# quick-start-k8s.sh

echo "🏥 Healthcare Matrix Bridge - Kubernetes Setup"

# Create namespace
kubectl create namespace healthcare-matrix

# Apply all configurations
kubectl apply -f https://raw.githubusercontent.com/your-repo/k8s-manifests.yaml

# Wait for deployments
echo "Waiting for deployments to be ready..."
kubectl wait --for=condition=available --timeout=300s deployment --all -n healthcare-matrix

# Port forward for local access
echo "Setting up port forwarding..."
kubectl port-forward -n healthcare-matrix svc/synapse 8008:8008 &
kubectl port-forward -n healthcare-matrix svc/fhir-server 8080:8080 &  
kubectl port-forward -n healthcare-matrix svc/bridge-service 3000:3000 &

echo "✅ Kubernetes deployment complete!"
echo "📋 Service URLs:"
echo "  Matrix Synapse: http://localhost:8008"
echo "  FHIR Server: http://localhost:8080/fhir"
echo "  Bridge Service: http://localhost:3000"

# Show pod status
kubectl get pods -n healthcare-matrix
```

---

## Troubleshooting Guide

### Common Issues and Solutions

**1. Matrix Synapse Connection Refused**
```bash
# Check if Synapse is running
docker ps | grep synapse
kubectl get pods -l app=synapse

# Check Synapse logs
docker logs synapse_container
kubectl logs -l app=synapse

# Common cause: Database not ready
# Solution: Ensure PostgreSQL is running and accessible
```

**2. FHIR Server 500 Errors**
```bash
# Check FHIR server logs
docker logs fhir-server_container
kubectl logs -l app=fhir-server

# Common cause: Database connection issues
# Solution: Verify PostgreSQL connection and schema
```

**3. Bridge Service Can't Connect to Matrix**
```bash
# Test Matrix connectivity from bridge
docker exec bridge_container curl http://synapse:8008/_matrix/client/versions

# Common cause: Application service registration missing
# Solution: Ensure app service config is loaded in Synapse
```

**4. Identity Resolution Failing**
```bash
# Test ma1sd health
curl http://localhost:8090/_matrix/identity/versions

# Test bridge identity service
curl -X POST http://localhost:3003/_ma1sd/backend/api/v1/identity/single \
  -H "Content-Type: application/json" \
  -d '{"lookup":{"medium":"email","address":"test@example.com"}}'

# Common cause: Backend integration misconfigured
# Solution: Check backend credentials and connectivity
```

### Performance Optimization

**Database Tuning:**
```sql
-- PostgreSQL optimization for Matrix/FHIR workloads
ALTER SYSTEM SET shared_buffers = '256MB';
ALTER SYSTEM SET effective_cache_size = '1GB';
ALTER SYSTEM SET random_page_cost = 1.1;
ALTER SYSTEM SET max_connections = 200;
SELECT pg_reload_conf();
```

**Matrix Synapse Optimization:**
```yaml
# homeserver.yaml optimizations
database:
  args:
    cp_min: 5
    cp_max: 20
    
caches:
  global_factor: 2.0
  
media_store_path: "/data/media"
max_upload_size: 50M
```

**Bridge Service Optimization:**
```javascript
// Connection pooling for bridge service
const pool = new Pool({
  host: 'postgres',
  port: 5432,
  database: 'bridge',
  user: 'bridge_user',
  password: 'bridge_password',
  max: 20, // Maximum pool size
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});
```

This deployment architecture guide provides hackathon participants with comprehensive options for deploying their healthcare communication platforms, from simple Docker Compose setups to production-ready Kubernetes deployments.