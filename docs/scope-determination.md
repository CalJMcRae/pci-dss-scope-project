# PCI DSS Scope Determination Document

## Document Information
| Field | Value |
|-------|-------|
| Organization | [Bluebridge Solutions] |
| Assessment Date | [4/15/2026] |
| Assessor | Callum McRae |
| Version | 1.0 |
| Status | Draft |

Bluebridge Solutions is a mid-sized e-commerce platform provider processing approximately 2.4 million annual card transactions and currently pursuing PCI DSS v4.0 compliance.

---

## 1. Executive Summary

### Scope Statement

At Bluebridge Solutions, the Cardholder Data Environment (CDE) includes the web application servers, payment processing services, and associated databases hosted in AWS that store, process, or transmit cardholder data. Supporting systems with direct or indirect connectivity to these components have been evaluated for inclusion within PCI DSS scope.

 Due to Bluebridge Solutions’ reliance on shared services (Active Directory, logging, backup, and patching), along with indirect access paths from the corporate and management zones, network segmentation at Bluebridge Solutions cannot be relied upon to reduce PCI DSS scope. Multiple trusted pathways and shared services create effective connectivity between in-scope and out-of-scope systems. As a result, these systems must be considered in scope for PCI DSS.

### Key Findings

- Segmentation between corporate network and CDE is partially implemented but not fully restrictive
- Shared services (Active Directory, logging) create potential indirect scope expansion
- Firewall rules allow broader access than required for business operations
- Segmentation controls cannot be relied upon to reduce PCI DSS scope due to multiple trusted pathways into the CDE

### Overall Risk Rating
[ ] Low | [ ] Medium | [X] High | [ ] Critical

---

## 2. Cardholder Data Environment Definition

### 2.1 Systems That STORE Cardholder Data

| System Name | Data Types Stored | Retention | Justification |
|--------|-----------------|------------------|-----------|
| Payment DB | Tokenized PAN | 30 Days | Required for Bluebridge Solutions’ transaction reconciliation processes |
| Logging System | Partial PAN (masked) | 7 Days |Used by Bluebridge Solutions engineering team for debugging failed payment transactions |

### 2.2 Systems That PROCESS Cardholder Data

| System | Function | Data | Justification |
|--------|----------|------|---------------|
| Web App | Payment processing | PAN, CVV | Core business function |
| Payment service | Authorization | PAN | required for transactional approval |

### 2.3 Systems That TRANSMIT Cardholder Data

| System | Function | Data | Justification |
|--------|----------|------|---------------|
|Web App → Payment Gateway | HTTPS | TLS 1.2+ | Secure transmission of card data |


---

## 3. Connected Systems Assessment

### 3.1 Direct Connections to CDE

| System | Connection Type | Business Purpose | In Scope? | Justification |
|--------|-----------------|------------------|-----------|---------------|
| App Server | Internal API | Payment Processing | Yes | Supports Bluebridge Solutions’ core payment processing workflow |
| Admin Workstation| SSH | Maintenance| Yes | Direct admin access |
| Jump Host | RDP/SSH | Admin access | Yes | Required for controlled administrative access to Bluebridge Solutions’ CDE systems |

### 3.2 Indirect Connections (Via Intermediary)

| System | Connection Path | Business Purpose | In Scope? | Justification |
|--------|-----------------|------------------|-----------|---------------|
| Corporate Laptop | VPN -> App Server | Admin access | Yes | Can reach CDE directly |

User workstations within the corporate network are considered out of scope for direct PCI DSS assessment, as they do not store, process, or transmit card holder data. However due to their ability to access the CDE via VPN and administrative pathways, they are treated as connected systems and introduce indirect risk that must be controlled through access restrictions and monitoring.

---

## 4. Security-Impacting Systems

### 4.1 Systems That Could Impact CDE Security

| System | Security Function | Impact if Compromised | In Scope? | Justification |
|--------|-----------------|------------------|-----------|---------------|
| Active Directory | Authentication | Full compromise of CDE | Yes | Centralized identity provider used across Bluebridge Solutions environments |
| SIEM | Logging | Missed breaches | Yes | Detection fail risk |
| Backup Infrastructure | Data protection | Exposure of CHD | Yes | Stores backups of Bluebridge Solutions’ CDE systems, potentially including cardholder data |
| Patch server | System updates | System compromise risk | Yes | Has privileged access |

---

## 5. Trust Relationship Analysis

### 5.1 Authentication Dependencies
| CDE System |Authenticate Source | Risk if compromised | Mitigation |
|--------|-----------------|------------------|-----------|
| App Server | Active Directory | Domain compromise = full access | MFA + Segmentation |


### 5.2 Shared Services

| Service | Used By CDE | Used By Non-CDE | Segmentation Effective? |
|---------|-------------|-----------------|------------------------|
| DNS | Yes | Yes | No |
| NTP | Yes | Yes | No |
| AD | Yes | Yes | No |
| Backup | Yes | Yes | No |
| Logging | Yes | Yes | No |
| Patching | Yes | Yes | No |

---

## 6. Segmentation Assessment

