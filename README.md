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

---

**8.** Click **Launch instance**. Wait until the instance **State** changes to `Running` and **Status checks** shows `2/2 checks passed`.

**9.** Click on the instance to view its details. Note:
   - **Public IPv4 address** (for SSH access)
   - **Private IPv4 address** (Web Servers will use this to mount NFS)

> **Expected Output**: NFS instance is `Running` with 2/2 status checks. Public and private IPs are visible.

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
> ![Terminal — SSH into NFS server, yum update complete, and lsblk showing nvme devices](screenshoots/1.png)

---

### 1.3 Verify Attached Disks

Run `lsblk` to confirm the three additional EBS volumes are attached and visible to the OS:

```bash
lsblk
```

> **Device naming note**: On modern instance types (t3, t2 in some regions), EBS volumes appear as **NVMe devices** (`nvme0n1`, `nvme1n1`, `nvme2n1`, `nvme3n1`) rather than the traditional `xvda/xvdb/xvdc/xvdd` names. Your root volume is always the disk with partitions; the extra disks will have no partitions yet.

Expected output example (NVMe naming):
```
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
nvme1n1     259:0    0   10G  0 disk
nvme2n1     259:1    0   10G  0 disk
nvme3n1     259:2    0   10G  0 disk
nvme0n1     259:3    0   10G  0 disk
├─nvme0n1p1 259:4    0    1M  0 part
├─nvme0n1p2 259:5    0  200M  0 part /boot/efi
└─nvme0n1p3 259:6    0  9.8G  0 part /
```

> **If you only see 1 or 2 extra disks**: Some or all of the three additional EBS volumes were not successfully attached at launch. Go to **AWS Console → EC2 → Volumes**, find the volumes in **"Available"** state in the **same Availability Zone** as your instance, right-click each → **Attach Volume** → select your `Project7-NFS` instance → **Attach**. Then re-run `lsblk` to confirm all three appear.

```bash
sudo df -h
```

> **Expected Output**: `lsblk` shows 4 disks total. The 3 extra disks have no mount points and no partitions.
> ![Terminal — lsblk output showing nvme devices with no partitions](screenshoots/5.png)

---

### 1.4 Configure LVM on the NFS Server

> **Important**: Unlike the previous WordPress project where volumes were formatted as `ext4`, this project uses **`xfs`** filesystem.

**Step 1 — Install LVM tools (if not present):**

```bash
sudo yum install lvm2 -y
```

**Step 2 — Create Physical Volumes on all three disks:**

> **Use the actual device names from your `lsblk` output.** On modern instance types the disks are NVMe devices. Replace `nvme1n1`, `nvme2n1`, `nvme3n1` with whatever your three extra disks are called (everything that is NOT the root disk with partitions).

```bash
sudo pvcreate /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1
```

If you're on an older instance type that uses traditional naming, use:
```bash
sudo pvcreate /dev/xvdb /dev/xvdc /dev/xvdd
```

Expected output:
```
  Physical volume "/dev/nvme1n1" successfully created.
  Physical volume "/dev/nvme2n1" successfully created.
  Physical volume "/dev/nvme3n1" successfully created.
```

**Step 3 — Create a Volume Group named `webdata-vg`:**

```bash
# NVMe naming:
sudo vgcreate webdata-vg /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1

# Traditional naming:
# sudo vgcreate webdata-vg /dev/xvdb /dev/xvdc /dev/xvdd

sudo vgs
```

**Step 4 — Create 3 Logical Volumes:**

```bash
sudo lvcreate -n lv-apps -L 9G webdata-vg   # For web server files
sudo lvcreate -n lv-logs -L 9G webdata-vg   # For web server logs
sudo lvcreate -n lv-opt  -L 9G webdata-vg   # For Jenkins (Phase 8)
sudo lvs
```

> **Expected Output**: `sudo lvs` shows `lv-apps`, `lv-logs`, and `lv-opt` each with ~9 GiB in `webdata-vg`.
> ![Terminal — pvcreate, vgcreate, lvcreate, and lvs output](screenshoots/6.png)

