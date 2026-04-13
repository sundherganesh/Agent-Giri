# Agent-Giri
Agent Giri Brother of Siri
# Architecture Flow

## Simple request
1. User prompt enters through API Gateway
2. Orchestrator classifies request
3. Step Functions starts a simple workflow
4. Retrieval and model gateway run
5. Validation/evaluation checks output
6. Guardrails checks policy/safety
7. Response is persisted and returned
## Complex multi-task request
1. User prompt enters through API Gateway
2. Orchestrator routes to a swarm-enabled workflow
3. Step Functions invokes planner
4. Planner decomposes task
4. Swarm coordinator fans out subtasks
5. Specialist agents run in parallel
6. Aggregator combines results
7. Validation/evaluation checks completeness, grounding, consistency
8. Guardrails checks compliance/policy
9. Human approval is triggered if needed
10. Final answer is stored and returned

# Recommended specialist agent types

You do not need too many at first. Start with 4 roles:

1. Planner Agent
2. Retrieval Specialist
3. Tool Specialist
4. Analysis Specialist
5. Aggregator Agent
6. Validation/Evaluation Service

That is enough to support swarm without turning the design into chaos.

## simple flow

```mermaid
flowchart TD
    A[User Request] --> B[Orchestrator]
    B --> C[Classify Intent / Risk / Mode]
    C --> D[Workflow Registry Lookup]
    D --> E[Resolve Active Workflow ARN]
    E --> F[Start Step Functions Execution]
    F --> G[Execution Updates / Final Result]
    G --> B
```

# Enterprise Architecture with trust boundaries

```mermaid
flowchart LR
    %% ========== External ==========
    subgraph Z0[Trust Boundary 0 - User / External Zone]
        U[User]
        UI[Chat UI / Web App]
    end

    %% ========== Edge ==========
    subgraph Z1[Trust Boundary 1 - Edge / DMZ]
        CF[CloudFront]
        WAF[AWS WAF]
        APIGW[API Gateway<br/>REST / WebSocket / SSE]
    end

    %% ========== App Control ==========
    subgraph Z2[Trust Boundary 2 - Application Control Plane]
        ORCH[Chat Orchestrator API<br/>Session • Auth • Routing • Policy]
        REG[Workflow / Agent Registry]
        REDIS[Redis Cache<br/>Ephemeral Session • Working Memory • Coordination]
        SF[Step Functions<br/>Durable Workflow Orchestration]
        SQS[SQS<br/>Async Queue / Buffer]
    end

    %% ========== Intelligence ==========
    subgraph Z3[Trust Boundary 3 - Intelligence / Agent Execution Zone]
        PLAN[Planner Agent]
        SWARM[Swarm Coordinator]
        A1[Specialist Agent A<br/>Retrieval / Research]
        A2[Specialist Agent B<br/>Tool / API Execution]
        A3[Specialist Agent C<br/>Analysis / Reasoning]
        AGG[Aggregator Agent]
        RET[Retrieval Service]
        MODEL[Model Gateway]
        TOOL[Tool Gateway / Adapters]
    end

    %% ========== Trust / Quality ==========
    subgraph Z4[Trust Boundary 4 - Trust, Risk, and Quality Controls]
        VAL[Validation & Evaluation Service<br/>Grounding • Completeness • Scoring]
        GUARD[Guardrails / Policy Service<br/>Prompt • Tool • Response Checks]
        HIL[Human Approval / Oversight]
    end

    %% ========== Data / Operations ==========
    subgraph Z5[Trust Boundary 5 - Data and Platform Operations Zone]
        AUR[Aurora PostgreSQL + pgvector<br/>Conversations • Audit • Embeddings]
        S3[S3 Artifact Store<br/>Prompts • Responses • Evaluation Artifacts]
        OBS[Observability<br/>OpenTelemetry / ADOT<br/>Logs • Metrics • Traces]
        SEC[Secrets / KMS / IAM]
    end

    %% ========== Managed / External Integrations ==========
    subgraph Z6[Trust Boundary 6 - Managed AI and External Systems]
        BED[Amazon Bedrock]
        EXT[Enterprise Systems / External APIs]
    end

    %% User flow
    U --> UI
    UI --> CF --> WAF --> APIGW

    %% Sync control path
    APIGW --> ORCH
    ORCH --> REG
    ORCH --> REDIS
    ORCH --> AUR

    %% Workflow path
    ORCH --> SF
    ORCH -. async .-> SQS

    %% Step Functions orchestration
    SF --> PLAN
    PLAN --> SWARM

    %% Swarm fan-out
    SWARM -. async .-> A1
    SWARM -. async .-> A2
    SWARM -. async .-> A3

    %% Specialist actions
    A1 --> RET
    A1 --> MODEL

    A2 --> TOOL
    A2 --> MODEL

    A3 --> RET
    A3 --> TOOL
    A3 --> MODEL

    %% Aggregation
    A1 --> AGG
    A2 --> AGG
    A3 --> AGG

    %% Trust/quality pipeline
    AGG --> VAL
    VAL --> GUARD
    SF --> HIL

    %% Managed services
    MODEL --> BED
    TOOL --> EXT

    %% Data persistence
    ORCH --> S3
    AGG --> AUR
    VAL --> AUR
    GUARD --> AUR
    RET --> AUR
    TOOL --> AUR

    VAL --> S3
    AGG --> S3

    %% Security / secrets
    ORCH --> SEC
    PLAN --> SEC
    A1 --> SEC
    A2 --> SEC
    A3 --> SEC
    TOOL --> SEC
    MODEL --> SEC

    %% Observability
    ORCH --> OBS
    SF --> OBS
    PLAN --> OBS
    SWARM --> OBS
    A1 --> OBS
    A2 --> OBS
    A3 --> OBS
    AGG --> OBS
    RET --> OBS
    MODEL --> OBS
    TOOL --> OBS
    VAL --> OBS
    GUARD --> OBS

    %% Response path
    GUARD --> ORCH
    ORCH --> APIGW --> UI
```