### 6.1 Segmentation Controls

| Control | Description | Effectiveness | Evidence |
|--------|-----------------|------------------|-----------|
| Firewall rules | Restricts inbound traffic | Weak | Broad Rules |
| VLAN separation | Logical Separation | Weak | Shared Routing |
| Access controls | Role Based | Medium | Some over-permission |
| Monitoring | SIEM Alerts | Medium | Limited coverage |

### 6.2 Segmentation Gaps

| Gap ID | Description | Risk | Remediation |
|--------|-----------------|------------------|------------------|
| GAP-001 | Overly permissive firewall rules | High | Restrict access |
| GAP-002 | Shared AD across environments | High | Separate domains |
| GAP-003 | Lack of segmentation testing | Med | Perform validation |
| GAP-004 | Shared backup infrastructure | High | Isolate backup systems or restrict access |
| GAP-005 | Centralized logging zones | Med | Separate logging or sanitize logs |
| GAP-006 | Centralized jump host provides administrative access to entire CDE | High | Harden jump host, restrict access, implement session recording and isolation |
| GAP-007 | Corporate endpoints can access CDE via VPN and jump host | High | Restrict VPN access, enforce device posture checks, isolate admin network |
| GAP-008 | Logging system may ingest masked or partial PAN data | Medium–High | Validate log contents and implement strict redaction controls| 

Based on the identified segmentation gaps and shared service dependencies within Bluebridge Solutions’ environment, the current network design does not provide effective isolation of the Cardholder Data Environment (CDE).

Multiple indirect access paths and trust relationships exist between the corporate, management, and CDE zones, increasing the risk of lateral movement.

As a result, segmentation cannot be relied upon to reduce PCI DSS scope in its current state.

---

## 7. QSA Challenge Preparation

### 7.1 Anticipated Questions

| Area | Likely Question | Your Response | Evidence |
|--------|-----------------|------------------|-----------|
| Shared Services | "How is AD segmented?" | It is not fully segmented | Network architecture diagrams showing AD placement across zones, firewall rule sets permitting authentication traffic between corporate and CDE systems, and AD configuration details demonstrating shared domain usage. |
| Admin Access | Who can access CDE? | Limited but overly broad | VPN access logs, jump host session logs, and system access control lists (ACLs) demonstrating which users and systems can initiate administrative connections to CDE systems, including authentication methods and privilege levels. |
| Logging | “Is the logging infrastructure segmented between CDE and non-CDE systems?” | Bluebridge Solutions utilizes a centralized logging architecture that ingests data from both CDE and non-CDE systems. While access controls exist, the lack of segmentation means a compromise of the logging platform could provide visibility into or indirect access to CDE systems. | Network diagram, SIEM architecture  |
| Backups | “How are backup systems segmented to prevent access to CDE data?” | Bluebridge Solutions’ backup infrastructure services both CDE and non-CDE systems and requires elevated privileges across environments. Due to the lack of strict segmentation, compromise of the backup system could result in unauthorized access to cardholder data or supporting systems. | Backup architecture diagram, access control policies |
| Segmentation Validation | “How do you verify segmentation is effective?” | Segmentation testing is not currently performed | Penetration test reports, segmentation test results (missing) |


---

## 8. Recommendations

### 8.1 Immediate Actions (0-30 days)

1. Bluebridge Solutions should implement least-privilege firewall rule sets between the corporate, management, and CDE zones by removing any "allow any" or overly broad access rules and restricting traffic strictly to required ports, protocols, and source/destination systems.

2. Identify and remove all unnecessary network and administrative access paths to the CDE, including direct access from corporate workstations, ensuring all access is routed through controlled and monitored entry points (e.g., hardened jump host).

3. Conduct an immediate review and reduction of administrative privileges across all systems with access to the CDE, enforcing role-based access control (RBAC), eliminating shared accounts, and requiring multi-factor authentication (MFA) for all privileged access.

### 8.2 Short-Term Actions (30-90 days)

1. Implement separation of identity infrastructure by decoupling Active Directory services between the corporate and CDE environments, or introducing strict tiering and trust boundaries, to prevent credential compromise in the corporate domain from directly impacting CDE systems.

2. Implement segmentation-aware centralized logging by restricting log ingestion into the SIEM to explicitly approved CDE sources, enforcing log filtering/redaction for sensitive data (e.g., PAN), and applying role-based access controls to ensure only authorized security personnel can access CDE-related logs. Validate logging configurations against PCI DSS requirements for data protection and retention.

3. Perform formal network segmentation testing to validate that CDE isolation controls are effective, including attempts to bypass firewall rules and access CDE systems from out-of-scope networks.


### 8.3 Long-Term Actions (90+ days)

1. Transition toward a Zero Trust security architecture by eliminating implicit trust between network zones, enforcing continuous identity verification, device posture checks, and least-privilege access for all interactions with the CDE, regardless of network location.

2. Redesign the network architecture to enforce true segmentation of the Cardholder Data Environment, including dedicated security zones, isolated identity and logging services, and removal of shared infrastructure dependencies that currently bridge corporate, management, and CDE environments. Incorporate segmentation testing and validation as a recurring control to ensure boundaries remain effective over time.


