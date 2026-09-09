# Hermes / MemPalace Investigation Status

## Purpose of This Document

This document is a checkpoint and handoff for an ongoing investigation into Hermes agent memory, context, prompt construction, retrieval, and persistence behavior.

It is intended to preserve the state of the investigation so that it can continue in a future conversation without depending on the original chat history or large pasted terminal outputs.

This document records:

* the known architecture
* the code paths discovered
* findings from source inspection
* relevant runtime behavior
* current areas of investigation
* unresolved questions
* recommended next steps

It does **not** claim that the investigation is complete.

---

# 1. Project Environment

## Memory Palace

Memory Palace / MemPalace is being developed as an external memory and knowledge system.

Relevant locations:

```text
/home/peter/ai/mempalace
```

KnowledgeGraph module:

```text
/home/peter/ai/mempalace/knowledge_graph.py
```

Palace database:

```text
/home/peter/ai/mempalace/data/palace.db
```

The intended architecture is that Memory Palace / KnowledgeGraph is authoritative for factual retrieval.

Known characteristics of the KnowledgeGraph include:

* temporal entity/relationship storage
* SQLite-based graph storage
* validity intervals
* provenance-related information
* entity relationships

Previous work included changes to:

* strict `valid_to > as_of` logic
* timezone handling

---

# 2. Hermes Environment

Relevant Hermes locations:

```text
~/.hermes/hermes-agent
```

Hermes memory database:

```text
~/.hermes/memory_store.db
```

Hermes plugin location:

```text
~/.hermes/hermes-agent/plugins/memory/holographic
```

Hermes configuration:

```text
~/.hermes/config.yaml
```

The MemPalace Hermes provider uses a ChromaBackend.

A previously observed problem was that prefetch sometimes returned empty results.

---

# 3. Investigation Goal

The current investigation is attempting to understand the actual relationship between:

1. Hermes conversation memory
2. Hermes context
3. Hermes system prompt construction
4. MemPalace retrieval
5. KnowledgeGraph retrieval
6. session persistence
7. context compression
8. background/review agents
9. prompt caching

The investigation is intentionally focused on understanding the **actual runtime architecture** rather than guessing from configuration names.

A major concern is determining why information may:

* be present in storage but unavailable to the model
* be retrieved inconsistently
* disappear after compression or session changes
* fail to enter the model context
* be confused between Hermes memory and MemPalace memory
* appear available in one agent/fork but not another

---

# 4. Important Architectural Distinction

One of the most important things to preserve from the investigation is that several different systems may be involved.

These should not automatically be treated as the same thing:

```text
Conversation History
        |
        v
Hermes Session Messages
        |
        +--------------------+
        |                    |
        v                    v
Context Compression      Session Persistence
        |                    |
        v                    v
Current Model Context    Session Database


Hermes Memory System
        |
        v
memory_store.db / memory provider


MemPalace
        |
        +----------------+
        |                |
        v                v
Chroma Backend      KnowledgeGraph
```

The investigation needs to determine exactly:

* which system retrieves what
* which system writes what
* which system injects information into prompts
* which system survives compression
* which system is available to background agents
* which system is authoritative

---

# 5. Hermes Prompt Architecture Findings

Source inspection found explicit session-level cached prompt state.

File:

```text
/home/peter/.hermes/hermes-agent/agent/agent_init.py
```

Relevant state includes:

```python
"_cached_system_prompt": None,
"_cached_system_prompt_static": None,
```

Additional comments indicate that the prompt is:

* built once
* rebuilt during compression
* partially structured for prompt caching

Relevant comments describe:

```text
Cached system prompt (built once, rebuilt on compression)
```

and a separately stored:

```text
cross-session-stable prefix
```

There is also a frozen workspace snapshot:

```python
"_frozen_workspace_snapshot": None,
```

The comments indicate that workspace/git state is pinned on the first prompt build and replayed on rebuilds so that changing repository state does not create prompt-cache divergence before volatile prompt sections.

### Important implication

The system prompt is not necessarily reconstructed from scratch every request.

It appears to be active runtime state.

That means investigation of memory/context behavior must account for:

* cached prompts
* prompt rebuilds
* compression-triggered rebuilds
* provider/model changes
* static versus dynamic prompt regions

---

