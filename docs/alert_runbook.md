# Alert Runbook

This document provides first-response actions for each alert in the observability system.

## Infrastructure Alerts

### HighCPUUsage
**Severity**: P2 | **Duration Threshold**: 5 minutes at 85%+

**What it means**:
- CPU is at 85-95% utilization consistently
- System is approaching capacity but still responsive
- Indicates heavy workload or inefficient process

**Immediate Actions**:
1. Check which process is consuming CPU:
   ```bash
   top -o %CPU
   # or
   ps aux --sort=-%cpu | head -10
   ```
2. Review application logs for errors or performance issues
3. Check if legitimate workload has increased (batch job, backup, etc.)
4. Monitor the next 5 minutes - if trend continues, escalate to P1

**Root Cause Investigation**:
- Inefficient database queries
- Memory pressure causing swapping
- Missing indexes in application
- Insufficient resource allocation
- Denial of service attack

**Resolution Actions**:
1. If process is runaway: `kill -9 <pid>` or restart service
2. If legitimate heavy load: scale horizontal or increase instance size
3. Optimize application code or queries
4. Implement rate limiting for external APIs

**Prevention**:
- Set up CPU usage trending alerts
- Implement autoscaling based on CPU thresholds
- Regular code profiling and optimization

---

### CriticalCPUUsage
**Severity**: P1 | **Duration Threshold**: 2 minutes at 95%+

**What it means**:
- CPU is at critical saturation (95%+)
- System is severely degraded, likely unresponsive
- Immediate intervention required

**Immediate Actions** (< 5 minutes):
1. **Emergency triage**:
   ```bash
   # Check system responsiveness
   uptime
   dmesg | tail -20  # Look for errors
   
   # List top CPU consumers
   ps aux --sort=-%cpu | head -5
   ```

2. **Identify the culprit**:
   - Is it the primary application?
   - Is it a legitimate batch job?
   - Is it an unexpected process?

3. **Emergency actions**:
   - If unknown process: `kill -9 <pid>`
   - If database: restart database service
   - If application: restart application service
   - If still critical: consider failover to standby

**Communication**:
- Notify ops team immediately
- Post incident in on-call Slack
- Prepare customer communication in case of impact

**Post-Resolution**:
- Review what caused the spike
- Implement safeguards to prevent recurrence
- Update capacity plans if needed

---

### HighMemoryUsage
**Severity**: P2 | **Duration Threshold**: 5 minutes at 80%+

**What it means**:
- System memory is 80-95% utilized
- Risk of OOM killer being triggered
- Swap may be actively used (slow)

**Immediate Actions**:
1. Check memory consumers:
   ```bash
   # By process
   ps aux --sort=-%mem | head -10
   
   # Memory distribution
   free -h
   vmstat 1 5
   
   # Check swap usage
   swapon -s
   ```

2. Check for memory leaks:
   ```bash
   # Java process
   jmap -histo <pid> | head -20
   
   # General inspection
   ps -o pid,vsz,rss | sort -k3 -nr | head -10
   ```

3. Look for stuck processes:
   ```bash
   ps aux | grep <process> | grep -v grep
   ```

**Root Cause Investigation**:
- Application memory leak
- Cache growing unbounded
- Too many open connections/file descriptors
- Misconfigured JVM heap/pagepool limits
- Database connection pool exhaustion

**Resolution Actions**:
1. If memory leak suspected: restart the service
2. Increase memory limit if hardware has available capacity
3. Implement memory limits in the application
4. Clear application caches if safe
5. Check application for configuration errors

**Prevention**:
- Regular memory leak testing
- Set max heap sizes appropriately
- Implement cache eviction policies
- Monitor swap usage trending

---

### CriticalMemoryUsage
**Severity**: P1 | **Duration Threshold**: 2 minutes at 95%+

**Immediate Actions** (Emergency response):
1. Assume OOM killer is about to activate
2. Identify and stop non-critical services
3. Restart affected application immediately
4. Consider failover if available
5. Do not delay - this will cause service crashes

**Communication**:
- Alert customer of potential disruption
- Engage on-call engineering immediately
- Prepare for emergency scaling

---