---

**Step 5 — Format all three Logical Volumes as `xfs`:**

```bash
sudo mkfs -t xfs /dev/webdata-vg/lv-apps
sudo mkfs -t xfs /dev/webdata-vg/lv-logs
sudo mkfs -t xfs /dev/webdata-vg/lv-opt
```

**Step 6 — Create mount point directories under `/mnt`:**

```bash
sudo mkdir -p /mnt/apps /mnt/logs /mnt/opt
```

**Step 7 — Mount the Logical Volumes:**

```bash
sudo mount /dev/webdata-vg/lv-apps /mnt/apps
sudo mount /dev/webdata-vg/lv-logs /mnt/logs
sudo mount /dev/webdata-vg/lv-opt  /mnt/opt
df -h
```

> **Expected Output**: `df -h` shows `/mnt/apps`, `/mnt/logs`, and `/mnt/opt` each mounted on their respective `xfs` logical volumes.
> ![Terminal — mkfs.xfs on all 3 LVs, mount commands, and df -h showing 3 mount points](screenshoots/7.png)

---

**Step 8 — Make mounts persistent across reboots via `/etc/fstab`:**

```bash
sudo blkid | grep webdata
```

Open `/etc/fstab` and add the three UUID entries (replace UUID values with your actual output from `blkid`):

```bash
sudo vi /etc/fstab
```

```
UUID=<lv-apps-uuid>  /mnt/apps  xfs  defaults  0 0
UUID=<lv-logs-uuid>  /mnt/logs  xfs  defaults  0 0
UUID=<lv-opt-uuid>   /mnt/opt   xfs  defaults  0 0
```

Save and exit (`:wq`), then verify:

```bash
sudo mount -a
sudo systemctl daemon-reload
df -h
```

> **Expected Output**: `mount -a` returns with no errors; `df -h` still shows all three volumes mounted.
> ![Terminal — blkid output, /etc/fstab with UUID entries, and mount -a success](screenshoots/8.png)

---

### 1.5 Install and Start the NFS Server

```bash
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```

> **Expected Output**: `nfs-server.service` shows status `active (running)` and is enabled to start on boot.
> ![Terminal — nfs-utils install complete; nfs-server.service active and enabled](screenshoots/9.png)

---

### 1.6 Set Ownership and Permissions on NFS Directories

The Web Servers need read, write, and execute permissions on the NFS shares:

```bash
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service
```

Confirm ownership:

```bash
ls -la /mnt/
```

> **Expected Output**: All three directories under `/mnt` are owned by `nobody:nobody` with `rwxrwxrwx` permissions.
> ![Terminal — chown and chmod output; ls -la /mnt showing nobody ownership and 777 perms](screenshoots/10.png)

---

### 1.7 Configure NFS Exports

**Step 1 — Find your subnet CIDR:**

In the AWS EC2 console, click on your NFS instance → **Networking** tab → click the **Subnet ID** link → look for the **IPv4 CIDR** column. It will look something like `172.31.32.0/20`.

**Step 2 — Edit the NFS exports file:**

```bash
sudo vi /etc/exports
```

Add the following (replace `<Subnet-CIDR>` with your actual subnet CIDR, e.g. `172.31.32.0/20`):

```
/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt  <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
```

Save and exit (`:wq`).

**What these options mean:**
- `rw` — clients can read and write
- `sync` — writes are committed to disk before responding
- `no_all_squash` — preserve client user IDs
- `no_root_squash` — allow root on the client to act as root on the NFS share

**Step 3 — Apply the export configuration:**

```bash
sudo exportfs -arv
```

Expected output:
```
exporting 172.31.32.0/20:/mnt/opt
exporting 172.31.32.0/20:/mnt/logs
exporting 172.31.32.0/20:/mnt/apps
```

