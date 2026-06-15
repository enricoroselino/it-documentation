# Apply Connection Security Hardening - Complete Guide

**Date**: 2026-06-15

---

## Architecture Overview

```
[Players] → [VPS Forwarder (1 CPU/1GB)] → [Tailscale] → [Homelab Game Server (64GB)]
            Public IP                                      All traffic from 1 IP
            Multiple accounts                              Port-based differentiation
```

---

## Part 1: VPS Forwarder (Public-Facing)

### Apply Configuration

```bash
# SSH to your VPS
ssh user@your-vps-ip

# Copy content from connection-security-hardening.txt
sudo nano /etc/sysctl.d/99-game-server.conf
# Paste the entire content, save and exit (Ctrl+X, Y, Enter)

# Apply immediately
sudo sysctl -p /etc/sysctl.d/99-game-server.conf

# Set connection tracking hash buckets
echo "options nf_conntrack hashsize=24576" | sudo tee /etc/modprobe.d/nf_conntrack.conf

# Reload nf_conntrack module (or reboot if it fails)
sudo modprobe -r nf_conntrack 2>/dev/null
sudo modprobe nf_conntrack

# If module reload fails, reboot
sudo reboot
```

### Verify VPS

```bash
# After reboot, verify settings
sysctl net.ipv4.ip_forward                    # Should be 1
sysctl net.netfilter.nf_conntrack_max         # Should be 98304
cat /sys/module/nf_conntrack/parameters/hashsize  # Should be 24576

# IMPORTANT: If hashsize is wrong (e.g., shows 8192), fix it immediately:
# Current hashsize
cat /sys/module/nf_conntrack/parameters/hashsize

# If NOT 24576, fix it now:
echo 24576 | sudo tee /sys/module/nf_conntrack/parameters/hashsize

# Verify the fix
cat /sys/module/nf_conntrack/parameters/hashsize
# Should now show: 24576

# Check the ratio (should be ~4 connections per bucket)
echo "Connections per bucket: $(($(cat /proc/sys/net/netfilter/nf_conntrack_max) / $(cat /sys/module/nf_conntrack/parameters/hashsize)))"

# Check current connections
cat /proc/sys/net/netfilter/nf_conntrack_count

# Monitor in real-time
watch -n 1 'echo "Connections: $(cat /proc/sys/net/netfilter/nf_conntrack_count) / $(cat /proc/sys/net/netfilter/nf_conntrack_max)"'
```

**Why hashsize matters:**
- **hashsize=8192** (wrong): Each bucket handles ~12 connections = slow lookups, hash collisions
- **hashsize=24576** (correct): Each bucket handles ~4 connections = fast lookups, better performance
- Critical for multiple game accounts from same IP

---

## Part 2: Homelab Game Server (Backend)

### Apply Configuration

```bash
# On your homelab server
# Copy content from connection-security-hardening-homelab.txt
sudo nano /etc/sysctl.d/99-game-server-homelab.conf
# Paste the entire content, save and exit

# Apply immediately
sudo sysctl -p /etc/sysctl.d/99-game-server-homelab.conf

# Set connection tracking hash buckets (CRITICAL for same-IP scenario)
echo "options nf_conntrack hashsize=786432" | sudo tee /etc/modprobe.d/nf_conntrack.conf

# Reload module
sudo modprobe -r nf_conntrack 2>/dev/null
sudo modprobe nf_conntrack

# If module reload fails, reboot
sudo reboot
```

### Verify Homelab

```bash
# Verify critical settings
sysctl net.netfilter.nf_conntrack_max         # Should be 3145728 (3M)
cat /sys/module/nf_conntrack/parameters/hashsize  # Should be 786432
sysctl net.ipv4.udp_mem                       # Should show UDP memory limits
sysctl fs.file-max                            # Should be 8388608

# IMPORTANT: If hashsize is wrong, fix it immediately:
# Current hashsize
cat /sys/module/nf_conntrack/parameters/hashsize

# If NOT 786432, fix it now:
echo 786432 | sudo tee /sys/module/nf_conntrack/parameters/hashsize

# Verify the fix
cat /sys/module/nf_conntrack/parameters/hashsize
# Should now show: 786432

# Check the ratio (should be ~4 connections per bucket)
echo "Connections per bucket: $(($(cat /proc/sys/net/netfilter/nf_conntrack_max) / $(cat /sys/module/nf_conntrack/parameters/hashsize)))"

# Check connection tracking
cat /proc/sys/net/netfilter/nf_conntrack_count

# View connections by protocol
sudo cat /proc/net/nf_conntrack | awk '{print $3}' | sort | uniq -c | sort -rn
```

