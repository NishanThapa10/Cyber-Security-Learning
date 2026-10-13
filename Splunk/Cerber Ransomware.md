# Incident Response Playbook: Investigating Host-Based Cerber Ransomware via Splunk

## 👤 Analyst Profile
* **Objective:** Host-Based Endpoint and Network Wire Forensic Reconstruction
* **Attack Lifecycle Framework:** Cyber Kill Chain (Delivery, Installation, C2)
* **Malware Strain Isolated:** Cerber Ransomware-as-a-Service (RaaS)

---

## 🚨 Case Study Context & Scenario
As a Blue Team Cybersecurity Analyst, I was tasked by the Chief Information Security Officer (CISO) to investigate a "Critical" severity incident ticket raised on **August 24th, 2016**. 

An internal employee (Bob Smith), utilizing a Windows 10 enterprise workstation named **`we8105desk`**, returned to his desk to find his files completely inaccessible, his wallpaper changed, and audio extortion blaring from his speakers. Preliminary human triage revealed Bob had discovered an unknown USB drive in the corporate parking lot and inserted it into his machine earlier that day.

---

## 🔍 Step-by-Step Forensic Timeline & SPL Query Manifest

### Q1: Identify the Local and Edge IP Addresses of the Compromised Host
* **Objective:** Determine the physical internal IP addresses assigned to Bob's workstation (`we8105desk`) during the breach timeline.
* **SPL Hunting Query (Endpoint Extraction):**
  ```spl
  index=botsv1 host=we8105desk
  ```
  *Analysis Workflow:* Instead of relying on empty statistics columns, checking the left-hand UI pane under `INTERESTING FIELDS` -> `Source_Address` revealed the host-internal workstation IP: **`192.168.2.50`**.
* **SPL Hunting Query (Network Edge Interface Mapping):**
  ```spl
  index=botsv1 "we8105desk" (sourcetype="XmlWinEventLog*" OR sourcetype="stream*")
  | stats values(src_ip) as "Network Edge IPs" values(Source_Address) as "Local Endpoint IPs" by host
  ```
* **Verified Answer:** 
  * Local Host Interface IP: **`192.168.2.50`**
  * Network Edge/Production Route IP: **`192.168.250.100`**

---

### Q2: Isolate the Physical Identity of the Attack Vector (USB Key)
* **Objective:** Extract the plain-text brand name or friendly identifier of the parking lot USB device from the Windows Registry hive telemetry.
* **SPL Hunting Query:**
  ```spl
  index=botsv1 sourcetype=winregistry friendlyname 
  | table host object data
  ```
* **Verified Answer:** **`MIRANDA_PRI`**
* **Forensic Significance:** Windows kernel structures immediately populate the `FriendlyName` property string inside `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Enum\USB` upon a physical connection event, establishing a permanent hardware fingerprint even if the physical drive is later unplugged.

---

### Q3, Q4 & Q5: Network Intrusion Detection Sieve & Policy Auditing
* **Objective:** Identify the specific IDS network alerts triggered by the Trojan, assess defense performance accuracy, and isolate the lowest frequency signature.
* **SPL Hunting Query:**
  ```spl
  index="botsv1" sourcetype="suricata" *cerber*
  ```
* **Verified Answers:**
  * **Malicious Activity Signatures:** Suricata captured three core alarms: `Cerber Onion Domain Lookup`, `Cerber Checkin 2`, and `Cerber Checkin Error ICMP Response`.
  * **Lowest Frequency Alert (Q5):** **`ETPRO TROJAN Ransomware/Cerber Checkin 2`** (Triggered exactly **1** time / 20% of sample cluster).
  * **Defense Performance Policy Audit (Q4):** Auditing the `alert.action` field returned a value of **`allowed`**. This indicates a critical defensive misconfiguration; the IDS operated purely in passive listening mode, allowing the ransomware outbound check-in communication to traverse the perimeter unchallenged.

---

### Q6 & Q7: Unmasking Initial Delivery and Ransom Redirection Domains
* **Objective:** Isolate the very first suspicious site hit (Delivery) and the final destination Fully Qualified Domain Name (FQDN) used for extortion (C2).
* **SPL Hunting Query (Advanced Multi-Value Array Filtering):**
  ```spl
  index="botsv1" sourcetype="stream:dns" src_ip="192.168.250.100" "query_type{}"=A 
  | where NOT (LIKE('query{}', "%microsoft%") OR LIKE('query{}', "%waynecorpinc%") OR LIKE('query{}', "%bing%") OR LIKE('query{}', "%windows%") OR LIKE('query{}', "%msftncsi%"))
  | table _time, "query{}", src_ip, dest_ip
  | sort _time
  ```
* **Verified Answers:**
  * **Question 7 Answer (First Exploit Domain):** **`solidaritedeproximite.org`** (Contacted chronologically first at `02:48:12 AM` during the initial access phase).
  * **Question 6 Answer (Ransom FQDN Redirect):** **`cerberhhyed5frqa.xmfir0.win`** (Resolved at `03:15:12 AM`, directing the victim to the Tor payment portal post-encryption).

---

### Q8: Identify the Crypto Code Payload Delivery Filename
* **Objective:** Extract the precise filename of the code payload pulled across the network cable.
* **SPL Hunting Query:**
  ```spl
  index=botsv1 sourcetype="stream:http" "solidaritedeproximite.org"
  | table _time, uri, http_method, status
  ```
* **Verified Answer:** **`mhtm.jpg`**
* **Forensic Significance:** Although the HTTP protocol status returned a `404` or `blocked` signal, this reflects advanced Exploit Kit architecture. Web gates often return false error strings to automated security crawlers or non-vulnerable engines to obfuscate the real landing folder paths traveling underneath.

---

### Q9: Technical Analysis of Adversarial Camouflage (Steganography)
* **Objective:** Classify the specific anti-forensic or obfuscation strategy used by the core encryptor file.
* **Verified Answer:** **Steganography**
* **Theoretical Breakdown:** Threat actors purposefully injected code layers directly into the binary layout of a standard photograph format (`.jpg`) to completely bypass traditional signature-based proxy scanners. Upon landing on the endpoint host, a local script component programmatically decoded the target pixels, unpacked the underlying compiled code, and executed the destructive ransomware loop dynamically inside host processing spaces.
