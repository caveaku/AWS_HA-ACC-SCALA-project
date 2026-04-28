# AWS High-Availability URL Shortener — Complete Build Guide

> **Project:** Scalable URL Shortener (like bit.ly)  
> **Region:** us-east-1  
> **Stack:** EC2 + ALB + RDS PostgreSQL + ElastiCache Redis + CloudFront + WAF + Route 53  
> **Your VPC ID:** vpc-029b6275a3c3d86b8

---

## Prerequisites

```bash
# Confirm AWS CLI is working
aws --version
aws sts get-caller-identity

# Set your VPC ID as a session variable (saves copy-paste errors)
export VPC_ID=vpc-029b6275a3c3d86b8
export REGION=us-east-1
```

---

## STEP 1 — Create Subnets (4 total across 2 AZs)

```bash
# --- PUBLIC SUBNETS (ALB + NAT Gateway) ---

aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1a}]'

aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1b}]'

# --- PRIVATE SUBNETS (EC2 + RDS) ---

aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.3.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-1a}]'

aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.4.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-1b}]'
```

### Save your subnet IDs from the output:

```bash
export PUB_SUBNET_1A=subnet-XXXXXXXXX   # public-1a
export PUB_SUBNET_1B=subnet-XXXXXXXXX   # public-1b
export PRI_SUBNET_1A=subnet-XXXXXXXXX   # private-1a
export PRI_SUBNET_1B=subnet-XXXXXXXXX   # private-1b
```

---

## STEP 2 — Internet Gateway + Route Tables

```bash
# Create and attach Internet Gateway
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=url-igw}]'

# Save the IGW ID from output
export IGW_ID=igw-XXXXXXXXX

aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID

# Create PUBLIC route table
aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]'

export PUBLIC_RT=rtb-XXXXXXXXX

# Add default route to IGW
aws ec2 create-route \
  --route-table-id $PUBLIC_RT \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

# Associate public subnets with public route table
aws ec2 associate-route-table \
  --route-table-id $PUBLIC_RT \
  --subnet-id $PUB_SUBNET_1A

aws ec2 associate-route-table \
  --route-table-id $PUBLIC_RT \
  --subnet-id $PUB_SUBNET_1B

# Enable auto-assign public IP on public subnets
aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUBNET_1A \
  --map-public-ip-on-launch

aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUBNET_1B \
  --map-public-ip-on-launch
```

---

## STEP 3 — NAT Gateway (lets private EC2 reach the internet)

```bash
# Allocate an Elastic IP for NAT Gateway
aws ec2 allocate-address --domain vpc

export EIP_ALLOC=eipalloc-XXXXXXXXX   # save AllocationId from output

# Create NAT Gateway in public subnet 1a
aws ec2 create-nat-gateway \
  --subnet-id $PUB_SUBNET_1A \
  --allocation-id $EIP_ALLOC \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=url-nat}]'

export NAT_GW=nat-XXXXXXXXX

# Wait for NAT GW to become available (~60 seconds)
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW

# Create PRIVATE route table
aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=private-rt}]'

export PRIVATE_RT=rtb-XXXXXXXXX

# Route outbound traffic through NAT
aws ec2 create-route \
  --route-table-id $PRIVATE_RT \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW

# Associate private subnets with private route table
aws ec2 associate-route-table \
  --route-table-id $PRIVATE_RT \
  --subnet-id $PRI_SUBNET_1A

aws ec2 associate-route-table \
  --route-table-id $PRIVATE_RT \
  --subnet-id $PRI_SUBNET_1B
```

---

## STEP 4 — Security Groups

