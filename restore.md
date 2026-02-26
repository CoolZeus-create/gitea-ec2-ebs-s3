# Gitea Restore Procedure

This document describes how to restore Gitea data from an S3 backup.

## Prerequisites
- AWS CLI installed and configured (IAM role attached to EC2)
- Docker installed
- EBS volume mounted at `~/data`

## Restore Steps

### Step 1 — List available backups in S3
```bash
aws s3 ls s3://gitea-backup-macanthony-2026/backups/
```
Note the filename of the backup you want to restore.

### Step 2 — Stop the Gitea container
```bash
docker stop gitea
```
> ⚠️ Always stop the container first to avoid data inconsistency during restore.

### Step 3 — Download the backup from S3
```bash
aws s3 cp s3://gitea-backup-macanthony-2026/backups/<backup-filename>.tar.gz /tmp/restore.tar.gz
```

Example:
```bash
aws s3 cp s3://gitea-backup-macanthony-2026/backups/gitea-backup-20260226T163145Z.tar.gz /tmp/restore.tar.gz
```

### Step 4 — Restore data into the EBS volume
```bash
sudo tar -xzf /tmp/restore.tar.gz -C ~/data/
```

### Step 5 — Fix file permissions
```bash
sudo chown -R 1000:1000 ~/data/
```

### Step 6 — Restart the Gitea container
```bash
docker start gitea
```

### Step 7 — Verify restore was successful
Open your browser and visit:
```
http://<ec2-public-ip>:3000
```
Sign in and confirm your repositories and data are present.

---

## Demonstration

To demonstrate restore, the repository `gitadmin/my-gitea-test2` was deleted from Gitea, then recovered using the above procedure. After restore, the repository and all commits were fully recovered.

## Notes
- The restore overwrites existing data in `~/data/` — make sure you download the correct backup
- The `nofail` option in `/etc/fstab` ensures the EBS volume mounts automatically on boot
- If Gitea shows the Initial Configuration page after restore, ensure `~/data/gitea/conf/app.ini` exists