# 6. Prompt Cache Parity Is Architecturally Important

Several code comments explicitly emphasize:

```text
prompt-cache parity
```

and:

```text
byte-identical
```

For example, background review tool restrictions are intentionally applied at dispatch time rather than changing the advertised tool schema.

The reason is explicitly to preserve prompt-cache parity.

### Important finding

Hermes developers are deliberately trying to keep portions of the prompt structurally identical across:

* parent agents
* review agents
* provider changes
* prompt rebuilds

Therefore, changing:

* tools
* workspace information
* system prompt content
* provider identity

may have architectural implications beyond simple functionality.

---

# 7. Cached Prompt Is Modified During Provider/Model Changes

File:

```text
agent/chat_completion_helpers.py
```

Contains:

```python
def rewrite_prompt_model_identity(agent, model: str, provider: str) -> None:
```

The function operates on:

```python
agent._cached_system_prompt
```

and rewrites the last occurrence of:

```text
Model:
Provider:
```

The comments state that this is not persisted and that the stored session row retains the primary provider/model labels so that restoring the primary can replay a byte-identical prompt.

### Important implication

`_cached_system_prompt` is not simply a static configuration artifact.

It can be modified during runtime.

This means the investigation should understand:

* when it is initially built
* what data enters it
* when it is rebuilt
* what data can modify it
* whether memory information is embedded in it
* whether MemPalace information is embedded in it
* whether rebuilds preserve memory-related content

---

# 8. Context Compression Architecture

Relevant file:

```text
/home/peter/.hermes/hermes-agent/agent/compression_facade.py
```

The compression system appears to handle:

* compression execution
* snapshot-based compression
* timeouts
* progress monitoring
* total timeout ceilings
* SessionDB interaction
* persisted marker synchronization
* session context rebinding

The module description states that it:

```text
Publishes the commit fence hard_interrupt() reads,
runs the compressor on a snapshot under the progress timeout,
mirrors _DB_PERSISTED_MARKER stamps back onto the live lists
and rebinds the session context.
```

### Important finding

Compression appears to operate on a snapshot rather than directly mutating live context during model processing.

Afterward it may:

* synchronize persistence markers
* update/rebind session context
* commit compressed state

---

# 9. Compression Timeout Behavior

Compression includes explicit fallback behavior.

The system can continue without compression if:

* the summary model stalls
* no summary progress occurs
* the total compression ceiling is reached

Relevant warnings indicate:

```text
No messages were dropped — continuing without compression.
```

The system also records compression timeout failures and uses a cooldown ladder to avoid repeatedly burning the full timeout budget.

### Important implication

Compression failure does not necessarily mean messages were deleted.

However, the investigation still needs to determine:

* what happens when compression succeeds
* exactly which messages are replaced
* whether summaries preserve memory-relevant information
* whether tool results survive
* whether external memory context survives
* whether compression causes retrieval differences

---

# 10. Compression Fallback Prompt Behavior

The compression facade includes:

```python
_timeout_fallback_prompt()
```

Behavior:

1. Use `_cached_system_prompt` if available.
2. Otherwise attempt to build a fresh system prompt.
3. If that fails, use the raw system message.

This confirms multiple possible prompt sources during compression-related processing.

### Investigation question

Does the fallback path include exactly the same memory/context information as the normal prompt construction path?

This has not yet been established.

---

# 11. Session Persistence

`agent_init.py` contains session persistence state including:

```python
"_session_messages": list,
"_session_persist_lock": threading.RLock,
"_pending_cli_user_message": None,
"_last_flushed_db_idx": 0,
"_session_db_created": False,
"_end_session_on_close": True,
"_persist_disabled": False,
```

Comments indicate:

* persistence races are considered
* message-level persistence markers are used
* duplicate DB writes are actively prevented
* helper agents may use continuation rows
* background review forks can disable persistence

### Important distinction

The existence of information in:

```text
SessionDB
```

does not automatically prove that the same information is currently in:

```text
Model Context
```

Likewise:

```text
Model Context
```

does not automatically prove the information is in:

```text
MemPalace
```

or:

```text
KnowledgeGraph
```

These layers must be investigated separately.

---

# 12. Persistence Markers

Compression code references:

```python
_DB_PERSISTED_MARKER
```

and contains logic to synchronize persisted markers between:

