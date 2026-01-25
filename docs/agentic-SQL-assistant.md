# Agentic ECS SQL Assistant - Project Overview

## Purpose

Deploy a SQL query assistant that converts natural language questions into SQL queries, checks for refinements or human feedback and then once approved executes the code using AWS Athena, and returns results. 

Features human-in-the-loop approval via SQS queues with optional configu provided for Google Chat notifications.

## Key Features

- **Schema-Aware**: Automatically loads database schema from AWS Glue catalog for context
- **Human Approval Workflow**: Agents pause for human review before execution.  Feedback given via SQS queues.
- **Query Refinement**: Humans can request modifications to generated SQL
- **Asynchronous Execution**: Runs on ECS Fargate, polls SQS for approval responses
- **Optional Notifications**: Sends rich info cards to Google Chat (optional)

## Technology Stack

| Component | Technology |
|-----------|------------|
| **LLM** | AWS Bedrock (Claude Sonnet 4) |
| **Orchestration** | LangGraph (state machine workflow) |
| **Data Warehouse** | AWS Athena |
| **Schema Source** | AWS Glue Data Catalog |
| **Approval Queue** | AWS SQS (request & response queues) |
| **Notifications** | Google Chat Webhooks (optional) |
| **Compute** | AWS ECS Fargate |
| **Container Registry** | AWS ECR |
| **Infrastructure** | Terraform |

## Use Cases

1. **Ad-hoc Data Analysis**: Non-SQL users can query data using natural language
2. **Exploratory Analytics**: Quickly iterate on queries with refinement workflow
3. **Governed Query Execution**: Human approval ensures queries are reviewed before execution
4. **Learning Tool**: See how natural language translates to SQL

## Project Structure

```
agentic-ecs-sql-assistant/
├── agents/
│   └── sql_generator_agent.py      # NL → SQL conversion
├── utils/
│   ├── athena_executor.py          # Query execution
│   ├── sqs_approval_handler.py     # SQS messaging
│   └── google_chat_notifier.py     # Optional notifications
├── workflows/
│   └── sql_assistant_workflow.py   # LangGraph orchestration
├── scripts/
│   ├── approve_query.py            # CLI approval tool
│   └── approval.json               # Approval template
├── config/
│   └── sql_assistant_config.yaml   # Configuration
├── terraform/
│   └── ECS, ECR, s3, IAM etc        # Infrastructure
├── main.py                          # Entry point
├── Dockerfile                       # Container definition
├── deploy.sh                        # Build and push image
└── requirements.txt                 # Python dependencies
```

## Workflow Summary

1. **Generate**: Agent generates SQL from natural language question
2. **Request Approval**: SQL sent to SQS, optional Google Chat notification
3. **Human Review**: Human examines SQL via `approve_query.py --list-pending`
4. **Decision**: Human approves, refines, or rejects via `approval.json`
5. **Execute**: Approved query runs on Athena
6. **Results**: Data returned, saved to S3 and optionally sent to Google Chat

## Stage 1 Scope (Current)

✅ SQL generation from natural language  
✅ SQS-based approval workflow  
✅ Query refinement capability  
✅ Athena query execution  
✅ Google Chat notifications (optional)  
✅ ECS Fargate deployment  

## Future Enhancements

**Stage 2: Visualization**
- Generate charts from query results
- Save visualizations to S3

## Quick Start

```bash
# Run locally
python main.py --question "Show me 5 customers"

# Run on ECS
aws ecs run-task \
  --cluster agentic-ecs-dev \
  --task-definition sql-assistant \
  --overrides '{"containerOverrides":[{"name":"sql-assistant","command":["python","main.py","--question","Your question here"]}]}'

# Check pending approvals
python scripts/approve_query.py --list-pending

# Approve a query
# 1. Create approval.json with thread_id and action
# 2. Run: python scripts/approve_query.py
```