```bash
# --- ALB Security Group (accepts traffic from the internet) ---
aws ec2 create-security-group \
  --group-name alb-sg \
  --description "Allow HTTP and HTTPS from internet" \
  --vpc-id $VPC_ID

export ALB_SG=sg-XXXXXXXXX

aws ec2 authorize-security-group-ingress \
  --group-id $ALB_SG \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id $ALB_SG \
  --protocol tcp --port 443 --cidr 0.0.0.0/0


# --- EC2 Security Group (only accepts traffic FROM the ALB) ---
aws ec2 create-security-group \
  --group-name ec2-sg \
  --description "Allow traffic only from ALB" \
  --vpc-id $VPC_ID

export EC2_SG=sg-XXXXXXXXX

aws ec2 authorize-security-group-ingress \
  --group-id $EC2_SG \
  --protocol tcp --port 3000 \
  --source-group $ALB_SG


# --- RDS Security Group (only accepts traffic FROM EC2) ---
aws ec2 create-security-group \
  --group-name rds-sg \
  --description "Allow Postgres from EC2 only" \
  --vpc-id $VPC_ID

export RDS_SG=sg-XXXXXXXXX

aws ec2 authorize-security-group-ingress \
  --group-id $RDS_SG \
  --protocol tcp --port 5432 \
  --source-group $EC2_SG


# --- Redis Security Group (only accepts traffic FROM EC2) ---
aws ec2 create-security-group \
  --group-name redis-sg \
  --description "Allow Redis from EC2 only" \
  --vpc-id $VPC_ID

export REDIS_SG=sg-XXXXXXXXX

aws ec2 authorize-security-group-ingress \
  --group-id $REDIS_SG \
  --protocol tcp --port 6379 \
  --source-group $EC2_SG
```

---

## STEP 5 — RDS PostgreSQL Multi-AZ

```bash
# Create DB Subnet Group (required by RDS)
aws rds create-db-subnet-group \
  --db-subnet-group-name url-db-subnet-group \
  --db-subnet-group-description "Private subnets for RDS" \
  --subnet-ids $PRI_SUBNET_1A $PRI_SUBNET_1B

# Store DB password in Secrets Manager (more secure than plain text)
aws secretsmanager create-secret \
  --name url-shortener/db-password \
  --secret-string "YourStrongPassword123!"

# Create RDS instance (Multi-AZ enabled)
aws rds create-db-instance \
  --db-instance-identifier url-shortener-db \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --engine-version 15.4 \
  --master-username urlapp \
  --master-user-password "YourStrongPassword123!" \
  --allocated-storage 20 \
  --storage-type gp3 \
  --multi-az \
  --db-subnet-group-name url-db-subnet-group \
  --vpc-security-group-ids $RDS_SG \
  --backup-retention-period 7 \
  --storage-encrypted \
  --no-publicly-accessible \
  --tags Key=Name,Value=url-shortener-db

# Wait for RDS to become available (~10 minutes)
aws rds wait db-instance-available \
  --db-instance-identifier url-shortener-db

# Get the RDS endpoint (save this — your app needs it)
aws rds describe-db-instances \
  --db-instance-identifier url-shortener-db \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text
```

### Create the database schema (run from an EC2 instance in the VPC):

```sql
-- Connect: psql -h <rds-endpoint> -U urlapp -d postgres
CREATE DATABASE urlshortener;
\c urlshortener

CREATE TABLE urls (
  id          BIGSERIAL PRIMARY KEY,
  short_code  VARCHAR(10) UNIQUE NOT NULL,
  long_url    TEXT NOT NULL,
  clicks      BIGINT DEFAULT 0,
  created_at  TIMESTAMP DEFAULT NOW(),
  expires_at  TIMESTAMP
);

CREATE INDEX idx_short_code ON urls(short_code);
```

---

## STEP 6 — ElastiCache Redis

