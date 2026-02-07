# Architecture & Procedure: Gmail + LLM + Quantum Scheduler (QUBO)

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![Docker](https://img.shields.io/badge/docker-ready-brightgreen.svg)](https://www.docker.com/)

This document describes the end-to-end technical design and operational procedure to build a **quantum-inspired scheduling system** where client requests arrive via Gmail, are processed using a Large Language Model (LLM), transformed into structured JSON, and optimized using QUBO-based quantum or hybrid solvers.

---

## 📋 Table of Contents

- [System Objective](#-system-objective)
- [End-to-End Architecture](#-end-to-end-architecture)
- [System Components](#-system-components)
- [Data Flow](#-data-flow)
- [Gmail Configuration](#-gmail-configuration)
- [LLM Processing](#-llm-processing)
- [JSON Schema](#-json-schema)
- [QUBO Model](#-qubo-model)
- [Development Environment](#-development-environment)
- [Future Enhancements](#-future-enhancements)

---

## 🎯 System Objective

Automatically optimize the allocation of technical events and presentations to consultants and time slots (09:00–17:00), respecting constraints and preferences using quantum optimization techniques.

**Key Goals:**
- 📧 Seamless email-based intake system via Gmail
- 🤖 Intelligent extraction of scheduling requirements using LLM
- ⚛️ Quantum-inspired optimization for complex scheduling constraints
- 🔄 Automated workflow with minimal human intervention

---

## 🏗️ End-to-End Architecture

```mermaid
graph TB
    Client[📧 Client Email] --> Gmail[Gmail Account<br/>quantum.scheduler@gmail.com]
    Gmail --> |Label: QS/IN| Ingestor[Gmail Ingestor Service<br/>Docker Container]
    Ingestor --> LLM[LLM Service<br/>NLP Extraction]
    LLM --> |Structured JSON| API[Scheduler API<br/>FastAPI/Flask]
    API --> DB[(PostgreSQL<br/>Database)]
    API --> Queue[Job Queue<br/>Redis/Celery]
    Queue --> Quantum[Quantum Worker<br/>QUBO Solver]
    Quantum --> |Solution| DB
    Quantum --> |Optimized Schedule| Notify[Notification Service]
    
    style Client fill:#e1f5ff
    style Gmail fill:#ea4335
    style Ingestor fill:#4285f4
    style LLM fill:#34a853
    style API fill:#fbbc04
    style Quantum fill:#9c27b0
    style DB fill:#607d8b
```

### System Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant G as Gmail
    participant I as Ingestor
    participant L as LLM
    participant A as API
    participant Q as Quantum Worker
    participant D as Database
    
    C->>G: Send scheduling request email
    G->>G: Apply label QS/IN
    I->>G: Poll for QS/IN emails
    G->>I: Return unread emails
    I->>L: Forward email content
    L->>L: Extract structured data
    L->>I: Return JSON + validation
    
    alt Valid JSON
        I->>A: POST /events/intake
        A->>D: Store event request
        A->>Q: Create optimization job
        Q->>Q: Build QUBO model
        Q->>Q: Solve optimization
        Q->>D: Store solution
        I->>G: Update label to QS/PROC
    else Invalid/Incomplete
        I->>G: Update label to QS/WAIT
        I->>C: Request clarification
    end
```

---

## 🔧 System Components

### Component Architecture

```mermaid
graph LR
    subgraph "Intake Layer"
        A[Gmail Ingestor]
        B[LLM Processor]
    end
    
    subgraph "Processing Layer"
        C[Scheduler API]
        D[Validation Service]
    end
    
    subgraph "Optimization Layer"
        E[QUBO Builder]
        F[Quantum Solver]
        G[Classical Fallback]
    end
    
    subgraph "Storage Layer"
        H[(PostgreSQL)]
        I[(Redis Cache)]
    end
    
    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    F -.fallback.-> G
    F --> H
    C --> I
    
    style A fill:#4285f4
    style B fill:#34a853
    style C fill:#fbbc04
    style E fill:#9c27b0
    style F fill:#9c27b0
```

---

## 📊 Data Flow

```mermaid
stateDiagram-v2
    [*] --> EmailReceived: Client sends email
    EmailReceived --> LabeledQS_IN: Gmail applies label
    LabeledQS_IN --> IngestorPolling: Ingestor reads
    IngestorPolling --> LLMProcessing: Extract content
    
    LLMProcessing --> ValidationCheck: Parse JSON
    
    ValidationCheck --> APIIntake: Valid JSON
    ValidationCheck --> LabelQS_WAIT: Missing fields
    ValidationCheck --> LabelQS_ERR: Parse error
    
    APIIntake --> DatabaseStore: Store request
    DatabaseStore --> QueueJob: Create optimization job
    QueueJob --> BuildQUBO: Model construction
    BuildQUBO --> SolveQUBO: Quantum solver
    SolveQUBO --> StoreSolution: Persist results
    StoreSolution --> LabelQS_PROC: Success
    StoreSolution --> [*]
    
    LabelQS_WAIT --> [*]
    LabelQS_ERR --> [*]
```

---

## 📧 Gmail Configuration

### Dedicated Gmail Account

**Account:** `quantum.scheduler@gmail.com`

### Operational Labels

| Label | Purpose | Action |
|-------|---------|--------|
| `QS/IN` | New scheduling requests | Auto-applied by filters |
| `QS/PROC` | Successfully processed | Applied after optimization |
| `QS/WAIT` | Missing or ambiguous information | Requires clarification |
| `QS/ERR` | Parsing or validation errors | Human review needed |

**Important:** Emails are never deleted, only relabeled for audit trail.

### Gmail Ingestor Service

**Docker Container Specifications:**

```mermaid
graph TB
    subgraph "Gmail Ingestor Container"
        Auth[OAuth2 Handler<br/>gmail.modify scope]
        Poll[Poller Service<br/>Every 60s]
        Extract[Content Extractor]
        Label[Label Manager]
    end
    
    Poll --> |Query: label:QS/IN is:unread| Gmail[Gmail API]
    Gmail --> Extract
    Extract --> |Subject, Body, Metadata| LLM[LLM Service]
    LLM --> Label
    Label --> Gmail
    
    style Auth fill:#ea4335
    style Poll fill:#4285f4
```

**Responsibilities:**
- ✅ OAuth2 authentication (`gmail.modify` scope)
- ✅ Polling query: `label:QS/IN is:unread`
- ✅ Extract subject, sender, body
- ✅ Forward content to LLM service
- ✅ Update labels based on processing outcome

---

## 🤖 LLM Processing

### LLM Input Structure

The LLM receives:
- **Full email body** (plaintext or HTML-stripped)
- **Metadata:** From, Subject, Date
- **Expected JSON schema** for validation

### LLM Output Requirements

```mermaid
graph LR
    Input[Email Content] --> LLM[LLM Processor]
    LLM --> Schema{Valid Schema?}
    Schema -->|Yes| JSON[Strict JSON Output]
    Schema -->|No| Missing[missing_fields array]
    Schema -->|Ambiguous| Questions[clarification_questions]
    
    JSON --> Valid[✓ Ready for API]
    Missing --> Wait[→ QS/WAIT]
    Questions --> Wait
    
    style LLM fill:#34a853
    style Valid fill:#4caf50
    style Wait fill:#ff9800
```

**Must Return:**
- ✅ Strict JSON compliant with schema
- ✅ `missing_fields[]` if information is incomplete
- ✅ `clarification_questions[]` if ambiguity exists

---

## 📄 JSON Schema

### Base JSON Schema (V1)

```json
{
  "client_name": "string",
  "event_title": "string",
  "description": "string",
  "preferred_consultant": "string | null",
  "consultant_required": false,
  "duration_min": 60,
  "time_window": {
    "earliest_date": "YYYY-MM-DD",
    "latest_date": "YYYY-MM-DD",
    "daily_start": "09:00",
    "daily_end": "17:00",
    "timezone": "Europe/Madrid"
  },
  "priority": 3
}
```

### Schema Validation Flow

```mermaid
graph TD
    JSON[JSON Input] --> V1{Required Fields?}
    V1 -->|Missing| Reject[Return Error]
    V1 -->|Complete| V2{Date Format?}
    V2 -->|Invalid| Reject
    V2 -->|Valid| V3{Time Range?}
    V3 -->|Invalid| Reject
    V3 -->|Valid| V4{Priority 1-5?}
    V4 -->|Invalid| Reject
    V4 -->|Valid| Accept[✓ Validated]
    
    Accept --> Store[(Store in DB)]
    
    style Accept fill:#4caf50
    style Reject fill:#f44336
```

---

## ⚛️ QUBO Model

### Quantum Optimization Overview

```mermaid
graph TB
    subgraph "QUBO Model Construction"
        Events[Events List] --> Vars[Binary Variables<br/>x(e,t,c)]
        Consultants[Consultants] --> Vars
        TimeSlots[Time Slots<br/>09:00-17:00] --> Vars
        
        Vars --> Constraints[Constraints Matrix]
        Constraints --> C1[Each event<br/>scheduled once]
        Constraints --> C2[No consultant<br/>overlap]
        Constraints --> C3[Valid working<br/>hours only]
        
        Constraints --> Obj[Objective Function]
        Obj --> O1[Minimize idle gaps]
        Obj --> O2[Respect preferences]
        Obj --> O3[Prioritize high-impact]
    end
    
    Obj --> QUBO[QUBO Matrix Q]
    QUBO --> Solver{Solver Type?}
    Solver -->|Quantum| QPU[QPU/Annealer]
    Solver -->|Hybrid| Hybrid[Hybrid Solver]
    Solver -->|Classical| SA[Simulated Annealing]
    
    QPU --> Solution[Optimal Schedule]
    Hybrid --> Solution
    SA --> Solution
    
    style QUBO fill:#9c27b0
    style Solution fill:#4caf50
```

### Binary Variables

$$x_{e,t,c} = \begin{cases} 1 & \text{if event } e \text{ is assigned to time slot } t \text{ with consultant } c \\ 0 & \text{otherwise} \end{cases}$$

### Constraints

1. **Each event scheduled exactly once:**
   $$\sum_{t,c} x_{e,t,c} = 1 \quad \forall e$$

2. **No consultant overlap:**
   $$\sum_{e} x_{e,t,c} \leq 1 \quad \forall t,c$$

3. **Valid working hours only:**
   $$x_{e,t,c} = 0 \text{ if } t \notin [09:00, 17:00]$$

### Objective Function

$$\min \left( \alpha \cdot \text{idle\_gaps} + \beta \cdot \text{preference\_penalty} + \gamma \cdot \text{priority\_penalty} \right)$$

Where:
- **α (idle gaps):** Minimize wasted time between events
- **β (preferences):** Respect consultant preferences
- **γ (priority):** Prioritize high-importance events

---

## 💻 Development Environment

### IDE & Tools

```mermaid
graph LR
    VSCode[Visual Studio Code] --> Copilot[GitHub Copilot]
    VSCode --> Docker[Docker Extension]
    VSCode --> Python[Python Extension]
    
    Copilot --> Assist[AI-Assisted Coding]
    Docker --> Containers[Container Management]
    Python --> Debug[Debugging & Testing]
    
    style VSCode fill:#007acc
    style Copilot fill:#000000
```

**Primary IDE:** Visual Studio Code  
**AI Assistant:** GitHub Copilot

### Repository Structure

```
quantum-scheduler/
├── gmail-ingestor/          # Email polling & processing
│   ├── Dockerfile
│   ├── src/
│   └── requirements.txt
├── scheduler-api/           # REST API & validation
│   ├── Dockerfile
│   ├── app/
│   └── requirements.txt
├── quantum-worker/          # QUBO solver
│   ├── Dockerfile
│   ├── solvers/
│   └── requirements.txt
└── docker-compose.yml       # Orchestration
```

### Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Ingestor** | Python 3.11+, Gmail API | Email polling |
| **LLM** | OpenAI API / Azure OpenAI | NLP extraction |
| **API** | FastAPI / Flask | REST endpoints |
| **Database** | PostgreSQL | Data persistence |
| **Queue** | Redis + Celery | Job management |
| **Solver** | D-Wave Ocean SDK / Qiskit | QUBO optimization |
| **Container** | Docker + Docker Compose | Orchestration |

---

## 🚀 Future Enhancements

### Roadmap

```mermaid
gantt
    title Development Roadmap
    dateFormat  YYYY-MM-DD
    section Phase 1
    Gmail Integration           :done, p1, 2026-02-01, 14d
    LLM Processing             :active, p2, 2026-02-08, 14d
    Basic QUBO Solver          :p3, 2026-02-15, 21d
    section Phase 2
    Gmail Push Notifications   :p4, 2026-03-01, 14d
    Google Calendar Sync       :p5, 2026-03-08, 14d
    section Phase 3
    Rolling-Horizon Opt        :p6, 2026-04-01, 30d
    Hybrid Quantum Solvers     :p7, 2026-04-15, 30d
    section Phase 4
    Full Observability         :p8, 2026-05-01, 21d
    Production Deployment      :p9, 2026-05-15, 14d
```

### Planned Features

- ✨ **Gmail push notifications** instead of polling
- 📅 **Google Calendar integration** for automatic event creation
- 🔄 **Rolling-horizon optimization** for dynamic rescheduling
- ⚡ **Hybrid quantum-classical solvers** for better performance
- 📊 **Full observability** (logs, metrics, distributed tracing)
- 🔐 **Enhanced security** with OAuth2 refresh tokens
- 🌐 **Multi-timezone support** for global teams
- 📱 **Mobile notifications** via Telegram/Slack
- 🎨 **Web dashboard** for schedule visualization

---

## 📖 Getting Started

### Prerequisites

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Install Docker Compose
sudo apt-get install docker-compose-plugin

# Clone repository
git clone https://github.com/rjamoriz/Architecture-Procedure-Gmail-LLM-Quantum-Scheduler-QUBO-.git
cd Architecture-Procedure-Gmail-LLM-Quantum-Scheduler-QUBO-
```

### Configuration

1. Set up Gmail OAuth2 credentials
2. Configure environment variables
3. Initialize database schema
4. Deploy with Docker Compose

*Detailed setup instructions coming soon...*

---

## 📝 License

MIT License - See [LICENSE](LICENSE) file for details

---

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

---

## 📧 Contact

Project Maintainer: [@rjamoriz](https://github.com/rjamoriz)

---

<div align="center">

**Built with ❤️ using Quantum Computing, AI, and Modern Cloud Architecture**

</div>
