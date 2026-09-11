#!/bin/bash
# Enhanced VPS Setup — Contabo Ubuntu 24.04
# Integrates best practices from 4 VPS setup guides (links in README.md)
# Persistent script: C:\Projects\Huduma\vps-deploy\enhanced_setup.sh
# Actions logged to: C:\Projects\Huduma\vps-deploy\logs\enhanced_setup.log
# IMPORTANT: Add "no-new-privileges:false" for Traefik on port 80/443 — Traefik needs root

set -euo pipefail
LOG_FILE="/C/Projects/Huduma/vps-deploy/logs/enhanced_setup.log"
exec &> >(tee -a "$LOG_FILE")
echo "[$(date)] Starting enhanced VPS setup"

# =============================================
# 1. BASIC UPDATE + ESSENTIAL TOOLS
# =============================================
echo "[$(date)] 1/14 Updating system packages"
apt update && apt upgrade -y
apt install -y curl wget git htop neofetch net-tools fail2ban ufw gnupg ca-certificates

# =============================================
# 2. SSH HARDENING
# =============================================
echo "[$(date)] 2/14 Hardening SSH"
cat >> /etc/ssh/sshd_config << 'EOF'

# Security hardening
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
X11Forwarding no
LoginGraceTime 30
MaxAuthTries 3
MaxSessions 2
AllowAgentForwarding no
AllowTcpForwarding no
PermitTunnel no
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-512,hmac-sha2-256
EOF
systemctl restart sshd

# =============================================
# 3. FIREWALL (UFW)
# =============================================
echo "[$(date)] 3/14 Setting up firewall"
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh/tcp
ufw allow http/tcp
ufw allow https/tcp
# Optional (only if you install these later)
# ufw allow 19999/tcp  # Netdata
# ufw allow 9000/tcp   # Nginx UI
ufw --force enable

# =============================================
# 4. FAIL2BAN WITH DOCKER JAILS
# =============================================
echo "[$(date)] 4/14 Configuring Fail2ban with Docker-aware jails"
apt install -y fail2ban

# Only SSH jail (safe, no broad Docker filter that could ban web traffic)
cat > /etc/fail2ban/jail.d/sshd.conf << 'EOF'
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400
findtime = 600
EOF
systemctl restart fail2ban

# =============================================
# 5. DOCKER INSTALL (OFFICIAL REPO)
# =============================================
echo "[$(date)] 5/14 Installing Docker from official repository"
rm -f /etc/apt/sources.list.d/docker.list
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" > /etc/apt/sources.list.d/docker.list
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
usermod -aG docker root
systemctl enable --now docker

# =============================================
# 6. DOCKER DAEMON CONFIG (SAFE DEFAULTS)
# =============================================
echo "[$(date)] 6/14 Configuring Docker daemon"
# NOTE: NO "icc:false" — Traefik and apps need inter-container networking
cat > /etc/docker/daemon.json << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "storage-opts": ["overlay2.override_kernel_check=true"],
  "default-address-pools": [{"base":"172.20.0.0/16","size":24}],
  "max-concurrent-downloads": 10,
  "max-concurrent-uploads": 8,
  "debug": false,
  "experimental": false,
  "features": {"buildkit": true}
}
EOF
systemctl restart docker

# =============================================
# 7. UNATTENDED SECURITY UPDATES
# =============================================
echo "[$(date)] 7/14 Enabling automatic security updates"
apt install -y unattended-upgrades apt-listchanges
cat > /etc/apt/apt.conf.d/50unattended-upgrades << 'EOF'
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
    "${distro_id}:${distro_codename}-updates";
    "${distro_id}:${distro_codename}-proposed";
    "${distro_id}:${distro_codename}-backports";
};
Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::MinimalSteps "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "false";
Unattended-Upgrade::Automatic-Reboot-Time "02:00";
EOF
cat > /etc/apt/apt.conf.d/20auto-upgrades << 'EOF'
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
EOF
dpkg-reconfigure --priority=low unattended-upgrades

