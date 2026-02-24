# A Security-Focused Guide to Setting Up OpenClaw on a Dedicated Windows 11 Machine

**Author:** Manus AI
**Date:** February 24, 2026

## 1. Introduction: What is OpenClaw?

OpenClaw, the open-source project formerly known as ClawdBot and Moltbot, is a self-hosted personal AI assistant that has seen a meteoric rise in popularity in early 2026. Its core function is to act as an orchestration layer, connecting powerful large language models (LLMs) from cloud providers like OpenAI or Manus to a user's personal data and applications. It grants the AI agent autonomous access to local files, shell commands, web browsers, and messaging platforms (such as WhatsApp, Telegram, and Slack), enabling it to perform a wide range of tasks on the user's behalf [1].

Unlike cloud-native assistants, OpenClaw runs on user-owned hardware. Its architecture is centered around a "Gateway" that acts as a control plane, managing the agent's tools, memory, and communication channels. The agent's capabilities are extended through "skills," which are typically simple Markdown files containing instructions. While this design offers immense power and flexibility, it also introduces significant security risks, as the agent operates with the full permissions of the user account under which it is run [2]. This guide provides a security-first approach to installing and operating OpenClaw on a dedicated Windows 11 machine, specifically for users who wish to connect to cloud APIs rather than run local models.

## 2. The 2026 Security Threat Landscape: A Critical Warning

The rapid adoption of OpenClaw has not gone unnoticed by security researchers or malicious actors. Within weeks of its viral explosion, a series of critical vulnerabilities and attack campaigns emerged, prompting warnings from major cybersecurity firms including Cisco, Trend Micro, Palo Alto Networks, and 1Password [3][4][5]. For any user, understanding this threat landscape is not optional—it is a prerequisite for safe operation.

> As of February 2026, OpenClaw has no bug bounty program and no dedicated security team. Running it with default settings on a machine with access to production credentials, messaging accounts, or sensitive data is extremely high-risk. Security researchers consistently recommend: if you cannot harden and monitor it, do not expose it. [6]

### Key Threats

| Threat Vector | Description | Severity |
| :--- | :--- | :--- |
| **Malicious Skills** | The most significant threat vector. Public skill registries like ClawHub were flooded with hundreds of malicious skills in a campaign dubbed "ClawHavoc." These skills disguise malware installers as "prerequisites" in their documentation, tricking the agent and user into executing them. The primary payload has been the **Atomic Stealer (AMOS)**, a potent infostealer targeting macOS and Windows credentials [4][6]. | **CRITICAL** |
| **Infostealer Malware** | Commodity infostealers (e.g., RedLine, Lumma) are now actively programmed to find and exfiltrate OpenClaw's configuration directory (`~/.openclaw/`). These attacks target plaintext credential files, including `openclaw.json` (gateway tokens), `device.json` (cryptographic keys), and memory files (`SOUL.md`, `MEMORY.md`) that contain sensitive user context and history [5][6]. | **HIGH** |
| **Remote Code Execution** | **CVE-2026-25253**, a critical vulnerability, allowed a one-click RCE attack. By tricking a user into visiting a malicious webpage, an attacker could steal the gateway authentication token and gain complete control over the host machine. While patched, it highlights the fragility of the system's security model [6]. | **CRITICAL** |
| **Prompt Injection** | The agent's browser and communication modules are susceptible to indirect prompt injection. An attacker can embed hidden instructions in webpages or emails. When the agent processes this content, it may inadvertently execute malicious commands, turning the open web into a command-and-control channel [6]. | **HIGH** |

Given the low specifications of the dedicated machine (3rd Gen Intel Core i5, integrated graphics), running local LLMs is not feasible. This guide will therefore focus exclusively on a cloud API setup, which simplifies the process but makes API key security the paramount concern.

## 3. Step-by-Step Secure Installation via WSL2

The **only** recommended method for running OpenClaw on Windows is inside the Windows Subsystem for Linux 2 (WSL2). This provides a crucial layer of isolation between the high-risk OpenClaw environment and the underlying Windows host operating system.

### Step 3.1: Install WSL2 and Ubuntu

First, you must install WSL2 and a stable Linux distribution. Ubuntu is the standard and is well-supported by the OpenClaw project.

1.  Open **PowerShell as an Administrator**.
2.  Run the following command to install WSL and the latest version of Ubuntu:
    ```powershell
    wsl --install
    ```
3.  Reboot your computer when prompted. After rebooting, the Ubuntu installation will complete, and you will be asked to create a username and password for the Linux environment. **Choose a strong, unique password.**

