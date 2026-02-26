# Lab: Usernames enumeration via different responses

_Leer esto en Español: [Readme_es.md](./Readme_es.md)_

[__Link to the Lab__](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-different-responses)

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

python exploit.py -t [URL] -u [Usernames_list] -p [Passwords_list]
```
<p align="center">
    <img src="./img/banner.png" width="45%" alt="Banner" />
    <img src="./img/succes.png" width="45%" alt="Banner" />
</p>

## Methodology & Ethics

>[!IMPORTANT]
>__Learning Notice:__ The following section details the vulnerability's mechanics using a pedagogical approach without spoilers. I encourage you to attempt the lab on your own before consulting this analysis. True mastery comes from persistent problem-solving.

---

## Lab Objective

The goal is to perform a Brute Force attack to compare server response status codes to verify if a credential combination is correct.
To solve it, the exploit must:

1. __Resource Validation:__ Verify the existence and accessibility of the user and password wordlists.

2. __Attack Execution:__ Orchestrate the delivery of payloads by combining each user with the full list of passwords.

3. __Response Analysis:__ Identify success patterns based on HTTP redirections.

4. __Post-Exploitation:__ Perform automated authentication to validate the session and confirm the lab's resolution.

## Technical Analysis of the Vulnerability

The web application features a __login__ functionality vulnerable to __Brute Force__ attacks because it lacks any defense mechanisms against this type of threat.

1. __Functionality Reconnaissance:__ Whenever we attempt to compromise a feature, the first step is to investigate its implementation.
During this phase, it was determined that:

- __Entry Vector:__ The application manages authentication via the __HTTP POST__ method.

- __Absence of Controls:__ The system does not implement __Rate Limiting__ mechanisms or _Account Lockout_ policies for failed attempts.

- __Server Behavior:__ When faced with invalid credentials, the server responds uniformly with a `200 OK` code, keeping the session on the current login page.


2. __Exploitation Strategy:__ Given the lack of protections, a parallel dictionary attack was chosen. The success detection logic is based on the HTTP redirection flow:

- __Success Detection:__ Valid authentication causes the server to issue a `302 Found` (or `303 See Other`) status code, redirecting the user to the account dashboard (`/my-account`).

- __Efficiency:__ The exploit utilizes a multi-threaded architecture to reduce execution time, ignoring `200 OK` responses and focusing filtering exclusively on redirection codes.

---

## Python Automation (The Exploit)

This lab is not difficult to perform using traditional tools like Burp Suite, which can execute these attacks via its Intruder module. However, the Community Edition of Burp Suite imposes a performance limitation known as __Throttling__.

__What is Throttling?__
__Throttling__ is a technique used to limit bandwidth or request rates. In __Burp Suite Community Edition__, the _Intruder_ module has a software-induced speed limit.

Unlike a real network issue, this delay is __deliberate__:

- __Burp Suite Pro:__ Sends requests at the maximum speed allowed by the server and your connection.

- __Burp Suite Community:__ Introduces a delay that typically increases progressively as the attack continues, drastically reducing the requests per second (__RPS__).

This is one of the primary reasons why it is essential to know how to create your own tools. By developing a custom exploit, we regain __full control__ over the network layer, eliminating the artificial speed restrictions imposed by free versions of commercial suites.

This __technical autonomy__ not only drastically optimizes execution time during an audit but also ensures the auditor has __total control over the execution logic and network state management__.

### Script Execution Logic

The exploit follows a modular flow designed to maximize response speed and ensure the integrity of the exfiltrated data. The execution lifecycle is detailed below:

1. __Validation and Preparation (Pre-flight)__
Before launching the first request, the script performs critical checks:

- __URL Normalization:__ Ensures the `/login` endpoint is reachable.

- __Wordlist Integrity:__ Verifies the existence of the user and password wordlists to prevent runtime errors.

- __Banner & CLI:__ Loads the command-line interface using `argparse`.

2. __Efficient Network Management (Keep-Alive)__
Unlike basic sequential attacks, this script utilizes a `requests.Session()` object.

- __Optimization:__ This allows the underlying TCP connection (_TLS Handshake_) to remain open throughout the attack, eliminating the unnecessary latency of opening and closing sockets for every attempt.

3. __Concurrent Orchestration (ThreadPoolExecutor)__
The core of the performance lies in the use of __parallel threads__:

- __Architecture:__ A team of __10 workers__ (_threads_) (You can chenge it) is implemented to consume tasks from a global queue.

- __Nested Loop:__ The script iterates over every user and password, sending these combinations to the thread pool asynchronously.

4. __The "Kill Switch" and Thread-Safety__
To avoid unnecessary resource consumption once the goal is reached:

- __Global State:__ A boolean variable `state` acts as a switch. As soon as a thread receives a `302/303` status code (Successful Redirection), it triggers the switch, and all other threads stop their activity immediately.

- __Concurrency Control (Lock):__ `threading.Lock()` is used to ensure the terminal and the `Credentials.txt` file do not suffer from `race conditions`, guaranteeing that logs remain legible and professional.

5. __Final Validation and DOM Parsing__
Once credentials are obtained, the script performs a final logical "two-factor" validation:

- __BeautifulSoup:__ Accesses the account and parses the HTML looking for the `div#account-content` container.

- __Confirmation:__ The script marks the lab as __SOLVED__ only if the username appears within the welcome paragraph.

---

## Mitigation and Best Practices

The success of this exploit relies on the lack of flow controls and the ability to distinguish between valid and invalid responses. To prevent brute force attacks and username enumeration, the following measures are recommended:

1. __Implementation of Rate Limiting__
The most effective defense against this specific script is to limit the number of requests a client can make in a given period.
- __By IP:__ Temporarily block or slow down connections that exceed a certain threshold (e.g., 5 attempts per minute).
-__By Account:__ If multiple failures are detected for the same user, apply an __exponential backoff__ delay in the response.

2. __Generic Error Messages__
The server should avoid giving hints about which part of the credentials was wrong.
- __Bad practice:__ "The password is incorrect" (Indicates the user exists).
- __Good practice:__ "Invalid username or password". The server should return the same message, the same HTTP status code, and preferably maintain a uniform response time to prevent __side-channel attacks__.

3. __Account Lockout Mechanisms__
Temporarily lock an account after $X$ failed attempts.
    >[!Note]
    >Caution must be exercised with this measure, as an attacker could use it to cause a massive Denial of Service (DoS) by locking the accounts of all legitimate users if their usernames are known.

4. __Multi-Factor Authentication (MFA/2FA)__
This is the most effective defense. Even if an attacker manages to exfiltrate the correct password, they will not be able to access the account without the second factor (__OTP__, __hardware key__, etc.), rendering the brute force attack impactless.

5. __Introduction of CAPTCHAs__
Implement challenges that require human interaction (such as reCAPTCHA or hCaptcha) after detecting automated behavior. This stops tools based on `requests` and `ThreadPoolExecutor` in their tracks.

>Authentication security should not depend on the complexity of the user's password, but on the system's ability to detect and neutralize automated traffic patterns.