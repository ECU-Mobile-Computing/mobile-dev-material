

---

# Graduate Research Assignment  
##  **Designing a Multi‑User, Occasionally‑Connected Sync Architecture for Enterprise Mobile Applications**


## **Assignment Overview**

Enterprise mobile applications used by distributed teams often operate in environments with **intermittent connectivity**. These systems must reliably:

- Capture changes made while offline  
- Sync those changes when connectivity returns  
- Download changes made by other users  
- Prevent data corruption  
- Maintain auditability for compliance (e.g., financial institutions)

Your task is to **research, design, and justify** a complete sync strategy suitable for a multi‑user .NET MAUI application backed by a central database and web services.

Your final submission must answer **all 11 research questions** below and propose a coherent architecture that addresses correctness, consistency, conflict resolution, and auditability.

---

##  **Research Questions (Required)**

Your paper must address each of the following in depth, using academic sources, industry patterns, and architectural reasoning.

---

### **1. How do you keep track of changes made on a device that need to be synced?**

Discuss options such as:

- Local write‑ahead logs  
- Change‑tracking tables  
- Versioned records  
- Timestamps vs. vector clocks  
- Tombstones for deletes  
- Domain‑level change units (aggregate roots, entity deltas)

Explain tradeoffs and justify your chosen approach.

---

### **2. How do you push changes to the central server? What data format and metadata are required?**

Address:

- Payload structure (JSON, protobuf, delta packets, full entities)  
- Required metadata (record ID, version, timestamps, device ID, user ID, operation type)  
- Transport considerations (REST, gRPC, message queues)  
- Security (auth, integrity, replay protection)

---

### **3. How do you pull down changes from other devices to your device?**

Explain:

- Incremental sync vs. full sync  
- Change feeds, event logs, or “last‑sync timestamp” queries  
- Ordering guarantees  
- Pagination and batching  
- How the device applies incoming changes to its local store

---

### **4. Does it matter whether you push or pull first?**

Analyze:

- Push‑first vs. pull‑first semantics  
- Scenarios where ordering prevents conflicts  
- How ordering affects correctness, latency, and user experience  
- Whether ordering should be configurable per domain

---

### **5. How do you handle conflicts when two devices modified the same data?**

Discuss at least three conflict resolution strategies:

- Last‑writer‑wins  
- Merge‑by‑field or merge‑by‑intent  
- Domain‑specific rules  
- Server‑authoritative resolution  
- Human‑in‑the‑loop resolution (rare but valid)

Provide examples and justify your chosen strategy.

---

### **6. Which timestamp should have higher priority?**  
Evaluate the implications of prioritizing:

- **Device modification time**  
- **Server receipt time**  
- **Server processing time (last processed wins)**  

Discuss clock drift, fairness, correctness, and auditability.

---

### **7. How do you recover from failed transactions (e.g., lost connection mid‑sync)?**

Explain:

- Idempotency  
- Retry strategies  
- Partial‑sync detection  
- Rollback vs. resume  
- Local transaction logs  
- Server‑side deduplication

---

### **8. Should the sync process be automatic when a connection is available or manually initiated?**

Compare:

- Automatic background sync  
- User‑initiated sync  
- Hybrid models (recommended in many enterprise apps)

Discuss UX, battery impact, network cost, and correctness.

---

### **9. What are the potential concerns for an automatic sync process?**

Examples:

- Excessive battery usage  
- High data consumption  
- Sync storms  
- Conflicts triggered without user awareness  
- Race conditions with user edits  
- Server overload

---

### **10. What are the potential concerns for a manually initiated sync process?**

Examples:

- Users forgetting to sync  
- Stale data  
- Conflicts accumulating  
- Operational risk in financial or regulated environments  
- Human error

---

### **11. Assume the mobile application is used by a publicly traded company or financial institution requiring an audit trail. How would you implement the required audit trail?**

Discuss:

- Immutable event logs  
- Append‑only audit tables  
- Per‑record change history  
- Digital signatures or hash chains  
- Regulatory compliance (SOX, PCI, FINRA)  
- How audit data is synced and stored  
- How auditors retrieve and verify data

---

## **Required Output Artifacts**

Your final submission must include:

### **A. Research Paper (10–15 pages)**  
Must include:

- Literature review of sync strategies  
- Comparison of industry systems (Firebase, Realm, Cosmos DB, DynamoDB Streams, Kafka, etc.)  
- Your proposed architecture  
- Justification for each design decision  
- Answers to all 11 research questions  
- Citations (must note the citations used in your paper)

---

### **B. Architecture Diagram Set**

Include:

- Device‑side architecture (local DB, change tracker, sync agent)  
- Server‑side architecture (API, event log, conflict resolver)  
- Sync sequence diagrams (push, pull, conflict resolution, retry)  
- Audit trail flow diagram

---

### **C. Prototype Demonstration**

A minimal .NET MAUI app demonstrating:

- Local offline store  
- A mock sync service  
- Push/pull simulation  
- Conflict resolution example  
- Logging/audit example

