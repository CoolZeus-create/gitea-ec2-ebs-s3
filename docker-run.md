# Docker Run Command

The following command was used to run the Gitea container with an EBS-backed bind mount:

```bash
docker run -d \
  --name gitea \
  -p 3000:3000 \
  -v ~/data:/data \
  --restart always \
  gitea/gitea:latest
```

## Explanation of Flags

- `-d` — Run container in detached (background) mode
- `--name gitea` — Name the container "gitea" for easy reference
- `-p 3000:3000` — Map host port 3000 to container port 3000 (Gitea web UI)
- `-v ~/data:/data` — Bind mount: host `~/data` (EBS volume) → container `/data` (Gitea data directory)
- `--restart always` — Automatically restart the container if it stops or after reboot
- `gitea/gitea:latest` — Official Gitea Docker image

## Verify Bind Mount
```bash
docker inspect gitea --format '{{json .Mounts}}'
```

Expected output should show:
```json
[{"Type":"bind","Source":"/home/ubuntu/data","Destination":"/data","Mode":"","RW":true,"Propagation":"rprivate"}]
```
