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
