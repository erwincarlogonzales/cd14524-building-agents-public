# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Udacity course repository on **Building Agents with Claude** (course cd14524). It is a modular, pedagogical codebase teaching agent development patterns progressively, from basic tool use through advanced RAG and evaluation systems.

**License:** CC BY-NC-ND 4.0 (Udacity Educational Content)
**Ownership:** Udacity (@udacity/public-content-admin)

## Architecture

The repository is structured as **10 progressive modules** plus a **capstone project**, each building on prior concepts. Each module contains:

- `lib/` directory with reusable abstractions
- Jupyter notebooks for demo and student exercises
- Incremental feature introduction

### Core Abstractions (Found in Every Module's `lib/`)

1. **LLM (llm.py)**
   - Wraps OpenAI client with `gpt-4o-mini` or configurable model
   - Manages tool registration and payload construction
   - Methods: `__init__(model, temperature, tools, api_key)`, `register_tool(tool)`, `_build_payload(messages)`, `invoke(input)`
   - Returns `AIMessage` with optional `tool_calls`

2. **Messages (messages.py)**
   - Pydantic `BaseMessage` with `content` and `role` fields
   - Subtypes: `SystemMessage`, `UserMessage`, `AIMessage`, `ToolMessage`
   - Union type `AnyMessage` used throughout
   - Each has `.dict()` method for OpenAI API compatibility

3. **Tooling (tooling.py)**
   - `Tool` class wraps functions with introspection (`inspect.signature`, `get_type_hints`)
   - Auto-generates JSON schema from type hints (supports `Literal`, `Optional`, `List`, `Dict`, primitives)
   - `ToolCall` represents invocation with `tool_name` and `arguments`
   - Tools converted to OpenAI format for `tool_choice="auto"`

4. **State Machine (state_machine.py)** *(Introduced Module 3)*
   - `Step[StateSchema]`: Generic step with `step_id` and `logic` callable
   - `EntryPoint` and `Termination` special steps
   - `Transition`: source → targets list with optional `condition` function
   - `StateMachine[StateSchema]`: orchestrates flow, returns `Run` object with execution history
   - `Resource`: container for shared vars passed to steps
   - Steps receive state (TypedDict-based) and return updated state dict

5. **Memory (memory.py)** *(Introduced Module 4)*
   - `ShortTermMemory`: dataclass managing `sessions` dict
   - Tracks conversation history or objects per session
   - Methods: `create_session()`, `delete_session()`, `append()`, `get_session()`, etc.

6. **Agents (agents.py)** *(Introduced Module 3)*
   - `AgentState` TypedDict defines state shape (messages, scratchpad, etc.)
   - `Agent` class runs agentic loops using state machine + LLM
   - Implements tool-call handling: invoke tool, capture result, append `ToolMessage`
   - Loop terminates when LLM returns `tool_calls=None`

7. **Parsers (parsers.py)** *(Introduced Module 2)*
   - `JSONParser`: extracts JSON from LLM response
   - `XMLParser`: extracts XML blocks
   - Handles markdown code blocks

### Module-Specific Patterns

- **Module 2 (Structured Outputs)**: Parsers for deterministic output
- **Module 5 (External APIs)**: Tools making HTTP requests
- **Module 6 (Web Search)**: Semantic web search tools
- **Module 7 (Databases)**: SQL query execution tools
- **Module 8–9 (RAG)**: `VectorStore`, `RAG` class (retrieve → augment → generate), document loaders
- **Module 10 (Evaluation)**: Agent benchmarking against datasets

### Project (Capstone)

- `project/starter/` contains a multi-game agent
- `lib/agents.py`: Agent implementation for game environments
- `lib/documents.py`: Context/rule loading
- `games/` JSON files: Game scenarios (001–015)
- Notebooks: `Udaplay_01_starter_project.ipynb`, `Udaplay_02_starter_project.ipynb`

## Development Workflow

### Running Code

- **Jupyter Notebooks**: All exercises are `.ipynb` files (demo, starter, solution)
  - Use Jupyter or VS Code's native notebook support
  - Each module references the shared `lib/` for that module
