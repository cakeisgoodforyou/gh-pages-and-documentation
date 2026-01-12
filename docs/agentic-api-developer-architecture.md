---
layout: default
title: Agentic API Developer - Architecture
---

# Architecture Deep Dive

## System Overview

The Agentic API Developer implements a **multi-agent orchestration pattern** where three specialized AI agents work together to autonomously generate and execute API integration code.

[← Back to Main Documentation](index)

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                          User Interface                          │
│                    (API / CLI / Web Console)                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Orchestrator Agent                          │
│                     (Claude 3 Haiku)                            │
│  • Workflow coordination                                        │
│  • Sequential task management                                   │
│  • Error handling & retries                                     │
└───────┬─────────────────────────────────────────┬───────────────┘
        │                                         │
        ▼                                         ▼
┌──────────────────────┐              ┌─────────────────────────┐
│  invoke_generator    │              │  invoke_executor        │
│      Lambda          │              │      Lambda             │
└──────┬───────────────┘              └───────┬─────────────────┘
       │                                      │
       ▼                                      ▼
┌──────────────────────┐              ┌─────────────────────────┐
│  Generator Agent     │              │  Executor Agent         │
│  (Claude Haiku 4.5)  │              │  (Claude 3 Haiku)       │
│  • Writes Python code│              │  • Validates code       │
│  • API knowledge     │              │  • Executes in sandbox  │
└──────┬───────────────┘              └───────┬─────────────────┘
       │                                      │
       ▼                                      ▼
┌──────────────────────┐              ┌─────────────────────────┐
│  generate_code       │              │  validate_code          │
│      Lambda          │              │      Lambda             │
│  • Cleans code       │              │  • AST parsing          │
│  • Stores in S3      │              │  • Security checks      │
└──────────────────────┘              └───────┬─────────────────┘
                                              │
                                              ▼
                                      ┌─────────────────────────┐
                                      │  execute_code           │
                                      │      Lambda             │
                                      │  • Runs code            │
                                      │  • Calls Stripe API     │
                                      └─────────────────────────┘
```

---

## Component Details

### 1. Orchestrator Agent

**Purpose:** Master coordinator that manages the entire workflow

**Responsibilities:**
- Receives user requests
- Delegates code generation to Generator Agent
- Delegates code execution to Executor Agent
- Returns final results to user

**Model:** Claude 3 Haiku (optimized for fast coordination, high quotas)

**Key Features:**
- Sequential task execution (no parallelism)
- Error propagation and reporting
- Session management
- Never writes code itself - always delegates

**Action Groups:**
- `invoke_generator` - Triggers code generation
- `invoke_executor` - Triggers code execution

---

### 2. Generator Agent

**Purpose:** AI code writer specialized in API integrations

**Responsibilities:**
- Understands natural language task descriptions
- Writes complete, executable Python code
- Includes error handling and best practices
- Calls storage service for code persistence

**Model:** Claude Haiku 4.5 (latest model for best code quality)

**Key Features:**
- Generates code from scratch (not templates)
- Understands multiple APIs (Stripe, GitHub, OpenWeather)
- Includes proper imports and structure
- Uses environment variables for API keys

**Action Groups:**
- `generate_code` - Stores generated code

**Example Code Output:**
```python
import stripe
import os

stripe.api_key = os.environ.get('STRIPE_API_KEY')

if not stripe.api_key:
    raise ValueError("STRIPE_API_KEY not set")

try:
    customer = stripe.Customer.create(
        name='John Doe',
        email='john@example.com'
    )
    print(f"Created customer: {customer.id}")
except stripe.error.StripeError as e:
    print(f"Stripe error: {e}")
