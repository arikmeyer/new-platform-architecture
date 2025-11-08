# Comprehensive Architecture Evaluation: SwitchUp on Windmill.dev

**Date**: 2025-11-08
**Evaluation Focus**: Windmill-native implementation feasibility for all domains
**Key Assumptions**:
1. All domains except /lifecycle (Supabase Postgres) run as Windmill scripts/flows
2. AI-first = deterministic playbooks with LLM steps, easy evolution to agents
3. State as source of truth (not events; avoid event sourcing complexity)

---

## Executive Summary

After deep investigation into Windmill.dev's capabilities, **your architecture is not only feasible but exceptionally well-suited for Windmill**. The platform provides native features that directly address your operational scalability goals:

✅ **All domains CAN be implemented in Windmill** (Provider, Offer, Optimisation, Service, Growth, Case)
✅ **State-first pattern is natively supported** via Supabase integration + Windmill Resources/States
✅ **AI evolution path is clear** with Windmill AI + AI Agent steps
✅ **Domain boundaries map to folder structure** with permission isolation
✅ **Process Dispatcher pattern leverages Windmill's dynamic dispatch**

**Key Finding**: Your architecture demonstrates sophisticated understanding of Windmill's strengths. The concerns are not about feasibility but about **operational patterns and specific implementation decisions**.

---

## Part 1: Domain Feasibility Analysis

### ✅ /lifecycle Domain: Supabase Postgres (External to Windmill)

**Architecture Decision**: Core business state lives in Supabase Postgres, accessed by Windmill scripts

**Windmill Support**:
- **Native Supabase integration**: First-class PostgreSQL resource type
- **Direct SQL execution**: Scripts can run `SELECT/INSERT/UPDATE` queries directly
- **Connection pooling**: Pinned resources (`-- database resource_path`) eliminate connection overhead
- **Database webhooks**: Supabase can trigger Windmill flows on INSERT/UPDATE/DELETE
- **Row-Level Security (RLS)**: Windmill supports Supabase auth for RLS-protected tables

**Business Intent Catalog Implementation**:
```python
# /lifecycle/contract/confirm-activation (Windmill Script)
import wmill
from typing import TypedDict

class ConfirmActivationInput(TypedDict):
    contract_id: str
    activation_date: str
    trace_id: str

def main(
    db: dict,  # Supabase resource
    input: ConfirmActivationInput
) -> dict:
    """
    Command Handler: Transition contract to ACTIVE state
    Validates state machine rules, updates Postgres, returns result
    """
    # Query current state
    result = wmill.execute_query(
        db,
        f"SELECT status FROM contracts WHERE id = '{input['contract_id']}'"
    )

    # Validate state transition (PENDING → ACTIVE)
    if result[0]['status'] != 'PENDING':
        raise ValueError(f"Invalid transition: {result[0]['status']} → ACTIVE")

    # Execute state change
    wmill.execute_query(
        db,
        f"""
        UPDATE contracts
        SET status = 'ACTIVE',
            activation_date = '{input['activation_date']}',
            updated_at = NOW()
        WHERE id = '{input['contract_id']}'
        RETURNING id, status, activation_date
        """
    )

    # Log structured event (trace_id for observability)
    print(json.dumps({
        "event": "contract.activated",
        "trace_id": input['trace_id'],
        "contract_id": input['contract_id']
    }))

    return {"status": "success", "contract_id": input['contract_id']}
```

**State-First Pattern**:
- ✅ Supabase = authoritative state
- ✅ No event sourcing required
- ✅ Windmill queries state on-demand
- ✅ Database webhooks trigger flows when state changes (optional)

**Assessment**: **PERFECT FIT**. Your /lifecycle domain as Supabase + Windmill scripts is the recommended pattern.

---

### ✅ /provider Domain: Windmill Scripts (API + RPA)

**Architecture Decision**: Provider interactions (API calls, RPA, data extraction) as Windmill scripts

**Windmill Support**:

#### 1. **API Integration**
- 100+ pre-built integrations (REST API resource types)
- Custom HTTP clients in Python/TypeScript
- OAuth 2.0 flows supported
- Retry logic with exponential backoff (built-in)

#### 2. **RPA/Browser Automation**
- **Playwright integration** (native support)
- Chromium-equipped workers (tag: `chromium`)
- Headless browser automation
- Perfect for unstable provider portals

#### 3. **Email Processing**
- **Email triggers**: Unique email addresses per flow
- Automatic parsing (`parsed_email` + `raw_email`)
- Attachment handling (S3 upload)
- Preprocessors for extraction

#### 4. **LLM Data Extraction**
- Native OpenAI integration
- GPT-4 for document parsing
- Structured output via JSON Schema

**Provider Knowledge Graph Implementation**:

Option 1: **Windmill Resources** (Recommended for Phase 1)
```json
// Resource: provider/vattenfall_config (type: provider_config)
{
  "provider_id": "vattenfall",
  "api": {
    "base_url": "https://api.vattenfall.de/v2",
    "auth_type": "oauth2",
    "oauth_resource": "provider/vattenfall_oauth",
    "rate_limit": {"requests_per_minute": 60},
    "endpoints": {
      "cancel_contract": "/contracts/{id}/cancel",
      "submit_meter_reading": "/meterpoints/{id}/readings"
    }
  },
  "rpa": {
    "portal_url": "https://mein.vattenfall.de",
    "selectors": {
      "login_email": "#email-input",
      "login_password": "#password-input"
    }
  },
  "service_regions": ["DE-BE", "DE-HH", "DE-BB"],
  "supported_product_types": ["ELECTRICITY", "GAS"]
}
```

Access in scripts:
```python
def main(provider_id: str = "vattenfall"):
    config = wmill.get_resource(f"provider/{provider_id}_config")
    # Use config for API calls or RPA
```

Option 2: **Supabase Table** (for complex queries)
```sql
CREATE TABLE provider_configs (
  provider_id TEXT PRIMARY KEY,
  config JSONB NOT NULL,
  version INTEGER NOT NULL,
  active BOOLEAN DEFAULT true,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

**Provider Capability Example**:
```python
# /provider/api/initiate-cancellation (Windmill Script)
from playwright.async_api import async_playwright

