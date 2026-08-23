# Deploying BigBlueButton 3.0 Docker on Coolify

This fork keeps the upstream `bigbluebutton/docker` templating workflow
(`docker-compose.tmpl.yml` + `.env` + `mod/` → `./scripts/generate-compose`), and
adds a **Coolify mode** (`COOLIFY_MODE=true`) that makes the generated
`docker-compose.yml` deployable as a Coolify-managed **Docker Compose** resource.

## Why the template, not a hand-patched file

`docker-compose.yml` is generated, not static. Coolify-specific changes live in
`docker-compose.tmpl.yml` behind a `COOLIFY_MODE` switch, so:

- upstream template changes can still be merged and regenerated,
- nothing is hand-edited into the generated file (no drift).

## Architecture decisions

### Networking: bridge-only, no `network_mode: host`

Upstream runs `webrtc-sfu`, `coturn`, `bbb-webrtc-recorder` and dev-mode
`html5-dev` with `network_mode: host`. Coolify injects its own `networks:` block
into every service of a Compose resource, which is **mutually exclusive** with
`network_mode` and fails the deployment.

In Coolify mode all four services join the existing `bbb-net` bridge with static
IPs, and the media traffic is exposed via **published UDP port ranges**:

| Service           | bbb-net IP | Published ports                    | Purpose                          |
| ----------------- | ---------- | ---------------------------------- | -------------------------------- |
| `webrtc-sfu`      | 10.7.7.11  | `RTP_PORT_RANGE_START-END`/udp     | mediasoup WebRTC media           |
| `coturn`          | 10.7.7.13  | 3478/tcp+udp, `COTURN_*_RANGE`/udp | STUN/TURN relay                  |
| `bbb-webrtc-recorder` | 10.7.7.23 | – (bridged to SFU internally)   | WebRTC recording peer            |
| `html5-dev`       | 10.7.7.24  | – (dev only)                       | dev html5 client                 |