```

---

### 3. Executor Agent

**Purpose:** Code validator and execution manager

**Responsibilities:**
- Validates Python syntax and structure
- Performs security checks
- Executes code in sandbox environment
- Returns execution results

**Model:** Claude 3 Haiku (sufficient for validation tasks, high quotas)

**Key Features:**
- Two-phase execution (validate, then execute)
- Sandboxed Python environment
- Captures stdout/stderr
- Prevents dangerous operations

**Action Groups:**
- `validate_code` - AST parsing and security
- `execute_code` - Sandbox execution

---

## Lambda Functions

### invoke_generator Lambda

**Trigger:** Orchestrator Agent  
**Purpose:** Bridge between Orchestrator and Generator  
**Runtime:** Python 3.12  
**Memory:** 512 MB

**Flow:**
1. Receives task description from Orchestrator
2. Invokes Generator Agent via `bedrock-agent-runtime`
3. Parses streaming response
4. Returns generated code to Orchestrator

**Key Code:**
```python
response = bedrock_agent_runtime.invoke_agent(
    agentId=GENERATOR_AGENT_ID,
    agentAliasId=GENERATOR_ALIAS_ID,
    sessionId=str(uuid.uuid4()),
    inputText=f"Generate code for: {task_description}"
)
```

---

### invoke_executor Lambda

**Trigger:** Orchestrator Agent  
**Purpose:** Bridge between Orchestrator and Executor  
**Runtime:** Python 3.12  
**Memory:** 512 MB

**Flow:**
1. Receives Python code from Orchestrator
2. Invokes Executor Agent
3. Parses execution results
4. Returns output to Orchestrator

---

### generate_code Lambda

**Trigger:** Generator Agent  
**Purpose:** Validate and store generated code  
**Runtime:** Python 3.12  
**Memory:** 1024 MB

**Flow:**
1. Receives raw Python code from Generator
2. Cleans markdown formatting (removes \`\`\`python)
3. Stores in S3 with timestamp
4. Returns cleaned code

**S3 Key Format:**
```
generated/stripe/Create_a_Stripe_customer_20260112_070704.py
```

---

### validate_code Lambda

**Trigger:** Executor Agent  
**Purpose:** Security and syntax validation  
**Runtime:** Python 3.12  
**Memory:** 512 MB

**Validation Checks:**
- ✅ Valid Python syntax (AST parsing)
- ✅ No dangerous imports (subprocess, eval, etc.)
- ✅ No filesystem writes
- ✅ No network access (except allowed APIs)

**Blocked Operations:**
```python
DANGEROUS_MODULES = [
    'subprocess', 'os.system', 'eval', 
    'exec', '__import__', 'compile'
]
```

---

### execute_code Lambda

**Trigger:** Executor Agent  
**Purpose:** Sandbox code execution  
**Runtime:** Python 3.12  
**Memory:** 1024 MB  
**Dependencies:** stripe==11.1.1

**Security Features:**
- Restricted `__builtins__`
- Limited global scope
- No filesystem access
- Captures stdout/stderr
- Timeout protection (30s)

**Environment:**
```python
restricted_globals = {
    '__builtins__': __builtins__,
    'os': os,  # For environment variables only
    'json': json,
    'stripe': stripe,  # Allowed API SDK
    'print': print
}

exec(code, restricted_globals)
```

**Logs Stored In:**
- CloudWatch: Real-time execution logs
- S3: Execution results as JSON

---

## Data Flow

### Successful Request Flow

```
1. User: "Create a Stripe customer named Alice"
   ↓
2. Orchestrator thinks: "I need to generate code, then execute it"
   ↓
3. Orchestrator → invoke_generator(task="Create Stripe customer Alice")
   ↓
4. invoke_generator → Generator Agent
   ↓
5. Generator Agent thinks: "I'll write Python code using Stripe SDK"
   ↓
6. Generator Agent → generate_code(code="import stripe...", task="...")
   ↓
7. generate_code Lambda:
   - Cleans code
   - Stores: s3://bucket/generated/stripe/Create_Stripe_20260112.py
   - Returns: {code: "...", s3_key: "..."}
   ↓
8. Generator Agent → Returns code to invoke_generator
   ↓
9. invoke_generator → Returns code to Orchestrator
   ↓
10. Orchestrator → invoke_executor(code="import stripe...")
    ↓
11. invoke_executor → Executor Agent
    ↓
12. Executor Agent → validate_code(code="...")
    ↓
13. validate_code Lambda:
    - Parses AST ✓
    - Checks security ✓
    - Returns: {valid: true}
    ↓
14. Executor Agent → execute_code(code="...")
    ↓
15. execute_code Lambda:
    - Loads Stripe API key from Secrets Manager
    - Executes code in sandbox
    - Captures output: "Created customer: cus_abc123"
    - Stores log: s3://bucket/executions/stripe/success_20260112.json
    - Returns: {success: true, output: "Created customer: cus_abc123"}
    ↓
16. Executor Agent → Returns result to invoke_executor
    ↓
17. invoke_executor → Returns result to Orchestrator
    ↓
18. Orchestrator → Returns to user:
    "Successfully created Stripe customer Alice (cus_abc123)"
```

**Total Time:** ~8-12 seconds  
**API Calls:** 6 (3 agents × 2 invocations each)  
**Cost:** ~$0.01

---

### Error Handling Flow

```
1. Validation fails at validate_code
   ↓
2. validate_code returns: {valid: false, error: "SyntaxError"}
   ↓
3. Executor Agent sees validation error
   ↓
4. Executor Agent returns error to Orchestrator
   ↓
5. Orchestrator returns to user:
   "Code validation failed: SyntaxError at line 5"
