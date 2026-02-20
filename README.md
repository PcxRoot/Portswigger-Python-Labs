# 🛡️ PortSwigger Labs: Custom Security Toolbox

![Python](https://img.shields.io/badge/Language-Python3-blue?style=for-the-badge&logo=python)
![Cybersecurity](https://img.shields.io/badge/Focus-Ethical_Hacking-red?style=for-the-badge)

Welcome to my personal cybersecurity research lab. This repository documents my journey through the **PortSwigger Web Security Academy**. My philosophy is simple: don't just solve the lab—understand the vulnerability at the protocol level and build your own tools to exploit it.

## 🚀 Why this project?

As a cybersecurity student, I believe that building tools is the best way to master a concept. While **Burp Suite Professional** is the industry standard, its **Community Edition** has limitations (like rate-limiting/throttling in the Intruder).

I develop **custom Python scripts** to overcome these hurdles, focusing on:
1. **Efficiency:** Bypassing speed throttles to perform faster automated attacks.
2. **Logic Mastery:** Building exploits that dynamically handle sessions, CSRF tokens, and complex authentication flows.
3. **Deep Learning:** If you can code the exploit from scratch, you truly understand the bug.

---

## ⚖️ Ethical & Legal Disclaimer

This content is for **educational and research purposes only**.
- All tools are designed to be used within the controlled environments of PortSwigger Academy.
- I do not provide "flags" or direct answers; I explain the **methodology** and the **underlying logic**.
- The author is not responsible for any misuse of this code in unauthorized environments.

---

## 📂 Repository Structure

| Category | Description | Status |
| :--- | :--- | :--- |
| [💉 SQL Injection](./sql-injection) | Data extraction via UNION and Blind SQLi techniques. | In progress |

---

## 🛠️ General Requirements

To run most scripts, you will need:
- Python 3.x
- Libraries: `requests`, `beautifulsoup4`
- Installation: `pip install -r requirements.txt` (available in each subdirectory).

---

> "Cybersecurity isn't about the tools you use; it's about how you understand the data flowing between the client and the server."
