# MSB Affordability Features

This investigation documents two distinct features for deposit limit management across Web (MAX) and Native (TBD) platforms.

---

## Two Features

| Feature | What It Is | When Shown | Control Owner |
|---------|------------|------------|---------------|
| **[Deposit Limit Widget](./DEPOSIT-LIMIT-WIDGET.md)** | Progress bar showing limit usage | Always visible (when limits active) | MAX |
| **[Spend Budget Alert](./SPEND-BUDGET-ALERT.md)** | Threshold notification | At 50%, 75%, 90%, 100% thresholds | AMS |

> **Naming:** See [TERMINOLOGY.md](./TERMINOLOGY.md) for standard terms.

---

## Screenshots

| Feature | Desktop Web | Native |
|---------|-------------|--------|
| **Deposit Limit Widget** | ![Widget](./images/desktopweb_msb_deposit_limit_widget.png) | ![Native](./images/msb_native_deposit_limit_widget.jpg) |
| **Spend Budget Alert** | ![Alert](./images/desktop_spend_budget_banner.png) | [Conceptual](./images/native_spend_budget_banner_concept.md) |

---

## Quick Comparison

| Aspect | Deposit Limit Widget | Spend Budget Alert |
|--------|---------------------|-------------------|
| **Purpose** | Show current usage | Warn at threshold |
| **Visibility** | Always | Conditional |
| **Dismissible** | No | Yes |
| **Control Owner** | MAX (throttles) | AMS (throttles) |
| **Web Data Source** | MSB `/budget-status` | AMS `budgetThresholdHit` → MSB |
| **Native System** | `BudgetLimitsCard` GraphQL | `AccountBannersCard` GraphQL |

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                           CLIENTS                                       │
├──────────────────────────────┬─────────────────────────────────────────┤
│   WEB (MAX)                  │   NATIVE (TBD)                          │
│   • platformConfig           │   • GraphQL BudgetLimitsCard            │
│   • budgetThresholdHit       │   • GraphQL AccountBannersCard          │
└──────────────┬───────────────┴──────────────┬──────────────────────────┘
               │                              │
               │                              ▼
               │               ┌──────────────────────────────┐
               │               │      CET FRAMEWORK           │
               │               │  (@flutter-global/react-     │
               │               │   native-cet-framework)      │
               │               │  • Authentication wrapper    │
               │               │  • Session management        │
               │               │  • Customer service calls    │
               │               └──────────────┬───────────────┘
               │                              │
               └──────────────┬───────────────┘
                              ▼
               ┌──────────────────────────────┐
               │         MAX BACKEND          │
               │   • UserContextInterceptor   │
               │   • BFF GraphQL Gateway      │
               └──────────────┬───────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   │
┌─────────────────┐  ┌─────────────────┐         │
│   MSB BACKEND   │  │  AMS BACKEND    │         │
│ (Widget Data)   │  │ (Alert Data)    │         │
│ /budget-status  │  │ budgetThreshold │         │
│ /deposit-limits │  │ HitBannerService│         │
└────────┬────────┘  └────────┬────────┘         │
         │                    │                   │
         │                    ▼                   │
         │           ┌─────────────────┐         │
         │           │   MSB BACKEND   │◄────────┘
         │           │ (AMS calls MSB  │
         │           │  for % data)    │
         │           └────────┬────────┘
         │                    │
         └────────────┬───────┘
                      ▼
               ┌──────────────────────────────┐
               │   PLS (Payment Limits)       │
               └──────────────────────────────┘
```

**Data Flow Summary:**
| Feature | Web Flow | Native Flow |
|---------|----------|-------------|
| Deposit Limit Widget | MAX → MSB → PLS | TBD → CET → BFF → MAX → MSB → PLS |
| Spend Budget Alert | MAX → AMS → MSB → PLS | TBD → CET → BFF → MAX → AMS → MSB → PLS |

---

## Documentation

| Document | Description |
|----------|-------------|
| [TERMINOLOGY.md](./TERMINOLOGY.md) | Naming conventions |
| [DEPOSIT-LIMIT-WIDGET.md](./DEPOSIT-LIMIT-WIDGET.md) | Widget data flow & components |
| [SPEND-BUDGET-ALERT.md](./SPEND-BUDGET-ALERT.md) | Alert data flow & AMS integration |
| [THROTTLES.md](./THROTTLES.md) | Feature flags, circuit breakers & controls |