* worker snapshot messages
* live messages

The comments explicitly mention avoiding a duplicate-row bug.

### Important implication

There is nontrivial machinery preventing compressed/snapshotted messages from being persisted incorrectly.

This is another reason not to assume:

```text
current conversation list
```

and:

```text
persistent conversation history
```

are identical representations.

---

# 13. Background Review Agents

Relevant file:

```text
/home/peter/.hermes/hermes-agent/agent/background_review.py
```

The review system creates child/review agents.

Relevant concepts include:

```text
_active_children
```

and:

```python
def _review_tool_whitelist(review_agent, task_cfg)
```

The review fork tool whitelist is intentionally dispatch-side only.

The comments explicitly state that the advertised:

```text
tools[]
```

remains byte-identical to the parent.

---

# 14. Background Review Memory Behavior

The review tool configuration includes:

```python
memory_on = review_agent._memory_enabled or review_agent._user_profile_enabled
```

If memory is enabled, review toolsets include:

```text
memory
skills
```

Otherwise:

```text
skills
```

are enabled.

### Important finding

Memory availability is explicitly conditional for review agents.

Therefore, a background review agent may not necessarily have the same memory capabilities as the parent.

This needs to be investigated when diagnosing inconsistent memory behavior.

---

# 15. Background Review File Access

Review agents explicitly whitelist:

```text
read_file
search_files
```

The comments explain that denying these tools caused a large number of denied tool calls and prevented the self-improvement loop from functioning.

Read-only file access is allowed.

Write tools remain denied.

The comments explicitly describe:

```text
write_file
patch
terminal
```

as denied.

Autonomous maintenance is expected to go through:

```text
skill_manage
```

validation.

### Important finding

Review agents may be capable of inspecting files even when they cannot directly modify them.

This matters if future investigation involves determining what information a review agent can access compared with:

* foreground agent
* memory provider
* session context

---

# 16. Background Review Persistence

Session state comments indicate:

```text
True on the background review fork: never persist
```

through:

```python
"_persist_disabled": False
```

when configured accordingly.

### Important implication

Background review forks should not automatically be expected to behave like ordinary persistent conversations.

The investigation should treat them as separate runtime contexts.

---

# 17. Tool Schema Versus Tool Permission

An important architectural distinction discovered in the background review system is:

```text
Advertised Tool Schema
        !=
Actual Dispatch Permission
```

A tool can potentially remain in the inherited advertised schema while dispatch-side logic controls whether the tool is actually allowed.

This was specifically done to preserve:

```text
prompt-cache parity
```

### General lesson

When investigating what an Hermes agent can actually do, do not rely only on the tool list visible to the model.

Investigate:

1. advertised tool definitions
2. enabled toolsets
3. dispatch whitelist
4. profile configuration
5. agent-specific restrictions

---

# 18. Summary/API Message Construction

File:

```text
agent/chat_completion_helpers.py
```

Contains logic related to summary calls and API message construction.

Relevant behavior includes:

```python
effective_system = agent._cached_system_prompt or ""
```

Then:

```text
ephemeral_system_prompt
```

may be appended.

The resulting effective system prompt is inserted as:

```json
{
  "role": "system"
}
```

before other API messages.

### Important implication

The actual prompt sent to a model may be composed from:

```text
_cached_system_prompt
+
ephemeral_system_prompt
+
prefill_messages
+
conversation messages
```

This composition needs to be mapped precisely.

---

# 19. Message Sanitization

The summary/API path performs several cleanup operations.

These include:

* sanitizing tool calls
* removing schema-foreign keys
* removing internal underscore-prefixed keys
* sanitizing orphaned tool results
* evicting stale outbound tool images
* dropping thinking-only assistant messages where needed
* merging user messages where needed

### Important investigation concern

The model-facing message list is not necessarily identical to the internal Hermes message list.

There may be several transformations between:

```text
Stored Message
        ↓
Internal Message
        ↓
Compression/Summary Message
        ↓
Sanitized API Message
        ↓
Provider Request
```

Therefore, debugging missing context requires examining the actual send path.

---

# 20. Provider-Specific Message Behavior

The source includes provider-specific handling for:

* Anthropic
* OpenAI-compatible APIs
* Codex Responses
* Nous
* LM Studio
* OpenRouter-related behavior

