# Kubernetes Voting App - Troubleshooting Guide

## Architecture Overview

This is a microservices voting application deployed on Kubernetes with the following components:

### Data Flow Architecture
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ Voting App  │───▶│   Redis     │───▶│   Worker    │───▶│ PostgreSQL  │───▶│ Result App  │
│  (Python)   │    │ (In-Memory) │    │   (.NET)    │    │ (Database)  │    │  (Node.js)  │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
      │                                                                            │
      ▼                                                                            ▼
┌─────────────┐                                                        ┌─────────────┐
│   User UI   │                                                        │ Results UI  │
│ Port: 30080 │                                                        │ Port: 30081 │
└─────────────┘                                                        └─────────────┘
```

### Kubernetes Services & Pods Architecture
```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    KUBERNETES CLUSTER                                    │
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                              VOTE-APP NAMESPACE                                 │   │
│  │                                                                                 │   │
│  │  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐          │   │
│  │  │  VOTING SERVICE │     │   REDIS SERVICE │     │    DB SERVICE    │          │   │
│  │  │   (NodePort)    │     │   (ClusterIP)   │     │   (ClusterIP)    │          │   │
│  │  │   Port: 30080   │     │   Port: 6379    │     │   Port: 5432     │          │   │
│  │  │                 │     │                 │     │                  │          │   │
│  │  │       │         │     │       │         │     │        │         │          │   │
│  │  └───────┼─────────┘     └───────┼─────────┘     └────────┼─────────┘          │   │
│  │          │                       │                        │                    │   │
│  │          ▼                       ▼                        ▼                    │   │
│  │  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐          │   │
│  │  │ voting-app-pod  │────▶│   redis-pod     │◀────│ postgres-pod    │          │   │
│  │  │                 │     │                 │     │                 │          │   │
│  │  │ Image: vote:v1  │     │ Image: redis    │     │ Image: postgres │          │   │
│  │  │ Port: 80        │     │ Port: 6379      │     │ Port: 5432      │          │   │
│  │  │                 │     │                 │     │                 │          │   │
│  │  │ ENV:            │     │                 │     │ ENV:            │          │   │
│  │  │ REDIS_HOST=redis│     │                 │     │ POSTGRES_USER   │          │   │
│  │  └─────────────────┘     └─────────────────┘     │ POSTGRES_DB     │          │   │
│  │                                  ▲               └─────────────────┘          │   │
│  │                                  │                        ▲                   │   │
│  │                                  │                        │                   │   │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐ │   │
│  │  │                    WORKER DEPLOYMENT                                    │ │   │
│  │  │                                                                         │ │   │
│  │  │  ┌─────────────────┐                                                   │ │   │
│  │  │  │ Init Container  │  Wait for Redis & DB to be ready                 │ │   │
│  │  │  │ (busybox)       │  ┌─────────────────────────────────────────────┐ │ │   │
│  │  │  └─────────────────┘  │ until nc -z redis 6379; do sleep 2; done   │ │ │   │
│  │  │           │            │ until nc -z db 5432; do sleep 2; done      │ │ │   │
│  │  │           ▼            └─────────────────────────────────────────────┘ │ │   │
│  │  │  ┌─────────────────┐                                                   │ │   │
│  │  │  │ worker-app-pod  │──────────────────────────────────────────────────┼─┘   │
│  │  │  │                 │                                                   │     │
│  │  │  │ Image: worker:v1│  ENV:                                             │     │
│  │  │  │                 │  REDIS_HOST=redis                                 │     │
│  │  │  │                 │  POSTGRES_HOST=db                                 │     │
│  │  │  │                 │  POSTGRES_USER=postgres                           │     │
│  │  │  └─────────────────┘                                                   │     │
│  │  └─────────────────────────────────────────────────────────────────────────┘     │
│  │                                                                                 │   │
│  │  ┌─────────────────┐     ┌─────────────────┐                                  │   │
│  │  │ RESULT SERVICE  │     │ result-app-pod  │                                  │   │
│  │  │   (NodePort)    │────▶│                 │──────────────────────────────────┘   │
│  │  │   Port: 30081   │     │ Image: result:v1│                                      │
│  │  │                 │     │ Port: 80        │                                      │
│  │  │                 │     │                 │                                      │
│  │  │                 │     │ ENV:            │                                      │
│  │  │                 │     │ POSTGRES_HOST=  │                                      │
│  │  │                 │     │ postgres-service│                                      │
│  │  └─────────────────┘     └─────────────────┘                                      │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────────┘