### Step 3.2: Enable systemd in WSL

The OpenClaw gateway requires `systemd` to be installed and run as a persistent background service. You must enable it manually.

1.  Open your newly installed Ubuntu terminal.
2.  Create and edit the `wsl.conf` file with the following command:
    ```bash
    sudo nano /etc/wsl.conf
    ```
3.  Add the following lines to the file:
    ```ini
    [boot]
    systemd=true
    ```
4.  Press `Ctrl+X`, then `Y`, then `Enter` to save and exit.
5.  Close the Ubuntu terminal and return to your PowerShell (Admin) window. Shut down WSL to apply the changes:
    ```powershell
    wsl --shutdown
    ```

### Step 3.3: Install Dependencies and OpenClaw

Now, re-open your Ubuntu terminal and proceed with installing OpenClaw and its dependencies.

1.  **Install Node.js and pnpm:** OpenClaw requires a recent version of Node.js (v22+).
    ```bash
    # Install nvm (Node Version Manager)
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
    
    # Reload shell to use nvm
    source ~/.bashrc
    
    # Install and use Node.js v22
    nvm install 22
    nvm use 22
    
    # Install pnpm
    npm install -g pnpm
    ```
2.  **Clone and Install OpenClaw:**
    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```
3.  **Run the Onboarding Wizard:** This is the final step to configure the agent. **Crucially, do not enter your API keys directly when prompted.** We will configure them securely in the next section.
    ```bash
    pnpm openclaw onboard --install-daemon
    ```
    Follow the prompts to set up your user, but skip any API key entry for now.

## 4. Security Hardening and Network Isolation

Default settings are not secure. You must actively harden your installation, focusing on API key protection and strict network isolation.

### Step 4.1: Protect Your API Keys

Your cloud API keys are the crown jewels. If stolen, an attacker can use them to run up enormous bills. **Use a new set of API keys for this instance with the lowest possible spending limits.**

Instead of placing keys in the `openclaw.json` file, use environment variables within your WSL2 instance. This provides a basic but important layer of protection against simple file-scraping infostealers.

1.  Open your Ubuntu terminal and edit your shell's profile:
    ```bash
    nano ~/.bashrc
    ```
2.  Add the following lines to the end of the file, replacing the placeholder values with your actual keys:
    ```bash
    export OPENAI_API_KEY='sk-YourSecretOpenAIKey'
    export MANUS_API_KEY='YourSecretManusKey'
    # Add any other keys you plan to use, e.g., ANTHROPIC_API_KEY
    ```
3.  Press `Ctrl+X`, `Y`, and `Enter` to save. Reload the shell for the changes to take effect:
    ```bash
    source ~/.bashrc
    ```
    OpenClaw will automatically detect and use these environment variables.

### Step 4.2: Isolate the Machine with Windows Defender Firewall

This dedicated machine should be treated as a hostile device on your network. We will configure the Windows Firewall to block all incoming connections by default and only allow access from your main Manus computer.

1.  On the OpenClaw machine, open **Windows Defender Firewall with Advanced Security**.
2.  Go to **Inbound Rules** and create a **New Rule...**
3.  Select **Port** and click Next.
4.  Select **TCP** and **Specific local ports**. Enter `18789` (the default OpenClaw gateway port). Click Next.
5.  Select **Allow the connection**. Click Next.
6.  Keep all profiles (Domain, Private, Public) checked. Click Next.
7.  Go to the **Scope** tab for the new rule.
8.  Under **Remote IP address**, select **These IP addresses** and click **Add...**. Enter the **local IP address of your main Manus computer**. 
9.  Give the rule a descriptive name (e.g., "Allow Manus to OpenClaw") and click **Finish**.
10. **Crucially**, right-click on **Windows Defender Firewall with Advanced Security** in the left pane, go to **Properties**, and for each profile (Domain, Private, Public), set **Inbound connections** to **Block (default)**. This ensures only your explicitly defined rule is allowed.

### Step 4.3: Expose the WSL Service with Port Forwarding

By default, services running inside WSL are not accessible from the external network. You must forward the port from the Windows host to the WSL instance.

1.  Open **PowerShell as an Administrator** on the OpenClaw machine.
2.  Run the following script. This will find your WSL IP address and create the forwarding rule.
    ```powershell
    $WslIp = (wsl -- hostname -I).Trim().Split(" ")[0]
    if (-not $WslIp) { throw "WSL IP not found." }
    
    netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=18789 connectaddress=$WslIp connectport=18789
    ```
    **Note:** The WSL IP can change on reboot. If you lose connection after a restart, you must re-run this script.

## 5. Safe Operating Procedures and Skill Management

Your installation is now hardened, but your operational habits are the final layer of defense.

*   **Avoid All Public Skills:** The single most critical security advice is to **never install skills from public repositories like ClawHub**. The risk of malware is unacceptably high. Only use the default, built-in skills provided with the core OpenClaw installation or skills you have written yourself and fully understand.

*   **Enable Sandbox Mode:** In your `~/.openclaw/openclaw.json` file inside WSL, ensure that non-main sessions are sandboxed. This forces the agent to use Docker for tasks initiated in group chats or channels, limiting the blast radius of a compromised skill.
    ```json
    {
      "agent": {
        "model": "openai/gpt-4-turbo-preview" 
      },
      "agents": {
        "defaults": {
          "sandbox": {
            "mode": "non-main"
          }
        }
      }
    }
    ```

*   **Regularly Update:** Keep OpenClaw and all system packages (both Windows and Ubuntu) up to date to ensure you have the latest security patches.
    ```bash
    # Inside WSL
    cd ~/openclaw
    git pull
    pnpm install
    pnpm build
    
    # Update Ubuntu
    sudo apt update && sudo apt upgrade -y
    ```

## 6. Recommendations for a Dedicated Agent Box

Operating this older PC as a dedicated, isolated "agent box" is the correct security posture. It serves as a digital air gap between the high-risk agent and your trusted primary computer.

1.  **Physical and Network Isolation:** Keep the machine on a separate network segment if possible (e.g., a guest Wi-Fi network). The firewall rules we created are the minimum for a single flat network.
2.  **No Personal Data:** Do not log into any personal accounts (email, social media, etc.) on the Windows host of the OpenClaw machine. Do not store any personal files on it. Treat the entire machine as disposable.
3.  **Headless Operation:** Once set up, you do not need to keep a monitor and keyboard attached. You can manage the machine remotely from your main computer via SSH into the WSL instance (using the port forwarding and firewall rules you set up for port 22) or using Windows Remote Desktop.
4.  **Monitoring:** From your main Manus computer, you can connect to the OpenClaw Control UI by navigating to `http://<OpenClaw-Machine-IP>:18789`. Regularly check the agent's logs for any unusual activity or unexpected API costs.