### HighDiskUsage / CriticalDiskUsage
**Severity**: P2 (80%) / P1 (95%) | **Duration Threshold**: 5 minutes

**What it means**:
- `/` partition is filling up
- At critical levels, services may fail to write logs
- Database may crash if unable to write data

**Immediate Actions**:
1. Identify what's consuming space:
   ```bash
   df -h
   du -sh /*  # top-level directories
   du -sh /var/log/*  # check logs
   
   # Find large files
   find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null
   
   # Check for deleted-but-held files
   lsof | grep deleted
   ```

2. Check for log spam:
   ```bash
   wc -l /var/log/* | sort -n | tail -10
   zgrep "ERROR" /var/log/application.log* | wc -l
   ```

3. Cleanup actions (if safe):
   ```bash
   # Rotate old logs
   log rotated /var/log/*
   
   # Clear old cache
   rm -rf /tmp/cache/*
   
   # Remove old backups
   find /backups -mtime +30 -delete
   ```

**Root Cause Investigation**:
- Application logging too verbosely
- Database transaction logs not cleaned up
- Full backups stored on same disk
- Unmonitored application writing large files
- Cache directory growing unbounded

**Resolution Actions**:
1. **Immediate**: Free space by:
   - Archiving old logs
   - Removing old backups
   - Clearing temporary files
   - Stopping debug logging

2. **Short-term**: Add disk space or mount new volume

3. **Long-term**: Implement log rotation, enable compression

**Prevention**:
- Implement log rotation (logrotate, fluent-bit)
- Monitor disk growth trends
- Set filesystem quotas per application
- Automate old file cleanup

---

## VPN Alerts

### VPNTunnelDown
**Severity**: P1 | **Duration**: 1 minute

**What it means**:
- VPN tunnel is not operational
- Market is disconnected from hub
- Data cannot flow between market and HQ
- This affects ALL applications in that market

**Immediate Actions** (< 2 minutes):
1. **Verify the alert**:
   ```bash
   # Check tunnel status from home gateway
   ipsec status
   ipsec list
   
   # Try tunnel ping
   ping -I <tunnel_ip> <remote_ip>
   
   # Check tunnel logs
   tail -50 /var/log/strongswan/charon.log
   ```

2. **Check network connectivity**:
   ```bash
   # Primary ISP connection
   ping 8.8.8.8
   traceroute <remote_gateway>
   
   # Check for ISP issues
   curl -I https://api.isp-status.example.com
   ```

3. **Check tunnel configuration**:
   ```bash
   # Verify PSK and peer configuration
   grep -A 5 "conn market-to-hub" /etc/ipsec.conf
   
   # Check if VPN service is running
   systemctl status strongswan
   ```

**Escalation Flowchart**:
```
VPN Down?
├─ ISP connection down?
│  └─ YES: Contact ISP support (Tier 1)
├─ Remote gateway unreachable?
│  └─ YES: Contact remote site (Tier 2)
├─ Local VPN service down?
│  └─ YES: Restart service (Tier 3)
├─ Configuration error?
│  └─ YES: Fix & reload ipsec (Tier 3)
└─ Other: Escalate to network engineer (Tier 4)
```

**Resolution Actions**:

Option 1: Local VPN service issue
```bash
# Restart VPN service
sudo systemctl restart strongswan

# Verify it comes up
sleep 5
ipsec status
```

Option 2: Configuration issue
```bash
# Reload configuration
sudo ipsec reload

# Or full restart if needed
sudo systemctl restart strongswan
```

Option 3: ISP/connectivity issue
- Contact ISP support
- Request status page check
- Provide ISP ticket number for escalation
- Consider failover to backup ISP if available

**Post-Resolution**:
- Document root cause in ticket
- Update primary contact list for ISP
- Review tunnel redundancy options

---

### VPNTunnelDegraded
**Severity**: P2 | **Duration**: 3 minutes

**What it means**:
- Tunnel is operational but showing degradation
- Either high latency (>200ms) OR packet loss (>2%)
- Applications using tunnel will experience slowness
- Performance SLA may be at risk