External Access:
┌─────────────┐    ┌─────────────┐
│   Browser   │    │   Browser   │
│             │    │             │
│ :30080      │    │ :30081      │
│ (Voting)    │    │ (Results)   │
└─────────────┘    └─────────────┘
```

### Service-to-Pod Mapping
```yaml
# How Services Connect to Pods via Selectors

voting-service (NodePort) ──────────▶ voting-app-pod
  selector: name=voting-app-pod           labels: name=voting-app-pod

redis (ClusterIP) ──────────────────▶ redis-pod  
  selector: name=redis-pod                labels: name=redis-pod

db (ClusterIP) ─────────────────────▶ postgres-pod
  selector: name=postgres-pod             labels: name=postgres-pod

result-service (NodePort) ──────────▶ result-app-pod
  selector: name=result-app-pod           labels: name=result-app-pod

worker-app-deployment ──────────────▶ worker-app-pod
  selector: app=demo-voting-app           labels: app=demo-voting-app
           tier=worker                           tier=worker
```

## How Services Interconnect

### Connection Flow with YAML Configuration

#### 1. **User → Voting App**
```yaml
# voting-app-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: voting-service
spec:
  type: NodePort                    # Exposes to external traffic
  selector:
    name: voting-app-pod           # Connects to voting-app-pod
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080               # External access port
```

#### 2. **Voting App → Redis**
```yaml
# voting-app-pod.yaml
env:
- name: REDIS_HOST
  value: "redis"                  # Points to redis service
- name: REDIS
  value: "redis"                  # Alternative env var

# redis-alias-service.yaml  
apiVersion: v1
kind: Service
metadata:
  name: redis                     # Service name matches env var
spec:
  selector:
    name: redis-pod              # Connects to redis-pod
  ports:
  - port: 6379
    targetPort: 6379
```

#### 3. **Worker → Redis & PostgreSQL**
```yaml
# worker-app-pod.yaml (Deployment)
spec:
  template:
    spec:
      initContainers:
      - name: wait-for-services
        command: ['sh', '-c']
        args:
        - |
          until nc -z redis 6379; do sleep 2; done      # Wait for Redis
          until nc -z db 5432; do sleep 2; done         # Wait for PostgreSQL
      
      containers:
      - name: worker-app
        env:
        - name: REDIS_HOST
          value: "redis"            # Connects to redis service
        - name: POSTGRES_HOST  
          value: "db"               # Connects to db service
```

#### 4. **Result App → PostgreSQL**
```yaml
# result-app-pod.yaml
env:
- name: POSTGRES_HOST
  value: "postgres-service"       # Points to postgres service
- name: POSTGRES_USER
  value: "postgres"
- name: POSTGRES_PASSWORD
  value: "postgres"

# postgres-alias-service.yaml
apiVersion: v1
kind: Service  
metadata:
  name: db                        # Service name for worker
spec:
  selector:
    name: postgres-pod           # Connects to postgres-pod
  ports:
  - port: 5432
    targetPort: 5432
```

#### 5. **User → Result App**
```yaml
# result-app-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: result-service
spec:
  type: NodePort                  # Exposes to external traffic
  selector:
    name: result-app-pod         # Connects to result-app-pod
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30081             # External access port
```

### Label-Selector Relationships

```yaml
# How Kubernetes matches Services to Pods

