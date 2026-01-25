# Agentic ECS SQL Assistant - Architecture

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  ECS Fargate Task (main.py)                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  1. Load Config & Initialize Components                   │  │
│  │     - SQLGeneratorAgent (Bedrock LLM)                     │  │
│  │     - AthenaExecutor                                      │  │
│  │     - SQSApprovalHandler                                  │  │
│  │     - GoogleChatNotifier (optional)                       │  │
│  │     - SQLAssistantWorkflow (LangGraph)                    │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  2. Fetch Schema from Glue Catalog                        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                          │                                       │
│                          ▼                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  3. Run LangGraph Workflow                                │  │
│  │                                                            │  │
│  │     ┌─────────────────────────────────────────┐           │  │
│  │     │  Node: generate_sql                     │           │  │
│  │     │  → Calls Bedrock to generate SQL        │           │  │
│  │     └─────────────────────────────────────────┘           │  │
│  │                     │                                      │  │
│  │                     ▼                                      │  │
│  │     ┌─────────────────────────────────────────┐           │  │
│  │     │  Node: await_human_approval             │           │  │
│  │     │  → Sends to SQS request queue           │◄──────────┼──┼─── SQS Request Queue
│  │     │  → Sends to Google Chat (optional)      │           │  │
│  │     │  → Polls SQS response queue (10 min)    │◄──────────┼──┼─── SQS Response Queue
│  │     └─────────────────────────────────────────┘           │  │
│  │                     │                                      │  │
│  │                     ▼                                      │  │
│  │     ┌─────────────────────────────────────────┐           │  │
│  │     │  Conditional Router                     │           │  │
│  │     │  → "approve" → execute_query            │           │  │
│  │     │  → "refine"  → handle_refinement        │           │  │
│  │     │  → "reject"  → finalize                 │           │  │
│  │     └─────────────────────────────────────────┘           │  │
│  │             │           │            │                     │  │
│  │             ▼           ▼            ▼                     │  │
│  │     ┌──────────┐  ┌──────────┐  ┌──────────┐             │  │
│  │     │ execute  │  │ refine   │  │ finalize │             │  │
│  │     │ _query   │  │ (loop)   │  │          │             │  │
│  │     └──────────┘  └──────────┘  └──────────┘             │  │
│  │          │                            │                   │  │
│  │          ▼                            ▼                   │  │
│  │     ┌──────────────────────────────────────┐             │  │
│  │     │  Node: finalize                      │             │  │
│  │     │  → Send workflow details to chat     │             │  │
│  │     └──────────────────────────────────────┘             │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ├─────────► AWS Bedrock (Claude)
                          ├─────────► AWS Athena (query execution)
                          ├─────────► AWS Glue (schema catalog)
                          ├─────────► S3 (query results)
                          └─────────► Google Chat (optional notifications)

┌─────────────────────────────────────────────────────────────────┐
│  Human Operator (Laptop/Anywhere)                               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  1. Check pending approvals                               │  │
│  │     python scripts/approve_query.py --list-pending        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                          │                                       │
│                          ▼                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  2. Create approval.json                                  │  │
│  │     {                                                      │  │
│  │       "thread_id": "session_xxx",                         │  │
│  │       "action": "approve",                                │  │
│  │       "feedback": null                                    │  │
│  │     }                                                      │  │
│  └───────────────────────────────────────────────────────────┘  │
│                          │                                       │
│                          ▼                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  3. Send approval                                         │  │
│  │     python scripts/approve_query.py                       │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────┬───────────────────────────────────────────────────┘
              │
              └──────────► SQS Response Queue
```

## Component Details

### 1. SQLGeneratorAgent
- **Purpose**: Converts natural language to SQL
- **LLM**: Claude Sonnet 4 via AWS Bedrock
- **Input**: User question + Glue schema context
- **Output**: Structured JSON (SQL, explanation, confidence, tables_used)
- **Capabilities**: 
  - Initial SQL generation
  - SQL refinement based on human feedback
  - Schema-aware query construction

### 2. AthenaExecutor
- **Purpose**: Execute SQL queries on AWS Athena
- **Operations**:
  - Start query execution
  - Poll for completion (configurable timeout/interval)
  - Retrieve results as pandas DataFrame
  - Return metadata (rows, data scanned, execution time)
- **Output**: Results saved to S3, returned to workflow

### 3. SQSApprovalHandler
- **Purpose**: Manage human approval workflow via SQS
- **Two Queues**:
  - **Request Queue**: ECS task sends approval requests here
  - **Response Queue**: Human approval script sends responses here
- **Operations**:
  - `send_approval_request()`: Sends SQL + metadata to request queue
  - `poll_for_approval()`: Long-polls response queue (20s intervals, 30min timeout)
  - `send_approval_response()`: Used by CLI script to send approval
  - `get_pending_approvals()`: Lists all pending requests

### 4. GoogleChatNotifier (Optional)
- **Purpose**: Send rich card notifications to Google Chat
- **Operations**:
  - `send_approval_request()`: Sends formatted card with SQL, explanation, thread_id
  - `send_query_result()`: Sends execution summary (rows, cost, time, S3 path)
  - `send_simple_message()`: Sends plain text updates
- **Graceful Degradation**: Workflow continues if webhook is null/unavailable

### 5. SQLAssistantWorkflow (LangGraph)
- **Purpose**: Orchestrate the multi-step agent workflow
- **State Management**: WorkflowState TypedDict tracks:
  - User question, schema, thread_id
  - Generated SQL, explanation, confidence
  - Human approval status, feedback
  - Query results, execution status
  - Error messages
- **Nodes**:
  - `generate_sql`: Calls SQLGeneratorAgent
  - `await_human_approval`: Sends to SQS, polls for response
  - `execute_query`: Calls AthenaExecutor
  - `handle_refinement`: Regenerates SQL with feedback
  - `finalize`: Sends final notifications
- **Conditional Routing**: Based on human approval decision


## Infrastructure Components

### AWS Resources (Terraform)
- **ECS Cluster**: Runs Fargate tasks
- **ECS Task Definition**: Container config, IAM role, env vars
- **ECR Repository**: Stores Docker images
- **SQS Queues** (2):
  - `agentic-ecs-dev-sql-approval-requests`
  - `agentic-ecs-dev-sql-approval-responses`
- **SQS Dead Letter Queues** (2): For failed messages
- **IAM Roles**:
  - Task execution role (pull image, write logs)
  - Task role (Bedrock, Athena, Glue, SQS, S3)
- **CloudWatch Log Group**: `/ecs/agentic-ecs-dev-sql-assistant`

## Failure Modes & Recovery

| Failure | Detection | Recovery |
|---------|-----------|----------|
| **LLM timeout/error** | Exception in generate_sql_node | Workflow ends, error logged |
| **Athena query failure** | execute_query returns success=False | Finalize node reports error |
| **SQS approval timeout** | 30 min poll timeout reached | Workflow ends, "timeout" status |
| **Malformed approval** | JSON parsing error in CLI script | User notified, no message sent |
| **Google Chat webhook down** | HTTP error | Warning logged, workflow continues |
| **Schema not found** | Glue API error | Workflow exits early |

## Monitoring & Observability

- **CloudWatch Logs**: All stdout/stderr from ECS task
- **SQS Metrics**: Message counts, age, dead letter queue depth
- **Athena Query History**: All executed queries visible in Athena console
- **Workflow State**: Logged at each node transition
- **Agent Outputs**: SQL generation, analysis logged to CloudWatch
