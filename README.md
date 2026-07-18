# Step-by-Step Process — AWS Architecture (Multi-AZ, ALB + ASG)

This document walks through how this architecture was built, step by step.

Referance screenshots:
<img width="693" height="708" alt="Screenshot 2026-07-18 211841" src="https://github.com/user-attachments/assets/1384b8f8-4a7b-4e32-bd78-803595854edb" />

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

Referance screenshots:

<img width="1920" height="1080" alt="Screenshot (330)" src="https://github.com/user-attachments/assets/f92f2b70-f906-44e4-beea-8840c92db54c" />

<img width="1920" height="1080" alt="Screenshot (331)" src="https://github.com/user-attachments/assets/5b85a395-109f-4025-aad0-f6be2e144fdb" />

<img width="1920" height="1080" alt="Screenshot (332)" src="https://github.com/user-attachments/assets/a7dce5c6-0bca-4fb0-995a-30b533645dcb" />

---

## 2. Bastion Host Setup

- Launched an EC2 instance named Bastian host in a **public subnet** of the created VPC to act as the Bastion host.
- Created a **Security Group** for the Bastion:
  - Inbound rule: **SSH** allowed only from **my IP**

Referance screenshots:

<img width="1920" height="1080" alt="Screenshot (333)" src="https://github.com/user-attachments/assets/bbb482ee-a824-49dc-bc44-b36394867698" />

<img width="1920" height="1080" alt="Screenshot (334)" src="https://github.com/user-attachments/assets/b31bf4c9-f199-4613-9bf9-ea1e123e95b8" />

<img width="1920" height="1080" alt="Screenshot (350)" src="https://github.com/user-attachments/assets/39c84952-aaf4-45cc-8c0d-b93a753bf12f" />

<img width="1920" height="1080" alt="Screenshot (336)" src="https://github.com/user-attachments/assets/11ae5d77-efed-4356-a769-8fc1ef8e3f03" />

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

Referance screenshots:

<img width="1920" height="1080" alt="Screenshot (338)" src="https://github.com/user-attachments/assets/ea0a231b-dee2-4415-8802-bf9ab100cc6c" />

<img width="1920" height="1080" alt="Screenshot (339)" src="https://github.com/user-attachments/assets/059bf699-5414-4208-87f6-7c05a3917fa4" />

<img width="1920" height="1080" alt="Screenshot (340)" src="https://github.com/user-attachments/assets/1d9151de-1dfa-42ba-b4b2-ac80657707e7" />

<img width="1920" height="1080" alt="Screenshot (341)" src="https://github.com/user-attachments/assets/8bfb5522-633c-4a4d-b8e2-ae980b2ec44e" />

<img width="1920" height="1080" alt="Screenshot (342)" src="https://github.com/user-attachments/assets/0df5c237-f4ff-424c-b047-71760a9d752e" />

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
  - click on next,... create.

Referance screenshots:

<img width="1920" height="1080" alt="Screenshot (343)" src="https://github.com/user-attachments/assets/79a64124-ae1a-4520-a806-b5104b591590" />

<img width="1920" height="1080" alt="Screenshot (344)" src="https://github.com/user-attachments/assets/9c38437e-cd5e-43e8-8332-9143722f4046" />

<img width="1920" height="1080" alt="Screenshot (345)" src="https://github.com/user-attachments/assets/b60d3947-9718-4156-96cb-5eace2884088" />

<img width="1920" height="1080" alt="Screenshot (346)" src="https://github.com/user-attachments/assets/3a61ede5-128d-4e3a-84d9-0175aae73e45" />

<img width="1920" height="1080" alt="Screenshot (347)" src="https://github.com/user-attachments/assets/e635b1ad-3cb6-4f43-94f5-3a4493adf31d" />

<img width="1920" height="1080" alt="Screenshot (348)" src="https://github.com/user-attachments/assets/d73d2894-d6d6-4a9e-9f72-160b027be7ea" />

---

## 5. Connecting to Private EC2 Instances via Bastion and Deploy a Test Application

Since the app instances live in private subnets, the Bastion host is used as a jump box:

1. Copy the PEM key file to the Bastion host:
   ```
   scp -i /path/to/key.pem /path/to/key.pem ubuntu@<bastion-public-ip>:/home/ubuntu
   ```
   (To find the local path of your `.pem` file: `ls | grep Project.pem` or `pwd` in the folder where it was downloaded.)

2. SSH into the Bastion host, then from there SSH into the private EC2 instance using the same key.

ssh -i key.pem ubuntu@public_ip_of_ec2

3. Deploying a test application
- Connected to a private EC2 instance (via Bastion).
- Created a simple `index.html` file (sample template from W3Schools).
- Started a quick test web server:
  ```
  python3 -m http.server 8000
  ```

Referance screenshots:

<img width="1920" height="1080" alt="Screenshot (352)" src="https://github.com/user-attachments/assets/efde7174-d634-4ffa-b6c2-ae7425e0f8bc" />

<img width="1920" height="1080" alt="Screenshot (354)" src="https://github.com/user-attachments/assets/38368cd0-567e-4d6a-8d57-4fba7bd027d7" />

