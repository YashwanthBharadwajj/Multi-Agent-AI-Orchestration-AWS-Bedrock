# Multi-Agent Orchestration System

A production-ready multi-agent AI system built on AWS Bedrock that orchestrates specialized agents for market intelligence, product analysis, and strategic report generation.

## Architecture Overview

This system implements a **supervisor-based orchestration pattern** where a central Supervisor Agent dynamically routes tasks to specialized worker agents based on the nature of the query. Each agent has access to specific tools and knowledge bases, enabling structured reasoning and collaborative problem-solving.

![Multi-Agent System Architecture](architecture_diagram.png)

---

## System Components

### 1. Supervisor Agent

The **Supervisor Agent** serves as the central orchestrator and entry point for all user interactions.

**Responsibilities:**
- Receives and analyzes incoming user prompts
- Decomposes complex queries into subtasks
- Routes tasks to appropriate specialized agents
- Aggregates responses from worker agents
- Maintains conversation context and state
- Handles error recovery and fallback logic

**Implementation Details:**
- Built on AWS Bedrock with Claude/Anthropic foundation model
- Uses function calling to invoke worker agents
- Implements retry logic with exponential backoff
- Maintains session state for multi-turn conversations

---

### 2. Product Insight Agent

The **Product Insight Agent** specializes in internal product knowledge and competitive analysis.

**Capabilities:**
- Retrieves product documentation and specifications
- Analyzes feature comparisons
- Provides historical product performance data
- Answers questions about internal product roadmaps

**Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│                  Product Insight Agent                   │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │           Product Knowledge Base                 │    │
│  │  ┌─────────────┐      ┌──────────────────┐      │    │
│  │  │ OpenSearch  │◄────►│  product-bucket  │      │    │
│  │  │   Index     │      │      (S3)        │      │    │
│  │  │             │      │                  │      │    │
│  │  │ - Embeddings│      │ - Product docs   │      │    │
│  │  │ - Metadata  │      │ - Specifications │      │    │
│  │  │ - Vectors   │      │ - Release notes  │      │    │
│  │  └─────────────┘      └──────────────────┘      │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │           Data Store Action Group               │    │
│  │  ┌─────────────┐      ┌──────────────────┐      │    │
│  │  │   Lambda    │─────►│  report-bucket   │      │    │
│  │  │  Function   │      │      (S3)        │      │    │
│  │  │             │      │                  │      │    │
│  │  │ - Format    │      │ - .txt outputs   │      │    │
│  │  │ - Validate  │      │ - Structured     │      │    │
│  │  │ - Store     │      │   reports        │      │    │
│  │  └─────────────┘      └──────────────────┘      │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**RAG Pipeline:**
1. User query is embedded using Amazon Titan Embeddings
2. OpenSearch performs semantic similarity search
3. Top-k relevant chunks are retrieved from product-bucket
4. Context is assembled and passed to the agent's LLM
5. Agent generates response grounded in retrieved documents

---

### 3. Market Analyst Agent

The **Market Analyst Agent** handles real-time market research and competitive intelligence.

**Capabilities:**
- Performs live web searches for market data
- Analyzes competitor announcements and news
- Gathers industry trends and analyst reports
- Synthesizes external market intelligence

**Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│                  Market Analyst Agent                    │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │           Web Search Action Group               │    │
│  │  ┌─────────────┐      ┌──────────────────┐      │    │
│  │  │   Lambda    │─────►│     Tavily       │──────┼───►│ Web
│  │  │  Function   │      │    Search API    │      │    │
│  │  │             │      │                  │      │    │
│  │  │ - Query     │      │ - Web search     │      │    │
│  │  │   formation │      │ - News search    │      │    │
│  │  │ - Result    │      │ - Structured     │      │    │
│  │  │   parsing   │      │   results        │      │    │
│  │  └─────────────┘      └──────────────────┘      │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │           Data Store Action Group               │    │
│  │  ┌─────────────┐      ┌──────────────────┐      │    │
│  │  │   Lambda    │─────►│  report-bucket   │      │    │
│  │  │  Function   │      │      (S3)        │      │    │
│  │  └─────────────┘      └──────────────────┘      │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**Web Search Integration:**
- Uses Tavily API for structured web search results
- Lambda function handles query formation and result parsing
- Results are cached to reduce API costs and latency
- Supports both general web search and news-specific queries

---

### 4. Reporter Agent

The **Reporter Agent** synthesizes outputs from all other agents into coherent, actionable reports.

**Capabilities:**
- Aggregates findings from Product Insight and Market Analyst agents
- Structures information into executive-ready formats
- Generates recommendations and action items
- Stores final reports for retrieval and audit

**Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│                    Reporter Agent                        │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │           Data Store Action Group               │    │
│  │  ┌─────────────┐      ┌──────────────────┐      │    │
│  │  │   Lambda    │─────►│  report-bucket   │      │    │
│  │  │  Function   │      │      (S3)        │      │    │
│  │  │             │      │                  │      │    │
│  │  │ - Template  │      │ - Final reports  │      │    │
│  │  │   rendering │      │ - Versioned      │      │    │
│  │  │ - Markdown  │      │ - Timestamped    │      │    │
│  │  │   generation│      │                  │      │    │
│  │  └─────────────┘      └──────────────────┘      │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**Report Generation Pipeline:**
1. Receives structured outputs from other agents
2. Applies report templates based on query type
3. Generates executive summary and detailed sections
4. Stores versioned report in S3 with metadata
5. Returns formatted response to Supervisor

---

## Data Flow

### Request Flow

