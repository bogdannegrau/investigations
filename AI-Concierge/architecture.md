# AI Concierge — Architecture Diagrams

> Executive-friendly Mermaid diagrams aligned with the Slingshot AI Concierge vision.  
> Render these in any Mermaid-compatible viewer (GitHub, Notion, Confluence, VS Code, etc.)

---

## 1. High-Level System Overview

This is the full picture — how the CCC (ConciergeInteraction Handler) sits at the centre of everything.

```mermaid
graph TB
    subgraph Clients ["👤 Customer Touch Points"]
        HLP["🌐 Help Pages<br/><i>support.skybet.com</i>"]
        APP["📱 Native App<br/><i>CET Framework</i>"]
        PROD["🎰 Product Pages<br/><i>Sky Bet / Sky Vegas / etc</i>"]
    end

    subgraph CCC ["🏢 support.&lt;&lt;domain&gt;&gt;.com — ConciergeInteraction Handler"]
        direction TB
        GW["🚪 API Gateway<br/><i>REST + WebSocket</i>"]
        HANDLER["⚙️ ConciergeInteraction Handler<br/><i>AI provider · context · customerSession · webScrape</i>"]
        CONV["💬 Conversation Manager<br/><i>Session store · Context assembler</i>"]
        ROUTER["🔀 AI Router<br/><i>Provider selection · A/B testing</i>"]
        GUARD["🛡️ Guardrails Engine<br/><i>Input/Output filtering · RG triggers</i>"]
        COST["💰 Cost Controller<br/><i>Token budgets · Rate limits</i>"]
        CONFIG["⚙️ Configuration Service<br/><i>Feature flags · Provider switch</i>"]
        WS["🔌 WebSocket Handler<br/><i>Streaming · Bidirectional</i>"]
        ESC["🧑‍💼 Escalation Handler<br/><i>Context packaging</i>"]
    end

    subgraph LP ["💬 LivePerson"]
        LP_INTENT["Intent Detection"]
        LP_LEX["AWS Lex Bot"]
        LP_IBM["IBM Bot"]
        LP_HUMAN["👤 Human Agent"]
    end

    subgraph AWS ["☁️ AWS"]
        BEDROCK["🧠 Bedrock<br/><i>Intent detection &<br/>routing to MCP</i>"]
        subgraph MCPs ["MCP Tools"]
            HLP_MCP["📚 Help Pages MCP<br/><i>HLP API</i>"]
            FBR_MCP["🎫 My Bets MCP<br/><i>FBR API</i>"]
            CPP_MCP["👤 Customer Profile MCP<br/><i>CPP API</i>"]
        end
    end

    subgraph Context ["📋 Context Gathering Options"]
        OPT1["Option 1: webScraper<br/><i>(customerSession)</i>"]
        OPT2["Option 2: Scrape fallback<br/><i>(scrape with customerSession)</i>"]
        OPT3["Option 3: API scraping<br/><i>FBR via session · My Bets page</i>"]
    end

    HLP -->|"search query"| GW
    APP -->|"init(context)"| GW
    PROD -->|"query"| GW

    GW --> HANDLER
    HANDLER --> CONV
    HANDLER --> GUARD
    CONV --> ROUTER
    ROUTER --> COST
    HANDLER --> WS

    ROUTER -->|"context"| BEDROCK
    HANDLER -->|"context"| LP_INTENT
    ESC -->|"conversation + context"| LP_INTENT

    LP_INTENT --> LP_LEX
    LP_INTENT --> LP_IBM
    LP_INTENT --> LP_HUMAN

    BEDROCK --> HLP_MCP
    BEDROCK --> FBR_MCP
    BEDROCK --> CPP_MCP

    HANDLER --> OPT1
    HANDLER --> OPT2
    HANDLER --> OPT3

    WS -->|"VA response"| APP
    WS -->|"passAI_response"| HLP

    style CCC fill:#FFF3E0,stroke:#E65100,stroke-width:3px
    style LP fill:#E3F2FD,stroke:#1565C0,stroke-width:2px
    style AWS fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px
    style Clients fill:#F3E5F5,stroke:#6A1B9A,stroke-width:2px
    style Context fill:#FFF9C4,stroke:#F9A825,stroke-width:1px,stroke-dasharray: 5 5
```

---

