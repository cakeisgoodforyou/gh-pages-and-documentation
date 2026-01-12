---
layout: default
title: Agentic API Developer
---

# Agentic API Developer

### A Multi-Agent System for generating and testing python code designed to interact with third party APIs 

This system featured Orchestrator, Generator and Executor agents that work together to generate and test python code for 
interacting with development endpoints for third party APIs.

Built with Infrastructure as Code (Terraform), this system demonstrates advanced agent orchestration patterns, AI-powered development workflows and is designed to be 
a secure and production ready deployment.

- [View Code on GitHub](https://github.com/cakeisgoodforyou/agentic-api-developer) 
- [See Architecture](agentic-api-developer-architecture)

---

## Key Features
- ✅ **Multi-Agent Orchestration**: Three specialized agents working together autonomously
- ✅ **AI Code Generation**: Claude writes production-ready Python code for API integrations  
- ✅ **Fully Automated Deployment**: Zero click-ops, complete Terraform IaC
- ✅ **Secure Sandbox Execution**: Isolated Python execution environment with validation
- ✅ **Real API Integration**: Live Stripe API testing with secret management
- ✅ **Code Versioning**: All generated code stored in S3 with timestamps
- ✅ **Production Security**: Least privilege IAM, encrypted secrets, rate limiting

---

## What Gets Deployed

***Note that costs are approximate and will vary depending on your usage. Review AWS pricing documentation before deployment and rollback all infra when finished with the project!***

| Component | Purpose | Cost/Month |
|-----------|---------|------------|
| 3 Bedrock Agents | Orchestrator, Generator, Executor | ~$2.00 |
| 5 Lambda Functions | Agent actions & code execution | ~$0.50 |
| S3 Bucket | Store generated code | ~$0.10 |
| Secrets Manager | Store API keys (Stripe) | ~$0.40 |
| CloudWatch Logs | Agent & Lambda logging | ~$1.00 |
| **Total** | | **~$5/month** |

---

## Architecture Overview

```
User Request
    ↓
Orchestrator Agent (Claude Haiku)
    ↓
invoke_generator Lambda
    ↓
Generator Agent (Claude Haiku 4.5)
    ├─ Writes Python code using AI
    └─ Calls generate_code Lambda to validates & store code in S3
    ↓
Orchestrator receives code
    ↓
invoke_executor Lambda
    ↓
Executor Agent (Claude Haiku)
    ├─ Calls validate_code Lambda
    │   └─ Additional AST parsing & security checks
    └─ Calls execute_code Lambda
        └─ Calls secret manager for auth key / creds
        └─ Sandbox execution (default Stripe SDK)
    ↓
Return results to user
```

---

## Quick Start

### Prerequisites
- AWS Account with Bedrock access enabled
- Terraform >= 1.5
- AWS CLI configured
- Stripe test API key

### 1. Clone and Setup

```bash
git clone https://github.com/yourusername/agentic-api-developer.git
cd agentic-api-developer
```

### 2. Build Lambda Packages

```bash
cd src/lambda/execute_code
pip install -r requirements.txt -t dependencies/
cd ../../../

chmod +x build_all.sh
./build_all.sh
```

### 3. Configure Terraform Backend

Edit `terraform/backend.tf`:

```hcl
terraform {
  backend "s3" {
    bucket         = "<your-terraform-state-bucket>"
    key            = "agentic-api-developer/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "<your-state-lock-table>"
    encrypt        = true
  }
}
```

### 4. Configure Project Settings

Edit `terraform/locals.tf`:

```hcl
locals {
  project_name = "agentic-api-developer"
  environment  = "dev"  # or test, prod
  region       = "us-east-1"
}
```

### 5. Store Stripe API Key

```bash
aws secretsmanager create-secret \
  --name agentic-api-developer/stripe/test-key \
  --secret-string '{"api_key":"sk_test_YOUR_STRIPE_KEY"}' \
  --region us-east-1
```

Get your test key from: https://dashboard.stripe.com/test/apikeys

### 6. Deploy Infrastructure

```bash
cd terraform
terraform init
terraform apply
```

Review the plan and type `yes` to deploy.

---

## Post Deployment Testing

### Test via Python Script

Open debug-trace.py file and add or update your own test cases as required.
Recommended to start with existing test cases to confirm all is working as expected.

Run:
```bash
usage-and-testing/debug-trace.py <orchestrator_agent_id> <orchestrator_agent_alias>
```

### Verify Results

**1. Check Stripe Dashboard**
- Visit: https://dashboard.stripe.com/test/customers
- Look for the newly created customer

**2. Check Generated Code in S3**
```bash
aws s3 ls s3://agentic-api-developer-dev-code-storage/generated/stripe/ --recursive
```

**3. View CloudWatch Logs**
```bash
aws logs tail /aws/lambda/agentic-api-developer-dev-execute-code --follow
```

---

## Agent Prompts

### Orchestrator Agent
Coordinates the workflow between Generator and Executor agents. Never writes code itself - delegates to specialists.

### Generator Agent  
Writes production-ready Python code using Claude's knowledge. Includes:
- Proper imports and error handling
- Environment variable API key management
- Comprehensive Stripe error catching
- Clean, executable code

### Executor Agent
Validates and executes generated code:
1. AST parsing for syntax validation
2. Security checks (blocks dangerous operations)
3. Sandbox execution with limited permissions
4. Returns execution results

---

## Testing Scenarios

### Scenario 1: Create Customer
```python
"Create a Stripe test customer named Alice Smith with email alice@test.com"
```

### Scenario 2: List Customers
```python
"List the last 5 Stripe customers"
```

### Scenario 3: Payment Intent
```python
"Create a Stripe payment intent for 2500 cents (25 USD) with currency usd"
```

---

## Troubleshooting

### Rate Limiting Errors

**Symptom:** `ThrottlingException` in CloudWatch logs

**Solution:** Wait 2-3 minutes between tests. Bedrock has request rate limits.

**Long-term fix:** Request quota increase in AWS Service Quotas console for Claude models.

### "No module named 'stripe'" Error

**Symptom:** Execute code Lambda fails with import error

**Solution:** 
```bash
cd src/lambda/execute_code
pip install -r requirements.txt -t dependencies/
cd ../../../
./build_all.sh
cd terraform
terraform apply
```

### Agent Not Calling Actions

**Symptom:** Agent responds conversationally but doesn't invoke Lambdas

**Solution:** Prepare a new agent version:
```bash
AGENT_ID=$(terraform output -raw orchestrator_agent_id)
aws bedrock-agent prepare-agent --agent-id "$AGENT_ID" --region us-east-1
sleep 30
terraform apply
```

### Invalid Model Identifier

**Symptom:** `validationException: The provided model identifier is invalid`

**Solution:** Verify you're using models available in Bedrock Agents (not just Runtime):
```bash
aws bedrock list-foundation-models \
  --region us-east-1 \
  --by-provider anthropic \
  --query 'modelSummaries[].modelId'
```

Update `terraform/variables.tf` with valid model IDs.

---

## Security Considerations

### IAM Permissions
- All roles follow least-privilege principle
- Agents can only invoke their designated actions
- Lambdas have minimal S3/Secrets access
- Production deployment should further restrict permissions

### Code Execution Sandbox
- No filesystem write access
- No network access (except Stripe API)
- Limited Python builtins
- Blocks dangerous operations (subprocess, eval, etc.)

### API Key Management
- Stripe keys stored in AWS Secrets Manager
- Encrypted at rest
- Never logged or exposed in code
- Separate secrets per environment

### Generated Code Storage
- All code versioned with timestamps
- Stored in private S3 bucket
- Encrypted at rest
- Lifecycle policies recommended for production

---

## Cost Optimization Tips

1. **Use Claude 3 Haiku for orchestration** (lower cost, adequate for coordination)
2. **Set S3 lifecycle policies** (delete old generated code after 30 days)
3. **Use smaller Lambda memory** (512MB sufficient for most operations)
4. **Delete test deployments** when not in use
5. **Monitor CloudWatch logs retention** (7 days sufficient for dev)

---

## Extending the System

### Add New APIs

1. **Update Generator prompt** (`src/prompts/generator.txt`)
   - Add example of where to get the auth key for new API
   - Include API-specific error handling

2. **Update execute_code Lambda**
   - Add new API uses an SDK, add to `requirements.txt`
   - Create secret for new API key
   - Update IAM permissions for Secrets access

### Futre Features

- **Multi-API workflows** (combine Stripe + email notifications)
- **Code refinement loops** (Refiner agent fixes errors)
- **Discovery agent** (KB integration for API documentation)
- **Approval workflows** (human-in-the-loop for sensitive operations)

---

## Project Structure

```
agentic-api-developer/
├── src/
│   ├── action_groups/          # OpenAPI schemas for agent actions
│   │   ├── execute_code_schema.json
│   │   ├── generate_code_schema.json
│   │   ├── invoke_executor_schema.json
│   │   ├── invoke_generator_schema.json
│   │   └── validate_code_schema.json
│   ├── lambda/                 # Lambda function handlers
│   │   ├── execute_code/
│   │   │   ├── handler.py
│   │   │   └── requirements.txt
│   │   ├── generate_code/
│   │   │   └── handler.py
│   │   ├── invoke_executor/
│   │   │   └── handler.py
│   │   ├── invoke_generator/
│   │   │   └── handler.py
│   │   └── validate_code/
│   │       └── handler.py
│   └── prompts/                # Agent system prompts
│       ├── executor.txt
│       ├── generator.txt
│       └── orchestrator.txt
├── terraform/                  # Infrastructure as Code
│   ├── agent_executor.tf
│   ├── agent_generator.tf
│   ├── agent_orchestrator.tf
│   ├── backend.tf
│   ├── cloudwatch.tf
│   ├── data.tf
│   ├── iam_agent_roles.tf
│   ├── iam_lambda_roles.tf
│   ├── lambda.tf
│   ├── locals.tf
│   ├── outputs.tf
│   ├── s3.tf
│   ├── variables.tf
│   └── versions.tf
├── build_all.sh
└── README.md
```

---

## Important Callouts

### Test Env Isolation and Blocked Operations

- ✅ No dangerous imports (subprocess, eval, etc.)
- ✅ No filesystem writes

**Blocked Operations:**
```python
DANGEROUS_MODULES = [
    'subprocess', 'os.system', 'eval', 
    'exec', '__import__', 'compile'
]
```

### Model Selection

The project uses **mixed Claude models** for optimal cost/performance:

- **Orchestrator:** Claude 3 Haiku (high quota, fast coordination)
- **Generator:** Claude Haiku 4.5 (latest model, best code generation)
- **Executor:** Claude 3 Haiku (high quota, simple validation tasks)

This balances quality where it matters (code generation) with cost efficiency.

### Properties Array Format

Bedrock Agents send parameters as a **properties array**, not a simple JSON object:

```json
{
  "properties": [
    {"name": "task_description", "value": "..."},
    {"name": "api_name", "value": "stripe"}
  ]
}
```

All Lambda handlers include parsing logic to convert this to a dict.

### Agent Preparation Required

After any changes to:
- Agent prompts
- Foundation models
- Action groups

You must **prepare a new version** and update the alias:

```bash
aws bedrock-agent prepare-agent --agent-id <AGENT_ID>
terraform apply
```

### Rate Limits

Claude models in Bedrock have conservative rate limits for new accounts:
- Claude 3 Haiku: ~100 requests/minute
- Claude Haiku 4.5: ~10 requests/minute (via inference profiles)

Request quota increases via AWS Service Quotas for production use.

---

## Clean Up

To avoid ongoing charges:

```bash
cd terraform

# Delete all resources
terraform destroy

# Confirm with 'yes'
```

Also delete:
- Stripe secret in Secrets Manager
- CloudWatch log groups (if desired)
- S3 Terraform state bucket

---