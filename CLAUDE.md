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

The framework is an educational reimplementation of LangChain/LangGraph patterns. Some layers are fully implemented, others are scaffolded stubs awaiting implementation through the tutorial. The implementation status is noted below for each layer.

#### The Runnable Interface (Foundation)

Every composable component extends the `Runnable` base class (`src/core/runnable.js`):
- `invoke(input, config)` — Process single input, triggers callback lifecycle (start/end/error)
- `stream(input, config)` — Async generator yielding output chunks (default: yields full invoke result)
- `batch(inputs, config)` — Process multiple inputs in parallel via `Promise.all()`
- `pipe(next)` — Returns a new `RunnableSequence([this, next])` for composition

Subclasses must implement `_call(input, config)`. The public `invoke()` wraps it with `CallbackManager` hooks. `RunnableSequence` chains steps where output of one feeds the next; only the final step streams. `RunnableParallel` is exported but not yet implemented.

```javascript
// Composition pattern used throughout
const chain = prompt.pipe(llm).pipe(parser);
const result = await chain.invoke({ topic: "AI agents" });
```

---

#### Layer 1: Core — FULLY IMPLEMENTED

`src/core/` — Runnable, Messages, RunnableConfig

**Runnable classes:**
- `Runnable` — Base class, callback lifecycle in `invoke()`, abstract `_call()`
- `RunnableSequence` — Sequential composition, `pipe()` appends to existing steps
- `RunnableParallel` — Placeholder (throws "not yet implemented")

**Message types** (`src/core/message.js`):
All extend `BaseMessage` with `content`, `additionalKwargs`, `timestamp`, and auto-generated `id`.
- `SystemMessage` (type: `'system'`) — Instructions for the AI; `toPromptFormat()` → `{role: 'system'}`
- `HumanMessage` (type: `'human'`) — User input; `toPromptFormat()` → `{role: 'user'}`
- `AIMessage` (type: `'ai'`) — Assistant responses; has `toolCalls` array, `hasToolCalls()`, `getToolCall(index)`
- `ToolMessage` (type: `'tool'`) — Tool results; has `toolCallId` linking to the originating call

**Message utilities:**
- `messagesToPromptFormat(messages)` — Convert array to LLM-ready format
- `filterMessagesByType(messages, type)` — Filter by message type
- `getLastMessages(messages, n)` — Get last N messages
- `mergeConsecutiveMessages(messages)` — Merge same-type consecutive messages
- `BaseMessage.fromJSON(json)` — Deserialize via `MESSAGE_TYPES` registry

**RunnableConfig** (`src/core/context.js`):
- Fields: `callbacks`, `metadata`, `tags`, `recursionLimit` (default 25), `configurable`
- `merge(other)` — Merges configs (concatenates callbacks/tags, shallow-merges metadata/configurable)
- `child(options)` — Creates child config inheriting parent values

---

#### Layer 2: LLM — LlamaCppLLM FULLY IMPLEMENTED

`src/llm/` — LLM wrappers for local inference

**`LlamaCppLLM`** (extends `Runnable`) — The only fully implemented LLM class:
- **Lazy initialization**: Model loads on first `invoke()`/`stream()`, not at construction
- **Constructor options**: `modelPath` (required), `temperature` (0.7), `topP` (0.9), `topK` (40), `maxTokens` (2048), `repeatPenalty` (1.1), `contextSize` (4096), `batchSize` (512), `verbose`, `stopStrings`, `chatWrapper`
- **Input handling**: Accepts `string` (auto-wrapped in `HumanMessage`) or `Message[]`
- **Config overrides**: `temperature`, `topP`, `topK`, `maxTokens`, `repeatPenalty`, `stopStrings`, `clearHistory`, `seed`
- **Returns**: `AIMessage` instances
- **Streaming**: `onTextChunk` callback + 10ms polling, yields `AIMessage` chunks
- **Batch**: Sequential (not parallel) — local models are CPU-bound; each item gets `clearHistory: true`
- **`dispose()`**: Releases model/context resources — must be called when done

**Stubs** (throw "not yet implemented"):
- `BaseLLM` — Intended abstract base
- `ChatModel` — Chat-specific interface
- `StreamingLLM` — Streaming mixin (streaming is already in LlamaCppLLM)

---

#### Layer 3: Prompts — FULLY IMPLEMENTED

`src/prompts/` — Template system with variable interpolation

All prompt templates extend `BasePromptTemplate` (which extends `Runnable`).

**`BasePromptTemplate`**: Abstract base with `_validateInput()` and `_mergePartialAndUserVariables()`. Supports `partialVariables` for default values.

**`PromptTemplate`**: Simple `{variable}` placeholder interpolation.
- Auto-detects variables from template string via regex `\{(\w+)\}`
- `format(values)` → returns formatted string
- Factory: `PromptTemplate.fromTemplate(templateString)`

**`ChatPromptTemplate`**: Multi-message template returning `Message[]`.
- Takes `promptMessages` as `[role, template]` tuples
- Roles: `"system"`, `"human"`/`"user"`, `"ai"`/`"assistant"`
- `format(values)` → returns array of typed `Message` objects
- Factory: `ChatPromptTemplate.fromMessages([["system", "..."], ["human", "..."]])`

**`FewShotPromptTemplate`**: Few-shot learning with examples.
- Combines `prefix` + formatted `examples` (via `examplePrompt`) + `suffix`
- Examples joined by `exampleSeparator` (default: `'\n\n'`)

**`PipelinePromptTemplate`**: Multi-stage prompt composition.
- `pipelinePrompts`: `[{name, prompt}]` — each prompt's output becomes a variable for the `finalPrompt`
- Auto-collects input variables, excluding internally-generated ones