┌─────────────────────────────────────────────────────────────────┐
│                    SERVICE SELECTORS                            │
├─────────────────────────────────────────────────────────────────┤
│ voting-service:                                                 │
│   selector: { name: voting-app-pod }                           │
│                                                                 │
│ redis:                                                          │
│   selector: { name: redis-pod }                                │
│                                                                 │
│ db:                                                             │
│   selector: { name: postgres-pod }                             │
│                                                                 │
│ result-service:                                                 │
│   selector: { name: result-app-pod }                           │
│                                                                 │
│ worker-app-deployment:                                          │
│   selector: { app: demo-voting-app, tier: worker }            │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      POD LABELS                                 │
├─────────────────────────────────────────────────────────────────┤
│ voting-app-pod:                                                 │
│   labels: { name: voting-app-pod, app: demo-voting-app }       │
│                                                                 │
│ redis-pod:                                                      │
│   labels: { name: redis-pod, app: demo-voting-app }           │
│                                                                 │
│ postgres-pod:                                                   │
│   labels: { name: postgres-pod, app: demo-voting-app }        │
│                                                                 │
│ result-app-pod:                                                 │
│   labels: { name: result-app-pod, app: demo-voting-app }      │
│                                                                 │
│ worker-app-pod:                                                 │
│   labels: { app: demo-voting-app, tier: worker }              │
└─────────────────────────────────────────────────────────────────┘
```

### Environment Variable Connections

```yaml
# How apps find each other through environment variables

voting-app-pod:
  env:
    REDIS_HOST: "redis"          ──────▶ redis service ──────▶ redis-pod

worker-app-pod:  
  env:
    REDIS_HOST: "redis"          ──────▶ redis service ──────▶ redis-pod
    POSTGRES_HOST: "db"          ──────▶ db service ────────▶ postgres-pod

result-app-pod:
  env:
    POSTGRES_HOST: "postgres-service" ─▶ db service ────────▶ postgres-pod
```

### Network Communication Paths

```
External User ──(NodePort 30080)──▶ voting-service ──▶ voting-app-pod
                                           │
                                           ▼
                                    (REDIS_HOST=redis)
                                           │
                                           ▼
                                    redis service ──▶ redis-pod
                                           ▲
                                           │
                                    (REDIS_HOST=redis)
                                           │
                                    worker-app-pod ──(POSTGRES_HOST=db)──▶ db service ──▶ postgres-pod
                                                                                  ▲
                                                                                  │
                                                                        (POSTGRES_HOST=postgres-service)
                                                                                  │
External User ──(NodePort 30081)──▶ result-service ──▶ result-app-pod ─────────┘
```

## Components

### 1. **Voting App** (Frontend)
- **Image**: `kodekloud/examplevotingapp_vote:v1`
- **Purpose**: Web interface for users to vote (Cats vs Dogs)
- **Port**: 80 (exposed via NodePort 30080)
- **Dependencies**: Redis for storing votes

### 2. **Redis** (Cache)
- **Image**: `redis:alpine`
- **Purpose**: Temporary storage for incoming votes
- **Port**: 6379
- **Type**: ClusterIP service

### 3. **Worker** (Background Processor)
- **Image**: `kodekloud/examplevotingapp_worker:v1`
- **Purpose**: Processes votes from Redis to PostgreSQL
- **Type**: Deployment (for reliability)
- **Dependencies**: Both Redis and PostgreSQL

### 4. **PostgreSQL** (Database)
- **Image**: `postgres:9.4`
- **Purpose**: Persistent storage for processed votes
- **Port**: 5432
- **Database**: Contains `votes` table

### 5. **Result App** (Results Frontend)
- **Image**: `kodekloud/examplevotingapp_result:v1`
- **Purpose**: Displays voting results from database
- **Port**: 80 (exposed via NodePort 30081)
- **Dependencies**: PostgreSQL for reading results

## Issues Encountered & Solutions

### 🔴 Issue 1: Internal Server Error in Voting App

**Error**: 
```
redis.exceptions.ConnectionError: Error -3 connecting to redis:6379. 
Temporary failure in name resolution.
```

**Root Cause**: The voting app was hardcoded to connect to hostname `redis`, but our service was named `redis-service`.

**Solution**: Created an alias service named `redis`:
```yaml
# Service/redis-alias-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:
    name: redis-pod
  ports:
  - port: 6379
    targetPort: 6379
```

### 🔴 Issue 2: Worker CrashLoopBackOff

**Error**:
```
System.Net.Internals.SocketExceptionFactory+ExtendedSocketException: 
Resource temporarily unavailable
```

**Root Cause**: 
1. Worker couldn't resolve DNS names for services
2. Worker expected services named `redis` and `db`, not `redis-service` and `postgres-service`

**Solutions Applied**:

1. **Added Init Container** to wait for services:
```yaml
initContainers:
- name: wait-for-services
  image: busybox:1.35
  command: ['sh', '-c']
  args:
  - |
    until nc -z redis 6379; do sleep 2; done
    until nc -z db 5432; do sleep 2; done
