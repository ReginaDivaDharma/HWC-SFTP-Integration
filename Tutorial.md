# 🔐 SFTP Integration with Huawei Cloud CDM

Transfer files securely from a source VM to Huawei Cloud OBS using **SFTP** and **Cloud Data Migration (CDM)**.

---

## 📖 What is SFTP?

Imagine you have a secret document you need to send safely to a friend on a different computer. You'd want to encrypt it first, then send it securely — that's exactly what SFTP does.

**SFTP (Secure File Transfer Protocol)** is a protocol for transferring files safely between two computers over a network. In this tutorial, we'll use it to move data from a source VM (hosted on Huawei Cloud ECS) into an OBS bucket in another Huawei Cloud environment, using CDM as the bridge.

---

## 🗺️ Architecture Overview

```
Local Machine
     │
     │  scp (upload file)
     ▼
Source ECS VM  ──────────────────────────────────────────┐
/home/sftpuser/uploads/                                  │
                                                         │  SFTP (port 22)
                                              CDM Cluster◄┘
                                                    │
                                                    │  Internal / HTTPS
                                                    ▼
                                          Destination (OBS / DLI)
```

---

## ✅ Prerequisites

- A Huawei Cloud account with access to **ECS** and **CDM**
- A source VM (ECS) that is accessible via the public internet
- A destination environment (e.g., OBS bucket)
- Basic familiarity with Linux terminal commands

---

## 🚀 Step-by-Step Guide

### Step 1 — Prepare Your Data File

Prepare the file or folder you want to transfer. Huawei Cloud CDM's SFTP supports formats like **CSV, ZIP**, and more.

> 💡 **Tip:** Compress your files into a `.zip` before transferring to reduce size and speed up the process.

---

### Step 2 — Open Port 22 on Your Source ECS

CDM connects to your source VM via SFTP, which uses **port 22** (same as SSH). You need to make sure this port is open.

#### 2a. Add an Inbound Rule in the Security Group

Go to: **Huawei Cloud Console → ECS → Security Groups → Inbound Rules**

| Direction | Protocol | Port | Source |
|-----------|----------|------|--------|
| Inbound | TCP | 22 | 0.0.0.0/0 (or restrict to CDM IP) |

#### 2b. Check the Linux Firewall on ECS

```bash
# Check firewall status
systemctl status firewalld

# If active, allow port 22
firewall-cmd --permanent --add-port=22/tcp
firewall-cmd --reload
```

#### 2c. Test SSH Access

```bash
ssh root@<your-ECS-public-IP>
```

> ✅ If SSH works, SFTP will work — they use the same port!

> ❓ **Does the destination also need port 22 open?**  
> **No.** CDM connects to your *source* via port 22 (SFTP) to pull the file, then pushes to the destination using its own internal protocol (e.g., HTTPS for OBS). Only the source needs port 22 open.

---

### Step 3 — Create an SFTP User on Your Source ECS

SSH into your ECS and run the following:

```bash
# Create SFTP user
useradd -m sftpuser
passwd sftpuser        # Set a password when prompted

# Create the uploads folder
mkdir -p /home/sftpuser/uploads

# Set correct ownership and permissions
chown root:root /home/sftpuser
chmod 755 /home/sftpuser
chown sftpuser:sftpuser /home/sftpuser/uploads
```

---

### Step 4 — Upload Your File to ECS

Run this from your **local machine** (not ECS):

**Windows:**
```bash
scp C:/Users/yourname/stock_market_data.csv root@<your-ECS-public-IP>:/root/Documents/Folder_SFTP/
```

**Linux / Mac:**
```bash
scp /local/path/stock_market_data.csv root@<your-ECS-public-IP>:/root/Documents/Folder_SFTP/
```

---

### Step 5 — Copy the File into the SFTP Uploads Folder

Back on your **ECS terminal**, run:

```bash
cp /root/Documents/Folder_SFTP/stock_market_data.csv /home/sftpuser/uploads/
```

Then verify it's there:

```bash
ls -lh /home/sftpuser/uploads/
```

> ⚠️ **This is the critical step.** CDM can only see files inside `/home/sftpuser/uploads/`. The file must be here before running the CDM job.

---

### Step 6 — Set Up CDM Links

In the **Huawei Cloud CDM Console**:

1. Go to **Cluster Management → Job Management**
2. Before creating a job, create your **Links** first

> 💡 **What is a Link?**  
> A Link tells CDM how to connect to your source and destination. Think of it as registering your environments so CDM can "recognize" them before any data transfer begins.

3. Click **Create Link** and select **SFTP** as the source type
4. Fill in the SFTP link details:

| Field | Value |
|-------|-------|
| Host | Your ECS public IP |
| Port | `22` |
| Username | `sftpuser` |
| Password | Your sftpuser password |

5. Create a second link for your **destination** (e.g., OBS)

---

### Step 7 — Create and Run the CDM Job

1. Go to **Job Management → Create Job**
2. Set **Source** to your SFTP link
3. Set the **File Path** to `/home/sftpuser/uploads/stock_market_data.csv`
4. Set **Destination** to your OBS (or DLI) link
5. Click **Save and Run** ✅

---

## 🔁 Quick Reference: Full Flow

```
1. Prepare file locally
        ↓
2. Open port 22 on source ECS security group
        ↓
3. Create sftpuser + uploads folder on ECS
        ↓
4. scp file from local → ECS /root/Documents/Folder_SFTP/
        ↓
5. cp file → /home/sftpuser/uploads/   ← CDM reads from here
        ↓
6. Create SFTP + Destination links in CDM
        ↓
7. Create & run CDM job
        ↓
8. File arrives at destination (OBS / DLI)
```

---

## 📌 Notes

- SFTP and SSH share **port 22** — securing one secures the other
- Always verify the file exists in `/home/sftpuser/uploads/` before running the CDM job
- For large datasets, zip your files first to improve transfer speed
- Restrict the Security Group source IP to your CDM cluster IP for better security

---

## 📄 License

MIT License — feel free to use and adapt this tutorial.