**Why hashsize matters for homelab (same-IP scenario):**
- With 3M connections from SAME SOURCE IP, large hash table is CRITICAL
- **hashsize=786432** ensures fast port-based lookup
- Small hashsize = severe performance degradation with multiple accounts

---

## Part 3: Increase File Limits (Prevent "Too Many Open Files" Error)

**Why**: Each game connection uses file descriptors. Multiple accounts = need higher limits.

### Quick Setup (Copy-paste this entire block)

```bash
# Create limits configuration
sudo tee -a /etc/security/limits.conf > /dev/null <<'EOF'

# File descriptor limits for game server
* soft nofile 1048576
* hard nofile 1048576
* soft nproc 65536
* hard nproc 65536
root soft nofile 1048576
root hard nofile 1048576
EOF

# Set systemd limits
sudo sed -i 's/#DefaultLimitNOFILE=/DefaultLimitNOFILE=1048576/' /etc/systemd/system.conf
echo "DefaultLimitNOFILE=1048576" | sudo tee -a /etc/systemd/system.conf

sudo sed -i 's/#DefaultLimitNOFILE=/DefaultLimitNOFILE=1048576/' /etc/systemd/user.conf
echo "DefaultLimitNOFILE=1048576" | sudo tee -a /etc/systemd/user.conf

# Reload systemd
sudo systemctl daemon-reexec

# Apply now (or logout/login)
ulimit -n 1048576

echo "File limits configured!"
```

### Verify

```bash
# Check current limit
ulimit -n
# Should show: 1048576

# If not, logout and login again, then check
```

**Done! Do this on both VPS and Homelab.**

---

## Part 4: Verify Both Systems Working Together

### On VPS (Monitor Forwarding)

```bash
# Install monitoring tools
sudo apt install -y iftop nethogs

# Watch Tailscale traffic
sudo iftop -i tailscale0

# Monitor connections
watch -n 2 'ss -s'

# Check UDP stats
netstat -su | grep -i "packet\|drop\|error"
```

### On Homelab (Monitor Game Server)

```bash
# Watch connection count (should see many from same IP)
watch -n 1 'cat /proc/sys/net/netfilter/nf_conntrack_count'

# View active connections
ss -tunapo | wc -l

# Check for same-IP connections
sudo netstat -ntu | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -5

# Monitor memory
free -h

# Watch UDP errors
watch -n 5 'netstat -su | grep -i drop'
```

---

## Part 5: Test with Game Accounts

### Start Testing

```bash
# 1. Start first game account - verify connection
# 2. Start second account - check connection count increases
# 3. Continue adding accounts while monitoring

# On homelab, watch connection growth:
watch -n 1 'cat /proc/sys/net/netfilter/nf_conntrack_count'

# Check all connections are from Tailscale IP:
sudo netstat -ntu | grep ESTABLISHED | head -20
```

### Expected Behavior

✅ **VPS**: Connection count rises as accounts connect  
✅ **Homelab**: All connections show same source IP (Tailscale VPS)  
✅ **Homelab**: Each account uses different source port  
✅ **No UDP drops** in `netstat -su`  
✅ **Connections persist** for hours without timeout  

---

## Troubleshooting

### Common Issue: Wrong Hashsize

**Problem**: `cat /sys/module/nf_conntrack/parameters/hashsize` shows wrong value (e.g., 8192 instead of 24576)

**Symptoms**:
- Slow connection lookups
- Performance degradation with multiple accounts
- High CPU usage in conntrack

**Fix immediately** (no reboot needed):
```bash
# Check current value
cat /sys/module/nf_conntrack/parameters/hashsize

# For VPS (should be 24576):
echo 24576 | sudo tee /sys/module/nf_conntrack/parameters/hashsize

# For Homelab (should be 786432):
echo 786432 | sudo tee /sys/module/nf_conntrack/parameters/hashsize

# Verify
cat /sys/module/nf_conntrack/parameters/hashsize

# Make permanent (already done earlier, but verify):
cat /etc/modprobe.d/nf_conntrack.conf
# Should contain: options nf_conntrack hashsize=24576 (VPS) or hashsize=786432 (Homelab)
```

**Why this happens**:
- Module loaded before `/etc/modprobe.d/nf_conntrack.conf` was created
- Need to either reload module or set manually

---

### VPS Issues

