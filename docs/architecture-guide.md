# Build Log — Console Steps & Field-Level Configuration

This document is the granular, field-by-field build log behind the narrative in the [top-level README](../README.md). Use it as a checklist if you're reproducing this architecture in your own account. Region used throughout: **Mumbai (ap-south-1)**.

---

## Step 1: Create the VPC

- VPC Dashboard → Create VPC
- Name: `MyVPC`
- CIDR: `10.0.0.0/16`

## Step 2: Create Subnets

| Subnet | Type | CIDR | AZ |
|---|---|---|---|
| Public-1 | Public | 10.0.1.0/24 | AZ1 |
| Public-2 | Public | 10.0.2.0/24 | AZ2 |
| Private-App-1 | Private | 10.0.3.0/24 | AZ1 |
| Private-App-2 | Private | 10.0.4.0/24 | AZ2 |
| Private-DB-1 | Private | 10.0.5.0/24 | AZ1 |
| Private-DB-2 | Private | 10.0.6.0/24 | AZ2 |

*(Splitting App and DB into separate private subnets is optional but a good habit — it lets you tighten routing/NACLs per tier later. To keep it simpler, you could reuse Private-1/Private-2 for both app and DB.)*

## Step 3: Internet Connectivity

**3a. Internet Gateway (for public subnets)**
- Create an Internet Gateway → attach to `MyVPC`
- Create a **Public Route Table**
  - Route: `0.0.0.0/0 → Internet Gateway`
  - Associate with Public-1 and Public-2

**3b. NAT Gateway (for private subnets)**
- Allocate an Elastic IP
- Create a NAT Gateway in Public-1, attach the Elastic IP
- Create a **Private Route Table**
  - Route: `0.0.0.0/0 → NAT Gateway`
  - Associate with all four private subnets

> Without this, EC2 instances in the private subnets have no path to the internet, so the `yum install` commands in the boot script would fail.
>
> **Cost note:** NAT Gateway is **not** free-tier eligible — it bills hourly plus per-GB processed. One NAT Gateway is fine for a learning project. For production HA, put a NAT Gateway in each AZ so one AZ's outage doesn't take down the other AZ's outbound traffic.

## Step 4: Security Groups

**1. ALB-SG**
- Inbound: HTTP (80) from `0.0.0.0/0`
- Inbound: HTTPS (443) from `0.0.0.0/0`

**2. EC2-SG**
- Inbound: HTTP (80) from `ALB-SG` only

**3. RDS-SG**
- Inbound: MySQL (3306) from `EC2-SG` only
- Without this, the database either has no valid inbound rule (app can't connect) or someone opens it too broadly (security risk) — so it's created explicitly, scoped to just the app tier's security group.

## Step 5: IAM Role for EC2

Auto Scaling launches instances with no way to manage or monitor them unless a role is attached.

- IAM → Roles → Create Role → EC2 use case
- Attach managed policies:
  - `AmazonSSMManagedInstanceCore` (enables connecting via **Session Manager** instead of SSH/bastion — no open inbound port needed)
  - `CloudWatchAgentServerPolicy` (lets the instance push custom metrics/logs)
- Name it `EC2-App-Role` — attach this to the Launch Template in Step 6

## Step 6: Create a Launch Template

Auto Scaling needs a template to launch from, rather than a standalone instance:

- EC2 → Launch Templates → Create
- Name: `rs-app-server-LT`
- AMI: Amazon Linux
- Instance type: `t3.micro`
- Network: `MyVPC`, subnet left to the Auto Scaling Group (Step 9) to choose across AZs
- Security group: `EC2-SG`
- IAM instance profile: `EC2-App-Role`
- User data:
```bash
#!/bin/bash
yum update -y
amazon-linux-extras install nginx1 -y
systemctl start nginx
systemctl enable nginx
echo "<h1>Hello from AWS Server</h1>" > /usr/share/nginx/html/index.html
```
> `amazon-linux-extras` is for the **Amazon Linux 2** AMI. If you pick **Amazon Linux 2023** instead, use `dnf install nginx -y` (or `yum install nginx -y`, symlinked to dnf) — `amazon-linux-extras` doesn't exist on AL2023.

## Step 7: Create Target Group + Load Balancer

**Target Group**
- Type: Instances, Protocol: HTTP 80
- Health check path: `/` (or a dedicated `/health` endpoint if your app has one)
- Target optimizer: leave **Off (Default)** — it needs an agent installed on the instances and isn't relevant for a simple web app
- This is what tells the ALB when an instance is actually healthy.

**Application Load Balancer**
- EC2 → Load Balancers → Create → Application Load Balancer
- Scheme: internet-facing
- Subnets: Public-1, Public-2
- Security group: `ALB-SG`
- Listener: HTTPS 443 → attach your `redsparrowenterprise.in` ACM certificate → forward to the Target Group above (TLS terminates here; traffic to the targets stays plain HTTP since it never leaves the private subnets)
- Listener: HTTP 80 → **redirect** to HTTPS:443 (not forward) — so plain `http://` requests get bounced to `https://` instead of served insecurely
- Route 53: point your domain's A record (alias) at this ALB

## Step 8: Create RDS Database

- RDS → Create database → MySQL, Free tier (or appropriate tier)
- **DB Subnet Group**: create this explicitly *before* the "Create database" wizard, via RDS console → **Subnet groups** → Create DB subnet group → select `MyVPC` → add AZs ap-south-1a/1b → explicitly check Private-DB-1 and Private-DB-2. Then select this named group in the wizard's "DB subnet group" dropdown.
  - Don't use the wizard's inline "Create new DB Subnet Group" option — it auto-selects subnets across your VPC without letting you choose which ones, so it won't reliably stick to just your two intended private subnets.
- VPC security group: `RDS-SG`
- Public access: **No**
- Set master username & password
- Multi-AZ deployment: enable if you want automatic failover (recommended, small cost increase)
- Enable automated backups (set a retention period, e.g. 7 days)

## Step 9: Auto Scaling Group

**Create the ASG**
- EC2 → Auto Scaling Groups → Create
- Launch template: the one from Step 6
- VPC subnets: Private-App-1, Private-App-2 (both AZs — this is what actually gives you high availability)
- Attach to the Target Group from Step 7
- Min: 2, Max: 4, Desired: 2

**Add a scaling policy**
Min/Max alone don't make the group scale; a policy tells it *when*:
- Policy type: **Target tracking scaling**
- Metric: Average CPU Utilization
- Target value: 60%
- This automatically adds instances when average CPU crosses 60% and removes them when load drops, within your Min/Max bounds.

## Step 10: Monitoring

Metrics alone don't notify anyone — they need to be turned into an action:

- CloudWatch → Alarms → Create Alarm
  - CPU Utilization > 80% for 5 minutes → notify an **SNS topic** (email/SMS)
  - Unhealthy host count > 0 on the Target Group → notify SNS
- Enable **detailed monitoring** on the Launch Template if you want 1-minute granularity instead of 5-minute

---

## Optional Next-Level Hardening

- **Secrets Manager**: store the RDS username/password there instead of hardcoding it anywhere in the app
- **WAF**: attach AWS WAF to the ALB for basic protection against common web exploits
- **Infrastructure as Code**: once this works via console, rebuilding it in Terraform or CloudFormation makes it repeatable and makes mistakes like the missing NAT Gateway visible in a code review instead of a failed deployment
