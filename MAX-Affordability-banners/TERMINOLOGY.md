# Terminology Guide

> **Purpose:** Standard naming conventions for MSB affordability features across all documentation.

---

## Two Distinct Features

This investigation covers **two separate UI features** that serve different purposes and are controlled by **different backend systems**:

| Feature | Control Owner | Data Flow |
|---------|---------------|-----------|
| Deposit Limit Widget | MAX | MAX → MSB → PLS |
| Spend Budget Alert | AMS | MAX → AMS → MSB → PLS |

### 1. Deposit Limit Widget

**What it is:** A persistent UI component showing the user's current deposit limit usage with a visual progress bar.

| Attribute | Description |
|-----------|-------------|
| **Display** | Always visible in My Account (when user has active limits) |
| **Purpose** | Show current limit usage and remaining amount |
| **Visual** | Progress bar + amounts + reset date |
| **Interaction** | Click → Opens MSB Spend Budget hub |
| **Control Owner** | **MAX** (throttles: `affordability`, `HVNA-593_unified...`) |

**Screenshot:**
![Deposit Limit Widget](images/desktopweb_msb_deposit_limit_widget.png)

**Shows:**
```
┌─────────────────────────────────────────────────────────────┐
│  🔒 Deposit limit                            Remaining: £50 │
│     £0 / £50                                                │
│     ████████████████████████████████████ (0% used)         │
│     Monthly limit                 Resets: 01.05.2026, 01:00 │
└─────────────────────────────────────────────────────────────┘
```

**Code Names:**
- Web (MAX): `BudgetStatus`, `deposit-limit-card`, `affordability-section`
- Native (TBD): `BudgetLimitsCard`, `Budget.web.tsx`, `BudgetStats`
- API: `/budget-status`, `/deposit-limits`

---

### 2. Spend Budget Alert

**What it is:** A dismissible notification/alert that appears when the user crosses spending thresholds.

| Attribute | Description |
|-----------|-------------|
| **Display** | Conditional - only when threshold crossed (50%, 75%, etc.) |
| **Purpose** | Alert user they're approaching their limit |
| **Visual** | Alert box with message + action buttons |
| **Interaction** | "Find out more" or "Dismiss" |
| **Control Owner** | **AMS** (throttle: `ams.banner.throttles.budgetThresholdHit`) |

**Screenshot:**
![Spend Budget Alert](images/desktop_spend_budget_banner.png)

**Shows:**
```
┌─────────────────────────────────────────────────────────────┐
│  ⚠️  Over 50% of Spend Budget reached              [×]      │
│                                                             │
│  Heads up, You've €100 remaining of your monthly Spend     │
│  Budget. It'll reset in 25 days.                           │
│                                                             │
│  ┌─────────────────┐  ┌────────────┐                       │
│  │  Find out more  │  │  Dismiss   │                       │
│  └─────────────────┘  └────────────┘                       │
└─────────────────────────────────────────────────────────────┘
```

**Code Names:**
- Web (MAX): `budgetThresholdHit`, `BUDGET_THRESHOLD_HIT` banner type
- AMS: `BudgetThresholdHitBannerService`, `BannerType.BUDGETTHRESHOLDHIT`
- Native (TBD): `AccountBannersCard`, `bannerDetails`
- API: `budgetThresholdHit` in banners response, `AccountBannersCard` GraphQL

---

## Naming Conventions

### ✅ Preferred Terms (Use These)

| Feature | Primary Name | Acceptable Alternatives |
|---------|--------------|------------------------|
| Progress bar component | **Deposit Limit Widget** | Widget, Limit Widget |
| Threshold notification | **Spend Budget Alert** | Alert, Threshold Alert |

### ❌ Avoid These (Ambiguous)

| Term | Problem | Instead Use |
|------|---------|-------------|
| "Banner" alone | Ambiguous - could mean widget or alert | Specify: "Alert" or "Widget" |
| "Budget Status" alone | Could mean widget or API endpoint | "Deposit Limit Widget" or "budget-status API" |
| "Affordability" alone | Too broad - covers both features | Specify which feature |
| "Progress Indicator" alone | Could be confused with loading | "Deposit Limit Widget" |

---

## Quick Reference Table

| Aspect | Deposit Limit Widget | Spend Budget Alert |
|--------|---------------------|-------------------|
| **Control Owner** | MAX | AMS |
| **Visibility** | Always (when limits active) | Only at thresholds |
| **Purpose** | Show usage | Warn user |
| **Data Source** | MSB `/budget-status` or `/deposit-limits` | AMS `budgetThresholdHit` (calls MSB internally) |
| **Dismissible** | No | Yes |
| **Web Component** | `deposit-limit-card.component.js` | `banner.component.js` |
| **Native Component** | `Budget.web.tsx` | `AccountBannersCard` normalizer |
| **Throttle** | MAX: `affordability`, `HVNA-593_unified...` | AMS: `ams.banner.throttles.budgetThresholdHit` |

---

## API Naming

| API/Endpoint | Returns Data For | Owner |
|--------------|------------------|-------|
| `/budget-status` | Deposit Limit Widget (legacy) | MSB |
| `/deposit-limits` | Deposit Limit Widget (HVNA-593) | MSB |
| `/page-context` | Configuration (not directly widget/alert) | MSB |
| `budgetThresholdHit` (banner response) | Spend Budget Alert (web) | AMS |
| `AccountBannersCard` (GraphQL) | Spend Budget Alert (native) | AMS via MAX |
| `BudgetLimitsCard` (GraphQL) | Deposit Limit Widget (native) | MSB via MAX |

---

## Service Glossary

| Service | Full Name | Role in These Features |
|---------|-----------|------------------------|
| **TBD** | The Big Deal (Native App) | Native client, uses GraphQL cards |
| **CET** | Customer Engagement Toolkit | Authentication wrapper for all native customer service calls |
| **BFF** | Backend For Frontend | GraphQL gateway for native apps |
| **MAX** | My Account X | Web frontend/backend, controls Widget throttles |
| **AMS** | Account Messaging Service | Controls Spend Budget Alert, calls MSB for % data |
| **MSB** | My Spend Budget | Provides budget data to both MAX and AMS |
| **PLS** | Payment Limits Service | Source of truth for deposit limits |

### CET Framework Details

**Package:** `@flutter-global/react-native-cet-framework`

**Role:** Wraps all customer service operations in native apps, providing:
- Authentication context for API calls
- Session management (`ApiSession`, `loginCookies`)
- Login/registration flows (`useLogin`, `useJoinNow`)
- React context (`CetContext`)

**Native Data Flow:**
```
TBD App → CET Framework → BFF → MAX/AMS/MSB → PLS
              ↑
    Handles auth tokens
    and session state
```