```bash
# Create subnet group for ElastiCache
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name url-redis-subnet-group \
  --cache-subnet-group-description "Private subnets for Redis" \
  --subnet-ids $PRI_SUBNET_1A $PRI_SUBNET_1B

# Create Redis cluster
aws elasticache create-cache-cluster \
  --cache-cluster-id url-redis \
  --engine redis \
  --engine-version 7.0 \
  --cache-node-type cache.t3.micro \
  --num-cache-nodes 1 \
  --cache-subnet-group-name url-redis-subnet-group \
  --security-group-ids $REDIS_SG \
  --tags Key=Name,Value=url-redis

# Wait for Redis to be available (~5 minutes)
aws elasticache wait cache-cluster-available \
  --cache-cluster-id url-redis

# Get the Redis endpoint
aws elasticache describe-cache-clusters \
  --cache-cluster-id url-redis \
  --show-cache-node-info \
  --query 'CacheClusters[0].CacheNodes[0].Endpoint.Address' \
  --output text
```

---

## STEP 7 — Application Code (Node.js)

```bash
# Create project directory on your local machine
mkdir url-shortener && cd url-shortener
npm init -y
npm install express pg redis nanoid dotenv
```

### `app.js`

```javascript
const express = require('express');
const { Pool } = require('pg');
const { createClient } = require('redis');
const { nanoid } = require('nanoid');
require('dotenv').config();

const app = express();
app.use(express.json());

// PostgreSQL connection
const db = new Pool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  port: 5432,
  ssl: { rejectUnauthorized: false }
});

// Redis connection
const redis = createClient({ url: `redis://${process.env.REDIS_HOST}:6379` });
redis.connect().catch(console.error);

// Health check (ALB pings this)
app.get('/health', (req, res) => res.json({ status: 'ok' }));

// Create short URL
app.post('/api/shorten', async (req, res) => {
  const { longUrl } = req.body;
  if (!longUrl) return res.status(400).json({ error: 'longUrl required' });

  const shortCode = nanoid(7);
  await db.query(
    'INSERT INTO urls (short_code, long_url) VALUES ($1, $2)',
    [shortCode, longUrl]
  );
  res.json({ shortUrl: `https://${process.env.DOMAIN}/${shortCode}` });
});

