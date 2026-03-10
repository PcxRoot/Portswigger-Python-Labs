# Lab: 2FA simple bypass

_Leer esto en Español: [Readme_es.md](./Readme_es.md)_

[__Link to the Lab__](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-simple-bypass)

> [!NOTE]
> **Lab Analysis:** If you are looking to understand the vulnerability in depth, a **detailed technical explanation (no spoilers)** regarding the attack mechanics and database logic is provided right below the usage section.
> Jump directly [there](#methodology--ethics).

# Automation Script

This directory contains an exploit developed in Python designed to automate the detection and exploitation of the vulnerability found in this lab.

### __Usage__

>Create a Python virtual environment (Recommended)
```
python -m venv venv
```

>Activate the virtual environment
>- Linux
>```bash
>source venv/bin/activate
>```
>- Windows
>```
>venv\Scripts\activate --> Símbolo del sistema (CMD)
>venv\Scripts\activate.ps1 --> PowerShell
>```

>Install dependencies
```python
pip install -r requirements.txt
```

>Run the script
```python
python exploit.py -h --> Show help menu

python exploit.py -t [URL]
```

![Output example](./img/Output%20exploit.png)

---

## Methodology & Ethics

>[!IMPORTANT]
>__Learning Notice:__ The following section details the vulnerability's mechanics using a pedagogical approach without spoilers. I encourage you to attempt the lab on your own before consulting this analysis. True mastery comes from persistent problem-solving.

---

## Lab Objective

The goal of this lab is to bypass the Two-Factor Authentication (2FA) enforced during the redirection that occurs after a successful login with valid credentials.

To solve it, the exploit must:
1. __Login with credentials:__ Authenticate using the credentials provided by the lab.

2. __2FA Bypass:__ After logging in, bypass the second factor by navigating directly to the `/my-account` endpoint.

### Technical Analysis of the Vulnerability

The web application verifies the user's identity by requiring a temporary unique code sent via email after correct credentials have been entered.

In this scenario, we possess the victim's credentials but lack access to their email, which would effectively prevent us from accessing the account under normal conditions.

The flaw here does not lie within the 2FA code itself, but in __flawed session state management__. The server assumes the user will only reach their dashboard if they follow the intended navigation flow, yet it fails to strictly validate this state on the server side.

#### The Broken Authentication Flow

In a secure implementation, access to `/my-account` should be restricted by a "_flag_" or state confirming that both authentication factors have been completed. However, the following occurs in this lab:

1. __Step 1 (Credentials):__ We enter a valid username and password. The server identifies us and generates an active session.

2. __Step 2 (Redirection):__ The server automatically redirects us to `/login2` to request the 2FA code.

3. __The Logic Flaw:__ Although we are on the 2FA screen, the session is already considered "authenticated" for the `/my-account` endpoint. The server trusts the session cookie granted after the first step and fails to verify if the second factor has been validated before serving protected content.

## 🐍 Python Automation (The Exploit)

While manual exploitation is straightforward, automation helps develop skills in Scripting for Pentesting and HTTP state management.

Script Execution Logic:

1. __URL Normalization:__ Ensures the target URL is processed correctly regardless of a trailing slash (`/`).

2. __POST Request with Credentials:__ Performs the initial `POST` request using the correct credentials.

3. __2FA Bypass:__ Once credentials are submitted and the session is redirected to the `/login2` _endpoint_, the script immediately performs a `GET` request to the `/my-account` _endpoint_. Since the server already considers the session "authenticated" and treats the 2FA as a client-side hurdle rather than a server-side requirement, access is granted.

4. __Success Verification:__ To confirm the exploit was successful, the script parses the server's response for the username "carlos". If it exists within the first paragraph of the response, the exploit is marked as successful; otherwise, it is considered a failure.

## Mitigation

To correct this vulnerability, the server should implement a __partial authentication state__. The session should not grant access to protected resources until a specific attribute (e.g., `mfa_verified: true`) is updated in the session database after the correct 2FA code has been submitted and verified.