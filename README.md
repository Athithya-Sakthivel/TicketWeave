# TicketWeave

## TicketWeave is a support-triage agent built around two constraints: `inference cost` and `agent safety`.

### It receives customer messages over WebSocket, classifies every message before any tool runs, and routes the request to the appropriate support team via a deterministic intent-to-team map. Refunds, credits, pickup scheduling, and other customer-impacting operations are intentionally excluded — the agent's job is to assist human decision-making, not to act autonomously.

#### The result runs on **~$23/month of infrastructure with zero inbound ports**, secured by a **5-layer architecture** spanning edge (Cloudflare Tunnel + WAF), application (OIDC + short-TTL JWT), container (non-root + Trivy), network (private subnets + least-privilege IAM), and data (SSM Parameter Store + RDS encryption).

**Core stack:** `Amazon ECS` · `Cloudflare Tunnel` · `FastAPI` · `LangGraph` · `DSPy` · `Amazon Bedrock (Llama 3 8B)` · `FastMCP` · `PostgreSQL` · `S3` · `DynamoDB` · `RDS` · `OWASP ZAP`

---

## At a glance

| Metric                                    |            Result |
| ----------------------------------------- | ----------------: |
| **Infrastructure cost (staging)**         |    **~$23/month** |
| **Infrastructure cost (production)**      |    **~$38/month** |
| **Inline policy retrieval latency**       |         **<5 ms** |
| **DAST critical findings**                |             **0** |
| **DAST high findings**                    |             **0** |
| **Inbound application ports**             |             **0** |
| **JWT lifetime**                          |    **15 minutes** |
| **Team assignment**                       | **Deterministic** |

> **Measurement note:** The figures above reflect the architecture, configuration, and recorded scan/cost estimates in this repository. Actual AWS costs, latency, and security-scan results will vary with region, workload, traffic, and configuration.

---

![TicketWeave workflow](src/offline/images/agentops.gif)


### Notes on the demo

The GIF shows a two-turn escalation: the customer asks about returning a pair of shoes, is told the return window has closed, replies that the item arrived damaged, and confirms escalation.

**1. Return-window math is done in Python, not by the LLM.**  
The agent computes `delivery_date + return_window_days` and injects it as plain text (`OPEN until 2026-05-24` or `CLOSED since 2026-05-05`). The model only echoes the pre-computed status. This keeps hallucination risk on factual fields near zero.

**2. Confirmations and escalations bypass the LLM entirely.**  
"yes escalate" carries no new intent. Rather than pay for a second inference call, the agent strips confirmation words, assembles the ticket from graph state, and routes to `service_center` with a 2-hour SLA. The v1 limitation: the summary on this path uses current-turn state only, so a pure confirmation can fall back to a generic string. A one-turn lookback is planned for v2.

**3. What the LLM does *not* decide**

- **Team assignment** is a hard-coded `INTENT → TEAM` table.
- **Priority** is derived from the urgency score (`≥9 → critical`, `≥7 → high`, else `medium`).
- **Return-window status** is computed in Python — for the demo order (Nike Air Zoom Running Shoes, delivered 2026-04-25, 10-day window) the status is `CLOSED since 2026-05-05` — and injected as plain text.
- **Order filtering** narrows the order list to the ones the customer actually mentioned, before the prompt is built.

---

## How the Agent Is Constrained

Four LangGraph nodes. Full implementation notes in [docs/agent_service.md](docs/agent_service.md).

1. **Guardrail classifier** — DSPy-compiled triage program (Llama 3 8B, temperature 0) classifies safety, intent, urgency, and sentiment. Compiled from 50 labeled examples using `BootstrapFewShot`. Every message is classified **before** any context is fetched or any tool is called. Unsafe → immediate human escalation.
2. **Context gatherer** — Fetches customer profile and last 5 orders (with product names) from the MCP server. Skips gracefully if no customer identifier is present.
3. **Conversational agent** — LLM with two tools: `search_policies` (inline RAG) and `create_ticket`. Policy questions get an instant answer; issues needing human action produce a structured ticket with a concise summary and suggested action.
4. **Human escalate** — Skips the LLM entirely; creates a high-priority ticket via MCP for messages the guardrail rejects.