The code explicitly handles provider incompatibilities including:

* encrypted reasoning state
* thinking signatures
* multimodal tool content
* strict schema validation
* long context tier availability
* reasoning parameters
* API mode differences

### Important implication

Context problems may be provider-specific.

A conversation that behaves correctly under one provider may not necessarily send exactly the same wire-format context under another provider.

---

# 21. Model/Provider Failover

The code includes detailed failover reason categories.

Examples include:

```text
authentication failed
billing or quota exhausted
rate limit
provider overloaded
provider server error
request timeout
context window exceeded
request payload too large
encrypted reasoning state rejected
thinking signature rejected
long-context tier unavailable
```

### Important investigation concern

Provider fallback could potentially change:

* model
* API mode
* message serialization
* reasoning support
* context limits

while the user may experience the system as one continuous Hermes conversation.

Future investigation should determine whether provider fallback occurred during any observed memory/context problem.

---

# 22. Known Hermes Memory Architecture

Hermes includes a memory system separate from ordinary conversation history.

Known relevant location:

```text
~/.hermes/memory_store.db
```

Known plugin:

```text
~/.hermes/hermes-agent/plugins/memory/holographic
```

The holographic provider implements hybrid retrieval including:

* SQLite FTS5
* Jaccard reranking
* trust weighting
* HRR fallback

Known capabilities/modes include:

* hybrid scoring
* entity probing
* related facts discovery
* multi-entity reasoning
* contradiction detection

### Important investigation question

Exactly how does the holographic memory provider interact with:

```text
MemPalace
```

and:

```text
KnowledgeGraph
```

at runtime?

This should not be assumed.

---

# 23. MemPalace Integration

The MemPalace Hermes provider uses:

```text
ChromaBackend
```

A previously observed issue was:

```text
Prefetch sometimes returned empty.
```

This remains important.

Future investigation should determine:

1. What triggers prefetch?
2. Which query is generated?
3. Which backend receives the query?
4. Whether Chroma contains the expected data.
5. Whether results are filtered.
6. Whether results are reranked away.
7. Whether results are retrieved but never injected.
8. Whether results are injected into the system prompt, conversation context, tool context, or another structure.

---

# 24. KnowledgeGraph Architecture

The KnowledgeGraph is intended to be authoritative for factual retrieval.

Relevant characteristics:

* temporal entity relationships
* SQLite storage
* entity relationship reasoning
* validity intervals
* timezone-aware temporal handling

Previous work changed the interpretation of:

```text
valid_to
```

to strict:

```text
valid_to > as_of
```

rather than inclusive behavior.

### Important architectural principle

KnowledgeGraph should not necessarily be treated as another conversation history store.

The intended role is structured factual knowledge and relationships.

---

# 25. Important Distinction: Knowledge Versus Conversation

The investigation and broader project architecture should preserve this distinction:

## Conversation Context

Useful for:

* current task
* immediate reasoning
* recent discussion
* temporary working state

## Hermes Memory

Potentially useful for:

* remembered facts
* learned information
* long-term conversational information

## MemPalace

Potentially useful for:

* external durable memory
* semantic retrieval
* structured retrieval
* source-backed knowledge

## KnowledgeGraph

Useful for:

* entities
* relationships
* temporal facts
* structured factual reasoning

These systems may overlap but should not automatically be merged conceptually.

---

# 26. Current Major Questions

The investigation has not yet answered the following.

## Question 1

What exactly builds `_cached_system_prompt`?

Need to locate:

```python
_build_system_prompt()
```

and inspect:

* inputs
* memory injection
* MemPalace injection
* KnowledgeGraph injection
* workspace injection
* profile injection

---

## Question 2

Where does Hermes memory enter model context?

Need to trace:

```text
Memory Retrieval
        ↓
Memory Result
        ↓
Context Injection
        ↓
Prompt/API Message
```

---

## Question 3

Where does MemPalace retrieval enter model context?

Need to determine whether MemPalace results become:

* system prompt content
* user context
* tool results
* memory-context tags
* hidden prefill
* ephemeral system prompt content
* something else

---

## Question 4

Does context compression preserve external memory context?

Need to determine what happens to:

* Hermes memory
* MemPalace context
* KnowledgeGraph context
* tool results
* retrieved documents

