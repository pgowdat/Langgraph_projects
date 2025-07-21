TOOL CALL AGENT 

This project demonstrates how to build a simple, powerful conversational agent using LangChain and LangGraph. The agent can answer general questions and use custom tools to fetch specific, real-time data, like stock prices.

🚀 How It Works
The core of this application is a state machine or a graph that manages the flow of conversation. The agent decides whether to answer a question directly or to use a tool based on the user's query.
Here’s the step-by-step workflow:
1. State Management
The conversation's memory is managed by a State object. In this code, it's a simple dictionary that holds one key: messages.
Python

class State(TypedDict):
    messages: Annotated[list, add_messages]
	• TypedDict: This ensures our state object has a well-defined structure.
	• messages: A list that stores the entire conversation history (user queries and AI responses).
	• Annotated[list, add_messages]: This is a special LangGraph helper. Instead of overwriting the messages with each new turn, it appends them to the list, maintaining the context of the conversation.
	
2. Tool Definition
We equip our agent with a custom tool to fetch stock prices.
Python

@tool
def get_stock_price(symbol: str) -> float:
    '''Return the current price of a stock given the stock symbol'''
    # ... implementation ...
	• The @tool decorator from LangChain automatically makes this Python function available to the language model.
	• The function's docstring is crucial! The model uses the description ('Return the current price of a stock...') to understand what the tool does and when to use it.
3. Graph Construction 🏗️
We use StateGraph to define the agent's logic as a series of nodes and edges.
Nodes (The Steps)
	1. chatbot Node: This node represents the "brain" of our agent. It takes the current conversation state and invokes the Language Model (LLM). The LLM is "bound" to our tools, meaning it knows they exist and can decide to call them.
	2. tools Node: This is a special ToolNode. If the chatbot node decides to use a tool, this node is responsible for actually executing the tool function (e.g., get_stock_price('AAPL')) and returning the result.
Edges (The Connections)
	1. START → chatbot: Every conversation starts at the chatbot node.
	2. chatbot → tools OR END (Conditional Edge): This is the most critical part. After the chatbot node runs:
		○ If the LLM's response includes a request to call a tool (like get_stock_price), the graph follows the edge to the tools node.
		○ If the LLM's response is a final answer to the user, the graph follows the edge to the END state, and the turn is over.
		○ The built-in tools_condition function handles this routing logic automatically.
	3. tools → chatbot: After the tools node executes the function, the result (e.g., the stock price) is passed back to the chatbot node. The LLM now sees the tool's output and uses this new information to generate a final, human-readable answer.



✨ Code Examples Explained
The script includes three different invocations to test the agent's abilities.
	1. Tool-Using Query
Python

graph.invoke({"messages": [{"role": "user", "content": "What is the price of AAPL stock right now?"}]})
		○ Flow: START → chatbot (sees "AAPL price", decides to use tool) → tools (executes get_stock_price('AAPL')) → chatbot (sees the price is 100.4, formulates the final answer) → END.
		○ Output: "The current price of AAPL stock is $100.4."
	2. General Knowledge Query
Python

