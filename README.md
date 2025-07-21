# 🧠 Tool Call Agent with Memory

This project shows how to build a **conversational AI agent** using [LangChain](https://github.com/langchain-ai/langchain) and [LangGraph](https://github.com/langchain-ai/langgraph) that can both answer user questions and use **external tools** (like fetching real-time stock prices). It also maintains **independent memory** for multiple conversations, so each session keeps its own context and history.

## Table of Contents

- [Overview](#overview)
- [How the Agent Works](#how-the-agent-works)
  - [State Management](#state-management)
  - [Tool Integration](#tool-integration)
  - [Graph Construction](#graph-construction)
- [Using Conversation Memory](#using-conversation-memory)
- [Code Examples](#code-examples)
- [Understanding Parallel Conversations](#understanding-parallel-conversations)
- [Summary](#summary)

## Overview

**Tool Call Agent** is a Python project that demonstrates:
- How to combine language models with external tools (like a stock price lookup function) using LangChain.
- How to use LangGraph to set up the reasoning flow for when to use a tool and when to answer directly.
- How to save and retrieve conversation history using memory, so each chat session can reference past answers and context.

## How the Agent Works

### State Management

At the heart of this chatbot is a **state object** that stores the whole conversation history:

```python
class State(TypedDict):
    messages: Annotated[list, add_messages]
```
- **TypedDict:** Enforces that our `State` dictionary always contains a `messages` key.
- **messages:** This key stores a list of all question/answer pairs between the user and the agent.
- **Annotated[list, add_messages]:** Instead of replacing the list each turn, this special helper ensures new messages are added to the conversation, preserving earlier context.

**Why?**   
This approach lets the agent always “see” the history of the chat and refer to previous questions or answers to avoid repeating itself or losing context.

### Tool Integration

You can define **custom tools** for the agent to use when specialized information is needed. Here’s how to add a tool for stock prices:

```python
@tool
def get_stock_price(symbol: str) -> float:
    '''Return the current price of a stock given the stock symbol'''
    # ... implementation ...
```

- **@tool Decorator:** Registers this Python function as a "tool" available to the LLM (Language Learning Model).
- **Docstring:** Explains the tool’s function so the model knows what kind of queries it’s designed for.

**What happens?**  
When you ask for a stock price, the chatbot will decide to use this tool rather than just guessing.

### Graph Construction

Conversation logic is built as a **graph of nodes and edges**:

#### Nodes

- **chatbot:** The central reasoning node. It looks at the history and decides whether it can answer or needs to call a tool.
- **tools:** This node actually runs the Python tool and returns its output to the chatbot.

#### Edges

- **START → chatbot:** All chats start here.
- **chatbot → tools:** If the agent needs to use a tool, it routes here.
- **chatbot → END:** If no tool is needed and an answer can be given directly.
- **tools → chatbot:** After tool use, returns to the chatbot so the answer can be formulated using the tool’s result.

**How does this help?**  
You can easily extend the agent with more steps, decision points, or new tools by modifying the graph structure.

## Using Conversation Memory

To track context over multiple messages (and even different simultaneous chats), **memory** (checkpointing) is integrated:

```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
graph = builder.compile(checkpointer=memory)
```

- **MemorySaver:** Stores conversation states in memory (can be replaced by databases for production).
- **checkpointer=memory:** Ensures all state updates are automatically saved and retrievable.
- **thread_id:** Each session (chat window) is identified by a unique thread_id, so questions and answers aren’t mixed between users or chat sessions.

**Why use memory?**  
If your user refers to answers from earlier in the conversation ("add this to my previous total"), the agent will remember—even across several turns—because all chat history is saved and tied to that conversation's thread.

## Code Examples

### Example 1: Using a Tool

**User asks:** “What is the price of AAPL stock right now?”

```python
graph.invoke({"messages": [{"role": "user", "content": "What is the price of AAPL stock right now?"}]})
```

- **Agent Flow:** Recognizes a stock price is needed → Calls `get_stock_price("AAPL")` → Returns the price.
- **Sample Output:** `"The current price of AAPL stock is $100.4."`

### Example 2: Simple Factual Question

**User asks:** “Who invented theory of relativity? print person name only”

```python
graph.invoke({"messages": [{"role": "user", "content": "Who invented theory of relativity? print person name only"}]})
```

- **Agent Flow:** Already knows the answer, no tool needed.
- **Sample Output:** `"Albert Einstein"`

### Example 3: Multi-Step Calculation

**User asks:**  
“I want to buy 20 AMZN stocks using current price. Then 15 MSFT. What will be the total cost?”

```python
msg = "I want to buy 20 AMZN stocks using current price. Then 15 MSFT. What will be the total cost?"
graph.invoke({"messages": [{"role": "user", "content": msg}]})
```

- **Agent Flow:** Calculates price for AMZN → Calculates price for MSFT → Sums up and replies.
- **Sample Output:**  
    `"The total cost for 20 AMZN stocks and 15 MSFT stocks would be $6014.5."`

## Understanding Parallel Conversations

**Memory** enables the chatbot to manage multiple chats at the same time, keeping their history separate.

| Conversation  | thread_id | Example scenario                                                                                    |
|---------------|-----------|-----------------------------------------------------------------------------------------------------|
| 💰 1          |   '1'     | User calculates cost for AMZN & MSFT stocks, then carries forward the total in a follow-up request. |
| 📈 2          |   '2'     | Another user asks about AAPL price, and in a second turn, adds MSFT to their separate total.        |

### Example:

```python
config1 = { 'configurable': { 'thread_id': '1'} }
config2 = { 'configurable': { 'thread_id': '2'} }
```

- Running a query with `config1` saves to thread 1; running with `config2` saves to thread 2.
- Each session only remembers its *own* history. Users can have uninterrupted, context-rich multi-step chats without interference.

## Summary

**Tool Call Agent** lets you:
- Build a chat agent that intelligently decides when to answer and when to use external tools.
- Scale to multiple, memory-persistent conversations simultaneously.
- Easily integrate more tools or logic by adjusting the graph structure.

No matter how many users—or how complex their queries—each conversation is context-aware, accurate, and independent.

*Tip: For real applications at scale, use a persistent memory backend instead of in-memory memory (e.g., Redis or Postgres). And manage your code with [Git](https://github.com/) for best practices in AI project version control[1].*



