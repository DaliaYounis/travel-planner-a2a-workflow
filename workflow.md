[← Documentation home](README.md)

# Workflow

## User journey

1. Open the client and enter a destination, days, total USD budget, and traveler count.
2. Select **Plan trip**. Previous output is cleared and form inputs are disabled.
3. Watch **Discovery** as the orchestrator reads the three agent cards.
4. Watch **Research and budget** while those two specialists run concurrently.
5. Watch **Itinerary** after both prerequisite results pass validation.
6. Review the itinerary, expand days, inspect the budget and tips, and read the plan checks.
7. Select **Clear** to remove the output and progress, or submit another request after planning ends.

## Execution flow

```mermaid
flowchart TD
  Form[Trip form] --> Validate[API validates request]
  Validate -->|Invalid| Bad[HTTP 400 JSON error]
  Validate -->|Valid| Discover[Open SSE stream and discover agents]
  Discover --> Search[Search agent: MCP web search and model summary]
  Discover --> Budget[Budget agent: model estimate and calculated totals]
  Search --> Join[Wait for both completed results and validate them]
  Budget --> Join
  Join --> Itinerary[Itinerary agent: draft and assemble plan]
  Itinerary --> Checks[Orchestrator validates plan and scores checks]
  Checks --> Result[SSE result then done]
  Result --> UI[Browser renders itinerary]
```

The implementation is in `server/services/orchestration/planTrip.js`. Agent discovery is sequential. Search and budget execution use `Promise.all`; itinerary generation starts only after both succeed.

```mermaid
sequenceDiagram
  participant UI as Browser
  participant API as Orchestrator
  participant S as Search agent
  participant B as Budget agent
  participant I as Itinerary agent
  participant MCP as MCP search server
  UI->>API: POST /api/plan/stream with trip JSON
  API->>S: GET agent card
  API->>B: GET agent card
  API->>I: GET agent card
  par Research
    API->>S: SendMessage(request)
    S-->>API: Working task and ID
    S->>MCP: web_search(query)
    MCP-->>S: Search results
    API->>S: GetTask until completed
    S-->>API: Structured research artifact
  and Budget
    API->>B: SendMessage(request)
    B-->>API: Working task and ID
    API->>B: GetTask until completed
    B-->>API: Structured budget artifact
  end
  API->>I: SendMessage({request, search, budget})
  I-->>API: Working task and ID
  API->>I: GetTask until completed
  I-->>API: Structured itinerary artifact
  API->>API: Validate and score
  API-->>UI: result {plan, evaluation}
  API-->>UI: done {ok: true}
```

Each specialist uses a structured model completion internally. The diagram abbreviates repeated polling; progress events reach the browser throughout execution.

## Browser stream events

The browser sends a POST using `fetch` and reads an SSE response. It does not open an A2A connection to any agent.

| Event | Payload | Effect |
| --- | --- | --- |
| `phase` | `phase`, `message` | Updates the phase indicator and log. Values are `discovery`, `parallel`, and `synthesis`. |
| `agent` | `name`, `card` | Adds discovered agent details to the board and log. |
| `task` | `name`, `state`, `taskId` | Updates task status and logs a shortened ID. |
| `result` | `plan`, `evaluation` | Renders the plan and checks, and clears the phase indicator. |
| `error` | `message` | Records an error that the stream reader surfaces to the page. |
| `done` | `ok` | Marks stream completion; success uses `true`, handled failure uses `false`. |

See `server/controllers/planController.js`, `client/src/services/planApi.js`, and `client/src/hooks/useTripPlanner.js`.

## Failure and cancellation paths

- Invalid input returns HTTP 400 before SSE starts.
- Discovery, tool, model, task, or result-validation errors stop the workflow. For a connected browser, the controller sends `error`, then `done` with `ok: false`.
- An agent returns `TASK_STATE_FAILED` with a text error artifact when its executor fails. The orchestrator turns that failure into a planning error.
- Each A2A HTTP request has a 15-second deadline. Each task polling loop has a 120-second deadline and waits 400 ms between unfinished polls.
- If either parallel specialist fails, remaining orchestration requests are aborted and itinerary generation is skipped.

The app does not implement a workflow-level retry or reconnection/resume mechanism. After addressing an error, submit a new request.
