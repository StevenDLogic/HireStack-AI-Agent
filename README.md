# HireStack-AI-Agent

HireStack-AI-Agent is an open-source conversational AI assistant built to interact with HCM (Human Capital Management) systems via API and database operations. Its primary goal is to let users perform common HR workflows using natural language, while keeping control over sensitive actions through an approval mechanism.

---

## Key Features

- **Natural language operations**: Ask the agent questions or issue commands in plain text (e.g., "Find John Doe" or "Schedule a meeting for tomorrow at 10am").
- **Tool-based execution**: The agent uses a tool system to map intent into deterministic actions (search, create, update, delete).
- **Human-in-the-loop approvals**: Potentially destructive operations can be reviewed and approved before execution.
- **Modular tool set**: Add or remove tools in a single place to control agent capabilities.
- **Configurable history management**: Choose between message-based or token-based history handling to manage prompt size.

---

## Architecture Overview

### Core Components

- **Agent server** (`mint_agent/server.py`): HTTP + WebSocket server that accepts chats and drives the agent workflow.
- **Graph nodes** (`mint_agent/agent_graph/nodes`): Each node represents a processing stage (prompt assembly, LLM call, tool validation, tool execution, history pruning).
- **Tool controller** (`mint_agent/tools/ToolController.py`): Defines which tools are available and manages safety checks.
- **LLM controllers** (`mint_agent/llm`): Responsible for communicating with supported LLM providers.
- **Credential manager** (`mint_agent/agent_api/CredentialManager.py`): Stores and retrieves API credentials used to access MintHCM.

### Conversation Flow (high-level)

1. Client sends user message via WebSocket.
2. The conversation is routed through graph nodes:
   - Prompt is constructed (system + history + user message).
   - LLM generates a response and potentially selects a tool.
   - Tool selection is validated against a safety list.
   - If a tool requires approval, it is flagged for human review.
   - Approved tools execute actions against MintHCM.
   - Results are returned to the client.

---

## Tools (Capabilities)

Tools are implemented as self-contained modules in `mint_agent/tools`.
They can be grouped into categories like read-only queries or state-changing actions.

### Primary Tool Categories

- **Search / Read**
  - `MintSearchTool` — query records in a module.
  - `MintGetUsersTool` — list system users.
  - `MintGetModuleNamesTool` / `MintGetModuleFieldsTool` — inspect available modules and fields.
  - `CalendarTool` / `AvailabilityTool` — provide local date context and scheduled items.

- **Create / Update / Delete**
  - `MintCreateMeetingTool` — schedule meetings.
  - `MintCreateRecordTool` — create generic records.
  - `MintUpdateFieldsTool` — update fields on existing records.
  - `MintDeleteRecordTool` — delete records.
  - Relationship tools (`MintCreateRelTool`, `MintDeleteRelTool`, `MintGetRelTool`) — manage relational links between records.

---

## Configuration

### Tool Lists

The tool controller defines three lists:

- `available_tools` — all tool implementations present in the system.
- `default_tools` — tools enabled by default for every session.
- `safe_tools` — subset of tools that can run without additional approval.

Adjust these lists by editing `mint_agent/tools/ToolController.py`.

### Prompts

System prompts and tool prompts are managed in `mint_agent/prompts/PromptController.py`.
Modify prompt templates there to change how the agent is instructed.

### History Management

The agent supports two main approaches to pruning conversation history:

- **Message-based**
  - `KEEP_N_MESSAGES`: Keep only the N most recent messages.
  - `SUMMARIZE_N_MESSAGES`: Summarize older messages when the history grows.

- **Token-based** (Anthropic only)
  - `KEEP_N_TOKENS`: Keep only the most recent tokens in conversation history.
  - `SUMMARIZE_N_TOKENS`: Summarize older messages to stay within a token budget.

Configure these modes in the history manager node (`mint_agent/agent_graph/nodes/history_manager.py`).

---

## Setup & Running

### 1) Prerequisites

- Python 3.11+ (or compatible version) with a virtual environment.
- A running MongoDB instance.
- MintHCM instance with API credentials.

### 2) Install Dependencies

From the repository root:

```sh
poetry install
```

### 3) Environment Configuration

Copy the example env file and fill in required values:

```sh
cp .env_example .env
```

### 4) Generate Encryption Key

```sh
poetry run generate_key
```

### 5) Add MintHCM Credentials

```sh
poetry run generate_credentials
```

Follow prompts to provide:
- MintHCM username
- Mint user ID
- client_id
- client secret

### 6) Run the Server

Development mode (auto-reload):

```sh
poetry run dev
```

Production mode:

```sh
poetry run prod
```

### 7) Optional: Test Chat UI

Run the local test chat interface (may require adjusting websocket address in `mint_agent/utils/chat.html`):

```sh
poetry run test_chat
```

---

## Contributing

- Add new tools under `mint_agent/tools/` and register them in `ToolController`.
- Update prompts in `mint_agent/prompts/PromptController.py`.
- Ensure new features include unit tests where appropriate.

---

## Notes

- The agent is designed as a framework; it is not tied to a single HCM product. MintHCM is used as a reference implementation.
- Some operations (especially those that change data) should always be reviewed before committing.
- The system is built with extensibility in mind, so you can add integrations or replacement LLM backends as needed.