> **Expected Output**: `exportfs -arv` lists all three exported paths with your subnet CIDR.
> ![Terminal — /etc/exports content and exportfs -arv output showing all 3 exports](screenshoots/11.png)

---

### 1.8 Open NFS Ports in the Security Group

**Step 1 — Check which port NFS is running on:**

```bash
rpcinfo -p | grep nfs
```

Expected output (NFS listens on **port 2049**):
```
100003   3   tcp  2049  nfs
100003   4   tcp  2049  nfs
100227   3   tcp  2049  nfs_acl
```

**Step 2 — Add Inbound Rules in AWS:**

Go to **EC2** → **Security Groups** → find `Project7-NFS-SG` → click **Edit inbound rules** → **Add rule** for each of the following, setting **Source** to your **subnet CIDR**:

| Type | Protocol | Port Range | Source |
|---|---|---|---|
| NFS | TCP | 2049 | `<Subnet-CIDR>` |
| Custom TCP | TCP | 111 | `<Subnet-CIDR>` |
| Custom UDP | UDP | 111 | `<Subnet-CIDR>` |
| Custom UDP | UDP | 2049 | `<Subnet-CIDR>` |

Click **Save rules**.

> **Expected Output**: `rpcinfo` shows port 2049; Security Group inbound rules show all four NFS-related rules.
> ![Terminal — rpcinfo output; AWS console — Security Group with NFS inbound rules](screenshoots/12.png)

---

## Phase 2: Provision and Configure the Database Server

### 2.1 Launch the Database Server EC2 Instance

**1.** In the AWS EC2 console, click **Launch instances**.

**2.** Name it `Project7-DB`.

**3.** Under **AMI**, search for and select **Ubuntu Server 24.04 LTS** (64-bit x86).

**4.** Select **Instance type**: `t2.micro`.

**5.** Select your existing **Key pair**.

**6.** Under **Network settings** → **Edit**:
   - Keep the same VPC and Subnet as the NFS server (same subnet = same CIDR)
   - **Auto-assign public IP**: Enable
   - Create a new Security Group named `Project7-DB-SG`
   - Add inbound rules:
     - `SSH` → Port `22` → Source: `My IP`
     - `MySQL/Aurora` → Port `3306` → Source: `<Subnet-CIDR>` (so Web Servers on the same subnet can connect)

**7.** Keep default storage (8 GiB root). Click **Launch instance**.

> **Expected Output**: DB server instance is `Running` with 2/2 status checks; Security Group shows port 3306 open to subnet CIDR.
> ![AWS console — DB server instance running; Security Group with port 3306 open to subnet CIDR](screenshoots/13.png)

---

### 2.2 SSH into the Database Server

```bash
ssh -i "your-key.pem" ubuntu@<DB-Server-Public-IP>
```

> **Note**: On Ubuntu EC2 instances, the default username is `ubuntu`.

Update the system:

```bash
sudo apt update && sudo apt upgrade -y
```

---

### 2.3 Install MySQL Server

```bash
sudo apt install mysql-server -y
sudo systemctl enable --now mysql
sudo systemctl status mysql
```

> **Expected Output**: `mysql.service` is `active (running)` and enabled.
> ![Terminal — SSH into DB server; apt install mysql-server complete; mysql.service active](screenshoots/14.png)

---

### 2.4 Configure MySQL to Accept Remote Connections

By default, MySQL only listens on `127.0.0.1`. Change the bind address so Web Servers can connect:

```bash
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
```

Find the line:

```
bind-address = 127.0.0.1
```

Change it to:

```
bind-address = 0.0.0.0
```

Save and exit, then restart MySQL:

```bash
sudo systemctl restart mysql
```

---

### 2.5 Create the Tooling Database and User

```bash
sudo mysql
```

Inside the MySQL shell, run the following SQL commands:

