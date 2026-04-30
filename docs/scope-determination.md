# PCI DSS Scope Determination Document

## Document Information
| Field | Value |
|-------|-------|
| Organization | Bluebridge Solutions |
| Assessment Date | 4/15/2026 |
| Assessor | Callum McRae |
| Version | 1.0 |
| Status | Draft |
| Merchant Level | Level 2 (SAQ eligible — 1–6M annual transactions) |

Bluebridge Solutions is a mid-sized e-commerce platform provider processing approximately 2.4 million annual card transactions and currently pursuing PCI DSS v4.0 compliance.

---

## 1. Executive Summary

### Scope Statement

At Bluebridge Solutions, the Cardholder Data Environment (CDE) includes the web application servers, payment processing services, and associated databases hosted within Bluebridge Solutions' on-premise network environment that store, process, or transmit cardholder data. Supporting systems with direct or indirect connectivity to these components have been evaluated for inclusion within PCI DSS scope.

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
| Payment DB | Tokenized PAN | 30 Days | Required for transaction reconciliation. Tokenized values only — raw PAN is never written to this system. Token-to-PAN mapping is managed exclusively by the Tokenization Service . |
| Logging System | Partial PAN (masked) | 7 Days |Used by Bluebridge Solutions engineering team for debugging failed payment transactions |
| HSM Appliance | Encryption keys, transient PAN during processing | Transient only | PAN is processed in-memory during tokenization and never persisted to disk. |

### 2.2 Systems That PROCESS Cardholder Data

| System | Function | Data | Justification |
|--------|----------|------|---------------|
| Web App | Payment processing | PAN, CVV | Core business function |
| Payment service | Authorization | PAN | required for transactional approval |
| Tokenization Service | Converts PAN to tokens for storage and transmission | PAN (transient) | Core scope system — directly handles raw PAN during token generation. PAN is never persisted post-tokenization. |

### 2.3 Systems That TRANSMIT Cardholder Data

| System | Function | Data | Justification |
|--------|----------|------|---------------|
| Web Server (DMZ) | Receives PAN from customer browser, forwards to Payment App | PAN | TLS 1.2+ | First point of PAN receipt; transmits to Payment App within CDE |
|Web App → Payment Gateway | HTTPS | TLS 1.2+ | Secure transmission of card data |
| Payment App → Card Database | Internal CDE | PAN | Encrypted internal transmission; both endpoints within CDE scope |


---

## 3. Connected Systems Assessment

### 3.1 Direct Connections to CDE

| System | Connection Type | Business Purpose | In Scope? | Justification |
|--------|-----------------|------------------|-----------|---------------|
| App Server | Internal API | Payment Processing | Yes | Supports Bluebridge Solutions’ core payment processing workflow |
| Admin Workstation| SSH | Maintenance| Yes | Dedicated admin workstation with direct SSH access to CDE systems. Distinct from general corporate laptops — see Section 3.2 for VPN-based indirect access paths. |
| Jump Host | RDP/SSH | Admin access | Yes | Required for controlled administrative access to Bluebridge Solutions’ CDE systems |
| Log Collector | Syslog forwarding | Log aggregation to SIEM | Yes | Located within CDE boundary; outbound log forwarding to Management Zone creates a connection path that must be controlled and monitored |

### 3.2 Indirect Connections (Via Intermediary)

| System | Connection Path | Business Purpose | In Scope? | Justification |
|--------|-----------------|------------------|-----------|---------------|
| Corporate Laptop | VPN -> App Server | Admin access | Yes | Corporate workstation subnet can reach the CDE indirectly via VPN and jump host. The entire subnet must be treated as introducing indirect scope risk, not just individual laptops. Access controls and monitoring must apply at the subnet level. |
| External Payment Gateway | Internet-facing API | Payment authorisation | Yes — third party | Third-party service provider; must maintain evidence of their current PCI DSS compliance status |

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
| Jump Host (10.2.1.50) | Privileged access gateway | Full administrative access to all CDE systems | Yes | Single point of administrative entry to CDE; compromise bypasses all other access controls. Requires hardening per Requirement 8.6.1 |

---

## 5. Trust Relationship Analysis

### 5.1 Authentication Dependencies
| CDE System |Authenticate Source | Risk if compromised | Mitigation |
|--------|-----------------|------------------|-----------|
| App Server | Active Directory | Domain compromise = full access | MFA + Segmentation |
| Payment App | Active Directory | Domain compromise = full access | MFA + dedicated CDE service accounts |
| Card Database | Active Directory | Domain compromise = DB access | MFA + DB-specific service accounts with least privilege |
| HSM  | Local authentication (assumed) | Local credential compromise | HSM-specific access controls per FIPS 140-2 |
| Tokenization Service | Active Directory | Domain compromise = tokenization bypass | MFA + isolated service account |


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
| Firewall rules | Restricts inbound traffic | Weak | Broad Rules EV-001 |
| VLAN separation | Logical Separation | Weak | Shared Routing EV-001, EV-002 |
| Access controls | Role Based | Medium | Some over-permission EV-003, EV-006 |
| Monitoring | SIEM Alerts | Medium | Limited coverage EV-004|

### 6.2 Segmentation Gaps

