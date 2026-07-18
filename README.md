# Step-by-Step Process — AWS HA & Secure Architecture (Multi-AZ, ALB + ASG)

This document walks through how this architecture was built, step by step.

---

## 1. VPC Creation

- Went to **VPC → Create VPC → VPC and more** (this auto-creates subnets, route tables, IGW).
- **Name tag auto-generation:** `AWS-PROD-example`
- **IPv4 CIDR:** `10.0.0.0/16`
- **Number of Availability Zones:** 2
- **Number of public subnets:** 2
- **Number of private subnets:** 2
- **NAT Gateways:** 1 per AZ (zonal)
- **VPC endpoints:** None
- **DNS options:** Enabled DNS resolution
- Clicked **Create VPC**

This automatically provisions the VPC, 2 public subnets, 2 private subnets, an Internet Gateway, NAT Gateways (1 per AZ), and the associated route tables (1 public route table, 2 private route table per AZ).

---

## 2. Bastion Host Setup

- Launched an EC2 instance named Bastian host in a **public subnet** of the created VPC to act as the Bastion host.
- Created a **Security Group** for the Bastion:
  - Inbound rule: **SSH** allowed only from **my IP**

---

## 3. Launch Template (for Auto Scaling Group)

- Went to **EC2 → Launch Templates → Create launch template**
- **Name:** `aws-prod-example`
- **Template version description:** `App deployment in private subnets`
- Checked **"Provide guidance to help me set up a template that I can use with EC2 Auto Scaling"**
- **AMI:** Browsed and selected an Ubuntu free-tier AMI
- **Instance type:** `t3.micro`
- **Key pair:** Selected an existing key pair (or created a new one)
- **Network settings:**
  - Subnet: Left as **"Don't include in launch template"** — since instances will be launched across 2 private subnets, the subnet is chosen at the Auto Scaling Group level instead of being fixed in the template.
  - Created a new **Security Group** for the app instances:
    - **Name:** `template-sg`
    - **VPC:** Selected the VPC created above
    - **Inbound rules:**
      - Type: SSH → Source: Bastion Security Group
      - Type: Custom TCP → Port: `8000` → Source: Anywhere (updated to the ALB's security group once the ALB was created)
- Clicked **Create launch template**

---

## 4. Auto Scaling Group (ASG) Creation

- **Name:** `aws-prod-example`
- **Launch template:** Selected the launch template created in Step 3
- **Subnets/VPC:** Selected the VPC and the 2 private subnets
- **Availability Zone distribution:** Selected **"Balanced best effort"** — Auto Scaling tries to maintain the desired capacity even if one AZ can't launch instances. The ALB continues routing traffic only to healthy instances, so the app stays available even if instances aren't perfectly evenly distributed across AZs.
- **Capacity:**
  - Minimum: 1
  - Desired: 2
  - Maximum: 4

---

## 5. Connecting to Private EC2 Instances via Bastion

Since the app instances live in private subnets, the Bastion host is used as a jump box:

1. Copy the PEM key file to the Bastion host:
   ```
   scp -i /path/to/key.pem /path/to/key.pem ubuntu@<bastion-public-ip>:/home/ubuntu
   ```
   (To find the local path of your `.pem` file: `ls | grep Project.pem` or `pwd` in the folder where it was downloaded.)

2. SSH into the Bastion host, then from there SSH into the private EC2 instance using the same key.

ssh -i key.pem ubuntu@public_ip_of_ec2

---

## 6. Deploy a Test Application

- Connected to a private EC2 instance (via Bastion).
- Created a simple `index.html` file (sample template from W3Schools).
- Started a quick test web server:
  ```
  python3 -m http.server 8000
  ```

---

## 7. Load Balancer & Target Group Creation

**Target Group:**
- Target type: Instances
- Name: `aws-prod-example-tg`
- Protocol: HTTP, Port: `8000`
- VPC: Selected the created VPC
- Health check: HTTP
- Registered the private EC2 instances as targets
- Created the Target Group

**Application Load Balancer:**
- Name: `aws-prod-example-alb`
- Scheme: Internet-facing
- VPC: Selected the created VPC, across 2 AZs, using the 2 public subnets
- Security Group: Created a new SG allowing **HTTP (port 80)** from the internet
- Listener: HTTP:80 → Forward to the Target Group created above

Finally, updated the instance security group inbound rule for port `8000` from allow all traffic to the ALB's security group, replacing the temporary "Anywhere" rule.

---

## Result

- Traffic flows: **Internet → IGW → ALB (public subnets) → Target Group → EC2 instances (private subnets)**
- Auto Scaling Group maintains 2–4 healthy instances across both AZs based on load.
- Only the Bastion host is SSH-reachable from the internet (and only from a whitelisted IP); app instances are only reachable via the Bastion or the ALB.