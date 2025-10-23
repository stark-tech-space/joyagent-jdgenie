# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

JoyAgent-JDGenie is an open-source, production-ready multi-agent AI system developed by JD.com. Unlike other agent frameworks (which are SDKs requiring further development), this is a complete end-to-end product with:
- Full React frontend UI
- Spring Boot backend for agent orchestration
- Python FastAPI services for specialized tools
- Pre-built core agents (Report, Search, Code, File, Data Analysis)
- DataAgent capabilities for structured data analysis
- Multi-modal RAG support

**Branch Note**: The `data_agent` branch includes DataAgent features with DGP protocol, intelligent Q&A, and diagnostic analysis capabilities.

## Architecture

### Three-Layer System
```
┌─────────────────────────────────────────────┐
│  Frontend (ui/)                             │
│  React + TypeScript + Vite                  │
│  Port: 3000                                 │
└──────────────────┬──────────────────────────┘
                   │ HTTP/SSE
┌──────────────────▼──────────────────────────┐
│  Backend (genie-backend/)                   │
│  Spring Boot + Java 17                      │
│  Multi-agent orchestration                  │
│  Port: 8080                                 │
└──────────────────┬──────────────────────────┘
                   │ HTTP
┌──────────────────▼──────────────────────────┐
│  Python Tools (genie-tool/)                 │
│  FastAPI + Python 3.11                      │
│  Report generation, search, analysis        │
│  Port: 1601                                 │
└─────────────────────────────────────────────┘
```

