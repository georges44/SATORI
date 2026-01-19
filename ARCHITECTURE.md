# SATORI Architecture

Complete system design for Economic Operating System supporting autonomous agents.

---

## System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      SATORI Full Stack                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CLIENT TIER (Edge - Jetson AGX Orin)                           │
│  ├── Model Inference                                            │
│  │   ├── Mistral 7B (text/NLP)                                 │
│  │   ├── Whisper-small (audio)                                 │
│  │   └── YOLO-nano (lightweight vision)                        │
│  │                                                              │
│  ├── Learning (LoRA Fine-tuning)                               │
│  │   ├── Load base model                                       │
│  │   ├── Fine-tune on local data (45 min cycle)               │
│  │   ├── Extract LoRA weights (10-50 MB)                      │
│  │   └── Send to Event Bus                                     │
│  │                                                              │
│  └── Event Producer                                            │
│      ├── Task requests                                         │
│      ├── Task completions                                      │
│      └── Learning updates                                      │
│                                                                 │
│                      ↓ Event Bus ↓                              │
│              (Redis Pub/Sub + Secure Channel)                   │
│                                                                 │
│  ECONOMIC CORE (Orchestration Layer)                            │
│  ├── Credit System                                             │
│  │   ├── Fixed supply (e.g., 100,000 credits/month)           │
│  │   ├── Dynamic pricing (emergent from demand)                │
│  │   └── Transaction ledger (all operations recorded)          │
│  │                                                              │
│  ├── Entity Management                                         │
│  │   ├── Agents (autonomous decision makers)                   │
│  │   ├── Services (operations available)                       │
│  │   ├── Resources (constraints/capabilities)                  │
│  │   └── State tracking                                        │
│  │                                                              │
│  ├── Marketplace & Routing                                     │
│  │   ├── LLM Router (picks best model for task)               │
│  │   ├── Cost optimization (local vs cloud)                    │
│  │   ├── Task scheduling                                       │
│  │   └── Load balancing                                        │
│  │                                                              │
│  └── Federated Learning Aggregation                            │
│      ├── Wait for updates from N devices                       │
│      ├── Quality scoring (validation loss)                     │
│      ├── Weighted average aggregation                          │
│      └── Broadcast new base model                              │
│                                                                 │
│                      ↓ Event Bus ↓                              │
│                                                                 │
│  CLOUD TIER (GPU Server)                                        │
│  ├── High-Compute Operations                                   │
│  │   ├── Stable Diffusion (image generation)                  │
│  │   ├── Video analysis (frame extraction + processing)       │
│  │   ├── GPT-4V (complex vision understanding)                │
│  │   └── Model aggregation (FL central node)                   │
│  │                                                              │
│  ├── Storage                                                   │
│  │   ├── Model weights (base + aggregate)                      │
│  │   ├── Validation datasets                                   │
│  │   └── Historical aggregates                                 │
│  │                                                              │
│  └── Monitoring                                                │
│      ├── Aggregation quality checks                            │
│      ├── Model performance tracking                            │
│      └── System health metrics                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Phase 2 Implementation Details

### 2.1 Economic Foundation ✅

**Purpose:** Core economic engine (credits, pricing, ledger)

**Components:**
```python
class CreditLedger:
    """Immutable transaction record"""
    - fixed_supply: int (e.g., 10,000 credits/cycle)
    - transactions: List[Transaction]
    - balance_by_entity: Dict[Entity, int]
    
    Methods:
    - record_transaction(from, to, amount, operation)
    - get_balance(entity)
    - compute_prices_from_demand()
    
class PricingEngine:
    """Dynamic pricing (emergent from supply/demand)"""
    - base_price: float (per operation type)
    - demand_multiplier: float (0.5 to 2.0)
    - calculate_price(operation_type, current_load)
```

**Data Flow:**
```
Agent wants task
  ↓
Router proposes price
  ↓
Agent checks budget
  ↓
If approved: Execute
  ↓
Service delivers result
  ↓
Ledger records: -cost_credit, +service_output
  ↓
Next pricing cycle: Adjust prices based on demand
```

### 2.2 Entity System ✅

**Purpose:** Unified representation of agents, services, resources

**Components:**
```python
class Entity:
    """Base entity (agent, service, resource)"""
    - id: str
    - type: str (agent|service|resource)
    - capabilities: List[str]
    - constraints: Dict[str, Any]
    - state: Dict[str, Any]

class Agent(Entity):
    """Autonomous decision maker"""
    - budget: int (credits available)
    - goals: List[str]
    - decision_logic: Callable
    - learning_data: LocalDataset

class Service(Entity):
    """Operation available for agents"""
    - cost_per_call: int (credits)
    - execution_time: float (seconds)
    - required_infrastructure: str (jetson|gpu)
    - success_rate: float

class Resource(Entity):
    """Hardware/computational resource"""
    - capacity: int (max parallel operations)
    - utilization: float (0-1)
    - location: str (edge|cloud)
```

