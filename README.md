Here's a well-formatted and comprehensive **`README.md`** file for your Tool Call Agent project, incorporating all the information you provided. This README is Markdown-compliant and structured with sections, code blocks, and emojis for clear documentation.

# 🧠 Tool Call Agent

This project demonstrates how to build a simple yet powerful **conversational agent** using [LangChain](https://github.com/langchain-ai/langchain) and [LangGraph](https://github.com/langchain-ai/langgraph). The agent can answer general questions and dynamically invoke **custom tools** (such as fetching real-time stock prices) as needed. It also supports **memory and multi-threaded conversations**, enabling contextual understanding across multiple sessions.

## 🚀 Features

- Conversational AI agent with LangChain & LangGraph  
- Integrates **custom tool** (e.g., stock price fetcher)
- Automatic **tool invocation logic**
- **Stateful agent memory** for conversation continuity
- **Multi-threaded session support** (via `thread_id`)

## 🛠️ How It Works

### 1. State Management

Conversation memory is managed using a `State` object.

```python
from typing_extensions import Annotated, TypedDict
from langgraph.graph import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]
```

- `TypedDict`: Defines the structured schema for storing chat messages.
- `messages`: List that holds user queries & AI responses.
- `Annotated[list, add_messages]`: Ensures conversation history is **appended**, not replaced every turn.

### 2. Tool Definition

Define tools that can be used by the agent:

```python
from langchain.tools import tool

@tool
def get_stock_price(symbol: str) -> float:
    '''Return the current price of a stock given the stock symbol'''
    # ... implementation ...
```

- The `@tool` decorator makes this function available to the language model.
- The **docstring** helps the model understand the tool’s purpose.

### 3. Graph Construction 🏗️

Define your logic as a **state graph**:

#### Nodes

- **`chatbot` Node**: Core "brain" of the agent - invokes the LLM.
- **`tools` Node**: Executes tools like `get_stock_price`.

#### Edges

- `START → chatbot`: Entry point
- `chatbot → tools` *(if tool is needed)*  
  `chatbot → END` *(if response is final)*
- `tools → chatbot`: Returns tool result to LLM for constructing reply

> The decision-making logic is automatically handled by a built-in condition handler: `tools_condition`.

## ✨ Code Examples

### 📌 Tool-Using Query

```python
graph.invoke({
    "messages": [{"role": "user", "content": "What is the price of AAPL stock right now?"}]
})
```

📈 **Flow**:  
`START → chatbot → tools → chatbot → END`  
💬 **Output**:  
`"The current price of AAPL stock is $100.4."`

### 🧠 General Knowledge Query

```python
graph.invoke({
    "messages": [{"role": "user", "content": "Who invented theory of relativity? print person name only"}]
})
```

📘 **Flow**:  
`START → chatbot → END`  
💬 **Output**:  
`"Albert Einstein"`

### ➗ Complex Multi-Step Query

```python
msg = "I want to buy 20 AMZN stocks using current price. Then 15 MSFT. What will be the total cost?"
graph.invoke({
    "messages": [{"role": "user", "content": msg}]
})
```

🔂 **Flow**:  
`chatbot → tools (AMZN) → chatbot → tools (MSFT) → chatbot → END`  
💬 **Example Output**:  
`"The total cost for 20 AMZN stocks and 15 MSFT stocks would be $6014.5."`

## 🧠 Tool Call Agent with Memory

### 1. State Management

Same as before, using:

```python
class State(TypedDict):
    messages: Annotated[list, add_messages]
```

### 2. Conversation Memory (Checkpointing) 🧠

```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
graph = builder.compile(checkpointer=memory)
```

- `MemorySaver`: Stores state in-memory (use Redis/Postgres in production).
- `checkpointer=memory`: Ensures conversation progress is saved.
- `thread_id`: Uniquely identifies each conversation session.

### 3. Graph Logic Overview

- Structure is the same as standard tool-using agents
- Difference: conversations retain context across turns, scoped by `thread_id`

## 🧪 Code Examples with Memory

### 🧵 Conversation 1 (`thread_id: '1'`)

#### 🔢 First Turn: Complex Cost Calculation

```python
config1 = { 'configurable': { 'thread_id': '1'} }

msg = "I want to buy 20 AMZN stocks using current price. Then 15 MSFT. What will be the total cost?"
graph.invoke({"messages": [{"role": "user", "content": msg}]}, config=config1)
```

💬 **Agent**: Calculates total cost and persists state under thread `'1'`.

#### ➕ Second Turn: Follow-up with Reference to Previous Total

```python
msg = "Using the current price tell me the total price of 10 RIL stocks and add it to previous total cost"
graph.invoke({"messages": [{"role": "user", "content": msg}]}, config=config1)
```

🧠 Agent pulls in previous memory, executes new tool call, and adds up totals correctly.

### 🧵 Conversation 2 (`thread_id: '2'`)

#### 📈 First Turn

```python
config2 = { 'configurable': { 'thread_id': '2'} }

msg = "Tell me the current price of 5 AAPL stocks."
graph.invoke({"messages": [{"role": "user", "content": msg}]}, config=config2)
```

🧠 Stored under thread `'2'`.

#### ➕ Second Turn

```python
msg = "Tell me the current price of 5 MSFT stocks and add it to previous total"
graph.invoke({"messages": [{"role": "user", "content": msg}]}, config=config2)
```

🧠 Previous total here relates **only** to thread `'2'` — fully isolated from thread `'1'`.

## 🧵 Parallel Sessions: How `thread_id` Works

| Conversation | `thread_id` | Description |
|--------------|-------------|-------------|
| 💰 Thread 1   | `'1'`       | Cumulative stock cost queries, memory-aware follow-ups |
| 📈 Thread 2   | `'2'`       | Independent stock price queries with own state |

You can imagine each `thread_id` as a **separate chat window** with its own memory context.

## 📚 Sources

- [LangChain Docs](https://docs.langchain.com/)
- [LangGraph Docs](https://docs.langchain.com/langgraph/)

## 📎 Conclusion

The **Tool Call Agent** showcases how to combine LLMs with custom tools using LangChain/LangGraph, along with **graph-based conditional logic** and **stateful memory** to handle complex queries and maintain multiple independent conversations.

Feel free to contribute, fork, or customize for your own use cases!

> 💡 Tip: For production use, consider replacing `MemorySaver` with persistent stores like Redis or Postgres to scale across users and sessions.

Let me know if you'd like the same as a downloadable file or want me to generate badges or setup instructions!