// Redirect (cache-aside pattern)
app.get('/:code', async (req, res) => {
  const { code } = req.params;

  // 1. Check Redis cache first
  const cached = await redis.get(`short:${code}`);
  if (cached) {
    await db.query('UPDATE urls SET clicks = clicks + 1 WHERE short_code = $1', [code]);
    return res.redirect(301, cached);
  }

  // 2. Cache miss — query DB
  const result = await db.query(
    'SELECT long_url FROM urls WHERE short_code = $1', [code]
  );
  if (result.rows.length === 0) return res.status(404).send('Not found');

  const longUrl = result.rows[0].long_url;

  // 3. Populate cache with 24h TTL
  await redis.setEx(`short:${code}`, 86400, longUrl);
  await db.query('UPDATE urls SET clicks = clicks + 1 WHERE short_code = $1', [code]);

  res.redirect(301, longUrl);
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

### `.env`

```bash
DB_HOST=your-rds-endpoint.rds.amazonaws.com
DB_USER=urlapp
DB_PASSWORD=YourStrongPassword123!
DB_NAME=urlshortener
REDIS_HOST=your-redis-endpoint.cache.amazonaws.com
DOMAIN=short.yourdomain.com
```

---

## STEP 8 — Create AMI (EC2 Launch Template)

```bash
# Launch a temporary EC2 to build the AMI
aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \
  --instance-type t3.medium \
  --key-name your-key-pair \
  --subnet-id $PUB_SUBNET_1A \
  --security-group-ids $EC2_SG \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=url-builder}]'

export BUILDER_INSTANCE=i-XXXXXXXXX
```

### On the builder EC2 (SSH in and run):

```bash
# Install Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install PM2 (process manager)
sudo npm install -g pm2

# Copy your app files here, then:
npm install
pm2 start app.js --name url-shortener
pm2 startup systemd
pm2 save

# Test it works
curl http://localhost:3000/health
```

```bash
# Back on your local machine — create AMI from the builder instance
aws ec2 create-image \
  --instance-id $BUILDER_INSTANCE \
  --name "url-shortener-ami-v1" \
  --no-reboot

export AMI_ID=ami-XXXXXXXXX

# Wait for AMI to be available
aws ec2 wait image-available --image-ids $AMI_ID

# Terminate the builder instance
aws ec2 terminate-instances --instance-ids $BUILDER_INSTANCE
```

---

## STEP 9 — IAM Role for EC2

```bash
# Create trust policy file
cat > ec2-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Create the role
aws iam create-role \
  --role-name url-shortener-ec2-role \
  --assume-role-policy-document file://ec2-trust-policy.json

# Attach policies (SSM for remote access, Secrets for DB password)
aws iam attach-role-policy \
  --role-name url-shortener-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam attach-role-policy \
  --role-name url-shortener-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite

# Create instance profile and attach role
aws iam create-instance-profile \
  --instance-profile-name url-shortener-profile

aws iam add-role-to-instance-profile \
  --instance-profile-name url-shortener-profile \
  --role-name url-shortener-ec2-role
```

---

## STEP 10 — Launch Template + Auto Scaling Group

```bash
# Create Launch Template
aws ec2 create-launch-template \
  --launch-template-name url-shortener-lt \
  --version-description "v1" \
  --launch-template-data '{
    "ImageId": "'"$AMI_ID"'",
    "InstanceType": "t3.medium",
    "IamInstanceProfile": { "Name": "url-shortener-profile" },
    "SecurityGroupIds": ["'"$EC2_SG"'"],
    "UserData": "'"$(base64 -w 0 <<'USERDATA'
#!/bin/bash
cd /home/ubuntu/url-shortener
pm2 restart url-shortener
USERDATA
)"'",
    "TagSpecifications": [{
      "ResourceType": "instance",
      "Tags": [{"Key": "Name", "Value": "url-shortener-ec2"}]
    }]
  }'

export LT_ID=lt-XXXXXXXXX

# Create Target Group (ALB needs this first)
aws elbv2 create-target-group \
  --name url-shortener-tg \
  --protocol HTTP \
  --port 3000 \
  --vpc-id $VPC_ID \
  --health-check-protocol HTTP \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --target-type instance

export TG_ARN=arn:aws:elasticloadbalancing:us-east-1:XXXX:targetgroup/url-shortener-tg/XXXX

# Create Auto Scaling Group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name url-shortener-asg \
  --launch-template LaunchTemplateId=$LT_ID,Version=1 \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 2 \
  --vpc-zone-identifier "$PRI_SUBNET_1A,$PRI_SUBNET_1B" \
  --target-group-arns $TG_ARN \
  --health-check-type ELB \
  --health-check-grace-period 120 \
  --tags Key=Name,Value=url-shortener-asg,PropagateAtLaunch=true

# Add CPU-based Auto Scaling Policy (scale out when CPU > 70%)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name url-shortener-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 70.0,
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

---

## STEP 11 — Application Load Balancer

```bash
# Create ALB in the PUBLIC subnets
aws elbv2 create-load-balancer \
  --name url-shortener-alb \
  --subnets $PUB_SUBNET_1A $PUB_SUBNET_1B \
  --security-groups $ALB_SG \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --tags Key=Name,Value=url-shortener-alb

export ALB_ARN=arn:aws:elasticloadbalancing:us-east-1:XXXX:loadbalancer/app/url-shortener-alb/XXXX
export ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' --output text)

echo "ALB DNS: $ALB_DNS"

# Create HTTP listener (redirects to HTTPS)
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions '[{
    "Type": "redirect",
    "RedirectConfig": {
      "Protocol": "HTTPS",
      "Port": "443",
      "StatusCode": "HTTP_301"
    }
  }]'

# Create HTTPS listener (requires ACM certificate — see Step 12)
# Run this AFTER getting your certificate ARN
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=$CERT_ARN \
  --default-actions '[{
    "Type": "forward",
    "TargetGroupArn": "'"$TG_ARN"'"
  }]'