Enterprise interaction patterns
1) Synchronous path

Used for:

simple chat request
direct RAG
low-latency response

Path:

UI -> API Gateway -> Orchestrator
Orchestrator may invoke a small workflow or route directly
final answer still goes through validation/guardrails
response streamed back
2) Asynchronous path

Used for:

long-running agent workflows
multi-agent swarm tasks
tool-heavy flows
approval-required tasks

Path:

Orchestrator -> Step Functions
Step Functions -> swarm / specialists
aggregation -> validation -> guardrails
final persisted result -> streamed/polled to UI

3) Human-in-the-loop path

Used for:

high-risk recommendations
actions against downstream systems
policy-sensitive output

Path:

Step Functions pauses
approval task is triggered
upon approval/rejection workflow resumes

Recommended trust-boundary rules

Here are the rules I would apply to each boundary.

Between Z1 and Z2

Only API Gateway should call the Orchestrator API.
No direct external traffic to agent services.

Between Z2 and Z3

Only approved workflow-triggered or orchestrator-approved calls should enter the agent zone.
Do not let agents be called directly from the UI.

Between Z3 and Z4

All aggregated outputs must pass through validation/evaluation and guardrails before returning to the user or downstream systems.

Between Z3 and Z6

Agents should never call external APIs directly.
All external calls should go through the Tool Gateway or Model Gateway.

Between all zones and Z5

Aurora is the durable source of truth.
Redis is ephemeral only.
Secrets should be fetched through approved IAM paths only.


# Exec version

```mermaid
flowchart LR
    A[Users / Chat UI] --> B[Edge Security<br/>CloudFront • WAF • API Gateway]
    B --> C[Control Plane<br/>Orchestrator • Registry • Step Functions • Redis • SQS]
    C --> D[Intelligence Plane<br/>Planner • Swarm • Specialist Agents • Retrieval • Model Gateway • Tool Gateway]
    D --> E[Trust & Quality<br/>Validation • Guardrails • Human Approval]
    E --> F[Data & Operations<br/>Aurora pgvector • S3 • Observability • Secrets/KMS]
    D --> G[Managed AI / Enterprise Systems<br/>Bedrock • External APIs]
```


# 1. External / User Zone
+--------------------------------------------------+
|            External / User Zone                  |
|--------------------------------------------------|
|  [ User ]  →  [ Chat UI / Web App ]              |
+--------------------------------------------------+

# 2. Edge /DMZ Zone
+--------------------------------------------------+
|            Edge / DMZ Zone                       |
|--------------------------------------------------|
|  [ CloudFront ] → [ AWS WAF ] → [ API Gateway ]  |
|                     (REST / WebSocket / SSE)     |
+--------------------------------------------------+

