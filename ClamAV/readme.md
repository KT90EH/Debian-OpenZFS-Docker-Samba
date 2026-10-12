# ClamAV in Docker – Home Server Setup

Simple, low-maintenance ClamAV setup for occasional manual scans on a home server running Docker + OpenZFS + Samba.  
Designed for machines that are only powered on when needed.

AI-assisted using Grok an Claude.

---

## 1. Create the Quarantine ZFS Dataset

Create a dedicated, hardened dataset for quarantined files:

```bash
sudo zfs create -o exec=off -o setuid=off -o devices=off fastpool/quarantine
```

Optional but recommended properties:

```bash
sudo zfs set compression=lz4 fastpool/quarantine
sudo zfs set atime=off fastpool/quarantine
```

### Set group-based permissions (so you can use Cockpit File Manager)

```bash
# Create a group for quarantine access
sudo groupadd quarantine

# Add your normal user to the group (replace "youruser" with your actual username)
sudo usermod -aG quarantine youruser

# Set ownership and permissions
sudo chown root:quarantine /fastpool/quarantine
sudo chmod 770 /fastpool/quarantine
```

**Important:** Log out and back in (or reboot) after adding yourself to the group so the new group membership takes effect.

After this:
- Root and members of the `quarantine` group can browse, read, and delete files (including via Cockpit File Manager).
- Everyone else is blocked.
- The strong ZFS properties (`exec=off`, `setuid=off`, `devices=off`) remain active.

**Snapshot note:** Exclude `fastpool/quarantine` from any automatic snapshot tool (sanoid, zfs-auto-snapshot, etc.). Otherwise deleted threats will linger in snapshots.

---

## 2. Docker Compose

```yaml
services:
  clamav:
    image: clamav/clamav:stable          # or pin an exact version, e.g. clamav/clamav:1.4
    container_name: clamav
    environment:
      # Disable clamd – we only use clamscan (saves ~1 GB RAM)
      CLAMAV_NO_CLAMD: "true"
      TZ: "America/Los_Angeles"
    volumes:
      # Persist virus definitions so they don’t re-download every time
      - clamav-db:/var/lib/clamav

      # === Paths that Option B is allowed to move infected files from ===
      # (no :ro – required for --move to work)
      - /bulkpool/mediavault:/scan/mediavault
      - /fastpool/documents/perry:/scan/perry
      - /fastpool/homebox:/scan/homebox

      # === Paths that should never be moved (live app data) ===
      # Keep these read-only so Option B cannot touch them
      - /fastpool/appdata/configs:/scan/configs:ro
      - /fastpool/appdata/databases:/scan/databases:ro
      - /fastpool/appdata/cache:/scan/cache:ro

      # Quarantine folder (writable)
      - /fastpool/quarantine:/quarantine
    security_opt:
      - no-new-privileges:true
    tmpfs:
      - /tmp
      - /run
    restart: "no"
    mem_limit: 3g
    healthcheck:
      disable: true          # Healthcheck pings clamd, which we disabled

volumes:
  clamav-db:
```

**Important notes about the mounts:**
- Paths **without** `:ro` can have infected files moved to quarantine (Option B).
- Paths **with** `:ro` are protected — Option B will report them but cannot move them.
- This prevents a false positive in a live config/database/cache from breaking a running service.

**Path mapping (for reading logs later):**
| Container path          | Host path                      |
|-------------------------|--------------------------------|
| `/scan/mediavault`      | `/bulkpool/mediavault`         |
| `/scan/perry`           | `/fastpool/documents/perry`    |
| `/scan/homebox`         | `/fastpool/homebox`            |
| `/scan/configs`         | `/fastpool/appdata/configs`    |
| `/scan/databases`       | `/fastpool/appdata/databases`  |
| `/scan/cache`           | `/fastpool/appdata/cache`      |
| `/quarantine`           | `/fastpool/quarantine`         |

### First-time container creation

**Do not stop the container immediately** after the first `up`. The initial signature download must finish.

```bash
docker compose up -d

# Wait until the database files exist (can take several minutes on first run)
docker exec clamav sh -c 'ls /var/lib/clamav/main.c[vl]d /var/lib/clamav/daily.c[vl]d'

# Once the files are listed, you can stop the container
docker stop clamav
```

After that you can use the script or the one-liners normally.

**Quick check that the container runs as root** (needed for `--move` and some share permissions):

This will only work if the container is running.  ⚠️

```bash
docker exec clamav id
# Expected: uid=0(root) gid=0(root) ...
```

