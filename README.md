# Project 01: Reviewing Network Segmentation for PCI DSS

## Overview

PCI DSS is one of the fastest ways to expose whether someone understands real-world GRC or just compliance theatre. This project requires you to treat network segmentation as a **risk claim that must be defended**, not a diagram exercise.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SEGMENTATION: COMPLIANCE VS REALITY                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   COMPLIANCE THEATRE                    REAL GRC WORK                       │
│   ──────────────────                    ─────────────                       │
│   Draw a box around CDE                 Define what ACTUALLY is CDE         │
│   Label it "segmented"                  Examine trust relationships         │
│   Move on                               Question implicit trust             │
│                                         Document where design fails         │
│                                         Anticipate QSA challenges           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## The Scenario

**Company:** MidTech Retail Solutions
**Industry:** E-commerce platform provider
**Annual Card Transactions:** ~2.4 million
**Current State:** Pursuing PCI DSS v4.0 compliance

The company processes credit card payments through their web application and has implemented what they believe is network segmentation. Your role is to assess whether this segmentation genuinely reduces PCI scope or creates a false sense of security.

### Current Network Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           INTERNET                                          │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PERIMETER FIREWALL                                  │
│                         (Palo Alto PA-850)                                  │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                ┌───────────────┴───────────────┐
                │                               │
                ▼                               ▼
┌───────────────────────────┐   ┌───────────────────────────────────────────┐
│        DMZ ZONE           │   │              CORPORATE ZONE               │
│      (10.1.0.0/24)        │   │             (10.2.0.0/16)                 │
├───────────────────────────┤   ├───────────────────────────────────────────┤
│                           │   │                                           │
│  ┌─────────────────────┐  │   │  ┌─────────────────┐  ┌────────────────┐  │
│  │   Web Servers (3)   │  │   │  │  AD Domain      │  │  File Servers  │  │
│  │   10.1.0.10-12      │  │   │  │  Controllers    │  │  10.2.1.20-22  │  │
│  └─────────────────────┘  │   │  │  10.2.1.10-11   │  └────────────────┘  │
│                           │   │  └─────────────────┘                      │
│  ┌─────────────────────┐  │   │                                           │
│  │   Load Balancer     │  │   │  ┌─────────────────┐  ┌────────────────┐  │
│  │   10.1.0.5          │  │   │  │  Jump Host      │  │  Workstations  │  │
│  └─────────────────────┘  │   │  │  10.2.1.50      │  │  10.2.2.0/24   │  │
│                           │   │  └─────────────────┘  └────────────────┘  │
└───────────────────────────┘   └───────────────────────────────────────────┘
                │
                │ (Claimed Segmentation Boundary)
                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CARDHOLDER DATA ENVIRONMENT (CDE)                        │
│                           (10.3.0.0/24)                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐  │
│  │   Payment App       │  │   Card Database     │  │   HSM Appliance     │  │
│  │   Server            │  │   Server            │  │                     │  │
│  │   10.3.0.10         │  │   10.3.0.20         │  │   10.3.0.30         │  │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘  │
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐                           │
│  │   Tokenization      │  │   Log Collector     │                           │
│  │   Service           │  │   (Splunk Forwarder)│                           │
│  │   10.3.0.40         │  │   10.3.0.50         │                           │
│  └─────────────────────┘  └─────────────────────┘                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MANAGEMENT ZONE                                     │
│                          (10.4.0.0/24)                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐  │
│  │   SIEM (Splunk)     │  │   Backup Server     │  │   Patch Management  │  │
│  │   10.4.0.10         │  │   10.4.0.20         │  │   (WSUS)            │  │
│  │                     │  │                     │  │   10.4.0.30         │  │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Existing Documentation Claims

The company's current documentation states:
1. "The CDE is fully segmented from the corporate network"
2. "Only authorized systems can communicate with the CDE"
3. "Administrative access requires going through the jump host"
4. "All CDE traffic is logged and monitored"

---

## What You Need to Do

### Phase 1: Define the True CDE Boundary

**Task:** Identify what actually constitutes the CDE, not what people wish it were.

**Questions to Answer:**
- Which systems store, process, or transmit cardholder data?
- Which systems are "connected to" systems that handle CHD?
- What about systems that could impact the security of the CDE?
- Where does cardholder data actually flow?

**Deliverable:** A data flow diagram showing actual CHD movement

```
EXAMPLE DATA FLOW ANALYSIS:

Customer Browser
       │
       ▼ (HTTPS - PAN entered)
┌──────────────┐
│  Web Server  │ ─────────────────────────────────────────┐
└──────────────┘                                          │
       │                                                  │
       ▼ (API Call - PAN in transit)                      │
┌──────────────┐                                          │
│ Payment App  │                                          │
└──────────────┘                                          │
       │                                                  │
       ├──────────────────────┐                           │
       │                      │                           │
       ▼                      ▼                           ▼
┌──────────────┐    ┌──────────────┐            ┌──────────────┐
│     HSM      │    │   Card DB    │            │  Log Server  │
│ (Encryption) │    │  (Storage)   │            │  (Contains   │
└──────────────┘    └──────────────┘            │   PAN logs?) │
                                                └──────────────┘

QUESTION: Is the log server in scope? What data does it receive?
```

### Phase 2: Examine Trust Relationships

**Task:** Map all trust relationships that cross the segmentation boundary.

**Areas to Investigate:**

