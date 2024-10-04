# Agents

AI agents built on large language models control the path to solving a complex problem. They can typically act on feedback to refine their plan of action, a capability that can improve performance and help them accomplish more sophisticated tasks.

We recommend reading the [following article](https://research.ibm.com/blog/what-are-ai-agents-llm) to learn more.

## Implementation in Bee Agent Framework

An agent can be thought of as a program powered by LLM. The LLM generates structured output that is then processed by your program.

Your program then decides what to do next based on the retrieved content. It may leverage a tool, reflect, or produce a final answer.
Before the agent determines the final answer, it performs a series of `steps`. A step might be calling an LLM, parsing the LLM output, or calling a tool.

Steps are grouped in an `iteration`, and every update (either complete or partial) is emitted to the user.

### Bee Agent

Our Bee Agent is based on the `ReAct` ([Reason and Act](https://arxiv.org/abs/2210.03629)) approach.

Hence, the agent in each iteration produces one of the following outputs.

For the sake of simplicity, imagine that the input prompt is "What is the current weather in Las Vegas?"

First iteration:

```
thought: 'The user wants to know the current weather in Las Vegas. I can use OpenMeteo to get the current weather.'
tool_name: 'OpenMeteo'
tool_input: {"location": {"name": "Las Vegas"}, "start_date": "2024-10-04", "end_date": "2024-10-04", "temperature_unit": "celsius"}
tool_caption: 'Retrieving current weather forecast for Las Vegas.'
```

> [!NOTE]
>
> Agent emitted 4 complete updates in the following order (`thought, `tool_name`, `tool_input`, `tool_caption`) and tons of partial updates in the same order.
> Partial update means that new tokens are being added to the iteration. Updates are always in strict order: You first get many partial updates for thought, followed by a final update for thought (that means no final updates are coming for a given key).

Second iteration:

```
thought: "I have the current weather in Las Vegas. I can now provide the user with the current temperature and other weather information."
final_answer: "The first commercial typewriter was invented in 1874.";
```

For more complex tasks, the agent may do way more iterations.

In the following example, we will transform the knowledge gained into code.

```typescript
const response = await agent
  .run({ prompt: "What is the current weather in Las Vegas?" })
  .observe((emitter) => {
    emitter.on("update", async ({ data, update, meta }) => {
      // to log only valid runs (no errors), check if meta.success === true
      console.log(`Agent Update (${update.key}) 🤖 : ${update.value}`);
      console.log("-> Iteration state", data);
    });

    emitter.on("partialUpdate", async ({ data, update, meta }) => {
      // to log only valid runs (no errors), check if meta.success === true
      console.log(`Agent Partial Update (${update.key}) 🤖 : ${update.value}`);
      console.log("-> Iteration state", data);
    });

    // you can observe other events such as "success" / "retry" / "error" / "toolStart" / "toolEnd", ...

    // To see all events, uncomment the following code block
    // emitter.match("*.*", async (data: unknown, event) => {
    //   const serializedData = JSON.stringify(data).substring(0, 128); // show only part of the event data
    //   console.trace(`Received event "${event.path}"`, serializedData);
    // });
  });
```

### Behaviour

You can alter the agent's behavior in the following ways.

#### Setting execution policy

```typescript
await agent.run(
  { prompt: "What is the current weather in Las Vegas?" },

  {
    signal: AbortSignal.timeout(60 * 1000), // 1 minute timeout
    execution: {
      // How many times an agent may repeat the given step before it halts (tool call, llm call, ...)
      maxRetriesPerStep: 3,

      // How many retries can the agent occur before halting
      totalMaxRetries: 10,

      // Maximum number of iterations in which the agent must figure out the final answer
      maxIterations: 20,
    },
  },
);
```

> [!NOTE]
>
> The default is zero retries and no timeout.

##### Overriding prompt templates

The agent uses the following prompt templates.

1. System Prompt

2. User Prompt (to reformat the user's prompt)

3. User Empty Prompt

4. Tool Error

5. Tool Input Error (validation error)

6. Tool No Result Error

7. Tool Not Found Error

Please refer to the [following example](../examples/agents/bee_advanced.ts) to see how to modify them.

## Creating your own agent

To create your own agent, you must implement the agent's base class (`BaseAgent`).

The example can be found [here](../examples/agents/custom_agent.ts).