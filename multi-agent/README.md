# NovaMart Multi-Agent RAG

NovaMart is an enterprise-style customer-support application built with the
Strands Agents SDK, Amazon Bedrock, and Amazon Bedrock AgentCore. It combines
specialized agents, shared DynamoDB workflow state, parallel policy retrieval,
guardrails, memory, and AWS observability without changing the request flow
between the components.

![NovaMart architecture](diagrams/architecture-overview.png)

## What the project demonstrates

- A coordinator that routes each customer request to the appropriate worker.
- Inventory, policy, refund, and communication agents with focused tools.
- Parallel retrieval from returns, shipping, and warranty Knowledge Bases.
- Optimistic locking for shared workflow state in DynamoDB.
- Bedrock Guardrails for unsafe, private, or unrelated requests.
- AgentCore Runtime deployment, AgentCore Memory, CloudWatch, and X-Ray.
- Repeatable validation scripts for each major implementation milestone.

## Architecture

```text
Customer request
       |
       v
OrchestratorAgent
   |       |        |          |
   v       v        v          v
Inventory Policy  Refund  Communication
   |       |
   |       +--> ReturnsPolicyRetriever
   |       +--> ShippingPolicyRetriever
   |       +--> WarrantyPolicyRetriever
   |
   +--> DynamoDB order and customer data

All agents share WorkflowStateTable in DynamoDB.
```

![Agent and tool relationships](diagrams/agents-and-tools-reference.png)

### Agent responsibilities

| Agent | Responsibility |
| --- | --- |
| `OrchestratorAgent` | Classifies requests, starts sessions, and coordinates work. |
| `InventoryAgent` | Reads customer, order, and inventory information. |
| `PolicyAgent` | Runs the three policy retrievers concurrently and combines evidence. |
| `RefundAgent` | Evaluates return and refund eligibility and starts approved refunds. |
| `CommunicationAgent` | Produces the final customer-facing answer. |

![Request flow](diagrams/request-flow-scenarios.png)

## Repository layout

The existing directory structure is intentionally preserved:

```text
.
├── config.py                    # Environment and AWS resource configuration
├── requirements.txt             # Python dependencies
├── src/
│   ├── agent_orchestrator.py    # Agent graph and CLI entry point
│   ├── agent_observability.py   # Logging, tracing, and trace summaries
│   ├── agent_utils.py           # Shared output and workflow helpers
│   ├── agentcore_cli.py         # AgentCore CLI integration
│   ├── bedrock_kb_retrieval.py  # Bedrock Knowledge Base wrapper
│   └── demo.py                  # One local end-to-end request
├── infrastructure/
│   ├── starter_stack.yaml       # CloudFormation resources
│   ├── seed_data.py              # Deterministic demo data
│   └── cleanup.py               # Resource cleanup helper
├── tests/test_agent.py           # Milestone validation suite
├── diagrams/                    # Architecture diagrams
├── screenshots/                 # Validation evidence
└── agentcore/                   # AgentCore runtime and CDK configuration
```

## Requirements

- Python 3.12
- An AWS account with access to Bedrock in `us-east-1`
- Permissions for Bedrock, AgentCore, DynamoDB, S3, S3 Vectors,
  CloudFormation, CloudWatch, X-Ray, and `iam:PassRole`
- Node.js 20 or newer
- `uv`
- AWS credentials configured through the AWS CLI or another supported method

Install the AgentCore CLI:

```powershell
npm install -g @aws/agentcore@0.30.0
agentcore --version
```

## Local setup

```powershell
git clone https://github.com/ayomikunadaramola/novamart-multi-agent-rag.git
cd novamart-multi-agent-rag

py -3.12 -m venv .venv
.venv\Scripts\activate
python -m pip install -r requirements.txt
copy .env.example .env
python config.py
```

Open `.env` and provide the resource identifiers created for your AWS account.
Do not commit `.env` or credentials. The checked-in `.env.example` is the
source of truth for the required variable names.

## AWS deployment flow

