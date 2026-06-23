# AWS-VPC-with-public_private-subnet
To create an VPC and secure it

# 🌐 AWS VPC with Auto Scaling & Load Balancer

A production-style AWS infrastructure project demonstrating a **highly available web application** architecture using VPC, private/public subnets, Auto Scaling Groups, and an Application Load Balancer.

---

## 📐 Architecture Overview

```
                        Internet
                           │
                    ┌──────▼──────┐
                    │    IGW      │
                    │  (Internet  │
                    │   Gateway)  │
                    └──────┬──────┘
                           │
              ┌────────────▼────────────┐
              │         VPC             │
              │   CIDR: 10.0.0.0/16     │
              │                         │
              │  ┌─────────────────┐    │
              │  │  Public Subnet  │    │
              │  │  (AZ-1 & AZ-2)  │    │
              │  │                 │    │
              │  │  ┌───────────┐  │    │
              │  │  │    ALB    │  │    │
              │  │  │  (DemoLB) │  │    │
              │  │  └─────┬─────┘  │    │
              │  └────────┼────────┘    │
              │           │             │
              │  ┌────────▼────────┐    │
              │  │ Private Subnets │    │
              │  │                 │    │
              │  │ ┌─────┐ ┌─────┐ │    │
              │  │ │ EC2 │ │ EC2 │ │    │
              │  │ │ AZ1 │ │ AZ2 │ │    │
              │  │ └─────┘ └─────┘ │    │
              │  │  (Auto Scaling) │    │
              │  └─────────────────┘    │
              └─────────────────────────┘
```

---

## 🛠️ What This Project Covers

- ✅ Custom VPC with public and private subnets across 2 Availability Zones
- ✅ Internet Gateway for public subnet internet access
- ✅ NAT Gateway for private subnet outbound access
- ✅ Auto Scaling Group with min/max/desired capacity
- ✅ Application Load Balancer distributing traffic across private instances
- ✅ Security Groups for LB and EC2 instances
- ✅ Python HTTP server serving HTML from private EC2s

---

## 📋 Step-by-Step Setup Guide

### Step 1 — Create a VPC

1. Go to **AWS Console → VPC → Create VPC**
2. Select **"VPC and more"** (creates subnets, route tables automatically)
3. Configure:
   ```
   Name:             demo-vpc
   IPv4 CIDR:        10.0.0.0/16
   Availability Zones: 2
   Public Subnets:   2
   Private Subnets:  2
   NAT Gateway:      1 per AZ (or 1 for cost saving)
   ```
4. Click **Create VPC**

> 💡 This auto-creates: 2 public subnets, 2 private subnets, route tables, and IGW

---

### Step 2 — Create a Security Group for the Load Balancer

1. Go to **EC2 → Security Groups → Create security group**
2. Configure:
   ```
   Name:        demo_SG
   VPC:         demo-vpc
   
   Inbound rules:
     Type: Custom TCP  Port: 8000  Source: 0.0.0.0/0
     Type: SSH         Port: 22    Source: My IP
   
   Outbound rules:
     Type: All traffic  Destination: 0.0.0.0/0
   ```

---

### Step 3 — Create a Security Group for EC2 Instances

1. Create another security group:
   ```
   Name:        demo_EC2_SG
   VPC:         demo-vpc
   
   Inbound rules:
     Type: Custom TCP  Port: 8000  Source: demo_SG (LB security group)
     Type: SSH         Port: 22    Source: My IP
   
   Outbound rules:
     Type: All traffic  Destination: 0.0.0.0/0
   ```

> 🔒 EC2s only accept traffic from the Load Balancer, not directly from the internet

---

### Step 4 — Create a Launch Template

1. Go to **EC2 → Launch Templates → Create launch template**
2. Configure:
   ```
   Name:           Demo
   AMI:            Ubuntu Server 22.04 LTS (same region!)
   Instance type:  t3.micro
   Key pair:       demo.pem (create or use existing)
   Security group: demo_EC2_SG
   ```
3. Under **Advanced → User data**, paste:
   ```bash
   #!/bin/bash
   apt update -y
   apt install python3 -y
   mkdir -p /home/ubuntu/html_file
   echo "<html><body><h1>Hello from $(hostname)</h1><p>Private subnet EC2</p></body></html>" > /home/ubuntu/html_file/index.html
   cd /home/ubuntu/html_file
   python3 -m http.server 8000 &
   ```

---

### Step 5 — Create Target Group

1. Go to **EC2 → Target Groups → Create target group**
2. Configure:
   ```
   Target type:     Instances
   Name:            demoTG
   Protocol:        HTTP
   Port:            8000
   VPC:             demo-vpc
   
   Health checks:
     Protocol:  HTTP
     Path:      /
     Port:      8000
   ```
3. Skip registering targets (ASG will do this automatically)

---

### Step 6 — Create Application Load Balancer

