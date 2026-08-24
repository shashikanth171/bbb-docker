# Migrating BigBlueButton recordings to Coolify (docker) with object storage

Second step of a two-step migration. Assumes recordings are already backed by
S3 on the native install (see [s3-recordings.md](s3-recordings.md)) and the
**same bucket folder** (`<bucket>/bbb-data`) is the data root. Because native
and docker mount the same folder, **no recordings are re-uploaded** — this is a
cutover.

This doc covers deploying this fork's docker image as a Coolify-managed Docker
Compose resource. Read [COOLIFY.md](COOLIFY.md) first for the general Coolify
deployment model (bridge networking, published UDP ranges, TLS options).

## Layout model (unchanged from native)

```
<bucket>/
└── bbb-data/                    <- data root, same as native
    ├── bigbluebutton/           <- maps to /var/bigbluebutton in the containers
    ├── freeswitch-meetings/
    ├── mediasoup/
    └── bbb-webrtc-recorder/
```

On the Coolify host the bucket folder is mounted with rclone, and the compose
data volumes point at that mount.

## Phase 5 — Deploy on Coolify with object-storage data volumes

### 5.1 Mount the object-storage folder on the Coolify host

```bash
curl https://rclone.org/install.sh | sudo bash
sudo apt-get install -y fuse3
rclone config        # same remote <remote> as the native VM
```

```ini
# /etc/systemd/system/mnt-bbb-object-storage.mount
[Unit]
After=network-online.target
Wants=network-online.target

[Mount]
What=<remote>:<bucket>/bbb-data
Where=/mnt/bbb-object-storage
Type=rclone
Options=rw,nofail,_netdev,args2env,vfs_cache_mode=writes,config=/root/.config/rclone/rclone.conf,cache_dir=/var/cache/rclone
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mnt-bbb-object-storage.mount

# pre-create subdirectories so bind mounts resolve; match container UIDs
# (recording UID 998, etc.)
mkdir -p /mnt/bbb-object-storage/{bigbluebutton,freeswitch-meetings,mediasoup,bbb-webrtc-recorder}
chown -R 998:wheel /mnt/bbb-object-storage
```

### 5.2 Bake the absolute mount path into the compose file

Coolify does **not** interpolate resource env vars into compose *volume* paths
— it only injects them into the container environment. So `BBB_DATA_DIR` alone
will not move the data volumes; the generated `docker-compose.yml` must have the
**absolute** mount path baked in. See commit `96a71d3`.

Regenerate the compose with the object-storage root set, e.g.:

```bash
# set your env (DOMAIN, EXTERNAL_IPv4, secrets, COOLIFY_MODE=true, ...)
BBB_DATA_DIR=/mnt/bbb-object-storage ./scripts/generate-compose-coolify
```

This rewrites `docker-compose.yml` so every `bigbluebutton`,
`freeswitch-meetings`, `mediasoup` and `bbb-webrtc-recorder` volume points at
`/mnt/bbb-object-storage/...` instead of the default `./data`.

### 5.3 Deploy as a Coolify Compose resource

1. In Coolify, create a **Docker Compose** resource pointing at this repo's
   `docker-compose.yml` (and `mod/` + `conf/` for build contexts if building
   from source).
2. Set the container environment vars (DOMAIN, SHARED_SECRET, COOLIFY_MODE,
   ENABLE_* flags, etc.) per `sample.env` / `COOLIFY.md`.
3. Do **not** rely on `BBB_DATA_DIR` to control volumes under Coolify — the
   absolute paths are already baked in by step 5.2.
4. Deploy.

### 5.4 Verify

1. Record a short meeting; confirm playback at the normal `/playback/<id>` URL.
2. Confirm new objects appear under `<remote>:<bucket>/bbb-data`.
3. Confirm **old** recordings (from the native phase) play from the mount.
4. Keep the native VM (or a copy of `/var/bigbluebutton`) for rollback for a
   few days, then delete it.

## Caveats

- Coolify may rename containers with a `-<uuid>` suffix. The upstream `periodic`
  maintenance service `docker exec`s containers by their hardcoded names
  (`bbb-freeswitch`, `recordings`). If renamed, those jobs log non-fatal errors;
  check actual names with `docker ps` and adjust if they matter.
- `vfs_cache_mode=writes` is mandatory on the mount.
- The data-root folder name must match the native phase exactly.
- See [COOLIFY.md](COOLIFY.md) for the full Coolify caveats (TURN-over-TLS on
  443, IPv6, TLS options).
