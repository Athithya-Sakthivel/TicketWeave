# TicketWeave

**Ticket triage agent for e-commerce support teams, built on Amazon ECS with a 5-layer security architecture and cost-optimized infrastructure.**

TicketWeave receives customer messages over WebSocket and processes them through a LangGraph workflow running on ECS-managed instances behind Cloudflare Tunnel. Every message is first evaluated by a DSPy-optimized guardrail for **safety, intent, urgency, and sentiment**. The agent then retrieves customer context from an MCP server and *deterministically routes the request to the appropriate support team*.

For tickets requiring human intervention, an LLM generates a concise summary and recommended next action for the support agent. **Refunds, credits, pickup scheduling, and other customer-impacting operations are intentionally excluded.** TicketWeave is designed to assist human decision-making, not autonomously execute business actions.

---

## At a glance

| Metric                                    |            Result |
| ----------------------------------------- | ----------------: |
| **Optimized infrastructure cost**         |    **~$23/month** |
| **Estimated cost reduction vs. baseline** |           **91%** |
| **Inline policy retrieval latency**       |         **<5 ms** |
| **DAST critical findings**                |             **0** |
| **DAST high findings**                    |             **0** |
| **Inbound application ports**             |             **0** |
| **JWT lifetime**                          |    **15 minutes** |
| **Team assignment**                       | **Deterministic** |

> **Measurement note:** The figures above reflect the architecture, configuration, and recorded scan/cost estimates in this repository. Actual AWS costs, latency, and security-scan results will vary with region, workload, traffic, and configuration.

## Agent Demo 

![TicketWeave workflow](src/offline/images/agentops.gif)

**Core stack:** `Amazon ECS` · `Cloudflare Tunnel` · `FastAPI` · `LangGraph` · `DSPy` · `Amazon Bedrock (Llama 3 8B)` · `FastMCP` · `PostgreSQL` · `S3` · `DynamoDB` · `RDS` · `OWASP ZAP`

---

## System Architecture. [Docs](docs/architecture.md)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/8a7acac1-bf67-44e4-b832-b08c29924fd4" />

---

### Networking Flow
```
Internet → Cloudflare Tunnel → ECS Host (host network mode) → Agent Service → MCP Server → RDS
```
- **Zero inbound ports:** Security groups have no ingress rules; all traffic enters via Cloudflare Tunnel
- **No load balancer:** Cloudflare Tunnel terminates on each EC2 host, eliminating ALB/NLB costs
- **Private subnets:** RDS PostgreSQL deployed in private subnets, accessible only from ECS hosts

---

## Platform & Infrastructure . [Docs](docs/infra.md)

### Container Deployment Pipeline
- **ECS Cluster:** 2 × t4g.small ARM64 managed instances across multiple AZs
- **CI/CD:** GitHub Actions with OIDC authentication to ECR—no static credentials
- **Image Security:** Immutable ECR tags, Trivy scanning on push (blocks CRITICAL vulnerabilities)
- **Deployment:** Rolling updates via `aws ecs update-service --force-new-deployment`
- **IaC:** OpenTofu for all AWS resources, scripted Cloudflare Tunnel + DNS provisioning
- **Observability:** JSON logs with correlation IDs, CloudWatch metric filters, dashboards, and alarms

---

## Security (5 Layers) . [Docs](docs/security.md)

```
Edge (Cloudflare) → Application (FastAPI) → Container (Docker) → Network (AWS) → Data (PostgreSQL/SSM)
```

| Layer | Controls |
|-------|---------|
| **Edge** | Cloudflare Tunnel (mTLS, SSL strict), WAF (OWASP Top 10), DDoS, bot management. Zero inbound ports. |
| **Application** | Google OAuth 2.0 + PKCE, ES256 JWTs (15‑min TTL), admin RBAC, parameterized queries, CORS restricted to origin. |
| **Container** | Non‑root user (UID 1000), Python 3.12 slim, Trivy scan on push (blocks CRITICAL), immutable ECR tags. |
| **Network** | RDS in private subnets, ECS security group only. Least‑privilege IAM: SSM read, Bedrock invoke, S3 read, DynamoDB write. |
| **Data** | Secrets in SSM Parameter Store (encrypted). RDS encrypted at rest + TLS. Gitleaks full‑history scan + pre‑commit hook. |