when compression occurs.

---

## Question 5

What causes empty MemPalace prefetch?

Known observation:

```text
Prefetch sometimes returned empty.
```

Need to identify whether the failure is:

```text
No query
No documents
Wrong backend
Filtering
Reranking
Injection failure
Timing
Configuration
```

---

## Question 6

Does cached system prompt contain stale memory?

Because:

```python
_cached_system_prompt
```

is persistent runtime state and can survive until rebuilds, investigate whether memory/context information can become stale.

---

## Question 7

When is `_cached_system_prompt` rebuilt?

Known indication:

```text
rebuilt on compression
```

Need to identify all other triggers.

---

## Question 8

What is the exact difference between:

```text
_cached_system_prompt
```

and:

```text
_cached_system_prompt_static
```

Need to inspect:

* construction
* lifetime
* purpose
* cache markers
* dynamic additions

---

## Question 9

What information is lost between internal messages and API messages?

Need to trace:

```text
agent messages
```

through:

```text
_sanitize_api_messages()
```

and related functions.

---

## Question 10

Do background/review agents have access to the same memory context?

Known:

* memory is conditional
* persistence can be disabled
* tools are restricted
* prompt schema is preserved

Unknown:

* exact memory retrieval behavior
* MemPalace availability
* KnowledgeGraph availability

---

# 27. Recommended Investigation Order

The next investigation should proceed in this order.

---

## Step 1 — Locate System Prompt Builder

Find and inspect:

```text
_build_system_prompt
```

Determine:

* every input
* every data source
* memory integration
* profile integration
* workspace integration
* cache boundaries

This is currently one of the highest-priority steps.

---

## Step 2 — Map Prompt Construction

Build an actual diagram like:

```text
Config
  │
  ▼
Agent Initialization
  │
  ▼
_build_system_prompt()
  │
  ▼
_cached_system_prompt
  │
  ├── provider/model rewrite
  │
  ├── compression rebuild
  │
  └── ephemeral additions
         │
         ▼
Effective System Prompt
         │
         ▼
API Messages
```

Then identify where:

```text
Hermes Memory
MemPalace
KnowledgeGraph
```

enter.

---

## Step 3 — Trace Memory Retrieval

Find the code path for:

```text
memory query
```

through:

```text
retrieval
```

through:

```text
reranking
```

through:

```text
context injection
```

Do not stop at confirming retrieval.

The important question is:

```text
Did the model actually receive the result?
```

---

## Step 4 — Trace MemPalace Prefetch

Identify:

* prefetch caller
* query generation
* backend call
* result count
* filtering
* reranking
* final context injection

Add temporary understanding/documentation only if modification is later explicitly allowed.

The investigation should initially remain read-only unless a change is specifically approved.

---

## Step 5 — Inspect Compression Boundary

Determine:

```text
Before Compression Context
```

versus:

```text
After Compression Context
```

Specifically track:

* user messages
* assistant messages
* tool calls
* tool results
* memory context
* external retrieval context

---

## Step 6 — Inspect Actual Outbound Request

The most useful eventual diagnostic may be determining the actual message payload immediately before the provider request.

This is important because several transformations occur after internal message storage.

The investigation should compare:

```text
Internal Agent State
```

with:

```text
Actual Provider Request
```

---

# 28. Important Files Identified

## Agent initialization

```text
/home/peter/.hermes/hermes-agent/agent/agent_init.py
```

Relevant concepts:

```text
_cached_system_prompt
_cached_system_prompt_static
_session_messages
_frozen_workspace_snapshot
session persistence
prompt caching
```

---

## Background review

```text
/home/peter/.hermes/hermes-agent/agent/background_review.py
```

Relevant concepts:

```text
review agents
memory enablement
tool whitelisting
read-only file access
prompt-cache parity
```

---

## Compression facade

```text
/home/peter/.hermes/hermes-agent/agent/compression_facade.py
```

Relevant concepts:

```text
context compression
snapshot processing
timeouts
SessionDB
persisted markers
context rebinding
```

---

## Chat completion helpers

```text
/home/peter/.hermes/hermes-agent/agent/chat_completion_helpers.py
```

Relevant concepts:

```text
rewrite_prompt_model_identity
effective_system
message sanitization
provider failover
summary API messages
```

