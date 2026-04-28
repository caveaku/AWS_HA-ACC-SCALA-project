# AWS High-Availability, Auto-Scaling URL Shortener

A production-grade URL shortener (like bit.ly) built entirely on AWS, demonstrating **high availability**, **auto-scaling**, and **scalable architecture** across all layers of the stack.

---

## What This Project Builds

A fully functional URL shortening service where users paste a long URL and receive a short link. Every redirect is cached in Redis, every short code is persisted in PostgreSQL, and the entire system tolerates AZ failures without downtime.

---

## Architecture

```
Internet
    │
    ▼
Route 53 (DNS + health checks)
    │
    ▼
CloudFront (CDN + WAF + HTTPS termination)
    │
    ├──► S3 (React static frontend — served at edge)
    │
    ▼
Application Load Balancer (public subnets, multi-AZ)
    │
    ├──► EC2 Auto Scaling (AZ 1a, private subnet) ──► ElastiCache Redis ──► RDS Primary (1a)
    │                                                                              │ sync replication
    └──► EC2 Auto Scaling (AZ 1b, private subnet) ──► ElastiCache Redis ──► RDS Standby (1b)
```

---

## AWS Services Used

| Service | Role |
|---|---|
| **EC2 + Auto Scaling Group** | Runs the Node.js app; scales out when CPU > 70% |
| **Application Load Balancer** | Distributes traffic across AZs; health-checks every 30s |
| **RDS PostgreSQL (Multi-AZ)** | Stores all short codes; standby replica in a second AZ |
| **ElastiCache Redis** | Cache-aside layer; reduces DB reads with a 24h TTL |
| **CloudFront** | CDN for the frontend; routes `/api/*` and `/:code` to the ALB |
| **S3** | Hosts the React static build; not publicly exposed (CloudFront only) |
| **WAF** | Rate-limits to 1,000 req/IP; AWS Managed Rules block common attacks |
| **Route 53** | DNS + health checks; automatic failover if ALB becomes unhealthy |
| **ACM** | Free TLS certificate, auto-renewed |
| **Secrets Manager** | Stores DB password; EC2 role fetches it at runtime |
| **IAM** | Least-privilege EC2 role (SSM access + Secrets read) |
| **CloudWatch** | Alarms on 5xx rate, RDS CPU, ASG max capacity, Redis hit rate |
| **SNS** | Sends alarm notifications to email |

---

## Network Layout

| Subnet | CIDR | AZ | Purpose |
|---|---|---|---|
| public-1a | 10.0.1.0/24 | us-east-1a | ALB + NAT Gateway |
| public-1b | 10.0.2.0/24 | us-east-1b | ALB (second AZ) |
| private-1a | 10.0.3.0/24 | us-east-1a | EC2 instances + RDS primary |
| private-1b | 10.0.4.0/24 | us-east-1b | EC2 instances + RDS standby |

Private subnets reach the internet via a NAT Gateway (outbound only). EC2 instances are never directly reachable from the public internet.

---

## Security Model

- **ALB SG** — accepts port 80/443 from `0.0.0.0/0` only
- **EC2 SG** — accepts port 3000 from the ALB security group only
- **RDS SG** — accepts port 5432 from the EC2 security group only
- **Redis SG** — accepts port 6379 from the EC2 security group only
- **WAF** — rate-limits per IP and applies AWS Managed Rules (SQLi, XSS, etc.)
- **RDS** — encrypted at rest (gp3), not publicly accessible, 7-day backup retention
- **S3** — all public access blocked; served exclusively through CloudFront

---

## Application (Node.js)

Two endpoints:

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | ALB health check — returns `{ status: "ok" }` |
| `POST` | `/api/shorten` | Accepts `{ longUrl }`, returns `{ shortUrl }` |
| `GET` | `/:code` | Looks up code in Redis → PostgreSQL, redirects (HTTP 301) |

The redirect uses a **cache-aside pattern**:
1. Check Redis — if found, redirect immediately and increment click count.
2. On cache miss, query PostgreSQL, write result to Redis (TTL: 24h), then redirect.

Short codes are 7-character Base62 strings generated with `nanoid`, giving **3.5 trillion unique URLs**.

---

## High Availability & Resilience

| Failure Scenario | How the System Handles It |
|---|---|
| EC2 instance dies | ASG detects unhealthy instance and launches a replacement (~3 min) |
| Entire AZ goes down | ALB + ASG + RDS Multi-AZ continue serving traffic from the other AZ |
| RDS primary fails | Automatic failover to standby replica (~60 seconds); Redis serves cached reads during the gap |
| Traffic spike | ASG scales out when average CPU exceeds 70% for 2 consecutive minutes |
| DDoS / abuse | WAF blocks IPs exceeding 1,000 req/min; CloudFront absorbs edge traffic |