By following this guide, you can experiment with the power of OpenClaw while significantly mitigating the severe security risks highlighted by the research community. Treat the agent as a powerful but untrusted tool, and never expose it to sensitive data or systems.

---

### References

[1] GitHub. *openclaw/openclaw*. [https://github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)

[2] 1Password Blog. *From magic to malware: How OpenClaw's agent skills become an attack surface*. [https://1password.com/blog/from-magic-to-malware-how-openclaws-agent-skills-become-an-attack-surface](https://1password.com/blog/from-magic-to-malware-how-openclaws-agent-skills-become-an-attack-surface)

[3] Cisco Blogs. *Personal AI Agents like OpenClaw Are a Security Nightmare*. [https://blogs.cisco.com/ai/personal-ai-agents-like-openclaw-are-a-security-nightmare](https://blogs.cisco.com/ai/personal-ai-agents-like-openclaw-are-a-security-nightmare)

[4] Trend Micro. *Malicious OpenClaw Skills Used to Distribute Atomic MacOS Stealer*. [https://www.trendmicro.com/en_us/research/26/b/openclaw-skills-used-to-distribute-atomic-macos-stealer.html](https://www.trendmicro.com/en_us/research/26/b/openclaw-skills-used-to-distribute-atomic-macos-stealer.html)

[5] eSecurity Planet. *Infostealers Target OpenClaw AI Configuration Files*. [https://www.esecurityplanet.com/threats/infostealers-target-openclaw-ai-configuration-files/](https://www.esecurityplanet.com/threats/infostealers-target-openclaw-ai-configuration-files/)

[6] Adversa AI Blog. *OpenClaw security guide 2026: CVE-2026-25253, Moltbook breach & hardening*. [https://adversa.ai/blog/openclaw-security-101-vulnerabilities-hardening-2026/](https://adversa.ai/blog/openclaw-security-101-vulnerabilities-hardening-2026/)

[7] OpenClaw Docs. *Windows (WSL2)*. [https://docs.openclaw.ai/platforms/windows](https://docs.openclaw.ai/platforms/windows)