---

## MemPalace

```text
/home/peter/ai/mempalace
```

---

## KnowledgeGraph

```text
/home/peter/ai/mempalace/knowledge_graph.py
```

---

## Palace database

```text
/home/peter/ai/mempalace/data/palace.db
```

---

## Hermes memory database

```text
~/.hermes/memory_store.db
```

---

## Holographic memory plugin

```text
~/.hermes/hermes-agent/plugins/memory/holographic
```

---

# 29. Investigation Rules / Preferences

Previous investigation work followed a strict preference for:

## Read-Only Investigation First

Do not:

* modify source code
* create experimental files
* modify databases
* patch Hermes
* change configuration

until the existing architecture and problem are understood.

Prefer:

1. inspect
2. search
3. trace
4. compare
5. identify evidence
6. form hypothesis
7. verify
8. only then consider modification

---

# 30. User Workflow Preferences

When continuing terminal-based investigation:

* explain what is being investigated before commands
* put commands at the end of the response
* provide commands in manageable batches
* preferably no more than approximately 5 commands at once
* ask for output before moving to the next batch
* avoid changing files during read-only investigation

---

# 31. Current Best Working Model

The current working hypothesis is **not yet a conclusion**.

Hermes appears to have several independent layers capable of affecting whether information reaches the model:

```text
Persistent Memory
        │
        ▼
Memory Retrieval
        │
        ▼
Context Injection
        │
        ▼
Cached System Prompt / Ephemeral Prompt
        │
        ▼
Conversation Messages
        │
        ▼
Compression / Sanitization
        │
        ▼
Provider-Specific Serialization
        │
        ▼
Model
```

MemPalace and KnowledgeGraph may enter somewhere in this pipeline, but the exact insertion points still need to be established.

Therefore, a memory problem could occur even if:

```text
Database contains information
```

because failure could occur later in:

```text
retrieval
reranking
filtering
context injection
prompt caching
compression
message sanitization
provider serialization
```

---

# 32. Most Important Things Not to Lose

The following findings should be preserved even if the investigation changes direction.

### 1.

`_cached_system_prompt` exists and is important runtime state.

### 2.

The system prompt is rebuilt during compression.

### 3.

Prompt-cache parity is an explicit architectural concern.

### 4.

Background review agents are not identical to foreground agents.

### 5.

Memory availability is conditional for review agents.

### 6.

Session persistence and model context are separate concerns.

### 7.

Internal Hermes messages and provider API messages are transformed representations.

### 8.

Compression uses snapshots and can rebind session context.

### 9.

Provider/model switching can modify cached prompt identity.

### 10.

MemPalace prefetch has previously returned empty and requires an end-to-end trace.

### 11.

KnowledgeGraph is intended as structured authoritative factual retrieval, not merely conversation history.

### 12.

The investigation must trace information all the way to the actual model request rather than stopping when information is found in a database or retrieval function.

---

# 33. Recommended Starting Point for the Next Investigation Session

Start by locating and inspecting:

```text
_build_system_prompt
```

Then answer these exact questions:

1. What builds `_cached_system_prompt`?
2. What builds `_cached_system_prompt_static`?
3. Where does Hermes memory enter?
4. Where does MemPalace enter?
5. Where does KnowledgeGraph enter?
6. What survives prompt rebuilding?
7. What survives compression?
8. What is actually sent to the provider?

Do not begin by modifying MemPalace or Hermes.

First produce a complete path diagram of:

```text
Stored Information
        ↓
Retrieval
        ↓
Context Construction
        ↓
Prompt
        ↓
Compression/Sanitization
        ↓
Provider Request
        ↓
Model
```

---

# STATUS

## Investigation Status

```text
IN PROGRESS
```

## Architecture Understanding

```text
PARTIAL
```

## Root Cause Identified

```text
NO
```

## Read-Only Investigation

```text
YES
```

## Highest Priority

```text
Trace _build_system_prompt and the complete path by which
Hermes Memory / MemPalace / KnowledgeGraph information reaches
the actual provider request.
```

---

# CHANGE LOG

## Initial checkpoint

Created to preserve an investigation that had become difficult to continue because large terminal output and pasted source results were exceeding conversation/file limits.

This document should be updated rather than replaced as the investigation continues.
