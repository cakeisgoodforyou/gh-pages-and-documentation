## SQLAssistantWorkflow - Detailed Flow

### Class Initialization

```python
def __init__(self, sql_generator, athena_executor, sqs_handler, google_chat_notifier, config):
    self.sql_generator = sql_generator
    self.athena_executor = athena_executor
    self.sqs_handler = sqs_handler
    self.google_chat = google_chat_notifier
    self.config = config
    
    # Build the workflow graph
    self.workflow = self._build_workflow()
    
    # Compile workflow (no checkpointer, SQS handles state)
    self.app = self.workflow.compile()
```

### Workflow Graph Construction (`_build_workflow`)

```python
workflow = StateGraph(WorkflowState)

# Add nodes (each is a function that processes state)
workflow.add_node("generate_sql", self._generate_sql_node)
workflow.add_node("await_human_approval", self._await_approval_node)
workflow.add_node("execute_query", self._execute_query_node)
workflow.add_node("handle_refinement", self._handle_refinement_node)
workflow.add_node("finalize", self._finalize_node)

# Set entry point
workflow.set_entry_point("generate_sql")

# Define edges (linear flow)
workflow.add_edge("generate_sql", "await_human_approval")

# Conditional routing after approval
workflow.add_conditional_edges(
    "await_human_approval",
    self._route_after_approval,  # Router function
    {
        "execute": "execute_query",
        "refine": "handle_refinement",
        "reject": "finalize"
    }
)

workflow.add_edge("handle_refinement", "await_human_approval")  # Loop back
workflow.add_edge("execute_query", "finalize")
workflow.add_edge("finalize", END)

return workflow.compile()
```

**WorkflowState Structure:**
```python
class WorkflowState(TypedDict):
    # Input
    messages: list                 # Conversation history
    user_question: str            # Original question
    glue_schema: dict             # DB schema
    thread_id: str                # Unique workflow ID
    
    # SQL Generation
    generated_sql: str            # Generated SQL
    sql_explanation: str          # Explanation
    tables_used: list[str]        # Tables in query
    confidence: str               # high/medium/low
    
    # Human Approval
    human_approval: bool | None   # Approval status
    human_feedback: str | None    # Refinement feedback
    refinement_count: int         # Number of refinements
    approval_timeout: bool        # Timeout flag
    
    # Execution
    query_results: dict | None    # Athena results
    execution_success: bool | None
    
    # Error Handling
    error: str | None
    next_step: str
```

---

## Node Execution Details

### Node 1: `_generate_sql_node`

**Called when:** Workflow starts OR after refinement

**What it does:**
```python
def _generate_sql_node(self, state: WorkflowState) -> WorkflowState:
    # Call SQLGeneratorAgent to generate SQL
    result = self.sql_generator.generate_sql(
        user_question=state["user_question"],
        schema_context=state["glue_schema"]
    )
    
    # Update state with results
    return {
        "generated_sql": result["sql"],
        "sql_explanation": result["explanation"],
        "tables_used": result["tables_used"],
        "confidence": result["confidence"],
        "messages": [AIMessage(content=f"Generated SQL: {result['sql']}...")]
    }
```

**Output:** State updated with generated SQL and metadata

---

### Node 2: `_await_approval_node`

**Called when:** After SQL generation

**What it does:**
```python
def _await_approval_node(self, state: WorkflowState) -> WorkflowState:
    thread_id = state["thread_id"]
    
    # 1. Send approval request to SQS
    sqs_success = self.sqs_handler.send_approval_request(
        thread_id=thread_id,
        user_question=state["user_question"],
        generated_sql=state["generated_sql"],
        explanation=state["sql_explanation"],
        confidence=state["confidence"],
        tables_used=state.get("tables_used", [])
    )
    
    # 2. Send notification to Google Chat (if configured)
    if self.google_chat:
        self.google_chat.send_approval_request(...)
    
    # 3. Poll SQS for approval response (blocks here)
    timeout_seconds = self.config.get('approval_timeout_seconds', 1800)
    poll_interval = self.config.get('sqs_polling_interval', 20)
    
    approval_response = self.sqs_handler.poll_for_approval(
        thread_id=thread_id,
        timeout_seconds=timeout_seconds,
        poll_interval=poll_interval
    )
    
    # 4. Return approval decision or timeout
    if approval_response is None:
        return {"human_approval": False, "approval_timeout": True, ...}
    else:
        return {
            "human_approval": approval_response.get('action'),  # "approve"/"refine"/"reject"
            "human_feedback": approval_response.get('feedback'),
            ...
        }
```

**Key behavior:**
- **Blocks** on `poll_for_approval()` for up to 30 minutes (default)
- Uses SQS long polling (20 second intervals)
- Returns immediately when approval received
- Returns timeout status if 10 minutes elapsed

---

### Router: `_route_after_approval`

**Called when:** After `_await_approval_node` completes

**What it does:**
```python
def _route_after_approval(self, state: WorkflowState) -> Literal["execute", "refine", "reject"]:
    action = state.get("human_approval")  # Contains "approve"/"refine"/"reject"
    
    if action == "approve":
        logger.info("Human approved - proceeding to execution")
        return "execute"
    elif action == "refine":
        logger.info("Human requested refinement")
        return "refine"
    elif action == "reject":
        logger.info("Human rejected - end processing")
        return "reject"
    else:
        # Default to reject if unclear
        logger.warning("Unclear approval status - defaulting to reject")
        return "reject"
```

