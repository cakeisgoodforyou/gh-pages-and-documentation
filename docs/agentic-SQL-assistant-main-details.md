# Agentic ECS SQL Assistant - Main & Workflow Details

## main.py Execution Flow

### 1. Initialization Phase

```python
args = parser.parse_args()
config = load_config(args.config)
logger = setup_logging(config)
```

### 2. Component Initialization

```python
# Initialize all components
sql_generator, athena_executor, sqs_handler, google_chat, workflow = initialize_components(config)
```

**What `initialize_components()` creates:**

| Component | Initialization | Key Config |
|-----------|----------------|------------|
| **sql_generator** | `SQLGeneratorAgent(llm, config['sql_generator'])` | System prompt, max_refinements |
| **athena_executor** | `AthenaExecutor(workgroup, database, output_location, ...)` | Workgroup, database, S3 path, polling settings |
| **sqs_handler** | `SQSApprovalHandler(request_queue_url, response_queue_url, region)` | Two SQS queue URLs |
| **google_chat** | `GoogleChatNotifier(webhook_url)` or `None` | Google Chat webhook (optional) |
| **workflow** | `SQLAssistantWorkflow(sql_generator, athena_executor, sqs_handler, google_chat, config)` | All above components + workflow config |

**SQS Handler Requirements:**
- `request_queue_url`: Where approval requests are sent
- `response_queue_url`: Where approval responses are received

**Google Chat Notifier:**
- `webhook_url`: If null or "null", notifier is set to `None`
- Workflow checks `if self.google_chat:` before sending notifications

### 3. Schema Loading

```python
# Fetch database schema from AWS Glue
glue_schema = get_glue_schema(
    database=config['athena']['database'],
    catalog_id=config['glue'].get('catalog_id'),
    region=config['glue']['region']
)
```

**Returns:** Dictionary of table schemas
```python
{
    "customer": {
        "columns": [{"Name": "c_custkey", "Type": "bigint"}, ...],
        "partition_keys": [],
        ...
    },
    "orders": {...},
    ...
}
```

### 4. Workflow Execution

```python
# Generate unique thread ID
thread_id = f"session_{datetime.now().strftime('%Y%m%d_%H%M%S')}"
# Run the workflow
final_state = workflow.run(
    user_question=user_question,
    glue_schema=glue_schema,
    thread_id=thread_id
)
```

### 5. Results Display

```python
# Extract final state and display results
if final_state:
    for node_output in final_state.values():
        if node_output.get('execution_success'):
            # Display: rows, data scanned, execution time, S3 path
        elif node_output.get('error'):
            # Display error
        elif node_output.get('approval_timeout'):
            # Display timeout message
```

---