async def main(
    provider_id: str,
    contract_id: str,
    cancellation_date: str
):
    """
    Attempts API cancellation, falls back to RPA if API fails
    """
    config = wmill.get_resource(f"provider/{provider_id}_config")

    # Try API first
    if config['api'].get('endpoints', {}).get('cancel_contract'):
        try:
            result = await call_provider_api(config, contract_id, cancellation_date)
            return {"method": "api", "result": result}
        except APIError as e:
            print(f"API failed: {e}, falling back to RPA")

    # Fallback to RPA
    async with async_playwright() as p:
        browser = await p.chromium.launch()
        page = await browser.new_page()
        # Navigate to provider portal, perform cancellation
        await page.goto(config['rpa']['portal_url'])
        # ... RPA logic
        return {"method": "rpa", "result": "success"}
```

**Assessment**: **EXCELLENT FIT**. Windmill's email triggers, Playwright support, and LLM integration are purpose-built for provider automation.

---

### ✅ /offer Domain: Windmill Scripts + Supabase

**Architecture Decision**: Offer data model + processing pipeline as scripts, data storage in Supabase

**Key Challenge**: Universal Service Ontology complexity

**Windmill Support**:

#### 1. **Data Modeling**
Store offer configurations in Supabase, process via Windmill scripts:

```sql
-- Supabase schema
CREATE TABLE offer_blueprints (
  blueprint_id TEXT PRIMARY KEY,
  category TEXT, -- 'RecurringFee', 'Feature', etc.
  schema JSONB,  -- JSON Schema for instances
  version INTEGER
);

CREATE TABLE offer_configurations (
  config_id TEXT PRIMARY KEY,
  provider_id TEXT,
  product_name TEXT,
  config JSONB,  -- Full ontology structure
  status TEXT,   -- 'draft', 'active', 'archived'
  version INTEGER
);
```

#### 2. **Processing Pipeline as Scripts**

```python
# /offer/processing/ingest-and-normalize (Windmill Script)
def main(
    db: dict,  # Supabase resource
    source_id: str,
    raw_data: dict
) -> list[dict]:
    """
    Pipeline: Fetch → Slice → Map → Normalize → Validate
    Returns array of normalized offer data
    """
    # 1. Fetch source config
    source_config = wmill.get_resource(f"offer/sources/{source_id}")

    # 2. Slice into individual offers
    offers = slice_offers(raw_data, source_config['slice_rules'])

    # 3. Map source fields to internal elements
    mapped = [map_fields(offer, source_config['mappings']) for offer in offers]

    # 4. Normalize (units, dates, formats) - use LLM for ambiguous cases
    normalized = [normalize_offer(m) for m in mapped]

    # 5. Validate against blueprints
    validated = [validate_against_schema(db, n) for n in normalized]

    return validated
```

#### 3. **Shared Logic via Imports**

```python
# /offer/lib/common_logic.py (Shared module, no main function)
def normalize_price(value: str, unit: str) -> dict:
    """Shared normalization logic"""
    # Complex normalization rules
    return {"value_cents": int(float(value) * 100), "currency": "EUR"}

# Import in other scripts:
# from f.offer.lib.common_logic import normalize_price
```

#### 4. **Offer Evaluation as Flow**

```
Flow: /offer/processing/get-aggregate-offers
├─ Step 1: Query Supabase for entityConfigVersionIds
├─ Step 2: For-loop over each config
│  ├─ Resolve inheritance chain
│  ├─ Aggregate elements
│  ├─ Evaluate policies (call /optimisation scripts)
│  └─ Return aggregate offer
└─ Step 3: Return array of evaluated offers
```

**Complexity Management Recommendation**:

Given your "ULTRATHINK" directive, I must address the elephant in the room:

**The Universal Service Ontology (8 abstraction layers) is implementable in Windmill BUT**:

**Phase 1 Recommendation**: Start with **pragmatic denormalization**
- Store 2-3 providers as JSONB in Supabase with simple schemas
- Extract patterns after implementing 5+ providers
- Build ontology incrementally based on **real** variance, not imagined

**Phase 2+**: Evolve to full ontology
- Blueprints → Definitions → Instances as Supabase tables
- Processing pipeline as versioned scripts
- AI-assisted schema generation (via Windmill AI)

**Why?** Windmill is perfect for iterative development:
- Scripts are versioned (never overwritten)
- Git sync enables rollback
- Easy to refactor as patterns emerge

**Assessment**: **FEASIBLE BUT RECOMMEND PHASED APPROACH**. Start simple, leverage Windmill's versioning to evolve sophistication.

---

### ✅ /optimisation Domain: Windmill Scripts (Pure Functions)

**Architecture Decision**: Stateless calculation engine as Python/Rust scripts

**Windmill Support**:

#### 1. **Pure Function Scripts**
Perfect match - Windmill scripts are naturally stateless:

```python
# /optimisation/calculate-savings (Windmill Script)
from typing import TypedDict
import polars as pl  # Fast dataframe library

class SavingsInput(TypedDict):
    current_contract: dict
    candidate_offers: list[dict]
    user_context: dict

def main(input: SavingsInput) -> list[dict]:
    """
    Pure function: same input → same output
    No state, no side effects
    """
    # Complex savings calculations
    df = pl.DataFrame(input['candidate_offers'])
    # ... projection logic, bonus calculations, etc.

    return df.with_columns([
        pl.col("annual_cost") - input['current_contract']['annual_cost']
    ]).to_dicts()
```

#### 2. **Complex Calculations**
- Python: Polars/Pandas for data processing
- Rust: Compile to WASM for performance-critical paths (Windmill supports Rust)
- ML models: Load from S3, run inference

#### 3. **Routing Strategies**

```python
# /optimisation/routing_strategies/percentage_rollout (Windmill Script)
import random

def main(
    variants: list[dict],  # [{"path": "...", "weight": 50}, ...]
    user_context: dict
) -> str:
    """
    Strategy resolver for A/B testing
    Returns target path based on weighted random selection
    """
    user_hash = hash(user_context.get('user_id', ''))
    random.seed(user_hash)  # Deterministic per user

    total_weight = sum(v['weight'] for v in variants)
    roll = random.randint(1, total_weight)

    cumulative = 0
    for variant in variants:
        cumulative += variant['weight']
        if roll <= cumulative:
            return variant['path']
