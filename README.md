# DevOps Tooling Website Solution on AWS

## Project Overview

This project implements a **DevOps Tooling Website** — a centralized web application that gives a DevOps team easy access to commonly used tools (Jenkins, Kubernetes, Artifactory, Rancher, Grafana, etc.). The solution uses a **3-Tier Architecture** on AWS consisting of:

- **3 stateless Web Servers** (RHEL 8) serving the PHP application
- **1 NFS Server** (RHEL 8) acting as shared file storage for all Web Servers
- **1 Database Server** (Ubuntu 24.04) running MySQL

All three Web Servers mount the same directory from the NFS server and connect to the same MySQL database — meaning they are **stateless**: any Web Server can be added or removed at any time without losing data.

**Technologies Used:**

| Component | Details |
|---|---|
| Infrastructure | AWS EC2 |
| Web Server OS | Red Hat Enterprise Linux 8 |
| Database OS | Ubuntu Server 24.04 LTS |
| Storage Server OS | Red Hat Enterprise Linux 8 + NFS |
| Web Server Software | Apache (httpd) + PHP 7.4 |
| Database | MySQL |
| Programming Language | PHP |
| Code Repository | GitHub |

**Architecture:**

```
           Browser (Client)
                |
        ┌───────┼───────┐
        ↓       ↓       ↓
      WS-1    WS-2    WS-3       ← 3x Web Servers (RHEL 8, Apache+PHP)
        |       |       |
        └───────┼───────┘
                ↓
           DB Server             ← MySQL (Ubuntu 24.04)

        ↑       ↑       ↑
      WS-1    WS-2    WS-3
        └───────┼───────┘
                ↓
           NFS Server            ← Shared /var/www (RHEL 8)
```

---

## Phase 1: Provision and Configure the NFS Server

### 1.1 Launch the NFS Server EC2 Instance

**1.** Sign in to the **AWS Management Console** and navigate to **EC2** → **Instances** → click **Launch instances**.

**2.** In the **Name** field, enter `Project7-NFS`.

**3.** Under **Application and OS Images (Amazon Machine Image)**, click **Browse more AMIs** → search for `Red Hat Enterprise Linux 8` → select the **RHEL 8** AMI (64-bit x86).

> **Expected Output**: The AMI section shows "Red Hat Enterprise Linux 8" selected.
> ![AWS console — NFS instance name and RHEL 8 AMI selected](screenshoots/1.png)

---

**4.** Under **Instance type**, select `t2.micro` (or `t3.micro` if t2.micro is unavailable in your region).

**5.** Under **Key pair (login)**, select your existing `.pem` key pair or create a new one. This is required to SSH into the instance.

**6.** Under **Network settings**, click **Edit**:
   - **VPC**: Leave as default
   - **Subnet**: Leave as default (note which subnet you are in — you will need its CIDR later)
   - **Auto-assign public IP**: Enable
   - **Firewall (Security groups)**: Create a new security group named `Project7-NFS-SG`
   - Add an **Inbound Rule**: Type = `SSH`, Protocol = `TCP`, Port = `22`, Source = `My IP`

**7.** Under **Configure storage**, you see the default 8 GiB root volume. Click **Add new volume** three times to add three additional EBS volumes:
   - Volume 1: `10 GiB`, `gp3`
   - Volume 2: `10 GiB`, `gp3`
   - Volume 3: `10 GiB`, `gp3`

> **Expected Output**: The storage section shows 4 volumes total — 1 root + 3 additional EBS volumes.
> ![AWS console — NFS instance type, key pair, security group, and 4 EBS volumes configured](screenshoots/2.png)

---

**8.** Click **Launch instance**. Wait until the instance **State** changes to `Running` and **Status checks** shows `2/2 checks passed`.

**9.** Click on the instance to view its details. Note:
   - **Public IPv4 address** (for SSH access)
   - **Private IPv4 address** (Web Servers will use this to mount NFS)

> **Expected Output**: NFS instance is `Running` with 2/2 status checks. Public and private IPs are visible.
> ![AWS console — NFS instance running, instance details showing public and private IP](screenshoots/3.png)

---

### 1.2 SSH into the NFS Server

Open your terminal and connect using your `.pem` key:

```bash
ssh -i "your-key.pem" ec2-user@<NFS-Server-Public-IP>
```

> **Note**: On RHEL EC2 instances, the default username is `ec2-user`.

Once connected, update the system:

```bash
sudo yum -y update
```

> **Expected Output**: Terminal shows successful SSH login banner; `yum update` completes with `Complete!`.
> ![Terminal — SSH into NFS server and yum update complete](screenshoots/4.png)

---

### 1.3 Verify Attached Disks

Run `lsblk` to confirm the three additional EBS volumes are attached and visible to the OS:

```bash
lsblk
```

You should see your root disk (e.g., `xvda`) plus three additional disks (`xvdb`, `xvdc`, `xvdd`) — or `nvme` names depending on the instance type. The extra disks will have no partitions yet.

```bash
sudo df -h
```

> **Expected Output**: `lsblk` shows 4 disks. The 3 extra disks have no mount points and no partitions.
> ![Terminal — lsblk output showing xvdb, xvdc, xvdd with no partitions](screenshoots/5.png)