The mediasoup listen IP is set to `0.0.0.0` inside the container while the
**announced** IP stays `${EXTERNAL_IPv4}` (the server's public IPv4), so
browsers send media to the host-published ports. The SFU's baked config that
previously pointed at the host (`clientHost`/`mcs-host`/`mcs-address`/`plainRtp`)
is overridden via env vars to the SFU's own bridge IP.

### TLS: Coolify's proxy terminates TLS (Option B)

- `ENABLE_HTTPS_PROXY=false` — the bundled haproxy is NOT used. Coolify's proxy
  (Traefik/Caddy) owns ports 80/443 and terminates TLS for the assigned FQDN.
- nginx listens on **48087** (plain HTTP, the upstream "external reverse proxy"
  port). In Coolify, assign the FQDN to the **`nginx` service** with port 48087,
  e.g. `https://bbb.example.org:48087` in the Coolify domain field.
- nginx honors `X-Forwarded-Proto` (added in this fork) so BBB still generates
  `https` URLs/cookies even though it receives plain HTTP from Coolify's proxy.
- **Caveat:** the bundled haproxy also provided TURN-over-TLS on 443 (ALPN
  `stun.turn`). With Option B that path is gone; TURN still works on 3478. If
  you need TURN-over-443, a dedicated FQDN/publish on the coturn service can be
  added, or use an external TURN server.

### Media routing in Coolify mode

- Browser → Coolify proxy (TLS) → nginx:48087 → `bbb-web`/html5/Greenlight.
- Browser WebSocket → proxy → nginx → `webrtc-sfu:3008` (bridged).
- Browser RTP/UDP → host `RTP_PORT_RANGE_*` (published) → `webrtc-sfu`.
- Browser STUN/TURN → host 3478 + `COTURN_*_RANGE` (published) → `coturn`.
- `freeswitch`, `redis`, `bbb-web`, `postgres`, ... stay on `bbb-net` static IPs,
  exactly as upstream.

## RTP / TURN port range sizing

Each WebRTC participant typically uses 2–4 UDP ports (audio+video send/recv,
plus one more for screenshare for presenters). `mediasoup` binds one port per
transport, so for **~100 concurrent participants** you need roughly **300–500
ports**. The default range `24577–25000` (424 ports) covers ~100–150 users; widen
it if you expect more. The range **must** not overlap `COTURN_PORT_RANGE_*` and
**must** be open in the host firewall (both UDP).

TURN relay ports are only consumed when a client cannot reach the media server
directly (blocked UDP, symmetric NAT). Keep this range smaller: default
`32769–33268` (500 ports).

These are set in the **Coolify environment variables** (not baked), see below.

## Required Coolify environment variables

Set every variable below in the Coolify **Environment Variables** panel of the
resource. They map 1:1 to the `.env` keys the template uses.

| Variable | Example | Notes |
| --- | --- | --- |
| `COOLIFY_MODE` | `true` | Must be `true` (no host networking) |
| `BBB_DATA_DIR` | `/mnt/bbb-object-storage` | Data root for recordings + all volumes. Set to an S3-backed mount for object storage (default `./data`) |
| `DOMAIN` | `bbb.example.org` | BBB FQDN, pointed at the server |
| `EXTERNAL_IPv4` | `203.0.113.5` | Server's public IPv4 |
| `RTP_PORT_RANGE_START` | `24577` | SFU media range (UDP) |
| `RTP_PORT_RANGE_END` | `25000` | |
| `COTURN_PORT_RANGE_START` | `32769` | TURN relay range (UDP) |
| `COTURN_PORT_RANGE_END` | `33268` | |
| `SHARED_SECRET` | random | BBB API shared secret |
| `TURN_SECRET` | random | coturn auth secret (same value as `TURN_SECRET` used by SFU) |
| `ETHERPAD_API_KEY` | random | |
| `RAILS_SECRET` | random | Greenlight secret key base |
| `POSTGRESQL_SECRET` | random | Postgres password |
| `FSESL_PASSWORD` | random | FreeSWITCH ESL password |
| `ENABLE_HTTPS_PROXY` | `false` | Must be `false` (Coolify proxy owns 80/443) |
| `ENABLE_GREENLIGHT` | `true` | Greenlight front-end |
| `ENABLE_COTURN` | `true` | Bundled TURN server |
| `ENABLE_COLLABORA` | `true`/`false` | Etherpad office editing |
| `ENABLE_RECORDING` | `true`/`false` | Recording + recorder services |
| `ENABLE_WEBHOOKS` | `false` | BBB webhooks |
| `IGNORE_TLS_CERT_ERRORS` | (empty) | leave empty |
| `EXTERNAL_IPv6` | (empty) | IPv6 not supported in Coolify bridge mode |
| `SIP_IP_ALLOWLIST` | (empty) | optional SIP dial-in |

Any other `.env` key from `sample.env` (welcome message, sounds, locale, SMTP,
S3, …) can be set the same way; the template passes unknown `${VAR}` through to
the generated compose.

## Import into Coolify (Docker Compose resource)

1. **Push this fork** to GitHub and add it as a repository in Coolify
   (`Add resource → Docker Compose`).
2. Coolify clones the repo and uses the committed `docker-compose.yml`. If the
   generated file is missing, run locally and commit it:
   ```bash
   cp sample.env .env        # fill in your values
   ./scripts/generate-compose-coolify --no-build
   git add -f docker-compose.yml
   git commit -m "regenerate Coolify compose"
   git push
   ```
3. **FQDN assignment** in Coolify:
   - Assign the FQDN **to the `nginx` service only**, with port **48087**
     (e.g. `https://bbb.example.org:48087`). Coolify's proxy serves it on
     80/443 and routes to nginx:48087.
   - Do **not** assign FQDNs to `webrtc-sfu`, `coturn` or any other service —
     those are reached via published ports, not the proxy.
4. **Environment Variables**: set all required variables from the table above
   (or reference a shared environment file).
5. **Ports**: ensure the host firewall allows `80/443/tcp` and the two UDP
   ranges (`24577–25000`, `32769–33268`). Coolify publishes the ports declared
   in the compose file automatically; no `network_mode` is present, so there is
   no "mutually exclusive network_mode and networks" error.
6. **Submodules**: with `--no-build` no source build happens (prebuilt
   `alangecker/bbb-docker-*` images are pulled). Without `--no-build` you must
   check out submodules first: `git submodule update --init --recursive`.
7. Deploy. Then create the Greenlight admin:
   ```bash
   docker compose -p <coolify-project> exec greenlight bundle exec rake admin:create
   ```

## Testing WebRTC (do not stop at "containers Up")

Containers being healthy does **not** prove media works. After deploying:

1. Open the Greenlight URL, create a room, join it in **two different browsers**
   (or one browser + phone).
2. Enable camera + mic in both. Verify **two-way audio and video**.
3. Start a **screenshare** in one client, verify it renders in the other.
4. If recording is enabled, record a short meeting and play back the recording
   from the client.

Useful checks if media fails:
- `docker compose exec webrtc-sfu cat /etc/bigbluebutton/bbb-webrtc-sfu/production.yml`
  → announced IP must be the public IPv4 / bridge IPs.
- On the server: `ss -lun | grep -E '24577|25000'` → SFU must be bound.
- `docker compose logs coturn` → look for relay allocation errors.
- Browser `chrome://webrtc-internals` → inspect ICE candidates (expect the
  `EXTERNAL_IPv4` candidate over UDP, not just TURN-relayed).

## Object storage for recordings

Recordings are **not** stored via Greenlight's S3 settings — Greenlight's S3
config (`S3_*` env vars) only covers presentation file uploads (ActiveStorage).
Recorded playback is produced by the `recordings` container into
`/var/bigbluebutton/published` and served by nginx at `/playback/...` URLs that
Greenlight links to directly. So object storage must back the **data directory**.

Set `BBB_DATA_DIR` to a host path that is an object-storage-backed FUSE mount
(e.g. rclone or s3fs) of your S3/MinIO bucket. Every BBB data volume
(`bigbluebutton`, `freeswitch-meetings`, `mediasoup`, `bbb-webrtc-recorder`)
then lives on object storage:

```
# in Coolify env vars
BBB_DATA_DIR=/mnt/bbb-object-storage
```

### Example: mount an S3 bucket with rclone on the Coolify host

```bash
# once: configure rclone (choose "s3", or minio with provider "Minio")
rclone config

# make a systemd unit or just test it:
mkdir -p /mnt/bbb-object-storage
rclone mount bbb:recordings /mnt/bbb-object-storage \
    --vfs-cache-mode writes \
    --daemon
```

Notes:
- Use `--vfs-cache-mode writes` so the recording pipeline (resque workers,
  ffmpeg, nginx serving) sees normal local write semantics.
- Create the subdirectories before starting BBB
  (`bigbluebutton`, `freeswitch-meetings`, `mediasoup`, `bbb-webrtc-recorder`)
  so bind mounts resolve cleanly, and ensure write access for the container
  UIDs (recording UID 998, etc.).
- Existing local recordings: copy `./data/bigbluebutton` into the bucket first.
- Restart the BBB resource after setting `BBB_DATA_DIR`.

This keeps the upstream volume layout (`/var/bigbluebutton`, ...) unchanged, so
no nginx or recording-pipeline config changes are needed.

## Known caveats

- **`periodic` maintenance container**: Coolify may rename containers with a
  `-<uuid>` suffix. The upstream `periodic` service `docker exec`s containers by
  their hardcoded names (`bbb-freeswitch`, `recordings`) to resync clocks and
  purge old recordings. If Coolify renames them, these maintenance jobs log
  errors. They are non-fatal; if they matter to you, check the actual container
  names (`docker ps`) after first deploy and adjust accordingly.
- **TURN-over-TLS on 443** is lost in Option B (see TLS section).
- **IPv6** is not supported in Coolify bridge mode; leave `EXTERNAL_IPv6` empty.

## Upgrading / merging upstream

```bash
git fetch upstream
git merge upstream/develop          # resolves template conflicts if any
./scripts/generate-compose-coolify --no-build
git add -f docker-compose.yml && git commit -m "regenerate after upstream merge"
```

## Multi-instance / shared API note

The stack is self-contained (BBB API + media + optional Greenlight). For
multiple BBB domains sharing a single API backend with per-domain Greenlight
instances, deploy one full stack as the API and run additional Greenlight-only
Compose resources (image `bigbluebutton/greenlight:v3.8.0`) that point at the
shared API via `BIGBLUEBUTTON_ENDPOINT=https://<api-domain>/bigbluebutton/api`
and `BIGBLUEBUTTON_SECRET=<shared-secret>`. Each Coolify resource then gets its
own FQDN.