```

#### 4. **Caching for Performance**
Use Windmill States for caching expensive calculations:

```python
def main(offer_id: str):
    # Check cache
    cache_key = f"savings_calc_{offer_id}"
    cached = wmill.get_state(cache_key)
    if cached:
        return cached

    # Expensive calculation
    result = complex_calculation(offer_id)

    # Cache for 1 hour (use separate TTL management)
    wmill.set_state(cache_key, result)
    return result
```

**Assessment**: **PERFECT FIT**. Windmill's stateless script model is ideal for pure calculation functions.

---

### ✅ /service Domain: Windmill HTTP Routes + Scripts

**Architecture Decision**: User/agent interactions via Windmill HTTP routes + backend scripts

**Windmill Support**:

#### 1. **API Gateway via HTTP Routes**
Windmill can **fully replace** a traditional API gateway:

```yaml
# HTTP Route: POST /api/v1/contracts/{contract_id}/cancel
Route Path: /api/v1/contracts/:contract_id/cancel
Method: POST
Authentication: Windmill Auth (JWT)
Runnable: /service/api/cancel_contract_flow
```

Flow receives:
```json
{
  "contract_id": "C123",  // From path param
  "reason": "...",         // From request body
  "user": {...}            // From JWT token
}
```

#### 2. **Authentication Options**
- **Windmill Auth**: JWT tokens for your app users
- **External JWT**: Generate your own tokens with custom permissions
- **API Keys**: Simple token-based auth
- **HMAC Signatures**: For third-party webhooks

#### 3. **User Self-Service Flow**

```
Flow: /service/api/update-user-address
├─ Step 1: Validate input (JSON Schema)
├─ Step 2: Authorize (check user owns resource)
├─ Step 3: Call /case/address_management/update-user-address (orchestration)
├─ Step 4: Return response
```

**Strict Boundary**: /service flows ONLY call /case flows, never tool domains directly.

#### 4. **Agent Support Interface**

```python
# /service/agent/get-case-context (Windmill Script)
def main(
    db: dict,
    contract_id: str,
    user_id: str
) -> dict:
    """
    Aggregates context for support agents
    Called by support UI or AI agent
    """
    # Query Supabase for contract, user, tasks, timeline
    # Return comprehensive context
    return {
        "contract": {...},
        "user": {...},
        "recent_tasks": [...],
        "timeline": [...]
    }
```

**Assessment**: **EXCELLENT FIT**. Windmill HTTP routes + JWT auth provide full API gateway capabilities.

---

### ✅ /growth Domain: Windmill Scripts + Scheduled Flows

**Architecture Decision**: Content generation, onboarding funnels as scripts and flows

**Windmill Support**:

#### 1. **Content Generation with LLM**

```python
# /growth/content/generate-article (Windmill Script)
from openai import OpenAI

def main(
    openai_resource: dict,
    topic: str,
    keywords: list[str]
) -> dict:
    """
    Generate SEO-optimized article using GPT-4
    """
    client = OpenAI(api_key=openai_resource['api_key'])

    prompt = f"""
    Write a comprehensive guide about {topic}.
    Include keywords: {', '.join(keywords)}
    Format: Markdown with H2/H3 structure
    """

    response = client.chat.completions.create(
        model="gpt-4-turbo",
        messages=[{"role": "user", "content": prompt}]
    )

    article = response.choices[0].message.content

    # Store in CMS (via API or Supabase)
    return {"article": article, "word_count": len(article.split())}
```

#### 2. **Personalized Onboarding Flows**

```
Flow: /growth/onboarding/adaptive-funnel
├─ Step 1: Identify user segment (ML model)
├─ Step 2: Branch on segment
│  ├─ Branch A: Price-sensitive flow
│  ├─ Branch B: Convenience-focused flow
│  └─ Branch C: Eco-conscious flow
├─ Step 3: Dynamic content generation
├─ Step 4: Suspend for user interaction (approval step)
├─ Step 5: Resume based on user input
└─ Step 6: Call /case/onboarding/complete-activation
```

#### 3. **A/B Testing with Routing**
Use Process Dispatcher pattern for growth experiments:

```json
// Resource: growth/onboarding/routing.json
{
  "initiate-new-user": {
    "strategy": {
      "path": "/optimisation/routing_strategies/percentage_rollout",
      "args": {
        "variants": [
          {"path": "/growth/onboarding/funnel_v1_playbook", "weight": 50},
          {"path": "/growth/onboarding/funnel_v2_agent", "weight": 50}
        ]
      }
    }
  }
}
```

**Assessment**: **PERFECT FIT**. Windmill AI + flow branching + suspend/resume = ideal for growth experimentation.

---

### ✅ /case Domain: Windmill Flows (Orchestration)

**Architecture Decision**: All business processes as Windmill flows, three-tier structure

**Windmill Support**:

#### 1. **Three-Tier Implementation**

**Tier 3: Ingestion Flows**
```
Flow: /case/ingestion/process-provider-email
Trigger: Email (unique address)
├─ Step 1: Parse email (LLM extraction)
├─ Step 2: Classify event type (price increase, confirmation, etc.)
└─ Step 3: Call Process Dispatcher with event data
```

**Tier 2: Process Dispatcher**
```
Flow: /case/meta/dispatch-process
Input: {process_name, user_context, input_args}
├─ Step 1: Call /case/meta/resolve-process-target (Strategy Resolver)
├─ Step 2: Dynamic dispatch to target path (inner flow call)
└─ Step 3: Return result
```

Dynamic dispatch in Windmill:
```python
# /case/meta/resolve-process-target (Script)
def main(process_name: str, user_context: dict) -> str:
    """Reads routing config, executes strategy, returns target path"""
    config = wmill.get_resource(f"case/routing/{process_name}")

    # Execute strategy script
    strategy_result = wmill.run_script_by_path(
        config['strategy']['path'],
        args={"variants": config['variants'], "user_context": user_context}
    )

    return strategy_result  # e.g., "/case/lifecycle_management/handle-price-increase_v2"