## 2. CCC Internal Architecture — What We Build

This diagram focuses purely on the CCC service internals — our build scope.

```mermaid
graph LR
    subgraph Entry ["Entry Points"]
        REST["POST /v1/concierge/query<br/><i>Stage 1: AI Summary</i>"]
        WSE["WS /v1/concierge/chat<br/><i>Stage 2: AI Mode</i>"]
    end

    subgraph Core ["CCC Core"]
        AUTH["🔐 Auth Middleware<br/><i>JWT / Session validation</i>"]
        CONV["💬 Conversation<br/>Manager"]
        CTX["📋 Context<br/>Assembler"]
    end

    subgraph AI ["AI Layer"]
        ROUTER["🔀 AI Router"]
        GUARD_IN["🛡️ Input<br/>Guardrails"]
        GUARD_OUT["🛡️ Output<br/>Guardrails"]
        STREAM["📡 Response<br/>Streamer"]
    end

    subgraph Providers ["Provider Registry"]
        P_BEDROCK["☁️ AWS Bedrock"]
        P_ANTHROPIC["🤖 Anthropic"]
        P_GOOGLE["🔍 Google"]
        P_OPENAI["💡 OpenAI"]
    end

    subgraph Tools ["MCP Orchestrator"]
        TOOL_REG["🔧 Tool Registry"]
        T_HLP["📚 HLP"]
        T_FBR["🎫 FBR"]
        T_CPP["👤 CPP"]
    end

    subgraph Control ["Control Plane"]
        CONFIG["⚙️ Config<br/><i>Feature flags<br/>Provider switch<br/>Prompt templates</i>"]
        COSTS["💰 Cost Control<br/><i>Token budgets<br/>Rate limits<br/>Circuit breaker</i>"]
        METRICS["📊 Observability<br/><i>Logs · Costs<br/>Quality · Latency</i>"]
    end

    REST --> AUTH
    WSE --> AUTH
    AUTH --> CONV
    CONV --> CTX
    CTX --> GUARD_IN
    GUARD_IN --> ROUTER

    ROUTER --> P_BEDROCK
    ROUTER --> P_ANTHROPIC
    ROUTER --> P_GOOGLE
    ROUTER --> P_OPENAI

    ROUTER --> TOOL_REG
    TOOL_REG --> T_HLP
    TOOL_REG --> T_FBR
    TOOL_REG --> T_CPP

    P_BEDROCK --> GUARD_OUT
    P_ANTHROPIC --> GUARD_OUT
    P_GOOGLE --> GUARD_OUT
    P_OPENAI --> GUARD_OUT
    GUARD_OUT --> STREAM

    CONFIG -.->|"controls"| ROUTER
    COSTS -.->|"limits"| ROUTER
    METRICS -.->|"monitors"| STREAM

    style Core fill:#FFF3E0,stroke:#E65100,stroke-width:2px
    style AI fill:#E3F2FD,stroke:#1565C0,stroke-width:2px
    style Providers fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px
    style Tools fill:#FCE4EC,stroke:#C62828,stroke-width:2px
    style Control fill:#F3E5F5,stroke:#6A1B9A,stroke-width:2px
```

---

## 3. Customer Journey — Sequence Flow

Shows the full flow from search → AI Summary → AI Mode → Escalation.