**Immediate Actions**:
1. **Quantify the degradation**:
   ```bash
   # Check tunnel metrics
   # From Prometheus query: vpn_tunnel_latency_ms{market="market-ke"}
   
   # Manual test
   ping -c 10 -I <tunnel_ip> <remote_ip>
   
   # Check packet loss
   ping -c 100 <remote_ip> | grep loss
   ```

2. **Identify cause**:
   - Network congestion on tunnel?
   - ISP line quality issues?
   - Remote site CPU high?
   - Local CPU high affecting VPN process?

3. **Immediate mitigations**:
   ```bash
   # Check CPU usage on gateway
   top
   
   # Check QoS settings
   tc class show dev eth0
   
   # Monitor ongoing
   watch -n 1 'ping -c 1 <remote>'
   ```

**Root Cause Investigation**:
- ISP network congestion (check with ISP)
- VPN encryption/decryption bottleneck
- Host CPU contention
- MTU mismatch causing packet fragmentation
- Competing traffic on link (BW saturation)

**Resolution Actions**:

For high latency:
1. Check if due to encryption: `ipsec status --verbose`
2. Move VPN to dedicated gateway if on app server
3. Enable hardware acceleration if available
4. Contact ISP about network path quality

For packet loss:
1. Check MTU setting: `ip link show`
2. Try increasing MTU if possible
3. Enable TCP MSS clamping
4. Contact ISP - may indicate line quality issues

**Prevention**:
- Implement SLA monitoring for each tunnel
- Alert on latency trends (not just absolute threshold)
- Regular ISP performance reviews
- Redundant tunnel for each market (failover)

---

### VPNHighLatency / VPNCriticalLatency
**Severity**: P2 (>150ms) / P1 (>300ms)

**What it means**:
- Tunnel latency is above acceptable SLA threshold
- Financial transactions will be slower
- Real-time operations impacted
- P1 = financial SLA violation

**Immediate Actions**:
1. Check ISP status - is this widespread?
2. Verify remote site is operational
3. Check local gateway CPU/memory
4. Perform traceroute to identify where latency occurs:
   ```bash
   traceroute -I <remote_ip>
   mtr -n -c 100 <remote_ip>
   ```

5. Notify application teams of latency
6. If P1: activate incident response

**Resolution Same as VPNTunnelDegraded above**

---

### VPNPacketLoss / VPNCriticalPacketLoss
**Severity**: P2 (1-5%) / P1 (>5%)

**What it means**:
- Network is dropping packets
- Applications using TCP will see retransmissions (latency)
- UDP applications (VoIP, video) will see degradation
- P1 = tunnel is practically unusable

**Immediate Actions** (Same as VPNTunnelDegraded):
1. Verify with manual ping test
2. Check ISP line quality
3. Notify ISP if persistent
4. Consider failover if packet loss >5%

---

### VPNBandwidthSaturation / VPNBandwidthExceeded
**Severity**: P2 (80% util) / P1 (95% util)

**What it means**:
- Tunnel is becoming congested
- New transactions may be queued/delayed
- Risk of timeout failures
- P1 = tunnel capacity exhausted

**Immediate Actions**:
1. Identify bulk traffic:
   ```bash
   # View tunnel traffic
   ifstat -i <tunnel_interface>
   
   # Check top talkers
   iftop -i <tunnel_interface>
   ```

2. Check for:
   - Backup/sync operations
   - Database replication
   - Large file transfers
   - Legitimate increase in transaction volume

3. If P1: temporarily throttle non-critical traffic or failover

**Resolution Actions**:
1. Contact ISP for bandwidth increase
2. Implement traffic prioritization (QoS)
3. Optimize application data transfer
4. Implement caching at market level to reduce inter-market traffic

---

## Security Alerts

### FailedAuthAttempts / CriticalFailedAuthAttempts
**Severity**: P2 (>5/sec) / P1 (>20/sec)

**What it means**:
- Excessive failed authentication attempts
- P2 = unusual but might be legitimate
- P1 = likely brute force attack or service misconfiguration