- **Python Scripts**: Direct execution requires `sys.path.append('lib')`
- **API Keys**: OpenAI API key required (`OPENAI_API_KEY` env var or passed to `OpenAI()`)

### Key Import Pattern

Modules use relative imports within notebooks:
```python
from lib.llm import LLM
from lib.messages import UserMessage, AIMessage, SystemMessage
from lib.tooling import Tool
from lib.agents import Agent  # if module >= 3
from lib.state_machine import StateMachine, Step, EntryPoint, Termination  # if module >= 3
```

### Dependency Management

- **No requirements.txt or pyproject.toml**: Course assumes students install OpenAI SDK
  - `pip install openai` (uses `gpt-4o-mini` by default)
- **Optional**: Module 8+ uses Chroma for vector DB (`pip install chroma-db` if extending)

## Code Patterns & Conventions

### TypedDict for State

All stateful agents/systems use TypedDict for type-safe state schemas:
```python
class AgentState(TypedDict):
    messages: List[BaseMessage]
    scratchpad: str
    result: Optional[str]
```

### State Machine Composition

Workflows defined by connecting `Step` instances:
```python
entry = EntryPoint()
step1 = Step("analyze", analyze_func)
step2 = Step("respond", respond_func)
termination = Termination()

sm = StateMachine(
    initial_state=entry,
    transitions=[
        Transition(entry.step_id, [step1.step_id]),
        Transition(step1.step_id, [step2.step_id]),
        Transition(step2.step_id, [termination.step_id]),
    ],
    steps=[entry, step1, step2, termination],
    state_schema=AgentState
)

run = sm.run(initial_state)  # returns Run with history
```

### Tool Definition

Functions with type hints become tools:
```python
def search_web(query: str, num_results: Literal[5, 10, 20]) -> str:
    """Search the web for information."""
    # implementation
    return results

tool = Tool(search_web, name="web_search", description="Search the web...")
llm.register_tool(tool)
```

Type hints are **essential** for schema generation.

### Agent Loop

Canonical agentic loop:
```python
messages = [SystemMessage(content=system_prompt)]
while True:
    response = llm.invoke(messages)
    messages.append(response)
    
    if response.tool_calls:
        for tc in response.tool_calls:
            tool_result = llm.tools[tc.tool_name].func(**tc.arguments)
            messages.append(ToolMessage(
                content=tool_result,
                tool_call_id=tc.id,
                name=tc.tool_name
            ))
    else:
        break  # LLM is done
```

## Common Tasks

### Adding a New Tool

1. Write function with docstring and type hints
2. Wrap with `Tool(func, name="...", description="...")`
3. Register with `llm.register_tool(tool)`
4. Type hints determine schema (Literal → enum, List[T] → array, etc.)

### Extending an Agent

1. Add new steps to state machine with `Step(step_id, logic_func)`
2. Update transitions to connect the new step
3. Logic function receives state TypedDict, returns dict with updated fields
4. Use `Resource` to pass shared objects (LLM, vector stores) into steps

### Using Vector Search (Modules 8+)

```python
from lib.vector_db import VectorStore

vs = VectorStore()  # defaults to Chroma in-memory
vs.add(documents, metadatas=[...])
results = vs.query(query_texts=["question"])  # returns {documents, distances, metadatas}
```

### Debugging State Machines

- `Run` object contains `steps_executed` (list of Step IDs) and `history` (state snapshots)
- Inspect `run.history[-1]` to see final state
- Check conditionals if transitions not routing correctly

## Notes for Extension

- **No async/await**: All code is synchronous (suitable for learning)
- **No caching**: LLM calls are not cached
- **Stateless functions**: Steps should be pure; side effects go in Resource vars
- **TypedDict strictness**: State fields not in schema are silently dropped by `Step.run()`
- **Tool schema inference**: Only standard types supported; complex objects serialize to JSON strings

---

**Course URL**: Udacity CD14524 - Building Agents
**Last Updated**: 2024
