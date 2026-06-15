# Apply Connection Security Hardening

**Target**: 1 CPU / 1GB RAM Ubuntu 24.04 VPS  
**Purpose**: Forward game traffic to homelab via Tailscale (24/7 multiple accounts)

---

## Apply Configuration

### 1. Copy and apply sysctl settings

```bash
# Copy the entire content from connection-security-hardening.txt to your VPS
# Then apply it:
sudo sysctl -p /etc/sysctl.d/99-game-server.conf
```

### 2. Set connection tracking buckets

```bash
# Create modprobe config
echo "options nf_conntrack hashsize=24576" | sudo tee /etc/modprobe.d/nf_conntrack.conf

# Reboot to apply
sudo reboot
```

---

## Verify

```bash
# Check if IPv4 forwarding is enabled
sysctl net.ipv4.ip_forward
# Should return: net.ipv4.ip_forward = 1

# Check connection tracking limit
sysctl net.netfilter.nf_conntrack_max
# Should return: net.netfilter.nf_conntrack_max = 98304

# Check current connections
cat /proc/sys/net/netfilter/nf_conntrack_count
```

---

## Monitor

```bash
# Watch connection usage
watch -n 1 'cat /proc/sys/net/netfilter/nf_conntrack_count'

# Check memory
free -h

# Check for UDP drops
netstat -su | grep drop
```

---

Done. Settings persist across reboots automatically.