```mermaid
sequenceDiagram
    actor Customer
    participant UI as Help Pages UI
    participant CCC as CCC Handler
    participant Guard as Guardrails
    participant Router as AI Router
    participant Bedrock as AWS Bedrock
    participant HLP as HLP MCP
    participant FBR as FBR MCP
    participant LP as LivePerson

    Note over Customer,LP: 🔍 Stage 1 — AI Summary

    Customer->>UI: Search "what happened to my bets"
    UI->>UI: Show traditional search results (existing)
    UI->>CCC: POST /v1/concierge/query<br/>{query, product, context}
    CCC->>Guard: Validate input
    Guard-->>CCC: ✅ Clean
    CCC->>Router: Route to provider
    Router->>Bedrock: Send query + system prompt
    Bedrock->>HLP: Tool call: search_articles("bet lost")
    HLP-->>Bedrock: [Winnings explained, Dead Heat, Rule 4, ...]
    Bedrock-->>Router: Streamed summary response
    Router-->>CCC: Token stream
    CCC-->>UI: SSE chunks (typewriter effect)
    UI-->>Customer: AI Summary card appears above results

    Note over Customer,LP: 💬 Stage 2 — AI Mode Chat

    Customer->>UI: Clicks "AI Mode →"
    UI->>CCC: WS connect /v1/concierge/chat<br/>{session_id from Stage 1}
    CCC-->>UI: {type: connected, history: [...]}

    Customer->>UI: "Check my last bet"
    UI->>CCC: {type: message, content: "Check my last bet"}
    CCC->>Guard: Validate input
    CCC-->>UI: {type: tool_call, tool: "fbr"}
    CCC->>Router: Route with tool access
    Router->>Bedrock: Query + tools [FBR, HLP]
    Bedrock->>FBR: Tool call: get_recent_bets(customer_id)
    FBR-->>Bedrock: Bet #SB-29841773 details
    Bedrock-->>Router: Streamed personalised response
    Router-->>CCC: Token stream
    CCC-->>UI: {type: chunk} stream + {type: suggestions}
    UI-->>Customer: Bet details + Rule 4 explanation

    Note over Customer,LP: 🧑‍💼 Escalation to Human

    Customer->>UI: "Let me talk to someone"
    UI->>CCC: {type: escalate}
    CCC->>CCC: Package conversation context
    CCC->>LP: Pass context via Engagement Attributes
    CCC-->>UI: {type: escalation, message: "Connecting..."}
    UI->>LP: Open LP Messenger window
    LP-->>Customer: Agent receives context + conversation history
```

---

## 4. AI Provider Strategy — Parallel Trial Architecture

Shows how CCC enables trialling multiple LLM providers simultaneously.

```mermaid
graph TB
    subgraph CCC ["CCC AI Router"]
        REQ["Incoming Request"]
        SELECTOR["Model Selector<br/><i>Rules Engine</i>"]
        AB["A/B Experiment<br/>Framework"]

        subgraph Trial ["Parallel Trial Mode"]
            T1["Trial A<br/>AWS Bedrock<br/>Claude 3.5"]
            T2["Trial B<br/>Anthropic Direct<br/>Claude 4"]
            T3["Trial C<br/>Google<br/>Gemini 2.0"]
        end

        EVAL["Response Evaluator<br/><i>Quality · Latency · Cost</i>"]
        PICK["Best Response<br/>Selector"]
        FALLBACK["Fallback Chain<br/><i>Primary → Secondary → Tertiary</i>"]
    end

    REQ --> SELECTOR
    SELECTOR -->|"single provider"| FALLBACK
    SELECTOR -->|"experiment mode"| AB
    AB --> T1
    AB --> T2
    AB --> T3
    T1 --> EVAL
    T2 --> EVAL
    T3 --> EVAL
    EVAL --> PICK
    FALLBACK --> PICK

    style CCC fill:#FFF3E0,stroke:#E65100,stroke-width:2px
    style Trial fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px
```

---

## 5. Context Gathering Strategy

Three approaches from the Slingshot vision for getting customer-specific context.

```mermaid
graph TB
    HANDLER["ConciergeInteraction Handler<br/><i>customerSession available</i>"]

    subgraph Opt1 ["Option 1 — Preferred: MCP via API"]
        direction LR
        API_HLP["HLP API<br/><i>Help articles</i>"]
        API_FBR["FBR API<br/><i>My Bets data</i>"]
        API_CPP["CPP API<br/><i>Customer profile</i>"]
    end

    subgraph Opt2 ["Option 2 — webScraper with customerSession"]
        SCRAPE["Web Scraper<br/><i>Scrapes product pages<br/>using customer session</i>"]
        PAGES["My Bets page<br/>Account page<br/>Product pages"]
    end

    subgraph Opt3 ["Option 3 — Client passes context"]
        CLIENT["Client provides URL<br/><i>Native app maps screen<br/>to web equivalent URL</i>"]
        CCC_FETCH["CCC fetches page content<br/><i>or passes URL to AI agent</i>"]
    end

    HANDLER -->|"✅ Best: structured data"| Opt1
    HANDLER -->|"🔄 Fallback: scrape"| Opt2
    HANDLER -->|"📱 Native: URL mapping"| Opt3

    SCRAPE --> PAGES

    CLIENT --> CCC_FETCH

    style Opt1 fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px
    style Opt2 fill:#FFF9C4,stroke:#F9A825,stroke-width:2px
    style Opt3 fill:#E3F2FD,stroke:#1565C0,stroke-width:2px
```

