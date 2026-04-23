# CCC Service — Detailed Component Specifications

> What exactly needs to be built inside CCC, with enough detail to start engineering.

---

## 1. ConciergeInteraction Handler

This is the core entry point — exactly as described in the Slingshot vision. It is the handler that:
- The Help Pages search invokes for AI Summary (Stage 1) and AI Mode (Stage 2)
- The CET Framework invokes from native apps via `SupportConciergeInteraction(context)`
- Any product page can invoke for embedded AI support

### Responsibilities

| Concern | Detail |
|---------|--------|
| **AI Provider** | Selects which LLM provider to route to (configurable, hot-swappable) |
| **Context** | Assembles conversation context from MCPs, page data, and history |
| **customerSession** | Validates and forwards authenticated session for personalised queries |
| **webScrape** | Falls back to scraping product pages when API data is unavailable |

### Init Contract (from CET Framework / Native App)

```typescript
interface ConciergeInitContext {
  product: 'skybet' | 'skyvegas' | 'skycasino' | 'skypoker' | 'skybingo';
  locale: string;                  // e.g. 'en-gb'
  page: string;                    // current page URL or screen identifier
  authenticated: boolean;
  customerSession?: string;        // session token for authenticated calls
  initialQuery?: string;           // pre-populated search/question
  referrer?: string;               // where the customer came from
}
```

### Handler Lifecycle

```
1. init(context)
   ├── Validate context
   ├── Resolve AI provider from Config Service
   ├── Establish session in Conversation Manager
   └── Return session_id + capabilities

2. query(session_id, message)       ← Stage 1 (summary) or Stage 2 (chat turn)
   ├── Input guardrails check
   ├── Assemble full context (history + MCP data)
   ├── Route to AI provider
   ├── Stream response through output guardrails
   └── Return streamed response + metadata

3. escalate(session_id)
   ├── Package conversation for LP
   ├── Trigger LP handoff
   └── Return escalation confirmation
```

---

## 2. AI Router — Provider Abstraction

The AI Router is what gives us **provider independence**. Every LLM call goes through this layer.

### Interface

```typescript
interface AIProvider {
  id: string;                      // 'bedrock', 'anthropic', 'google', 'openai'
  name: string;
  
  // Core capability
  chat(request: ChatRequest): AsyncIterable<ChatChunk>;
  
  // Tool/MCP support
  supportsTools: boolean;
  
  // Health & cost
  healthCheck(): Promise<boolean>;
  costPerToken: { input: number; output: number };
}

interface ChatRequest {
  messages: Message[];
  tools?: ToolDefinition[];        // MCP tools available
  systemPrompt: string;
  maxTokens: number;
  temperature: number;
  stream: boolean;
}

interface ChatChunk {
  type: 'text' | 'tool_call' | 'tool_result' | 'done' | 'error';
  content?: string;
  toolCall?: { name: string; arguments: Record<string, unknown> };
  usage?: { inputTokens: number; outputTokens: number };
}
```

### Provider Selection Logic

```typescript
function selectProvider(request: ChatRequest, config: RouterConfig): AIProvider {
  // 1. Check A/B experiment assignment
  if (config.experiment?.active) {
    return config.experiment.assignProvider(request.sessionId);
  }
  
  // 2. Check feature flags (e.g., "use anthropic for skybet")
  const flaggedProvider = config.featureFlags.getProvider(request.product);
  if (flaggedProvider) return flaggedProvider;
  
  // 3. Check cost budget (route to cheaper model if budget low)
  if (costController.isNearLimit()) {
    return config.providers.cheapest;
  }
  
  // 4. Default primary with fallback chain
  return config.providers.primary;  // fallback chain: primary → secondary → tertiary
}
```

### Fallback Behaviour

```
Request → Primary Provider (e.g., Bedrock)
              │
              ├── Success → Return response
              │
              ├── Timeout (5s) → Secondary Provider (e.g., Anthropic Direct)
              │                       │
              │                       ├── Success → Return response
              │                       └── Failure → Tertiary (e.g., Google)
              │
              └── Error → Secondary Provider
```

---

## 3. MCP Orchestrator

Manages the **Model Context Protocol** tools that the LLM can invoke during conversations.

### Tool Definitions

