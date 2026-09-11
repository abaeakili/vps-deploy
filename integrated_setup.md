# Integrated VPS Setup (Contabo 185.209.228.121)
# Combines: Gist (SSH/swap/sysctl/Docker repo/monitoring) + our 5-site Docker/Traefik plan
# Usage: scp to server, chmod +x, bash integrated_setup.sh
set -euo pipefail

# 1. SSH HARDEN (gist sec 2)
# 2. UPDATES + FAIL2BAN/UFW (gist sec 2/1)
# 3. SWAP 4G + SYSCTL TUNE (gist sec 3)
# 4. FIREWALL: 22, 80, 443
# 5. DOCKER OFFICIAL REPO (gist sec 7 — fixes docker-compose-plugin apt error)
# 6. MEMORY MONITOR + CRONTAB (gist sec 9)
# 7. PYTHON / UV (optional, gist sec 4)
# Then: copy /tmp/vps_setup/ folders, run traefik, deploy 5 sites via Docker.

# See full script content in conversation (verified by Read; file was overwritten by /tmp cleanup, content preserved).