```

2. **Created PostgreSQL alias service**:
```yaml
# Service/postgres-alias-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  selector:
    name: postgres-pod
  ports:
  - port: 5432
    targetPort: 5432
```

3. **Updated environment variables**:
```yaml
env:
- name: REDIS_HOST
  value: "redis"          # Changed from "redis-service"
- name: POSTGRES_HOST
  value: "db"             # Changed from "postgres-service"
```

### 🔴 Issue 3: Missing Database Table

**Error**:
```
Error performing query: error: relation "votes" does not exist
```

**Solution**: Added table creation to PostgreSQL startup:
```yaml
command: ["/bin/bash"]
args: ["-c", "docker-entrypoint.sh postgres & sleep 10 && PGPASSWORD=postgres psql -h localhost -U postgres -d postgres -c 'CREATE TABLE IF NOT EXISTS votes (id VARCHAR(255) NOT NULL UNIQUE, vote VARCHAR(255) NOT NULL);' && wait"]
```

### 🔴 Issue 4: Duplicate Services

**Issue**: Had both `db-service.yaml` and `postgres-service.yaml` pointing to same pod.

**Solution**: Removed duplicate `db-service.yaml` to avoid conflicts.

## Key Configuration Changes Made

### Environment Variables Fixed:
```yaml
# Voting App
env:
- name: REDIS_HOST
  value: "redis-service"
- name: REDIS
  value: "redis-service"      # Added this
- name: REDIS_URL
  value: "redis://redis-service:6379"  # Added this

# Worker App  
env:
- name: REDIS_HOST
  value: "redis"             # Changed from "redis-service"
- name: POSTGRES_HOST
  value: "db"                # Changed from "postgres-service"
```

### Service Aliases Created:
- `redis` → points to `redis-pod`
- `db` → points to `postgres-pod`

## Useful Kubernetes Commands for Troubleshooting

### 🔍 **Debugging Commands**

```bash
# Check all resources in namespace
kubectl get all -n vote-app

# Check pod status and restarts
kubectl get pods -n vote-app -o wide

# Check service endpoints
kubectl get endpoints -n vote-app

# Check service details
kubectl describe service <service-name> -n vote-app
```

### 📋 **Log Analysis**

```bash
# Check specific pod logs
kubectl logs <pod-name> -n vote-app

# Follow logs in real-time
kubectl logs <pod-name> -n vote-app --follow

# Check logs for specific container in multi-container pod
kubectl logs <pod-name> -c <container-name> -n vote-app

# Check init container logs
kubectl logs <pod-name> -c wait-for-services -n vote-app

# Get logs from deployment
kubectl logs -l tier=worker -n vote-app
```

### 🔧 **Testing Connectivity**

```bash
# Test service connectivity from within cluster
kubectl exec -it <pod-name> -n vote-app -- nslookup <service-name>

# Test Redis connectivity
kubectl exec -it redis-pod -n vote-app -- redis-cli ping

# Test PostgreSQL connectivity
kubectl exec -it postgres-pod -n vote-app -- psql -U postgres -d postgres -c "\dt"

# Check if table exists
kubectl exec -it postgres-pod -n vote-app -- psql -U postgres -d postgres -c "SELECT * FROM votes;"
```

### 🚀 **Deployment Commands**

```bash
# Apply all configurations
kubectl apply -f . -n vote-app

# Apply specific file
kubectl apply -f <filename>.yaml -n vote-app

# Delete and recreate pod
kubectl delete pod <pod-name> -n vote-app
kubectl apply -f <pod-file>.yaml -n vote-app

# Restart deployment
kubectl rollout restart deployment <deployment-name> -n vote-app

# Scale deployment
kubectl scale deployment <deployment-name> --replicas=2 -n vote-app
```

### 🌐 **Access Applications**

```bash
# Get service URLs (for minikube)
minikube service voting-service -n vote-app --url
minikube service result-service -n vote-app --url

