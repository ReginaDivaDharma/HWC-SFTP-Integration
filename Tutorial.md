# 🔐 SFTP Integration with Huawei Cloud CDM

A step-by-step guide to securely transferring files from a source VM to Huawei Cloud OBS using **SFTP** and **Cloud Data Migration (CDM)**.

---

## 📖 What is SFTP?

Think of it this way — if you had a confidential document that needed to reach someone on a completely different machine, you wouldn't just send it in the open. You'd want it encrypted and transmitted securely. That's the whole point of SFTP.

**SFTP (Secure File Transfer Protocol)** handles exactly that: safe, encrypted file transfers between two machines over a network. In this guide, we'll walk through how to move data from a Huawei Cloud ECS instance into an OBS bucket in a separate Huawei Cloud environment, with CDM acting as the middleman.

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
- A source VM (ECS) reachable via the public internet
- A destination environment ready (e.g., an OBS bucket)
- Basic comfort with Linux terminal commands

---

## 🚀 Step-by-Step Guide

### Step 1 — Prepare Your Data File

Get the file or folder you want to transfer ready on your local machine. CDM's SFTP supports common formats like **CSV, ZIP**, and others.

> 💡 **Tip:** Zipping your files before transfer is highly recommended — smaller file sizes mean faster transfers.

---

### Step 2 — Open Port 22 on Your Source ECS

CDM reaches your source VM over SFTP, which runs on **port 22** — the same port as SSH. Before anything else, make sure this port isn't being blocked.

#### 2a. Add an Inbound Rule to the Security Group

Navigate to: **Huawei Cloud Console → ECS → Security Groups → Inbound Rules**

| Direction | Protocol | Port | Source |
|-----------|----------|------|--------|
| Inbound | TCP | 22 | 0.0.0.0/0 (or restrict to your CDM cluster IP) |

#### 2b. Check the Firewall on the ECS Itself

```bash
# Check if the firewall is running
systemctl status firewalld

# If it's active, allow port 22 through
firewall-cmd --permanent --add-port=22/tcp
firewall-cmd --reload
```

#### 2c. Verify SSH Access Works

```bash
ssh root@<your-ECS-public-IP>
```

> ✅ If you can SSH in successfully, SFTP is good to go — same port, same door.

> ❓ **Does the destination environment also need port 22 open?**  
> **No.** CDM only needs port 22 to reach into your *source* and pull the file. It then pushes the data to the destination using its own protocol (e.g., HTTPS for OBS). The destination doesn't need port 22 at all.

---

### Step 3 — Create an SFTP User on the Source ECS

Once you're SSH'd into your ECS, run the following to set up a dedicated SFTP user and an uploads directory:

```bash
# Create the SFTP user
useradd -m sftpuser
passwd sftpuser        # You'll be prompted to set a password

# Create the uploads directory
mkdir -p /home/sftpuser/uploads

# Apply the correct ownership and permissions
chown root:root /home/sftpuser
chmod 755 /home/sftpuser
chown sftpuser:sftpuser /home/sftpuser/uploads
```

---

### Step 4 — Transfer Your File to ECS

From your **local machine** (not from within ECS), push your file over using `scp`:

**Windows:**
```bash
scp C:/Users/yourname/your-file root@<your-ECS-public-IP>:/root/Documents/Folder_SFTP/
```

**Linux / Mac:**
```bash
scp /local/path/your-file root@<your-ECS-public-IP>:/root/Documents/Folder_SFTP/
```

---

### Step 5 — Move the File into the SFTP Uploads Folder

Back inside your **ECS terminal**, copy the file into the uploads directory:

```bash
cp /root/Documents/Folder_SFTP/your-file /home/sftpuser/uploads/
```

Confirm it landed correctly:

```bash
ls -lh /home/sftpuser/uploads/
```

> ⚠️ **This step is non-negotiable.** CDM will only be able to see files that are inside `/home/sftpuser/uploads/`. If the file isn't here, the job will fail.

---

### Step 6 — Create Links in CDM

Links are how CDM "recognizes" your source and destination before any data movement happens. Think of them as saved connection profiles — you set them up once, and CDM knows where to reach in and where to push out.

In the **Huawei Cloud CDM Console**:

1. Go to **Cluster Management → Job Management**
2. Create your **Links** before setting up any jobs

#### 6a. Source Link (SFTP)

Click **Create Link**, select **SFTP**, and fill in the connection details:

![SFTP Link Setup](https://github.com/user-attachments/assets/27efd3b9-e1b5-48ae-976c-8190007d6a6f)

> 💡 Always click **Test Connection** before saving. Only save the link once the test comes back successful.

#### 6b. Destination Link (OBS)

Before creating the OBS link, make sure you have:
- An OBS bucket already created
- Your **AK/SK** credentials ready → [How to get your AK/SK](https://support.huaweicloud.com/intl/en-us/iam_faq/iam_01_0618.html)

![OBS Link Setup](https://github.com/user-attachments/assets/dc8b3fd2-b625-4ee3-a32f-ad9434ca857c)

Fill in your bucket name, AK/SK, and give the link any name you like. Test the connection before saving.

> ⚠️ **If the OBS connection test fails**, try removing `:443` from the end of the endpoint URL and test again. If it still doesn't work, check the correct endpoint for your region here: [OBS Endpoint Reference](https://console-intl.huaweicloud.com/apiexplorer/#/endpoint/OBS)

---

### Step 7 — Create and Run the Migration Job

With both links in place, it's time to kick off the actual transfer.

1. Go to **Job Management → Create Job**
2. Select your **SFTP link** as the source and your **OBS link** as the destination
3. Set the source file path to `/home/sftpuser/uploads/your-file`

![CDM Job Setup](https://github.com/user-attachments/assets/bfad994b-b74e-444e-ae15-1467debf571a)

4. Click through **Next → Save and Run**

CDM will take it from there. Once the job completes, head over to your OBS bucket to confirm the file arrived. That's it — you're done! 🎉

---

## 🔁 Quick Reference: Full Flow

```
1. Prepare and compress your file locally
        ↓
2. Open port 22 on the source ECS security group
        ↓
3. Create sftpuser + uploads folder on ECS
        ↓
4. scp file from local → ECS /root/Documents/Folder_SFTP/
        ↓
5. cp file into /home/sftpuser/uploads/   ← CDM reads from here
        ↓
6. Create SFTP + OBS links in CDM
        ↓
7. Create and run the CDM migration job
        ↓
8. File arrives at your destination (OBS / DLI)
```

---

## 📌 FYI yeah

- SFTP and SSH share port 22 — if one works, the other will too
- The file **must** be in `/home/sftpuser/uploads/` before running the CDM job
- Zipping files beforehand will noticeably speed up large transfers
- For better security, restrict the Security Group inbound rule to your CDM cluster's IP rather than `0.0.0.0/0`

---