**Returns:** String matching edge names in `add_conditional_edges`
- `"execute"` → goes to `execute_query` node
- `"refine"` → goes to `handle_refinement` node
- `"reject"` → goes to `finalize` node

---

### Node 3a: `_handle_refinement_node` (if action="refine")

**Called when:** Human requests changes

**What it does:**
```python
def _handle_refinement_node(self, state: WorkflowState) -> WorkflowState:
    refinement_count = state.get("refinement_count", 0)
    max_refinements = self.config.get('max_refinements', 3)
    
    # Check refinement limit
    if refinement_count >= max_refinements:
        return {"error": "Max refinements reached", ...}
    
    # Call agent to refine SQL
    result = self.sql_generator.refine_sql(
        original_sql=state["generated_sql"],
        user_feedback=state["human_feedback"],
        schema_context=state["glue_schema"]
    )
    
    # Update state with refined SQL
    return {
        "generated_sql": result["sql"],
        "sql_explanation": result["explanation"],
        "tables_used": result["tables_used"],
        "confidence": result["confidence"],
        "refinement_count": refinement_count + 1,
        "human_approval": None,  # Reset for next approval
        "human_feedback": None,
        ...
    }
```

**After this node:** Workflow loops back to `await_human_approval`

---

### Node 3b: `_execute_query_node` (if action="approve")

**Called when:** Human approves SQL

**What it does:**
```python
def _execute_query_node(self, state: WorkflowState) -> WorkflowState:
    sql = state["generated_sql"]
    database = self.config.get("database", "default")
    
    # Execute query on Athena
    result = self.athena_executor.execute_query(sql, database)
    
    if result["success"]:
        # Format success message
        rows = result["rows_returned"]
        data_gb = result["data_scanned_bytes"] / (1024**3)
        exec_time_sec = result["execution_time_ms"] / 1000
        
        return {
            "query_results": result,
            "execution_success": True,
            "messages": [AIMessage(content=f"Query executed successfully! Rows: {rows}, ...")]
        }
    else:
        return {
            "query_results": result,
            "execution_success": False,
            "error": result.get("error"),
            ...
        }
```

**`result` dictionary contains:**
```python
{
    'success': True,
    'data': DataFrame,            # Query results
    'rows_returned': 10,
    'data_scanned_bytes': 12345,
    'execution_time_ms': 1234,
    'query_execution_id': 'abc-123',
    'output_location': 's3://bucket/path/abc-123.csv'
}
```

---

### Node 4: `_finalize_node`

**Called when:** Workflow completes (approve/reject/timeout)

**What it does:**
```python
def _finalize_node(self, state: WorkflowState) -> WorkflowState:
    thread_id = state.get("thread_id")
    
    # Send results to Google Chat (if configured)
    if self.google_chat and state.get("execution_success"):
        results = state.get("query_results", {})
        self.google_chat.send_query_result(
            thread_id=thread_id,
            user_question=state.get("user_question"),
            success=True,
            rows_returned=results.get("rows_returned"),
            data_scanned_gb=results.get("data_scanned_bytes") / (1024**3),
            execution_time_sec=results.get("execution_time_ms") / 1000,
            output_location=results.get("output_location")
        )
    
    return {
        "messages": [AIMessage(content="Workflow completed")],
        "next_step": "complete"
    }
```

---

## Workflow Execution: `run()` Method

```python
def run(self, user_question: str, glue_schema: dict, thread_id: str):
    # Create initial state
    initial_state = {
        "messages": [HumanMessage(content=user_question)],
        "user_question": user_question,
        "glue_schema": glue_schema,
        "thread_id": thread_id,
        "refinement_count": 0,
        "approval_timeout": False
    }
    
    # Run workflow to completion
    final_state = None
    for event in self.app.stream(initial_state):
        final_state = event
        
        # Log progress
        for node_name, node_output in event.items():
            logger.debug(f"Node '{node_name}' completed")
            
            if 'messages' in node_output:
                for msg in node_output['messages']:
                    logger.info(f"Agent message: {msg.content}")
    
    return final_state
```

**Key points:**
- `self.app.stream()` executes the workflow step-by-step
- Each iteration yields current node's output
- Workflow blocks at `await_human_approval` node during SQS polling
- Returns final state when workflow reaches END

---


## Key Design Patterns

### 1. State Accumulation
Each node returns a partial state dict that gets merged into the full state:
```python
return {"generated_sql": sql, "confidence": "high"}
# This updates only those two fields, other fields persist
```

### 2. No Checkpointing
Unlike the original design, we don't use LangGraph checkpointing because:
- SQS handles state persistence (messages in queue)
- Each workflow run is independent
- No need to resume from interrupt (SQS polling blocks instead)

### 3. Conditional Routing
The router function returns a string that matches edge definitions:
```python
workflow.add_conditional_edges(source, router_fn, {"edge_name": "target_node"})
```

### 4. Loop Handling
Refinement loops back to approval:
```
generate_sql → await_approval → refine → await_approval → execute
```
Prevented from being infinite by `refinement_count` limit.

### 5. Error Propagation
Errors set `state["error"]` and typically route to `finalize` node for cleanup.