**Registry:**
```
All entities tracked in central registry
├── 8+ entities loaded at startup
├── Capabilities indexed for fast lookup
└── State synchronized across core
```

### 2.3 LLM Router ✅

**Purpose:** Route tasks to optimal LLM based on cost/quality

**Components:**
```python
class LLMRouter:
    """Multi-LLM routing with cost optimization"""
    - models: {
        "local": Mistral7B (edge, 1 credit/call),
        "anthropic": Claude (cloud, 50 credits/call),
        "openai": GPT-4 (cloud, 100 credits/call)
      }
    
    Methods:
    - evaluate_task(task_description) → (model, cost, latency)
    - execute(task, model) → result
    - route_optimally(task, budget_constraint) → best_model
```

**Routing Logic:**
```
Task arrives: "Summarize 100 emails"
  ↓
Router evaluates:
├── Local (Mistral): 100 credits, 2min latency
├── Claude: 500 credits, 10s latency
└── GPT-4: 1000 credits, 5s latency
  ↓
Agent budget: 600 credits
  ↓
Best choice: Claude (within budget, good quality)
  ↓
Execute on Claude
```

### 2.4 Agent Base ⏳

**Purpose:** Lifecycle + autonomy levels

**Components:**
```python
class AgentBase:
    """Base for all autonomous agents"""
    - lifecycle_state: enum (idle|working|learning|aggregating)
    - autonomy_level: int (0=manual, 1=scripted, 2=adaptive, 3=fully_autonomous)
    - reward_function: Callable
    - planning_horizon: int (steps ahead)
    
    Methods:
    - decide(state, options) → action
    - execute(action) → result
    - learn(feedback, reward)
    - aggregate_weights() → new_model
```

**Autonomy Levels:**
```
Level 0: Human-directed
├── Agent follows explicit instructions
└── No learning/adaptation

Level 1: Scripted
├── Agent follows rules (if-then-else)
└── Minimal adaptation

Level 2: Adaptive
├── Agent learns from feedback
├── Optimizes for budget efficiency
└── Adapts to price changes

Level 3: Fully Autonomous
├── Agent self-improves (FL)
├── Makes own routing decisions
├── Negotiates with other agents
└── SATORI's target
```

### 2.5-2.7 Agents (Text, Image, Video) ⏳

**Purpose:** Concrete implementations for different modalities

**Text Agent:**
```python
class TextAgent(AgentBase):
    """Handles NLP operations"""
    - models: [Mistral7B (edge), GPT-4 (cloud)]
    - operations: [summarize, classify, generate, translate]
    - cost_per_call: 1-100 credits (dynamic)
```

**Image Agent:**
```python
class ImageAgent(AgentBase):
    """Handles image operations"""
    - models: [YOLO-nano (edge), StableDiffusion (cloud)]
    - operations: [detect, classify, generate, enhance]
    - cost_per_call: 10-100 credits
```

**Video Agent:**
```python
class VideoAgent(AgentBase):
    """Handles video operations"""
    - models: [YOLO (edge), GPT-4V (cloud)]
    - operations: [frame_extract, analyze, summarize, tag]
    - cost_per_call: 50-200 credits
```

### 2.8 Event Bus ⏳

**Purpose:** Async communication between all components

**Architecture:**
```
Redis Pub/Sub:
├── Channel: agent_requests
│   └── {agent_id, task, budget}
├── Channel: service_responses
│   └── {service_id, result, cost_actual}
├── Channel: learning_updates
│   └── {agent_id, weights_delta, quality_score}
└── Channel: price_updates
    └── {operation_type, new_price, rationale}
```

**Message Types:**
```python
TaskRequest = {
    "agent_id": str,
    "operation": str,
    "parameters": dict,
    "budget": int,
    "deadline": float
}

TaskResult = {
    "task_id": str,
    "result": Any,
    "cost_actual": int,
    "execution_time": float,
    "quality_score": float
}

LearningUpdate = {
    "agent_id": str,
    "model_type": str,
    "weights_delta": bytes,  # LoRA update
    "quality_on_validation": float,
    "timestamp": float
}
```

**Guarantees:**
```
✅ At-least-once delivery (no loss)
✅ Ordering maintained per channel
✅ Fault tolerance (Redis persistence)
⚠️ Eventually consistent (async)
```

---

## Data Flow Example

### Simple Transaction

```
1. Agent decides to call text_summarize service
   Agent: "I have 100 credits, summarize this email"
   
2. LLM Router evaluates options
   ├── Local Mistral: 5 credits, 2s
   ├── Claude: 30 credits, 0.5s
   └── GPT-4: 100 credits, 0.3s
   
3. Route optimally (budget 100)
   Decision: Claude (quality/cost balance)
   
4. Execute task
   Claude summarizes email
   Returns: "Summary: [...]" (actual_cost: 28 credits)
   
5. Record transaction
   Ledger: Agent -28 cr, Summary_Service +28 cr
   
6. Next cycle: prices adjust based on demand
   Next hour: More demand for summarization
   → Price rises: 30 → 35 credits
```

