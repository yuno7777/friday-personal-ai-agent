# F.R.I.D.A.Y. — Stark Industries Voice Agent
> **Fully Responsive Intelligent Digital Assistant for You**

[![Python Version](https://img.shields.io/badge/Python-3.11+-blue.svg)](https://www.python.org/)
[![FastMCP](https://img.shields.io/badge/FastMCP-Server-blue.svg)](https://github.com/jlowin/fastmcp)
[![LiveKit](https://img.shields.io/badge/LiveKit-Agents-orange.svg)](https://github.com/livekit/agents)
[![Gemini](https://img.shields.io/badge/Gemini-2.5_Flash-green.svg)](https://ai.google.dev/)
[![License](https://img.shields.io/badge/License-MIT-brightgreen.svg)](LICENSE)

F.R.I.D.A.Y. is a high-fidelity, Tony Stark-inspired AI voice assistant designed around a modular architecture. The system is split into two primary components: a real-time voice streaming pipeline powered by LiveKit Agents, and a system-control backend powered by a Model Context Protocol (MCP) server. They communicate locally via Server-Sent Events (SSE) to orchestrate system diagnostics, fetch real-time global intelligence briefs, and manage visual dashboards.

---

## Architectural Highlights

> [!IMPORTANT]  
> **Unified API Credentials**  
> F.R.I.D.A.Y. uses Google Gemini for both cognitive reasoning (LLM) and voice generation (TTS). The entire pipeline runs using a single standard Google AI Studio key (`GOOGLE_API_KEY`). You do not need active OpenAI credentials or Google Cloud Platform Service Account JSON files.

---

## System Architecture

```
                  ┌────────────────────────────────────────┐
                  │          Audio Stream (Microphone)     │
                  └───────────────────┬────────────────────┘
                                      │
                                      ▼
                  ┌────────────────────────────────────────┐
                  │        STT (Sarvam Saaras v3)          │
                  └───────────────────┬────────────────────┘
                                      │
                                      ▼  (User Transcription)
                  ┌────────────────────────────────────────┐
                  │       LLM (Gemini 2.5 Flash)           │◄───────┐
                  └───────────────────┬────────────────────┘        │ (SSE / HTTP)
                                      │                             ▼
                                      │                    ┌───────────────────┐
                                      │                    │    MCP SERVER     │
                                      ├───────────────────►│   (FastMCP)       │
                                      │ (Silent Invocation)│ ─ get_world_news  │
                                      │                    │ ─ world_monitor   │
                                      ▼                    │ ─ system_info     │
                  ┌────────────────────────────────────────┐└───────────────────┘
                  │  Gemini TTS ("Aoede" Digital Voice)    │
                  └───────────────────┬────────────────────┘
                                      │
                                      ▼
                  ┌────────────────────────────────────────┐
                  │         Audio Stream (Speakers)        │
                  └────────────────────────────────────────┘
```

### The WSL-to-Windows Network Bridge

> [!TIP]  
> **Host Gateway Resolution**  
> When running the Voice Agent inside Windows Subsystem for Linux (WSL), the pipeline automatically resolves the Windows host gateway IP by inspecting `/etc/resolv.conf` or executing `ip route show default` to maintain an SSE connection on port `8000` of the host.

---

## Project Structure

```
friday-tony-stark-demo/
├── server.py           # Starts the FastMCP host server (SSE on port 8000)
├── agent_friday.py     # Starts the LiveKit Voice Agent pipeline
├── pyproject.toml      # Project dependency and script manifest
├── .env.example        # Environment template file
├── .env                # Git-ignored private local environment file
└── friday/             # MCP server package
    ├── config.py       # Dotenv configurations and class environments
    ├── tools/          # System-level capabilities (callable by the LLM)
    │   ├── web.py      # get_world_news, open_world_monitor, search_web, fetch_url
    │   ├── system.py   # get_current_time, get_system_info
    │   └── utils.py    # format_json, word_count
    ├── prompts/        # Contextual prompt templates
    └── resources/      # Static data files and assets
```

---

## Installation & Setup

### 1. Prerequisites
* **Python** >= 3.11
* **LiveKit Cloud** Project (Create a project at [cloud.livekit.io](https://cloud.livekit.io))
* **Google AI Studio API Key** (Generate a key at [aistudio.google.com](https://aistudio.google.com/projects))

### 2. Environment Sync
Initialize your virtual environment and resolve dependencies:

```bash
# Clone the repository
git clone https://github.com/SAGAR-TAMANG/friday-tony-stark-demo.git
cd friday-tony-stark-demo

# Synchronize dependencies and construct the virtual environment
python -m uv sync
```

### 3. Configure Local Credentials
Create a private environment configuration file:

```bash
cp .env.example .env
```

Open `.env` and fill in the required keys:
```ini
# LiveKit Credentials
LIVEKIT_URL=wss://your-project.livekit.cloud
LIVEKIT_API_KEY=your-api-key
LIVEKIT_API_SECRET=your-api-secret

# Google AI Studio API Key (Required for LLM and TTS)
GOOGLE_API_KEY=your-google-api-key

# Optional Credentials (e.g. Sarvam, Groq, Deepgram)
SARVAM_API_KEY=your-sarvam-api-key
```

---

## Execution

> [!NOTE]  
> Both components must run simultaneously. The Voice Agent queries the FastMCP server at start to pull its interactive toolset.

### Terminal 1 — FastMCP Server (Host Backend)
Launch the system-control backend:
```bash
python -m uv run friday
```
*Starts the SSE FastMCP server at `http://127.0.0.1:8000/sse`.*

### Terminal 2 — Voice Agent (Communication Pipeline)
Launch the voice agent pipeline:
```bash
python -m uv run friday_voice
```
*Starts the pipeline and connects it to your LiveKit room. Open the [LiveKit Agents Playground](https://agents-playground.livekit.io), select your project, and start speaking with F.R.I.D.A.Y.*

---

## Component Layout

| Command | File Path | Core Responsibility |
|:---|:---|:---|
| `python -m uv run friday` | [server.py](file:///C:/Users/Abhishek%20Satarkar/Downloads/friday-tony-stark-demo-main/friday-tony-stark-demo-main/server.py) | Launches the SSE FastMCP server on port 8000. Registers and exposes system control, news scraping, and web launching tools to the AI agent. |
| `python -m uv run friday_voice` | [agent_friday.py](file:///C:/Users/Abhishek%20Satarkar/Downloads/friday-tony-stark-demo-main/friday-tony-stark-demo-main/agent_friday.py) | Connects to the real-time audio channel, transcribes using Sarvam, reasons using Gemini LLM, and speaks using native Gemini TTS. |

---

## Adding Custom Capabilities (Tools)

1. Create a function under the `friday/tools/` folder.
2. Decorate it using the `@mcp.tool()` wrapper:
   ```python
   @mcp.tool()
   def trigger_lab_lockdown(level: int) -> str:
       """Initiate Stark Lab security lockdown protocol."""
       return f"Level {level} security protocols engaged. All entryways sealed."
   ```
3. Register the module inside `friday/tools/__init__.py`.
4. Restart your FastMCP server. The agent will automatically recognize the capability on the next conversation loop.

---

## Tech Stack Specifications

* **FastMCP**: SSE-based Model Context Protocol server framework.
* **LiveKit Agents**: Ultra-low-latency real-time voice streaming pipeline.
* **Google Gemini 2.5 Flash**: Cognitive reasoning and processing engine.
* **Google Gemini TTS**: Native speech synthesis powered by the `"Aoede"` digital voice profile.
* **Sarvam Saaras v3**: Indian-English optimized Speech-to-Text.
* **uv**: High-speed Python package and environment manager.

---

## License
MIT License. Authorized Stark Industries Protocol.
