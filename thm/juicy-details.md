# CTF Writeup: Juicy Details (TryHackMe)

**Date:** April 27, 2026

**Platform:** TryHackMe

**Difficulty:** Beginner

**Focus:** Log Analysis, Forensics, Incident Response

---

## 1. Introduction
In the "Juicy Details" challenge, we step into the shoes of a SOC Analyst for one of the world's largest Juice Shops. An attacker has breached the network, and our objective is to analyze a provided zip file containing server logs (`access.log`, `auth.log`, `vsftpd.log`). The goal is to uncover the attacker's tools, identify vulnerable endpoints, and determine exactly what sensitive data was compromised during the incident. 

---

## 2. Reconnaissance & Enumeration (Task 2)

### Tool Identification
By analyzing the `access.log` file and examining the `User-Agent` strings, I identified the sequence of automated tools the attacker utilized. Scrolling through the logs revealed them in the following order:
* **Nmap:** `Mozilla/5.0 (compatible; Nmap Scripting Engine...`
* **Hydra:** `Mozilla/5.0 (Hydra)`
* **SQLmap:** `sqlmap/1.5.2#stable (http://sqlmap.org)`
* **Curl:** `curl/7.74.0`
* **Feroxbuster:** `feroxbuster/2.2.1`

### Vulnerable Endpoints
Reviewing the targets of these automated tools revealed the vulnerable areas of the application:
* **Brute-Force Target:** By filtering for the Hydra user-agent, I observed numerous `500` and `401` HTTP response codes directed at the `/rest/user/login` endpoint.
* **SQL Injection (SQLi) Target:** Similarly, tracking the SQLmap user-agent pointed directly to the `/rest/products/search` endpoint.
* **SQLi Parameter:** Examining the HTTP GET requests crafted by SQLmap (e.g., `/rest/products/search?q=...`), it was evident that the `q` query parameter was the injection point.
* **File Retrieval:** Toward the end of the `access.log`, the Feroxbuster scan revealed a directory enumeration attempt that successfully located the `/ftp` endpoint.

---

## 3. Stolen Data & Post-Exploitation (Task 3)

### Information Scraping & Authentication Bypass
* **Email Scraping:** Knowing that customers usually leave comments in feedback or review sections, I searched the `access.log` for keywords like "reviews". I found heavy traffic to the `/rest/products/*/reviews` endpoints, indicating the attacker scraped this section for user email addresses.
* **Successful Brute-Force:** To determine if Hydra successfully cracked an account, I filtered the Hydra login attempts in `access.log` for a `200 OK` HTTP status code. The brute-force was indeed successful, with the timestamp of the compromise being: `Yay, 11/Apr/2021:09:16:31 +0000`.

### Data Exfiltration
* **Database Dump via SQLi:** Looking at the final payloads delivered by SQLmap, I observed a UNION-based SQL injection (`UNION SELECT id, email, password, '4', '5'... FROM Users`). The attacker successfully retrieved the `id`, `email`, and `password` columns from the database.
* **File Download:** Switching over to the `vsftpd.log` file, the records explicitly showed the attacker successfully downloading two backup files: `www-data.bak` and `coupons_2013.md.bak`.
* **FTP Access:** The `vsftpd.log` also confirmed that the attacker accessed these files using the `ftp` service by utilizing an `anonymous` login.

### Shell Access
* **Gaining a Foothold:** Finally, reviewing the `auth.log` revealed the attacker's pivot to shell access. After a few initial failed attempts, the logs show an accepted password and an opened session for the `www-data` user over the `ssh` service.

---

## 4. Lessons Learned
* **Monitor User-Agents:** Implementing strict monitoring or blocking for known automated scanning and attack tools (Hydra, SQLmap, Feroxbuster) can thwart attacks early in the reconnaissance phase.
* **Implement Rate Limiting:** The sheer volume of brute-force attempts on `/rest/user/login` should have triggered a lockout or rate-limiting mechanism to prevent password guessing.
* **Sanitize Inputs:** The SQL injection vulnerability on the `q` parameter highlights the critical need for parameterized queries to prevent database dumping.
* **Secure FTP Configurations:** Anonymous FTP access should be disabled, and sensitive backup files (`.bak`) should never be stored in publicly accessible or anonymous directories.