**Problem**: Connection tracking fills up quickly
```bash
# Increase limit temporarily
sudo sysctl -w net.netfilter.nf_conntrack_max=131072

# Make permanent
echo "net.netfilter.nf_conntrack_max = 131072" | sudo tee -a /etc/sysctl.d/99-game-server.conf
```

**Problem**: UDP packet drops
```bash
# Check drops
netstat -su | grep drop

# Increase UDP memory
sudo sysctl -w net.ipv4.udp_mem="8192 16384 32768"
```

### Homelab Issues

**Problem**: "Cannot track new connections"
```bash
# Check if limit reached
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# View what's using connections
sudo cat /proc/net/nf_conntrack | wc -l
```

**Problem**: Port exhaustion from same IP
```bash
# This shouldn't happen with current config, but check:
ss -s | grep timewait

# View TIME_WAIT count
ss -tan | grep TIME_WAIT | wc -l

# Should be under 2M (current limit)
```

**Problem**: Game disconnects after idle
```bash
# Check UDP timeout settings
sysctl net.netfilter.nf_conntrack_udp_timeout
sysctl net.netfilter.nf_conntrack_udp_timeout_stream

# Should be 3600 (1hr) and 43200 (12hr)
```

---

## Monitoring Scripts

### VPS Monitoring Script

```bash
sudo tee /usr/local/bin/vps-status.sh > /dev/null <<'SCRIPT'
#!/bin/bash
echo "=== VPS Forwarder Status $(date) ==="
echo ""
echo "Connection Tracking: $(cat /proc/sys/net/netfilter/nf_conntrack_count) / $(cat /proc/sys/net/netfilter/nf_conntrack_max)"
echo ""
echo "Memory:"
free -h
echo ""
echo "UDP Stats:"
netstat -su | grep -E "packet receive|packet send|receive buffer|send buffer|dropped"
echo ""
echo "Tailscale Status:"
sudo tailscale status | head -5
SCRIPT
chmod +x /usr/local/bin/vps-status.sh
```

### Homelab Monitoring Script

```bash
sudo tee /usr/local/bin/homelab-status.sh > /dev/null <<'SCRIPT'
#!/bin/bash
echo "=== Homelab Game Server Status $(date) ==="
echo ""
echo "Connection Tracking: $(cat /proc/sys/net/netfilter/nf_conntrack_count) / $(cat /proc/sys/net/netfilter/nf_conntrack_max)"
echo ""
echo "Active Connections:"
ss -s
echo ""
echo "Memory:"
free -h
echo ""
echo "UDP Stats:"
netstat -su | grep -E "packet receive|packet send|dropped|errors"
echo ""
echo "Top Connection Sources:"
sudo netstat -ntu | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -5
SCRIPT
chmod +x /usr/local/bin/homelab-status.sh
```

### Run Monitoring

```bash
# On VPS
/usr/local/bin/vps-status.sh

# On Homelab
/usr/local/bin/homelab-status.sh

# Schedule daily checks
(crontab -l 2>/dev/null; echo "0 */6 * * * /usr/local/bin/homelab-status.sh >> /var/log/game-server-status.log") | crontab -
```

---

## Persistence Check

### After Reboot Verification

```bash
# On both VPS and Homelab after reboot:

# Check sysctl loaded
sudo sysctl -p /etc/sysctl.d/99-game-server*.conf

# Verify conntrack buckets
cat /sys/module/nf_conntrack/parameters/hashsize

# Check file limits
ulimit -n
# Should be 1048576

# Verify forwarding (VPS only)
sysctl net.ipv4.ip_forward
# Should be 1
```

---

## Summary Checklist

### VPS Setup
- [ ] Applied `/etc/sysctl.d/99-game-server.conf`
- [ ] Set conntrack hashsize to 24576
- [ ] IPv4 forwarding enabled
- [ ] File descriptor limits set
- [ ] Verified after reboot

### Homelab Setup
- [ ] Applied `/etc/sysctl.d/99-game-server-homelab.conf`
- [ ] Set conntrack hashsize to 786432
- [ ] File descriptor limits set (8M)
- [ ] Loose rp_filter for Tailscale
- [ ] Verified after reboot

### Testing
- [ ] VPS forwards traffic to homelab
- [ ] Multiple game accounts connect
- [ ] All show same source IP on homelab
- [ ] No UDP packet drops
- [ ] Connections persist 24/7
- [ ] Monitoring scripts running

---

**Configuration is production-ready for 24/7 grinding with multiple accounts!**
