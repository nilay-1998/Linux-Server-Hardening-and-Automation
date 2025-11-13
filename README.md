# Linux-Server-Hardening-and-Automation
 Automated user and permission management using Bash scripting.  Configured firewalls and SELinux for enhanced server security.  Implemented regular system update scripts and log monitoring for proactive server maintenance.
#!/bin/bash
# -----------------------------------------------
# Linux Server Hardening and Automation Script
# Author: Nilay
# Description: Automates user management, updates,
# firewall, SELinux, and log monitoring.
# -----------------------------------------------

LOGFILE="/var/log/server_hardening.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log() {
  echo "[$DATE] $1" | tee -a "$LOGFILE"
}

# 1️⃣ USER & PERMISSION MANAGEMENT
manage_users() {
  log "Starting user management..."
  
  # Example: Create secure users
  for user in devops admin auditor; do
    if id "$user" &>/dev/null; then
      log "User $user already exists."
    else
      useradd -m -s /bin/bash "$user"
      echo "$user:Secure@123" | chpasswd
      passwd -e "$user" # Force password change on first login
      log "User $user created and password policy enforced."
    fi
  done

  # Restrict root SSH login
  sed -i 's/^PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
  systemctl restart sshd
  log "Root SSH login disabled for security."

  # Set permissions for home directories
  chmod 700 /home/*
  log "User home directory permissions hardened."
}

# 2️⃣ FIREWALL & SELINUX CONFIGURATION
configure_security() {
  log "Configuring firewall and SELinux..."
  
  # Enable firewalld and allow only necessary ports
  systemctl enable firewalld --now
  firewall-cmd --permanent --add-service=ssh
  firewall-cmd --permanent --add-service=http
  firewall-cmd --reload
  log "Firewall configured: only SSH and HTTP allowed."

  # Enforce SELinux
  setenforce 1
  sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
  log "SELinux set to enforcing mode."
}

# 3️⃣ AUTOMATED SYSTEM UPDATES
system_updates() {
  log "Running system updates..."
  dnf update -y && log "System updated successfully." || log "System update failed!"
}

# 4️⃣ LOG MONITORING & ALERTS
monitor_logs() {
  log "Checking logs for suspicious activity..."

  # Look for failed SSH login attempts
  FAILED_LOGINS=$(grep "Failed password" /var/log/secure | tail -n 10)
  if [ -n "$FAILED_LOGINS" ]; then
    echo "$FAILED_LOGINS" | mail -s "Security Alert: Failed SSH Logins Detected" root@localhost
    log "Failed SSH login attempts detected and reported."
  else
    log "No failed SSH login attempts found."
  fi
}

# 5️⃣ SCHEDULED TASKS
setup_cron_jobs() {
  log "Setting up cron jobs..."
  CRON_FILE="/etc/cron.d/server_hardening"

  cat <<EOF > "$CRON_FILE"
# Run updates daily at 2 AM
0 2 * * * root /usr/bin/dnf -y update >> /var/log/auto_update.log 2>&1

# Run log monitoring every hour
0 * * * * root /bin/bash /root/server_hardening.sh --monitor >> /var/log/server_hardening.log 2>&1
EOF

  chmod 644 "$CRON_FILE"
  log "Cron jobs configured for updates and log monitoring."
}

# 🧠 MAIN EXECUTION
case "$1" in
  --monitor)
    monitor_logs
    ;;
  *)
    manage_users
    configure_security
    system_updates
    setup_cron_jobs
    monitor_logs
    log "Server hardening completed successfully!"
    ;;
esac