If it shows a non-root user, add `-u root` to the `docker exec` calls in the script.

---

## 3. Helper Script (with three options)

Save as `/usr/local/bin/clam-scan`:

```bash
#!/bin/bash
set -o pipefail          # Make pipelines return clamscan's real exit code

# ClamAV helper with three modes:
#   A) Scan only (report infected files)
#   B) Scan + move infected files to quarantine
#   C) Do nothing and exit

CONTAINER="clamav"
SCAN_PATH="/scan"
QUARANTINE="/quarantine"
LOG_FILE="/var/log/clam-scan.log"

# Common options – exclude .zfs directories (important if snapdir=visible)
CLAM_OPTS=(-r -i --exclude-dir='/\.zfs(/|$)')

wait_for_db() {
  echo "Waiting for virus database..."
  for i in {1..60}; do
    if docker exec "$CONTAINER" sh -c \
      'ls /var/lib/clamav/main.c[vl]d /var/lib/clamav/daily.c[vl]d >/dev/null 2>&1'; then
      return 0
    fi
    sleep 5
  done
  echo "Virus database not found. Check: docker logs $CONTAINER" >&2
  exit 1
}

start_container() {
  if ! docker inspect -f '{{.State.Running}}' "$CONTAINER" 2>/dev/null | grep -q true; then
    echo "Starting ClamAV container..."
    docker start "$CONTAINER"
  fi
  wait_for_db
}

scan_only() {
  start_container
  # Image already runs freshclam on start – no need to call it again
  # (a second call often hits a log-file lock and prints noisy errors)
  echo "=== $(date -Is) mode=scan ===" | tee -a "$LOG_FILE"
  echo "Signature date: $(docker exec "$CONTAINER" sh -c "stat -c '%y' /var/lib/clamav/daily.c[vl]d" 2>/dev/null)" | tee -a "$LOG_FILE"
  echo "Scanning $SCAN_PATH (report only – no files will be moved)..."
  docker exec "$CONTAINER" clamscan "${CLAM_OPTS[@]}" "$SCAN_PATH" | tee -a "$LOG_FILE"
}

scan_and_move() {
  start_container
  # Image already runs freshclam on start – no need to call it again
  echo "=== $(date -Is) mode=move ===" | tee -a "$LOG_FILE"
  echo "Signature date: $(docker exec "$CONTAINER" sh -c "stat -c '%y' /var/lib/clamav/daily.c[vl]d" 2>/dev/null)" | tee -a "$LOG_FILE"
  echo "Scanning $SCAN_PATH (infected files will be moved to $QUARANTINE)..."
  echo "Full container paths of any quarantined files are logged to $LOG_FILE"
  docker exec "$CONTAINER" clamscan "${CLAM_OPTS[@]}" --move="$QUARANTINE" "$SCAN_PATH" | tee -a "$LOG_FILE"
}

do_nothing() {
  echo "Cancelled. No scan performed."
  exit 0
}

# ---------- Main ----------
case "$1" in
  a|A|scan|report)
    scan_only
    ;;
  b|B|move|quarantine)
    scan_and_move
    ;;
  c|C|exit|quit|nothing)
    do_nothing
    ;;
  *)
    echo
    echo "Choose an option:"
    echo "  A) Scan only (report infected files, do not move them)"
    echo "  B) Scan and move infected files to /fastpool/quarantine"
    echo "  C) Do nothing and exit"
    echo
    read -rp "Enter A, B, or C: " choice
    case "$choice" in
      a|A)
        scan_only
        ;;
      b|B)
        scan_and_move
        ;;
      c|C)
        do_nothing
        ;;
      *)
        echo "Invalid choice. Exiting."
        exit 64          # Avoid collision with clamscan exit codes 0/1/2
        ;;
    esac
    ;;
esac

rc=$?
echo "Done."
exit $rc
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/clam-scan
```

Create the log file with sensible permissions:

```bash
sudo touch /var/log/clam-scan.log
sudo chown root:docker /var/log/clam-scan.log
sudo chmod 664 /var/log/clam-scan.log
```

### Limit log size with logrotate

Create `/etc/logrotate.d/clam-scan`:

```bash
sudo tee /etc/logrotate.d/clam-scan << 'EOF'
/var/log/clam-scan.log {
    weekly
    rotate 4
    compress
    delaycompress
    missingok
    notifempty
    create 664 root docker
}
EOF
```

This keeps 4 weeks of logs, compresses old ones, and recreates the file with the correct ownership/permissions after rotation.  
Test with: `sudo logrotate -d /etc/logrotate.d/clam-scan` (dry-run).