<img width="1920" height="1080" alt="Screenshot (355)" src="https://github.com/user-attachments/assets/765b19db-03c2-43d2-9ed5-8862da067f76" />

<img width="1920" height="1080" alt="Screenshot (356)" src="https://github.com/user-attachments/assets/3554cf1b-dc43-4c4a-8f60-3514cfc76b76" />

<img width="1920" height="1080" alt="Screenshot (357)" src="https://github.com/user-attachments/assets/e407346a-107a-49fa-90a7-0709090e10fb" />

<img width="1920" height="1080" alt="Screenshot (358)" src="https://github.com/user-attachments/assets/2f2b646a-f183-4c03-9739-51443597a402" />

<img width="1920" height="1080" alt="Screenshot (359)" src="https://github.com/user-attachments/assets/cfd120de-ea27-4305-b3a1-8b136631bf67" />

<img width="1920" height="1080" alt="Screenshot (360)" src="https://github.com/user-attachments/assets/82372005-94cd-4c5e-8d83-0ffc2956df2c" />

---

## 6. Load Balancer & Target Group Creation

**Target Group:**
- Target type: Instances
- Name: `aws-prod-example-tg`
- Protocol: HTTP, Port: `8000`
- VPC: Selected the created VPC
- Health check: HTTP
- Registered the private EC2 instances as targets
- Created the Target Group

Referance screenshots:

<img width="1920" height="1080" alt="Screenshot (361)" src="https://github.com/user-attachments/assets/5565dfbf-f85e-4ea8-97cc-f70e2776ec32" />

<img width="1920" height="1080" alt="Screenshot (362)" src="https://github.com/user-attachments/assets/7178fb52-b736-43a2-94a6-739d4448eb55" />

<img width="1920" height="1080" alt="Screenshot (363)" src="https://github.com/user-attachments/assets/07788d9b-cbc2-41de-87d7-4c9ae162027d" />

<img width="1920" height="1080" alt="Screenshot (364)" src="https://github.com/user-attachments/assets/7f3796e0-8048-422b-82ec-384e4639c480" />

<img width="1920" height="1080" alt="Screenshot (365)" src="https://github.com/user-attachments/assets/2cb09784-fa12-40e0-91c6-ad4a12667ceb" />


**Application Load Balancer:**
- Name: `aws-prod-example-alb`
- Scheme: Internet-facing
- VPC: Selected the created VPC, across 2 AZs, using the 2 public subnets
- Security Group: Created a new SG allowing **HTTP (port 80)** from the internet
- Listener: HTTP:80 → Forward to the Target Group created above

Finally, updated the instance security group inbound rule for port `8000` from allow all traffic to the ALB's security group, replacing the temporary "Anywhere" rule.

Referance screenshots:

<img width="1920" height="1080" alt="Screenshot (366)" src="https://github.com/user-attachments/assets/9ea22c3a-1a13-4471-8dae-0245113c0d71" />

<img width="1920" height="1080" alt="Screenshot (367)" src="https://github.com/user-attachments/assets/e22b28fa-11d0-404e-833e-57f00c355846" />

<img width="1920" height="1080" alt="Screenshot (368)" src="https://github.com/user-attachments/assets/1b89048d-8a0f-48d8-bbf7-49e92f3c2783" />

<img width="1920" height="1080" alt="Screenshot (370)" src="https://github.com/user-attachments/assets/78fdd943-1a25-4522-9696-37bf91b03362" />

<img width="1920" height="1080" alt="Screenshot (371)" src="https://github.com/user-attachments/assets/d6a1c3a1-0ed9-414a-8686-b9613bd6051b" />

<img width="1920" height="1080" alt="Screenshot (373)" src="https://github.com/user-attachments/assets/f0ada96f-9519-4491-8f1b-7c13c3540b59" />

<img width="1920" height="1080" alt="Screenshot (374)" src="https://github.com/user-attachments/assets/ca5d3024-a25e-4060-bc3a-d83e42651ede" />

<img width="1920" height="1080" alt="Screenshot (375)" src="https://github.com/user-attachments/assets/76f43642-e002-41eb-bb27-e960bc34a528" />

<img width="1920" height="1080" alt="Screenshot (377)" src="https://github.com/user-attachments/assets/2a0c6dbd-c631-4f6c-a9ea-652f45446672" />

<img width="1920" height="1080" alt="Screenshot (384)" src="https://github.com/user-attachments/assets/4cbb1c48-6289-439f-8387-f97703277321" />

<img width="1920" height="1080" alt="Screenshot (387)" src="https://github.com/user-attachments/assets/179d4bca-90f3-41a4-bc7e-640a9d4f0653" />

---

## Traffic flows:

-  **Internet → IGW → ALB (public subnets) → Target Group → EC2 instances (private subnets)**
- Auto Scaling Group maintains 2–4 healthy instances across both AZs based on load.
- Only the Bastion host is SSH-reachable from the internet (and only from a whitelisted IP); app instances are only reachable via the Bastion or the ALB.