**DAST (OWASP ZAP):** 0 Critical · 0 High · 3 Medium (CSP on Cloudflare challenge page — not exploitable) · 3 Low · 4 Informational  
**Threat model:** SQL Injection (none — parameterized queries) · XSS (none — auto‑escaping + CSP) · Credential theft (low — OIDC + short TTL JWT) · DDoS (none — Cloudflare) · Container escape (low — non‑root + minimal image). [DAST Report](src/reports/dast_scan_latest.json)

---

## AI Workflow . [Docs](docs/agent_service.md)

**Four LangGraph nodes**

1. **Guardrail classifier** – DSPy‑compiled triage program (Llama 3 8B, temperature 0) classifies safety, intent, urgency, and sentiment. Unsafe or urgency ≥ 10 → immediate human escalation.
2. **Context gatherer** – Fetches customer profile + last 5 orders (with product names) from the MCP server using the user's email.
3. **Ticket router** – LLM with two tools: `search_policies` (inline RAG) and `create_ticket`. Simple policy questions get an instant answer; issues needing human action produce a structured ticket with AI‑written summary and suggested action. Team assignment is **deterministic** (hard‑coded intent‑to‑team map), not left to the LLM.
4. **Human escalate** – Skips the LLM entirely; directly creates a high‑priority ticket via MCP for dangerous or extremely urgent messages.

**State & persistence**  
All conversation state is checkpointed to PostgreSQL after every node via `AsyncPostgresSaver` — the agent survives restarts and resumes any in‑flight conversation.

**MCP server** . [Docs](docs/mcp_server.md)  
Three tools exposed over FastMCP: `lookup_customer`, `get_recent_orders`, `create_ticket`. It owns **zero** business logic — all routing, summarisation, and policy decisions stay in the agent.

**Inline RAG**  . [Docs](docs/serverless_rag.md)
Policy documents are pre‑embedded with Bedrock Titan v2, stored as a single ~1.3 MB JSON file in S3, loaded once at startup, and searched with in‑process NumPy cosine similarity (<5 ms). No vector database required.

---

## Key Design Decisions . [Docs](docs/architecture.md)

| Decision | Why |
|----------|-----|
| **ECS over Fargate** | 86% compute cost reduction ($142 → $20/month) with managed instances |
| **Cloudflare Tunnel over ALB/WAF** | Eliminates $62/month in networking costs, zero inbound ports |
| **Inline RAG over OpenSearch** | Sufficient for policy ingestion, $190/month savings, sub‑5ms search latency |
| **Never act autonomously** | The agent cannot refund, credit, or schedule pickups. Those tools were deliberately removed from the MCP server. |
| **Deterministic routing** | A hard‑coded `INTENT → TEAM` table guarantees 100% predictable ticket assignment. |
| **DSPy guardrail first** | Every message is classified **before** any context is fetched or any tool is called. |

---

## Cost Breakdown . [Docs](docs/cost_optimizations.md)

| Category | Baseline | Optimized | Saving |
|----------|----------|-----------|--------|
| Compute | $142 (Fargate) | $20 (2 × t4g.small, ECS managed) | $122 |
| Networking | $62 (ALB + NAT + WAF) | $0 (Cloudflare Tunnel) | $62 |
| Database | — | $0 (RDS PostgreSQL, free tier) | — |
| Vector Search | $190 (OpenSearch) | ~$0.03 (S3 + NumPy) | $190 |
| Observability | $15–20 (X‑Ray) | ~$3 (CloudWatch) | $12–17 |
| **Total** | **~$414/month** | **~$23/month** | **91%** |

