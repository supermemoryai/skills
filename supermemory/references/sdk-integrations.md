# SDK Integrations

Supermemory provides integrations for all major AI frameworks. The core pattern is the same: fetch user context before the LLM call, store the conversation after.

## Packages

| Package | Framework | Language | Install |
|---------|-----------|----------|---------|
| `supermemory` | Core SDK | TypeScript | `npm install supermemory` |
| `supermemory` | Core SDK | Python | `pip install supermemory` |
| `@supermemory/tools` | OpenAI, Vercel AI SDK, Mastra | TypeScript | `npm install @supermemory/tools` |
| `supermemory-openai-sdk` | OpenAI function calling | Python | `pip install supermemory-openai-sdk` |
| `supermemory-pipecat` | Pipecat (Voice AI) | Python | `pip install supermemory-pipecat` |

## Core SDK

### TypeScript

```typescript
import Supermemory from 'supermemory';

const client = new Supermemory({ apiKey: process.env.SUPERMEMORY_API_KEY });

// Add a memory
await client.add({ content: "Meeting notes from Q1 planning", containerTags: ["user_123"] });

// Search memories
const response = await client.search.documents({
  q: "planning notes",
  containerTags: ["user_123"]
});

// Get user profile
const profile = await client.profile({ containerTag: "user_123" });
console.log(profile.profile.static);
console.log(profile.profile.dynamic);

// Add with metadata
await client.add({
  content: "Technical design doc",
  containerTags: ["user_123"],
  metadata: { category: "engineering", priority: "high" }
});

// Search with filters
const results = await client.search.documents({
  q: "design document",
  containerTags: ["user_123"],
  filters: {
    AND: [{ key: "category", value: "engineering" }]
  }
});

// List documents
const docs = await client.documents.list({ containerTags: ["user_123"], limit: 10 });

// Delete a document
await client.documents.delete({ docId: "doc_123" });
```

### Python

```python
from supermemory import Supermemory

client = Supermemory(api_key=os.environ.get("SUPERMEMORY_API_KEY"))

# Add a memory
client.add(content="Meeting notes from Q1 planning", container_tags=["user_123"])

# Search memories
response = client.search.documents(q="planning notes", container_tags=["user_123"])

# Get user profile
profile = client.profile(container_tag="user_123")
print(profile.profile.static)
print(profile.profile.dynamic)

# Add with metadata
client.add(
    content="Technical design doc",
    container_tags=["user_123"],
    metadata={"category": "engineering", "priority": "high"}
)

# Search with filters
results = client.search.documents(
    q="design document",
    container_tags=["user_123"],
    filters={"AND": [{"key": "category", "value": "engineering"}]}
)

# List documents
docs = client.documents.list(container_tags=["user_123"], limit=10)

# Delete a document
client.documents.delete(doc_id="doc_123")
```

## The Standard Pattern

Every integration follows this flow:

```
1. Fetch user profile + relevant memories  →  client.profile(containerTag, q)
2. Inject context into system prompt       →  profile.static + profile.dynamic + searchResults
3. Run LLM                                 →  (your LLM call)
4. Store conversation                      →  client.add(content, containerTag)
```

## OpenAI SDK Integration

### withSupermemory Wrapper (Zero-Config)

```typescript
import OpenAI from "openai"
import { withSupermemory } from "@supermemory/tools/openai"

const openai = new OpenAI()
const client = withSupermemory(openai, "user-123", {
  mode: "full",        // "profile" | "query" | "full"
  addMemory: "always", // "always" | "never"
})

// Use normally — memories auto-injected into system prompts
const response = await client.chat.completions.create({
  model: "gpt-5",
  messages: [
    { role: "system", content: "You are a helpful assistant." },
    { role: "user", content: "What's my favorite programming language?" }
  ]
})

// Also works with Responses API
const response2 = await client.responses.create({
  model: "gpt-5",
  instructions: "You are a helpful assistant.",
  input: "What do you know about me?"
})
```

### Function Calling Tools

```typescript
import { getToolDefinitions, createToolCallExecutor } from "@supermemory/tools/openai"

const toolDefinitions = getToolDefinitions()
const executeToolCall = createToolCallExecutor(process.env.SUPERMEMORY_API_KEY!, {
  projectId: "your-project-id",
})

const completion = await client.chat.completions.create({
  model: "gpt-5",
  messages: [{ role: "user", content: "What do you remember about my preferences?" }],
  tools: toolDefinitions,
})

if (completion.choices[0]?.message.tool_calls) {
  for (const toolCall of completion.choices[0].message.tool_calls) {
    const result = await executeToolCall(toolCall)
  }
}
```

### Python Function Calling