# Port forwarding (alternative access method)
kubectl port-forward service/voting-service 8080:80 -n vote-app
kubectl port-forward service/result-service 8081:80 -n vote-app

# Check NodePort services
kubectl get services -n vote-app
```

### 🔄 **Recovery Commands**

```bash
# If worker keeps crashing
kubectl delete deployment worker-app-deployment -n vote-app
kubectl apply -f worker-app-pod.yaml -n vote-app

# If services are not working
kubectl delete service <service-name> -n vote-app
kubectl apply -f Service/<service-file>.yaml -n vote-app

# Complete restart of application
kubectl delete -f . -n vote-app
kubectl apply -f . -n vote-app
```

### 📊 **Monitoring Commands**

```bash
# Watch pod status in real-time
kubectl get pods -n vote-app --watch

# Check resource usage
kubectl top pods -n vote-app
kubectl top nodes

# Check events for troubleshooting
kubectl get events -n vote-app --sort-by='.lastTimestamp'

# Describe problematic pod
kubectl describe pod <pod-name> -n vote-app
```

## Application URLs

After successful deployment:

- **Voting Interface**: `http://<minikube-ip>:30080` or use `minikube service voting-service -n vote-app --url`
- **Results Interface**: `http://<minikube-ip>:30081` or use `minikube service result-service -n vote-app --url`

## Common Troubleshooting Tips

1. **Always check logs first**: `kubectl logs <pod-name> -n vote-app`
2. **Verify service endpoints**: `kubectl get endpoints -n vote-app`
3. **Check DNS resolution**: Services must be accessible by their names within the cluster
4. **Environment variables matter**: Applications often expect specific hostnames
5. **Init containers help**: Use them to wait for dependencies before starting main containers
6. **Namespace consistency**: Always use `-n vote-app` for all commands

## Success Indicators

✅ All pods show `Running` status  
✅ Worker logs show "Connected to db" and "Found redis"  
✅ Voting app accessible without Internal Server Error  
✅ Results app shows vote counts  
✅ No CrashLoopBackOff status on any pods  

## File Structure

```
voting-app-k8s/
├── Service/
│   ├── redis-service.yaml
│   ├── postgres-service.yaml
│   ├── voting-app-service.yaml
│   ├── result-app-service.yaml
│   ├── redis-alias-service.yaml      # Added for compatibility
│   └── postgres-alias-service.yaml   # Added for compatibility
├── voting-app-pod.yaml
├── redis-pod.yaml
├── postgres-pod.yaml
├── result-app-pod.yaml
├── worker-app-pod.yaml               # Deployment with init container
└── README.md                         # This file
```

## YAML File Wiring & Service Connections

### Complete File-to-File Wiring Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           YAML FILES INTERCONNECTION                                    │
└─────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐
│   voting-app-pod    │    │ redis-alias-service │    │     redis-pod       │
│      .yaml          │───▶│      .yaml          │───▶│      .yaml          │
│                     │    │                     │    │                     │
│ env:                │    │ metadata:           │    │ metadata:           │
│ - REDIS_HOST=redis  │    │   name: redis       │    │   name: redis-pod   │
│ - REDIS=redis       │    │ selector:           │    │ labels:             │
│                     │    │   name: redis-pod   │    │   name: redis-pod   │
└─────────────────────┘    └─────────────────────┘    └─────────────────────┘

┌─────────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐
│ worker-app-pod.yaml │    │postgres-alias-service│    │   postgres-pod      │
│   (Deployment)      │───▶│      .yaml          │───▶│      .yaml          │
│                     │    │                     │    │                     │
│ env:                │    │ metadata:           │    │ metadata:           │
│ - REDIS_HOST=redis  │────┐ name: db           │    │   name: postgres-pod│
│ - POSTGRES_HOST=db  │    │ selector:           │    │ labels:             │
│                     │    │   name: postgres-pod│    │   name: postgres-pod│
└─────────────────────┘    └─────────────────────┘    └─────────────────────┘
          │
          └─────────────────────────────────────────────────────────────────┐
                                                                            │