```
1. User submits prompt
          │
          ▼
2. Supervisor Agent receives and analyzes prompt
          │
          ├──────────────────────────────────────┐
          │                                      │
          ▼                                      ▼
3a. Product Insight Agent              3b. Market Analyst Agent
    (if product analysis needed)           (if market research needed)
          │                                      │
          ▼                                      ▼
4a. RAG retrieval from                 4b. Web search via Tavily
    OpenSearch + S3                            │
          │                                      │
          ▼                                      ▼
5a. Generate product insights          5b. Generate market analysis
          │                                      │
          └──────────────────┬───────────────────┘
                             │
                             ▼
                   6. Reporter Agent
                      aggregates outputs
                             │
                             ▼
                   7. Store report in S3
                             │
                             ▼
                   8. Return to Supervisor
                             │
                             ▼
                   9. Final response to user
```

### Parallel Execution

The system supports parallel agent execution when tasks are independent. Same-priority tasks (e.g., Product Insight and Market Analyst) run concurrently, while dependent tasks (e.g., Reporter) wait for upstream agents to complete.

---

## AWS Services Used

| Service | Purpose |
|---------|---------|
| **Amazon Bedrock** | Foundation model hosting and agent orchestration |
| **AWS Lambda** | Action group execution (web search, data store operations) |
| **Amazon S3** | Document storage (product-bucket, report-bucket) |
| **Amazon OpenSearch** | Vector store for RAG embeddings and semantic search |
| **Tavily API** | External web search integration |
| **Amazon CloudWatch** | Logging, monitoring, and observability |
| **AWS IAM** | Access control and permissions |

---

## Action Groups

Action Groups are the mechanism by which agents interact with external tools and services.

### Web Search Action Group

- **Purpose:** Enables real-time web search capabilities via Tavily API
- **Lambda Function:** Handles query formation, API calls, and result parsing
- **Parameters:** Query string, search type (web/news), max results
- **Returns:** Array of search results with source URLs

### Data Store Action Group

- **Purpose:** Stores and retrieves structured data from S3
- **Lambda Function:** Handles file operations with versioning
- **Operations:** Store (content, filename, type) and Retrieve (filename)
- **Returns:** S3 URI, version ID, content, and metadata

---

## Knowledge Base Configuration

### Product Knowledge Base

- **Data Source:** S3 bucket (product-bucket) containing product docs, specifications, and release notes
- **Chunking Strategy:** Fixed-size chunking with 512 max tokens and 20% overlap
- **Embedding Model:** Amazon Titan Embeddings
- **Vector Store:** OpenSearch Serverless for semantic search

---

## Deployment

### Prerequisites

- AWS Account with Bedrock access enabled
- AWS CLI configured with appropriate credentials
- Terraform or AWS CDK for infrastructure deployment
- Python 3.11+ for Lambda functions

### Infrastructure as Code

Infrastructure is deployed using Terraform with modular configuration for:
- Bedrock Agent setup with foundation model and instructions
- Action Groups for web search and data store operations
- Knowledge Base with S3 data source and OpenSearch vector store
- Lambda functions for action group execution
- IAM roles and policies for least-privilege access

### Deployment Steps

1. Clone repository
2. Install dependencies
3. Configure AWS credentials
4. Deploy infrastructure with Terraform
5. Deploy Lambda functions
6. Sync knowledge base documents to S3
7. Start knowledge base ingestion job

---

## Monitoring & Observability

### CloudWatch Metrics

| Metric | Description |
|--------|-------------|
| `AgentInvocations` | Total agent invocations per agent type |
| `AgentLatency` | End-to-end latency per agent |
| `KnowledgeBaseHits` | RAG retrieval success rate |
| `WebSearchLatency` | Tavily API response time |
| `ErrorRate` | Agent execution failures |

### Logging Structure

All agent invocations are logged with structured JSON including:
- Timestamp and request ID
- Agent name and action
- Input (user prompt, session ID)
- Output (routed agents, execution plan)
- Latency and status

---

## Error Handling

### Retry Strategy

Agent invocations use exponential backoff retry with:
- Maximum 3 attempts
- Wait time starting at 2 seconds, max 10 seconds
- Retry on throttling exceptions

### Fallback Behavior

| Failure Scenario | Fallback Action |
|------------------|-----------------|
| Web search timeout | Return cached results or skip market analysis |
| Knowledge base unavailable | Use agent's base knowledge with disclaimer |
| Reporter agent failure | Return raw agent outputs without synthesis |
| Complete system failure | Return graceful error message with request ID |

---

## Security Considerations

- **IAM Least Privilege**: Each Lambda function has minimal required permissions
- **S3 Encryption**: All buckets use SSE-S3 encryption at rest
- **VPC Isolation**: Lambda functions run in private subnets
- **API Key Management**: Tavily API key stored in AWS Secrets Manager
- **Audit Logging**: All agent invocations logged to CloudWatch

---

## Performance Optimization

### Caching Strategy

- **Web Search Results**: Cached in ElastiCache for 1 hour
- **Knowledge Base Embeddings**: Pre-computed and indexed in OpenSearch
- **Report Templates**: Loaded at Lambda cold start

### Latency Targets

| Operation | Target | P99 |
|-----------|--------|-----|
| Supervisor routing | < 200ms | 500ms |
| RAG retrieval | < 500ms | 1s |
| Web search | < 2s | 5s |
| Full pipeline | < 10s | 30s |

---

## Future Enhancements

- [ ] Add streaming response support for real-time output
- [ ] Implement agent memory for multi-session context
- [ ] Add cost tracking and budget alerts
- [ ] Support additional knowledge bases (financial data, CRM)
- [ ] Implement human-in-the-loop for high-stakes decisions

---

## License

MIT License - see [LICENSE](LICENSE) for details.

---

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/new-agent`)
3. Commit changes (`git commit -am 'Add new agent capability'`)
4. Push to branch (`git push origin feature/new-agent`)
5. Open a Pull Request