---

# TicketWeave — Step-by-Step Deployment Guide

## Prerequisites

1. **Docker installed and running _without_ sudo access.** Non-root is required for the Dev Container. If needed, run `sudo usermod -aG docker $USER && newgrp docker`.
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

> Ctrl + Shift + P -> Paste `Dev containers: Rebuild Container Without Cache` and Enter. First-time build takes 5-15 minutes depending on network speed. You may create a github codespace instead if network is slow

---

## Authenticate GitHub CLI

Open a new terminal:

```sh
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
gh auth login
```

Recommended authentication flow:

```text
? What account do you want to log into? GitHub.com
? What is your preferred protocol for Git operations? SSH
? Generate a new SSH key to add to your GitHub account? No
? How would you like to authenticate GitHub CLI? Login with a web browser
```

---

## Create a New GitHub Repository

```sh
export REPO_NAME="TicketWeave-1"

git remote remove origin 2>/dev/null || true

gh repo create "$REPO_NAME" --private >/dev/null 2>&1

REMOTE_URL="https://github.com/$(gh api user | jq -r .login)/$REPO_NAME.git"

git remote add origin "$REMOTE_URL" 2>/dev/null || true
git branch -M main 2>/dev/null || true

git push -u origin main
git pull
git remote -v

echo "[INFO] Private repository '$REPO_NAME' created and pushed."
```

---

# Phase 1 — Infrastructure Foundation

## 1.1 Set Up Cloudflare Tunnel and DNS

[Docs](src/infra/cloudflare/README.md)

This step creates the DNS record and provisions the Cloudflare Tunnel used to route HTTPS/WSS traffic into the environment without a load balancer.

Set your own values before running:

```sh
export CLOUDFLARE_ACCOUNT_ID="..."
export CLOUDFLARE_GLOBAL_API_KEY="..."
export CLOUDFLARE_EMAIL="YOUR_CLOUDFLARE_EMAIL"
export DOMAIN="YOUR_DOMAIN"
```

Then:

```sh
bash src/infra/cloudflare/run.sh --apply
```

> Prefer a narrowly scoped Cloudflare API token where the provisioning scripts support it. Never commit the credential or place it directly in the README.

![Cloudflare infrastructure](src/offline/images/edge_tf_outputs.png)

---

## 1.2 Provision AWS Infrastructure

[Docs](docs/infra.md)

Provision:

* VPC and subnets
* ECS cluster
* 2 × `t4g.small` ARM64 instances
* S3 policy-embedding storage
* DynamoDB rate-limiting counters
* RDS PostgreSQL
* ECR repositories
* IAM roles and policies

Example:

```sh
export TF_VAR_region="ap-south-1"
export TF_VAR_github_repository="YOUR_GITHUB_OWNER/YOUR_REPOSITORY"

bash src/infra/aws/run.sh --create --env staging
```

![AWS infrastructure](src/offline/images/aws.png)

---

# Phase 2 — Data Preparation

TicketWeave uses a fictional e-commerce company named **Kestral** for development and testing.

The data-preparation phase creates:

* `users`
* `products`
* `orders`
* `billing`
* `tickets`

It then generates Bedrock Titan v2 embeddings for six internal policy Markdown files (~59 chunks) and uploads the resulting `embeddings.json` file to S3.

[PostgreSQL schema docs](docs/pg_tables.md)
[Inline RAG docs](docs/serverless_rag.md)

### Seed PostgreSQL and Index Policies

The following flow temporarily authorizes the current public IP against the RDS security group, runs the synthetic-data setup, builds policy embeddings, and uploads them to S3.

```sh
export MY_IP=$(curl -s ifconfig.me)
export SG_ID=$(tofu -chdir=src/infra/aws output -raw rds_security_group_id)

aws ec2 authorize-security-group-ingress \
    --group-id "$SG_ID" \
    --protocol tcp \
    --port 5432 \
    --cidr "${MY_IP}/32" \
    --region "${TF_VAR_region:-ap-south-1}"

export DATABASE_URL="$(tofu -chdir=src/infra/aws output -raw rds_connection_string)"

python3 src/offline/simulate_company/setup_postgres.py

bash src/offline/index-policies/commands.sh
```

