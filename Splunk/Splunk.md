# Splunk Learning 
 
## 🛠️ What I Actually Did Today
1. **Installed Splunk Enterprise** on my local Windows machine.
2. **Ingested Data:** Uploaded a historical web sample file called `tutorialdata.zip` which contains real server web logs (`access_combined_wcookie`) and database logs (`vendor_sales`) for an online game store called Buttercup Games.
3. **Troubleshooting a Duplicate Bug:** I accidentally uploaded the ZIP file twice, which doubled my event counts and messed up my math! To fix it, I stopped the background `Splunkd Service` using the Windows Services panel, manually deleted the database folder path (`C:\Program Files\Splunk\var\lib\splunk\defaultdb`), restarted the service, and re-uploaded the file just ONCE to get the magic number: **109,864 healthy events**.

---

## 🧠 Core Concepts I Mastered for the Exam
* **Time Range Matters:** Splunk defaults to searching the *Last 24 hours*. Because lab data is historical, it will show `0 events` unless you change the picker dropdown to **All time**.
* **Source vs Sourcetype:** `source` is the actual file container on disk (the ZIP file path). `sourcetype` is the format of the data inside (e.g., Apache web logs vs MySQL database logs). Adding `sourcetype` makes searches way faster because Splunk ignores unrelated files.
* **The Pipe (`|`):** Acts like an assembly line conveyor belt. It takes the raw filtered results from the left side and feeds them into the calculation command on the right side.

---

## 💻 Every Single Command I Practiced & What They Do

### 1. The Broad Search
```spl
source="tutorialdata.zip:*"
```
* **What it does:** Uses a wildcard (`*`) to bring back absolutely every single log line extracted from my uploaded lab ZIP file.

### 2. Basic Filtering (Keywords & Fields)
```spl
source="tutorialdata.zip:*" buttercupgames
```
* **What it does:** Acts like Google. Pulls out any log mentioning the website URL text. (Gave me exactly 5,327 events).

```spl
source="tutorialdata.zip:*" categoryId=sports
```
* **What it does:** Searches inside a specific field. Isolates logs where users clicked on the "Sports" game category. (Gave me 115 events).

### 3. Error Hunting with Boolean Logic
```spl
buttercupgames (error OR fail* OR severe)
```
* **What it does:** Looks for the company name AND any of the bad words in brackets. The `fail*` wildcard matches *fail, failure, failed, or failing*. This instantly grabbed the 427 times the system glitched or payments failed.

### 4. Grouping and Counting (`stats`)
```spl
source="tutorialdata.zip:*" | stats count by categoryId
```
* **What it does:** Strips away raw text and builds a summary table counting how many clicks happened per category. I clicked the **Visualization tab** on this to see my very first **Bar Graph** summary!

### 5. Deep-Dive User Auditing
```spl
source="tutorialdata.zip:*" clientip=87.194.216.51 | stats count by action, status
```
* **What it does:** Investigates our highest-activity user. It breaks their clicks down by *what they did* (`action`) and *the HTTP code result* (`status`). I discovered they successfully bought items 134 times (Legit customer!), but also hit massive server errors (500 and 503 codes), showing the web servers were crashing under load.

### 6. Leaderboards with Limits
```spl
sourcetype=access_* status=200 action=purchase | top limit=1 clientip
```
* **What it does:** Finds successful checkouts (`status=200 action=purchase`), feeds them to the `top` command, and uses `limit=1` to isolate the number-one single biggest shopper on the entire site.

### 7. Advanced Profiling
```spl
source="tutorialdata.zip:*" sourcetype=access_* status=200 action=purchase clientip=87.194.216.51 
| stats count, distinct_count(productId), values(productId) by clientip
```
* **What it does:** Builds a security baseline of our top user. 
  * `count` shows total checkouts (134).
  * `distinct_count` shows how many *different* games they bought (14 unique games).
  * `values` lists the exact game IDs they own in a clean list.

### 8. The Ultimate Automated Subsearch
```spl
sourcetype=access_* status=200 action=purchase [ search sourcetype=access_* status=200 action=purchase | top limit=1 clientip | table clientip ] | stats count AS "Total Purchased" by productId
```
* **What it does:** **The holy grail of Lab 6.** The square brackets run *first*, find the top IP address automatically, and hand it to the outer search. This means the query completely automates the investigation—even if a different hacker or customer becomes the top user tomorrow, this code will find them automatically without me changing a single character.

