# 🦾 OpenClaw & NemoClaw: Comprehensive Guide and Interview Prep
> Created for: Saad · Date: July 12, 2026

This guide contains everything you need to know about **OpenClaw** and **NemoClaw** architectures, configurations, security models, and interview preparation. 

---

## 📌 Part 1: High-Level Overview

### 1. What is OpenClaw?
**OpenClaw** is an open-source, multi-channel AI agent gateway. It is designed to act as a personal or enterprise agent host that:
* **Interacts across multiple channels:** Integrates directly with Discord, Telegram, WhatsApp, Slack, Matrix, and SMS.
* **Maintains state and memory:** Uses structured Markdown files (`AGENTS.md`, `SOUL.md`, `MEMORY.md`) to maintain persistent context, vibe, and memories across restarts.
* **Runs local tools & skills:** Executes custom Node.js and Python scripts (skills) for tasks like reading emails, querying calendars, controlling devices, or searching the web.

### 2. What is NemoClaw?
**NemoClaw** is NVIDIA’s secure distribution of OpenClaw. It adds enterprise-grade sandboxing and security features:
* **OpenShell Sandbox:** Runs the agent execution environment inside secure Docker containers (or WSL2 on Windows) to prevent malicious code from infecting the host system.
* **Inference Key Protection:** Ensures API keys (like OpenRouter or Google Gemini keys) are stored securely on the host and are never exposed inside the container runtime.
* **Network Policies:** Restricts outbound internet access from the sandbox based on tiers (`restricted`, `balanced`, `permissive`).

---

## 🏗️ Part 2: Technical Architecture & Flow

### 1. The Gateway Model
OpenClaw runs as a Node.js background service (often as a Windows Scheduled Task or systemd service) listening on a local port (default: `18789`). It exposes:
* **WebSocket / JSON-RPC API:** Used by the Control UI (Dashboard) and terminal UI (TUI).
* **OpenAI-Compatible Endpoints:** `/v1/chat/completions` and `/v1/models`.

### 2. Sandbox Execution Flow
When a tool or agent prompt is executed:
```
[User Chat Input] ──> [OpenClaw Gateway (Host)]
                             │
                             ▼ (Launch Command)
                [NemoClaw Sandbox (Container)]
                             │
       (Wants LLM)           ▼
  [Agent Code] ──> [inference.local (Proxy)] 
                             │
                             ▼ (Intercept & Attach API Keys)
                 [OpenClaw Gateway (Host)]
                             │
                             ▼ (Outbound Request)
            [Inference Provider (OpenRouter/Google)]
```

* **Safety:** The agent inside the container thinks it's talking to `inference.local`. It has no access to the actual `sk-or-v1-...` API key, ensuring security even if the agent is compromised.

---

## ⚙️ Part 3: Workspace & Persona Configurations

The workspace (located at `~/.openclaw/workspace/`) contains critical `.md` files that dictate the agent's behavior. Modifying these directly alters the agent's personality and tools.

| File | Purpose | Key Concept |
|---|---|---|
| **`IDENTITY.md`** | Defines the basic identity details of the agent (Name: `Junior Saad`, creature type, emoji, and avatar image). | Altering this changes the agent's name and how it introduces itself. |
| **`SOUL.md`** | Defines the core personality, vibe, rules, and communication boundaries (concise vs thorough, opinionated, etc.). | This is the core system prompt override. It shapes the conversational tone. |
| **`USER.md`** | Stores profile information about you, the user (Name: `Saad`, timezone: `Asia/Karachi`, context, preferences, and active projects). | Grounding file. The agent refers to this to customize help and schedule events. |
| **`TOOLS.md`** | A local notes file storing machine-specific variables (SSH endpoints, camera IPs, voice profiles) separate from reusable skills. | Changing this changes the variables utilized by the agent's tools. |
| **`HEARTBEAT.md`** | List of periodic checklists (e.g. checking inbox, weather) run automatically by the gateway background thread. | If empty or comments-only, periodic polls are skipped to prevent token burn. |
| **`AGENTS.md`** | The "Operational Manual" containing guidelines for memory maintenance, WhatsApp formatting, and execution red-lines. | Dictates general rules of action, output formatting, and command approval requirements. |