### Federated Learning Cycle

```
Hour 1: Local Learning (Jetson)
├── Agent loads Mistral 7B base model
├── Fine-tunes on local customer emails (45 min)
├── Extracts LoRA weights (20 MB)
└── Publishes to Event Bus: learning_updates

Hour 1+: Aggregation (Cloud)
├── Waits for updates from N Jetson devices
├── Once all received:
│   ├── Compute quality scores (validation loss)
│   ├── Weighted average: new_weights = sum(weight_i × delta_i)
│   ├── Validate on held-out dataset
│   └── If quality OK: publish new base model
│
└── Jetson devices subscribe to new model

Hour 2: Next Cycle
└── All Jetson devices load improved base model
    (learned from all other organizations)
```

---

## Infrastructure Requirements

### Edge (Jetson)

```
Hardware:
├── NVIDIA Jetson AGX Orin (12-core GPU, 64GB LPDDR5X)
├── 1TB NVMe SSD
└── Network: 100 Mbps+

Models loaded:
├── Mistral 7B (quantized) — 4GB
├── Whisper-small — 500MB
└── YOLO-nano — 200MB

Max parallel operations: 5-10
```

### Cloud (GPU Server)

```
Hardware:
├── 2x NVIDIA A100 (80GB HBM2)
├── 128 GB RAM
└── Network: 10Gbps+

Models available:
├── Stable Diffusion 1.5 (8GB)
├── GPT-4V (API)
├── YOLO-large (2GB)
└── Custom fine-tuned models

Max parallel operations: 20-50
```

### Central (Economic Core)

```
Database:
├── Redis (ledger, cache, pub/sub)
├── PostgreSQL (historical data, analytics)
└── MinIO (model weights storage)

Capacity:
├── 100K transactions/day
├── 1000 agents simultaneous
└── 50 model aggregations/day
```

---

## API Interfaces

### Agent Interface

```python
class Agent:
    def request_task(self, operation: str, params: dict) -> Task:
        """Request a task, get cost estimate"""
        pass
    
    def execute_task(self, task: Task) -> Result:
        """Execute if budget allows"""
        pass
    
    def upload_learning(self, weights: bytes, quality: float) -> None:
        """Upload LoRA weights to Event Bus"""
        pass
```

### Service Interface

```python
class Service:
    def estimate_cost(self, operation: str) -> int:
        """Return credit cost"""
        pass
    
    def execute(self, params: dict) -> Result:
        """Run operation, return result"""
        pass
    
    def get_metadata(self) -> ServiceMetadata:
        """Return capabilities, latency, etc."""
        pass
```

### Router Interface

```python
class Router:
    def find_best_service(
        self, 
        operation: str, 
        budget: int,
        latency_constraint: float
    ) -> Service:
        """Find optimal service for task"""
        pass
    
    def route_with_fallback(self, operation: str) -> List[Service]:
        """Return ranked list of services"""
        pass
```

---

## Security & Privacy

### Data Isolation

```
Jetson (Edge):
✅ All customer data stays local
✅ No external API calls with data
✅ LoRA weights (model changes, not data) sent to cloud
✅ Encrypted communication with cloud

Cloud (GPU):
✅ Receives only LoRA weights (no raw data)
✅ Aggregates weights from multiple organizations
✅ No ability to reconstruct original data
✅ Regular security audits
```

### Authentication

```
Jetson ← → Cloud:
├── TLS 1.3 encryption
├── API key authentication (rotating)
└── Rate limiting per agent
```

---

## Monitoring & Observability

```python
class Metrics:
    - transactions_per_second
    - average_latency_by_operation
    - credit_price_by_operation (hourly)
    - agent_budget_utilization
    - learning_quality_scores
    - aggregation_success_rate
    - error_rates_by_service
```

**Dashboards:**
- Real-time transaction volume
- Price evolution over time
- Agent performance (accuracy, cost efficiency)
- Learning progress (model quality improvement)
- Resource utilization (Jetson, GPU, Cloud)

---

## Extensibility

**Adding new modalities:**
```
1. Create new Agent subclass
2. Register with Entity system
3. Add to LLM Router
4. Define Event Bus channels
5. Test with synthetic data
```

**Adding new infrastructure:**
```
1. Deploy hardware (Jetson or GPU)
2. Configure endpoint in Cloud
3. Register in Router
4. Monitor metrics
5. Enable auto-scaling
```

**Adding new business logic:**
```
1. Extend PricingEngine (custom formulas)
2. Extend AgentBase (new decision logic)
3. Extend LLMRouter (new models)
4. Extend EventBus (new message types)
```

---

## Next Steps

1. **Complete Phase 2** — Event Bus fully operational
2. **Deploy Y1 pilots** — 5-10 sector demonstrations
3. **Publish AAMAS 2026** — Federated Learning economics
4. **Scale to Y2** — 50+ enterprise customers
5. **Expand to EU** — Pan-European sovereign IA