1. Go to **EC2 → Load Balancers → Create load balancer → Application LB**
2. Configure:
   ```
   Name:            DemoLB
   Scheme:          Internet-facing
   IP type:         IPv4
   VPC:             demo-vpc
   Mappings:        Select BOTH public subnets (AZ1 + AZ2)
   Security groups: demo_SG
   ```
3. Add listener:
   ```
   Protocol: HTTP
   Port:     8000
   Action:   Forward to → demoTG
   ```
4. Click **Create load balancer**

> ⚠️ LB must be in PUBLIC subnets. EC2s go in PRIVATE subnets.

---

### Step 7 — Create Auto Scaling Group

1. Go to **EC2 → Auto Scaling Groups → Create**
2. Configure:
   ```
   Name:             demo_auto_scaling
   Launch template:  Demo (from Step 4)
   
   Network:
     VPC:     demo-vpc
     Subnets: BOTH private subnets
   
   Load balancing:
     Attach to existing LB → demoTG
   
   Capacity:
     Desired: 2
     Minimum: 1
     Maximum: 2
   ```
3. Click **Create Auto Scaling group**

> ✅ ASG will automatically launch 2 EC2 instances in the private subnets

---

### Step 8 — SSH into EC2 & Start Server (Manual method)

```bash
# Fix key permissions
chmod 400 ~/Downloads/demo.pem

# SSH into instance (use public IP if bastion, or private IP via VPN)
ssh -i ~/Downloads/demo.pem ubuntu@<EC2-PUBLIC-IP>

# Create and serve HTML
mkdir html_file && cd html_file
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>Demo VPC Project</title></head>
<body>
  <h1>This is a demo project in private subnet of VPC</h1>
  <p>Served via Application Load Balancer</p>
</body>
</html>
EOF

# Start HTTP server
python3 -m http.server 8000
```

---

### Step 9 — Access via Load Balancer

1. Go to **EC2 → Load Balancers → DemoLB**
2. Copy the **DNS name**
3. Open in browser:
   ```
   http://<DNS-NAME>:8000
   ```
   Example:
   ```
   http://demolb-365163093.eu-north-1.elb.amazonaws.com:8000
   ```

> 🎉 You should see your HTML page served from a private subnet EC2!

---

## 🔍 Verify Load Balancing is Working

Check that both instances are healthy:

1. **EC2 → Target Groups → demoTG → Targets tab**
2. Both instances should show `healthy`
3. Refresh the LB URL multiple times — you'll see requests going to both `10.0.x.x` IPs (alternating between AZ1 and AZ2)

From the EC2 terminal, watch the logs:
```
10.0.15.28 - - [23/Jun/2026] "GET / HTTP/1.1" 200 -   ← AZ1 health check
10.0.20.10 - - [23/Jun/2026] "GET / HTTP/1.1" 200 -   ← AZ2 health check
```

---

## 🧹 Cleanup (Important — Avoid Charges)

Delete resources in this order to avoid dependency errors:

```
1. Auto Scaling Group     → EC2 → Auto Scaling Groups → Delete
2. Load Balancer          → EC2 → Load Balancers → Delete
3. Target Group           → EC2 → Target Groups → Delete
4. EC2 Instances          → Terminate (after ASG deleted)
5. NAT Gateway            → VPC → NAT Gateways → Delete  ⚠️ charges hourly
6. Elastic IP             → EC2 → Elastic IPs → Release  ⚠️ charges when idle
7. Launch Template        → EC2 → Launch Templates → Delete
8. Security Groups        → EC2 → Security Groups → Delete
9. VPC                    → VPC → Your VPCs → Delete VPC
```

> ⚠️ **NAT Gateway** (~$0.045/hr) and **Elastic IP** charge even when idle. Delete these first!

---

## 🧠 Key Concepts Learned

| Concept | What it does |
|---|---|
| VPC | Isolated virtual network in AWS |
| Public Subnet | Has route to IGW — accessible from internet |
| Private Subnet | No direct internet access — protected |
| IGW | Allows internet ↔ public subnet traffic |
| NAT Gateway | Allows private subnet → internet (outbound only) |
| NACL | Stateless firewall at subnet boundary |
| Security Group | Stateful firewall at instance level |
| ALB | Distributes HTTP traffic across multiple instances |
| Auto Scaling | Automatically manages EC2 count based on capacity settings |
| Target Group | Group of EC2s that the LB routes traffic to |
| Launch Template | Blueprint for EC2 instances in ASG |

---



## 🛡️ Security Best Practices Applied

- EC2 instances are in **private subnets** (not directly reachable from internet)
- Load Balancer is the **only public entry point**
- EC2 security group only accepts traffic **from the LB security group**
- SSH access restricted to **specific IP** only
- Private key permissions set to **chmod 400**


## 📌 Technologies Used

![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Bash](https://img.shields.io/badge/Bash-4EAA25?style=for-the-badge&logo=gnu-bash&logoColor=white)