```typescript
// HLP MCP — Help Pages
const hlpTools: ToolDefinition[] = [
  {
    name: 'hlp_search_articles',
    description: 'Search Sky Bet help/support articles by query',
    parameters: {
      query: { type: 'string', description: 'Search query' },
      product: { type: 'string', description: 'Product filter', enum: ['skybet', 'skyvegas', ...] },
      limit: { type: 'number', description: 'Max results', default: 5 }
    }
  },
  {
    name: 'hlp_get_article',
    description: 'Get full content of a specific help article',
    parameters: {
      slug: { type: 'string', description: 'Article slug e.g. "rule-4-deductions"' }
    }
  }
];

// FBR MCP — My Bets (requires auth)
const fbrTools: ToolDefinition[] = [
  {
    name: 'fbr_get_recent_bets',
    description: 'Get customer recent bet history with outcomes and settlements',
    parameters: {
      limit: { type: 'number', default: 5 },
      status: { type: 'string', enum: ['all', 'won', 'lost', 'void', 'pending'] }
    }
  },
  {
    name: 'fbr_get_bet_detail',
    description: 'Get detailed information about a specific bet including deductions',
    parameters: {
      betId: { type: 'string', description: 'Bet reference number' }
    }
  }
];

// CPP MCP — Customer Profile (requires auth)
const cppTools: ToolDefinition[] = [
  {
    name: 'cpp_get_profile_summary',
    description: 'Get customer account status, verification state, and limits',
    parameters: {}
  },
  {
    name: 'cpp_get_verification_status',
    description: 'Check if customer has pending verification requirements',
    parameters: {}
  }
];
```

### Tool Execution Flow

```
LLM responds with tool_call
        │
        ▼
MCP Orchestrator
  ├── Validate tool name exists in registry
  ├── Check auth requirements (FBR/CPP need customerSession)
  ├── Check rate limits for this tool
  ├── Execute API call to backend service
  ├── Transform response to LLM-friendly format
  └── Feed result back to LLM for next step
```

### Parallel Execution

When the LLM requests multiple tools simultaneously (e.g., "get bet details AND find Rule 4 article"), the orchestrator fans out:

```typescript
async function executeTools(toolCalls: ToolCall[]): Promise<ToolResult[]> {
  return Promise.all(
    toolCalls.map(call => executeToolCall(call))
  );
}
```

---

## 4. Guardrails Engine

Gambling-specific safety layer that every message passes through.

### Input Guardrails

| Check | Action | Example |
|-------|--------|---------|
| **PII Detection** | Redact before sending to LLM | Card numbers, passwords |
| **Prompt Injection** | Block & log | "Ignore previous instructions..." |
| **Off-topic** | Redirect to support scope | "What's the weather?" |
| **Distress Detection** | Trigger RG pathway | "I've lost everything" |

### Output Guardrails

| Check | Action | Example |
|-------|--------|---------|
| **Financial Advice** | Strip/rephrase | Never predict bet outcomes |
| **Gambling Encouragement** | Block | Never suggest placing bets |
| **Competitor Mentions** | Neutralise | Don't recommend other bookmakers |
| **Accuracy** | Source-check | Ensure claims match HLP article content |
| **Responsible Gambling** | Inject when relevant | Link to safer gambling resources |

### Responsible Gambling Triggers

```typescript
const RG_PATTERNS = [
  /lost (everything|all my money|too much)/i,
  /can't stop (betting|gambling)/i,
  /addicted/i,
  /borrow(ed|ing)? money to (bet|gamble)/i,
  /help me stop/i
];

// When triggered:
// 1. Acknowledge the customer's feelings empathetically
// 2. Provide SaferGambling resources link
// 3. Offer self-exclusion information
// 4. Suggest GamCare helpline: 0808 8020 133
// 5. Do NOT continue normal support flow
```

---

## 5. Cost Controller

Token-level budget management to prevent runaway AI costs.

### Budget Hierarchy

```
Global Daily Budget: £X,000
  └── Per-Product Budget: £X00
       └── Per-Session Budget: X,000 tokens
            └── Per-Turn Budget: X,000 tokens
```

### Model Tier Routing

| Query Complexity | Model Tier | Cost | Example |
|-----------------|------------|------|---------|
| Simple FAQ | Small/Fast | £0.001 | "How do I withdraw?" |
| Multi-article synthesis | Medium | £0.01 | "Why was my bet less than expected?" |
| Personalised + tools | Large | £0.05 | "Check my last bet and explain the deduction" |

### Circuit Breaker

```typescript
if (dailyCost > dailyBudget * 0.9) {
  // Switch to cheapest model
  router.forceProvider('cheapest');
  alertOps('Cost approaching daily limit');
}

if (dailyCost > dailyBudget) {
  // Disable AI, fall back to traditional search
  featureFlags.disable('ai_concierge');
  alertOps('AI Concierge disabled — daily budget exceeded');
}
```

---

## 6. Configuration Service

Hot-swappable configuration without deployments.

### Config Schema