**`SystemMessagePromptTemplate`**: Always returns a `SystemMessage` object.
- Wraps a `PromptTemplate` internally
- `fromTemplateWithPartials()` for default specialization

---

#### Layer 4: Output Parsers — FULLY IMPLEMENTED

`src/output-parsers/` — Transform raw LLM text into structured data

All parsers extend `BaseOutputParser` (which extends `Runnable`). Each provides `parse(text)` and `getFormatInstructions()` (to include in prompts so the LLM knows the expected format). `_call()` handles both string and `Message` inputs.

- **`StringOutputParser`**: Trims whitespace, optionally strips markdown code blocks (`stripMarkdown: true`)
- **`JsonOutputParser`**: Extracts JSON from text/markdown blocks; optional `schema` for type validation. Multi-strategy extraction: direct parse → markdown blocks → `{...}` patterns → `[...]` patterns
- **`StructuredOutputParser`**: Full schema validation with `responseSchemas` — supports `type`, `description`, `enum`, `required` fields. Factory: `fromNamesAndDescriptions()`
- **`ListOutputParser`**: Auto-detects numbered, bullet, comma-separated, or newline-separated lists. Optional custom `separator`
- **`RegexOutputParser`**: Extracts via regex capture groups; `outputKeys` maps groups to named fields

**Error handling**: `OutputParserException` with `llmOutput` and `originalError` for debugging.

---

#### Layer 5: Chains — STUBS ONLY (not yet implemented)

`src/chains/` — All classes throw "not yet implemented"

Scaffolded classes:
- `BaseChain` — Abstract base for chains
- `LLMChain` — Prompt → LLM → Parser pipeline
- `SequentialChain` — Multiple chains in sequence
- `RouterChain` — Route to different chains based on input
- `MapReduceChain` — Parallel map + reduce for large inputs
- `TransformChain` — Pure data transformation (no LLM)

**Note**: The `pipe()` method on `Runnable` already provides basic chaining (`prompt.pipe(llm).pipe(parser)`). These chain classes are intended to add higher-level orchestration patterns.

---

#### Layer 6: Tools — STUBS ONLY (not yet implemented)

`src/tools/` — All classes throw "not yet implemented"

Scaffolded classes:
- `BaseTool` — Abstract tool interface
- `ToolExecutor` — Safe tool execution with error handling
- `ToolRegistry` — Central tool management

Built-in tools (`src/tools/builtin/`):
- `Calculator`, `WebSearch`, `WebScraper`, `FileReader`, `FileWriter`, `CodeExecutor`

---

#### Layer 7: Agents — STUBS ONLY (not yet implemented)

`src/agents/` — All classes throw "not yet implemented"

Scaffolded classes:
- `BaseAgent` — Abstract agent interface
- `AgentExecutor` — Main agent-tool interaction loop
- `ToolCallingAgent` — Function calling agent
- `ReActAgent` — Reasoning + Acting pattern
- `StructuredChatAgent` — JSON-structured output agent
- `ConversationalAgent` — Multi-turn conversation agent

---

#### Layer 8: Memory — STUBS ONLY (not yet implemented)

`src/memory/` — All classes throw "not yet implemented"

Scaffolded classes:
- `BaseMemory` — Abstract memory interface
- `BufferMemory` — Full conversation buffer
- `WindowMemory` — Sliding window (recent messages only)
- `SummaryMemory` — Auto-summarizing older messages
- `VectorMemory` — Semantic similarity search via embeddings
- `EntityMemory` — Entity extraction and tracking

---

#### Layer 9: Graph — STUBS ONLY (not yet implemented)

`src/graph/` — All classes throw "not yet implemented"

Scaffolded classes:
- `StateGraph` — State machine builder
- `MessageGraph` — Message-centric graph for conversations
- `CompiledGraph` — Executable graph instance
- `GraphNode`, `GraphEdge`, `ConditionalEdge` — Graph primitives
- `Checkpoint` — State snapshot
- `BaseCheckpointer`, `MemoryCheckpointer`, `FileCheckpointer` — Persistence
- `END` constant (`'__end__'`) — Graph termination marker

---

#### Layer 10: Utils — CallbackManager IMPLEMENTED, rest are stubs

`src/utils/` — Cross-cutting utilities

**Implemented:**
- **`CallbackManager`**: Dispatches events (`handleStart`, `handleEnd`, `handleError`, `handleLLMNewToken`, `handleChainStep`) to registered callbacks. Uses `Promise.all()` for parallel dispatch. Error isolation via `_safeCall()`.
- **`BaseCallback`**: Abstract interface with `onStart`, `onEnd`, `onError`, `onLLMNewToken`, `onChainStep`
- **`ConsoleCallback`**: Formatted console logging with optional colors and verbose mode
- **`MetricsCallback`**: Tracks call counts, cumulative time, and errors per Runnable; `getReport()` for summaries
- **`FileCallback`**: Buffers events and `flush()` writes JSON to file

**Stubs** (throw "not yet implemented"):
- `TokenCounter`, `RetryManager`, `TimeoutManager`, `Logger`, `SchemaValidator`

---

#### Implementation Status Summary

| Layer | Module | Status |
|-------|--------|--------|
| 1 | Core (Runnable, Messages, Config) | **Fully implemented** |
| 2 | LLM (LlamaCppLLM) | **Fully implemented** |
| 3 | Prompts (all 5 template types) | **Fully implemented** |
| 4 | Output Parsers (all 5 parsers) | **Fully implemented** |
| 5 | Chains | Stubs only |
| 6 | Tools | Stubs only |
| 7 | Agents | Stubs only |
| 8 | Memory | Stubs only |
| 9 | Graph | Stubs only |
| 10 | Utils (CallbackManager + callbacks) | **Partially implemented** |

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