---

## 🧠 Part 4: Memory System
OpenClaw has a hybrid memory system to balance storage efficiency and security:
1. **Daily Logs (`memory/YYYY-MM-DD.md`):** Raw logs detailing decisions, commands run, and occurrences during that day.
2. **Long-Term Memory (`MEMORY.md`):** Curated, distilled insights. 
   * **Security Rule:** `MEMORY.md` is strictly loaded only in direct, private sessions with the user. It is **never** loaded in group chats (like Discord or Slack groups) to prevent leaking private user details.

---

## 💼 Part 5: Interview Q&A Preparation

### Q1: What is the main security risk of running autonomous AI agents, and how does NemoClaw solve it?
* **Answer:** The biggest risk is **arbitrary code execution (ACE)** and **data exfiltration**. If an agent is given shell/file write tools and downloads a malicious script (or generates bad code), it can compromise the host machine. 
* **NemoClaw's Solution:** It wraps the agent inside a Docker/WSL container (sandbox) and applies network access policies. The host's files are protected, and API keys are stored on the host, proxied through `inference.local` so the containerized agent cannot access or leak them.

### Q2: How does OpenClaw handle credentials and third-party API integration safely?
* **Answer:** Through **SecretRef-backed credentials**. Credentials are configured on the gateway host level in `openclaw.json` (such as OpenRouter keys or Google API keys). The gateway runs an authorization interceptor. When the container agent sends a request, the gateway validates the query, mounts the keys at the request time, and sends it to the target API without storing them in the execution sandbox.

### Q3: Explain the difference between Cron Tasks and Heartbeat Tasks in OpenClaw.
* **Answer:**
  * **Heartbeat Tasks:** Best for batching periodic checks (like calendar/email) that can drift in time. It reduces token consumption by combining multiple checks into one agent turn.
  * **Cron Tasks:** Best for precise schedules (e.g., "Every Monday at 9:00 AM sharp"). They run in isolated sessions with specific thinking levels, independent of main chat history.

### Q4: If an agent runs a command that requires user verification (like deleting a database or sending an email), how does OpenClaw handle this?
* **Answer:** It uses the **Execution Approval Gateway**. The gateway pauses execution and puts the action in the `execApprovalQueue`. It alerts the user via the Dashboard or CLI (e.g. `openclaw approvals list`). The agent cannot proceed until the user issues an approval decision (`allow-once`, `allow-always`, or `deny`).

### Q5: How do you configure a custom OpenAI-compatible endpoint (like OpenRouter) for OpenClaw?
* **Answer:** In `~/.openclaw/openclaw.json`, you define a custom provider under `models.providers` with the base URL, authentication type, and key:
  ```json
  "models": {
    "mode": "merge",
    "providers": {
      "custom-openrouter-ai": {
        "baseUrl": "https://openrouter.ai/api/v1",
        "api": "openai-completions",
        "apiKey": "sk-or-v1-YOUR_KEY",
        "models": [
          {
            "id": "deepseek/deepseek-v4-flash",
            "name": "DeepSeek V4 Flash (OpenRouter)",
            "contextWindow": 128000,
            "maxTokens": 8192
          }
        ]
      }
    }
  }
  ```
  Then, you set the default primary model to `"custom-openrouter-ai/deepseek/deepseek-v4-flash"` in `agents.defaults.model.primary`.

---

## ⚡ Part 6: Essential Commands Reference

* **Start Gateway:** `openclaw gateway start`
* **Restart Gateway:** `openclaw gateway restart`
* **Check Status:** `openclaw status` / `openclaw status --deep`
* **Check Configured Models:** `openclaw models list`
* **Check Approvals:** `openclaw approvals list`
* **Check Logs:** `openclaw logs --follow`
* **Open Dashboard:** `openclaw dashboard`
* **Run Doctor Fix:** `openclaw doctor --fix`
