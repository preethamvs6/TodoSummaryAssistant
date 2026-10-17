# AWS Cloud Architecture & Setup Guide

This document explains the production cloud architecture for hosting the **Todo Summary Assistant** using AWS EC2, AWS RDS (MySQL), Security Groups, and IAM roles.

---

## 1. High-Level Architecture Overview

![AWS Architecture Diagram](architecture-diagram.png)

```
[ Internet Users / Admins ]
        │
        │ HTTP (80), HTTPS (443), SSH (22)
        ▼
 ┌─────────────────────────────────────────────────────────────┐
 │ AWS EC2 Instance (Docker Host) - T3.medium / T3.micro       │
 │ Security Group: sg-ec2                                      │
 │                                                             │
 │  ┌───────────────────────────────────────────────────────┐  │
 │  │ Docker Runtime Engine                                 │  │
 │  │                                                       │  │
 │  │  ├── React Frontend         (Port 80 / 3000)          │  │
 │  │  ├── Spring Boot Backend    (Port 8080)               │──┼───────┐
 │  │  ├── Prometheus             (Port 9090)               │  │       │
 │  │  └── Grafana                (Port 3001)               │  │       │
 │  └───────────────────────────────────────────────────────┘  │       │
 │                                                             │       │
 │  Attached IAM Role: TodoApp-EC2-Role                        │       │
 │  (SSM Management + CloudWatch Logging, No AWS Keys)         │       │
 └─────────────────────────────────────────────────────────────┘       │
                                                                       │ Port 3306
                                                                       │ (Strictly sg-ec2)
                                                                       ▼
                                ┌──────────────────────────────────────────────┐
                                │ AWS RDS MySQL Instance                       │
                                │ Security Group: sg-rds                       │
                                │                                              │
                                │  • Inbound Port 3306 strictly from sg-ec2    │
                                │  • Publicly Accessible: NO                   │
                                │  • Automated Daily Snapshots & Multi-AZ      │
                                └──────────────────────────────────────────────┘
```

---

## 2. AWS Infrastructure Design (EC2 + RDS)

1. **Host Compute (AWS EC2)**:
   * A single Linux instance (`t3.medium` or free-tier `t3.micro`) running Ubuntu 22.04 LTS and the Docker daemon.
   * Runs the complete containerized stack: React Frontend, Spring Boot Backend, Prometheus metrics collector, and Grafana monitoring dashboard.
   * Assigned an Elastic IP (or public IPv4) for DNS and incoming user traffic.

2. **Managed Database (AWS RDS for MySQL)**:
   * Production MySQL runs on a dedicated AWS RDS instance (`db.t3.micro`).
   * Database is **not containerized** in production to leverage AWS managed automated backups, point-in-time recovery (PITR), and Multi-AZ high availability.
   * Public accessibility is set to **No** (private database).

---

## 3. Security Groups (Least Privilege Chaining)

### A. EC2 Security Group (`sg-ec2`)
* **Inbound Rules**:
  * Port 80 (HTTP) from `0.0.0.0/0` (Public web traffic).
  * Port 443 (HTTPS) from `0.0.0.0/0` (Secure web traffic).
  * Port 22 (SSH) restricted to `YOUR_ADMIN_IP/32` (Remote administration).
* **Outbound Rules**:
  * All traffic (`0.0.0.0/0`) allowed to pull Docker images from Docker Hub and reach external APIs (Cohere LLM & Slack Webhook).

### B. RDS Security Group (`sg-rds`)
* **Inbound Rules**:
  * Port 3306 (MySQL) strictly from **Source: `sg-ec2`** (Security Group Chaining).
  * *NO direct internet or public access is permitted.*
* **Outbound Rules**:
  * All outbound traffic allowed.

---

## 4. IAM Role for EC2 (No Hardcoded Credentials)

Instead of saving AWS access keys (`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`) on the EC2 server, an **IAM Instance Profile** is attached directly to the instance:
* **Role Name**: `TodoApp-EC2-Role`
* **Attached Policies**:
  1. `AmazonSSMManagedInstanceCore`: Allows secure AWS Systems Manager Session Manager access without requiring open SSH port 22.
  2. `CloudWatchAgentServerPolicy`: Allows streaming server metrics and container logs directly to AWS CloudWatch.

---

## 5. Secret Management

Sensitive variables (Database credentials, Cohere API key, and Slack webhook URL) are never committed to version control:
* **Production Storage**: Stored in **AWS Systems Manager (SSM) Parameter Store** (SecureString) or injected securely via CI/CD secrets (Jenkins credentials store).
* **EC2 Runtime Injection**: Supplied at container startup as environment variables (`-e DB_URL=... -e DB_PASSWORD=...`).
