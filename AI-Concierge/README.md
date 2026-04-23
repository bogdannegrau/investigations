# AI Concierge — Design & Technical Specification

> **Project:** AI-powered customer support concierge for Sky Bet  
> **Status:** Design Phase  
> **Date:** April 2026  
> **Scope Focus:** CCC Service — the central orchestration layer we own and build  
> **Based on:** [Slingshot AI Concierge](./slingshot-diagram.png) — original vision document by CJ

### Slingshot Vision Alignment

This design builds directly on the **Slingshot AI Concierge** proposal which establishes:

- **ConciergeInteraction Handler** hosted on `support.<<domain>>.com` as the central orchestrator
- **CET Framework** integration for native apps via `SupportConciergeInteraction(context)`
- **LivePerson** as existing provider (with intent detection → AWS Lex / IBM bot / Human)
- **AWS Bedrock** for new AI path with intent detection and MCP routing
- **Three context strategies**: webScraper, scrape with customerSession, API scraping
- **WebSocket handler** for full bidirectional AI interaction
- **Progressive Enhancement**: Support domain owns the implementation, clients adopt incrementally

---

## Table of Contents

1. [Vision & Current State](#1-vision--current-state)
2. [UX Design — AI Summary Mode](#2-ux-design--ai-summary-mode)
3. [UX Design — AI Chat Mode (AI Mode)](#3-ux-design--ai-chat-mode-ai-mode)
4. [Architecture Overview](#4-architecture-overview)
5. [CCC Service — What We Build](#5-ccc-service--what-we-build)
6. [MCP Integration Layer](#6-mcp-integration-layer)
7. [Communication Protocol](#7-communication-protocol)
8. [LivePerson Escalation](#8-liveperson-escalation)
9. [Build Scope & Priorities](#9-build-scope--priorities)

---

## 1. Vision & Current State

### Current State — support.skybet.com

The help centre presents a hero search bar ("How can we help?") with categories below (Deposit & Withdraw, Promotions & Rewards, Safer Gambling, Account Help, Betting, Market Rules, etc.).

**Search behaviour:** As the user types in the search bar (e.g. "what happened to my bets"), a dropdown of **suggested articles** appears inline below the search input. These are the articles that currently power Stage 1 — the AI Summary will be generated from these same search-suggested articles. The search endpoint lives on `/data/search` (authenticated, returns JSON article matches).

The "Contact us" button at the bottom opens **LivePerson** messenger (loaded via `lpTag.js` — `window.LivePerson.openMessenger()`).

### Problem

A customer searching **"what happened to my bets"** sees suggested articles like *"Winnings not what you expected?"*, *"Dead Heat"*, *"Rule 4 Deductions"*, *"Non Runners"* — but must self-triage across multiple articles to find their answer. There's no synthesis, no personalisation, no conversation.

### Vision

Transform the search experience through two progressive AI stages:

| Stage | Experience | Google Analogy |
|-------|-----------|----------------|
| **Stage 1** (in progress) | AI-generated summary card above search results | Google AI Overview |
| **Stage 2** | Full conversational AI Mode with context-aware chat | Google AI Mode |

---

## 2. UX Design — AI Summary Mode (Stage 1)

When a customer searches, an AI Summary card appears above the traditional article list.

### Wireframe — Search Results with AI Summary

```
┌─────────────────────────────────────────────────────────┐
│  ☁ Sky Bet Support                                      │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  🔍 why did I lose my previous bet            ✕   │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │ ✨ AI Summary                                     │  │
│  │                                                   │  │
│  │  There are several reasons your bet may not have  │  │
│  │  paid out as expected:                            │  │
│  │                                                   │  │
│  │  • **Dead Heat** — If two or more selections     │  │
│  │    tied, your winnings are divided proportionally │  │
│  │  • **Rule 4 Deduction** — A late withdrawal in   │  │
│  │    Horse Racing reduces your payout              │  │
│  │  • **Non-Runner / Void** — A selection that      │  │
│  │    didn't participate is voided, reducing         │  │
│  │    accumulator returns                           │  │
│  │  • **Each Way terms** — Place terms depend on    │  │
│  │    runners under starter's orders                │  │
│  │                                                   │  │
│  │  Sources: Winnings not what you expected? ·       │  │
│  │  Dead Heat · Rule 4 Deductions · Non Runners     │  │
│  │                                                   │  │
│  │  ┌─────────────────────────────────────────────┐ │  │
│  │  │  🤖 Ask a follow-up...          [AI Mode →] │ │  │
│  │  └─────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ── Search Results ─────────────────────────────────    │
│                                                         │
│  📄 Winnings not what you expected?                     │
│     This article explains reasons you may not have...   │
│                                                         │
│  📄 Dead Heat                                           │
│     This article explains what a dead heat is...        │
│                                                         │
│  📄 Rule 4 Deductions                                   │
│     Read this article if you placed a bet on a...       │
│                                                         │
│  📄 Cash Out                                            │
│     Cash Out allows you to lock in a profit or...       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Key UX Decisions — Stage 1
- Summary card uses a **subtle gradient background** to differentiate from articles
- **Source links** connect to existing support articles (trust & transparency)
- **"AI Mode →"** button is a clear CTA to enter conversational mode
- Summary streams in token-by-token (typewriter effect) for perceived speed

---

## 3. UX Design — AI Chat Mode (Stage 2)

Clicking **"AI Mode"** transforms the page into a conversational interface. The search query becomes the first message. This mirrors Google's AI Mode transition.

### Wireframe — AI Mode Chat

```
┌─────────────────────────────────────────────────────────┐
│  ☁ Sky Bet Support          [← Back to Search]          │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │                   🤖 AI Mode                      │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌─ You ─────────────────────────────────────────────┐  │
│  │ Why did I lose my previous bet?                   │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌─ AI Concierge ────────────────────────────────────┐  │
│  │                                                   │  │
│  │  There are several reasons your bet may not have  │  │
│  │  paid out as expected:                            │  │
│  │                                                   │  │
│  │  • **Dead Heat** — two or more selections tied    │  │
│  │  • **Rule 4 Deduction** — late withdrawal in      │  │
│  │    Horse Racing                                   │  │
│  │  • **Non-Runner / Void** — selection didn't       │  │
│  │    participate                                    │  │
│  │  • **Each Way terms** — place terms depend on     │  │
│  │    total runners                                  │  │
│  │                                                   │  │
│  │  Would you like me to check your specific bet?    │  │
│  │  I can look up your recent bets if you're logged  │  │
│  │  in.                                              │  │
│  │                                                   │  │
│  │  📎 Sources: Winnings not what you expected? ·    │  │
│  │  Dead Heat · Rule 4 · Non Runners                 │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌─ Suggested follow-ups ────────────────────────────┐  │
│  │  💬 "Check my last bet"                           │  │
│  │  💬 "What is a Rule 4 deduction?"                 │  │
│  │  💬 "How is a dead heat calculated?"              │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  💬 Type your message...                    [➤]   │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  🧑‍💼 Need a human? [Talk to an agent]             │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Deep Dive — Personalised Context (Logged-in User)

When user says "Check my last bet", the AI calls the **FBR MCP** (My Bets) and responds with:

```
┌─ AI Concierge ──────────────────────────────────────────┐
│                                                         │
│  I found your most recent bet:                          │
│                                                         │
│  ┌─ Bet #SB-29841773 ───────────────────────────────┐  │
│  │ 🏇 3:40 Cheltenham — Lucky Star (Won)            │  │
│  │ Odds: 5/1 → Paid: £4.23 (expected £6.00)         │  │
│  │                                                   │  │
│  │ ⚠️  Rule 4 Deduction Applied: 20p in the £        │  │
│  │ Reason: "Midnight Express" was a late withdrawal  │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  A **Rule 4 deduction of 20p in the £** was applied     │
│  because "Midnight Express" (priced at 3/1) withdrew    │
│  after you placed your bet. This reduced your payout    │
│  from £6.00 to £4.23.                                   │
│                                                         │
│  📎 Full Rule 4 explanation                             │
│                                                         │
│  Is there anything else about this bet?                 │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Architecture Overview

See [architecture.md](./architecture.md) for full Mermaid diagrams.

**Key architectural principle:** CCC is the **only** service we fully own and control. It sits between all consumers (Help Pages, Native App, other product pages) and all AI providers (AWS Bedrock, Anthropic, Google, OpenAI). This gives us:

- **Provider independence** — switch LLM providers without changing consumers
- **Cost control** — token budgets, rate limiting, model routing
- **Guardrails** — content filtering, PII redaction, gambling-specific safety
- **Observability** — unified logging, cost tracking, quality metrics

---

## 5. CCC Service — What We Build

> **CCC = Customer Concierge Controller**

### 5.1 Core Components

```
CCC Service
├── API Gateway Layer
│   ├── REST endpoint: POST /v1/concierge/query      (Stage 1 - Summary)
│   ├── WebSocket endpoint: /v1/concierge/chat        (Stage 2 - AI Mode)
│   └── Auth middleware (JWT/session validation)
│
├── Conversation Manager
│   ├── Session store (conversation history per user)
│   ├── Context assembler (merges MCP data + conversation)
│   └── Turn controller (manages multi-turn flow)
│
├── AI Router
│   ├── Provider registry (Bedrock, Anthropic, Google, OpenAI)
│   ├── Model selector (rules engine: cost, latency, capability)
│   ├── A/B experiment framework (trial providers in parallel)
│   ├── Fallback chain (primary → secondary → tertiary)
│   └── Response streamer (SSE/WS token streaming)
│
├── MCP Orchestrator
│   ├── Tool registry (HLP, FBR, CPP, future MCPs)
│   ├── Tool call planner (decides which MCPs to invoke)
│   ├── Parallel executor (fan-out to multiple MCPs)
│   └── Response aggregator
│
├── Guardrails Engine
│   ├── Input filter (PII detection, prompt injection defence)
│   ├── Output filter (gambling-safe responses, no financial advice)
│   ├── Topic boundary (stay within support scope)
│   └── Responsible gambling triggers (detect distress signals)
│
├── Cost Controller
│   ├── Token budget per session / per user / global
│   ├── Model tier routing (cheap model for simple, expensive for complex)
│   ├── Circuit breaker (cost ceiling per hour/day)
│   └── Usage analytics & billing aggregation
│
├── Configuration Service
│   ├── Feature flags (enable/disable AI Mode per product)
│   ├── Provider switch (hot-swap LLM without deploy)
│   ├── Prompt templates (versioned, A/B testable)
│   └── MCP endpoint registry
│
└── Escalation Handler
    ├── Confidence scorer (when to suggest human agent)
    ├── Context packager (serialise conversation for LP)
    ├── LivePerson handoff trigger
    └── Escalation analytics
```

### 5.2 API Contract

#### Stage 1 — AI Summary (REST)

```
POST /v1/concierge/query
Content-Type: application/json
Authorization: Bearer <token>     # optional — anonymous allowed

{
  "query": "why did I lose my previous bet",
  "product": "skybet",
  "locale": "en-gb",
  "context": {
    "page": "/app/home",
    "authenticated": false
  }
}

→ 200 OK (streamed via SSE or chunked transfer)
Content-Type: text/event-stream

data: {"type":"chunk","content":"There are several reasons"}
data: {"type":"chunk","content":" your bet may not have paid out"}
data: {"type":"sources","articles":["winning-rules-explained","dead-heat","rule-4-deductions"]}
data: {"type":"done","session_id":"sess_abc123","tokens_used":342}
```

#### Stage 2 — AI Mode Chat (WebSocket)

```
WS /v1/concierge/chat?session_id=sess_abc123

→ Client sends:
{"type":"message","content":"Check my last bet"}

← Server streams:
{"type":"tool_call","tool":"fbr","status":"calling"}
{"type":"tool_result","tool":"fbr","summary":"Found bet #SB-29841773"}
{"type":"chunk","content":"I found your most recent bet:"}
{"type":"chunk","content":"\n\n🏇 3:40 Cheltenham — Lucky Star"}
...
{"type":"suggestions","items":["What is Rule 4?","Check another bet"]}
{"type":"done","tokens_used":587}
```

### 5.3 Why WebSocket + SSE Hybrid?

| Concern | Recommendation | Rationale |
|---------|---------------|-----------|
| **Stage 1 Summary** | **SSE** (Server-Sent Events) | One-shot query → streamed response. Simple, HTTP-based, works through CDNs/proxies, auto-reconnect built-in. No need for bidirectional. |
| **Stage 2 Chat** | **WebSocket** | Multi-turn conversation needs bidirectional communication. Client sends messages, server streams responses. Lower latency than repeated HTTP. |
| **Native App (CET)** | **WebSocket** | Mobile frameworks handle WS well. Consistent with chat UX. |

**Why not pure WebSocket for everything?** SSE is simpler for Stage 1 — no connection upgrade, works through all proxies/CDNs, and the help page search is fundamentally request→response. WebSocket is warranted only when you need the persistent bidirectional channel of a chat.

---

## 6. MCP Integration Layer

### MCP Registry

| MCP | TLA | API | Purpose | Auth Required |
|-----|-----|-----|---------|---------------|
| Help Pages | HLP | HLP API | Search & retrieve support articles | No |
| My Bets | FBR | FBR API | Customer bet history, settlements, outcomes | Yes |
| Customer Profile | CPP | CPP API | Account details, verification status, limits | Yes |
| *(future)* Promotions | PRM | PRM API | Active offers, eligibility, T&Cs | Yes |
| *(future)* Payments | PAY | PAY API | Deposit/withdrawal status | Yes |

### How MCPs Work in CCC

CCC uses the **Model Context Protocol** pattern: each MCP exposes a set of **tools** that the LLM can invoke during a conversation. CCC acts as the MCP host.

```
Customer: "Why was my bet on Lucky Star only £4.23?"
                    │
                    ▼
         ┌── CCC AI Router ──┐
         │   LLM decides:    │
         │   1. Call FBR MCP │ ← get bet details
         │   2. Call HLP MCP │ ← get Rule 4 article
         └────────┬──────────┘
                  │
         ┌───────┴────────┐
         ▼                ▼
    FBR API           HLP API
    (bet data)        (article)
         │                │
         └───────┬────────┘
                 ▼
         LLM synthesises response
         with bet-specific context
```

---

## 7. Communication Protocol — Detailed

### WebSocket Connection Lifecycle

```
1. Client connects: ws://ccc.internal/v1/concierge/chat
2. Server authenticates (JWT in query param or first message)
3. Server sends: {"type":"connected","session_id":"sess_xyz"}
4. Client sends messages, server streams responses
5. Heartbeat: server pings every 30s, client pongs
6. Graceful close: either side sends close frame
7. Reconnect: client reconnects with session_id to resume
```

### Message Types

| Direction | Type | Purpose |
|-----------|------|---------|
| C→S | `message` | User chat message |
| C→S | `action` | User clicks suggestion chip |
| C→S | `escalate` | User requests human agent |
| S→C | `chunk` | Streamed token from LLM |
| S→C | `tool_call` | AI is calling an MCP (loading indicator) |
| S→C | `tool_result` | MCP call completed |
| S→C | `sources` | Referenced articles |
| S→C | `suggestions` | Follow-up suggestion chips |
| S→C | `escalation` | Handoff to LivePerson initiated |
| S→C | `done` | Response complete |
| S→C | `error` | Error with retry/fallback info |

---

## 8. LivePerson Escalation

### Current State
The site already loads `lpTag.js` and has a "Contact us" button calling `window.LivePerson.openMessenger()`.

### Escalation Flow

```
AI Chat (our UI)                    LivePerson
     │                                  │
     │  User: "Let me talk to someone"  │
     │  or AI confidence < threshold    │
     │                                  │
     ├──── Package context ────────────►│
     │     {                            │
     │       conversation_summary,      │
     │       customer_id,               │
     │       bet_references,            │
     │       ai_diagnosis               │
     │     }                            │
     │                                  │
     │  Show transition message:        │
     │  "Connecting you to an agent.    │
     │   They'll have the context of    │
     │   our conversation."             │
     │                                  │
     │◄──── LP window opens ────────────│
     │                                  │
     │  Our AI chat UI collapses/hides  │
     │  LP messenger takes over         │
     └──────────────────────────────────┘
```

### Context Passing to LP
- Use LP **Engagement Attributes** API to pass structured context
- Set `customerInfo`, `personalInfo`, and custom `sdes` (Structured Data Entities)
- Pass a `conversationSummary` field so the agent doesn't ask the customer to repeat themselves

### UX Compromise
Opening LP is a **separate window/widget** — not ideal but acceptable given:
- LP owns the agent routing, queuing, and agent desktop
- Building our own agent UI is out of scope
- Context handoff minimises customer repetition
- Future: explore LP's **Messaging Window API** to embed within our UI

---

## 9. Build Scope & Priorities

### What We Build (CCC Focus)

| # | Component | Priority | Effort | Description |
|---|-----------|----------|--------|-------------|
| 1 | **CCC Core Service** | 🔴 P0 | L | Node.js/TypeScript service with REST + WS endpoints |
| 2 | **AI Router + Provider Abstraction** | 🔴 P0 | L | Abstract LLM calls, support multiple providers, streaming |
| 3 | **HLP MCP** | 🔴 P0 | M | Search + retrieve help articles, expose as MCP tools |
| 4 | **Conversation Manager** | 🔴 P0 | M | Session storage, context windowing, turn management |
| 5 | **Guardrails Engine** | 🔴 P0 | M | Input/output filtering, topic boundaries, RG triggers |
| 6 | **AI Summary UI** (Stage 1) | 🔴 P0 | M | Frontend component for search results page |
| 7 | **Cost Controller** | 🟡 P1 | M | Token budgets, rate limiting, cost tracking |
| 8 | **AI Mode Chat UI** (Stage 2) | 🟡 P1 | L | Full chat interface, message rendering, suggestions |
| 9 | **FBR MCP** | 🟡 P1 | M | My Bets integration for personalised responses |
| 10 | **CPP MCP** | 🟡 P1 | M | Customer profile for context enrichment |
| 11 | **Config Service** | 🟡 P1 | S | Feature flags, provider switching, prompt management |
| 12 | **LP Escalation** | 🟢 P2 | M | Context packaging + LP handoff |
| 13 | **A/B Framework** | 🟢 P2 | M | Trial multiple LLMs in parallel |
| 14 | **CET Integration** | 🟢 P2 | M | Native app rendering via CET Framework |
| 15 | **Analytics Dashboard** | 🟢 P2 | S | Cost, quality, deflection metrics |

### What We Don't Build (Out of Scope)
- ❌ AWS Bedrock infrastructure (Platform team)
- ❌ LivePerson agent desktop customisation
- ❌ LLM model training/fine-tuning
- ❌ Help article authoring pipeline

---

## File Index

| File | Contents |
|------|----------|
| [README.md](./README.md) | This document — design overview |
| [architecture.md](./architecture.md) | Mermaid architecture diagrams |
| [ccc-components.md](./ccc-components.md) | Detailed CCC component specifications |
