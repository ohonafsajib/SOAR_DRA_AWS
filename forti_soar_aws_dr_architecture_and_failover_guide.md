# FortiSOAR Disaster Recovery Architecture on AWS (Using AWS DRS)

## 1. Purpose of This Document
This document explains **how Disaster Recovery (DR) works for FortiSOAR deployed on AWS EC2 using AWS Elastic Disaster Recovery (DRS)**. It clarifies what AWS DRS does automatically, what must be configured manually, and how integrations (FortiGate, FortiAnalyzer, APIs, log sources) continue to function seamlessly after a failover.

This document is intended for **engineering, SRE, and security teams** and can be exported and saved as a DR reference.

---

## 2. Current Architecture Overview

- Two AWS Regions:
  - **DC (Primary)** – East
  - **DR (Secondary)** – West
- Single-node **FortiSOAR** (no clustering)
- OS: **RHEL on EC2**
- Storage:
  - Multiple **EBS volumes**
  - **LVM (PV / VG / LV)** for data directories
- Key data paths:
  - `/var/lib/pgsql`
  - `/var/lib/elasticsearch`
  - `/var/lib/rabbitmq`
  - `/opt`
  - `/var/log`

FortiSOAR is a **stateful workload**, so both compute and storage recovery are required.

---

## 3. What AWS Elastic Disaster Recovery (DRS) Does

AWS DRS provides **continuous block-level replication** from the source EC2 (DC) to the DR region.

### Key Capabilities
- Replicates **entire EBS disks** (block-level)
- Preserves:
  - LVM layout (PVs, VGs, LVs)
  - Filesystems
  - Application data
- Does **not** keep a running EC2 in DR
- Creates the DR EC2 **only during failover**

DRS supports:
- Very low RPO (seconds)
- RTO in minutes

---

## 4. How AWS DRS Works (Step-by-Step)

### 4.1 Source Region (DC)
1. AWS DRS Agent is installed on the FortiSOAR EC2
2. Agent continuously tracks disk block changes
3. Data is securely replicated to the DR region

### 4.2 DR Region (Before Failover)
- No active EC2 instance
- No running FortiSOAR
- Replicated EBS data stored in staging resources

### 4.3 During Failover
1. Failover is initiated (manual or automated)
2. AWS DRS:
   - Creates a new EC2 instance
   - Creates EBS volumes from replicated data
   - Attaches volumes to the EC2
3. Instance boots
4. Linux automatically detects LVM
5. Filesystems mount using `/etc/fstab`
6. FortiSOAR services start

Result: **Same server state, same data, same mount points**

---

## 5. What AWS DRS Does NOT Do Automatically

AWS DRS does **not**:
- Preserve public or private IP addresses across regions
- Update DNS records
- Reconfigure security groups automatically
- Update integrations or external systems
- Validate application-level consistency

These must be handled by design and process.

---

## 6. IP Addressing and DNS Strategy

### Public IPs
- Public IPs are **region-scoped**
- Same IP cannot be reused across regions

### Recommended Solution: DNS-Based Access

Use a DNS hostname:
```
fortisoar.company.com
```

- All integrations and users connect via DNS
- DNS points to DC during normal operation
- DNS is updated to DR during failover
- DNS TTL should be **≤ 60 seconds**

This enables seamless failover without reconfiguration.

---

## 7. Integration Continuity After Failover

### Works Seamlessly If:
- Integrations use **DNS hostname**
- TLS certificates are hostname-based
- No IP-based allowlists are enforced

### Will Break If:
- IP addresses are hardcoded
- Integrations trust a specific IP
- Firewalls allow only DC IPs

### Affected Integrations
- FortiGate log forwarding
- FortiAnalyzer
- FortiDeceptor
- API-based integrations

---

## 8. Security Groups and Networking

### Security Groups
- Must be **pre-created in DR region**
- Must match DC rules exactly
- Required ports must be open (example):
  - 443 (HTTPS)
  - FortiSOAR ingestion ports

AWS DRS attaches security groups during recovery but does not design them.

### VPC and Subnets
- DR VPC must exist
- Subnets must be mapped during DRS configuration
- CIDR ranges should be compatible with DC

---

## 9. Storage and LVM Behavior

- All FortiSOAR data is on EBS
- LVM metadata is stored on disk
- AWS DRS replicates disk blocks

Therefore:
```
PV → VG → LV → Filesystem → Data
```

is restored automatically without manual intervention.

---

## 10. Failover Runbook (High-Level)

### During Failover
1. Initiate AWS DRS recovery
2. EC2 instance is created in DR
3. Attach Elastic IP (if used)
4. Update DNS to point to DR
5. Validate mounts and services
6. Verify FortiSOAR functionality

### Post-Failover Validation
- Logs received from FortiGate
- API integrations responding
- PostgreSQL healthy
- Elasticsearch healthy

---

## 11. Key Takeaways

- AWS DRS **automates server and data recovery**, not networking identity
- DNS-based access is mandatory for seamless DR
- Security groups and VPC must be prepared in advance
- No manual LVM recreation is required
- DR must be tested periodically

---

## 12. Final Statement

AWS Elastic Disaster Recovery ensures FortiSOAR can be recovered quickly with identical data and storage layout. Seamless operation after failover depends on DNS-based access, consistent networking, and proper integration design.

