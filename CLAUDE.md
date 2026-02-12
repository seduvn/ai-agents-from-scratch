# CLAUDE.md

## Project Overview

**AI Agents From Scratch** is an educational JavaScript project that teaches how to build AI agents from first principles using local LLMs via `node-llama-cpp`. It has two phases:

- **Phase 1 (Examples):** 10 progressive examples from basic LLM usage to ReAct agents
- **Phase 2 (Framework + Tutorial):** A reimplementation of LangChain/LangGraph core concepts with step-by-step lessons

The project uses **no build system** and runs plain ES module JavaScript directly with Node.js.

## Quick Reference

```bash
# Install dependencies
npm install

# Run any example
node examples/01_intro/intro.js
node examples/07_simple-agent/simple-agent.js
node examples/09_react-agent/react-agent.js

# Download a model (requires models/ directory)
npx --no node-llama-cpp pull --dir ./models hf:Qwen/Qwen3-1.7B-GGUF:Q8_0 --filename Qwen3-1.7B-Q8_0.gguf
```

## Project Structure

```
ai-agents-from-scratch/
├── examples/                  # Phase 1: 10 progressive working examples
│   ├── 01_intro/              # Basic LLM interaction
│   ├── 02_openai-intro/       # OpenAI API usage (optional)
│   ├── 03_translation/        # System prompts & specialization
│   ├── 04_think/              # Reasoning patterns
│   ├── 05_batch/              # Parallel processing
│   ├── 06_coding/             # Streaming & token control
│   ├── 07_simple-agent/       # Function calling (tools)
│   ├── 08_simple-agent-with-memory/  # Persistent state
│   ├── 09_react-agent/        # ReAct pattern
│   └── 10_aot-agent/          # Atom of Thought planning
├── src/                       # Phase 2: Educational framework (~3100 lines)
│   ├── core/                  # Runnable, Message types, RunnableConfig
│   ├── llm/                   # LlamaCppLLM, ChatModel, StreamingLLM
│   ├── prompts/               # PromptTemplate, ChatPromptTemplate, FewShot, Pipeline
│   ├── output-parsers/        # String, JSON, Structured, List, Regex parsers
│   ├── chains/                # LLMChain, SequentialChain, RouterChain, MapReduce
│   ├── tools/                 # BaseTool, ToolExecutor, built-in tools
│   ├── agents/                # AgentExecutor, ReActAgent, ToolCallingAgent
│   ├── memory/                # Buffer, Window, Summary, Vector, Entity memory
│   ├── graph/                 # StateGraph, MessageGraph, checkpointing
│   ├── utils/                 # Callbacks, TokenCounter, Retry, Logger
│   └── index.js               # Public API (all exports)
├── tutorial/                  # Step-by-step lessons for Phase 2
│   ├── 01-foundation/         # Runnable, Messages, LLM wrapper, Context
│   ├── 02-composition/        # Prompts, Parsers, Chains, Memory
│   └── projects/              # Capstone projects
├── helper/                    # Debugging utilities
│   ├── prompt-debugger.js     # Shows exactly what the LLM receives
│   └── json-parser.js         # JSON parsing helper
├── models/                    # GGUF model files (gitignored, user-provided)
├── .env.example               # OPENAI_API_KEY template (optional)
└── package.json               # ES modules, 3 dependencies
```

## Technology Stack

- **Language:** JavaScript (ES modules)
- **Runtime:** Node.js 18+
- **Core dependency:** `node-llama-cpp` ^3.14.0 (local LLM inference with GGUF models)
- **Optional:** `openai` ^6.7.0 (only for example 02), `dotenv` ^17.2.3
- **No build step, no transpilation, no bundler**

## Key Conventions

### Code Style
- ES module syntax (`import`/`export`) throughout — the project uses `"type": "module"` in package.json
- No linter or formatter is configured; follow existing patterns
- Clear, descriptive variable names
- Comments explain *why*, not just *what*
- Each example is self-contained (one main JS file per example)

### Example Structure
Every example folder (`examples/XX_name/`) contains exactly:
- `<name>.js` — Working, runnable code
- `CODE.md` — Line-by-line code explanation
- `CONCEPT.md` — High-level concepts, patterns, and real-world applications

### Framework Architecture (src/)
The core abstraction is the **Runnable** interface. Every composable component extends `Runnable`:
- `invoke(input, config)` — Process single input
- `stream(input, config)` — Stream output in chunks (async generator)
- `batch(inputs, config)` — Process multiple inputs in parallel
- `pipe(next)` — Compose with another Runnable into a sequence

**Architecture layers** (each in its own `src/` subdirectory):
1. **Core** — `Runnable`, `BaseMessage` types (Human, AI, System, Tool), `RunnableConfig`
2. **LLM** — `BaseLLM` abstract, `LlamaCppLLM` wrapper, `ChatModel`, `StreamingLLM`
3. **Prompts** — Template system with variable interpolation and multi-message support
4. **Output Parsers** — Transform raw LLM output into structured data
5. **Chains** — Compose Prompt + LLM + Parser into pipelines
6. **Tools** — `BaseTool` with `ToolExecutor` for safe execution and `ToolRegistry`
7. **Agents** — Decision-making loops: `ReActAgent`, `ToolCallingAgent`, `ConversationalAgent`
8. **Memory** — Conversation state: `BufferMemory`, `WindowMemory`, `SummaryMemory`, `VectorMemory`
9. **Graph** — State machines: `StateGraph`, `MessageGraph`, conditional edges, checkpointing
10. **Utils** — `CallbackManager`, `TokenCounter`, `RetryManager`, `Logger`, `SchemaValidator`

### Message Types
Typed conversation messages (mirroring LangChain patterns):
- `SystemMessage` — Instructions for the AI
- `HumanMessage` — User input
- `AIMessage` — Assistant responses
- `ToolMessage` — Tool call results

All extend `BaseMessage` and support serialization.

## Development Workflow

### Running Examples
```bash
node examples/<folder>/<name>.js
```
Examples require a GGUF model in `./models/`. See `DOWNLOAD.md` for model download instructions.

### Adding a New Example
1. Create `examples/XX_name/` directory
2. Add the main JS file, `CODE.md`, and `CONCEPT.md`
3. Follow the progressive complexity pattern (each builds on the previous)
4. Keep examples self-contained — import only from `node-llama-cpp` or `src/`

### Modifying the Framework (src/)
- Every module in `src/` has its own `index.js` that re-exports public symbols
- The top-level `src/index.js` aggregates all module exports
- When adding a new class, export it from the appropriate module's `index.js` and from `src/index.js`
- Extend `Runnable` for any new composable component
- Follow the existing pattern: base class with abstract methods, then concrete implementations

## Testing

There is **no automated test suite**. The project uses a manual verification approach:
- Run examples directly and verify output
- The `package.json` test script is a placeholder (`echo "Error: no test specified" && exit 1`)

## Environment & Configuration

- Copy `.env.example` to `.env` and set `OPENAI_API_KEY` if using example 02 (OpenAI)
- All other examples use local models only — no API keys needed
- Models go in `./models/` (gitignored). Recommended: Qwen3-1.7B-Q8_0
- Minimum 8GB RAM (16GB recommended) for local inference

## Files to Never Commit
Per `.gitignore`:
- `models/` — Large GGUF model files
- `node_modules/`
- `.env` — API keys
- `.idea/` — IDE configuration
- `*.txt`, `internal/`, `ui/`, `frontend*`, `VIDEO_SCRIPT.md`