graph.invoke({"messages": [{"role": "user", "content": "Who invented theory of relativity? print person name only"}]})
		○ Flow: START → chatbot (knows the answer, doesn't need a tool) → END.
		○ Output: "Albert Einstein"
	3. Complex, Multi-Step Query
Python

msg = "I want to buy 20 AMZN stocks using current price. Then 15 MSFT. What will be the total cost?"
graph.invoke({"messages": [{"role": "user", "content": msg}]})
		○ Flow: This is a more complex task. The LLM understands it needs to perform multiple steps. It will first call the tool for AMZN, then for MSFT. It will then use the results from both tool calls to perform the final calculation.
		○ Path: chatbot → tools (for AMZN) → chatbot → tools (for MSFT) → chatbot (calculates total) → END.
		○ Output: A message summarizing the total cost, like "The total cost for 20 AMZN stocks and 15 MSFT stocks would be $6014.5." (20 * 150.0 + 15 * 200.3).



---------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Tool Call agent with memory 


1. State Management
The conversation's memory is managed by a State object. In this code, it's a simple dictionary that holds one key: messages.
Python

class State(TypedDict):
    messages: Annotated[list, add_messages]
	• TypedDict: This ensures our state object has a well-defined structure.
	• messages: A list that stores the entire conversation history (user queries and AI responses).
	• Annotated[list, add_messages]: This is a special LangGraph helper. Instead of overwriting the messages with each new turn, it appends them to the list, maintaining the context of the conversation.


2. Conversation Memory (Checkpointing) 🧠
This is the new, powerful feature. We introduce a checkpointer to save the state of our graph after every step.
Python

from langgraph.checkpoint.memory import MemorySaver
 An in-memory checkpointer to save conversation states
memory = MemorySaver()
 Compile the graph with the checkpointer
graph = builder.compile(checkpointer=memory)
	• MemorySaver: This is a simple checkpointer that stores all conversation histories in your computer's memory. For production, you could swap this with a persistent database like Redis or Postgres.
	• checkpointer=memory: When compiling the graph, we connect it to our memory. Now, the graph will automatically save its state.
	• thread_id: To manage multiple conversations, we use a thread_id. Think of it as a unique ID for each chat session. The agent uses this ID to retrieve the correct history for a follow-up question, ensuring conversations don't get mixed up.

3. Graph Logic
The graph's structure is identical to a standard tool-using agent. The magic happens in how the graph is invoked, using the thread_id to maintain state.
	• chatbot Node: Takes the conversation history and decides whether to respond directly or use a tool.
	• tools Node: Executes the tool if the chatbot decides to use one.
	• Conditional Edges: Route the flow between the chatbot and tools nodes based on the LLM's decision. After a tool is used, the flow always returns to the chatbot so it can use the tool's output to form an answer.
	

✨ Code Examples Explained

The script simulates two parallel conversations, identified by thread_id: '1' and thread_id: '2'.
Conversation 1: thread_id: '1' (Complex Calculation)
This thread shows how the agent remembers previous calculations.
	• First Turn:
Python

 config1 specifies which conversation we're in
config1 = { 'configurable': { 'thread_id': '1'} }
msg = "I want to buy 20 AMZN stocks using current price. Then 15 MSFT. What will be the total cost?"
graph.invoke({"messages": [{"role": "user", "content": msg}]}, config=config1)

The agent uses the get_stock_price tool for both AMZN and MSFT, calculates the total, and responds. The state, including this total, is saved under thread_id: '1'.
	• Second Turn (Follow-up):
Python

msg = "Using the current price tell me the total price of 10 RIL stocks and add it to previous total cost"
graph.invoke({"messages": [{"role": "user", "content": msg}]}, config=config1)

Because we use the same config1, the agent retrieves the history for thread_id: '1'. It understands what "previous total cost" refers to, fetches the price for RIL, and adds the new amount to the old total.
	
Conversation 2: thread_id: '2' (A Separate Chat)
This thread runs in parallel and is completely independent of the first one.
	• First Turn:
Python

config2 = { 'configurable': { 'thread_id': '2'} }
msg = "Tell me the current price of 5 AAPL stocks."
graph.invoke({"messages": [{"role": "user", "content": msg}]}, config=config2)

The agent calculates the cost for 5 AAPL stocks. This state is saved under thread_id: '2'.
	• Second Turn (Follow-up):
Python

msg = "Tell me the current price of 5 MSFT stocks and add it to previous total"
graph.invoke({"messages": [{"role": "user", "content": msg}]}, config=config2)

	The agent, using config2, retrieves the history for thread_id: '2'. The "previous total" it recalls is the cost of the Apple stocks, not the total from Conversation 1. It then adds the cost of 5 MSFT stocks to it.
	
	
	The code uses two threads simply for demonstration purposes. There isn't a technical limit of two; you could create as many threads as needed. Using two is just the clearest way to show the system's key feature: its ability to handle multiple, isolated conversations at the same time without mixing them up.
	
	The Two Parallel Conversations
	A thread_id acts like a unique identifier for a specific chat session. Here’s a breakdown of the two independent conversations in the code:
	Conversation 1 (Thread '1') 💰
	This conversation focuses on a cumulative calculation.
		1. First Message: The user asks for the total cost of 20 AMZN stocks and 15 MSFT stocks. The agent calculates this total and saves it in the memory associated with thread_id: '1'.
		2. Follow-up Message: The user then asks to add the cost of 10 RIL stocks to the "previous total cost." Because the agent is using the same thread_id, it correctly recalls the first total and adds the new amount to it.
	Conversation 2 (Thread '2') 📈
	This conversation is completely separate and has its own independent memory.
		1. First Message: The user asks for the price of 5 AAPL stocks. The agent calculates this amount and stores it in the memory for thread_id: '2'.
		2. Follow-up Message: The user asks to add the cost of 5 MSFT stocks to the "previous total." The agent accesses the memory for thread_id: '2', recalling the total from the AAPL stock query. It correctly ignores everything that happened in Conversation 1.
	In short, the two threads demonstrate that the agent can maintain distinct contexts, just like having two separate chat windows open with a chatbot.
 Sources<img width="1007" height="2528" alt="image" src="https://github.com/user-attachments/assets/49975d08-e3d0-4bf6-b26c-1665099108ff" />