---

## 9. Residual Risk Statement

After implementing recommendations, the following risks remain:

| Risk | Likelihood | Impact | Acceptance Required? |
|--------|-----------------|------------------|-----------|
| Shared services risk within the Bluebridge Solutions environment | Med | High | Yes |
| Insider Admin abuse | Low | High | Yes |
| Residual risk from shared identity and logging infrastructure enabling lateral movement into CDE | Medium | High | Yes |
| Risk of administrative credential compromise via jump host or VPN pathways | Low–Medium | High | Yes |

---

## Appendices

### A. Network Diagram

┌─────────────────────────────────────────────────────────────────────────────┐
│                                 INTERNET                                    │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
                    ┌───────────────────────────┐
                    │   Perimeter Firewall      │
                    │   (Palo Alto PA-850)     │
                    └─────────────┬────────────┘
                                  │
          ┌───────────────────────┴────────────────────────┐
          │                                                │
          ▼                                                ▼
┌───────────────────────────┐              ┌──────────────────────────────────┐
│          DMZ ZONE         │              │        CORPORATE ZONE            │
│       (10.1.0.0/24)       │              │        (10.2.0.0/16)             │
├───────────────────────────┤              ├──────────────────────────────────┤
│  Web Servers (10.1.0.10-12)│              │  AD Domain Controllers          │
│  Load Balancer (10.1.0.5)  │              │  (10.2.1.10-11)                 │
│                            │              │                                 │
│  (Receives PAN via HTTPS)  │              │  Corporate Workstations         │
└──────────────┬────────────┘              │  (10.2.2.0/24)                  │
               │                           │                                 │
               │ (API Calls)               │  Jump Host (10.2.1.50)          │
               ▼                           │  (Admin Access Path)            │
┌─────────────────────────────────────────────────────────────────────────────┐
│                 CARDHOLDER DATA ENVIRONMENT (CDE)                           │
│                        (10.3.0.0/24)                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Payment App Server (10.3.0.10)                                             │
│        │                                                                    │
│        ├───────────────┬────────────────────────────┐                        │
│        ▼               ▼                            ▼                        │
│  HSM Appliance   Card Database              Log Collector                   │
│  (10.3.0.30)     (10.3.0.20)                (10.3.0.50)                      │
│  (Encryption)    (Stores CHD)               (Forwards logs)                 │
│                                                                             │
│  Tokenization Service (10.3.0.40)                                           │
│                                                                             │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                │ (Logs / Backups / Patching Traffic)
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MANAGEMENT ZONE                                      │
│                         (10.4.0.0/24)                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SIEM (Splunk)       Backup Infrastructure        Patch Server (WSUS)               │
│  (10.4.0.10)          (10.4.0.20)          (10.4.0.30)                       │
│                                                                             │
│  (Shared services supporting both CDE and corporate systems)                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

                                ▲
                                │
                    ┌───────────┴────────────┐
                    │   VPN Access           │
                    │ (Corporate Users)      │
                    └───────────┬────────────┘
                                │
                                ▼
                         Corporate Laptop
                               │
                               ▼
                           Jump Host
                               │
                               ▼
                               CDE


This network diagram illustrates the current segmentation design and highlights key trust relationships between the corporate, management, and CDE environments. Despite logical segmentation, shared services such as Active Directory, logging, and backup systems introduce indirect connectivity paths that weaken isolation of the CDE.                               

### B. Data Flow Diagram
Customer Browser
      │
      ▼ (HTTPS - PAN entry)
Web Server (DMZ)
      │
      ▼ (API Call - PAN in transit)
Payment App (CDE)
      │
      ▼
External Payment Gateway (3rd Party)
      ├───────────────┬───────────────┐
      ▼               ▼               ▼
   HSM           Card Database     Log Collector
(encryption)     (storage)        (Potential masked PAN exposure)

                         │
                         ▼
                      SIEM (Mgmt Zone)

                         │
                         ▼
                   Backup Infrastructure

This diagram illustrates that cardholder data may flow into logging and backup systems, indicating that these systems must be considered in scope due to potential storage or exposure of sensitive data.

Cardholder data is transmitted to a third-party payment processor, introducing external dependency risk and requiring validation of secure transmission and contractual controls    

This creates implicit scope expansion, as systems receiving logs or backups from the CDE may also fall within PCI DSS scope.        

### C. Evidence Index
| Evidence ID | Description | Location |
|-------------|-------------|----------|
EV-001 | Firewall rule sets between zones | Palo Alto config
EV-002 | Network architecture diagram | Internal documentation
EV-003 | Active Directory access controls | AD console
EV-004 | SIEM logging configuration | Splunk dashboard
EV-005 | Backup access permissions and retention configuration | Backup infrastructure configuration
EV-006 | VPN and jump host access logs | Access logs

---

## Approval

| Role | Name | Date | Signature |
|------|------|------|-----------|
| Assessor | Callum McRae |  |  |
| Reviewer | Security Manager |  |  |
| Approver | CISO |  |  |