![Kestral test data](src/offline/images/simulate_kestral.png)

> Remove the temporary RDS ingress rule after initialization if it is no longer required. The intended steady-state architecture keeps the database accessible only from the ECS environment.

---

# Phase 3 — Application Deployment

## 3.1 Trigger CI Workflows

GitHub Actions builds and pushes the `agent-service` and `mcp-server` images to ECR.

Authentication uses GitHub Actions OIDC rather than long-lived AWS credentials.

Configure repository secrets:

```sh
gh secret set AWS_ACCOUNT_ID \
  --body "$(aws sts get-caller-identity --query Account --output text)"

gh secret set AWS_REGION \
  --body "$TF_VAR_region"
```

Trigger a deployment build:

```sh
echo " " >> src/workloads/agent-service/infra_tests.sh
echo " " >> src/workloads/mcp-server/test_locally.sh

git add .
git commit -m "Rebuild application container images"
git push origin main
```

The CI workflow:

1. authenticates to AWS through OIDC
2. builds the Docker images
3. scans the images with Trivy
4. blocks CRITICAL findings
5. pushes immutable ECR image tags

![CI pipeline](src/offline/images/ci.png)

---

## 3.2 Store OAuth Secrets in AWS SSM Parameter Store

The agent service authenticates users with Google OAuth. Microsoft Entra ID can optionally be configured.

Credentials are stored in SSM Parameter Store rather than in application source code.

OAuth provider documentation:

* [Google OAuth](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/google/)
* [Microsoft Entra ID](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/ms_entra_id)

Example:

```sh
export GOOGLE_CLIENT_ID="..."
export GOOGLE_CLIENT_SECRET="..."

# Optional Microsoft OAuth
# export MICROSOFT_CLIENT_ID="..."
# export MICROSOFT_CLIENT_SECRET="..."
# export MICROSOFT_TENANT_ID="..."

export DOMAIN="YOUR_DOMAIN"

bash src/scripts/ssm-put.sh
```

> Treat OAuth client secrets exactly like passwords: do not commit them, paste them into issues, or place them in shell history when avoidable.

---

## 3.3 Force Redeploy ECS Services

Once the images are available in ECR, force rolling deployments:

```sh
aws ecs update-service \
  --cluster agentops-staging-cluster \
  --service agentops-staging-cluster-agent \
  --force-new-deployment \
  --region ap-south-1

aws ecs update-service \
  --cluster agentops-staging-cluster \
  --service agentops-staging-cluster-mcp \
  --force-new-deployment \
  --region ap-south-1
```

After deployment completes, the application is available at:

```text
https://<DOMAIN>
```

![ECS redeployment](src/offline/images/force_reload_ecs.png)

---

# Phase 4 — Teardown

Destroy the Cloudflare resources first so traffic stops routing before the AWS environment is removed.

```sh
bash src/infra/cloudflare/run.sh --destroy

bash src/infra/aws/run.sh --destroy \
  --env staging \
  --yes-delete
```

---

# Key Takeaways

* **Guardrails run before context retrieval or tool invocation.**
* **Human agents retain control of all business-impacting decisions.**
* **Team assignment is deterministic rather than LLM-selected.**
* **Conversation state is checkpointed after every LangGraph node.**
* **Inline RAG removes the need for a dedicated vector database for the current corpus.**
* **Cloudflare Tunnel removes the need for an internet-facing load balancer in this design.**
* **Container images are scanned and published through an OIDC-based CI/CD pipeline.**
* **The MCP layer exposes narrow tools and does not contain business-routing logic.**
* **The infrastructure is designed around security boundaries, predictable behavior, and low operating cost.**