```

---

## Infrastructure Components

### S3 Buckets

**Code Storage Bucket**
- Name: `agentic-api-developer-{env}-code-storage`
- Purpose: Store generated Python code
- Structure:
  ```
  ├── generated/
  │   ├── stripe/
  │   │   └── Create_customer_20260112.py
  │   └── github/
  └── executions/
      └── stripe/
          └── success_20260112.json
  ```
- Encryption: AES-256
- Versioning: Enabled
- Lifecycle: Optional (delete after 30 days)

---

### Secrets Manager

**Stripe API Key Secret**
- Name: `agentic-api-developer/stripe/test-key`
- Format: 
  ```json
  {"api_key": "sk_test_..."}
  ```
- Encryption: KMS
- Access: Only execute_code Lambda

---

### CloudWatch Log Groups

| Log Group | Retention | Purpose |
|-----------|-----------|---------|
| `/aws/lambda/...-invoke-generator` | 7 days | Agent invocation logs |
| `/aws/lambda/...-invoke-executor` | 7 days | Agent invocation logs |
| `/aws/lambda/...-generate-code` | 7 days | Code generation logs |
| `/aws/lambda/...-validate-code` | 7 days | Validation logs |
| `/aws/lambda/...-execute-code` | 7 days | Execution logs |
| `/aws/bedrock/agents/...` | 7 days | Agent thinking logs |

---

### IAM Roles & Policies

**Agent Roles** (3)
- Orchestrator Agent Role
- Generator Agent Role
- Executor Agent Role

**Permissions:**
- `bedrock:InvokeModel` - Use Claude models
- `bedrock:InvokeAgent` - Call other agents
- `s3:PutObject` / `s3:GetObject` - Code storage

**Lambda Roles** (5)
- invoke_generator Lambda Role
- invoke_executor Lambda Role
- generate_code Lambda Role
- validate_code Lambda Role
- execute_code Lambda Role

**Permissions:**
- `bedrock:InvokeAgent` - Call Bedrock agents
- `s3:PutObject` / `s3:GetObject` - Code storage
- `secretsmanager:GetSecretValue` - Stripe API key (execute_code only)
- `logs:CreateLogGroup` / `logs:PutLogEvents` - CloudWatch

---

## Security Architecture

### Defense in Depth

**Layer 1: Agent Prompts**
- Agents told to never execute dangerous operations
- Explicit instructions to validate before execution

**Layer 2: IAM Permissions**
- Least privilege access
- Agents can only invoke designated actions
- Lambdas have minimal S3/Secrets access

**Layer 3: Code Validation**
- AST parsing catches syntax errors
- Security checks block dangerous imports
- Pattern matching for risky operations

**Layer 4: Sandbox Execution**
- Restricted Python environment
- No subprocess/os.system
- No file writes
- Limited global scope
- Timeout protection

**Layer 5: Network Isolation**
- execute_code Lambda has no VPC access
- Only outbound HTTPS to Stripe API
- No inbound connections

---

## Scalability Considerations

### Current Limits

- **Concurrent executions:** 1 (sequential by design)
- **Request rate:** Limited by Bedrock quotas (~10-100 req/min)
- **Code execution timeout:** 30 seconds
- **Max code size:** 5 MB (Lambda limit)

### Scaling Strategies

**For Higher Throughput:**
1. Request Bedrock quota increases
2. Use multiple agent instances
3. Implement request queuing (SQS)
4. Add parallel execution paths

**For Multiple APIs:**
1. Create API-specific agent variants
2. Add router agent for API selection
3. Separate Lambda functions per API

**For Production:**
1. Add DynamoDB for session state
2. Implement circuit breakers
3. Add API rate limiting
4. Use Step Functions for complex workflows

---

## Monitoring & Observability

### Key Metrics

**Agent Performance:**
- Invocation count per agent
- Average response time
- Error rate by agent
- Token usage

**Lambda Performance:**
- Invocation count
- Duration
- Error rate
- Throttles

**Code Quality:**
- Syntax error rate
- Security violation rate
- Execution success rate
- API call success rate

## Cost Breakdown

### Per Request Estimate

| Component | Cost |
|-----------|------|
| Orchestrator Agent (2 calls) | $0.002 |
| Generator Agent (2 calls) | $0.003 |
| Executor Agent (2 calls) | $0.002 |
| Lambda executions (5) | $0.001 |
| S3 storage | $0.000 |
| Secrets Manager call | $0.000 |
| **Total per request** | **~$0.008** |

### Monthly Estimate (1000 requests)

| Component | Cost |
|-----------|------|
| Bedrock API calls | $3.00 |
| Lambda | $.50 |
| S3 storage (1 GB) | $0.10 |
| Secrets Manager | $0.40 |
| CloudWatch Logs | $1.00 |
| **Total** | **~$5.00** |

---

[← Back to Main Documentation](index)