┌─────────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐│
│ result-app-pod.yaml │    │ result-app-service  │    │voting-app-service   ││
│                     │───▶│      .yaml          │    │      .yaml          ││
│                     │    │                     │    │                     ││
│ env:                │    │ metadata:           │    │ metadata:           ││
│ - POSTGRES_HOST=    │    │   name: result-service│  │   name: voting-service│
│   postgres-service  │    │ selector:           │    │ selector:           ││
│                     │    │   name: result-app-pod│  │   name: voting-app-pod│
└─────────────────────┘    └─────────────────────┘    └─────────────────────┘│
                                                                            │
                           ┌─────────────────────┐                         │
                           │   voting-app-pod    │◀────────────────────────┘
                           │      .yaml          │
                           │                     │
                           │ metadata:           │
                           │   name: voting-app-pod│
                           │ labels:             │
                           │   name: voting-app-pod│
                           └─────────────────────┘
```

### Detailed YAML Wiring Connections

#### 1. **Voting App Connection Chain**
```yaml
# File: voting-app-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: voting-app-pod
  labels:
    name: voting-app-pod        # ◀── Service selector targets this
spec:
  containers:
  - name: voting-app
    env:
    - name: REDIS_HOST
      value: "redis"            # ──▶ Points to redis service name
    - name: REDIS
      value: "redis"            # ──▶ Points to redis service name

# File: voting-app-service.yaml  
apiVersion: v1
kind: Service
metadata:
  name: voting-service          # ◀── External access name
spec:
  type: NodePort
  selector:
    name: voting-app-pod        # ──▶ Matches pod label above
  ports:
  - port: 80
    nodePort: 30080            # ◀── External port for users

# File: redis-alias-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: redis                   # ◀── Matches REDIS_HOST env var
spec:
  selector:
    name: redis-pod            # ──▶ Points to redis pod

# File: redis-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis-pod
  labels:
    name: redis-pod            # ◀── Service selector targets this
```

#### 2. **Worker App Connection Chain**
```yaml
# File: worker-app-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker-app-deployment
spec:
  selector:
    matchLabels:
      app: demo-voting-app
      tier: worker              # ◀── Deployment manages pods with these labels
  template:
    metadata:
      labels:
        app: demo-voting-app
        tier: worker            # ◀── Pod gets these labels
    spec:
      initContainers:
      - name: wait-for-services
        args:
        - |
          until nc -z redis 6379; do sleep 2; done     # ──▶ Waits for redis service
          until nc -z db 5432; do sleep 2; done        # ──▶ Waits for db service
      containers:
      - name: worker-app
        env:
        - name: REDIS_HOST
          value: "redis"        # ──▶ Points to redis service
        - name: POSTGRES_HOST
          value: "db"           # ──▶ Points to db service

# File: redis-alias-service.yaml (already shown above)
# File: postgres-alias-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: db                      # ◀── Matches POSTGRES_HOST env var
spec:
  selector:
    name: postgres-pod         # ──▶ Points to postgres pod

# File: postgres-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-pod
  labels:
    name: postgres-pod         # ◀── Service selector targets this
```

#### 3. **Result App Connection Chain**
```yaml
# File: result-app-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: result-app-pod
  labels:
    name: result-app-pod       # ◀── Service selector targets this
spec:
  containers:
  - name: result-app
    env:
    - name: POSTGRES_HOST
      value: "postgres-service" # ──▶ Points to postgres service (legacy name)

# File: result-app-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: result-service         # ◀── External access name
spec:
  type: NodePort
  selector:
    name: result-app-pod       # ──▶ Matches pod label above
  ports:
  - port: 80
    nodePort: 30081           # ◀── External port for users