---

## Auto Scaling Policy

- **Minimum instances:** 2 (one per AZ for baseline HA)
- **Maximum instances:** 10
- **Scale-out trigger:** Average CPU > 70% for 2 minutes (cooldown: 60s)
- **Scale-in cooldown:** 300 seconds (prevents thrashing)

---

## Key Numbers

| Metric | Value |
|---|---|
| RDS failover time | ~60 seconds |
| ASG instance replacement | ~3 minutes |
| Route 53 health check interval | 30 seconds |
| Redis cache TTL | 86,400s (24 hours) |
| ALB health check path | `/health` every 30s |
| Auto-scale trigger | CPU > 70% for 2 min |
| Unique short codes (7-char Base62) | 3.5 trillion |
| Estimated monthly AWS cost | ~$145–$200 |
| Target uptime SLA | 99.99% |

---

## Build Guide

See [`aws-url-shortener-guide.md`](./aws-url-shortener-guide.md) for the complete step-by-step AWS CLI commands to provision every resource from scratch, in order:

1. Subnets (4 across 2 AZs)
2. Internet Gateway + Route Tables
3. NAT Gateway
4. Security Groups
5. RDS PostgreSQL Multi-AZ
6. ElastiCache Redis
7. Node.js Application Code
8. AMI + EC2 Launch Template
9. IAM Role for EC2
10. Auto Scaling Group + Scaling Policy
11. Application Load Balancer
12. ACM Certificate + Route 53
13. S3 Bucket (frontend assets)
14. CloudFront Distribution
15. WAF Web ACL
16. CloudWatch Alarms + Dashboard
17. End-to-end verification
18. High-availability failure simulation

---

## Region

All resources deploy to **us-east-1** unless otherwise noted. WAF for CloudFront must be created in us-east-1 regardless of application region.

---

## Interview Q&A

This section covers the questions most commonly asked when presenting this project in a technical interview.

---

### "Walk me through the architecture."

> "I built a URL shortener that mirrors a production bit.ly-style service. Traffic enters at Route 53, which does DNS routing and health checks. It hits CloudFront, which serves the React frontend from S3 at the edge and passes API calls and redirect requests down to an Application Load Balancer. The ALB sits in two public subnets across two availability zones and forwards traffic to EC2 instances running in private subnets. Those instances run a Node.js app that checks Redis first on every redirect — if the short code is cached, it redirects immediately. On a cache miss it queries RDS PostgreSQL, writes the result to Redis with a 24-hour TTL, and then redirects. RDS is Multi-AZ so there's a synchronised standby in the second AZ. WAF sits in front of CloudFront and blocks rate-limit abusers and common web exploits."

---

### "Why did you put EC2 in private subnets?"

> "Defense in depth. If an EC2 instance is compromised, it has no public IP and is unreachable from the internet directly. All inbound traffic must come through the ALB, which is the only entry point. The EC2 security group only allows port 3000 from the ALB security group — not from any CIDR range. Outbound internet access for things like package updates goes through a NAT Gateway, so the instances can reach out but nothing can reach in."

---

### "Why Redis? Why not just hit the database every time?"

> "A URL shortener's read-to-write ratio is extremely high. Once a link is created, it might be clicked thousands of times but never updated. Without caching, every redirect would hit PostgreSQL, which would quickly become the bottleneck. Redis sits in-memory and responds in under a millisecond. I used the cache-aside pattern — the app checks Redis first, and only falls back to Postgres on a miss. The 24-hour TTL balances freshness against cache efficiency. In practice, popular links stay warm indefinitely because they're accessed constantly before TTL expires."

---

### "Why RDS Multi-AZ instead of a read replica?"

> "Multi-AZ is about durability and automatic failover, not read scaling. The standby in the second AZ is synchronously replicated — every write is acknowledged only after it lands on both nodes. If the primary fails, RDS promotes the standby automatically in about 60 seconds. A read replica uses asynchronous replication, which means there's replication lag and it requires your application to split reads and writes across two endpoints. For this project, the priority was zero data loss and automatic recovery, not read throughput — which is already handled by Redis anyway."

---

### "How does the auto-scaling work and why 70% CPU?"

> "The Auto Scaling Group uses a target-tracking policy tied to average CPU utilisation across all instances. When the group average crosses 70%, ASG launches new EC2s from the launch template within 60 seconds. I chose 70% because it leaves enough headroom — if a spike hits, new instances are warming up before you run out of capacity. Scale-in has a 300-second cooldown to avoid thrashing: without it, the group could scale in while a new wave of traffic is already arriving. Minimum is 2 so there's always one instance per AZ — you never drop below baseline HA."

---

### "What happens if an entire availability zone goes down?"