# 3. Application Control Plane
+-----------------------------------------------------------------------------------+
|                  Application Control Plane                                         |
|-----------------------------------------------------------------------------------|
|                                                                                   |
|  [ Chat Orchestrator API ]                                                        |
|   - Session Management                                                            |
|   - Auth / Identity Context                                                       |
|   - Prompt Routing                                                                |
|   - Policy Pre-check                                                              |
|                                                                                   |
|  [ Workflow / Agent Registry ]                                                    |
|   - Workflow mapping                                                              |
|   - Versioning                                                                    |
|                                                                                   |
|  [ Step Functions ]                                                               |
|   - Durable orchestration                                                         |
|   - Retry / branching / audit                                                     |
|                                                                                   |
|  [ SQS ]                                                                          |
|   - Async buffer                                                                  |
|                                                                                   |
|  [ Redis Cache ]                                                                  |
|   - Ephemeral session                                                             |
|   - Working memory                                                                |
|   - Coordination                                                                  |
|                                                                                   |
+-----------------------------------------------------------------------------------+

# 4. Intelligence / Agent Execution Zone
+--------------------------------------------------------------------------------------------------+
|                 Intelligence / Agent Execution Zone                                               |
|--------------------------------------------------------------------------------------------------|
|                                                                                                  |
|  ┌────────────────────────────┐        ┌────────────────────────────┐                            |
|  |       Planner Agent        | -----> |     Swarm Coordinator       |                            |
|  └────────────────────────────┘        └───────────────┬────────────┘                            |
|                                                      /     |       \                              |
|                                                     /      |        \                             |
|                                                    v       v         v                            |
|                                          ┌────────────┐ ┌────────────┐ ┌────────────┐            |
|                                          | Specialist | | Specialist | | Specialist |            |
|                                          | Agent A    | | Agent B    | | Agent C    |            |
|                                          | (Retrieval)| | (Tools)    | | (Analysis) |            |
|                                          └─────┬──────┘ └─────┬──────┘ └─────┬──────┘            |
|                                                |              |              |                   |
|                                                v              v              v                   |
|                                       ┌──────────────┐  ┌──────────────┐                         |
|                                       | Retrieval     |  | Tool Gateway |                         |
|                                       | Service       |  | / Adapters   |                         |
|                                       └──────┬───────┘  └──────┬───────┘                         |
|                                              |                 |                                 |
|                                              v                 v                                 |
|                                         ┌────────────────────────────┐                           |
|                                         |     Model Gateway           |                           |
|                                         |  (LLM Routing / Control)    |                           |
|                                         └────────────────────────────┘                           |
|                                                                                                  |
|                                 ┌────────────────────────────┐                                   |
|                                 |     Aggregator Agent        |                                   |
|                                 |  (Merge + Resolve Output)   |                                   |
|                                 └────────────────────────────┘                                   |
|                                                                                                  |
+--------------------------------------------------------------------------------------------------+

# 5. Trust, Risk and Quality Layer

+----------------------------------------------------------------------------------+
|              Trust, Risk, and Quality Layer                                      |
|----------------------------------------------------------------------------------|
|                                                                                  |
|  [ Validation & Evaluation Service ]                                             |
|   - Grounding check                                                              |
|   - Completeness                                                                 |
|   - Scoring / confidence                                                         |
|                                                                                  |
|  [ Guardrails / Policy Service ]                                                 |
|   - Prompt filtering                                                             |
|   - Tool restrictions                                                           |
|   - Output safety                                                                |
|                                                                                  |
|  [ Human Approval ]                                                              |
|   - High-risk workflows                                                          |
|   - Compliance review                                                            |
|                                                                                  |
+----------------------------------------------------------------------------------+

# 6. Data and Operations Zone

+--------------------------------------------------------------------------------------+
|                    Data and Platform Operations Zone                                 |
|--------------------------------------------------------------------------------------|
|                                                                                      |
|  [ Aurora PostgreSQL + pgvector ]                                                    |
|   - Conversations                                                                   |
|   - Audit trail                                                                     |
|   - Embeddings                                                                      |
|                                                                                      |
|  [ S3 Artifact Store ]                                                               |
|   - Prompt/response archives                                                        |
|   - Evaluation outputs                                                              |
|                                                                                      |
|  [ Observability (ADOT) ]                                                            |
|   - Logs                                                                            |
|   - Metrics                                                                         |
|   - Traces                                                                          |
|                                                                                      |
|  [ Secrets Manager / KMS / IAM ]                                                     |
|   - Credentials                                                                     |
|   - Encryption                                                                      |
|                                                                                      |
+--------------------------------------------------------------------------------------+

# 7. External / Managed Services
+--------------------------------------------------------------+
|        Managed AI and External Systems                       |
|--------------------------------------------------------------|
|                                                              |
|  [ Amazon Bedrock ]                                          |
|   - LLM inference                                            |
|                                                              |
|  [ External APIs / Enterprise Systems ]                       |
|   - Market data                                              |
|   - Internal services                                        |
|                                                              |
+--------------------------------------------------------------+