```

---

## STEP 12 — ACM Certificate + Route 53

```bash
# Request a certificate (replace with your actual domain)
aws acm request-certificate \
  --domain-name short.yourdomain.com \
  --validation-method DNS \
  --region us-east-1

export CERT_ARN=arn:aws:acm:us-east-1:XXXX:certificate/XXXX

# Get DNS validation records (add these to your DNS registrar or Route 53)
aws acm describe-certificate \
  --certificate-arn $CERT_ARN \
  --query 'Certificate.DomainValidationOptions'

# --- Route 53 Hosted Zone ---
aws route53 create-hosted-zone \
  --name yourdomain.com \
  --caller-reference $(date +%s)

export ZONE_ID=ZXXXXXXXXX

# Create alias A record pointing to ALB
cat > r53-alb-record.json << EOF
{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "short.yourdomain.com",
      "Type": "A",
      "AliasTarget": {
        "HostedZoneId": "Z35SXDOTRQ7X7K",
        "DNSName": "$ALB_DNS",
        "EvaluateTargetHealth": true
      }
    }
  }]
}
EOF

aws route53 change-resource-record-sets \
  --hosted-zone-id $ZONE_ID \
  --change-batch file://r53-alb-record.json

# Create health check on ALB
aws route53 create-health-check \
  --caller-reference $(date +%s) \
  --health-check-config '{
    "Type": "HTTPS",
    "FullyQualifiedDomainName": "short.yourdomain.com",
    "ResourcePath": "/health",
    "RequestInterval": 30,
    "FailureThreshold": 3
  }'
```

---

## STEP 13 — S3 Bucket (Frontend Static Assets)

```bash
# Create S3 bucket for React frontend
aws s3 mb s3://url-shortener-frontend-$(date +%s) --region us-east-1

export S3_BUCKET=url-shortener-frontend-XXXXXXXXX

# Block all public access (CloudFront will serve it, not public S3 URLs)
aws s3api put-public-access-block \
  --bucket $S3_BUCKET \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,\
    BlockPublicPolicy=true,RestrictPublicBuckets=true

# Build and deploy your React app
# npm run build (in your frontend project)
aws s3 sync ./build s3://$S3_BUCKET --delete
```

---

## STEP 14 — CloudFront Distribution

```bash
# Create CloudFront distribution
aws cloudfront create-distribution \
  --distribution-config '{
    "CallerReference": "url-shortener-dist-1",
    "Comment": "URL Shortener CDN",
    "Enabled": true,
    "DefaultRootObject": "index.html",
    "Origins": {
      "Quantity": 2,
      "Items": [
        {
          "Id": "s3-origin",
          "DomainName": "'"$S3_BUCKET"'.s3.amazonaws.com",
          "S3OriginConfig": { "OriginAccessIdentity": "" }
        },
        {
          "Id": "alb-origin",
          "DomainName": "'"$ALB_DNS"'",
          "CustomOriginConfig": {
            "HTTPSPort": 443,
            "OriginProtocolPolicy": "https-only"
          }
        }
      ]
    },
    "DefaultCacheBehavior": {
      "TargetOriginId": "s3-origin",
      "ViewerProtocolPolicy": "redirect-to-https",
      "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
      "Compress": true,
      "AllowedMethods": { "Quantity": 2, "Items": ["GET","HEAD"] }
    },
    "CacheBehaviors": {
      "Quantity": 2,
      "Items": [
        {
          "PathPattern": "/api/*",
          "TargetOriginId": "alb-origin",
          "ViewerProtocolPolicy": "redirect-to-https",
          "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
          "AllowedMethods": {
            "Quantity": 7,
            "Items": ["GET","HEAD","OPTIONS","PUT","POST","PATCH","DELETE"]
          }
        },
        {
          "PathPattern": "/?*",
          "TargetOriginId": "alb-origin",
          "ViewerProtocolPolicy": "redirect-to-https",
          "DefaultTTL": 60,
          "MaxTTL": 60,
          "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
          "AllowedMethods": { "Quantity": 2, "Items": ["GET","HEAD"] }
        }
      ]
    },
    "PriceClass": "PriceClass_100",
    "Aliases": { "Quantity": 1, "Items": ["short.yourdomain.com"] },
    "ViewerCertificate": {
      "ACMCertificateArn": "'"$CERT_ARN"'",
      "SSLSupportMethod": "sni-only",
      "MinimumProtocolVersion": "TLSv1.2_2021"
    }
  }'
