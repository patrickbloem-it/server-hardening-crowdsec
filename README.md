# Server Hardening with CrowdSec: Migration from Fail2Ban

Hardening a **Netbird VPN Management Server** with **CrowdSec**, **Caddy**, and **nftables** on Ubuntu 24.04 LTS.

---

## Table of Contents

- [Why CrowdSec?](#why-crowdsec)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Caddy Configuration](#caddy-configuration)
- [CrowdSec Configuration](#crowdsec-configuration)
- [Firewall Bouncer Setup](#firewall-bouncer-setup)
- [Validation & Monitoring](#validation--monitoring)
- [Troubleshooting](#troubleshooting)

---

## Why CrowdSec?

**Fail2Ban** is reactive and isolated. **CrowdSec** is collaborative and preventive.

| Aspect | Fail2Ban | CrowdSec |
|--------|----------|----------|
| **Detection** | Waits for N failed attempts | Matches patterns + threat intelligence |
| **Scope** | Local only | Community blocklist (CAPI) |
| **Language** | Python (regex-heavy) | Go (performant, JSON-native) |
| **Response Time** | Minutes (after threshold) | Seconds (community data) |

**Key Benefits:**
- 🛡️ **Preventive:** Block attackers before they cause damage (Community Blocklist).
- ⚡ **Performant:** Go engine vs. Python daemon; ~99% less log noise.
- 🤝 **Collaborative:** Share threat data globally via CAPI.
- 🔧 **Modern:** Native JSON log parsing for Caddy.

---

## Prerequisites

```bash
# Ubuntu 24.04 LTS
uname -a
# Expected: Linux [hostname] 6.x.x-xx-generic x86_64 GNU/Linux

# Verify nftables is active (Ubuntu 24.04 default)
iptables -V | grep nf_tables

# Time synchronization (critical for CrowdSec)
timedatectl set-ntp true
timedatectl status
```

**Package Requirements:**
- `curl`, `gnupg`, `lsb-release`, `software-properties-common`

---

## Installation

### Step 1: Repository Setup

```bash
# Add CrowdSec repository and GPG key
curl -s https://install.crowdsec.net | sudo sh

# Update package list
sudo apt update
```

### Step 2: Install CrowdSec Security Engine

```bash
# Install the core security engine
sudo apt install -y crowdsec

# Verify installation
sudo cscli version

# Enable and start service
sudo systemctl enable crowdsec
sudo systemctl start crowdsec
sudo systemctl status crowdsec --no-pager
```

### Step 3: Install Essential Collections

```bash
# Linux security collections (SSH, syslog, etc.)
sudo cscli collections install crowdsecurity/linux

# Caddy log parsing
sudo cscli collections install crowdsecurity/caddy-logs

# Whitelists (reduce false positives)
sudo cscli parsers install crowdsecurity/whitelists

# Reload engine
sudo systemctl reload crowdsec
```

### Step 4: Install nftables Firewall Bouncer

```bash
# Install the firewall bouncer for nftables
sudo apt install -y crowdsec-firewall-bouncer-nftables

# Verify bouncer registration
sudo cscli bouncers list

# Expected output: 
# Name: crowdsec-firewall-bouncer-nftables
# Status: active

# Enable and start bouncer
sudo systemctl enable crowdsec-firewall-bouncer
sudo systemctl start crowdsec-firewall-bouncer
sudo systemctl status crowdsec-firewall-bouncer --no-pager
```

---

## Caddy Configuration

### Caddyfile: JSON Logging

Structured JSON logs allow CrowdSec to parse efficiently without regex overhead.

**File:** `/etc/caddy/Caddyfile`

```caddy
{
    log {
        output file /var/log/caddy/access.log {
            roll_size 100mb
            roll_keep 5
            roll_keep_for 720h
        }
        format json
        level info
    }
}

# Your reverse proxy blocks here
your-domain.com {
    reverse_proxy localhost:8080
}
```

### Verify Log Output

```bash
# Start Caddy (or reload if already running)
sudo systemctl restart caddy

# Check JSON format
sudo tail -f /var/log/caddy/access.log | jq '.'

# Expected JSON structure:
# {
#   "level": "info",
#   "ts": 1735651200.5,
#   "logger": "http.log.access",
#   "msg": "handled request",
#   "request": {
#     "method": "GET",
#     "uri": "/dashboard",
#     "proto": "HTTP/2.0",
#     "remote_addr": "203.0.113.42:54321",
#     "headers": {
#       "User-Agent": "Mozilla/5.0...",
#       ...
#     }
#   },
#   "status": 200,
#   "duration": 0.025
# }
```

---

## CrowdSec Configuration

### Create Caddy Acquisition File

CrowdSec monitors specific log files via acquisition configs.

**File:** `/etc/crowdsec/acquis.d/caddy.yaml`

```yaml
filenames:
  - /var/log/caddy/access.log
labels:
  type: caddy
```

### Fine-Tune Ban Duration

By default, CrowdSec bans for 4 hours. For persistent botnet IPs, extend to 48 hours.

**File:** `/etc/crowdsec/profiles.yaml.local` (create if doesn't exist)

```yaml
name: default
debug: false
rules:
  - type: ban
    duration: 48h
    exemptedRules: []
notifications: []
```

### Enable Community Blocklist

Download and update the community-contributed blocklist automatically.

```bash
# Install the blocklist collection
sudo cscli collections install crowdsecurity/community-blocklist

# Verify it's installed
sudo cscli collections list

# Reload
sudo systemctl reload crowdsec
```

**How it works:**
1. CrowdSec CAPI aggregates threat signals from all users.
2. Once consensus is reached, an IP lands on the **Community Blocklist**.
3. Your nftables bouncer downloads this list periodically.
4. Attackers are blocked *before* they send the first malicious packet.

---

## Firewall Bouncer Setup

### Verify nftables Integration

The bouncer automatically creates firewall rules. Inspect them:

```bash
# List CrowdSec nftables chains
sudo nft list ruleset | grep -i crowdsec

# Expected output shows:
# chain crowdsec-drop (priority filter -1; policy accept;)
# [counters showing dropped packets]
```

### Check Bouncer Configuration

**File:** `/etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml`

Key settings:
```yaml
# How often to fetch decisions
update_frequency: 10s

# Which firewall to use (nftables is default on Ubuntu 24.04)
firewall_type: nftables

# IPv4 and IPv6 support
support_ipv6: true
```

### Test the Bouncer

```bash
# Check active decisions (bans)
sudo cscli decisions list

# Example output:
# Duration | Scope  | Value | Decision | Reason
# 48h      | Ip     | 192.0.2.100 | ban | crowdsecurity/http-crawl-non_statics
# 48h      | Ip     | 198.51.100.42 | ban | crowdsecurity/ssh-bf
```

---

## Validation & Monitoring

### System Health Check

```bash
# All services running?
sudo systemctl status crowdsec crowdsec-firewall-bouncer caddy --no-pager

# Logs in good health?
sudo journalctl -u crowdsec -n 20 -f
```

### Performance Metrics

**Display aggregated statistics:**

```bash
# Full metrics dashboard
sudo cscli metrics show

# Breakdown by acquisition source
sudo cscli metrics show acquisition

# Expected output:
# crowdsec/http-crawl-non_statics      | 142 hits | 28 decisions
# crowdsecurity/ssh-bf                 | 89 hits  | 15 decisions
# ...
```

### Alert History

```bash
# Show recent alerts (why decisions were made)
sudo cscli alerts list

# Detailed alert for an IP
sudo cscli alerts list --ip 192.0.2.100

# Example:
# Alert ID: 123
# Remediation: ban
# Duration: 48h
# Reason: crowdsecurity/http-crawl-non_statics (Web scraper detected)
# Events: 145 in 5 minutes
```

### Monitor Live Blocking

```bash
# Watch packets being dropped in real-time
sudo nft monitor

# Or check nftables statistics
sudo nft list ruleset | grep -A 5 "crowdsec-drop"
```

---

## Troubleshooting

### Bouncer Not Authenticating

```bash
# Bouncer fails to connect to LAPI?
sudo systemctl restart crowdsec-firewall-bouncer

# Regenerate API credentials
sudo apt reinstall -y crowdsec-firewall-bouncer-nftables

# Verify bouncer is registered
sudo cscli bouncers list
```

### Caddy Logs Not Parsed

```bash
# Check if CrowdSec is reading the log file
sudo cscli metrics show acquisition

# If parsing fails, verify JSON format
sudo tail -1 /var/log/caddy/access.log | jq .

# Ensure caddy collection is installed
sudo cscli collections list | grep caddy

# If missing, install it
sudo cscli collections install crowdsecurity/caddy-logs
sudo systemctl reload crowdsec
```

### No Bans Being Applied

```bash
# Verify nftables bouncer is running
sudo systemctl status crowdsec-firewall-bouncer

# Check if decisions are being made
sudo cscli alerts list

# Verify nftables rules exist
sudo nft list ruleset | grep crowdsec

# If empty, restart bouncer
sudo systemctl restart crowdsec-firewall-bouncer
```

### High CPU Usage

```bash
# This is rare, but check if parser routines need tuning
sudo cat /etc/crowdsec/config.yaml | grep routines

# Increase parser routines for high-log-volume environments
# parser_routines: 4  # (default: 1, adjust based on CPU cores)
```

---

## Useful Commands Reference

```bash
# General
sudo cscli version                    # Version info
sudo cscli config show               # Current config
sudo cscli hub list                  # All installed collections

# Monitoring
sudo cscli metrics show              # Full statistics
sudo cscli metrics show acquisition  # Log parsing rate
sudo cscli alerts list               # Recent alerts
sudo cscli decisions list            # Active bans

# Collections & Parsers
sudo cscli collections install [name]    # Install a collection
sudo cscli parsers install [name]        # Install a parser
sudo cscli scenarios list                # Installed detection scenarios

# Bouncers
sudo cscli bouncers list             # Registered bouncers
sudo cscli bouncers add [name]       # Register new bouncer

# Debugging
sudo journalctl -u crowdsec -f       # Live logs
sudo journalctl -u crowdsec-firewall-bouncer -f  # Bouncer logs
sudo nft list ruleset                # Firewall rules
```

---

## Next Steps

1. **Automate backups** of `/etc/crowdsec/` for disaster recovery.
2. **Enable CAPI enrollment** for better threat intelligence correlation.
3. **Monitor dashboards** at [console.crowdsec.net](https://console.crowdsec.net).
4. **Fine-tune scenarios** by adjusting ban durations and whitelist rules.
5. **Integrate with Slack/Email** for alert notifications.

---

## References

- [CrowdSec Official Docs](https://docs.crowdsec.net/)
- [Caddy Reverse Proxy](https://caddyserver.com/)
- [CrowdSec vs Fail2Ban](https://www.crowdsec.net/blog/crowdsec-not-your-typical-fail2ban-clone)
- [Ubuntu 24.04 nftables Documentation](https://wiki.ubuntu.com/NftablesRoadmap)

---

**Last Updated:** December 31, 2025  
**Tested On:** Ubuntu 24.04 LTS (Noble Numbat) | CrowdSec v1.6+