| Gap ID | Description | Risk | Remediation |
|--------|-----------------|------------------|------------------|
| GAP-001 | Overly permissive firewall rules | High | Implement least-privilege firewall rules between all zones; remove broad permit rules and restrict traffic to required ports, protocols, and explicit source/destination pairs. |
| GAP-002 | Shared AD across environments | High | Implement Active Directory tiering or deploy a dedicated CDE-specific identity provider isolated from the corporate domain. At minimum, enforce strict trust boundaries, privileged access workstations for CDE admin accounts, and separate service accounts that cannot authenticate across environments. |
| GAP-003 | Lack of segmentation testing | Med | Conduct formal segmentation penetration testing at least annually and after any significant network change, per PCI DSS Requirement 11.4.5. Testing must attempt to traverse from out-of-scope networks into the CDE and validate that all identified access paths are blocked. |
| GAP-004 | Shared backup infrastructure | High | Dedicate separate backup infrastructure to CDE systems or enforce strict access controls ensuring backup agents and storage targets for CDE systems are inaccessible from non-CDE environments. Validate that backup data containing CHD is encrypted at rest and that access is restricted to authorised personnel only. |
| GAP-005 | Centralized logging zones | Med | Implement log filtering and redaction controls to prevent PAN or sensitive authentication data from being written to the centralised SIEM. Restrict access to CDE-sourced log data to authorised security personnel. Consider a dedicated log collector for CDE systems with one-way log forwarding to prevent reverse access paths. |
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
| Segmentation Validation | “How do you verify segmentation is effective?” | Segmentation testing is not currently performed | No segmentation penetration test reports on file. Gap acknowledged — see GAP-003 |


---

## 8. Recommendations

### 8.1 Immediate Actions (0-30 days)

1. Bluebridge Solutions should implement least-privilege firewall rule sets between the corporate, management, and CDE zones by removing any "allow any" or overly broad access rules and restricting traffic strictly to required ports, protocols, and source/destination systems. Per PCI DSS v4.0 Requirements 1.3.1 and 1.3.2

2. Identify and remove all unnecessary network and administrative access paths to the CDE, including direct access from corporate workstations, ensuring all access is routed through controlled and monitored entry points (e.g., hardened jump host). Per Requirement 7.2.1

3. Conduct an immediate review and reduction of administrative privileges across all systems with access to the CDE, enforcing role-based access control (RBAC), eliminating shared accounts, and requiring multi-factor authentication (MFA) for all privileged access. Per Requirements 8.4.2 and 7.2.2

### 8.2 Short-Term Actions (30-90 days)

1. Implement separation of identity infrastructure by decoupling Active Directory services between the corporate and CDE environments, or introducing strict tiering and trust boundaries, to prevent credential compromise in the corporate domain from directly impacting CDE systems. Per Requirement 8.6.1

2. Implement segmentation-aware centralized logging by restricting log ingestion into the SIEM to explicitly approved CDE sources, enforcing log filtering/redaction for sensitive data (e.g., PAN), and applying role-based access controls to ensure only authorized security personnel can access CDE-related logs. Validate logging configurations against PCI DSS requirements for data protection and retention. Per Requirements 10.3.2 and 10.5.1

3. Perform formal network segmentation testing to validate that CDE isolation controls are effective, including attempts to bypass firewall rules and access CDE systems from out-of-scope networks. Per Requirement 11.4.5


### 8.3 Long-Term Actions (90+ days)

1. Transition toward a Zero Trust security architecture by eliminating implicit trust between network zones, enforcing continuous identity verification, device posture checks, and least-privilege access for all interactions with the CDE, regardless of network location. Per Requirements 7.2.1 and 1.3.2

2. Redesign the network architecture to enforce true segmentation of the Cardholder Data Environment, including dedicated security zones, isolated identity and logging services, and removal of shared infrastructure dependencies that currently bridge corporate, management, and CDE environments. Incorporate segmentation testing and validation as a recurring control to ensure boundaries remain effective over time.


---

## 9. Residual Risk Statement

After implementing recommendations, the following risks remain:

| Risk | Likelihood | Impact | Acceptance Required? | Owner |
|--------|-----------------|------------------|-----------|-------|
| Shared services risk within the Bluebridge Solutions environment | Med | High | Yes | CISO |
| Insider Admin abuse | Low | High | Yes | CISO + HR |
| Residual risk from shared identity and logging infrastructure enabling lateral movement into CDE | Medium | High | Yes | CISO |
| Risk of administrative credential compromise via jump host or VPN pathways | Low–Medium | High | Yes | Security Operations Manager |

---

## Appendices

### A. Network Diagram
<pre> ```text
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
                               CDE  ``` </pre>


This network diagram illustrates the current segmentation design and highlights key trust relationships between the corporate, management, and CDE environments. Despite logical segmentation, shared services such as Active Directory, logging, and backup systems introduce indirect connectivity paths that weaken isolation of the CDE.                               

### B. Data Flow Diagram
<pre> ```text Customer Browser
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
                   Backup Infrastructure ``` </pre>

This diagram illustrates that cardholder data may flow into logging and backup systems, indicating that these systems must be considered in scope due to potential storage or exposure of sensitive data.

Cardholder data is transmitted to a third-party payment processor, introducing external dependency risk and requiring validation of secure transmission and contractual controls    

This creates implicit scope expansion, as systems receiving logs or backups from the CDE may also fall within PCI DSS scope.        

### C. Evidence Index

| Evidence ID | Description | Location |
|-------------|-------------|----------|
| EV-001 | Firewall rule sets between zones | Palo Alto config |
| EV-002 | Network architecture diagram | Internal documentation |
| EV-003 | Active Directory access controls | AD console |
| EV-004 | SIEM logging configuration | Splunk dashboard |
| EV-005 | Backup access permissions and retention configuration | Backup infrastructure configuration |
| EV-006 | VPN and jump host access logs | Access logs |

---

## Approval

| Role | Name | Date | Signature |
|------|------|------|-----------|
| Assessor | Callum McRae |  |  |
| Reviewer | Security Manager |  |  |
| Approver | CISO |  |  |
