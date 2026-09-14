# TicketWeave

**AI-powered ticket triage system for e-commerce support teams, deployed on Amazon ECS with a 5-layer security architecture and cost optimizations**

`TicketWeave` receives customer messages over WebSocket and processes them through a LangGraph workflow running on ECS-managed instances behind Cloudflare Tunnel—no load balancers, no public IPs, zero inbound ports. Each message is first classified by a DSPy-optimized guardrail for safety, intent, urgency, and sentiment. The agent then retrieves customer context—including profile information and recent orders—from an MCP server before deterministically routing the request to the appropriate support team.

For tickets requiring human intervention, an LLM-powered ticket router generates a concise summary and a recommended next action to assist support agents. Customer-impacting operations such as refunds, credits, and pickup scheduling are intentionally excluded from the system. TicketWeave is designed exclusively to enhance human decision-making, not to automate business actions.

**Technology Stack:** `Amazon ECS` · `Cloudflare Tunnel` · `FastAPI` · `LangGraph` · `DSPy` · `Amazon Bedrock (Llama 3 8B)` · `FastMCP` · `PostgreSQL` · `Amazon S3` · `Amazon CloudWatch`

---

## Architecture . [Docs](docs/architecture.md)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/8a7acac1-bf67-44e4-b832-b08c29924fd4" />

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


# Get started

## Prerequisites
1. **Docker installed, running *without* sudo access**
2. **Visual Studio Code with the Dev Containers extension installed (for a deterministic environments): [https://code.visualstudio.com/docs/devcontainers/containers](https://code.visualstudio.com/docs/devcontainers/containers)**
3. **An AWS account with sufficient IAM permissions (AdministratorAccess or equivalent) to manage**:
   * Amazon ECS (Elastic Container Service)
   * EC2, VPCs, Subnets, and Security Groups
   * Amazon S3, SSM, ECR
   * IAM Roles, Policies, and Instance Profiles
   **AWS Free Tier is sufficient for development and testing purposes.**
4. **A Cloudflare account with a registered domain, with permissions to manage DNS records and create Cloudflare Tunnels (cloudflared)**

## Clone the repo and build the devcontainer(Reproducible). This will take 10-20 minutes. 
```sh 
cd $HOME && rm -rf TicketWeave && git clone https://github.com/Athithya-Sakthivel/TicketWeave.git && cd TicketWeave && code .
```
> ctrl + shift + P -> paste `Dev containers: Rebuild Container Without Cache` and enter

### Open a new terminal and login to your gh account
```sh
git config --global user.name "Your Name" && git config --global user.email you@example.com
gh auth login

? What account do you want to log into? GitHub.com
? What is your preferred protocol for Git operations? SSH
? Generate a new SSH key to add to your GitHub account? No
? How would you like to authenticate GitHub CLI? Login with a web browser

! First copy your one-time code: <code>
- Press Enter to open github.com in your browser... 
✓ Authentication complete. Press Enter to continue...
```

### Create a private repo in your gh account

```sh
export REPO_NAME="TicketWeave-1" # or any name
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
Creates a `athithya.site` CNAME record → Cloudflare Tunnel and deploys a `cloudflared` daemon that routes HTTPS/WSS traffic into the VPC **without a load balancer or public IPs**. The tunnel terminates on each EC2 host and forwards to `localhost:8000` (agent‑service). Requires a browser login to your Cloudflare account.

```sh
export CLOUDFLARE_ACCOUNT_ID=
export CLOUDFLARE_GLOBAL_API_KEY=
export CLOUDFLARE_EMAIL="athithya651@gmail.com" # Replace with your email
export DOMAIN="athithya.site"  # replace with your domain
bash src/infra/cloudflare/run.sh --apply
```

![alt text](src/offline/images/edge_tf_outputs.png)

---

#### 1.2 Provision AWS Infrastructure. [Docs](docs/infra.md)
Provisions a VPC (public subnets for ECS, private subnets for RDS), a 2‑node ECS cluster on `t4g.small` ARM64 instances, S3 (policy embeddings), DynamoDB (rate‑limiting counters), RDS PostgreSQL (business data + LangGraph checkpoints), ECR repositories (immutable tags, scan‑on‑push), and least‑privilege IAM roles — all declared in OpenTofu.

```sh
export TF_VAR_region="ap-south-1"
export TF_VAR_github_repository="Athithya-Sakthivel/TicketWeave"   # replace with your GitHub repo
bash src/infra/aws/run.sh --create --env staging
```

![alt text](src/offline/images/aws.png)

---

### Phase 2: Data Preparation (Mimic a fictional e‑commerce company named Kestral)
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

> 🔑 **OAuth Setup:** [Google](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/google/#usage) | [Microsoft](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/ms_entra_id)

```sh
export GOOGLE_CLIENT_ID="..."
export GOOGLE_CLIENT_SECRET="..."
# Optional: Microsoft OAuth
# export MICROSOFT_CLIENT_ID="..."
# export MICROSOFT_CLIENT_SECRET="..."
# export MICROSOFT_TENANT_ID="..."
export DOMAIN="athithya.site"
bash src/scripts/ssm-put.sh
```

### Phase 3.3: Force Redeploy ECS Services

Once the CI pipeline pushes the new images to ECR, force a rolling update on both ECS services so they pull the latest image tags. After ~5 minutes the agent is accessible at `https://<DOMAIN>`.

```sh
aws ecs update-service --cluster agentops-staging-cluster --service agentops-staging-cluster-agent --force-new-deployment --region ap-south-1
aws ecs update-service --cluster agentops-staging-cluster --service agentops-staging-cluster-mcp --force-new-deployment --region ap-south-1
```

![alt text](src/offline/images/force_reload_ecs.png)

---

### Phase 4: Teardown

Destroys the Cloudflare DNS records and Tunnel, then tears down all AWS resources (VPC, ECS, RDS, S3, DynamoDB, ECR, IAM roles). Order matters: Cloudflare first so the tunnel stops routing traffic before the backend is removed.

```sh
bash src/infra/cloudflare/run.sh --destroy
bash src/infra/aws/run.sh --destroy --env staging --yes-delete
```

---


![alt text](src/offline/images/agentops.gif)

---


## Key Takeaways

- Guardrails execute before any tool invocation.
- All business-impacting decisions remain with human agents.
- Deterministic routing guarantees predictable ticket ownership.
- Conversation state is checkpointed after every LangGraph node.
- Inline RAG eliminates the need for a vector database.
- Infrastructure is optimized for reliability, security, and cost efficiency.
