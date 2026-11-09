# Distributed Model Context Protocol (MCP) AI System (n8n Workflows)

A distributed, microservice-style AI automation architecture built in [n8n](https://n8n.io/) using the **Model Context Protocol (MCP)** standard. This project separates tool provisioning from conversational intelligence: one workflow acts as a centralized **MCP Tool Server**, while the second operates as an **MCP Client AI Agent** consuming remote tools on demand over Server-Sent Events (SSE).

---

## 🏗 Distributed System Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                        WORKFLOW 1: MCP SERVER                          │
│                                                                        │
│               ┌──────────────────────────────────────┐                 │
│               │   MCP Server Trigger (SSE Endpoint)  │                 │
│               │         Path: /mcp/mcp-server        │                 │
│               └──────────────────┬───────────────────┘                 │
│                                  │                                     │
│     ┌──────────────┬─────────────┼──────────────┬──────────────┐       │
│     ▼              ▼             ▼              ▼              ▼       │
│  [Gmail]     [Google Sheets] [Google Docs]  [Calendar]     [SerpApi]   │
│ (Send Email)  (Get & Append) (Create/Update) (Schedule) (Google/Amazon)│
└──────────────────────────────────▲─────────────────────────────────────┘
                                   │
                                   │ SSE Network Transport
                                   │ (/mcp/mcp-server/sse)
                                   │
┌──────────────────────────────────┴─────────────────────────────────────┐
│                     WORKFLOW 2: MCP CLIENT AGENT                       │
│                                                                        │
│                       [Chat Trigger (UI/API)]                          │
│                                  │                                     │
│                                  ▼                                     │
│                        [ LangChain AI Agent ]                          │
│                        (OpenAI gpt-4o-mini)                            │
│                                  │                                     │
│                    ┌─────────────┴─────────────┐                       │
│                    ▼                           ▼                       │
│           [ Simple Memory ]            [ MCP Client Tool ]             │
│            (Window Buffer)             (Remote Tool Proxy)             │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 📦 Workflows Included

### 1. `MCP Server Workflow.json` (Tool Host)
Exposes local workspace integrations and utilities as standard MCP tools via an HTTP/SSE server endpoint (`@n8n/n8n-nodes-langchain.mcpTrigger`).

* **Endpoint Path:** `/mcp-server`
* **Transport:** Server-Sent Events (SSE)
* **Connected Remote Tools:**
  * **Gmail Tool:** Sends dynamic emails with AI-overridden parameters (`To`, `Subject`, `Message`).
  * **Google Sheets Tools (2 Nodes):** Reads (`Get`) and writes (`Append`) spreadsheet records.
  * **Google Docs Tools (2 Nodes):** Creates new documents in Google Drive and updates existing files with inserted text.
  * **Google Calendar Tool:** Creates events and checks calendar schedules.
  * **Calculator Tool:** Handles arithmetic and mathematical computations.
  * **SerpApi Google Search:** Fetches real-time web search results.
  * **SerpApi Amazon Search:** Executes product lookup queries specifically targeted at `amazon.in`.

### 2. `AI Agent - MCP enabled.json` (Client Consumer)
An autonomous conversational agent powered by OpenAI (`gpt-4o-mini`) that connects to the remote MCP server, automatically discovering and calling remote tools.

* **Trigger:** Web Chat Trigger (`chatTrigger`).
* **Memory:** Window Buffer Memory (`memoryBufferWindow`) preserving chat context across multi-turn sessions.
* **MCP Client Integration:** Connects to the server's SSE URL and filters only the tools needed for execution (`Amazon_search_in_SerpApi`, `Google_search_in_SerpApi`, `Calculator`, `Gmail`).

---

## ⚡ Key Advantages of This Architecture

* **Decoupled Security & Credentials:** Authentication credentials (Google OAuth2, SerpApi tokens) live strictly on the MCP Server workflow. The Client workflow only needs access to the LLM and the server URL.
* **Modular Reusability:** Multiple agents, frontends, or external applications (e.g., Claude Desktop, custom chat interfaces) can subscribe to the same n8n MCP Server simultaneously.
* **Fine-Grained Permissions:** The MCP Client selects and authorizes only specific subsets of tools, preventing the LLM from unauthorized tool misuse.

---

## 📋 Credentials & Prerequisites

| Integration | Credential Type in n8n | Target Node / Location |
|---|---|---|
| **OpenAI** | `openAiApi` | Client: `OpenAI Chat Model` (`gpt-4o-mini`) |
| **Gmail** | `gmailOAuth2` | Server: `Gmail` node |
| **Google Sheets** | `googleSheetsOAuth2Api` | Server: `Get` & `Append` nodes |
| **Google Docs** | `googleDocsOAuth2Api` | Server: `Create` & `Update` nodes |
| **Google Calendar** | `googleCalendarOAuth2Api` | Server: `Create1` node |
| **SerpApi** | `serpApi` | Server: `Google Search` & `Amazon Search` nodes |

---

## 🚀 Setup & Deployment

### Step 1: Deploy the MCP Server Workflow
1. In n8n, create a new workflow and import `MCP Server Workflow.json`.
2. Connect your credentials for Google Sheets, Docs, Calendar, Gmail, and SerpApi.
3. Toggle the workflow to **Active**.
4. Note your public webhook URL for the MCP trigger. It follows this structure:
   ```text
   https://<your-n8n-domain>/mcp/mcp-server/sse
   ```

### Step 2: Deploy the MCP Client Workflow
1. In n8n, create a second workflow and import `AI Agent - MCP enabled.json`.
2. Attach your **OpenAI API** credentials to the `OpenAI Chat Model` node.
3. Open the **MCP Client1** node:
   * Verify the `endpointUrl` matches your active MCP Server SSE URL from Step 1.
   * Under `includeTools`, review the enabled capabilities (`Amazon_search_in_SerpApi`, `Google_search_in_SerpApi`, `Calculator`, `Gmail`).
4. Toggle the workflow to **Active**.

---

## 💬 Testing the System

Open the built-in n8n chat modal on the **AI Agent - MCP enabled** workflow and issue multi-step instructions:

1. **Calculations:**
   > *"What is (450 * 12) / 4?"*
   > *(Calls the remote Calculator tool on the MCP server and returns the result.)*

2. **Web & Product Research:**
   > *"Find the price and name of top-selling noise cancelling headphones on Amazon India."*
   > *(Calls the remote Amazon SerpApi tool on the MCP server and summarizes findings.)*

3. **External Actions:**
   > *"Send an email to team@example.com summarizing the headphone recommendations we just found."*
   > *(Calls the remote Gmail tool on the MCP server using parameters retrieved from session memory.)*