> "Every critical layer spans two AZs. The ALB has nodes in both public subnets — it stops routing to the dead AZ within one health check interval (30 seconds). The ASG has instances in both private subnets — it replaces lost instances in the surviving AZ. RDS fails over to the standby in the second AZ in roughly 60 seconds. During that 60-second window, Redis continues serving cached redirects, so most users experience no interruption at all. Route 53 health checks also monitor the ALB endpoint and can reroute DNS if needed, though in a single-region setup the ALB itself handles AZ failover."

---

### "How did you handle security?"

> "I applied the principle of least privilege at every layer. Each component only accepts traffic from the one above it — the security group chain is: internet → ALB (80/443) → EC2 (3000) → RDS (5432) / Redis (6379). No EC2 instance has a public IP. The DB password is never in the code or environment files — it's stored in Secrets Manager and the EC2 IAM role has permission to read it at runtime. RDS is encrypted at rest. S3 has all public access blocked; CloudFront uses an Origin Access Identity to fetch assets. WAF adds a rate-limit rule and the AWS Managed Rule Group, which covers SQL injection, XSS, and known bad bots out of the box."

---

### "How do you know the system is healthy? How would you debug a production incident?"

> "I set up four CloudWatch alarms: ALB 5xx error rate, RDS CPU above 80%, ASG reaching maximum capacity, and Redis cache hit rate dropping below 70%. All alarms publish to an SNS topic that sends email. The CloudWatch dashboard shows ALB request count and p99 latency in real time. If I saw a spike in 5xx errors I'd check the ALB access logs first, then look at target group health to see which instances are failing health checks. A Redis hit-rate drop would tell me the cache is being bypassed — I'd check for expired keys or a memory pressure issue on the ElastiCache cluster."

---

### "How would you scale this to 10x or 100x traffic?"

> "Several levers. First, increase the ASG maximum and let auto-scaling handle it — the architecture already supports horizontal scaling. Second, add an ElastiCache cluster-mode Redis with sharding instead of a single node, which gives both higher throughput and HA for the cache layer. Third, move to RDS Aurora PostgreSQL — it separates storage from compute, supports up to 15 read replicas, and has a global database feature for multi-region. Fourth, if redirects become the dominant load, I'd push the redirect logic into a CloudFront Function or Lambda@Edge so it runs at the edge node closest to the user, completely bypassing the origin. At extreme scale I'd also shard the `urls` table by short code prefix."

---

### "Why CloudFront in front of the ALB? The ALB is already multi-AZ."

> "Three reasons. First, caching: CloudFront can cache redirect responses at edge nodes globally, which means frequent links never reach the ALB at all. Second, WAF integration: WAF for CloudFront must be created in us-east-1 but protects a global edge network — you get protection before traffic even enters your VPC. Third, the static frontend is served from S3 via CloudFront at the edge with zero latency hits to the origin — the ALB only sees API and redirect traffic, keeping it lean."

---

### "Why not use Lambda instead of EC2?"

> "Lambda would work and would be cheaper at low volume, but EC2 + ASG was the deliberate choice here to demonstrate auto-scaling concepts explicitly. Lambda also has cold-start latency on redirects, which matters because the redirect path is latency-sensitive — users notice a 500ms delay. With warm EC2 instances behind Redis, p99 redirect latency is under 10ms. Lambda would require a different persistence strategy too — you can't hold a persistent Redis or Postgres connection pool across invocations the same way. If the requirement were cost-efficiency at unpredictable burst traffic, Lambda + API Gateway would be the right trade-off."

---

### "Explain the short code generation. Could you get collisions?"

> "I use `nanoid` to generate 7-character codes from a 64-character alphabet (A–Z, a–z, 0–9, `-`, `_`), giving 64^7 = approximately 4.4 trillion combinations. In the database, `short_code` has a `UNIQUE` constraint, so a collision would raise a constraint violation — the app would retry with a new code. At typical URL shortener scale (even billions of links), the probability of collision is negligible. If the system reached that scale I'd switch to a distributed ID generator like Twitter Snowflake and encode the numeric ID in Base62 instead, which guarantees uniqueness without retries."

---

### "What would you do differently if you rebuilt this?"

> "A few things. I'd use Terraform or CDK instead of raw AWS CLI — the current setup requires running 80+ commands in order and is hard to reproduce. I'd add an SQS queue in front of the click-count update so that high-traffic links don't create write contention on the `urls` table. I'd also add a DynamoDB table for short code lookups as an alternative to Postgres — DynamoDB's single-digit millisecond reads at any scale would reduce dependency on Redis for cache warming. And I'd instrument the app with AWS X-Ray for distributed tracing so I can see exactly where latency is coming from across the ALB → EC2 → Redis → RDS call chain."