### 1. Create the foundational resources

Deploy `infrastructure/starter_stack.yaml` using CloudFormation in
`us-east-1`. Wait for `CREATE_COMPLETE`, then seed the deterministic demo
records:

```powershell
python infrastructure/seed_data.py
```

### 2. Configure policy Knowledge Bases

Create the returns, shipping, and warranty Knowledge Bases using the policy
documents and S3 Vectors resources described by the stack. Copy the resulting
IDs into `.env`, then validate the configuration:

```powershell
python config.py
agentcore validate
```

### 3. Deploy the runtime

The deployment wrapper keeps the runtime definition in `agentcore/agentcore.json`
and invokes the AgentCore CLI:

```powershell
python src/agent_orchestrator.py deploy
```

Run deployment again after changing guardrail, Knowledge Base, or observability
IDs. Updating `.env` alone does not change an already deployed runtime.

## Running the application

Run a local end-to-end request:

```powershell
python src/demo.py
```

Start the interactive customer-support experience:

```powershell
python src/agent_orchestrator.py chat
```

Useful commands include:

```powershell
python src/agent_orchestrator.py status
python src/agent_orchestrator.py test
python src/agent_orchestrator.py deploy
```

## Validation

The test runner accepts a milestone name:

```powershell
python tests/test_agent.py task2  # Multi-agent orchestration
python tests/test_agent.py task3  # Runtime deployment and guardrails
python tests/test_agent.py task4  # AgentCore Memory
python tests/test_agent.py task5  # Knowledge Base retrieval
python tests/test_agent.py task6  # Observability
```

Run every available milestone with:

```powershell
python tests/test_agent.py
```

These checks contact AWS resources and therefore require valid credentials,
configured IDs, and a deployed environment. The included evidence records a
successful assessment of 120/120 points; it is not a production SLA.

## Validation evidence

| Area | Evidence |
| --- | --- |
| Overall assessment | [100 percent](screenshots/02-final-assessment-100-percent.png) · [120 of 120](screenshots/03-final-assessment-120-of-120.png) |
| Runtime readiness | [AgentCore runtime](screenshots/01-agentcore-runtime-ready.png) |
| Orchestration | [40 of 40](screenshots/04-multi-agent-orchestration-40-of-40.png) |
| Deployment | [Deployment passed](screenshots/05-agentcore-deployment-passed.png) |
| Invocation | [Runtime response](screenshots/06-runtime-invocation-response.png) |
| Policy workflow | [Premium return response](screenshots/07-premium-return-policy-response.png) |
| Memory | [100 percent](screenshots/08-agentcore-memory-100-percent.png) |
| Knowledge Bases | [Bedrock Knowledge Bases](screenshots/09-bedrock-knowledge-bases.png) |
| Observability | [Overview](screenshots/10-observability.png) · [Trace details](screenshots/13-xray-trace-details.png) · [Service map](screenshots/12-xray-service-map.png) · [17-span trace](screenshots/11-xray-distributed-trace-17-spans.png) |

## Troubleshooting

- **Missing resource IDs:** run `python config.py` and compare `.env` with
  `.env.example`.
- **Deployment validation fails:** run `agentcore validate`, confirm Node.js
  and `uv` are installed, and check the AWS region.
- **Retrieval returns no policy:** verify the Knowledge Base IDs, data source
  sync status, and IAM permissions.
- **Local output is noisy:** use the trace summary and CloudWatch logs; the
  demo intentionally exposes agent tool activity.
- **AWS calls fail:** confirm the active profile, region, and required
  permissions before rerunning a milestone.

## Cleanup

After collecting the evidence you need, remove deployed resources to avoid
ongoing AWS charges:

```powershell
python infrastructure/cleanup.py
```

Review the cleanup output and remove any resources created manually, including
Knowledge Bases, S3 buckets, S3 Vectors indexes, AgentCore Memory, and runtime
artifacts. Keep the screenshots if they are needed for assessment.

## License

See [LICENSE](LICENSE) for the project license.
