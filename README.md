# Nmap and ChatGPT Integration with Wazuh

This project integrates Nmap, a network scanning tool, and ChatGPT, an AI language model, with Wazuh, a security monitoring platform. This integration enhances network security by providing detailed information about network activity and potential vulnerabilities.

## Overview

-   **Nmap (Network Mapper):** A powerful open-source scanner for network exploration and security auditing. It helps identify hosts and services on a network.
-   **ChatGPT:** An AI-powered language model (GPT-4 architecture) that provides human-like text responses. In this context, it assists in analyzing security audit data, threat hunting, and summarizing security issues.
-   **Wazuh:** A security monitoring tool that leverages the capabilities of Nmap and ChatGPT to improve an organization's security posture.

The integration demonstrates how Wazuh utilizes Nmap for network scanning and ChatGPT for intelligent analysis of the scan results, providing a more comprehensive security overview.

## Infrastructure

The following infrastructure was used to demonstrate the integration:

-   Wazuh 4.11
-   Windows 10/11 endpoint with Wazuh agent 4.11
-   Python 3.13.3 or later
-   Nmap v7.95 or later

## Nmap Integration

This section details how to configure Nmap to scan a Windows endpoint and send the results to Wazuh.

### Windows Endpoint Configuration

1.  **Install Python:** Download and install Python 3.13.3 or later (with pip pre-installed). Ensure "Use admin privileges when installing py.exe" and "Add python.exe to PATH" are checked during installation.
2.  **Install Nmap:** Download and install Nmap v7.95 or later, and add Nmap to the system's PATH.
3.  **Install python-nmap:** Use PowerShell to install the python-nmap library:

    ```powershell
    pip install python-nmap
    ```

4.  **Configure Nmap Scan with Python:**
    * Create a Python script (e.g., `nmapscan.py`) to perform the Nmap scan and format the output.
    * Convert the script into an executable using PyInstaller:

        ```powershell
        pip install pyinstaller
        pyinstaller -F C:\path\to\your\nmapscan.py
        ```
    * Copy the executable to `C:\Users\<USERNAME>\Documents\nmapscan.exe`.
5.  **Configure Wazuh Agent:** Edit the Wazuh agent's `ossec.conf` file to monitor the Nmap scan output.  Add the following within the `<ossec_config>` block:

    ```xml
    <localfile>
      <log_format>full_command</log_format>
      <command>C:\Users\<USERNAME>\Documents\nmapscan.exe</command>
      <frequency>604800</frequency>
    </localfile>
    ```
    Replace `<USERNAME>` with the actual username.
6.  **Restart Wazuh Agent:** Restart the Wazuh agent for the changes to take effect:

    ```powershell
    Restart-Service -Name wazuh
    ```

### Wazuh Server Configuration

1.  **Add Custom Rule:** Add a rule to `/var/ossec/etc/rules/local_rules.xml` to capture the Nmap scan results:

    ```xml
    <group name="windows, nmap, ">
      <rule id="100100" level="3">
        <decoded_as>json</decoded_as>
        <field name="nmap_port">\.+</field>
        <field name="nmap_port_service">\.+</field>
        <description>NMAP: Host scan. Port $(nmap_port) is open and hosting the $(nmap_port_service) service.</description>
        <options>no_full_log</options>
      </rule>
    </group>
    ```

2.  **Restart Wazuh Manager:** Restart the Wazuh manager:

    ```bash
    sudo systemctl restart wazuh-manager
    ```

3.  **View Scan Results:** The Nmap scan alerts can be viewed in the Wazuh dashboard under the Security events tab.

## ChatGPT Integration

This section describes how to integrate ChatGPT with Wazuh to provide more information about the open ports identified by Nmap.

### Wazuh Server Configuration

1.  **Add Custom Rules:** Add the following rules to `/var/ossec/etc/rules/local_rules.xml`:

    ```xml
    <group name="linux,chat_gpt">
      <rule id="100101" level="5">
        <if_sid>100100</if_sid>
        <field name="nmap_port">\d+</field>
        <description>NMAP: Host scan. Port $(nmap_port) is open.</description>
      </rule>
      <rule id="100103" level="5">
        <if_sid>100100</if_sid>
        <field name="nmap_port_service">^\s$</field>
        <description>NMAP: Port $(nmap_port) is open but no service is found.</description>
      </rule>
    </group>
    ```

2.  **Install Python Requests Library:** Install the `requests` library:

    ```bash
    sudo apt install python3-requests
    ```

3.  **Create Integration Script:** Create a Python script `/var/ossec/integrations/custom-chatgpt.py` to interact with the ChatGPT API.  (See the  `custom-chatgpt.py` script from the "ChatGPT Integration" section of the document)
4.  **Set Permissions:** Grant execute permissions and set the correct owner and group for the script:

    ```bash
    chmod 750 /var/ossec/integrations/custom-chatgpt.py
    chown root:wazuh /var/ossec/integrations/custom-chatgpt.py
    ```

5.  **Configure Wazuh Integration:** Add the following integration block to `/var/ossec/etc/ossec.conf` within the `<ossec_config>` block:

    ```xml
    <integration>
      <name>custom-chatgpt.py</name>
      <hook_url>[https://api.openai.com/v1/chat/completions](https://api.openai.com/v1/chat/completions)</hook_url>
      <api_key><YOUR_CHATGPT_API_KEY></api_key>
      <level>5</level>
      <rule_id>100101</rule_id>
      <alert_format>json</alert_format>
    </integration>
    ```

    Replace `<YOUR_CHATGPT_API_KEY>` with your actual ChatGPT API key.  Obtain an API key from [OpenAI](https://platform.openai.com/signup).
6.  **Add Custom Rule for ChatGPT Output:** Add the following rule to `/var/ossec/etc/rules/local_rules.xml`:

    ```xml
    <group name="local,linux,">
      <rule id="100102" level="6">
        <field name="chatgpt.nmap_port_service">\w+</field>
        <description>The service $(chatgpt.nmap_port_service) is on an open port.</description>
      </rule>
    </group>
    ```

7.  **Restart Wazuh Manager:** Restart the Wazuh manager:

    ```bash
    systemctl restart wazuh-manager
    ```

8.  **View Scan Results:** The enhanced Nmap scan alerts, enriched with ChatGPT information, can be viewed in the Wazuh dashboard.

