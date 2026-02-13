# PipesHub AI - GCP Deployment Guide

Complete guide for deploying PipesHub AI on Google Cloud Platform with Vertex AI for LLM inference, ClickUp for task context, and Google Docs for documentation.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites](#2-prerequisites)
3. [GCP Services to Provision](#3-gcp-services-to-provision)
4. [Step-by-Step Deployment](#4-step-by-step-deployment)
5. [Vertex AI Configuration](#5-vertex-ai-configuration)
6. [Connecting Google Docs](#6-connecting-google-docs)
7. [Connecting ClickUp](#7-connecting-clickup)
8. [DNS, SSL & Networking](#8-dns-ssl--networking)
9. [Environment Variables Reference](#9-environment-variables-reference)
10. [Monitoring & Operations](#10-monitoring--operations)
11. [Cost Estimates](#11-cost-estimates)

---

## 1. Architecture Overview

```
                        ┌─────────────────────────────────────┐
                        │          Cloud Load Balancer         │
                        │        (HTTPS termination)           │
                        └──────────┬──────────┬───────────────┘
                                   │          │
                    ┌──────────────▼──┐  ┌────▼──────────────┐
                    │  pipeshub.your   │  │ connector.your    │
                    │  domain.com:3000 │  │ domain.com:8088   │
                    └──────────┬──────┘  └────┬──────────────┘
                               │              │
                    ┌──────────▼──────────────▼───────────────┐
                    │         GKE Cluster / GCE VM            │
                    │                                          │
                    │  ┌─────────────────────────────────┐    │
                    │  │       pipeshub-ai container       │    │
                    │  │  - Node.js API (port 3000)       │    │
                    │  │  - Connector svc (port 8088)     │    │
                    │  │  - Indexing svc (port 8091)      │    │
                    │  │  - Query svc (port 8000)         │    │
                    │  │  - Docling parser (port 8081)    │    │
                    │  └────────────┬──────────────────────┘    │
                    └───────────────┼──────────────────────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                          │
   ┌──────▼───────┐  ┌─────────────▼──────────┐  ┌───────────▼────────┐
   │  Vertex AI    │  │   Managed Databases     │  │  Self-hosted on    │
   │  (Gemini LLM) │  │                         │  │  GCE / GKE         │
   │  (Embeddings) │  │  - Memorystore (Redis)  │  │                    │
   │               │  │  - MongoDB Atlas or     │  │  - ArangoDB        │
   └───────────────┘  │    self-hosted Mongo    │  │  - Qdrant          │
                      │                         │  │  - etcd            │
                      └─────────────────────────┘  │  - Kafka+Zookeeper │
                                                   └────────────────────┘
```

### What runs where

| Component | GCP Service | Notes |
|---|---|---|
| **PipesHub AI app** | GKE (Helm) or GCE (Docker Compose) | Main application container |
| **LLM inference** | Vertex AI | Gemini models via `vertexAI` provider |
| **Embeddings** | Vertex AI | `text-embedding-005` or similar |
| **Redis** | Memorystore for Redis | Managed, HA |
| **MongoDB** | MongoDB Atlas on GCP *or* self-hosted on GCE | Atlas recommended for managed experience |
| **ArangoDB** | GCE VM (self-hosted) | No managed GCP equivalent |
| **Qdrant** | GCE VM (self-hosted) or Qdrant Cloud | Vector DB |
| **Kafka + Zookeeper** | GCE VM (self-hosted) or Confluent Cloud | Event streaming |
| **etcd** | GCE VM (self-hosted) | Config store (or switch `KV_STORE_TYPE=redis`) |
| **Load Balancer** | Cloud Load Balancer | HTTPS termination |
| **DNS** | Cloud DNS | Domain management |
| **Container Registry** | Artifact Registry | Store Docker images |

---

## 2. Prerequisites

- A GCP project with billing enabled
- `gcloud` CLI installed and authenticated
- `kubectl` installed (if using GKE)
- `helm` v3 installed (if using GKE)
- Docker installed locally (to build/push images)
- A registered domain name (for HTTPS - required by PipesHub)
- The PipesHub repository cloned:
  ```bash
  git clone https://github.com/pipeshub-ai/pipeshub-ai.git
  cd pipeshub-ai
  ```

---

## 3. GCP Services to Provision

### 3.1 Enable APIs

```bash
gcloud services enable \
  container.googleapis.com \
  compute.googleapis.com \
  artifactregistry.googleapis.com \
  aiplatform.googleapis.com \
  redis.googleapis.com \
  dns.googleapis.com \
  iap.googleapis.com \
  secretmanager.googleapis.com \
  cloudresourcemanager.googleapis.com
```

### 3.2 Set project variables

```bash
export PROJECT_ID="your-gcp-project-id"
export REGION="us-central1"
export ZONE="us-central1-a"
export CLUSTER_NAME="pipeshub-cluster"

gcloud config set project $PROJECT_ID
gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE
```

---

## 4. Step-by-Step Deployment

You have two deployment paths. **Option A (GKE + Helm)** is recommended for production. **Option B (GCE + Docker Compose)** is simpler for a quick setup.

---

### Option A: GKE + Helm (Recommended for Production)

#### 4A.1 Create Artifact Registry repository

```bash
gcloud artifacts repositories create pipeshub \
  --repository-format=docker \
  --location=$REGION \
  --description="PipesHub container images"
```

#### 4A.2 Build and push the Docker image

```bash
# Authenticate Docker with Artifact Registry
gcloud auth configure-docker $REGION-docker.pkg.dev

# Build the image
docker build -t $REGION-docker.pkg.dev/$PROJECT_ID/pipeshub/pipeshub-ai:latest .

# Push the image
docker push $REGION-docker.pkg.dev/$PROJECT_ID/pipeshub/pipeshub-ai:latest
```

#### 4A.3 Create a GKE cluster

```bash
gcloud container clusters create $CLUSTER_NAME \
  --zone $ZONE \
  --num-nodes 3 \
  --machine-type e2-standard-8 \
  --disk-size 100 \
  --enable-ip-alias \
  --workload-pool=$PROJECT_ID.svc.id.goog

# Get credentials
gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE
```

**Minimum node specs:** The `pipeshub-ai` container needs ~10 GB RAM. With all infrastructure services, plan for at least 3x `e2-standard-8` (8 vCPU, 32 GB RAM each) nodes.

#### 4A.4 Create Memorystore for Redis

```bash
gcloud redis instances create pipeshub-redis \
  --size=2 \
  --region=$REGION \
  --redis-version=redis_7_0 \
  --network=default

# Get the IP
REDIS_HOST=$(gcloud redis instances describe pipeshub-redis \
  --region=$REGION --format='value(host)')
echo "Redis host: $REDIS_HOST"
```

#### 4A.5 Deploy infrastructure services on GKE

ArangoDB, Qdrant, Kafka, Zookeeper, etcd, and MongoDB don't have direct GCP managed equivalents (except MongoDB Atlas). Deploy them as part of the Helm chart or separately.

**For MongoDB**, you can either:
- Use the built-in MongoDB in the Helm chart (included by default)
- Use MongoDB Atlas (recommended for production) - create a cluster at https://cloud.mongodb.com and get the connection URI

#### 4A.6 Configure Helm values

Create a `gcp-values.yaml` file:

```yaml
replicaCount: 1

image:
  repository: <REGION>-docker.pkg.dev/<PROJECT_ID>/pipeshub/pipeshub-ai
  pullPolicy: Always
  tag: latest

service:
  type: ClusterIP

ingress:
  enabled: true
  className: "gce"  # GKE Ingress controller
  annotations:
    kubernetes.io/ingress.global-static-ip-name: "pipeshub-ip"
    networking.gke.io/managed-certificates: "pipeshub-cert"
  hosts:
    - host: pipeshub.yourdomain.com
      paths:
        - path: /
          pathType: ImplementationSpecific
          port: 3000
          serviceName: pipeshub-ai
    - host: pipeshub-connector.yourdomain.com
      paths:
        - path: /
          pathType: ImplementationSpecific
          port: 8088
          serviceName: pipeshub-ai
  tls:
    - secretName: pipeshub-tls
      hosts:
        - pipeshub.yourdomain.com
    - secretName: pipeshub-connector-tls
      hosts:
        - pipeshub-connector.yourdomain.com

resources:
  limits:
    cpu: 8
    memory: 10Gi
  requests:
    cpu: 4
    memory: 6Gi

persistence:
  enabled: true
  storageClass: "standard-rwo"  # GKE default SSD storage class
  size: 20Gi

config:
  nodeEnv: "production"
  logLevel: "info"
  allowedOrigins: "https://pipeshub.yourdomain.com"
  connectorPublicBackend: "https://pipeshub-connector.yourdomain.com"
  frontendPublicUrl: "https://pipeshub.yourdomain.com"

secretKey: "<generate-a-random-32-char-string>"

# Override Redis to use Memorystore
# In the Helm chart, set redis.enabled=false and pass env vars directly
redis:
  auth:
    enabled: false
  persistence:
    storageClass: "standard-rwo"

# ArangoDB
arango:
  auth:
    rootPassword: "<strong-password>"
  persistence:
    storageClass: "standard-rwo"
    size: 20Gi

# etcd
etcd:
  persistence:
    storageClass: "standard-rwo"
    size: 5Gi

# Qdrant
qdrant:
  apiKey: "<strong-api-key>"
  persistence:
    storageClass: "standard-rwo"
    size: 20Gi

# MongoDB (if using built-in)
mongodb:
  persistence:
    storageClass: "standard-rwo"
```

#### 4A.7 Deploy with Helm

```bash
cd deployment/helm

helm install pipeshub-ai ./pipeshub-ai \
  -n pipeshub --create-namespace \
  -f gcp-values.yaml
```

#### 4A.8 Reserve static IP and configure DNS

```bash
# Reserve a global static IP
gcloud compute addresses create pipeshub-ip --global

# Get the IP address
gcloud compute addresses describe pipeshub-ip --global --format='value(address)'
```

Point your DNS records to this IP:
- `pipeshub.yourdomain.com` -> A record -> static IP
- `pipeshub-connector.yourdomain.com` -> A record -> static IP

#### 4A.9 Set up Google-managed SSL certificate

```yaml
# Create a ManagedCertificate resource (apply via kubectl)
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: pipeshub-cert
  namespace: pipeshub
spec:
  domains:
    - pipeshub.yourdomain.com
    - pipeshub-connector.yourdomain.com
```

```bash
kubectl apply -f managed-cert.yaml -n pipeshub
```

---

### Option B: GCE + Docker Compose (Simpler)

#### 4B.1 Create a GCE VM

```bash
gcloud compute instances create pipeshub-vm \
  --zone=$ZONE \
  --machine-type=e2-standard-8 \
  --boot-disk-size=100GB \
  --boot-disk-type=pd-ssd \
  --image-family=ubuntu-2404-lts-amd64 \
  --image-project=ubuntu-os-cloud \
  --tags=pipeshub-server
```

#### 4B.2 SSH into the VM and install Docker

```bash
gcloud compute ssh pipeshub-vm --zone=$ZONE

# On the VM:
sudo apt-get update
sudo apt-get install -y docker.io docker-compose-plugin
sudo usermod -aG docker $USER
# Log out and back in for group change to take effect
```

#### 4B.3 Clone and configure

```bash
git clone https://github.com/pipeshub-ai/pipeshub-ai.git
cd pipeshub-ai/deployment/docker-compose

# Create .env from template
cp env.template .env
```

Edit `.env`:
```bash
NODE_ENV=production
SECRET_KEY=<generate-a-random-32-char-string>
FRONTEND_PUBLIC_URL=https://pipeshub.yourdomain.com
ARANGO_PASSWORD=<strong-password>
REDIS_PASSWORD=<strong-password>
MONGO_USERNAME=admin
MONGO_PASSWORD=<strong-password>
QDRANT_API_KEY=<strong-api-key>
KV_STORE_TYPE=etcd
```

#### 4B.4 Open firewall ports

```bash
gcloud compute firewall-rules create allow-pipeshub \
  --allow tcp:3000,tcp:8088 \
  --target-tags pipeshub-server \
  --description "PipesHub frontend and connector ports"
```

#### 4B.5 Start services

```bash
docker compose -f docker-compose.prod.yml -p pipeshub-ai up -d
```

#### 4B.6 Set up HTTPS with a reverse proxy

Since PipesHub requires HTTPS, set up Nginx + Certbot on the VM:

```bash
sudo apt-get install -y nginx certbot python3-certbot-nginx
```

Create `/etc/nginx/sites-available/pipeshub`:
```nginx
server {
    server_name pipeshub.yourdomain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}

server {
    server_name pipeshub-connector.yourdomain.com;

    location / {
        proxy_pass http://localhost:8088;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/pipeshub /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

# Get SSL certs
sudo certbot --nginx -d pipeshub.yourdomain.com -d pipeshub-connector.yourdomain.com
```

---

## 5. Vertex AI Configuration

PipesHub has **built-in Vertex AI support** for both LLM inference and embeddings. The providers `vertexAI` are registered in the codebase as valid providers for both `LLMProvider` and `EmbeddingProvider`.

However, the current codebase's `get_generator_model()` and `get_embedding_model()` functions in `backend/python/app/utils/aimodels.py` do not yet have an explicit `vertexAI` code path implemented (no `if provider == LLMProvider.VERTEX_AI.value` block). This means you have **two options**:

### Option 1: Use Gemini Provider (Recommended - Works Today)

The `gemini` provider is fully implemented and uses the Google AI API, which works with the same Gemini models available through Vertex AI. This is the simplest path.

In the PipesHub UI (Settings > AI Models):
- **Provider**: `gemini`
- **Model**: `gemini-2.0-flash` (or `gemini-2.5-pro`, etc.)
- **API Key**: Your Google AI API key (get from https://aistudio.google.com/apikey)

For embeddings:
- **Provider**: `gemini`
- **Model**: `text-embedding-004`
- **API Key**: Same Google AI API key

### Option 2: Use OpenAI-Compatible Endpoint (Vertex AI Native)

Vertex AI exposes an OpenAI-compatible endpoint. You can use the `openAICompatible` provider to point directly at Vertex AI:

1. **Set up authentication** on your GCE/GKE instance:
   ```bash
   # If on GKE with Workload Identity:
   gcloud iam service-accounts create pipeshub-vertex \
     --display-name="PipesHub Vertex AI"

   gcloud projects add-iam-policy-binding $PROJECT_ID \
     --member="serviceAccount:pipeshub-vertex@$PROJECT_ID.iam.gserviceaccount.com" \
     --role="roles/aiplatform.user"

   # Generate an access token for API calls
   ACCESS_TOKEN=$(gcloud auth print-access-token)
   ```

2. In the PipesHub UI (Settings > AI Models):

   **For LLM:**
   - **Provider**: `openAICompatible`
   - **Model**: `google/gemini-2.0-flash` (or `google/gemini-2.5-pro`)
   - **API Key**: Your `gcloud auth print-access-token` output (needs periodic refresh) or set up a proxy
   - **Endpoint**: `https://${REGION}-aiplatform.googleapis.com/v1/projects/${PROJECT_ID}/locations/${REGION}/endpoints/openapi`

   **For Embeddings:**
   - **Provider**: `openAICompatible`
   - **Model**: `text-embedding-005`
   - **API Key**: Same access token
   - **Endpoint**: Same Vertex AI endpoint

### Option 3: Add Native Vertex AI Support (Requires Code Change)

If you want first-class Vertex AI integration with service account auth (no token refresh), you would add a code block in `backend/python/app/utils/aimodels.py`. The `langchain-google-vertexai` package is already in the dependencies (`pyproject.toml`). This would involve adding:

```python
elif provider == LLMProvider.VERTEX_AI.value:
    from langchain_google_vertexai import ChatVertexAI

    return ChatVertexAI(
        model_name=model_name,
        temperature=0.2,
        max_output_tokens=MAX_OUTPUT_TOKENS,
        project=configuration.get("projectId", os.getenv("GOOGLE_CLOUD_PROJECT")),
        location=configuration.get("region", os.getenv("GOOGLE_CLOUD_REGION", "us-central1")),
    )
```

And similarly for embeddings:
```python
elif provider == EmbeddingProvider.VERTEX_AI.value:
    from langchain_google_vertexai import VertexAIEmbeddings

    return VertexAIEmbeddings(
        model_name=model_name,
        project=configuration.get("projectId", os.getenv("GOOGLE_CLOUD_PROJECT")),
        location=configuration.get("region", os.getenv("GOOGLE_CLOUD_REGION", "us-central1")),
    )
```

On GKE with Workload Identity, this would authenticate automatically via the service account, requiring no API keys.

### Recommended Vertex AI Models

| Use Case | Model | Notes |
|---|---|---|
| **LLM (general)** | `gemini-2.0-flash` | Fast, cost-effective |
| **LLM (complex reasoning)** | `gemini-2.5-pro` | Best quality |
| **Embeddings** | `text-embedding-005` | Latest Google embedding model |
| **OCR/Multimodal** | `gemini-2.0-flash` | Supports vision for scanned PDFs |

---

## 6. Connecting Google Docs

PipesHub has a **built-in Google Docs connector** (part of the Google Workspace connector group). It supports both personal sync and team sync via OAuth 2.0.

### 6.1 Set up Google Cloud OAuth Credentials

1. Go to **Google Cloud Console** > **APIs & Services** > **Credentials**
2. Click **Create Credentials** > **OAuth 2.0 Client ID**
3. Application type: **Web application**
4. Authorized redirect URIs:
   ```
   https://pipeshub-connector.yourdomain.com/connectors/oauth/callback/Docs
   ```
5. Note the **Client ID** and **Client Secret**

### 6.2 Enable Google APIs

```bash
gcloud services enable \
  docs.googleapis.com \
  drive.googleapis.com \
  sheets.googleapis.com \
  slides.googleapis.com \
  forms.googleapis.com \
  calendar-json.googleapis.com
```

### 6.3 Configure in PipesHub UI

1. Log into PipesHub at `https://pipeshub.yourdomain.com`
2. Go to **Settings** > **Connectors**
3. Find **Google Workspace** > **Docs**
4. Click **Configure**
5. Enter your **OAuth Client ID** and **Client Secret**
6. Click **Connect** - you'll be redirected to Google for authorization
7. Grant the requested permissions (Documents, Drive read access)
8. Once connected, configure sync:
   - **Sync Strategy**: Scheduled (every 60 min) or Manual
   - **Scope**: Personal (your docs) or Team (org-wide, requires Google Workspace admin)

### Available Google Workspace Connectors

All these work the same way (OAuth + Client ID/Secret):

| Connector | Data Indexed |
|---|---|
| **Google Drive** | All files in Drive (PDFs, Docs, Sheets, etc.) |
| **Google Docs** | Document content with formatting |
| **Google Sheets** | Spreadsheet data |
| **Google Slides** | Presentation content |
| **Google Forms** | Form questions and responses |
| **Google Calendar** | Calendar events |
| **Google Meet** | Meeting metadata |
| **Gmail** | Email content |

> **Tip**: If you want to index all Google Workspace content, use the **Google Drive** connector - it covers all file types including Docs, Sheets, and Slides. Use the individual connectors (Docs, Sheets, etc.) only if you need granular control.

---

## 7. Connecting ClickUp

PipesHub has `CLICKUP` defined in its connector types enum (`backend/python/app/connectors/enums/enums.py`), but the **ClickUp connector is not yet fully implemented** as a production connector. It's listed as a recognized connector type for future development.

### Current Options for ClickUp Integration

#### Option A: Use the Web Connector (Available Now)

PipesHub has a **Web connector** that can crawl and index web pages. You can use it to index your ClickUp workspace:

1. Go to **Settings** > **Connectors** > **Web**
2. Configure it to crawl your ClickUp pages
3. This gives you search over ClickUp content, but not real-time sync

#### Option B: Build a ClickUp Connector (Development Required)

The PipesHub connector framework makes it straightforward to add new connectors. Based on the existing patterns (see `CONNECTOR_INTEGRATION_PLAYBOOK.md`), a ClickUp connector would involve:

1. **Register the connector** in `backend/python/app/connectors/core/registry/connector.py`:
   ```python
   @ConnectorBuilder("ClickUp")\
       .in_group("ClickUp")\
       .with_description("Sync tasks, docs, and spaces from ClickUp")\
       .with_categories(["Project Management"])\
       .with_scopes([ConnectorScope.TEAM.value])\
       .with_auth([
           AuthBuilder.type(AuthType.API_TOKEN).fields([
               AuthField(
                   name="apiToken",
                   display_name="API Token",
                   placeholder="pk_...",
                   description="ClickUp API Token from Settings > Apps",
                   field_type="PASSWORD",
                   max_length=8000,
                   is_secret=True
               )
           ])
       ])\
       ...
   ```

2. **Create the connector source** at `backend/python/app/connectors/sources/clickup/connector.py` implementing:
   - `BaseConnector` class
   - Task/doc fetching from ClickUp API v2
   - Incremental sync via ClickUp webhooks

3. **ClickUp API setup**:
   - Get an API token from ClickUp: **Settings** > **Apps** > **API Token**
   - Or set up OAuth: https://clickup.com/api/developer-portal/authentication/

#### Option C: Use ClickUp's Built-in Integrations

While waiting for a native connector, you can:
- **Export ClickUp data to Google Docs/Drive** and index via the Google Drive connector
- **Use ClickUp's Slack integration** and index via the Slack connector
- **Push ClickUp updates to a Confluence space** and index via Confluence connector

---

## 8. DNS, SSL & Networking

### Required DNS Records

| Record | Type | Value |
|---|---|---|
| `pipeshub.yourdomain.com` | A | GCE VM IP or GKE Load Balancer IP |
| `pipeshub-connector.yourdomain.com` | A | Same IP |

### Why Two Domains?

PipesHub needs a public-facing connector URL for:
- OAuth callback redirects (Google, Slack, etc.)
- Webhook endpoints for real-time data sync
- API access from external services

### SSL/TLS (Required)

PipesHub enforces strict security. Browsers will show a white screen over HTTP. Options:
- **GKE**: Google-managed certificates (see section 4A.9)
- **GCE**: Certbot with Nginx (see section 4B.6)
- **Cloudflare**: Point DNS through Cloudflare for automatic SSL

---

## 9. Environment Variables Reference

Complete list of environment variables for GCP deployment:

### Core Settings

| Variable | Description | Example |
|---|---|---|
| `NODE_ENV` | Environment mode | `production` |
| `LOG_LEVEL` | Log verbosity | `info` |
| `SECRET_KEY` | Encryption key for tokens/secrets | Random 32+ char string |
| `ALLOWED_ORIGINS` | CORS allowed origins | `https://pipeshub.yourdomain.com` |
| `FRONTEND_PUBLIC_URL` | Public URL of the frontend | `https://pipeshub.yourdomain.com` |
| `CONNECTOR_PUBLIC_BACKEND` | Public URL of connector service | `https://pipeshub-connector.yourdomain.com` |

### Database Connections

| Variable | Description | Example |
|---|---|---|
| `ARANGO_URL` | ArangoDB connection URL | `http://arango:8529` |
| `ARANGO_PASSWORD` | ArangoDB root password | Strong password |
| `MONGO_URI` | MongoDB connection string | `mongodb://admin:pass@mongodb:27017/?authSource=admin` |
| `REDIS_HOST` | Redis hostname | Memorystore IP or `redis` |
| `REDIS_PORT` | Redis port | `6379` |
| `REDIS_PASSWORD` | Redis password | Password or empty |
| `QDRANT_HOST` | Qdrant hostname | `qdrant` |
| `QDRANT_API_KEY` | Qdrant API key | Strong API key |
| `ETCD_URL` | etcd connection URL | `http://etcd:2379` |
| `KAFKA_BROKERS` | Kafka broker addresses | `kafka-1:9092` |

### KV Store

| Variable | Description | Example |
|---|---|---|
| `KV_STORE_TYPE` | Backend for config store | `etcd` or `redis` |

### Performance Tuning

| Variable | Description | Default |
|---|---|---|
| `MAX_CONCURRENT_PARSING` | Parallel document parsing | `5` |
| `MAX_CONCURRENT_INDEXING` | Parallel indexing operations | `7` |
| `MAX_REQUESTS_PER_MINUTE` | API rate limit | `500` |
| `ACCESS_TOKEN_EXPIRY` | JWT access token lifetime | `24h` |
| `REFRESH_TOKEN_EXPIRY` | JWT refresh token lifetime | `720h` |

### Ollama (Optional - Local LLM)

| Variable | Description | Default |
|---|---|---|
| `OLLAMA_API_URL` | Ollama server URL | `http://host.docker.internal:11434` |

---

## 10. Monitoring & Operations

### Health Checks

```bash
# Node.js API
curl https://pipeshub.yourdomain.com/api/v1/health

# Connector service
curl https://pipeshub-connector.yourdomain.com/health
```

### Logs

```bash
# GKE
kubectl logs -f deployment/pipeshub-ai -n pipeshub

# GCE Docker Compose
docker compose -f docker-compose.prod.yml -p pipeshub-ai logs -f pipeshub-ai
```

### GCP Monitoring

- Enable **Cloud Monitoring** and **Cloud Logging** for your GKE cluster or GCE VM
- Set up alerts for:
  - Container restart count > 3
  - Memory usage > 85% on `pipeshub-ai` pod
  - Qdrant disk usage > 80%
  - Kafka consumer lag > 1000

### Backups

| Service | Backup Strategy |
|---|---|
| **MongoDB** | `mongodump` on schedule or Atlas automated backups |
| **ArangoDB** | `arangodump` on schedule |
| **Qdrant** | Snapshot API: `POST /collections/{name}/snapshots` |
| **Redis** | RDB snapshots (Memorystore handles this automatically) |
| **etcd** | `etcdctl snapshot save` |
| **Persistent disks** | GCE disk snapshots on schedule |

---

## 11. Cost Estimates

Rough monthly estimates for a small-to-medium deployment (USD):

| Service | Spec | Estimated Cost/mo |
|---|---|---|
| GKE cluster (3x e2-standard-8) | 24 vCPU, 96 GB RAM | ~$500 |
| *or* GCE VM (1x e2-standard-8) | 8 vCPU, 32 GB RAM | ~$170 |
| Memorystore Redis (2 GB) | Basic tier | ~$55 |
| Persistent SSDs (100 GB total) | pd-ssd | ~$17 |
| Cloud Load Balancer | Per-rule + traffic | ~$20 |
| Vertex AI (Gemini Flash) | ~1M tokens/day | ~$10-50 |
| Vertex AI (Embeddings) | ~500K tokens/day | ~$5-20 |
| MongoDB Atlas (M10) | If using Atlas | ~$60 |
| **Total (GKE path)** | | **~$650-750** |
| **Total (GCE path)** | | **~$300-400** |

> Vertex AI pricing is usage-based. Gemini 2.0 Flash is very cost-effective at $0.10/1M input tokens and $0.40/1M output tokens.

---

## Quick Start Checklist

- [ ] GCP project created with billing enabled
- [ ] Required APIs enabled
- [ ] GKE cluster or GCE VM provisioned
- [ ] Memorystore Redis instance created
- [ ] Docker image built and pushed to Artifact Registry
- [ ] DNS records configured (2 A records)
- [ ] SSL/TLS certificates provisioned
- [ ] Helm chart deployed (GKE) or Docker Compose started (GCE)
- [ ] Health checks passing
- [ ] AI Models configured in UI (Gemini for LLM + Embeddings)
- [ ] Google Workspace OAuth credentials created
- [ ] Google Docs connector connected and syncing
- [ ] ClickUp integration approach chosen
