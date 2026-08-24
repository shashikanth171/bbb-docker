# Storing BigBlueButton recordings on S3 (native install)

Back a **native** (bare-VM, `bbb-install.sh`) BigBlueButton installation with S3
object storage. A **folder in the bucket is the data root**; recordings are
written there by the recording pipeline and served by nginx, with no change to
playback URLs or pipeline config.

This is the first step of a two-step migration. The second step — moving to the
docker/Coolify deployment — is covered in [migrate-to-coolify.md](migrate-to-coolify.md).
Because both steps use the **same bucket folder as the data root**, no recordings
need to be re-uploaded at migration time.

## Layout model

BigBlueButton stores everything under `/var/bigbluebutton` (raw archive,
published and unpublished playback). Native BBB serves it at `/playback/...`.
The docker deployment mounts the same directory into its containers, so native
and docker agree on the same tree if they both use the same object-storage folder.

```
<bucket>/
└── bbb-data/                    <- data root (chosen folder in the bucket)
    ├── bigbluebutton/           <- maps to /var/bigbluebutton (native) and
    │                              $BBB_DATA_DIR/bigbluebutton (docker)
    ├── freeswitch-meetings/
    ├── mediasoup/
    └── bbb-webrtc-recorder/
```

- **Native:** mount `<remote>:<bucket>/bbb-data` over `/var/bigbluebutton`.
- **Docker:** mount the same `<remote>:<bucket>/bbb-data` at `BBB_DATA_DIR`.

The tree under the mount is identical in both, so migration is a cutover, not a
copy.

## Placeholders

| Placeholder | Meaning |
| ----------- | ------- |
| `<remote>`  | rclone remote name (e.g. `s3` for AWS, `minio` for MinIO, `r2` for Cloudflare R2, `wasabi`, ...) |
| `<bucket>`  | S3 bucket name |
| `bbb-data`  | folder **inside** the bucket that is the data root (choose once; keep identical for the docker step) |

## Phase 1 — Provision object storage

On the native VM:

```bash
# 1. Create the bucket and a data-root folder (bbb-data). Provider-dependent.
# 2. Create an access key scoped to the data root, e.g. for AWS:
#      s3:GetObject, s3:PutObject, s3:ListBucket, s3:DeleteObject
#      on arn:aws:s3:::<bucket>/bbb-data/*

# 3. Install rclone + fuse
curl https://rclone.org/install.sh | sudo bash
sudo apt-get install -y fuse3

# 4. Configure the S3 remote (choose "s3", or "minio" provider for MinIO;
#    set endpoint, region, access/secret keys)
rclone config
```

Verify the remote:

```bash
rclone mkdir <remote>:<bucket>/bbb-data
rclone lsd <remote>:<bucket>
```

## Phase 2 — Migrate existing recordings into the bucket

Do this **before** mounting over the live directory. Quiesce the recording
pipeline so the tree is not written to during the copy.

```bash
sudo systemctl stop bbb-rap-worker

rclone copy /var/bigbluebutton <remote>:<bucket>/bbb-data \
    --transfers 8 --checksum --fast-list

sudo systemctl start bbb-rap-worker

# verify integrity
rclone check /var/bigbluebutton <remote>:<bucket>/bbb-data
```

- Copy the **whole** `/var/bigbluebutton` (raw archive + published +
  unpublished), not just `published/` — future reprocessing needs `raw/`, and
  Greenlight links to `published/`.
- Keep the local copy until the docker step is verified (rollback).

## Phase 3 — Mount the data root over `/var/bigbluebutton`

Create a systemd mount unit so the mount survives reboots.

```ini
# /etc/systemd/system/var-bigbluebutton.mount
[Unit]
After=network-online.target
Wants=network-online.target

[Mount]
What=<remote>:<bucket>/bbb-data
Where=/var/bigbluebutton
Type=rclone
Options=rw,nofail,_netdev,args2env,vfs_cache_mode=writes,config=/root/.config/rclone/rclone.conf,cache_dir=/var/cache/rclone
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now var-bigbluebutton.mount

sudo chown -R bigbluebutton:bigbluebutton /var/bigbluebutton

sudo bbb-conf --restart
sudo bbb-conf --check
```

`vfs_cache_mode=writes` is required so the recording pipeline (resque workers,
ffmpeg) and the playback nginx see normal local write + read semantics.

## Phase 4 — Verify on native

1. Record a short meeting.
2. Confirm playback at the normal `/playback/<id>` URL.
3. Confirm new objects appear under `<remote>:<bucket>/bbb-data`:

   ```bash
   rclone lsd <remote>:<bucket>/bbb-data
   ```

4. Confirm **old** recordings (from Phase 2) still play from the mount.

## Caveats

- `vfs_cache_mode=writes` is mandatory — without it the recording pipeline and
  playback nginx break.
- S3 cannot store empty directories; create `bigbluebutton`,
  `freeswitch-meetings`, `mediasoup`, `bbb-webrtc-recorder` so bind mounts
  resolve, and ensure write access for the service UIDs.
- The data-root folder name (`bbb-data`) must be **identical** on the native
  VM and the later docker host — that is the single point that makes the
  migration seamless.

Next step: see [migrate-to-coolify.md](migrate-to-coolify.md).
