# CTF Writeup: Anthem (TryHackMe)

**Date:** March 16, 2026

**Platform:** TryHackMe

**Difficulty:** Easy

**Focus:** Recon, Enumeration, Windows Filesystem Exploration

---

## 1. Introduction
Anthem is a Windows-based machine that focuses on web enumeration and exploring the Umbraco CMS. The path to root involves finding hidden metadata and misconfigured file permissions.

---

## 2. Reconnaissance & Enumeration

### Network Scanning
I started with an `nmap` with '-sC -sV' flags to scan and identify open ports and services:
* **Port 80 (HTTP):** Microsoft HTTPAPI httpd 2.0
* **Port 3389 (RDP):** Microsoft Terminal Services

### Web Discovery
Accessing the web server, I checked `robots.txt`, which revealed:
* **Potential Password:** `UmbracoIsTheBest!`.
* **CMS:** Confirmed as **Umbraco**, an open-source .NET CMS.
* **Domain:** Identified as `anthem.com` from the page footer.

---

## 3. Information Gathering (OSINT & Metadata)

1.  **Metadata Hunting:** By inspecting the page source code, I found several flags hidden in `<meta>` tags and search placeholders.
    * `THM{G!T_G00D}`
    * `THM{L0L_WH0_US3S_M3T4}`
    * `THM{AN0TH3R_M3TA}`
2.  **User Identification:** * A blog post mentioned a "beloved admin" followed by a poem about **Solomon Grundy**.
    * Found an email for "Jane Doe" (`JD@anthem.com`).
    * **Logic:** Based on the naming convention, I deduced the Administrator's email was `SG@anthem.com`.

---

## 4. Initial Access

### The Umbraco Portal
Using the credentials found during enumeration (`SG@anthem.com` : `UmbracoIsTheBest!`), I successfully authenticated to the Umbraco login panel.

### Gaining Shell via RDP
To gain a desktop environment, I attempted to use the same credentials for **RDP**. 
* **Username:** `SG` (Derived by removing the domain from the email).
* **Password:** `UmbracoIsTheBest!`.

Upon login, I retrieved the user flag: `THM{N00T_NO0T}`.

---

## 5. Privilege Escalation

### Exploring the Filesystem
After auditing the filesystem with hidden files enabled, I discovered a hidden folder at the root of the drive: `C:\backup`.

### Password Recovery
Inside the backup folder, I found a file containing a cleartext string after modifying its permissions:
* **Admin Password:** `ChangeMeBaby1MoreTime`.

### Root Flag
I logged in as the **Administrator** and captured the final flag from the Desktop:
* **Root Flag:** `THM{Y0U_4R3_1337}`.

---

## 6. Lessons Learned
* **Examine Source Code:** Always check meta tags and comments for hidden information or flags.
* **Pattern Recognition:** Identifying email naming conventions (`JD` -> `SG`) is crucial for user enumeration.
* **Hidden Files:** Always check for hidden directories like `C:\backup` which may contain sensitive configuration files.
