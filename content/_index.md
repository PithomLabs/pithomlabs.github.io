The integration of the Restate Go SDK with traditional Golang web frameworks, databases, and distributed systems introduces fundamental pain points primarily centered around enforcing **durability** and **determinism**—principles that often conflict with native Go concurrency, external side effects, and typical microservice patterns.

Here are the key pain points and the required adaptations per do's and don'ts at the [Restate Go framework Rea](https://github.com/pithomlabs/rea)

---

## 1. Determinism Violations (Databases, External APIs, SaaS)

The most significant pain point is ensuring that operations with side effects or non-deterministic outcomes (which include almost all interactions with external systems like databases, SaaS APIs, or IaaS APIs) do not violate Restate's replay mechanism.

|Pain Point|Detail & Impact|Required Solution / Adaptation|
|:--|:--|:--|
|**Direct External/DB Calls**|**Never call external APIs or databases directly** from a durable handler. Doing so causes the call to **re-execute during replay**, resulting in duplicate side effects (e.g., charging a customer twice, creating duplicate logs, or inconsistent state updates).|**Wrap all non-deterministic operations in `restate.Run()`** (or `restate.RunAsync()`). Restate journals the outcome, ensuring the operation runs **exactly once**, and subsequent replays use the recorded result.|
|**Context Misuse in Run Block**|Inside the `restate.Run` or `ctx.run` closure (which receives `restate.RunContext`), you **cannot use the parent Restate context** (e.g., `ctx.get`, `ctx.sleep`, or nested `ctx.run`). Accidentally capturing the outer context breaks determinism.|Use the limited `restate.RunContext` exclusively within the closure, which is restricted to performing the side effect operation. Utility functions like `RunDo` prevent accidental capture of the outer context.|
|**Non-Deterministic Time/Random**|Using native Go functions like **`time.Now()`**, **`time.Sleep()`**, or **`rand.Float64()`** is forbidden because they return different values on replay, breaking the replay logic.|Use **`restate.Sleep()`** instead of `time.Sleep()` for durable delays. Use **`restate.Rand()`** or **`restate.UUID()`** for deterministic random numbers or IDs. Wrap `time.Now()` inside a `restate.Run` block to record the timestamp deterministically.|
|**Global State**|Using **global variables** is non-durable and the state is lost across replicas or retries.|Durable state management must exclusively use **`restate.Get()`** and **`restate.Set()`** (available in Virtual Objects and Workflows).|

## 2. Concurrency and Golang Web Frameworks

Integrating the Restate Go SDK requires abandoning native Go concurrency primitives for any durable, blocking Restate operation.

|Pain Point|Detail & Impact|Required Solution / Adaptation|
|:--|:--|:--|
|**Native Concurrency Misuse**|**Do not use Go's native goroutines, channels, or select statements** to combine blocking Restate operations (`ctx.call`, `future.Result()`, `ctx.sleep`). This prevents Restate from journaling the execution order, leading to non-determinism errors (RT0016) upon replay.|Use Restate's **durable combinators** like **`restate.Wait()`** (wait for all), **`restate.WaitFirst()`** (race operations/timeouts), and the Future pattern to coordinate asynchronous steps deterministically. Also, **do not call blocking future methods** (like `Response()` or `Result()`) **from inside a standard Go routine**.|
|**External/Internal Client Separation**|Standard Golang web frameworks (running outside Restate) **do not have the durable `restate.Context`**. Using the durable SDK client logic outside a handler results in calls bypassing the durability engine.|External applications (e.g., standard HTTP handlers/web frameworks) must exclusively use the specialized **`ingress.NewClient()`** to trigger Restate services. Conversely, you **must never use the Ingress Client inside a Restate handler**.|
|**Blocking Virtual Objects**|Sleeping or waiting for an external event (Awakeable) in an **exclusive handler** of a Virtual Object will **block all other calls to that specific object key**, impacting concurrency and leading to potential deadlocks or poor performance for that entity.|Avoid long pauses or external waits in exclusive Virtual Object handlers. Instead, use **Delayed Messages** to schedule future work, allowing the current handler to complete and unblock the key.|
|**Deadlocks in Objects**|Complex request-response patterns between **exclusive handlers of Virtual Objects** can cause cross-deadlocks (A $\to$ B and B $\to$ A) or cycle deadlocks, especially if calls are made on the same object key.|Carefully design communication flows in Virtual Objects to avoid mutual or cyclic requests between exclusive handlers, or use shared handlers for read-only access where possible.|

## 3. Integrating Pub/Sub and SaaS (External Coordination)

Interfacing with asynchronous systems that deliver callbacks (like webhooks from payment gateways or messages from Pub/Sub systems) requires using specific Restate primitives.

|Pain Point|Detail & Impact|Required Solution / Adaptation|
|:--|:--|:--|
|**Waiting for External Events**|Waiting for a signal from a system **outside** of the Restate environment (like a SaaS platform completing a task and sending a webhook) cannot use internal workflow primitives.|Use **`Awakeables`** (available in Basic Services and Virtual Objects). The service generates a unique opaque ID, which is passed to the external system. The external system then calls the Restate ingress API using this ID to resolve the `Awakeable` and resume the durable execution.|
|**Internal vs. External Signals**|Confusing internal signaling with external callbacks. For example, using an `Awakeable` to coordinate between handlers inside the same Workflow.|Use **`Durable Promises`** _exclusively_ within **Workflows** for coordinating signals and events between different handlers inside that workflow.|
|**Idempotency on Ingress**|When external systems (like a pub/sub consumer or SaaS webhook sender) retry sending events or making calls to Restate, they can cause duplicate handler execution if not managed.|The external client **must provide an idempotency key** in the call options or request header when invoking a service, ensuring exactly-once processing even if the external system retries the call.|

## 4. IaaS/PaaS/Deployment Constraints

Deploying Go services as durable Restate handlers imposes strict requirements on the infrastructure configuration and deployment workflow.

|Pain Point|Detail & Impact|Required Solution / Adaptation|
|:--|:--|:--|
|**Code Versioning (PaaS/IaaS)**|Changing the underlying service code (e.g., updating the container image) **without registering a new, immutable deployment version** causes a **Journal Mismatch** (RT0016) or non-deterministic errors when Restate attempts to replay old invocations.|Always deploy code changes using new, **immutable deployment versions**. The old version must be kept around until long-running invocations complete.|
|**Service Endpoint Security**|If the Go service is exposed as a publicly routable endpoint (common in serverless/PaaS deployment models), unrestricted access is dangerous.|**HTTPS must be used** between Restate and your service. Access must be secured, typically by using a validator or middleware to cryptographically verify the request signature, ensuring calls originate only from the correct Restate environment.|
|**Workflow Longevity and Cost**|Workflow state and execution history are only retained for a configured period (default 24 hours to 90 days). Long-running workflows (e.g., > 90 days) or those with frequent state updates can exceed limits.|Developers must actively configure workflow retention policy and monitor state size. State intended for indefinite, complex querying should remain in a separate database. Leveraging Restate's **suspension** feature (when waiting for timers/events) is crucial to **save costs** on Function-as-a-Service (FaaS) platforms.|

In essence, integrating the Restate Go SDK requires shifting from a non-deterministic, concurrent, shared-memory programming model (native Go/web frameworks/databases) to a **durable, deterministic, asynchronous message-passing model**. All external interaction must be explicitly mediated by the Restate Context primitives (`restate.Run`, `restate.Sleep`, `restate.Wait`, `Awakeable`) to maintain the exactly-once and fault-tolerant guarantees.

Integrating AI agents (especially those utilizing large language models or complex tool chains) with the Restate Go SDK introduces specific pain points and anti-patterns rooted in managing **stateful memory, long execution times, non-deterministic outputs**, and **resilient tool execution** within a durable environment.

The sources identify several requirements and constraints related to AI agents and development practices that highlight these pain points:

---

## 5. AI Agents

### 1. Managing Durable and Consistent Agent State (Memory)

A critical pain point is ensuring the agent maintains its conversational memory or context across multiple turns and survives infrastructure failures without corruption or loss of history.

| Pain Point | Detail & Impact | Required Restate Solution / Adaptation |
| :--- | :--- | :--- |
| **Non-Durable Context/Memory** | If the AI agent's session history (e.g., conversation memory, tool use context) is stored in native Go memory or global variables, it is **lost across service restarts, retries, or replica changes**. | The agent session must be implemented using a **Virtual Object**. The agent's persistent memory and context must be stored exclusively in the **K/V store using `restate.Get()` and `restate.Set()`**. |
| **Concurrent State Corruption** | If multiple user interactions try to update the same agent session simultaneously, standard microservice patterns lead to race conditions and lost updates, corrupting the chat integrity. | The Virtual Object guarantees **single-writer consistency per object key**. The agent's logic must be implemented in **exclusive handlers** (using `restate.ObjectContext`) to ensure only one message is processed at a time per session ID, thus maintaining chat integrity. |
| **Deadlocks due to Mutual Calls** | Complex AI workflows sometimes involve agents calling other agents (or the same agent key recursively) to delegate tasks. This pattern in Virtual Objects causes **deadlocks** if done via exclusive handlers. | **Do not put mutually calling agents within the same Virtual Object keyed instance**. Avoid complex request-response calls between exclusive handlers of Virtual Objects that lead to cross deadlocks (A → B and B → A) or cycle deadlocks. |

### 2. Ensuring Deterministic Tool Execution (Side Effects)

AI agents frequently execute external functions or "tools" (e.g., database lookups, API calls) which are non-deterministic side effects. Restate's durability conflicts with these tools if they are not correctly wrapped.

| Pain Point | Detail & Impact | Required Restate Solution / Adaptation |
| :--- | :--- | :--- |
| **Non-Resilient Tool Execution** | If a tool execution (an external HTTP call or database query) fails transiently (e.g., network timeout) and is not made durable, the agent flow might break or lose the tool's output needed for the next LLM step. | **Wrap all tool executions using `restate.Run()`** (or equivalent durable step feature). This makes tool calls resilient, ensures the operation runs exactly once, and persists the durable outcome in the execution journal. |
| **Loss of LLM Response on Retry** | If the LLM interaction itself is treated as a single, non-durable step, the costly response generated by the LLM is lost if the service crashes before the response is processed or recorded. | The LLM response itself (which is often a side effect) should be persisted. This is achieved by wrapping the model with the **`durableCalls` middleware** (or ensuring the result is part of a `restate.Run` block) to ensure **every LLM response is saved and can be replayed**. |
| **Long Latency and Timeouts** | LLM calls can be inherently slow (up to 3 minutes or more). The default 1-minute service timeout might be too short for these operations, leading to premature termination and retries. | Adjust the service's **inactivity timeout and abort timeout** settings to accommodate long-running operations, such as LLM calls. Implement **timeouts** for long-running external processes. |

### 3. Handling Agent Failures and Flow Control

Integrating AI agents means dealing with complex execution flows, requiring specific error handling and concurrency patterns.

| Pain Point | Detail & Impact | Required Restate Solution / Adaptation |
| :--- | :--- | :--- |
| **Incorrect Error Handling in Tools** | If an external AI agent framework (like the OpenAI SDK) catches internal Restate errors (such as suspension or a terminal error) before Restate can process them, it breaks the durable execution flow. | When defining tools for the OpenAI Agent SDK, set the `failure_error_function` to **raise Restate errors** (e.g., `TerminalError`) to ensure these critical errors are handled by Restate, rather than being consumed by the agent framework. |
| **Non-Deterministic Concurrency** | Trying to run multiple tool calls in parallel using native Go goroutines breaks determinism. | To parallelize AI agent tool steps, implement an orchestrator tool that uses **durable execution combinators** like `restate.Wait`, `restate.WaitFirst`, or `restate.RunAsync` to run multiple steps in parallel deterministically. |
| **Non-Deterministic ID/Time Generation** | Generating session IDs, request IDs, or timestamps within the agent logic using standard Go libraries (`time.Now()`, `rand.Float64()`, `uuid.New()`) results in replay mismatches. | Use the SDK's deterministic helpers: **`restate.UUID(ctx)`** or **`restate.Rand()`** for stable random values and unique IDs. Wrap `time.Now()` inside a `restate.Run` block to record the timestamp deterministically. |

### 4. Development and Tooling Overhead

Even outside of runtime constraints, developers face pain points when writing agent code.

| Pain Point | Detail & Impact | Required Restate Solution / Adaptation |
| :--- | :--- | :--- |
| **Anti-Pattern Misuse** | The complexity of durable execution means developers commonly make mistakes like misusing the parent context inside `restate.Run`. | Use helper wrappers like **`RunDo`** (which restricts the closure input to `restate.RunContext`) or **`SafeRun`** to provide minimal compile-time safety and clear documentation against accidentally capturing and using the outer context (e.g., calling `ctx.Sleep()` inside `restate.Run`). |
| **Coding Agent Performance** | AI coding agents (like Cursor or Claude Code) may struggle to generate correct Restate code due to the deterministic constraints. | To improve the performance and behavior of these coding agents, **add the relevant SDK's `AGENTS.md` rules file** to the agent's context (e.g., in the `.cursor/rules` folder). |

In summary, integrating AI agents essentially forces the service developer to adopt the **Virtual Object** model to handle the mandatory requirements of persistent memory and single-writer concurrency. Every interaction that involves the LLM or tool usage must be strictly governed by Restate primitives like `restate.Run` to satisfy the deterministic and exactly-once execution guarantees.

---

## 6. Metaphorical Summary

Integrating the Restate Go SDK is like building a complex machine using LEGO blocks (Restate primitives) inside a clean room (the durable handler). You can't reach outside the clean room to grab raw tools (native Go concurrency, `time.Now()`, direct HTTP calls) or dirty components (external databases/APIs) without explicitly passing them through a durable, logged checkpoint (`restate.Run`). If you cheat and use a raw tool inside the clean room, the resulting structure will crumble during inspection (replay failure) because the inspectors cannot verify the exact steps taken. The external world (web framework, SaaS) must communicate only through a dedicated, secured receptionist (Ingress Client) rather than directly interacting with the builders.