```sql
-- Create the application database
CREATE DATABASE tooling;

-- Create a dedicated user restricted to the Web Servers' subnet
CREATE USER 'webaccess'@'172.31.%' IDENTIFIED BY 'password';

-- Grant full privileges on the tooling database only
GRANT ALL PRIVILEGES ON tooling.* TO 'webaccess'@'172.31.%';

-- Apply privilege changes immediately
FLUSH PRIVILEGES;

-- Verify the database was created
SHOW DATABASES;

EXIT;
```

> **Note**: `'webaccess'@'172.31.%'` allows any host in the `172.31.x.x` range (your VPC) to connect.

> **Expected Output**: `SHOW DATABASES` lists `tooling`; each SQL statement returns `Query OK`.
> ![Terminal — MySQL session: CREATE DATABASE, CREATE USER, GRANT, FLUSH, SHOW DATABASES](screenshoots/15.png)

---

## Phase 3: Provision and Configure the Web Servers

Each of the three Web Servers must be configured identically. Launch all three instances first, then configure them one by one.

### 3.1 Launch Three Web Server EC2 Instances

**1.** In the AWS EC2 console, click **Launch instances**.

**2.** Under **Number of instances**, enter `3`.

**3.** Name them `Project7-Web` (AWS will append `-1`, `-2`, `-3`).

**4.** Under **AMI**, select **Red Hat Enterprise Linux 8** (same as the NFS server).

**5.** Select **Instance type**: `t2.micro`.

**6.** Select your existing **Key pair**.

**7.** Under **Network settings** → **Edit**:
   - Same VPC and Subnet as NFS and DB servers
   - **Auto-assign public IP**: Enable
   - Create a new Security Group named `Project7-Web-SG`
   - Add inbound rules:
     - `SSH` → Port `22` → Source: `My IP`
     - `HTTP` → Port `80` → Source: `0.0.0.0/0` (public web traffic)

**8.** Keep default storage (8 GiB root). Click **Launch instances**.

> **Expected Output**: Three Web Server instances are `Running` with 2/2 status checks each.
> ![AWS console — Three Web Server instances running; Security Group with SSH and HTTP open](screenshoots/16.png)

---

### 3.2 SSH into Web Server 1

```bash
ssh -i "your-key.pem" ec2-user@<WS-1-Public-IP>
```

---

### 3.3 Install NFS Client

```bash
sudo yum install nfs-utils nfs4-acl-tools -y
```

> **Expected Output**: `nfs-utils` and `nfs4-acl-tools` installed successfully with `Complete!`.
> ![Terminal — SSH into WS-1; yum install nfs-utils and nfs4-acl-tools complete](screenshoots/17.png)

---

### 3.4 Mount the NFS Share to `/var/www`

This mounts the NFS server's `/mnt/apps` directory to the Web Server's `/var/www`, making both point to the same files.

```bash
# Create the target directory
sudo mkdir -p /var/www

# Mount the NFS export (replace <NFS-Server-Private-IP> with actual private IP)
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP>:/mnt/apps /var/www
```

Verify the mount is active:

```bash
df -h
```

You should see a line like:
```
172.31.x.x:/mnt/apps   9.0G  104M  8.9G   2% /var/www
```

**Make the mount persist after reboot:**

```bash
sudo vi /etc/fstab
```