```

---

## STEP 15 — WAF (Web Application Firewall)

```bash
# Create WAF Web ACL (must be in us-east-1 for CloudFront)
aws wafv2 create-web-acl \
  --name url-shortener-waf \
  --scope CLOUDFRONT \
  --region us-east-1 \
  --default-action Allow={} \
  --rules '[
    {
      "Name": "RateLimitRule",
      "Priority": 1,
      "Statement": {
        "RateBasedStatement": {
          "Limit": 1000,
          "AggregateKeyType": "IP"
        }
      },
      "Action": { "Block": {} },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "RateLimitRule"
      }
    },
    {
      "Name": "AWSManagedRulesCommonRuleSet",
      "Priority": 2,
      "OverrideAction": { "None": {} },
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesCommonRuleSet"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "AWSManagedRulesCommonRuleSet"
      }
    }
  ]' \
  --visibility-config \
    SampledRequestsEnabled=true,\
    CloudWatchMetricsEnabled=true,\
    MetricName=url-shortener-waf
```

---

## STEP 16 — CloudWatch Monitoring + Alarms

```bash
# --- SNS Topic for alerts ---
aws sns create-topic --name url-shortener-alerts
export SNS_ARN=arn:aws:sns:us-east-1:XXXX:url-shortener-alerts

aws sns subscribe \
  --topic-arn $SNS_ARN \
  --protocol email \
  --notification-endpoint your-email@example.com

# --- ALARM 1: High error rate on ALB ---
aws cloudwatch put-metric-alarm \
  --alarm-name "ALB-High-5xx-Rate" \
  --alarm-description "ALB 5xx errors exceed 1%" \
  --metric-name HTTPCode_Target_5XX_Count \
  --namespace AWS/ApplicationELB \
  --dimensions Name=LoadBalancer,Value=$(echo $ALB_ARN | cut -d: -f6) \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions $SNS_ARN \
  --treat-missing-data notBreaching

# --- ALARM 2: RDS CPU high ---
aws cloudwatch put-metric-alarm \
  --alarm-name "RDS-High-CPU" \
  --alarm-description "RDS CPU exceeds 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/RDS \
  --dimensions Name=DBInstanceIdentifier,Value=url-shortener-db \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions $SNS_ARN

# --- ALARM 3: Auto Scaling activity ---
aws cloudwatch put-metric-alarm \
  --alarm-name "ASG-Max-Capacity-Reached" \
  --alarm-description "ASG at maximum capacity" \
  --metric-name GroupMaxSize \
  --namespace AWS/AutoScaling \
  --dimensions Name=AutoScalingGroupName,Value=url-shortener-asg \
  --statistic Maximum \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 10 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions $SNS_ARN

# --- ALARM 4: Redis cache hit rate drop ---
aws cloudwatch put-metric-alarm \
  --alarm-name "Redis-Low-Cache-Hit-Rate" \
  --alarm-description "Redis cache hit ratio below 70%" \
  --metric-name CacheHitRate \
  --namespace AWS/ElastiCache \
  --dimensions Name=CacheClusterId,Value=url-redis \
  --statistic Average \
  --period 300 \
  --evaluation-periods 3 \
  --threshold 70 \
  --comparison-operator LessThanThreshold \
  --alarm-actions $SNS_ARN