### How to use the script

**Interactive menu:**
```bash
clam-scan
```

**Direct options:**
```bash
clam-scan a          # Option A – scan only
clam-scan b          # Option B – scan + move
clam-scan c          # Option C – do nothing and exit
```

Exit codes:
- `0` = clean (or user chose Option C)
- `1` = infected file(s) found
- `2` = scan error
- `64` = invalid menu choice

**Note:** Prefer running as your normal user (member of the `docker` group). `sudo clam-scan` also works.

---

## 4. Dockhand One-Liners

These are less robust than the script (no database wait). Prefer the script when possible.

**Option A – scan only:**
```bash
docker start clamav && sleep 20 && docker exec clamav clamscan -r -i --exclude-dir='/\.zfs(/|$)' /scan
```

**Option B – scan + move:**
```bash
docker start clamav && sleep 20 && docker exec clamav clamscan -r -i --exclude-dir='/\.zfs(/|$)' --move=/quarantine /scan
```

---

## 5. Typical Workflow

1. Power on the server.
2. Run `clam-scan` (recommended) or one of the one-liners.
3. **First real scan:** Always start with Option A (report only). Your first full scan is the most likely to surface false positives (odd installers, old game files, archives, etc.). Review the report before you ever use Option B.
4. For subsequent routine scans, use Option A or B as needed.
5. Review any quarantined files in Cockpit File Manager at `/fastpool/quarantine`.
6. Check `/var/log/clam-scan.log` for the original container path (see mapping table above).
7. Delete confirmed threats from the quarantine folder.
8. **Remember:** The original file may still exist in older ZFS snapshots of the source dataset until those snapshots expire.
9. Stop the container when finished: `docker stop clamav`.

---

## 6. Testing the Setup (do this once)

Use the standard EICAR test string (harmless).  
**Note:** The first-time setup ends with the container stopped, so start it (and wait for the database) before testing.

```bash
# Create the test file (use sudo if your user cannot write to /fastpool/homebox)
mkdir -p /fastpool/homebox/eicar-test
echo 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*' \
  > /fastpool/homebox/eicar-test/eicar.txt

# Start the container and wait for the virus database
docker start clamav
docker exec clamav sh -c 'ls /var/lib/clamav/main.c[vl]d /var/lib/clamav/daily.c[vl]d'

# 1. Report-only (should flag it and leave the file alone)
docker exec clamav clamscan -r -i /scan/homebox/eicar-test

# 2. Move (should move it into quarantine)
docker exec clamav clamscan -r -i --move=/quarantine /scan/homebox/eicar-test

# Stop the container when finished
docker stop clamav
```

**What to verify:**
- Report-only run detects the file and leaves it in place.
- Move run places it in `/fastpool/quarantine`.
- The log (`/var/log/clam-scan.log` if you used the script, or the terminal output) shows the container path `/scan/homebox/eicar-test/eicar.txt`.
- You can see and delete the quarantined file via Cockpit File Manager (as a member of the `quarantine` group).  
  If Cockpit cannot delete it, the problem is the quarantine directory’s group/mode or your group membership has not refreshed yet — not ClamAV itself.

Then clean up (use `sudo` if needed):

```bash
rm -rf /fastpool/homebox/eicar-test
# and delete the quarantined copy from /fastpool/quarantine if desired
```

(If a real-time antivirus on the host flags the EICAR file, that is expected.)

**Alternative:** You can also run the test through the real script (`clam-scan a` then `clam-scan b`). That exercises the full code path (database wait, logging, exit codes), but it will scan everything under `/scan`, so it takes longer on the first run.

---

## 7. Optional Notes

- Modern ClamAV skips files larger than ~100 MB by default and treats them as clean (no error).  
  - This is usually fine for a media library full of videos.  
  - If you have large archives in `perry` or `homebox` that you want scanned, raise the limit for those paths (e.g. `--max-filesize=500M`).  
  - To skip even more files for speed, lower it (`--max-filesize=50M`).
- You can also scan specific sub-paths instead of everything under `/scan`.
- Image tag: `stable` changes less often than `latest`. You can pin an exact version if you prefer full control. Signatures still update via `freshclam` independently of the image version.

## 8. One follow-up worth doing. 

Since freshclam runs on its own in the background now, it's worth occasionally checking that it's actually succeeding. A quick check any time:

```bash
docker exec clamav sh -c "stat -c '%y' /var/lib/clamav/daily.c[vl]d"
```

If that timestamp is more than a day or two old, something's stuck and `docker logs clamav` will show why.

