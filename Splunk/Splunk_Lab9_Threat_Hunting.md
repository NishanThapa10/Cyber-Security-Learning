# Blue Team Threat Hunting: Incident Response Lifecycle using Splunk (BOTS v1)

## 👤 Author Profile
- **Degree:** Master of Information Technology (Cybersecurity)
- **Unit Focus:** Identity Access Management (IAM) and Security Operations (SecOps)
- **Lab Framework:** Lockheed Martin Cyber Kill Chain Mapping
- **Target Threat Actor Group:** PO1S0N1VY

---

## 🎯 Executive Summary & Investigation Objective
As a Security Operations Center (SOC) Analyst investigating an active incident targeting **Wayne Enterprises** (`imreallynotbatman.com`), I utilized **Splunk Enterprise** to ingest, parse, and analyze multi-layered log sources. The objective was to reconstruct the timeline of an Advanced Persistent Threat (APT) attack lifecycle, identify compromise indicators (IoCs), and trace how the corporate web infrastructure (`192.168.250.70`) was successfully breached.

Throughout this live environment deployment, I successfully tracked an adversary's footprints across four distinct phases of the **Cyber Kill Chain**.

---

## 🛠️ Technical Environment & Data Architecture
To establish comprehensive network visibility, I deployed and utilized several Splunk Technical Add-ons (TAs) to normalize incoming telemetry against the **Common Information Model (CIM)**:
* **`sourcetype=stream:http`** – Captured layer-7 application wire data (HTTP methods, POST headers, payloads).
* **`sourcetype=stream:dns`** – Intercepted outbound network domain lookups and server resolutions.
* **`sourcetype=suricata`** – Provided signature-based alert context from our network Network Intrusion Detection System (IDS).
* **`sourcetype=*Sysmon*`** – Delivered host-based operating system telemetry (Windows Event Code 1: Process Creation).

---

## 🕵️‍♂️ Investigation Breakdown & Splunk Search Queries

### Phase 1: Reconnaissance (Information Gathering)
* **Objective:** Identify unauthorized network scanning activity targeting corporate web infrastructure.
* **SPL Query Executed:**
  ```spl
  index=botsv1 imreallynotbatman.com sourcetype=stream* 
  | stats count(src_ip) as Requests by src_ip 
  | sort - Requests
  ```
* **Analysis & Findings:** I isolated a massive traffic spike originating from an external IP address: **`40.80.148.42`**, which was responsible for over 91% of baseline corporate web traffic. Pivoting to `sourcetype=suricata`, this IP repeatedly triggered alerts for Cross-Site Scripting (XSS), XML External Entity (XXE) injections, and Blind SQL Injection attempts. Inspecting the `src_headers` unmasked the exact scanning framework: **Acunetix Web Vulnerability Scanner (v10.0)**.

---

### Phase 2: Weaponization (Adversary Infrastructure Setup)
* **Objective:** Investigate stealthy secondary assets linked to the threat actor cluster.
* **Methodology (SIEM to OSINT Pivot):** A secondary external IP, **`23.22.63.114`**, appeared silently in firewall logs. Because internal telemetry lacked context, I conducted an Open Source Threat Intelligence (OSINT) pivot to LevelBlue Open Threat Exchange (OTX).
* **Analysis & Findings:** Passive DNS records exposed a targeted cluster of malicious domains utilizing **typosquatting** specifically designed to mimic our organization (e.g., `wayncorpinc.com`, `wanecorpinc.com`). Historical WHOIS data revealed the registration email fingerprint: **`lillian.rose@po1s0n1vy.com`**, providing definitive attribution to the threat group **PO1S0N1VY**.

---

### Phase 3: Exploitation (Gaining Access via Brute Force)
* **Objective:** Determine if the adversary successfully bypassed authentication filters.
* **SPL Query Executed:**
  ```spl
  index=botsv1 sourcetype=stream:http form_data=*username*passwd* dest_ip=192.168.250.70 src_ip=23.22.63.114 
  | rex field=form_data "passwd=(?<userpassword>\w+)" 
  | stats count by userpassword 
  | sort - count
  ```
* **Analysis & Findings:** The query revealed an active **Automated Dictionary Brute Force Attack** targeting the default administrative login console (`/joomla/administrator/index.php`) with 412 rapid, sequential password variations. 

* **The Infrastructure Hand-off:** Cross-correlating the broader server access timeline revealed that once the brute-force script on `23.22.63.114` hit a match, the human adversary switched connections back to their main reconnaissance box (`40.80.148.42`) to log in using the compromised corporate credentials: **`username=admin` / `password=batman`**.

---

### Phase 4: Installation (Persistent Malware Foothold)
* **Objective:** Track post-exploitation software deployment and extract binary signatures.
* **SPL Queries Executed:**
  ```spl
  index=botsv1 sourcetype=stream:http dest_ip="192.168.250.70" *.exe
  index=botsv1 "3791.exe"
  ```
* **Analysis & Findings:** Inspecting network protocol fields (`part_filename{}`), I discovered that immediately following administrative entry, the attacker uploaded a malicious web shell script named **`agent.php`** for persistent backdoor shell access, along with a Windows binary executable named **`3791.exe`**. 

Pivoting to host-based kernel security events, I targeted **EventCode=1 (Process Creation)** to extract the file's immutable cryptographic blueprint fingerprint:
* **Malware SHA-256 Hash DNA:** `ec78c938d8453739ca2a370b9c275971ec46caf6e479de2b2d04e97cc47fa45d`

---

### Phase 5: Command & Control (C2 Outbound Beaconing)
* **Objective:** Intercept the malware's remote control network communication infrastructure.
* **SPL Query Executed:**
  ```spl
  index=botsv1 sourcetype=stream:dns 23.22.63.114 
  | stats values("name{}")
  ```
* **Analysis & Findings:** By auditing network wire-level DNS lookups resolving back to the threat actor's infrastructure, I captured the precise external remote access command-and-control beacon domain used by the active malware layer to phone home: **`://jumpingcrab.com`**.

---

## 🎓 Master's Level Core Technical Competencies Learned
1. **SIEM Optimization Best Practice:** Learned that filtering searches via `index=` and strict `sourcetype` boundaries is a required operational strategy to minimize index cluster workload and eliminate alert fatigue.
2. **Wire Data vs. Log Data Reliability:** Understood that while host logs can be corrupted or deleted during a server compromise, passive network wire captures (`stream:http`/`stream:dns`) remain unalterable, proving indispensable for forensic verification.
3. **Adversary Infrastructure Decoupling:** Learned to identify sophisticated threat patterns where actors separate automated high-volume scan blocks from quieter, targeted manual deployment boxes to trick signature-based firewalls.
4. **Dynamic Data Regular Expressions:** Mastered on-the-fly parsing of unstructured web payloads using inline regex mapping (`| rex field=...`) to dynamically extract hidden login metadata and parameters.