Add this line at the bottom (replace with your NFS server's private IP):

```
<NFS-Server-Private-IP>:/mnt/apps /var/www nfs defaults 0 0
```

Save and exit.

> **Expected Output**: `df -h` shows `/var/www` mounted from the NFS server's private IP; `/etc/fstab` updated.
> ![Terminal — mount command, df -h showing /var/www NFS mount, and /etc/fstab entry](screenshoots/18.png)

---

### 3.5 Install Apache and PHP 7.4 via Remi's Repository

RHEL 8's default repos only carry PHP 7.2. PHP 7.4 is needed from Remi's repo:

```bash
# Install Apache
sudo yum install httpd -y

# Install EPEL repository
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm -y

# Install Remi's repository
sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm -y

# Reset PHP module and enable PHP 7.4 from Remi
sudo dnf module reset php -y
sudo dnf module enable php:remi-7.4 -y

# Install PHP and required extensions for the tooling app
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd -y

# Start and enable PHP-FPM (FastCGI Process Manager)
sudo systemctl start php-fpm
sudo systemctl enable php-fpm

# Allow Apache to execute memory-mapped files (required for PHP-FPM)
setsebool -P httpd_execmem 1
```

Start and enable Apache:

```bash
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl status httpd
```

> **Expected Output**: `httpd.service` and `php-fpm.service` both show as `active (running)` and enabled.

**Repeat steps 3.2 – 3.5 for Web Server 2 and Web Server 3** using their respective public IPs.

---

### 3.6 Verify NFS Shared Storage is Working

After configuring all three Web Servers, confirm they all share files through NFS.

On **Web Server 1**, check that Apache's default directory is visible:

```bash
ls /var/www
```

On the **NFS Server**, the same files should appear in `/mnt/apps`:

```bash
ls /mnt/apps
```

Test cross-server file sharing — create a file from Web Server 1, then check it from Web Server 2:

**On WS-1:**
```bash
touch /var/www/test.txt
ls /var/www/
```

**On WS-2 (new SSH session):**
```bash
ls /var/www/
# test.txt should be visible here too
```

**On NFS Server:**
```bash
ls /mnt/apps/
# test.txt should also be visible here
```

> **Expected Output**: `test.txt` created on WS-1 is immediately visible on WS-2 and on the NFS server's `/mnt/apps` — confirming NFS shared storage is working correctly.

---

### 3.7 Mount Apache Log Directory to NFS

Mount Apache's log directory to the NFS logs export so all Web Server logs are centralized:

```bash
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP>:/mnt/logs /var/log/httpd
```

Persist it in `/etc/fstab`:

```bash
sudo vi /etc/fstab
```

Add:

```
<NFS-Server-Private-IP>:/mnt/logs /var/log/httpd nfs defaults 0 0
```

Verify both NFS mounts are active:

```bash
df -h
```

> **Expected Output**: `df -h` now shows both `/var/www` and `/var/log/httpd` mounted from the NFS server.

---

### 3.8 Fork and Deploy the Tooling Application

**Step 1 — Fork the repository:**

Go to the **StegHub GitHub account** and fork the **tooling** repository to your own GitHub account by clicking the **Fork** button at the top-right of the repository page.

**Step 2 — Install Git and clone your fork on Web Server 1:**

```bash
sudo yum install git -y

git clone https://github.com/<your-github-username>/tooling.git
```

**Step 3 — Deploy the application files to `/var/www/html`:**

```bash
sudo cp -R tooling/html/. /var/www/html/
```

Verify the files are deployed:

```bash
ls /var/www/html/
```

You should see `index.php`, `functions.php`, `login.php`, and other PHP files.

> **Expected Output**: Repository cloned successfully; `ls /var/www/html` shows all tooling PHP application files.

> **Note**: Because `/var/www/html` is mounted from NFS, these files are automatically available on **all three Web Servers** — no need to deploy separately to each one.

---

**Step 4 — Open TCP port 80** in the Web Server's Security Group if not already done.

**Step 5 — Disable SELinux** (if you encounter a 403 Forbidden error):

```bash
sudo setenforce 0
```

To make this permanent so it survives reboots:

```bash
sudo vi /etc/sysconfig/selinux
```

Find the line `SELINUX=enforcing` and change it to:

```
SELINUX=disabled
```

Then restart Apache:

```bash
sudo systemctl restart httpd
```

---

### 3.9 Configure the Database Connection

The tooling application uses `functions.php` to connect to MySQL. Update it with the DB server's private IP and the credentials created in Phase 2.

```bash
sudo vi /var/www/html/functions.php
```

Locate the `db_connect()` function and update the connection string:

```php
$db = mysqli_connect('<DB-Server-Private-IP>', 'webaccess', 'password', 'tooling');
```

Replace `<DB-Server-Private-IP>` with the actual private IP of your DB server (e.g., `172.31.x.x`). Save and exit.

> **Expected Output**: `functions.php` saved with the correct DB server private IP, username `webaccess`, and database `tooling`.

---

### 3.10 Apply the Tooling Database Schema

The repository includes `tooling-db.sql` that creates the required database tables. Apply it from the Web Server:

```bash
# Install MySQL client to connect to the remote DB server
sudo yum install mysql -y

# Apply the schema (enter 'password' when prompted)
mysql -h <DB-Server-Private-IP> -u webaccess -p tooling < tooling/tooling-db.sql
```

Verify the tables were created:

```bash
mysql -h <DB-Server-Private-IP> -u webaccess -p tooling
```

```sql
SHOW TABLES;
EXIT;
```

> **Expected Output**: `tooling-db.sql` applies without errors; `SHOW TABLES` lists the `users` table (and others).

---

### 3.11 Create an Admin User in the Database

On the **DB Server**, log in to MySQL and manually insert an admin user into the `tooling` database:

```bash
sudo mysql
```

```sql
USE tooling;

INSERT INTO users (id, username, password, email, user_type, status)
VALUES (1, 'myuser', '5f4dcc3b5aa765d61d8327deb882cf99', 'user@mail.com', 'admin', '1');

-- Verify the user was inserted
SELECT id, username, email, user_type FROM users;

EXIT;
```

> **Note**: `5f4dcc3b5aa765d61d8327deb882cf99` is the MD5 hash of the word `password`. The tooling app uses MD5 for authentication.

> **Expected Output**: `INSERT` returns `Query OK, 1 row affected`; `SELECT` shows the `myuser` admin record.

---

## Phase 4: Final Verification

### 4.1 Access the Tooling Website

Open the website in your browser using any Web Server's public IP or DNS name:

```
http://<Web-Server-Public-IP>/index.php
```

You should see the **DevOps Tooling website login page**.

> **Expected Output**: The tooling website login page loads — showing the StegHub tooling login form.

---

### 4.2 Log In and Verify the Dashboard

Enter the admin credentials:
- **Username**: `myuser`
- **Password**: `password`

Click **Login**.

> **Expected Output**: Successful authentication — the DevOps Tooling dashboard is displayed, showing the list of DevOps tools (Jenkins, Kubernetes, Artifactory, etc.).

---

### 4.3 Verify Multi-Server Redundancy

To confirm all three Web Servers serve the same content, open the website using each server's public IP:

```
http://<WS-1-Public-IP>/index.php
http://<WS-2-Public-IP>/index.php
http://<WS-3-Public-IP>/index.php
```

All three should display the same login page and the same dashboard after login — because all three read from the same NFS share and the same MySQL database.

> **Expected Output**: All three Web Servers serve identical content, confirming the stateless architecture.

---

## Summary

The complete DevOps Tooling Website solution is now fully operational:

| Component | Instance Name | Role | OS |
|---|---|---|---|
| NFS Server | `Project7-NFS` | Shared file storage via LVM + NFS | RHEL 8 |
| Database Server | `Project7-DB` | MySQL — `tooling` DB, `webaccess` user | Ubuntu 24.04 |
| Web Server 1 | `Project7-Web-1` | Apache + PHP serving tooling app | RHEL 8 |
| Web Server 2 | `Project7-Web-2` | Apache + PHP serving tooling app | RHEL 8 |
| Web Server 3 | `Project7-Web-3` | Apache + PHP serving tooling app | RHEL 8 |

**Key architectural achievements:**
- All three Web Servers share `/var/www` from NFS — one deploy reaches all servers
- Apache logs from all Web Servers are centralized in NFS `/mnt/logs`
- The `/mnt/opt` NFS share is reserved for a future Jenkins installation
- The Web Servers are fully **stateless** — removing or adding one does not affect the application

---