# --- CloudWatch Dashboard ---
aws cloudwatch put-dashboard \
  --dashboard-name url-shortener-dashboard \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric", "x": 0, "y": 0, "width": 12, "height": 6,
        "properties": {
          "title": "ALB Request Count",
          "metrics": [["AWS/ApplicationELB","RequestCount","LoadBalancer","'"$(echo $ALB_ARN | cut -d: -f6)"'"]],
          "period": 60, "stat": "Sum"
        }
      },
      {
        "type": "metric", "x": 12, "y": 0, "width": 12, "height": 6,
        "properties": {
          "title": "p99 Latency (ms)",
          "metrics": [["AWS/ApplicationELB","TargetResponseTime","LoadBalancer","'"$(echo $ALB_ARN | cut -d: -f6)"'"]],
          "period": 60, "stat": "p99"
        }
      }
    ]
  }'
```

---

## STEP 17 — Verify Everything is Working

```bash
# 1. Check Auto Scaling Group health
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names url-shortener-asg \
  --query 'AutoScalingGroups[0].Instances[*].{ID:InstanceId,State:HealthStatus,AZ:AvailabilityZone}'

# 2. Check target group — all instances should show "healthy"
aws elbv2 describe-target-health \
  --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[*].{Instance:Target.Id,State:TargetHealth.State}'

# 3. Check RDS is Multi-AZ and available
aws rds describe-db-instances \
  --db-instance-identifier url-shortener-db \
  --query 'DBInstances[0].{Status:DBInstanceStatus,MultiAZ:MultiAZ,AZ:AvailabilityZone}'

# 4. Test the API end-to-end
curl -X POST https://short.yourdomain.com/api/shorten \
  -H "Content-Type: application/json" \
  -d '{"longUrl": "https://www.example.com/very/long/url/here"}'

# Expected: {"shortUrl":"https://short.yourdomain.com/Ab3xY7z"}

# 5. Test the redirect
curl -I https://short.yourdomain.com/Ab3xY7z
# Expected: HTTP/2 301, Location: https://www.example.com/very/long/url/here
```

---

## STEP 18 — Test High Availability (simulate failure)

```bash
# Force terminate one EC2 instance — ASG should replace it within 3 minutes
INSTANCE_ID=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names url-shortener-asg \
  --query 'AutoScalingGroups[0].Instances[0].InstanceId' \
  --output text)

aws ec2 terminate-instances --instance-ids $INSTANCE_ID

# Watch ASG spin up a replacement
watch -n 10 'aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names url-shortener-asg \
  --query "AutoScalingGroups[0].Instances[*].{ID:InstanceId,State:LifecycleState}"'

# Test RDS failover (takes ~60 seconds)
aws rds reboot-db-instance \
  --db-instance-identifier url-shortener-db \
  --force-failover

# Your app should continue working during failover via Redis cache
```

---

## Architecture Summary

```
Internet
    │
    ▼
Route 53 (DNS + health checks)
    │
    ▼
CloudFront (CDN + WAF + HTTPS)
    │
    ├──► S3 (React static assets — served at edge)
    │
    ▼
Application Load Balancer (public subnets, multi-AZ)
    │
    ├──► EC2 (AZ 1a, private subnet) ──► Redis ──► RDS Primary (1a)
    │                                                     │ sync replication
    └──► EC2 (AZ 1b, private subnet) ──► Redis ──► RDS Standby (1b)
```

---

## Key Numbers to Memorise for Interviews

| Metric | Value |
|---|---|
| RDS failover time | ~60 seconds |
| ASG instance replacement | ~3 minutes |
| Route 53 health check interval | 30 seconds |
| Redis cache TTL | 86,400s (24 hours) |
| ALB health check | /health every 30s |
| Auto-scale trigger | CPU > 70% for 2 min |
| Scale-in cooldown | 300 seconds |
| Base62 codes (7 chars) | 3.5 trillion unique URLs |
| Estimated monthly cost | ~$145–200 |
| Target uptime SLA | 99.99% |