---

## 6. LivePerson Integration & Escalation Path

The dual-path architecture: LP as legacy provider + CCC as new AI layer.

```mermaid
graph TB
    subgraph Current ["Current State — LivePerson Path"]
        C1["Customer clicks<br/>'Contact us'"]
        C2["LP Messenger opens<br/><i>lpTag.js</i>"]
        C3["LP Intent Detection"]
        C4["AWS Lex Bot"]
        C5["IBM Bot"]
        C6["👤 Human Agent"]
    end

    subgraph Future ["Future State — AI Concierge Path"]
        F1["Customer searches<br/>'what happened to my bets'"]
        F2["AI Summary appears<br/><i>Stage 1</i>"]
        F3["Customer enters AI Mode<br/><i>Stage 2</i>"]
        F4["CCC handles conversation<br/><i>with MCPs</i>"]
        F5{"Resolved?"}
        F6["✅ Customer satisfied"]
        F7["📦 Package context"]
        F8["Open LP Messenger<br/><i>with conversation context</i>"]
    end

    C1 --> C2 --> C3
    C3 --> C4
    C3 --> C5
    C3 --> C6

    F1 --> F2 --> F3 --> F4 --> F5
    F5 -->|"Yes"| F6
    F5 -->|"No — escalate"| F7
    F7 --> F8
    F8 --> C6

    style Current fill:#E3F2FD,stroke:#1565C0,stroke-width:2px
    style Future fill:#FFF3E0,stroke:#E65100,stroke-width:2px
```

---

## 7. Technology Stack Decision

```mermaid
mindmap
  root((CCC<br/>Service))
    Runtime
      Node.js / TypeScript
      Fastify or Express
      ws library for WebSocket
    Communication
      SSE for Stage 1 Summary
      WebSocket for Stage 2 Chat
      JSON message protocol
    Storage
      Redis — Session & conversation cache
      DynamoDB — Conversation archive
      S3 — Analytics & logs
    AI Integration
      AWS Bedrock SDK
      Anthropic SDK
      LangChain or Vercel AI SDK
        Provider abstraction
        Streaming support
        Tool calling
    MCP Layer
      HLP — Help article search & retrieval
      FBR — Customer bet history
      CPP — Customer profile & status
    Guardrails
      Custom input sanitiser
      PII redaction pipeline
      Responsible gambling detector
      Topic boundary enforcer
    Observability
      CloudWatch metrics
      Distributed tracing
      Cost per conversation tracking
    Deployment
      ECS / EKS
      ALB with WebSocket support
      Auto-scaling on connection count
```

---

## 8. CCC Build Phases — Roadmap

```mermaid
gantt
    title CCC Build Phases
    dateFormat YYYY-MM-DD
    axisFormat %b

    section Phase 1 — Foundation
    CCC Core Service + API Gateway     :p1a, 2026-05-01, 3w
    AI Router + Bedrock Integration     :p1b, after p1a, 2w
    HLP MCP (Help articles)             :p1c, after p1a, 2w
    Guardrails Engine v1                :p1d, after p1b, 2w
    AI Summary UI (Stage 1)             :p1e, after p1c, 2w

    section Phase 2 — AI Mode
    Conversation Manager                :p2a, after p1e, 2w
    WebSocket Handler                   :p2b, after p1e, 2w
    AI Mode Chat UI (Stage 2)           :p2c, after p2a, 3w
    FBR MCP (My Bets)                   :p2d, after p2a, 2w
    CPP MCP (Customer Profile)          :p2e, after p2d, 2w
    Cost Controller                     :p2f, after p2b, 2w

    section Phase 3 — Scale & Polish
    Config Service + Provider Switch    :p3a, after p2c, 2w
    A/B Provider Framework              :p3b, after p3a, 2w
    LP Escalation + Context Handoff     :p3c, after p2c, 2w
    CET Framework Integration           :p3d, after p3c, 2w
    Analytics Dashboard                 :p3e, after p3b, 2w
```

---

> **Next:** See [ccc-components.md](./ccc-components.md) for detailed component specifications.