```

```typescript
// dispatch-process Flow (Step 2: Dynamic call)
// Use Windmill's "Run script by path" action with variable path
const targetPath = results.resolve_target;
const result = await wmill.runScriptByPath(targetPath, input_args);
return result;
```

**Tier 1: Implementation Flows**
```
Flow: /case/lifecycle_management/handle-price-increase_playbook_v1
├─ Step 1: Get contract details (/lifecycle/contract/get-details)
├─ Step 2: Get market offers (/offer/processing/search-offers)
├─ Step 3: Calculate savings (/optimisation/calculate-savings)
├─ Step 4: Branch on savings threshold
│  ├─ If savings > €50: Auto-approve switch
│  └─ Else: Create task for manual review (suspend for approval)
├─ Step 5: Execute switch (/provider/api/initiate-cancellation)
└─ Step 6: Update contract state (/lifecycle/contract/accept-new-terms)
```

#### 2. **Hub-and-Spoke Enforcement**
Windmill folder permissions enforce communication rules:

- `/lifecycle/`: No outbound calls (only SQL queries)
- `/provider/`, `/offer/`, `/optimisation/`: Can only call `/lifecycle/` scripts
- `/case/`: Can call any domain
- `/service/`: Can only call `/case/`

Enforced via:
1. Code review in Git sync
2. Folder permissions (read-only for certain users)
3. Linting scripts (detect unauthorized cross-domain calls)

#### 3. **Configuration-Driven Routing**

Routing configs stored as Windmill Resources:
```json
// Resource: case/routing/handle-price-increase
{
  "description": "Process triggered when provider increases price",
  "input_schema": {
    "type": "object",
    "properties": {
      "contract_id": {"type": "string"},
      "new_price_cents": {"type": "integer"}
    },
    "required": ["contract_id", "new_price_cents"]
  },
  "strategy": {
    "path": "/optimisation/routing_strategies/percentage_rollout",
    "args": {
      "variants": [
        {
          "path": "/case/lifecycle_management/handle-price-increase_playbook_v1",
          "weight": 80,
          "description": "Stable deterministic flow"
        },
        {
          "path": "/case/lifecycle_management/handle-price-increase_agent_v1",
          "weight": 20,
          "description": "Experimental AI agent flow"
        }
      ]
    }
  },
  "experiment": {
    "hypothesis": "AI agent increases acceptance rate by 10%",
    "start_date": "2025-12-01",
    "end_date": "2025-12-31",
    "success_metric": "acceptance_rate"
  }
}
```

**Assessment**: **PERFECT MATCH**. Windmill flows + dynamic dispatch + inner flows = your exact architecture.

---

## Part 2: State-First Pattern (Not Event Sourcing)

### Your Requirement: "Source of trust should be states, not events"

**Windmill + Supabase Pattern**:

#### 1. **State Storage: Supabase Postgres**
```sql
CREATE TABLE contracts (
  id TEXT PRIMARY KEY,
  user_id TEXT,
  provider_id TEXT,
  status TEXT NOT NULL,  -- 'PENDING', 'ACTIVE', 'CANCELLED'
  current_price_cents INTEGER,
  activation_date TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### 2. **State Mutations: Command Handlers** (/lifecycle scripts)
```python
# /lifecycle/contract/confirm-activation
def main(db: dict, contract_id: str, activation_date: str):
    # Query current state
    current = query_contract(db, contract_id)

    # Validate transition
    if current['status'] != 'PENDING':
        raise ValueError("Invalid transition")

    # Update state
    execute_query(db, f"""
        UPDATE contracts
        SET status = 'ACTIVE', activation_date = '{activation_date}'
        WHERE id = '{contract_id}'
    """)

    return {"status": "success"}
```

#### 3. **State Reads: Query Handlers** (/lifecycle scripts)
```python
# /lifecycle/contract/get-details
def main(db: dict, contract_id: str) -> dict:
    """
    Returns authoritative state + computed properties
    """
    contract = query_contract(db, contract_id)

    # Computed properties (pure functions based on state)
    computed = {
        "is_bonus_eligible": is_within_days(contract['activation_date'], 90),
        "is_within_cancellation_window": check_cancellation_window(contract),
        "days_until_renewal": calculate_days_until_renewal(contract)
    }

    return {**contract, "computed": computed}
```

#### 4. **State-Driven Triggers: Supabase Webhooks** (Optional)
Instead of /lifecycle emitting events, use database triggers:

```sql
-- Supabase trigger function
CREATE OR REPLACE FUNCTION notify_contract_activated()
RETURNS TRIGGER AS $$
BEGIN
  PERFORM net.http_post(
    url := 'https://windmill.example.com/api/w/workspace/jobs/run/p/case/ingestion/handle-contract-activation',
    headers := jsonb_build_object('Authorization', 'Bearer TOKEN'),
    body := jsonb_build_object('contract_id', NEW.id, 'user_id', NEW.user_id)
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger on status change to ACTIVE
CREATE TRIGGER contract_activated
AFTER UPDATE OF status ON contracts
FOR EACH ROW
WHEN (NEW.status = 'ACTIVE' AND OLD.status != 'ACTIVE')
EXECUTE FUNCTION notify_contract_activated();
```

This triggers Windmill flow directly from Supabase when state changes.

#### 5. **Avoiding Event Sourcing Complexity**

**What you DON'T need**:
- ❌ Event store (Kafka, EventStoreDB)
- ❌ Event replay mechanisms
- ❌ Projections and read models
- ❌ Event versioning strategies
- ❌ Saga coordinators

**What you HAVE instead**:
- ✅ Current state in Postgres (single source of truth)
- ✅ Audit trail via `updated_at` + optional audit table
- ✅ State transitions validated in command handlers
- ✅ Flows query state on-demand (no stale data)
- ✅ Database webhooks for reactive flows (optional)

**Audit Trail (if needed)**:
```sql
CREATE TABLE contract_history (
  id SERIAL PRIMARY KEY,
  contract_id TEXT,
  previous_state JSONB,
  new_state JSONB,
  command TEXT,  -- 'ConfirmActivation', etc.
  executed_by TEXT,
  trace_id TEXT,
  executed_at TIMESTAMPTZ DEFAULT NOW()
);
```

Insert into history table in command handlers:
```python
# Before updating state
previous_state = query_contract(db, contract_id)

# Update state
execute_update(db, contract_id, new_values)

# Log to history
execute_query(db, f"""
    INSERT INTO contract_history
    (contract_id, previous_state, new_state, command, trace_id)
    VALUES ('{contract_id}', '{json.dumps(previous_state)}',
            '{json.dumps(new_state)}', 'ConfirmActivation', '{trace_id}')
""")
```

**Assessment**: ✅ **FULLY SUPPORTED**. State-first pattern with Supabase + Windmill scripts is simpler and more maintainable than event sourcing.

---

## Part 3: AI-First Architecture (Deterministic → Agent Evolution)

### Your Requirement: "Easy evolution from playbooks to agents without refactoring"

**Windmill's AI Capabilities**:

#### 1. **Phase 1: Deterministic Playbooks with LLM Steps**

```
Flow: /case/lifecycle_management/handle-price-increase_playbook_v1
├─ Step 1: Get contract (/lifecycle/contract/get-details) [Deterministic]
├─ Step 2: Extract price from email (LLM) [AI Step]
├─ Step 3: Get offers (/offer/processing/search-offers) [Deterministic]
├─ Step 4: Calculate savings (/optimisation/calculate-savings) [Deterministic]
├─ Step 5: Branch on threshold [Deterministic]
│  ├─ If savings > €50: Auto-approve
│  └─ Else: Manual review
└─ Step 6: Execute action (/provider/...) [Deterministic]
```

LLM step implementation:
```python
# Step 2: Extract price from email (Windmill Script)
from openai import OpenAI

def main(
    openai: dict,
    email_body: str,
    email_subject: str
) -> dict:
    """
    Uses LLM to extract structured data from unstructured email
    """
    client = OpenAI(api_key=openai['api_key'])

    response = client.chat.completions.create(
        model="gpt-4-turbo",
        messages=[{
            "role": "system",
            "content": """Extract price increase information from provider email.
            Return JSON: {"old_price_euros": float, "new_price_euros": float,
                         "effective_date": "YYYY-MM-DD"}"""
        }, {
            "role": "user",
            "content": f"Subject: {email_subject}\n\n{email_body}"
        }],
        response_format={"type": "json_object"}
    )

    return json.loads(response.choices[0].message.content)
```

#### 2. **Phase 2: AI Agent Flows**

Windmill supports **AI Agent steps** natively:

```
Flow: /case/lifecycle_management/handle-price-increase_agent_v1
└─ AI Agent Step (with tools)
   ├─ Tool 1: get_contract_details → /lifecycle/contract/get-details
   ├─ Tool 2: search_offers → /offer/processing/search-offers
   ├─ Tool 3: calculate_savings → /optimisation/calculate-savings
   ├─ Tool 4: initiate_cancellation → /provider/api/initiate-cancellation
   └─ Tool 5: update_contract → /lifecycle/contract/accept-new-terms
```

Agent configuration:
```json
{
  "provider": "openai",
  "model": "gpt-4-turbo",
  "system_prompt": "You are a contract optimization agent. When user's contract price increases, find better offers and execute the switch if savings exceed €50. Always prioritize user savings.",
  "tools": [
    {
      "name": "get_contract_details",
      "description": "Retrieve current contract state",
      "script_path": "/lifecycle/contract/get-details"
    },
    {
      "name": "search_offers",
      "description": "Find available market offers for contract type",
      "script_path": "/offer/processing/search-offers"
    },
    // ... more tools
  ],
  "max_iterations": 10,
  "require_approval_for": ["initiate_cancellation", "update_contract"]
}
```

The AI agent:
1. Receives goal: "Handle price increase for contract C123"
2. Calls tools (Windmill scripts) to gather context
3. Makes decisions based on context
4. Executes actions via tools
5. Returns result + reasoning chain

#### 3. **Evolution Path: No Refactoring Required**

```
Version 1 (Playbook): /case/.../handle-price-increase_playbook_v1
└─ Deterministic flow with LLM extraction step

Version 2 (Hybrid): /case/.../handle-price-increase_hybrid_v1
└─ Deterministic orchestration + AI decision step

Version 3 (Agent): /case/.../handle-price-increase_agent_v1
└─ Full AI agent with tool calling

All versions:
- Use SAME command handlers (/lifecycle/contract/...)
- Use SAME tool capabilities (/provider/..., /offer/...)
- Managed via SAME routing configuration
```

A/B test playbook vs agent:
```json
{
  "strategy": {
    "variants": [
      {"path": ".../handle-price-increase_playbook_v1", "weight": 80},
      {"path": ".../handle-price-increase_agent_v1", "weight": 20}
    ]
  },
  "success_metric": "user_acceptance_rate"
}
```

#### 4. **AI Guardrails & Observability**

**Confidence Thresholds via Suspend/Approve**:
```
Agent Flow with Approval Gate:
├─ AI Agent Step (makes recommendation)
├─ Branch on confidence score
│  ├─ If confidence > 0.95: Execute automatically
│  └─ Else: Suspend for human approval
│     └─ Send Slack message with resume URL
│     └─ Resume on approval
└─ Execute action
```

**Audit Trail**:
```python
# AI Agent step logs automatically include:
{
  "trace_id": "...",
  "agent_reasoning": "User's current contract costs €100/month. Found offer at €70/month. Savings of €360/year exceed threshold.",
  "tools_called": [
    {"tool": "get_contract_details", "result": {...}},
    {"tool": "search_offers", "result": {...}},
    {"tool": "calculate_savings", "result": {...}}
  ],
  "final_action": "initiate_cancellation",
  "confidence": 0.98
}
```

Stored in Supabase:
```sql
CREATE TABLE ai_decisions (
  id UUID PRIMARY KEY,
  trace_id TEXT,
  process_name TEXT,
  agent_version TEXT,
  reasoning JSONB,
  tools_called JSONB,
  final_action TEXT,
  confidence FLOAT,
  executed_at TIMESTAMPTZ DEFAULT NOW()
);
```

**Cost Management**:
```python
# Track LLM costs per process
CREATE TABLE llm_usage (
  id UUID PRIMARY KEY,
  trace_id TEXT,
  model TEXT,
  input_tokens INTEGER,
  output_tokens INTEGER,
  cost_usd DECIMAL(10,6),
  process_name TEXT,
  timestamp TIMESTAMPTZ DEFAULT NOW()
);
```

Dashboard queries:
```sql
-- Daily LLM costs by process
SELECT process_name, DATE(timestamp), SUM(cost_usd)
FROM llm_usage
WHERE timestamp > NOW() - INTERVAL '30 days'
GROUP BY process_name, DATE(timestamp);
```

**Assessment**: ✅ **EXCELLENT AI-READINESS**. Windmill's AI Agent steps + tool integration provide clear playbook → agent evolution path.

---

## Part 4: Critical Architecture Validations

### ✅ 1. Idempotency (Windmill is "at-least-once")

**Challenge**: Windmill guarantees "at-least-once" execution, not "exactly-once"

**Solution**: Implement idempotency keys in command handlers

```python
# /lifecycle/contract/confirm-activation (idempotent)
def main(
    db: dict,
    contract_id: str,
    activation_date: str,
    idempotency_key: str  # UUID generated by caller
):
    # Check if already executed
    existing = execute_query(db, f"""
        SELECT result FROM command_executions
        WHERE idempotency_key = '{idempotency_key}'
    """)

    if existing:
        return existing[0]['result']  # Return cached result

    # Execute command
    result = execute_activation(db, contract_id, activation_date)

    # Store execution
    execute_query(db, f"""
        INSERT INTO command_executions
        (idempotency_key, command, contract_id, result, executed_at)
        VALUES ('{idempotency_key}', 'ConfirmActivation',
                '{contract_id}', '{json.dumps(result)}', NOW())
    """)

    return result
```

**Which operations need idempotency?**
- ✅ Financial transactions (payments, credits)
- ✅ Provider API calls (cancellations, orders)
- ✅ User notifications (emails, SMS)
- ❌ Read queries (naturally idempotent)
- ❌ Pure calculations (naturally idempotent)

**Assessment**: ✅ **ADDRESSABLE**. Add idempotency keys to Business Intent Catalog commands.

---

### ✅ 2. Observability (Distributed Tracing)

**Implementation in Windmill**:

#### Trace ID Generation
```python
# /case/ingestion/process-provider-email (Ingestion Flow - Step 1)
import uuid

def generate_trace_id():
    trace_id = f"trace_{uuid.uuid4()}"
    # Store in custom flow state
    wmill.set_flow_state("trace_id", trace_id)
    return trace_id
```

#### Trace ID Propagation
```python
# Every script in the flow receives trace_id
def main(
    trace_id: str,  # Passed from flow input
    contract_id: str
):
    # Include in all logs
    print(json.dumps({
        "trace_id": trace_id,
        "contract_id": contract_id,
        "event": "processing_started"
    }))

    # Pass to downstream scripts
    result = wmill.run_script_by_path(
        "/lifecycle/contract/get-details",
        args={"contract_id": contract_id, "trace_id": trace_id}
    )
```

#### Structured Logging
All scripts use JSON logs:
```python
import json
import sys

def log(trace_id: str, event: str, data: dict):
    print(json.dumps({
        "timestamp": datetime.now().isoformat(),
        "trace_id": trace_id,
        "event": event,
        **data
    }), file=sys.stderr)
```

Windmill captures all stdout/stderr in job logs.

#### Centralized Logging Platform
1. Configure Windmill to ship logs to Datadog/OpenSearch
2. Query by trace_id to reconstruct full flow execution
3. Build dashboard:
   - Flow duration by process_name
   - Step failure rates
   - Provider API latency

**Assessment**: ✅ **FULLY SUPPORTED**. Windmill's structured logging + job history enable full distributed tracing.

---

### ✅ 3. State Machines (Explicit Transitions)

**Implementation**:

```python
# /lifecycle/lib/contract_state_machine.py (Shared module)
from enum import Enum

class ContractStatus(Enum):
    PENDING = "PENDING"
    ACTIVE = "ACTIVE"
    CANCELLED = "CANCELLED"
    EXPIRED = "EXPIRED"

VALID_TRANSITIONS = {
    ContractStatus.PENDING: [ContractStatus.ACTIVE, ContractStatus.CANCELLED],
    ContractStatus.ACTIVE: [ContractStatus.CANCELLED, ContractStatus.EXPIRED],
    ContractStatus.CANCELLED: [],  # Terminal state
    ContractStatus.EXPIRED: []     # Terminal state
}

def validate_transition(from_status: str, to_status: str) -> bool:
    """Validates state transition is allowed"""
    from_enum = ContractStatus(from_status)
    to_enum = ContractStatus(to_status)

    if to_enum not in VALID_TRANSITIONS[from_enum]:
        raise ValueError(
            f"Invalid transition: {from_status} → {to_status}. "
            f"Allowed: {[s.value for s in VALID_TRANSITIONS[from_enum]]}"
        )
    return True
```

Use in command handlers:
```python
# /lifecycle/contract/confirm-activation
from f.lifecycle.lib.contract_state_machine import validate_transition

def main(db: dict, contract_id: str):
    current = query_contract(db, contract_id)

    # Validate transition
    validate_transition(current['status'], 'ACTIVE')

    # Execute update
    # ...
```

**Generate Diagrams**:
```python
# Script to generate Mermaid state diagram from state machine definition
def generate_state_diagram():
    diagram = "stateDiagram-v2\n"
    for from_state, to_states in VALID_TRANSITIONS.items():
        for to_state in to_states:
            diagram += f"    {from_state.value} --> {to_state.value}\n"
    return diagram
```

**Assessment**: ✅ **IMPLEMENTABLE**. Use shared Python modules for state machine logic.

---

### ✅ 4. Long-Running Processes (Suspend/Resume)

**Pattern**: Multi-day processes with suspend

```
Flow: /case/lifecycle_management/retry-failed-order
├─ Step 1: Attempt order (/provider/api/submit-order)
├─ Step 2: If failed, suspend for 24 hours
│  └─ Use sleep step (Windmill supports sleep up to 7 days)
├─ Step 3: Resume, retry order
├─ Step 4: If failed again, suspend for human review
│  └─ Send email with resume URL
│  └─ Wait for approval
└─ Step 5: Resume, execute manual fallback
```

Windmill sleep:
```python
# Built-in sleep step in flow
# No code needed - configure in flow editor
# Duration: 86400 seconds (24 hours)
# Flow suspends at zero cost, resumes automatically
```

Human approval:
```python
# Step: Generate approval URLs
def main():
    resume_urls = wmill.get_resume_urls(approvals=1)

    # Send email
    send_email(
        to="ops@switchup.de",
        subject="Failed order requires review",
        body=f"Review and approve: {resume_urls['resume']}"
    )

    return {"resume_url": resume_urls['resume']}
```

**Assessment**: ✅ **PERFECT FIT**. Windmill's suspend/resume is purpose-built for long-running processes.

---

### ✅ 5. Provider Knowledge Graph

**Recommended Implementation**: Hybrid approach

**Phase 1**: Windmill Resources (simple, fast iteration)
```json
// Resource: provider/vattenfall_v1
{
  "provider_id": "vattenfall",
  "version": 1,
  "api": {...},
  "rpa": {...},
  "regions": [...]
}
```

**Phase 2**: Supabase Tables (complex queries, versioning)
```sql
CREATE TABLE provider_configs (
  provider_id TEXT,
  version INTEGER,
  config JSONB,
  active BOOLEAN,
  PRIMARY KEY (provider_id, version)
);

CREATE TABLE provider_api_endpoints (
  provider_id TEXT,
  endpoint_name TEXT,
  method TEXT,
  url_template TEXT,
  auth_type TEXT,
  PRIMARY KEY (provider_id, endpoint_name)
);

CREATE TABLE provider_rpa_selectors (
  provider_id TEXT,
  page TEXT,
  selector_name TEXT,
  selector_value TEXT,
  last_verified TIMESTAMPTZ,
  PRIMARY KEY (provider_id, page, selector_name)
);
```

**Versioning Strategy**:
```python
# /provider/lib/get_provider_config.py
def get_provider_config(db: dict, provider_id: str, version: int = None):
    """
    Returns provider config, defaults to latest active version
    """
    if version is None:
        query = f"""
            SELECT config FROM provider_configs
            WHERE provider_id = '{provider_id}' AND active = true
            ORDER BY version DESC LIMIT 1
        """
    else:
        query = f"""
            SELECT config FROM provider_configs
            WHERE provider_id = '{provider_id}' AND version = {version}
        """

    result = execute_query(db, query)
    return result[0]['config']
```

**Monitoring**:
```python
# Scheduled flow: /provider/monitoring/check-provider-health
# Runs every 15 minutes
# Tests API endpoints, reports failures
```

**Assessment**: ✅ **CRITICAL SUCCESS FACTOR**. Invest in Provider Knowledge schema upfront.

---

## Part 5: Architecture Strengths & Concerns

### ✅ **Major Strengths**

1. **Perfect Windmill Alignment**
   - Domain boundaries → Windmill folders
   - Capabilities → Scripts
   - Processes → Flows
   - Routing configs → Resources

2. **Hub-and-Spoke Enforceable**
   - Folder permissions
   - Git sync for code review
   - Linting for cross-domain call detection

3. **AI Evolution Path Clear**
   - Deterministic playbooks → AI agent flows
   - Same command handlers, same tools
   - A/B testing via routing config

4. **State-First Pattern Native**
   - Supabase = authoritative state
   - Command handlers validate transitions
   - No event sourcing complexity

5. **Operational Scalability Enabled**
   - Horizontal worker scaling
   - Suspend/resume for long processes
   - Email/webhook/schedule triggers
   - RPA for unstable provider portals

### ⚠️ **Key Concerns & Mitigations**

#### Concern 1: Offer Domain Over-Engineering

**Risk**: 8-layer ontology delays implementation

**Mitigation**:
- Phase 1: Hardcode 2-3 providers as JSONB
- Phase 2: Extract patterns after 5+ providers
- Phase 3: Build full ontology based on real variance

**Verdict**: Defer complexity, leverage Windmill's versioning

---

#### Concern 2: Provider Integrations Underspecified

**Risk**: Provider Knowledge Graph schema unclear, causes operational failures

**Mitigation**:
- **Define schema NOW** (Supabase tables)
- **Versioning strategy** (provider_id + version)
- **Monitoring dashboard** (API health, RPA selector validation)
- **Rollback capability** (switch to previous version)

**Verdict**: Elevate to P0 architecture decision

---

#### Concern 3: Idempotency Not Systematically Addressed

**Risk**: At-least-once delivery causes duplicate actions (double charges, duplicate emails)

**Mitigation**:
- Add `idempotency_key` parameter to all state-mutating commands
- Create `command_executions` table in Supabase
- Document which operations require idempotency

**Verdict**: Add to Business Intent Catalog spec

---

#### Concern 4: AI Audit Trail & Cost Management Missing

**Risk**: AI becomes expensive black box without oversight

**Mitigation**:
- Create `ai_decisions` table (reasoning, tools, confidence)
- Create `llm_usage` table (tokens, costs per process)
- Define confidence thresholds for auto-execution
- Build AI decision review dashboard

**Verdict**: Required for Phase 1

---

#### Concern 5: Observability Deferred

**Risk**: Can't debug distributed flows without tracing

**Mitigation**:
- **Define trace format NOW** (UUID, W3C Trace Context optional)
- **Propagation mechanism**: Pass `trace_id` to all scripts
- **Platform choice**: Datadog or OpenSearch (decide in architecture phase)
- **Dashboards**: Flow duration, step failures, provider API latency

**Verdict**: Architecture decision, not implementation detail

---

#### Concern 6: Flow Version Management Undefined

**Risk**: In-flight flows when deploying new version

**Mitigation**:
- **Immutability**: Never edit flows, always clone
- **Gradual rollout**: Use routing config to shift traffic
- **Suspended flows**: Document max suspension time (7 days = Windmill limit)
- **Migration plan**: Archive old versions after all instances complete

**Verdict**: Define operational procedures

---

## Part 6: Specific Feasibility Validations

### ✅ Can Windmill Handle SwitchUp's Scale?

**Performance Benchmarks** (from research):
- 26M tasks/month on single $5 worker
- 10ms cold start latency
- 13x faster than Airflow
- Horizontal worker scaling (0 → infinity)

**SwitchUp Estimated Scale**:
- 1M contracts
- 10 events/contract/year = 10M events
- Average 5 scripts/flow = 50M script executions/year
- ≈ 4M executions/month

**Verdict**: ✅ Windmill can handle 10x your expected scale on modest infrastructure

---

### ✅ Can Windmill Replace FastAPI/REST Framework?

**HTTP Routes Feature**:
- Custom paths with parameters (`/api/v1/contracts/:id`)
- All HTTP methods (GET, POST, PUT, DELETE)
- JWT authentication
- HMAC signature validation
- OpenAPI 3.1 spec generation

**Limitations**:
- No GraphQL (use REST)
- No gRPC (use REST or separate service)
- No WebSockets for persistent connections (use HTTP SSE instead)

**Verdict**: ✅ Windmill HTTP routes fully replace traditional API gateway for REST APIs

---

### ✅ Can Windmill Host Complex Business Logic?

**Language Support**:
- Python with full library support (Pandas, Polars, Pydantic)
- TypeScript with npm packages
- Rust for performance-critical paths

**Shared Logic**:
- Relative imports between scripts
- Folder-based modules
- Automatic dependency tracking

**Limitations**:
- No native Pydantic type inference (use TypedDict or dict)
- No direct dataclass support for UI generation

**Verdict**: ✅ Windmill supports complex Python/TypeScript business logic with shared modules

---

### ✅ Can Windmill Enforce Domain Boundaries?

**Mechanisms**:
1. **Folder permissions**: Top-level folders = permission scopes
2. **Git sync**: All changes via PR, enforceable in CI
3. **Code review**: Detect unauthorized cross-domain calls
4. **Linting scripts**: Scheduled flow that scans for violations

**Limitations**:
- Not enforced at runtime (scripts can technically call any path)
- Relies on team discipline + CI checks

**Verdict**: ✅ Enforceable via process, not platform restriction

---

### ✅ Can Windmill Support Multi-Team Development?

**Features**:
- Folders = projects with separate permissions
- Git sync per workspace
- Branch-based development (local dev → PR → merge → deploy)
- Workspace forks for testing

**Workflow**:
1. Developer clones workspace locally (`wmill sync pull`)
2. Develops in IDE with relative imports
3. Tests in dev workspace
4. Commits to Git, opens PR
5. CI validates (linting, routing config schema)
6. Merge triggers deploy to prod workspace

**Verdict**: ✅ Standard Git workflow supported

---

## Part 7: Final Recommendations

### 🎯 **Architecture is Sound - Focus on These**

#### Priority 0 (Before Implementation)

1. **Define Provider Knowledge Graph Schema**
   - Supabase tables: `provider_configs`, `provider_api_endpoints`, `provider_rpa_selectors`
   - Versioning strategy
   - Rollback procedures
   - Monitoring dashboard requirements

2. **Add Idempotency to Business Intent Catalog**
   - All commands take `idempotency_key: str` parameter
   - Create `command_executions` table
   - Document idempotency requirements per command

3. **Define Observability Standards**
   - Trace ID format (UUID)
   - Propagation mechanism (function parameter)
   - Platform choice (Datadog/OpenSearch)
   - Required dashboards (list metrics)

4. **AI Audit Requirements**
   - Create `ai_decisions` and `llm_usage` tables
   - Define confidence thresholds (>0.95 auto, <0.80 manual)
   - Cost monitoring alerts

#### Priority 1 (Phase 1 Simplifications)

5. **Simplify Offer Domain**
   - Start: 2-3 providers as JSONB in Supabase
   - Extract patterns after 5+ providers
   - Build ontology incrementally

6. **Start with Playbooks**
   - Deterministic flows with LLM extraction steps
   - Prove business value before agent evolution
   - A/B test playbook variants first

7. **Process Dispatcher Scope**
   - Phase 1: Simple percentage rollout
   - Defer: Complex strategies, experiment lifecycle management
   - Build sophistication as needed

#### Priority 2 (Operational Excellence)

8. **Flow Version Management**
   - Document deployment procedures
   - Gradual rollout process
   - Suspended flow handling (7-day limit)

9. **Provider Health Monitoring**
   - API endpoint tests (scheduled flows)
   - RPA selector validation
   - Alert on failures

10. **State Machine Definitions**
    - Document valid transitions per entity
    - Generate diagrams from code
    - Validate in command handlers

---

## Conclusion

### 🏆 **Overall Assessment: HIGHLY FEASIBLE**

Your architecture is **exceptionally well-designed for Windmill.dev**. The "Brain, Spine, Limbs" model, Hub-and-Spoke communication, and Configuration-Driven Agility all map perfectly to Windmill's primitives.

**Key Validations**:
- ✅ All domains implementable in Windmill (except /lifecycle = Supabase)
- ✅ State-first pattern natively supported (no event sourcing needed)
- ✅ AI evolution path clear (playbooks → agents via same tools)
- ✅ Operational scalability achievable (suspend/resume, horizontal scaling)
- ✅ Domain boundaries enforceable (folders + Git + process)

**Critical Success Factors**:
1. **Provider Knowledge Graph** schema defined upfront
2. **Idempotency** built into command handlers
3. **Observability** architected (trace ID, dashboards)
4. **AI Audit Trail** for compliance and cost management
5. **Start Simple** on Offer Domain, evolve based on real complexity

**Strategic Recommendation**:
Treat current spec as **North Star** (18-24 months), build **pragmatic Phase 1**:
- Implement 2-3 providers end-to-end
- Deterministic playbooks with LLM extraction
- Simple routing (percentage rollout)
- Provider Knowledge as Windmill Resources (migrate to Supabase later)

Leverage Windmill's versioning to evolve sophistication incrementally.

---

**You have a winning architecture. Execute with discipline on operational patterns (idempotency, observability, provider monitoring) and you'll achieve operational scalability.**