Additional service:
- **genie-client/** (Port 8188): MCP (Model Context Protocol) client wrapper

### Key Directories

**Frontend: `ui/`**
- `src/components/` - React components (ChatView, ActionView, PlanView)
- `src/services/` - API communication with backend
- `src/utils/` - SSE streaming, chat utilities
- Build tool: Vite 6.1, pnpm for package management

**Backend: `genie-backend/`**
- `src/main/java/com/jd/genie/agent/` - Core agent implementations
  - `agent/` - BaseAgent, ReactImplAgent, PlanningAgent, ExecutorAgent
  - `tool/` - Built-in tools (CodeInterpreter, Report, File, Search, DataAnalysis)
  - `llm/` - LLM configuration and token counting
  - `prompt/` - Prompt templates
- `src/main/java/com/jd/genie/data/` - DataAgent components
  - `jdbc/` - Multi-database support (MySQL, H2, ClickHouse)
  - `sql/` - NL2SQL parsing and generation
- `src/main/java/com/jd/genie/controller/` - REST endpoints
- `src/main/resources/application.yml` - **Critical configuration file**

**Python Tools: `genie-tool/`**
- `genie_tool/api/` - FastAPI route handlers
- `genie_tool/tool/` - Tool implementations (search, table RAG, analysis)
- `genie_tool/prompt/` - Prompt templates
- `genie_tool/db/` - Database operations and file storage
- `.env_template` - Environment variables template (copy to `.env`)

**MCP Client: `genie-client/`**
- Wrapper for MCP protocol servers
- Provides `/v1/tool/list` and `/v1/tool/call` endpoints

## Development Commands

### Initial Setup

**Prerequisites:**
- Java 17
- Python 3.11+
- Node.js 20+
- pnpm
- uv (Python package installer: `pip install uv`)

**Configuration (REQUIRED before first run):**
1. Backend LLM settings: `genie-backend/src/main/resources/application.yml`
   - Update `llm.default.base_url`, `apikey`, `model`, `max_tokens`
   - For DeepSeek: set `max_tokens: 8192` for `deepseek-chat`

2. Python tools environment: Copy `genie-tool/.env_template` to `genie-tool/.env`
   - Set `OPENAI_API_KEY`, `OPENAI_BASE_URL`, `DEFAULT_MODEL`
   - Set `SERPER_SEARCH_API_KEY` (get from https://serper.dev/)
   - For DeepSeek: set `DEEPSEEK_API_KEY`, `DEEPSEEK_API_BASE`, `DEFAULT_MODEL=deepseek/deepseek-chat`

### Building & Running

**Option 1: Docker (Recommended)**
```bash
# Build image
docker build -t genie:latest .

# Run container
docker run -d -p 3000:3000 -p 8080:8080 -p 1601:1601 --name genie-app genie:latest

# Access UI
# http://localhost:3000
```

**Option 2: One-command Local Startup (Recommended for development)**
```bash
# Check dependencies and ports
sh check_dep_port.sh

# Start all services (Ctrl+C to stop)
sh Genie_start.sh
```

This script will:
- Check configuration files
- Build backend (Maven)
- Initialize tool service database
- Create Python virtual environments
- Start all services (frontend, backend, tools, MCP client)
- Display progress and service URLs

**Option 3: Manual Service Management**
```bash
# Backend
cd genie-backend
sh build.sh              # Build with Maven
sh start.sh              # Start backend service

# Python Tools
cd genie-tool
uv sync                  # Install dependencies in .venv
source .venv/bin/activate
python -m genie_tool.db.db_engine  # Initialize database
sh start.sh              # Start tool service

# Frontend
cd ui
pnpm install
pnpm dev                 # Development mode
pnpm build               # Production build

# MCP Client
cd genie-client
uv venv
source .venv/bin/activate
uv sync
sh start.sh
```

### Testing

**Backend:**
```bash
cd genie-backend
mvn test
```

**Frontend:**
```bash
cd ui
pnpm test
```

**Python Tools:**
```bash
cd genie-tool
source .venv/bin/activate
pytest  # if tests exist
```

### Development Workflow

**Working on Frontend:**
- Hot reload enabled with Vite
- API endpoints configured in `ui/.env`
- SSE streaming in `src/utils/sse.ts`

**Working on Backend:**
- Restart required after Java changes: `sh start.sh`
- Check logs: `tail -f genie-backend/genie-backend_startup.log`
- Main entry: `GenieController.java`

**Working on Python Tools:**
- Restart required: `sh start.sh`
- Main entry: `genie-tool/server.py`
- Use `uv sync` after changing `pyproject.toml`

## Key Configuration Files

### Backend: `genie-backend/src/main/resources/application.yml`

Critical sections:
```yaml
llm:
  default:
    base_url: 'https://api.openai.com/v1'
    apikey: 'your-key-here'
    model: 'gpt-4.1'
    max_tokens: 16384

autobots:
  autoagent:
    planner:
      system_prompt: {...}
      model_name: gpt-4.1
    executor:
      system_prompt: {...}
      model_name: gpt-4.1
    tool:
      code_agent:
        desc: 'Code interpreter tool'
        params: {...}

spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/tenant_ftt_prod6
    username: root
    password: testpass123
```

**Important URLs in config:**
- `code_interpreter_url: "https://joyagent-llm.datamunger.io"`
- `deep_search_url: "https://joyagent-llm.datamunger.io"`
- `mcp_client_url: "http://127.0.0.1:8188"`
- `mcp_server_url: "https://mcp.api-inference.modelscope.net/..."`

### Python Tools: `genie-tool/.env`

```bash
OPENAI_API_KEY=your-api-key
OPENAI_BASE_URL=https://api.openai.com/v1
DEFAULT_MODEL=gpt-4
SERPER_SEARCH_API_KEY=your-serper-key

# For DeepSeek
DEEPSEEK_API_KEY=your-deepseek-key
DEEPSEEK_API_BASE=https://api.deepseek.com
```

### Frontend: `ui/.env`

```bash
VITE_API_BASE_URL=http://localhost:8080
```

## Agent System Architecture

### Agent Modes

1. **React Agent** - Single-step reasoning and action (fast, lightweight)
2. **Plan-Execute Pattern** - Multi-step: PlanningAgent → ExecutorAgent
3. **ReAct Pattern** - Traditional Reasoning-Acting-Observing loop

### Built-in Tools

Located in `genie-backend/src/main/java/com/jd/genie/agent/tool/common/`:
- **CodeInterpreterTool** - Python code execution
- **ReportTool** - HTML/PPT/Markdown report generation
- **FileTool** - File read/write operations
- **DeepSearchTool** - Internet search
- **DataAnalysisTool** - Data analysis on structured data
- **PlanningTool** - Task planning and decomposition

### Communication Flow

1. User sends query via frontend (React)
2. Backend receives via GenieController SSE endpoint
3. Agent orchestration layer determines agent mode
4. Agents use tools (local or via HTTP to Python services)
5. Results stream back via SSE to frontend
6. Frontend displays conversation, plans, files, and results

## Adding Custom Components

### Adding a Custom Tool

1. Implement `BaseTool` interface:
```java
// genie-backend/src/main/java/com/jd/genie/agent/tool/
public class WeatherTool implements BaseTool {
    @Override
    public String getName() {
        return "agent_weather";
    }

    @Override
    public String getDescription() {
        return "查询天气的智能体";
    }

    @Override
    public Map<String, Object> toParams() {
        // Return JSON schema for tool parameters
    }

    @Override
    public Object execute(Object input) {
        // Tool implementation
        return "今日天气晴朗";
    }
}
```

2. Register in `GenieController#buildToolCollection`:
```java
WeatherTool weatherTool = new WeatherTool();
toolCollection.addTool(weatherTool);
```

3. Restart backend: `sh start.sh`

### Adding MCP Tools

1. Update `application.yml`:
```yaml
mcp_server_url: "http://ip1:port1/sse,http://ip2:port2/sse"
```

2. Restart services: `sh start_genie.sh`

3. MCP tools automatically discovered and available to agents

## Database Configuration

### MySQL (Application Database)
- Required for backend operation
- Schema: `tenant_ftt_prod6` (configurable)
- Tables auto-created by MyBatis Plus

### Qdrant (Vector Database)
- Optional, for semantic search
- Configuration in `application.yml` under `autobots.data-agent.qdrantConfig`

### Elasticsearch
- Optional, for enhanced search
- Configuration in `application.yml` under `autobots.data-agent.es-config`

## Common Issues

### Service Won't Start
1. Check configuration files for placeholder values like `<input llm server here>`
2. Verify ports 3000, 8080, 1601, 8188 are not in use
3. Check logs: `genie-backend/genie-backend_startup.log`
4. Run `sh check_dep_port.sh` to diagnose

### Database Connection Errors
- Ensure MySQL is running and accessible
- Verify credentials in `application.yml`
- Check database schema exists

### LLM API Errors
- Verify API key is valid
- Check `base_url` is correct
- For DeepSeek, ensure `max_tokens: 8192`
- Review model name matches provider

### Python Service Errors
- Activate venv: `source genie-tool/.venv/bin/activate`
- Check `.env` file exists and is configured
- Reinstall dependencies: `uv sync`

## Performance Notes

- **SSE Streaming**: Full-pipeline streaming from LLM → Backend → Frontend
- **DAG Execution**: High-concurrency execution engine for parallel tool calls
- **Token Management**: Built-in token counter prevents LLM context overflow
- **Caching**: Tool results cached during conversation session

## Project Structure Pattern

When adding new features:
- **Frontend components** → `ui/src/components/[FeatureName]/`
- **Backend agents** → `genie-backend/src/main/java/com/jd/genie/agent/agent/`
- **Backend tools** → `genie-backend/src/main/java/com/jd/genie/agent/tool/common/`
- **Python tools** → `genie-tool/genie_tool/tool/`
- **API endpoints** → `genie-backend/src/main/java/com/jd/genie/controller/`

## Multi-Language Development

This is a polyglot codebase:
- **Java 17** (backend) - Spring Boot, Maven
- **Python 3.11+** (tools) - FastAPI, uv package manager
- **TypeScript** (frontend) - React, Vite, pnpm

When working across languages:
- Backend ↔ Python: HTTP REST APIs
- Frontend ↔ Backend: REST + SSE
- Changes to tool APIs require updates in both backend tool classes and Python implementations
