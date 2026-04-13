# ACL-Based Network Segmentation PoC — Finn's Urgent Care

> **From Ransomware to Resilience: Modernizing Urgent Care IT**  
> Cybersecurity Capstone (CIS4891) — Miami Dade College, Spring 2026  
> Author: Ziggy Dolce

## Overview

This repository contains the Proof of Concept (PoC) deliverable for my cybersecurity capstone project. The PoC demonstrates how **Access Control Lists (ACLs)** enforce network segmentation to prevent lateral movement in a healthcare environment — directly addressing the flat-network vulnerability that enabled a simulated ransomware incident at Finn's Urgent Care.

The full project applies the **NIST Risk Management Framework (RMF)**, **NIST SP 800-30**, and **HIPAA Security Rule** to design, implement, and validate a modernized cybersecurity architecture for a mid-sized urgent care facility in Miami-Dade County.

## The Problem

Finn's Urgent Care operated on a **flat, unsegmented LAN** where clinical workstations, administrative PCs, medical IoT devices, and servers all shared the same network space. When ransomware compromised a single administrative endpoint, it propagated freely across the entire network — encrypting the EHR system, backup NAS, and medical devices simultaneously.

## The Solution

This PoC validates that **extended ACLs applied to inter-VLAN routing interfaces** can contain a breach to a single network segment, preventing an attacker from pivoting to high-value clinical assets.

### What the PoC Proves

| Scenario | Before ACLs | After ACLs |
|----------|-------------|------------|
| Attacker → EHR Server | ✅ Full access | ❌ Blocked |
| Attacker → Medical IoT | ✅ Full access | ❌ Blocked |
| Attacker → Clinical PCs | ✅ Full access | ❌ Blocked |
| Attacker → HTTP on EHR | ✅ Page loads | ❌ Request Timeout |
| Clinical PC → HTTP on EHR | ✅ Page loads | ✅ Page loads |

The last row is critical — **legitimate clinical access is preserved** while all unauthorized lateral movement is eliminated.

## Network Architecture

```
                    ┌──────────┐
                    │   R1     │  Cisco 2901
                    │ (Router) │  Inter-VLAN Routing + ACLs
                    └────┬─────┘
                        │ Trunk (802.1Q)
                    ┌────┴─────┐
                    │   ASW1   │  Cisco 2960
                    │ (Switch) │  VLAN Enforcement
                    └──┬─┬─┬─┬─┘
                       │ │ │ │
          ┌────────────┘ │ │ └────────────┐
          │              │ │              │
    ┌─────┴──────┐ ┌─────┴─┴────┐   ┌─────┴──────┐
    │ VLAN 10    │ │ VLAN 30    │   │ VLAN 40    │
    │ Clinical   │ │ Admin/HR   │   │ Server     │
    │ Clinical-PC│ │ Admin-PC   │   │ EHR-Server │
    └────────────┘ │ Attacker-PC│   └────────────┘
                   └────────────┘
          ┌────────────┐
          │ VLAN 20    │
          │ IoT/Medical│
          │ IoT-Pump   │
          └────────────┘
```

## VLAN & IP Addressing

| VLAN | Name | Subnet | Gateway | Hosts |
|------|------|--------|---------|-------|
| 10 | Clinical Network | 192.168.10.0/24 | 192.168.10.1 | Clinical-PC (.10) |
| 20 | IoT / Medical | 192.168.20.0/24 | 192.168.20.1 | IoT-Pump (.10) |
| 30 | Admin / HR | 192.168.30.0/24 | 192.168.30.1 | Admin-PC (.10), Attacker-PC (.50) |
| 40 | Server / Data | 192.168.40.0/24 | 192.168.40.1 | EHQ-Server (.10) |

## Repository Structure

```
├── README.md                         # This file
├── LESSONS_LEARNED.md                 # Troubleshooting journal and areas of learning
├── docs/
│   └── PoC_Report.docx                # Full PoC report with embedded screenshots
├── configs/
│   ├── R1_running-config.txt          # Router running configuration
│   └── ASW1_running-config.txt        # Switch running configuration
├── screenshots/
│   ├── Packet_Tracer_Topology_View.png
│   ├── VLAN_Verification.png
│   ├── Sub_Interface_Verification.png
│   ├── Before_ACL_Attacker_to_PCs.png
│   ├── Before_ACLs_Attacker_HTTP_to_EHR.png
│   ├── After_ACLs_Attacker_to_PCs.png
│   ├── After_ACLs_HTTP_to_EHR.png
│   ├── After_ACLs_Clinical_PC_to_EHR_Server.png
│   └── ACL_Hit_Counters.png
└── acl-rules/
    └── acl_config.txt                 # Standalone ACL configuration commands
```

## ACL Policy Summary

```cisco
! ADMIN_ACL — Block all lateral movement from admin segment
ip access-list extended ADMIN_ACL
 deny   ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny   ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
 deny   ip 192.168.30.0 0.0.0.255 192.168.40.0 0.0.0.255
 permit ip 192.168.30.0 0.0.0.255 any

! IOT_ACL — Full isolation of medical devices
ip access-list extended IOT_ACL
 deny   ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny   ip 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255
 deny   ip 192.168.20.0 0.0.0.255 192.168.40.0 0.0.0.255
 permit ip 192.168.20.0 0.0.0.255 any

! CLINICAL_ACL — Permit EHR access only, deny everything else
ip access-list extended CLINICAL_ACL
 permit tcp 192.168.10.0 0.0.0.255 host 192.168.40.10 eq 80
 permit tcp 192.168.10.0 0.0.0.255 host 192.168.40.10 eq 443
 deny   ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
 deny   ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
 deny   ip 192.168.10.0 0.0.0.255 192.168.40.0 0.0.0.255
 permit ip 192.168.10.0 0.0.0.255 any
```

## MITRE ATT&CK Mapping

| Technique | ID | How ACLs Mitigate |
|-----------|----|-------------------|
| Network Service Discovery | T1046 | Ping sweeps across VLANs are denied |
| Lateral Movement (Remote Services) | T1021 | Cross-VLAN service access is blocked |
| Exploitation of Remote Services | T1210 | Compromised hosts cannot reach other segments |
| Impact (Data Encrypted) | T1486 | Ransomware cannot propagate to EHR or backups |

## Frameworks & Standards Applied

- **NIST SP 800-30** — Risk assessment methodology (likelihood × impact scoring)
- **NIST RMF** — Control selection and implementation lifecycle
- **NIST CSF v2.0** — Protect and Detect function alignment
- **HIPAA Security Rule** — Technical safeguards for PHI protection
- **MITRE ATT&CK** — Threat modeling and attack simulation mapping
- **CIS Benchmarks** — Cisco IOS hardening baseline

## Tools Used

- **Cisco Packet Tracer** — Network simulation and ACL testing
- **Cisco IOS CLI** — Router and switch configuration

## Author

**Ziggy Dolce** — Infrastructure & Security Architecture Lead  
Miami Dade College, CIS4891 Cybersecurity Capstone, Spring 2026

## License

This project was completed as part of an academic capstone for CIS4891 at Miami Dade College. Shared for educational and portfolio purposes.