```json
{
  "ai_concierge": {
    "enabled": true,
    "products": {
      "skybet": { "enabled": true, "stage2_enabled": true },
      "skyvegas": { "enabled": true, "stage2_enabled": false },
      "skycasino": { "enabled": false }
    },
    "providers": {
      "primary": "bedrock-claude-sonnet",
      "secondary": "anthropic-claude-sonnet",
      "tertiary": "google-gemini-2"
    },
    "experiment": {
      "active": true,
      "name": "provider-comparison-q2",
      "splits": {
        "bedrock-claude-sonnet": 50,
        "anthropic-claude-4": 30,
        "google-gemini-2": 20
      }
    },
    "cost": {
      "daily_budget_gbp": 500,
      "session_token_limit": 8000,
      "turn_token_limit": 2000
    },
    "guardrails": {
      "pii_redaction": true,
      "rg_detection": true,
      "topic_boundary": true
    },
    "escalation": {
      "confidence_threshold": 0.3,
      "max_turns_before_suggest": 5,
      "lp_context_passing": true
    }
  }
}
```

---

## 7. WebSocket Protocol — Full Specification

### Connection

```
Client → WS /v1/concierge/chat
         ?session_id=sess_abc123         // resume existing
         &token=<jwt>                     // auth (or send in first message)
         &product=skybet
```

### Message Types — Client → Server

```typescript
// User sends a chat message
{ type: 'message', content: 'Why did my bet only pay £4.23?' }

// User clicks a suggestion chip
{ type: 'action', action: 'suggestion', value: 'Check my last bet' }

// User requests escalation to human
{ type: 'escalate', reason: 'user_requested' }

// Heartbeat
{ type: 'ping' }
```

### Message Types — Server → Client

```typescript
// Connection established
{ type: 'connected', session_id: 'sess_abc123', capabilities: ['hlp', 'fbr', 'cpp'] }

// AI is calling a tool (show loading indicator)
{ type: 'tool_call', tool: 'fbr', action: 'get_recent_bets', status: 'calling' }

// Tool result summary (for UI context)
{ type: 'tool_result', tool: 'fbr', summary: 'Found 3 recent bets', status: 'complete' }

// Streamed text token
{ type: 'chunk', content: 'Your bet on Lucky Star' }

// Source articles referenced
{ type: 'sources', articles: [{ slug: 'rule-4-deductions', title: 'Rule 4 Deductions' }] }

// Follow-up suggestions
{ type: 'suggestions', items: ['What is Rule 4?', 'Check another bet', 'Talk to an agent'] }

// Response complete
{ type: 'done', tokens_used: 587, turn_id: 3 }

// Escalation initiated
{ type: 'escalation', lp_context_passed: true, message: 'Connecting you to an agent...' }

// Error
{ type: 'error', code: 'RATE_LIMITED', message: 'Please wait a moment', retryAfter: 5 }

// Heartbeat response
{ type: 'pong' }
```

### Session Resumption

If WebSocket disconnects, the client can reconnect with the same `session_id`. The server replays:
1. `connected` message with session state
2. Last `done` message (so client knows where conversation left off)
3. Client can then send new messages

---

## 8. Conversation Manager — Session Storage

### Session Structure

```typescript
interface ConversationSession {
  sessionId: string;
  customerId?: string;             // if authenticated
  product: string;
  locale: string;
  createdAt: Date;
  lastActivityAt: Date;
  
  // Conversation
  messages: Message[];             // full history
  turnCount: number;
  
  // Context gathered from MCPs
  mcpContext: {
    hlp?: ArticleReference[];      // articles retrieved
    fbr?: BetSummary[];            // bets looked up
    cpp?: ProfileSummary;          // customer profile
  };
  
  // Tracking
  tokensUsed: number;
  provider: string;                // which LLM served this session
  experimentGroup?: string;        // A/B test assignment
  escalated: boolean;
  resolvedByAI: boolean;
}
```

### Storage Strategy

| Store | Purpose | TTL |
|-------|---------|-----|
| **Redis** | Active session cache, fast read/write | 1 hour |
| **DynamoDB** | Session archive for analytics | 90 days |
| **S3** | Full conversation logs for compliance | As required |

---

## 9. LivePerson Context Handoff

### Structured Data Entities (SDEs) for LP

```javascript
// Pass via lpTag.sdes before opening messenger
lpTag.sdes = lpTag.sdes || [];
lpTag.sdes.push({
  type: 'ctmrinfo',
  info: {
    cstatus: 'ai-escalation',
    customerId: session.customerId,
    companyBranch: session.product
  }
});

// Custom SDE for AI conversation context
lpTag.sdes.push({
  type: 'service',
  service: {
    topic: 'ai-concierge-escalation',
    category: detectCategory(session),  // e.g., 'betting-query'
    serviceId: session.sessionId
  }
});

// Store full context in a retrievable location
// Agent can access via internal tool using sessionId
const escalationContext = {
  conversationSummary: generateSummary(session.messages),
  customerQuery: session.messages[0].content,
  aiDiagnosis: extractDiagnosis(session),
  betReferences: session.mcpContext.fbr?.map(b => b.betId),
  articlesReferenced: session.mcpContext.hlp?.map(a => a.slug),
  turnCount: session.turnCount,
  escalationReason: reason
};
```

---

> This is the **build specification** for CCC. Each section maps to a buildable component with clear interfaces.
