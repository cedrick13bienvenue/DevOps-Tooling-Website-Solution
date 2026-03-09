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