# Note: result-app still uses "postgres-service" but connects to same postgres-pod
# through the db service due to service aliasing
```

### Service Name Resolution Matrix

```yaml
┌─────────────────────┬─────────────────────┬─────────────────────┬─────────────────────┐
│    POD/CONTAINER    │   ENVIRONMENT VAR   │   SERVICE NAME      │    TARGET POD       │
├─────────────────────┼─────────────────────┼─────────────────────┼─────────────────────┤
│ voting-app-pod      │ REDIS_HOST=redis    │ redis               │ redis-pod           │
│ voting-app-pod      │ REDIS=redis         │ redis               │ redis-pod           │
├─────────────────────┼─────────────────────┼─────────────────────┼─────────────────────┤
│ worker-app-pod      │ REDIS_HOST=redis    │ redis               │ redis-pod           │
│ worker-app-pod      │ POSTGRES_HOST=db    │ db                  │ postgres-pod        │
├─────────────────────┼─────────────────────┼─────────────────────┼─────────────────────┤
│ result-app-pod      │ POSTGRES_HOST=      │ db (alias resolves) │ postgres-pod        │
│                     │ postgres-service    │                     │                     │
├─────────────────────┼─────────────────────┼─────────────────────┼─────────────────────┤
│ External Users      │ N/A                 │ voting-service      │ voting-app-pod      │
│ External Users      │ N/A                 │ result-service      │ result-app-pod      │
└─────────────────────┴─────────────────────┴─────────────────────┴─────────────────────┘
```

### File Dependencies & Load Order

```yaml
# Recommended kubectl apply order:

1. PODS FIRST (Create the targets):
   kubectl apply -f redis-pod.yaml -n vote-app
   kubectl apply -f postgres-pod.yaml -n vote-app
   kubectl apply -f voting-app-pod.yaml -n vote-app
   kubectl apply -f result-app-pod.yaml -n vote-app

2. SERVICES SECOND (Create the networking):
   kubectl apply -f Service/redis-alias-service.yaml -n vote-app
   kubectl apply -f Service/postgres-alias-service.yaml -n vote-app
   kubectl apply -f Service/voting-app-service.yaml -n vote-app
   kubectl apply -f Service/result-app-service.yaml -n vote-app

3. DEPLOYMENTS LAST (Create managed workloads):
   kubectl apply -f worker-app-pod.yaml -n vote-app

# Or apply all at once:
   kubectl apply -f . -n vote-app
```

### Cross-Reference Table: Files ↔ Services ↔ Pods

```yaml
┌─────────────────────────┬─────────────────────────┬─────────────────────────┐
│       YAML FILE         │     SERVICE CREATED     │      POD TARGETED       │
├─────────────────────────┼─────────────────────────┼─────────────────────────┤
│ redis-pod.yaml          │ (none - just pod)       │ redis-pod               │
│ redis-alias-service.yaml│ redis                   │ redis-pod               │
├─────────────────────────┼─────────────────────────┼─────────────────────────┤
│ postgres-pod.yaml       │ (none - just pod)       │ postgres-pod            │
│ postgres-alias-service  │ db                      │ postgres-pod            │
├─────────────────────────┼─────────────────────────┼─────────────────────────┤
│ voting-app-pod.yaml     │ (none - just pod)       │ voting-app-pod          │
│ voting-app-service.yaml │ voting-service          │ voting-app-pod          │
├─────────────────────────┼─────────────────────────┼─────────────────────────┤
│ result-app-pod.yaml     │ (none - just pod)       │ result-app-pod          │
│ result-app-service.yaml │ result-service          │ result-app-pod          │
├─────────────────────────┼─────────────────────────┼─────────────────────────┤
│ worker-app-pod.yaml     │ (none - deployment)     │ worker-app-pod-xxxxx    │
│ (Deployment creates pods│                         │ (auto-generated name)   │
│  automatically)         │                         │                         │
└─────────────────────────┴─────────────────────────┴─────────────────────────┘
```

### Environment Variable → Service → Pod Resolution Flow

```yaml
# How Kubernetes resolves connections:

voting-app-pod container:
  REDIS_HOST="redis" 
    ↓
  Kubernetes DNS lookup: redis.vote-app.svc.cluster.local
    ↓  
  Service: redis (ClusterIP: 10.100.184.52)
    ↓
  Service selector: name=redis-pod
    ↓
  Pod: redis-pod (IP: 10.244.0.67)
    ↓
  Container port: 6379

worker-app-pod container:
  POSTGRES_HOST="db"
    ↓
  Kubernetes DNS lookup: db.vote-app.svc.cluster.local  
    ↓
  Service: db (ClusterIP: 10.105.25.147)
    ↓
  Service selector: name=postgres-pod
    ↓
  Pod: postgres-pod (IP: 10.244.0.68)
    ↓
  Container port: 5432
```

This guide should help you understand the application architecture, common issues, and how to troubleshoot problems in your Kubernetes voting application.