\# 💉 SQL Injection (SQLi)



This directory contains my custom automated tools for detecting and exploiting SQL Injection vulnerabilities. Instead of relying on manual data extraction or commercial tools, these scripts leverage Python's speed and precision.



\## 🛠️ Techniques Covered

\- \*\*In-band (UNION based):\*\* Direct data extraction through the application's response.

\- \*\*Blind (Boolean-based):\*\* Automating thousands of requests to guess data bit by bit based on TRUE/FALSE responses.

\- \*\*Blind (Time-based):\*\* Using time delays to exfiltrate information.



\## 📂 Featured Labs

| Lab Name | Vulnerability | Script Focus |

| :--- | :--- | :--- |

| \[Lab: Blind SQLi with conditional responses](./lab-blind-boolean) | Blind Boolean | Character-by-character automated guessing. |

| \[Lab: UNION attack](./lab-union-data) | UNION-based | Table \& column enumeration. |



\## 🚀 Common Logic

Most scripts here use the `requests` library to manage sessions and headers, ensuring the attack bypasses basic filters.