| Connection Type | Question | Risk Level |
|----------------|----------|------------|
| Authentication | Does CDE use corporate AD? | HIGH |
| DNS | Where do CDE systems resolve DNS? | MEDIUM |
| Time Sync | NTP source for CDE systems? | LOW |
| Patching | How are CDE systems patched? | HIGH |
| Backup | Where do backups go? Do they contain CHD? | HIGH |
| Logging | Where do logs flow? What's in them? | MEDIUM |
| Administration | Who has admin access? From where? | HIGH |

**Deliverable:** Trust relationship matrix with risk assessment

### Phase 3: Identify Segmentation Weaknesses

**Task:** Document where the segmentation design doesn't hold up.

**Common Weaknesses to Look For:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COMMON SEGMENTATION FAILURES                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. SHARED SERVICES                                                         │
│     ┌─────────┐         ┌─────────┐                                        │
│     │Corporate│ ───────▶│   AD    │◀─────── ┌─────────┐                    │
│     │   Zone  │         │  Server │         │   CDE   │                    │
│     └─────────┘         └─────────┘         └─────────┘                    │
│                              │                                              │
│     If AD is compromised, attacker has credentials for CDE                  │
│                                                                             │
│  2. JUMP HOST AS SINGLE POINT                                               │
│     ┌─────────┐         ┌─────────┐         ┌─────────┐                    │
│     │  Admin  │ ───────▶│  Jump   │────────▶│   CDE   │                    │
│     │Workstatn│         │  Host   │         │ Systems │                    │
│     └─────────┘         └─────────┘         └─────────┘                    │
│                              │                                              │
│     Compromise jump host = access to entire CDE                             │
│                                                                             │
│  3. LOG AGGREGATION                                                         │
│     ┌─────────┐                             ┌─────────┐                    │
│     │   CDE   │ ───── logs (with PAN?) ────▶│  SIEM   │                    │
│     │ Systems │                             │(Outside │                    │
│     └─────────┘                             │  CDE)   │                    │
│                                             └─────────┘                    │
│     If logs contain CHD, SIEM is in scope                                   │
│                                                                             │
│  4. BACKUP PATHS                                                            │
│     ┌─────────┐         ┌─────────┐         ┌─────────┐                    │
│     │   CDE   │ ───────▶│ Backup  │────────▶│  Tape   │                    │
│     │   DB    │         │ Server  │         │ Storage │                    │
│     └─────────┘         └─────────┘         └─────────┘                    │
│                                                                             │
│     Backups contain CHD - entire backup chain is in scope                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Deliverable:** Segmentation gap analysis with specific findings

### Phase 4: Anticipate QSA Challenges

**Task:** Document what a Qualified Security Assessor would likely challenge.

**Template for Each Finding:**

```markdown
## Finding: [Title]

**Current State:**
[What exists today]

**Segmentation Claim:**
[What documentation says]

**Reality:**
[What actually happens]

**QSA Challenge:**
[What question the QSA will ask]

**Evidence Required:**
[What you'd need to prove segmentation]

**Recommendation:**
[How to address the gap]
```

### Phase 5: Document Scope Determination

**Task:** Create a final scope determination that would survive audit.

**Deliverable Structure:**

```
SCOPE DETERMINATION DOCUMENT

1. Executive Summary
   - Scope statement
   - Key findings
   - Risk rating

2. CDE Definition
   - Systems that store CHD
   - Systems that process CHD
   - Systems that transmit CHD

3. Connected Systems
   - Systems with direct connectivity
   - Justification for inclusion/exclusion

4. Security-Impacting Systems
   - Systems that could affect CDE security
   - Justification for inclusion/exclusion

5. Segmentation Assessment
   - Controls in place
   - Gaps identified
   - Residual risk

6. Recommendations
   - Immediate actions
   - Long-term improvements
```

---

## Success Criteria

Your project demonstrates strong GRC judgment if:

| Criteria | Evidence |
|----------|----------|
| **Defended exclusions** | You explicitly state what's OUT of scope and why |
| **Identified hidden scope** | You found systems others missed (logs, backups, shared services) |
| **Challenged assumptions** | You questioned "segmented" claims with specific technical evidence |
| **Anticipated auditor questions** | Each finding includes likely QSA challenge |
| **Provided actionable recommendations** | Not just "fix segmentation" but specific steps |
| **Documented residual risk** | You acknowledge what gaps remain even after improvements |

### Red Flags (Weak Project)

- Diagram only with no analysis
- Accepting documentation claims at face value
- No discussion of trust relationships
- Missing backup and logging paths
- Generic recommendations without specifics

---

## Deliverables Checklist

```
□ Completed data flow diagram (CHD movement)
□ Trust relationship matrix
□ Segmentation gap analysis (minimum 5 findings)
□ QSA challenge preparation document
□ Scope determination document
□ 5-minute video walkthrough explaining your reasoning
```

---

## Resources

- [PCI DSS v4.0 Requirements](https://www.pcisecuritystandards.org/)
- [PCI SSC Guidance on Scoping](https://www.pcisecuritystandards.org/document_library)
- [Network Segmentation Testing Guidance](https://www.pcisecuritystandards.org/document_library)

---

## Reflection Questions

After completing this project, answer:

1. What assumption in the original documentation was most dangerous?
2. If you were the QSA, what would be your first question?
3. What's the single change that would most reduce scope?
4. Where did you have to make a judgment call without clear guidance?
5. What residual risk remains even with perfect implementation of your recommendations?

---

*Remember: Compliance is not about diagrams. It's about whether your argument would still stand when things go wrong.*
