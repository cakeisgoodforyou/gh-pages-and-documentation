# Gen AI Data Engineering Assistant

## Overview
An AI-powered data analytics assistant that uses LangGraph workflows to plan, generate, and execute data queries with human-in-the-loop approval gates.

## Architecture
- **Framework**: LangGraph (state-based workflow orchestration)
- **LLM**: Google Vertex AI (Gemini 2.5 Flash)
- **Runtime**: Flask app on Google Cloud Run
- **Data Platform**: Google BigQuery
- **Storage**: Google Cloud Storage
- **Messaging**: Google Pub/Sub (for approvals)

## Core Components

### Agents
1. **Orchestrator** - Creates execution plans from user prompts
2. **Generator** - Generates SQL/Python code for each step
3. **Executor** - Executes code against BigQuery/GCS
4. **Analyzer** - Analyzes execution results
5. **Error Refiner** - Refines failed steps

### Workflow States
- **Planning** - Orchestrator creates step-by-step plan
- **Generation** - Generator creates executable code
- **Approval** - Human approves plan/code via Pub/Sub
- **Execution** - Executor runs code
- **Analysis** - Analyzer summarizes results
- **Refinement** - Error handling and retry logic

## Deployment

### Prerequisites
- GCP Project with enabled APIs:
  - Cloud Run
  - Artifact Registry
  - BigQuery
  - Cloud Storage
  - Pub/Sub
  - Vertex AI

### Terraform Setup
```bash
cd terraform/
terraform init
terraform plan -var="project_id=YOUR_PROJECT_ID"
terraform apply
```

### Docker Build & Deploy
```bash
./build.sh
```

## API Endpoints

### POST /run
Start a workflow with a user prompt or pre-defined plan.

curl -X POST <my-cloudrun-url>/run \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "TEST1",
    "prompt": "please get the schema for all tables in the project_id bigquery-public-data and dataset thelook_ecommerce and analyze which tables would be good testing customer analytics around churn"
  }'


**Request**:
```json
{
  "id": "request-123",
  "prompt": "Show me sales by region last week"
}
```

**Or with pre-defined plan**:
```json
{
  "id": "request-456",
  "plan_path": "gs://bucket/request-456/plan.json"
}
```

**Response**:
```json
{
  "status": "COMPLETE",
  "request_id": "request-123",
  "plan": {...},
  "execution": {...},
  "results": {...}
}
```

### GET /health
Health check endpoint.

## Approval Workflow

1. Workflow publishes approval request to Pub/Sub topic
2. Human approver pulls message and reviews plan/code
3. Approver publishes response (approve/reject/modify)
4. Workflow polls response subscription and continues

### CLI Tool
```bash
python scripts/approval_cli.py --environment dev
```

## Configuration

### Environment Variables
- `PROJECT_ID` - GCP project ID
- `ENVIRONMENT` - Environment name (dev/staging/prod)
- `GCS_BUCKET` - Bucket for workflow data
- `APPROVAL_TIMEOUT_SECONDS` - Timeout for human approval

### LLM Configuration
Edit `config/agent_llm_config.yaml`:
```yaml
default:
  project_id: "your-project-id"
  location: "us-central1"
  temperature: 0.2
  
agents:
  orchestrator:
    model: "gemini-2.5-flash"
```

## State Management

State flows through workflow as Pydantic models:
- `AgentState` - Top-level state container
- `MetaState` - Request metadata
- `PlanState` - Execution plan with steps
- `ExecutionState` - Execution history
- `ResultsState` - Final outputs and analysis

## Security Considerations

⚠️ **Before deploying**, review and update:
1. Hardcoded project ID in `config/agent_llm_config.yaml`
2. Hardcoded project ID in `variables.tf`
3. Public Cloud Run access (dev environment only by default)
4. Service account permissions in terraform files
5. Pub/Sub IAM bindings for approval workflows

## Local Development

```bash
# Install dependencies
pip install -r requirements.txt

# Set environment variables
export PROJECT_ID=your-project-id
export ENVIRONMENT=dev

# Run locally
python main.py
```

## Monitoring & Debugging

- Cloud Run logs: View in GCP Console
- State dumps: Logged at INFO level after each agent
- Pub/Sub dead letter queues: For failed approvals
- Execution records: Stored in state for each step