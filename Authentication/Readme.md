# 🔐 Authentication Vulnerabilities

This directory contains my custom automated tools for detecting and exploiting flaws in authentication mechanisms. These scripts focus on bypassing identity verification, automating credential attacks, and exploiting logic flaws in login workflows.

## 🛠️ Techniques Covered

- __Username Enumeration:__ Identifying valid users through subtle differences in server responses (messages, status codes, or response times).

- __Brute Force & Dictionary Attacks:__ High-speed credential testing using custom multi-threaded scripts to bypass "_throttling_."

- __MFA/2FA Bypass:__ Exploiting logic flaws in two-factor authentication (e.g., missing session validation or predictable codes).

- __Rate Limit Circumvention:__ Using header manipulation and session management to avoid account lockouts or IP blocking.

## 📂 Featured Labs

| Lab Name | Vulnerability | Script Focus | State |
| :--- | :--- | :--- | :--- |
| [Lab: Username enumeration via different responses](./enumeration-different-responses/) | Response-based Enumeration | Response parsing & Threading | :heavy_check_mark: |
| [Lab: 2FA simple bypass](./2FA%20simple%20bypass/) | 2FA Authentication Bypass via Forceful Browsing | Querying specific endpoints | :heavy_check_mark: |

## Common Logic
Authentication exploits require a higher degree of __state management__ compared to simple injections. Most scripts here implement:

- __Session Persistence:__ Using `requests.Session()` to handle cookies and maintain the state between the enumeration and exploitation phases.

- __CSRF Handling:__ Automated extraction of hidden anti-CSRF tokens from the DOM using `BeautifulSoup` to ensure every POST request is accepted by the server.

- __Concurrency (Threading):__ Implementation of `ThreadPoolExecutor` to maximize Requests Per Second (RPS), bypassing the artificial delays (throttling) of free security suites.

- __Regex Validation:__ Precision filtering of response bodies to detect successful login redirects (`302 Found`) or specific welcome messages.

## ⚖️ Methodology and Ethics
The tools provided here are for educational and ethical hacking purposes only. Automating authentication attacks requires a deep understanding of the target's "_Rate Limiting_" and "_Account Lockout_" policies to avoid unintended Denial of Service (DoS) or locking out legitimate users.