```python
from supermemory_openai import SupermemoryTools, execute_memory_tool_calls

tools = SupermemoryTools(
    api_key="your-supermemory-api-key",
    config={"project_id": "my-project"}
)

response = await client.chat.completions.create(
    model="gpt-5",
    messages=[...],
    tools=tools.get_tool_definitions()
)

if response.choices[0].message.tool_calls:
    tool_results = await execute_memory_tool_calls(
        api_key="your-supermemory-api-key",
        tool_calls=response.choices[0].message.tool_calls,
        config={"project_id": "my-project"}
    )
```

## Vercel AI SDK Integration

### User Profiles with Middleware

```typescript
import { generateText } from "ai"
import { withSupermemory } from "@supermemory/tools/ai-sdk"
import { openai } from "@ai-sdk/openai"

const modelWithMemory = withSupermemory(openai("gpt-5"), "user-123", {
  mode: "full",      // "profile" | "query" | "full"
  addMemory: "always" // enable to auto-save conversations
})

const result = await generateText({
  model: modelWithMemory,
  messages: [{ role: "user", content: "What do you know about me?" }]
})
```

### Custom Prompt Templates

```typescript
import { withSupermemory, type MemoryPromptData } from "@supermemory/tools/ai-sdk"

const claudePrompt = (data: MemoryPromptData) => `
<context>
  <user_profile>${data.userMemories}</user_profile>
  <relevant_memories>${data.generalSearchMemories}</relevant_memories>
</context>
`.trim()

const model = withSupermemory(anthropic("claude-3-sonnet"), "user-123", {
  mode: "full",
  promptTemplate: claudePrompt
})
```

### Memory Tools for Agents

```typescript
import { streamText } from "ai"
import { supermemoryTools } from "@supermemory/tools/ai-sdk"

const result = await streamText({
  model: anthropic("claude-3-sonnet"),
  prompt: "Remember that my name is Alice",
  tools: supermemoryTools("YOUR_SUPERMEMORY_KEY")
})

// Individual tools for more control
import { searchMemoriesTool, addMemoryTool } from "@supermemory/tools/ai-sdk"

const result = await streamText({
  model: openai("gpt-5"),
  prompt: "What do you know about me?",
  tools: {
    searchMemories: searchMemoriesTool("API_KEY", { projectId: "personal" }),
    createEvent: yourCustomTool,
  }
})
```

## Mastra Integration

### withSupermemory Wrapper

```typescript
import { Agent } from "@mastra/core/agent"
import { withSupermemory } from "@supermemory/tools/mastra"
import { openai } from "@ai-sdk/openai"

const agent = new Agent(withSupermemory(
  {
    id: "my-assistant",
    name: "My Assistant",
    model: openai("gpt-4o"),
    instructions: "You are a helpful assistant.",
  },
  "user-123",
  {
    mode: "full",
    addMemory: "always",
    threadId: "conv-456",
  }
))

const response = await agent.generate("What do you know about me?")
```

### Direct Processor Usage

```typescript
import { createSupermemoryProcessors } from "@supermemory/tools/mastra"

const { input, output } = createSupermemoryProcessors("user-123", {
  mode: "full",
  addMemory: "always",
  threadId: "conv-456",
})

const agent = new Agent({
  id: "my-assistant",
  model: openai("gpt-4o"),
  inputProcessors: [input],
  outputProcessors: [output],
})
```

## LangChain Integration (Python)

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage
from langchain_core.prompts import ChatPromptTemplate
from supermemory import Supermemory

llm = ChatOpenAI(model="gpt-4o")
memory = Supermemory()

def chat(user_id: str, message: str) -> str:
    profile_result = memory.profile(container_tag=user_id, q=message)

    static_facts = profile_result.profile.static or []
    dynamic_context = profile_result.profile.dynamic or []
    search_results = profile_result.search_results.results if profile_result.search_results else []

    context = f"""
User Background:
{chr(10).join(static_facts) if static_facts else 'No profile yet.'}

Recent Context:
{chr(10).join(dynamic_context) if dynamic_context else 'No recent activity.'}

Relevant Memories:
{chr(10).join([r.memory or r.chunk for r in search_results]) if search_results else 'None found.'}
"""

    prompt = ChatPromptTemplate.from_messages([
        SystemMessage(content=f"You are a helpful assistant.\n{context}"),
        HumanMessage(content=message)
    ])

    chain = prompt | llm
    response = chain.invoke({})

    memory.add(content=f"User: {message}\nAssistant: {response.content}", container_tag=user_id)
    return response.content
```

## LangGraph Integration (Python)

```python
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage
from supermemory import Supermemory

llm = ChatOpenAI(model="gpt-4o")
memory = Supermemory()

