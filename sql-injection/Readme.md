# 💉 SQL Injection (SQLi)



This directory contains my custom automated tools for detecting and exploiting SQL Injection vulnerabilities. Instead of relying on manual data extraction or commercial tools, these scripts leverage Python's speed and precision.



## 🛠️ Techniques Covered

- __In-band (UNION based):__ Direct data extraction through the application's response.

- __Blind (Boolean-based):__ Automating thousands of requests to guess data bit by bit based on TRUE/FALSE responses.

- __Blind (Time-based):__ Using time delays to exfiltrate information.



## 📂 Featured Labs


| Lab Name | Vulnerability | Script Focus | State |
| :--- | :--- | :--- | :--- |
| [Lab: SQLi where hidden data](./sqli-where-hidden-data/) | "WHERE clause" Boolean | Modified WHERE clause | <font color="gren">Completed</font> |
| [Lab: SQLi allowing login bypass](./sqli-login-bypass/) | "WHERE clause" Boolean Commenting | Modified WHERE clause | <font color="gren">Completed</font> |
| [Lab: Blind SQLi with conditional responses](./lab-blind-boolean) | Blind Boolean | Character-by-character automated guessing. | <font color="red">In progress</font> |
| [Lab: UNION attack](./lab-union-data) | UNION-based | Table \& column enumeration. | <font color="red">In progress</font> |



## 🚀 Common Logic

Most scripts here use the `requests` library to manage sessions and headers, ensuring the attack bypasses basic filters.