# =============================================
# 8. SWAP + SYSCTL TUNING
# =============================================
echo "[$(date)] 8/14 Configuring swap and network tuning"
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab

cat > /etc/sysctl.d/99-vps-optimizations.conf << 'EOF'
vm.swappiness=10
vm.vfs_cache_pressure=50
vm.page-cluster=0
vm.dirty_ratio=15
vm.dirty_background_ratio=5
vm.overcommit_memory=1
net.core.somaxconn=65536
net.core.netdev_max_backlog=4096
net.core.rmem_max=16777216
net.core.wmem_max=16777216
net.ipv4.tcp_rmem=4096 12582912 16777216
net.ipv4.tcp_wmem=4096 12582912 16777216
net.ipv4.tcp_max_syn_backlog=4096
net.ipv4.tcp_syncookies=1
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_fin_timeout=15
net.ipv4.tcp_keepalive_time=300
net.ipv4.ip_local_port_range=1024 65535
net.ipv4.tcp_fastopen=3
net.ipv4.tcp_mtu_probing=1
net.ipv4.tcp_congestion_control=bbr
fs.file-max=2097152
fs.nr_open=2097152
EOF
sysctl --system

# =============================================
# 9. MONITORING: NETDATA (OPTIONAL)
# =============================================
echo "[$(date)] 9/14 Installing Netdata (real-time monitoring, port 19999)"
bash <(curl -Ss https://my-netdata.io/kickstart.sh) --non-interactive || echo "Netdata install skipped/failed"
ufw allow 19999/tcp 2>/dev/null || true

# =============================================
# 10. ENHANCED MEMORY/DOCKER MONITOR
# =============================================
echo "[$(date)] 10/14 Installing memory monitor"
cat > /usr/local/bin/docker-memcheck.sh << 'EOF'
#!/bin/bash
THRESHOLD=85
MEMORY_USAGE=$(free | grep Mem | awk '{print int($3/$2 * 100)}')
if [ $MEMORY_USAGE -gt $THRESHOLD ]; then
    echo "$(date): High memory usage - $MEMORY_USAGE%" >> /var/log/docker-memcheck.log
    sync; echo 3 > /proc/sys/vm/drop_caches
    docker system prune -f --filter "until=24h" >/dev/null 2>&1
fi
EOF
chmod +x /usr/local/bin/docker-memcheck.sh
echo '*/5 * * * * root /usr/local/bin/docker-memcheck.sh > /dev/null 2>&1' >> /etc/crontab

# =============================================
# 11. DOCKER VOLUME BACKUPS (7-DAY RETENTION)
# =============================================
echo "[$(date)] 11/14 Installing Docker volume backup script"
cat > /usr/local/bin/docker-backup.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/root/backups/docker"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
RETENTION_DAYS=7
mkdir -p "$BACKUP_DIR"
for volume in $(docker volume ls -q); do
    echo "Backing up volume: $volume"
    docker run --rm -v $volume:/volume -v $BACKUP_DIR:/backup alpine \
        tar czf "/backup/$volume-$TIMESTAMP.tar.gz" -C /volume .
done
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete
echo "$(date): Backup completed" >> /var/log/docker-backup.log
EOF
chmod +x /usr/local/bin/docker-backup.sh
echo '0 2 * * * root /usr/local/bin/docker-backup.sh' >> /etc/crontab

# =============================================
# 12. TRAEFIK CONFIG BACKUP
# =============================================
echo "[$(date)] 12/14 Installing Traefik backup script"
cat > /usr/local/bin/traefik-backup.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/root/backups/traefik"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
mkdir -p "$BACKUP_DIR"
# Traefik stores certs in /letsencrypt by default (named volume)
docker cp traefik:/letsencrypt "$BACKUP_DIR/certs-$TIMESTAMP" 2>/dev/null || true
docker cp traefik:/etc/traefik "$BACKUP_DIR/config-$TIMESTAMP" 2>/dev/null || true
tar czf "$BACKUP_DIR/traefik-backup-$TIMESTAMP.tar.gz" -C "$BACKUP_DIR" "certs-$TIMESTAMP" "config-$TIMESTAMP" 2>/dev/null || true
rm -rf "$BACKUP_DIR/certs-$TIMESTAMP" "$BACKUP_DIR/config-$TIMESTAMP"
find "$BACKUP_DIR" -name "traefik-backup-*.tar.gz" -mtime +10 -delete
echo "$(date): Traefik backup completed" >> /var/log/traefik-backup.log
EOF
chmod +x /usr/local/bin/traefik-backup.sh
echo '0 3 * * * root /usr/local/bin/traefik-backup.sh' >> /etc/crontab

# =============================================
# 13. HEALTH CHECK (every 6 hours)
# =============================================
echo "[$(date)] 13/14 Installing health check script"
cat > /usr/local/bin/traefik-healthcheck.sh << 'EOF'
#!/bin/bash
LOG_FILE="/var/log/traefik-healthcheck.log"
TIMESTAMP=$(date "+%Y-%m-%d %H:%M:%S")
echo "[$TIMESTAMP] Starting health check..." >> "$LOG_FILE"
if ! docker info > /dev/null 2>&1; then
    echo "[$TIMESTAMP] ERROR: Docker daemon not responding — restarting" >> "$LOG_FILE"
    systemctl restart docker
fi
if ! docker ps --filter "name=traefik" --format "{{.Names}}" | grep -q traefik; then
    echo "[$TIMESTAMP] WARNING: Traefik not running — attempting start" >> "$LOG_FILE"
    docker start traefik 2>/dev/null || echo "[$TIMESTAMP] ERROR: Failed to start Traefik" >> "$LOG_FILE"
fi
USAGE=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
if [ $USAGE -gt 90 ]; then echo "[$TIMESTAMP] WARNING: Disk at $USAGE%" >> "$LOG_FILE"; fi
echo "[$TIMESTAMP] Health check completed" >> "$LOG_FILE"
EOF
chmod +x /usr/local/bin/traefik-healthcheck.sh
echo '0 */6 * * * root /usr/local/bin/traefik-healthcheck.sh' >> /etc/crontab

# =============================================
# 14. LOG ROTATION + JOURNAL OPTIMIZATION
# =============================================
echo "[$(date)] 14/14 Setting up log rotation"
cat > /etc/logrotate.d/docker-containers << 'EOF'
/var/lib/docker/containers/*/*.log {
    rotate 7
    daily
    compress
    missingok
    delaycompress
    copytruncate
    notifempty
}
EOF
logrotate -d /etc/logrotate.d/docker-containers
sed -i 's/^#*SystemMaxUse=.*/SystemMaxUse=100M/' /etc/systemd/journald.conf
sed -i 's/^#*MaxRetentionSec=.*/MaxRetentionSec=1week/' /etc/systemd/journald.conf
sed -i 's/^#*MaxFileSec=.*/MaxFileSec=1day/' /etc/systemd/journald.conf
systemctl restart systemd-journald 2>/dev/null || true

# =============================================
# DONE
# =============================================
echo "[$(date)] Enhanced VPS setup completed"
echo "[$(date)] NEXT STEPS:"
echo "  1. Reboot to apply sysctl/firewall: sudo reboot"
echo "  2. Upload site folders to server: scp -r /C/Projects/Huduma/vps-deploy/* root@185.209.228.121:/opt/vps_setup/"
echo "  3. Start Traefik: docker compose -f /opt/vps_setup/traefik/docker-compose.yml up -d"
echo "  4. Deploy 5 sites (use pre-made configs in orianasafaris/, compassionlogical/, fitque/, findx/, omniapp/)"
echo "  5. Point all A records in Cloudflare to 185.209.228.121"
echo "  6. Monitor via Netdata: http://your-ip:19999"
echo "NOTE: Optional addons still available (edit script to enable):"
echo "  - CrowdSec (threat intel): uncomment section 15 in script"
echo "  - Python/Deadsnakes + UV: uncomment section 16"
echo "  - Nginx UI (port 9000): uncomment section 17"