This does **not** need to be production‑ready—just enough to illustrate your design.

---

## **6‑Week Timeline Guide**

### **Week 1:** Topic familiarization, literature review  
### **Week 2:** Deep dive into sync patterns and conflict resolution  
### **Week 3:** Draft architecture + answer questions 1–6  
### **Week 4:** Answer questions 7–11 + audit trail design  
### **Week 5:** Build diagrams + optional prototype  
### **Week 6:** Final paper submission + presentation

---

## **Presentation Requirements (15 minutes)**

Your presentation must include:

- Summary of your sync strategy  
- Key architectural decisions  
- Conflict resolution model  
- Audit trail design  
- Risks and limitations  
- Recorded demo 

---


## **Graduate Research Assignment Rubric**  
### *Designing a Multi‑User, Occasionally‑Connected Sync Architecture for Enterprise Mobile Applications*  
**Total Points: 100**

---

### **1. Research Depth & Literature Review — 15 points**

**Excellent (13–15):**  
Demonstrates comprehensive understanding of offline‑sync systems; integrates academic sources, industry patterns, and modern distributed systems research. Shows clear synthesis rather than simple reporting.

**Competent (10–12):**  
Covers major concepts with adequate sources; demonstrates understanding but lacks deeper synthesis or misses emerging patterns.

**Developing (7–9):**  
Superficial coverage; relies on limited or outdated sources; lacks clear connection to enterprise sync problems.

**Insufficient (0–6):**  
Minimal research; missing citations; demonstrates little understanding of the domain.

---

### **2. Technical Accuracy & Completeness of the 11 Required Questions — 20 points**

**Excellent (18-20):**  
All 11 questions answered thoroughly, accurately, and with strong reasoning. Demonstrates mastery of sync mechanics, conflict resolution, metadata design, and auditability.

**Competent (14-17):**  
Answers all questions with generally correct reasoning; minor gaps or oversimplifications.

**Developing (6-13):**  
Several questions answered incompletely or inaccurately; reasoning lacks depth or clarity.

**Insufficient (0–5):**  
Major omissions; incorrect or unsupported claims; fails to address core sync concepts.

---

### **3. Architecture Design Quality — 20 points**

**Excellent (18–20):**  
Architecture is coherent, technically sound, and clearly justified. Includes device‑side, server‑side, and sync‑flow considerations. Demonstrates understanding of distributed systems constraints.

**Competent (15–17):**  
Architecture is mostly sound but may lack justification or omit minor components.

**Developing (10–14):**  
Architecture contains conceptual gaps, unclear flows, or unjustified decisions.

**Insufficient (0–9):**  
Architecture is incomplete, incorrect, or not aligned with offline‑first requirements.

---

### **4. Diagrams & Visual Communication — 10 points**

**Excellent (9–10):**  
Diagrams are clear, technically accurate, and enhance understanding. Includes sequence diagrams, data flow, and audit trail flow.

**Competent (7–8):**  
Diagrams are present and understandable but may lack detail or precision.

**Developing (5–6):**  
Diagrams are unclear, incomplete, or poorly labeled.

**Insufficient (0–4):**  
Missing diagrams or diagrams that do not reflect the described architecture.

---

### **5. Conflict Resolution & Timestamp Strategy Justification — 10 points**

**Excellent (9–10):**  
Provides a nuanced, domain‑aware conflict strategy with strong justification. Evaluates timestamp priority with awareness of clock drift, fairness, and audit requirements.

**Competent (7–8):**  
Provides a reasonable strategy with some justification; minor gaps.

**Developing (5–6):**  
Strategy is simplistic or poorly justified.

**Insufficient (0–4):**  
Missing or incorrect conflict resolution strategy.

---

### **6. Audit Trail Design — 10 points**

**Excellent (9–10):**  
Audit trail design is complete, compliant, and technically sound. Addresses immutability, append‑only logs, regulatory constraints, and sync implications.

**Competent (7–8):**  
Audit trail design is correct but lacks depth or regulatory awareness.

**Developing (5–6):**  
Audit trail is partially correct but misses key compliance or technical details.

**Insufficient (0–4):**  
Audit trail design is missing or incorrect.

---

### **7. Writing Quality, Organization, and Professionalism — 5 points**

**Excellent (5):**  
Paper is well‑organized, clear, and professional. Uses correct terminology and flows logically.

**Competent (4):**  
Generally clear with minor organizational issues.

**Developing (3):**  
Writing is understandable but disorganized or inconsistent.

**Insufficient (0–2):**  
Poorly written, unclear, or unprofessional.

---

### **8. Prototype Demonstration**

**Excellent (8-10):**  
Prototype demonstrates offline store, push/pull simulation, conflict resolution, and logging.

**Competent (5-7):**  
Prototype demonstrates partial functionality.

**Developing (1-4):**  
Prototype is minimal or incomplete.

**None (0):**  
No prototype submitted.

---

# ✔️ **Total: 100 points**



