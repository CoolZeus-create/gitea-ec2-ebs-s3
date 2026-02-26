# Persistent Gitea on EC2 (Docker + EBS + S3 Backup)

## Architecture Summary

This project deploys a self-hosted Gitea Git service on a single AWS EC2 instance (Ubuntu 22.04) running Docker. All persistent application data is stored on a separately attached EBS volume mounted at `~/data` on the host, which is bind-mounted into the Gitea container at `/data`. This separates compute (EC2) from state (EBS), ensuring data survives container restarts or replacements. An S3 bucket is used to store compressed `.tar.gz` backups of the Gitea data directory, uploaded via the AWS CLI using an IAM instance role — no credentials are hardcoded anywhere.

---

## Deployment Instructions

### Prerequisites
- AWS account with EC2, EBS, S3, and IAM access
- An SSH key pair (.pem or .ppk for PuTTY)

---

### Step 1: Launch EC2 Instance
1. Go to **EC2 → Launch Instance**
2. Choose **Ubuntu Server 22.04 LTS**, instance type `t2.micro`
3. Configure Security Group:
   - TCP 22 (SSH) → My IP
   - TCP 3000 (Gitea) → Anywhere
4. Launch and SSH into the instance:
```bash
ssh -i your-key.pem ubuntu@<ec2-public-ip>
```

---

### Step 2: Create and Attach EBS Volume
1. Go to **EC2 → Elastic Block Store → Volumes → Create Volume**
2. Size: 20 GB, same Availability Zone as your EC2 instance
3. Attach it to your EC2 instance via **Actions → Attach Volume**

---

### Step 3: Format and Mount the EBS Volume
```bash
sudo lsblk
sudo mkfs.ext4 /dev/nvme1n1
mkdir -p ~/data
sudo mount /dev/nvme1n1 ~/data
df -h
```

---

### Step 4: Make the Mount Persistent (fstab)
```bash
sudo blkid /dev/nvme1n1
# Copy the UUID from the output, then:
echo "UUID=<your-uuid> /home/ubuntu/data ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
sudo mount -a
df -h
sudo reboot
# After reboot, verify with: df -h
```

---

### Step 5: Install Docker
```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
# Log out and back in, then verify:
docker --version
docker ps
```

---

### Step 6: Run Gitea with EBS Bind Mount
```bash
sudo chown -R ubuntu:ubuntu ~/data

docker run -d \
  --name gitea \
  -p 3000:3000 \
  -v ~/data:/data \
  --restart always \
  gitea/gitea:latest

docker ps
docker inspect gitea --format '{{json .Mounts}}'
```

Visit `http://<ec2-public-ip>:3000` to complete the Gitea initial setup.

---

### Step 7: Persistence Test
```bash
# Stop and restart container
docker stop gitea
docker start gitea

# Remove and recreate container
docker stop gitea
docker rm gitea
docker run -d \
  --name gitea \
  -p 3000:3000 \
  -v ~/data:/data \
  --restart always \
  gitea/gitea:latest
```
Data survives both operations because it lives on the EBS volume.

---

## Backup Instructions

### Create S3 Bucket
1. Go to **S3 → Create Bucket**
2. Name: `gitea-backup-macanthony-2026` (globally unique)
3. Block all public access: ON

### Attach IAM Role (No Hardcoded Credentials)
1. Go to **IAM → Roles → Create Role → EC2 → AmazonS3FullAccess**
2. Name: `gitea-ec2-s3-role`
3. Attach to EC2: **Actions → Security → Modify IAM Role**

### Install AWS CLI
```bash
sudo snap install aws-cli --classic
aws --version
aws sts get-caller-identity  # verify role is working
```

### Run Backup Script
```bash
chmod +x ~/backup.sh
~/backup.sh
```

### Upload to S3
```bash
ARCHIVE="$(ls -t /tmp/gitea-backup-*.tar.gz | head -n 1)"
aws s3 cp $ARCHIVE s3://gitea-backup-macanthony-2026/backups/
aws s3 ls s3://gitea-backup-macanthony-2026/backups/
```

---

## Restore Instructions

### Step 1 — Stop the container
```bash
docker stop gitea
```

### Step 2 — Download backup from S3
```bash
aws s3 cp s3://gitea-backup-macanthony-2026/backups/<backup-filename>.tar.gz /tmp/restore.tar.gz
```

### Step 3 — Restore data
```bash
sudo tar -xzf /tmp/restore.tar.gz -C ~/data/
sudo chown -R 1000:1000 ~/data/
```

### Step 4 — Restart container
```bash
docker start gitea
```

### Step 5 — Verify
Visit `http://<ec2-public-ip>:3000` and confirm your repository is present.

---

## Notes
- Never hardcode AWS credentials in scripts. Always use IAM instance roles.
- Use `nofail` in fstab so the instance boots even if the EBS volume is temporarily unavailable.
- Always stop the Gitea container before restoring to avoid data inconsistency.
- Use `sudo chown -R 1000:1000 ~/data/` after restore to fix Gitea file permissions.

## AI Tools Used
Claude (Anthropic) was used to assist with step-by-step guidance, troubleshooting fstab persistence issues, Docker bind mount configuration, and generating this documentation.