| Intent | Team |
|--------|------|
| `return_request` · `cancellation_request` · `wrong_item_delivered` | `order_fulfillment` |
| `refund_status` · `payment_issue` | `payments` |
| `late_delivery` · `delivery_issue` | `logistics` |
| `damaged_product` · `defective_product` | `service_center` |
| `account_issue` · `complaint` | `senior_support` |
| `general_inquiry` (and any unmapped intent) | `general_support` |


---

## System Architecture

![Architecture diagram](https://github.com/user-attachments/assets/8a7acac1-bf67-44e4-b832-b08c29924fd4)

**Networking flow:**
```
Internet → Cloudflare Tunnel → ECS Host (host network mode) → Agent Service → MCP Server → RDS
```

- **Zero inbound ports:** Security groups have no ingress rules; all traffic enters via Cloudflare Tunnel
- **No load balancer:** Cloudflare Tunnel terminates on each EC2 host, eliminating ALB/NLB costs
- **Private subnets:** RDS PostgreSQL deployed in private subnets, accessible only from ECS hosts

Full details in [docs/architecture.md](docs/architecture.md).

---

## Key Design Decisions

| Decision | Why |
|----------|-----|
| **Cloudflare Tunnel over ALB/WAF/NAT** | Eliminates $62/month in networking costs, zero inbound ports |
| **ECS on ARM (t4g) over Fargate** | 86% compute cost reduction ($142 → $20/month) — Graviton is cheaper than x86, and ECS managed instances handle provisioning, OS patching, and lifecycle management |
| **Inline RAG over OpenSearch** | Sufficient at this corpus size (~60 chunks), $190/month savings, sub-5ms search latency |
| **Never act autonomously** | The agent cannot refund, credit, or schedule pickups. Those tools were deliberately removed from the MCP server. |
| **3 MCP tools instead of 9** | Cutting wallet credits, refund eligibility, and pickup scheduling removed an entire class of failure — the LLM no longer has a tool that can move money. Aligns with Microsoft's MCP research on tool-space interference ([source](https://www.microsoft.com/en-us/research/blog/tool-space-interference-in-the-mcp-era-designing-for-agent-compatibility-at-scale/)). |
| **Deterministic routing** | A hard-coded `INTENT → TEAM` table guarantees predictable ticket assignment. |
| **DSPy guardrail first** | Every message is classified **before** any context is fetched or any tool is called. |

---

## Cost-aware inference

To keep average Bedrock spend low, several branches avoid the LLM entirely:

- Policy lookups are answered directly from RAG chunks rather than re-summarized.
- Return-window dates are pre-computed in Python and injected into the prompt as plain text.
- Orders are filtered to the relevant ones before the prompt is built.
- Confirmations ("yes", "ok", "go ahead") and escalations bypass the LLM; the ticket is assembled from existing graph state.

## State & persistence

All conversation state is checkpointed to PostgreSQL after every node via `AsyncPostgresSaver` — the agent survives restarts and resumes any in-flight conversation.

## MCP server

Three tools exposed over FastMCP: `lookup_customer`, `get_recent_orders`, `create_ticket`. It owns **no decision logic** — all routing, summarization, and policy decisions stay in the agent. Details in [docs/mcp_server.md](docs/mcp_server.md).

## Inline RAG

Policy documents are pre-embedded with Bedrock Titan v2, stored as a single ~1.3 MB JSON file in S3, loaded once at startup, and searched with in-process NumPy cosine similarity (<5 ms). No vector database required. Details in [docs/serverless_rag.md](docs/serverless_rag.md).

---

## Failure Handling

The system is designed to degrade predictably rather than fail silently.

| Scenario | Behavior |
|---|---|
| **LLM returns malformed JSON** | Re-prompted with a corrective message, up to 5 reasoning steps. On exhaustion, the customer receives a generic human-escalation response and a ticket is created. |
| **Bedrock rate limit (429)** | Exponential backoff (2^attempt), up to 3 retries per call. |
| **MCP server cold start** | Client retries connection up to 5 times over ~10 seconds before raising. |
| **Container restart mid-conversation** | Graph state is checkpointed to Postgres after every node; the next message resumes from the last checkpoint. |
| **No customer match / no email** | Context gathering is skipped; the agent answers from policy RAG only. |
| **Transient Bedrock error during context lookup** | Context gatherer returns `customer_context: None`; the agent still produces a response and can escalate to a human. |
| **Guardrail false positive** | A second string check against a small hard-coded token set prevents DSPy false positives from silently swallowing a customer message. |

---

## Security — 5 Layers

```
Edge (Cloudflare) → Application (FastAPI) → Container (Docker) → Network (AWS) → Data (PostgreSQL/SSM)
```

| Layer | Controls |
|-------|---------|
| **Edge** | Cloudflare Tunnel (mTLS, SSL strict), WAF (OWASP Top 10), DDoS, bot management. Zero inbound ports. |
| **Application** | Google OAuth 2.0 + PKCE, ES256 JWTs (15-min TTL), admin RBAC, parameterized queries, CORS restricted to origin. |
| **Container** | Non-root user (UID 1000), Python 3.12 slim, Trivy scan on push (blocks CRITICAL), immutable ECR tags. |
| **Network** | RDS in private subnets, ECS security group only. Least-privilege IAM: SSM read, Bedrock invoke, S3 read, DynamoDB write. |
| **Data** | Secrets in SSM Parameter Store (encrypted). RDS encrypted at rest + TLS. Gitleaks full-history scan + pre-commit hook. |

**DAST (OWASP ZAP):** 0 Critical · 0 High · 3 Medium (CSP on Cloudflare challenge page — assessed as non-exploitable in this threat model) · 3 Low · 4 Informational

**Threat model:** SQL Injection (none — parameterized queries) · XSS (none — auto-escaping + CSP) · Credential theft (low — OIDC + short TTL JWT) · DDoS (none — Cloudflare) · Container escape (low — non-root + minimal image).

 - Full details in [docs/security.md](docs/security.md). 
 - [DAST report json](src/reports/dast_scan_latest.json).

---

## Platform & Infrastructure

- **ECS Cluster:** 2 × `t4g.small` ARM64 managed instances across multiple AZs
- **CI/CD:** GitHub Actions with OIDC authentication to ECR — no static credentials
- **Image Security:** Immutable ECR tags, Trivy scanning on push (blocks CRITICAL vulnerabilities)
- **Deployment:** Rolling updates via `aws ecs update-service --force-new-deployment`
- **IaC:** OpenTofu for all AWS resources, scripted Cloudflare Tunnel + DNS provisioning
- **Observability:** JSON logs with correlation IDs, CloudWatch metric filters, dashboards, and alarms

Full details in [docs/infra.md](docs/infra.md).

---

## Cost Breakdown

**~$23/month (staging) · ~$38/month (production with RDS always-on).**

| Category | Baseline | Optimized | Saving |
|----------|----------|-----------|--------|
| Compute | $142 (Fargate) | $20 (2 × t4g.small, ECS managed) | $122 |
| Networking | $62 (ALB + NAT + WAF) | $0 (Cloudflare Tunnel) | $62 |
| Database | — | $0 (RDS PostgreSQL, free tier) | — |
| Vector Search | $190 (OpenSearch) | ~$0.03 (S3 + NumPy) | $190 |
| Observability | $15–20 (X-Ray) | ~$3 (CloudWatch) | $12–17 |
| **Total** | **~$414/month** | **~$23/month** | **91%** |

> The baseline column reflects a `conventional equivalent architecture` (Fargate + ALB + NAT + OpenSearch) for comparison. Full derivation in [docs/cost_optimizations.md](docs/cost_optimizations.md).

---

# TicketWeave — Step-by-Step Deployment Guide

## Prerequisites

1. **Docker installed and running _without_ sudo access.** Non-root is required for the Dev Container stability. If needed, run `sudo usermod -aG docker $USER && newgrp docker`.
2. **Visual Studio Code with the Dev Containers extension installed** (for a deterministic development environment): https://code.visualstudio.com/docs/devcontainers/containers
3. **An AWS account** (AWS Free Tier is sufficient for development) with permissions to create:
   - **Amazon ECS & EC2** (container orchestration and compute)
   - **VPC, Subnets & Security Groups** (networking)
   - **Amazon RDS PostgreSQL** (application database & LangGraph checkpoints)
   - **Amazon S3** (policy embedding storage)
   - **Amazon ECR** (container registry)
   - **AWS Systems Manager Parameter Store** (secret management)
   - **IAM Roles & Instance Profiles** (least-privilege access)
4. **A Cloudflare account with a registered domain**, with permissions to manage DNS records and create Cloudflare Tunnels (`cloudflared`).

---

## Clone the Repository and Build the Dev Container

```sh
cd "$HOME" && rm -rf TicketWeave &&
git clone https://github.com/Athithya-Sakthivel/TicketWeave.git &&
cd TicketWeave && code .
```

> Ctrl + Shift + P -> Paste `Dev containers: Rebuild Container Without Cache` and Enter. First-time build takes 5-15 minutes depending on network speed. You may create a github codespace instead if your network is slow

---

### Open a new terminal and log in to your `gh` account as shown below

```sh
git config --global user.name "Your Name"
git config --global user.email you@example.com
gh auth login

? What account do you want to log into? GitHub.com
? What is your preferred protocol for Git operations? SSH
? Generate a new SSH key to add to your GitHub account? No
? How would you like to authenticate GitHub CLI? Login with a web browser

! First copy your one-time code: <code>
- Press Enter to open github.com in your browser...
✓ Authentication complete. Press Enter to continue...
```

---

## Create a New Repository in your GitHub account

```sh
export REPO_NAME="TicketWeave" # or any name
git remote remove origin 2>/dev/null || true
gh repo create "$REPO_NAME" --private >/dev/null 2>&1
REMOTE_URL="https://github.com/$(gh api user | jq -r .login)/$REPO_NAME.git"
git remote add origin "$REMOTE_URL" 2>/dev/null || true
git branch -M main 2>/dev/null || true
git push -u origin main
git pull
git remote -v
echo "[INFO] A private repo '$REPO_NAME' created and pushed. Only visible from your account."
```

---

### Phase 1: Infrastructure Foundation

#### 1.1 Set Up Cloudflare Tunnel and DNS. [Docs](src/infra/cloudflare/README.md)
Creates a CNAME record → Cloudflare Tunnel and deploys a `cloudflared` daemon that routes HTTPS/WSS traffic into the VPC **without a load balancer or public IPs**. The tunnel terminates on each EC2 host and forwards to `localhost:8000` (agent‑service). A Global API Key is required once for simplified automation. The script waits for you to authorize the Cloudflare Tunnel with your domain.

```sh
export CLOUDFLARE_ACCOUNT_ID=      # Cloudflare dashboard > Account Home > Search and enter "Copy account ID".
export CLOUDFLARE_GLOBAL_API_KEY=  # https://dash.cloudflare.com/profile/api-tokens > API Keys
export CLOUDFLARE_EMAIL=           # Replace with your email
export DOMAIN=                     # replace with your domain
bash src/infra/cloudflare/run.sh --apply
```

![alt text](src/offline/images/edge_tf_outputs.png)

---

#### 1.2 Provision AWS Infrastructure. [Docs](docs/infra.md)
Provisions a VPC (public subnets for ECS, private subnets for RDS), a 2‑node ECS cluster on `t4g.small` ARM64 instances, S3 (policy embeddings), DynamoDB (rate‑limiting counters), RDS PostgreSQL (business data + LangGraph checkpoints), ECR repositories (immutable tags, scan‑on‑push), and least‑privilege IAM roles — all declared in OpenTofu.

```sh
export TF_VAR_region="ap-south-1"
export TF_VAR_github_repository="<GH_USER_NAME>/$REPO_NAME"
bash src/infra/aws/run.sh --create --env staging
```

![alt text](src/offline/images/aws.png)

---

### Phase 2: Data Preparation (Seed the fictional e-commerce company Kestral)

- Creates the `users`, `products`, `orders`, `billing`, and `tickets` tables and populates them with synthetic data so the agent has customers to look up and orders to reference. [Docs](docs/pg_tables.md)
- Generates Bedrock Titan v2 embeddings for 6 internal policy Markdown files (~59 chunks), writes a single `embeddings.json` to S3, and loads it in‑memory at agent startup for sub‑5ms brute‑force cosine retrieval. [Docs](docs/serverless_rag.md)

The command below temporarily allows your IP into the RDS security group, runs the seed script, indexes the policy documents, and uploads the embeddings:

```sh
export MY_IP=$(curl -s ifconfig.me) SG_ID=$(tofu -chdir=src/infra/aws output -raw rds_security_group_id) && \
aws ec2 authorize-security-group-ingress \
    --group-id "$SG_ID" \
    --protocol tcp \
    --port 5432 \
    --cidr "${MY_IP}/32" \
    --region "${TF_VAR_region:-ap-south-1}" && \
export DATABASE_URL="$(tofu -chdir=src/infra/aws output -raw rds_connection_string)" && \
python3 src/offline/simulate_company/setup_postgres.py && \
bash src/offline/index-policies/commands.sh
```

![alt text](src/offline/images/simulate_kestral.png)

---

### Phase 3.1: Trigger CI Workflows (Build & Push Container Images)

Pushes the `AWS_ACCOUNT_ID` and `AWS_REGION` secrets to GitHub, then makes a trivial whitespace commit to trigger the CI pipelines. GitHub Actions authenticates to ECR via OIDC (no static credentials), builds the `agent-service` and `mcp-server` Docker images, scans them with Trivy, and pushes them to ECR with immutable tags.

```sh
gh secret set AWS_ACCOUNT_ID --body $(aws sts get-caller-identity --query Account --output text)
echo " " >> src/workloads/agent-service/infra_tests.sh
echo " " >> src/workloads/mcp-server/test_locally.sh
gh secret set AWS_REGION --body $TF_VAR_region
git add . && git commit -m "Rebuilding mcp and agent docker images" && git push origin main
```

![alt text](src/offline/images/ci.png)

---

### Phase 3.2: Store OAuth Secrets in AWS SSM Parameter Store

The agent‑service authenticates users via Google OAuth (Microsoft is optional). These secrets are stored in SSM Parameter Store — never in code or environment variables — and fetched at runtime by the ECS task role with KMS decryption. By default all google domains are allowed in both user and admin login.

> **OAuth Credentials:** [Google](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/google/#usage) | [Microsoft](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/ms_entra_id)

```sh
export GOOGLE_CLIENT_ID="..."            # Google OAuth client ID
export GOOGLE_CLIENT_SECRET="..."        # Google OAuth client secret

# export MICROSOFT_CLIENT_ID="..."
# export MICROSOFT_CLIENT_SECRET="..."
# export MICROSOFT_TENANT_ID="..."       # Primary tenant ID (single-tenant or common)
export DOMAIN=                           # Use the same $DOMAIN
bash src/scripts/ssm-put.sh
```

---

### Phase 3.3: Force Redeploy ECS Services

Once the CI pipeline pushes the new images to ECR, force a rolling update on both ECS services so they pull the latest image tags. 

```sh
aws ecs update-service --cluster agentops-staging-cluster --service agentops-staging-cluster-agent --force-new-deployment --region ap-south-1
aws ecs update-service --cluster agentops-staging-cluster --service agentops-staging-cluster-mcp --force-new-deployment --region ap-south-1
```

![alt text](src/offline/images/force_reload_ecs.png)

> After ~5 minutes the agent is accessible at `https://<DOMAIN>` as shown in the gif.

---

### Phase 4: Teardown

Destroys the Cloudflare DNS records and Tunnel, then tears down all AWS resources (VPC, ECS, RDS, S3, DynamoDB, ECR, IAM roles). Order matters: Cloudflare first so the tunnel stops routing traffic before the backend is removed.

```sh
bash src/infra/cloudflare/run.sh --destroy
bash src/infra/aws/run.sh --destroy --env staging --yes-delete
```

---

## Key Takeaways

- Guardrails execute before any tool invocation.
- All business-impacting decisions remain with human agents.
- Deterministic routing guarantees predictable ticket ownership.
- Conversation state is checkpointed after every LangGraph node.
- Inline RAG eliminates the need for a vector database.
- Infrastructure is optimized for reliability, security, and cost efficiency.

---

## Documentation

| Document | What it covers |
|---|---|
| [docs/architecture.md](docs/architecture.md) | System design, networking flow, design decisions |
| [docs/agent_service.md](docs/agent_service.md) | LangGraph nodes, state schema, prompt design, v1 limitations |
| [docs/mcp_server.md](docs/mcp_server.md) | The three MCP tools, schemas, and what was intentionally removed |
| [docs/serverless_rag.md](docs/serverless_rag.md) | Inline policy retrieval, when to upgrade to a vector DB |
| [docs/security.md](docs/security.md) | 5-layer security architecture, DAST report, threat model |
| [docs/infra.md](docs/infra.md) | OpenTofu modules, ECS cluster, IAM, observability |
| [docs/cost_optimizations.md](docs/cost_optimizations.md) | Full cost derivation and baseline comparison |
| [docs/pg_tables.md](docs/pg_tables.md) | Database schema for all tables |

---
