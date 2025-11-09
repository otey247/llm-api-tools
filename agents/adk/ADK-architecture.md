# ADK architecture: When to use sub-agents versus agents as tools

**November 7, 2025**
Dharini Chandrashekhar
Sr Solutions Acceleration Architect

[Try Gemini 2.5](link_to_try_gemini)
Our most intelligent model is now available on Vertex AI

[Try now](link_to_try_now)

At its simplest, an agent is an application that reasons on how to best achieve a goal based on inputs and tools at its disposal.

![Diagram illustrating agent/tool interaction](https://storage.googleapis.com/gweb-cloudblog-publish/images/image1_oGjJbVH.max-800x800.png)

As you build sophisticated multi-agent AI systems with the Agent Development Kit (ADK), a key architectural decision involves choosing between a sub-agent and an agent as a tool. This choice fundamentally impacts your system's design, how well it scales, and its efficiency. Choosing the wrong pattern can lead to massive overhead — either by constantly passing full conversational history to a simple function or by under-utilizing the context-sharing capabilities of a more complex system.

While both sub-agents and tools help break down complex problems, they serve different purposes. The key difference is how they handle control and context.

## Agents as tools: The specialist on call

An agent as a tool is a self-contained expert agent packaged for a specific, discrete task, like a specialized function call. The main agent calls the tool with a clear input and gets a direct output, operating like a transactional API. The main agent doesn't need to worry about how the tool works; it only needs a reliable result. This pattern is ideal for independent and reusable tasks.

**Key characteristics:**

* **Encapsulated and reusable:** The internal logic is hidden, making the tool easy to reuse across different agents.
* **Isolated context:** The tool runs in its own session and cannot access the calling agent’s conversation history or state.
* **Stateless:** The interaction is stateless. The tool receives all the information it needs in a single request.
* **Strict input/output:** It operates based on a well-defined contract.

## Sub-agents: The delegated team member

A sub-agent is a delegated team member that handles a complex, multi-step process. This is a hierarchical and collaborative relationship where the sub-agent works within the broader context of the parent agent's mission. Use sub-agents for tasks that require a chain of reasoning or a series of interactions.

**Key characteristics:**

* **Tightly coupled and integrated:** Sub-agents are part of a larger, defined workflow.
* **Shared context:** They operate within the same session and can access the parent's conversation history and state, allowing for more nuanced collaboration.
* **Stateful processes:** They are ideal for managing processes where the task requires several steps to complete.
* **Hierarchical delegation:** The parent agent explicitly delegates a high-level task and lets the sub-agent manage the process.

Here is a simple decision matrix that you can use to guide your architectural decision based on the task:

| Criterion | Agent as a tool | Sub-agent | Decision |
| :--- | :--- | :--- | :--- |
| **Task complexity** | Low to Medium | High | Use a tool for atomic functions. Use a sub-agent for complex workflows. |
| **Context & state** | Isolated/None | Shared | If the task is stateless, use a tool. If it requires conversational context, use a sub-agent. |
| **Reusability** | High | Low to Medium | For generic, widely applicable capabilities, build a tool. For specialized roles in a specific process, use a sub-agent. |
| **Autonomy & control** | Low | High | Use a tool for a simple request-response. Use a sub-agent for delegating a whole sub-problem. |

## Use cases in action

Let's apply this framework to some real-world scenarios.

### Use case 1: The data agent (NL2SQL and visualization)

A business user asks for the top 5 product sales in Q2 by region and wants a bar chart.

*   **Root Agent**: Receives the business user's request (NL), determines the necessary steps (SQL generation → Execution → Visualization), and delegates/sequences the tasks, before returning the response to the user.
*   **NL2SQL Agent**: **Use a tool.** The task is a single, reusable function: convert natural language to a SQL string, using metadata & schema for grounding.
*   **Database Executor**: **Use a tool.** This is a simple, deterministic function to execute the query and return data.
*   **Data Visualization Agent**: **Use a sub-agent.** The task is complex and multi-step. It involves analyzing the data returned by the database tool, and the original user query, selecting the right chart type, generating the visualization code, and executing it. Delegating this to a sub-agent allows the main orchestrator agent to maintain a high-level view while the sub-agent independently manages its complex internal workflow.

### Use case 2: The sophisticated travel planner

A user asks to plan a 5-day anniversary trip to Paris, with specific preferences for flights, hotels, and activities. This is an ambiguous, high-level goal that requires continuous context and planning.

*   **Travel planner**: **Use a root agent**, to maintain the overall goal ("5-day anniversary trip to Paris"), manage the flow between sub-agents, and aggregate the final itinerary.

    Note: You could implement a Context/Memory Manager Tool accessible to all agents, potentially using a simple key-value store (like Redis or a simple database) to delegate the storage of immutable decisions.

*   **Flight search**: **Use a sub-agent.** The task is not a simple search; involving multiple back-and-forth interactions with the user (e.g., "Is a layover in Dubai okay?") while managing the overall trip context (dates, destination, class).
*   **Hotel booking**: **Use a sub-agent.** It needs to maintain state and context (dates, location preference, 5-star rating) as it searches for and presents options.
*   **Itinerary generation**: **Use a sub-agent** to generate a logical, day-by-day itinerary. The agent must combine confirmed flights/hotels with user interests (e.g., art museums, fine dining), potentially using its own booking tools.

Using tools is inefficient; each call requires the full trip context, leading to redundancy and state loss. Sub-agents are better for these stateful, collaborative processes as they share session context.

## Get started

The decision between sub-agents and agents as tools is fundamental to designing an effective and scalable agentic system in ADK. As a guiding principle, remember:

*   **Use tools for discrete, stateless, and reusable capabilities.**
*   **Use sub-agents to manage complex, stateful, and context-dependent processes.**

By mastering this architectural pattern, you can design multi-agent systems that are modular and capable of solving complex, real-world problems.

*   [Check out these examples on GitHub](link_to_github_examples) to start building using ADK.
*   [Here is a fantastic blogpost](link_to_blogpost) that will help you build your first multi-agent workflow.

Posted in
Developers & Practitioners
AI & Machine Learning
