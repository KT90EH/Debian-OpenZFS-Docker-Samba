# Homebox Setup Guide  
Debian 13 + Docker + OpenZFS + Samba (LAN only)

## Recommended Image

Use the **rootless** image:

```
ghcr.io/sysadminsmedia/homebox:latest-rootless
```

| Variant       | Runs as          | Shell | Recommendation          |
|---------------|------------------|-------|-------------------------|
| Regular       | root (UID 0)     | Yes   | Avoid for long-term use |
| **Rootless**  | 65532:65532      | Yes   | **Best choice**         |
| Hardened      | 65532:65532      | No    | Maximum lockdown only   |

---

## 1. Create the ZFS Dataset

```bash
zfs create \
  -o recordsize=128K \
  -o compression=lz4 \
  -O dnodesize=auto \
  -o atime=off \
  -o xattr=sa \
  -o acltype=posixacl \
  fastpool/homebox
```

**Do not share this dataset via Samba.**  
Keep the Homebox data private to Docker.

### Why these settings?

- `recordsize=128K` — Good balance for SQLite database + uploaded photos/attachments
- `compression=lz4` — Fast and effective (you can use `zstd` if preferred)
- `atime=off` — Reduces unnecessary writes
- No Samba share — Avoids permission conflicts with UID 65532

---

## 2. User Account for UID 65532 (Optional but Recommended)

You do **not** strictly need a system account — the container only cares about the numeric UID/GID **65532**.

However, creating a no-login system account is useful because:

- `ls -l` shows a name instead of just `65532`
- Ownership is clearer in tools and documentation
- Better visibility in monitoring / auditing

### Create a no-login system account

```bash
# Create the group
groupadd --system --gid 65532 homebox

# Create the user (system account, no login, no home directory)
useradd --system \
  --uid 65532 \
  --gid 65532 \
  --shell /usr/sbin/nologin \
  --home-dir /nonexistent \
  --comment "Homebox container user" \
  homebox
```

### Verify

```bash
id homebox
# Expected: uid=65532(homebox) gid=65532(homebox) groups=65532(homebox)

getent passwd 65532
# Should show: homebox:x:65532:65532:Homebox container user:/nonexistent:/usr/sbin/nologin
```

### Set ownership

```bash
chown -R homebox:homebox /fastpool/homebox
chmod 700 /fastpool/homebox
```

(You can still use the numbers `chown -R 65532:65532` if you prefer.)

---

## 3. Generate a strong API key pepper

```bash
openssl rand -base64 48 > /fastpool/homebox/pepper
chmod 400 /fastpool/homebox/pepper
chown homebox:homebox /fastpool/homebox/pepper
```

---

## 4. Docker Compose Example

Create `docker-compose.yml`:

```yaml
services:
  homebox:
    image: ghcr.io/sysadminsmedia/homebox:latest-rootless
    container_name: homebox
    restart: unless-stopped
    environment:
      - TZ=Your/Timezone                  # e.g. America/Los_Angeles
      - HBOX_LOG_LEVEL=info
      - HBOX_LOG_FORMAT=text
      - HBOX_WEB_MAX_UPLOAD_SIZE=10
      - HBOX_OPTIONS_ALLOW_ANALYTICS=false
      - HBOX_AUTH_API_KEY_PEPPER=          # paste the content of the pepper file here
    volumes:
      - /fastpool/homebox:/data
    ports:
      - "3100:7745"                       # or bind to a specific LAN IP
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
```

### Optional extra hardening (already included above)

```yaml
security_opt:
  - no-new-privileges:true
cap_drop:
  - ALL
```

- `cap_drop: - ALL` removes every Linux capability from the container.
- `no-new-privileges:true` prevents the process from gaining new privileges later.

Homebox does not need any special capabilities, so this is safe.

---

## 5. Start the container

```bash
docker compose up -d
```

Access it at: `http://your-server-ip:3100`

---

## 6. Useful Extra Tips

- **Snapshots**: Create regular ZFS snapshots of `fastpool/homebox`. This is an excellent backup method.
- **Backups**: Occasionally export data from within Homebox (CSV/JSON) as a secondary backup.
- **Updates**: Prefer pinning a version tag (`vX.Y.Z-rootless`) and update deliberately, or use `:latest-rootless`.
- **Networking**: Prefer binding the port only to your LAN interface or putting it behind a reverse proxy with TLS.
- **Resources**: Homebox is extremely light (< 50 MB RAM idle). No special resource limits are needed.

---

## Quick Reference Commands

```bash
# Create no-login system account (optional but recommended)
groupadd --system --gid 65532 homebox
useradd --system --uid 65532 --gid 65532 --shell /usr/sbin/nologin \
  --home-dir /nonexistent --comment "Homebox container user" homebox

# Create dataset
zfs create -o recordsize=128K -o compression=lz4 -o atime=off -o xattr=sa fastpool/homebox
chown -R homebox:homebox /fastpool/homebox
chmod 700 /fastpool/homebox

# Generate pepper
openssl rand -base64 48 > /fastpool/homebox/pepper
chmod 400 /fastpool/homebox/pepper
chown homebox:homebox /fastpool/homebox/pepper

# Start
docker compose up -d

# View logs
docker logs -f homebox
```