class State(TypedDict):
    messages: Annotated[list, add_messages]
    user_id: str

def agent(state: State):
    user_id = state["user_id"]
    messages = state["messages"]
    user_query = messages[-1].content

    profile_result = memory.profile(container_tag=user_id, q=user_query)

    static_facts = profile_result.profile.static or []
    dynamic_context = profile_result.profile.dynamic or []
    search_results = profile_result.search_results.results if profile_result.search_results else []

    context = f"""
User Background:
{chr(10).join(static_facts) if static_facts else 'No profile yet.'}

Recent Context:
{chr(10).join(dynamic_context) if dynamic_context else 'No recent activity.'}

Relevant Memories:
{chr(10).join([r.memory or r.chunk for r in search_results]) if search_results else 'None found.'}
"""

    system = SystemMessage(content=f"You are a helpful assistant.\n\n{context}")
    response = llm.invoke([system] + messages)

    memory.add(content=f"User: {user_query}\nAssistant: {response.content}", container_tag=user_id)
    return {"messages": [response]}

graph = StateGraph(State)
graph.add_node("agent", agent)
graph.add_edge(START, "agent")
graph.add_edge("agent", END)
app = graph.compile()
```

## OpenAI Agents SDK (Python)

```python
from agents import Agent, Runner, function_tool
from supermemory import Supermemory

memory = Supermemory()

@function_tool
def search_memories(query: str, user_id: str) -> str:
    """Search the user's memories for relevant information."""
    results = memory.search.memories(q=query, container_tag=user_id, limit=5)
    if not results.results:
        return "No relevant memories found."
    return "\n".join([r.memory or r.chunk for r in results.results])

@function_tool
def save_memory(content: str, user_id: str) -> str:
    """Store something important about the user for later."""
    memory.add(content=content, container_tag=user_id)
    return f"Saved: {content}"

agent = Agent(
    name="assistant",
    instructions="You are a helpful assistant with memory. Save preferences, search memories first.",
    tools=[search_memories, save_memory],
    model="gpt-4o"
)
```

## CrewAI Integration (Python)

```python
from crewai import Agent, Task, Crew, Process
from supermemory import Supermemory

memory = Supermemory()

def build_context(user_id: str, query: str) -> str:
    result = memory.profile(container_tag=user_id, q=query)
    static = result.profile.static or []
    dynamic = result.profile.dynamic or []
    memories = result.search_results.results if result.search_results else []

    return f"""
User Profile:
{chr(10).join(static) if static else 'No profile data.'}

Current Context:
{chr(10).join(dynamic) if dynamic else 'No recent activity.'}

Relevant History:
{chr(10).join([m.memory or m.chunk for m in memories[:5]]) if memories else 'None.'}
"""

def create_agent_with_memory(user_id: str, role: str, goal: str, query: str) -> Agent:
    context = build_context(user_id, query)
    return Agent(
        role=role,
        goal=goal,
        backstory=f"You have this info about the user:\n{context}\nUse it to personalize your work.",
        verbose=True
    )
```

## Pipecat Integration (Voice AI, Python)

```python
from supermemory_pipecat import SupermemoryPipecatService
from supermemory_pipecat.service import InputParams

memory = SupermemoryPipecatService(
    api_key=os.getenv("SUPERMEMORY_API_KEY"),
    user_id="unique_user_id",
    session_id="session_123",
    params=InputParams(
        mode="full",            # "profile" | "query" | "full"
        search_limit=10,
        search_threshold=0.1,
    ),
)

# Position between context aggregator and LLM in pipeline
pipeline = Pipeline([
    transport.input(),
    stt,
    context_aggregator.user(),
    memory,                    # <- Supermemory
    llm,
    tts,
    transport.output(),
    context_aggregator.assistant(),
])
```

## Memory Search Modes

All integrations support three modes:

| Mode | Description | Use Case |
|------|-------------|----------|
| `profile` | User profile only (static + dynamic facts) | General personalization |
| `query` | Semantic search based on user message | Specific question answering |
| `full` | Both profile and search | Chatbots, assistants (recommended) |

## Authentication

All requests require a Bearer token. Get your API key from [console.supermemory.ai](https://console.supermemory.ai).

```bash
export SUPERMEMORY_API_KEY=your_api_key
```

### Scoped API Keys

Restrict keys to a single containerTag for multi-tenant apps:

```bash
curl https://api.supermemory.ai/v3/auth/scoped-key \
  --request POST \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  -d '{"containerTag": "my-project", "expiresInDays": 30}'
```

Scoped keys work on: `/v3/documents`, `/v3/memories`, `/v4/memories`, `/v3/search`, `/v4/search`, `/v4/profile`