**Immediate Actions**:
1. Check application logs for patterns:
   ```bash
   # Recent auth failures
   grep "Failed.*auth" /var/log/application.log | tail -50
   
   # Count by user
   grep "Failed.*auth" /var/log/application.log | grep -o "user=[^ ]*" | sort | uniq -c | sort -rn
   
   # Count by source IP
   grep "Failed.*auth" /var/log/auth.log | grep -o "[0-9.]\+" | sort | uniq -c | sort -rn
   ```

2. Determine if attack or misconfiguration - ask:
   - Is a known user locked out?
   - Is application trying to connect with wrong credentials?
   - Are all attempts from external IP?
   - Are attempts for multiple users?

3. For P1 (suspected brute force):
   ```bash
   # Block source IP at firewall
   iptables -A INPUT -s <attacker_ip> -j DROP
   
   # Or use fail2ban
   fail2ban-client set sshd banip <attacker_ip>
   
   # Check if already banned
   fail2ban-client status sshd
   ```

**Root Cause Investigation**:
- **Attack**: External attacker attempting unauthorized access
- **Misconfiguration**: Application using wrong credentials
- **User Lockout**: Valid user entering password incorrectly
- **Credential Rotation**: Credentials weren't updated everywhere

**Resolution Actions**:
For attacks:
1. Activate firewall rules to block source IP
2. Enable MFA for critical accounts
3. Implement account lockout policies
4. Increase password complexity requirements

For misconfiguration:
1. Fix application credentials
2. Restart affected service
3. Test connectivity
4. Update credential documentation

**Prevention**:
- Implement account lockout after N failures
- Use centralized auth (LDAP, Okta)
- Enforce MFA for administrative accounts
- Regular access reviews

---

### FirewallBlocksIncreasing / CriticalFirewallBlocks
**Severity**: P2 (>10/sec) / P1 (>50/sec)

**What it means**:
- Firewall is blocking many connections
- P2 = investigate to understand traffic
- P1 = likely attack or major traffic anomaly

**Immediate Actions**:
1. Check which rules are triggering:
   ```bash
   # View firewall logs
   tail -100 /var/log/firewall.log | grep DROP
   
   # Count by rule
   grep DROP /var/log/firewall.log | awk '{print $5}' | sort | uniq -c | sort -rn
   ```

2. Identify blocked traffic pattern:
   - Source IPs?
   - Destination ports?
   - Protocol type?
   - Internal vs external?

3. Determine if legitimate or attack:
   ```bash
   # Trace source IPs
   whois <source_ip>
   geoiplookup <source_ip>
   
   # Check if internal
   ping <source_ip>
   ```

**Resolution Actions**:
- **If attack**: Block source IPs, enable DDoS protection
- **If scanning**: Block port scanners at edge
- **If misconfiguration**: Fix firewall rule, whitelist legitimate traffic
- **If legitimate**: Create exception or expand firewall rules

**Prevention**:
- Regularly review and audit firewall rules
- Implement geo-blocking if traffic is regional
- Enable rate limiting for public services
- Deploy IDS/IPS for attack detection

---

### PortScanDetected / AggresivePortScan
**Severity**: P2 / P1

**What it means**:
- Network scanning activity detected on infrastructure
- P2 = isolated scan, investigate
- P1 = aggressive/targeted scan, immediate response

**Immediate Actions**:
1. Identify source and target:
   ```bash
   # Who is scanning?
   grep "port scan" /var/log/security.log | tail -20
   
   # What ports are being targeted?
   ```

2. Determine if internal or external:
   - Internal: Could be security team, network tools, or breach
   - External: Reconnaissance, looking for exploitable services

3. For external scanning:
   ```bash
   # Block at firewall
   ufw delete allow from <source_ip>
   # or
   iptables -A INPUT -s <source_ip> -j DROP
   
   # Enable aggressive IDS rules
   # Contact security team about advanced threat
   ```

**Root Cause Investigation**:
- Routine security scan (check with security team)
- Unauthorized reconnaissance survey
- Internal network compromise (if internal IP)

**Resolution Actions**:
1. Block suspicious source IPs
2. Review open ports - close unnecessary ones
3. Configure port knocker for critical services
4. Implement intrusion detection
5. If internal: investigate for compromise

---

### SecurityAnomaly / CriticalSecurityAnomaly
**Severity**: P2 / P1 (for `privilege_escalation`, `data_exfiltration`, `lateral_movement`)

**What it means**:
- ML model detected unusual behavior
- P1 anomalies suggest active attack

**Immediate Actions**:
1. Understand what was detected:
   - Privilege escalation: User attempted to run as root/admin
   - Data exfiltration: Large data transfer to external location
   - Lateral movement: Unusual inter-host connections
   - Behavior anomaly: Action outside normal pattern

2. For P1:
   ```bash
   # Isolate potentially compromised system
   # Block network access
   
   # Capture forensics
   tar czf forensics.tar.gz /var/log /proc /sys
   
   # Do NOT shut down yet - preserve memory
   md5sum -r / > filesystem.hash  # baseline
   ```

3. Notify security team immediately
4. Prepare incident response

**Prevention**:
- Implement EDR (Endpoint Detection & Response)
- Monitor for privilege escalation patterns
- Alert on unusual file access
- Baseline normal behavior for all users

---

## Certificate Alerts

### CertificateExpiringWarning / CertificateExpiring / CertificateExpired
**Severity**: P3 (>7 days) / P2 (1-7 days) / P1 (expired)

**What it means**:
- SSL/TLS certificate is expiring or expired
- P1 (expired): Services using cert will fail
- P2/P3: Plan for renewal

**Immediate Actions**:
1. Identify affected service:
   ```bash
   # Find certificate
   find / -name "*.crt" -o -name "*.pem" | xargs openssl x509 -noout -dates
   
   # Check specific service
   echo | openssl s_client -servername example.com -connect example.com:443
   ```

2. Days remaining:
   ```bash
   openssl x509 -in /path/to/cert.crt -noout -text | grep -A 1 "Not After"
   ```

**For Expired Certificates (P1)**:
1. This is urgent - services may be failing
2. Obtain new certificate immediately:
   - Request from CA
   - Self-signed if emergency (not for production)
3. Deploy certificate:
   ```bash
   cp /path/to/new/cert.crt /etc/ssl/certs/
   systemctl restart <service>
   ```
4. Verify:
   ```bash
   # SSL check
   openssl s_client -connect <service>:443
   # Or curl
   curl https://<service>/ -v
   ```

**For Expiring Certificates (P2/P3)**:
1. Request new certificate from CA (2-3 days lead time)
2. Generate CSR if using new CA:
   ```bash
   openssl req -new -key private.key -out certificate.csr
   ```
3. Plan deployment during maintenance window
4. Deploy and verify before switchover
5. Monitor logs for certificate warnings

**Prevention**:
- Set up certificate renewal automation (certbot for Let's Encrypt)
- Monitor expiration dates 30+ days in advance
- Test renewal process regularly

---

## Response Template

For any alert, follow this template:

```
1. VERIFY
   - Is alert still firing?
   - Is it a false positive?
   - Can you reproduce the issue?

2. ASSESS IMPACT
   - Is customer-facing functionality affected?
   - What's the scope (one market? all markets?)?
   - How long has it been firing?

3. COMMUNICATE
   - Notify team in Slack
   - Update status page if needed
   - Prepare customer communication

4. INVESTIGATE
   - Check logs
   - Verify configuration
   - Check recent changes

5. RESOLVE
   - Apply fix
   - Verify resolution
   - Monitor for recurrence

6. DOCUMENT
   - Root cause analysis
   - Prevention measures
   - Ticket/ticket reference
```

## Emergency Escalation

If you cannot resolve an alert within the response time SLA:

1. **Escalate** to on-call manager
2. **Document** what you've tried
3. **Engage** specialists if needed:
   - VPN issues → Network Engineer
   - Security issues → Security Team
   - Database issues → DBA
   - Application issues → App Team Lead

**On-Call Contact**: See PagerDuty escalation policy

---

## References

- [Market Infrastructure Access Guide](../docs/infrastructure.md)
- [VPN Tunnel Diagnostics](../docs/vpn-diagnostics.md)
- [Security Response Procedures](../docs/security-response.md)
- [Firewall Rules Inventory](../docs/firewall-rules.md)
