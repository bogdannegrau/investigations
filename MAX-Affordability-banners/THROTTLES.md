# Throttles & Controls

> **Purpose:** All configuration options that control visibility and behavior of the Deposit Limit Widget and Spend Budget Alert across service layers.

---

## Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              CONTROL LAYERS                                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  DEPOSIT LIMIT WIDGET:                                                               │
│  Web:    MAX → MSB → PLS                                                            │
│               ↑                                                                      │
│          Throttles, Circuit Breaker                                                 │
│                                                                                      │
│  Native: TBD → CET → BFF → MAX → MSB → PLS                                          │
│                             ↑                                                        │
│                        Throttles, Circuit Breaker                                   │
│                                                                                      │
│  SPEND BUDGET ALERT:                                                                 │
│  Web:    MAX → AMS → MSB → PLS                                                      │
│                ↑                                                                     │
│           Throttles (dateBetween, percentBetween), User Flags                       │
│                                                                                      │
│  Native: TBD → CET → BFF → MAX → AMS → MSB → PLS                                    │
│                                   ↑                                                  │
│                              Throttles, User Flags                                  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘

  CET Framework = Authentication wrapper for all native customer service calls
```

| Feature | Control Owner | Web Flow | Native Flow |
|---------|---------------|----------|-------------|
| Deposit Limit Widget | MAX | MAX → MSB → PLS | TBD → CET → BFF → MAX → MSB → PLS |
| Spend Budget Alert | AMS | MAX → AMS → MSB → PLS | TBD → CET → BFF → MAX → AMS → MSB → PLS |

---

## 1. Summary Table

### By Service Layer

| Service | Deposit Limit Widget | Spend Budget Alert |
|---------|---------------------|-------------------|
| **MAX** | ✅ Owner (throttles, circuit breaker) | Passthrough only |
| **AMS** | Not involved | ✅ Owner (throttles, user flags) |
| **MSB** | Data source (budget status) | Data source (budget status) |
| **PLS** | Source of truth (limits) | Source of truth (limits) |
| **TBD** | GraphQL `BudgetLimitsCard` | GraphQL `AccountBannersCard` |

### By Control Type

| Control Type | Deposit Limit Widget | Spend Budget Alert |
|--------------|---------------------|-------------------|
| **Feature Throttle** | MAX: `affordability`, `HVNA-593` | AMS: `ams.banner.throttles.budgetThresholdHit` |
| **User Flags** | MAX: `AFFORDABILITY_FLAGS` (legacy only) | AMS: `BESPOKE_NDL_SET`, `CDD_*` |
| **Circuit Breaker** | MAX → MSB | AMS → MSB |
| **Date Range** | N/A | AMS: `dateBetween` (e.g., days 1-20) |
| **Percent Range** | N/A | AMS: `percentBetween` (e.g., 50-90%) |
| **Rate Limit** | MAX: Hazelcast | MAX: Hazelcast (for API calls) |
| **GraphQL Card** | TBD: `BudgetLimitsCard` | TBD: `AccountBannersCard` |
| **Selector Filter** | TBD: PDL excluded | TBD: `rg` type excluded |

### Quick Reference: How to Control Each Feature

| Action | Deposit Limit Widget | Spend Budget Alert |
|--------|---------------------|-------------------|
| **Enable** | MAX: Turn ON `HVNA-593_unified...` throttle | AMS: Configure `budgetThresholdHit` in Okapi |
| **Disable globally** | MAX: Turn OFF all throttles | AMS: Remove throttle from Okapi |
| **Disable for user** | Remove affordability flags (legacy) | Ensure no CDD_*/BESPOKE_NDL_SET flags |
| **Limit visibility** | N/A | AMS: Adjust `dateBetween`/`percentBetween` |
| **Emergency shutoff** | MAX: Circuit breaker opens on failures | AMS: Circuit breaker opens on failures |

---

## 2. MAX Backend Controls (Deposit Limit Widget)

> **Applies to:** Deposit Limit Widget only

### 1.1 Feature Throttles

| Throttle | Controls | Effect |
|----------|----------|--------|
| `affordability` | Deposit Limit Widget (legacy) | When ON: fetches `/budget-status`, shows widget |
| `HVNA-593_unified_deposit_limits_widget_max` | Deposit Limit Widget (new) | When ON: fetches `/deposit-limits`, uses new UI |

**Location:** `max/application/src/main/java/com/betfair/service/max/constant/Constants.java`
```java
String THROTTLE_AFFORDABILITY = "affordability";
String THROTTLE_IS_UNIFIED_DEPOSIT_LIMITS_REDESIGN = "HVNA-593_unified_deposit_limits_widget_max";
```

**Logic (UserContextInterceptor.java):**
```java
// Legacy flow - requires affordability throttle + user flags
if (!throttleUtils.isFeatureActive(THROTTLE_IS_UNIFIED_DEPOSIT_LIMITS_REDESIGN) 
    && throttleUtils.isFeatureActive(THROTTLE_AFFORDABILITY) 
    && isAffordabilityFlow()) {
    // Fetch budget-status
}

// New flow - only requires HVNA-593 throttle
if (throttleUtils.isFeatureActive(THROTTLE_IS_UNIFIED_DEPOSIT_LIMITS_REDESIGN)) {
    // Fetch deposit-limits
}
```

### 1.2 User Flags (AFFORDABILITY_FLAGS)

**Required for legacy `affordability` throttle flow:**

| Flag | Description |
|------|-------------|
| `rgiRequiredAccountLimited` | RGI required - account limited |
| `sgiCompleteNdlSet` | Bespoke NDL set |
| `sofInvestigationAccountLimited` | SOF investigation - account limited |
| `sofInvestigationAccountLimitedHighRisk` | SOF investigation - high risk |
| `cddAccountLimited` | CDD account limited |
| `cddInvestigationInProgress` | CDD investigation in progress |
| `cddInvestigationComplete` | CDD investigation complete |

**Location:** `Constants.java:229`
```java
List<String> AFFORDABILITY_FLAGS = ImmutableList.of(
    RGI_REQUIRED_ACCOUNT_LIMITED, BESPOKE_NDL_SET,
    SOF_INVESTIGATION_ACCOUNT_LIMITED, SOF_INVESTIGATION_ACCOUNT_LIMITED_HIGH_RISK,
    CDD_ACCOUNT_LIMITED, CDD_INVESTIGATION_IN_PROGRESS, CDD_INVESTIGATION_COMPLETE
);
```

**Note:** New HVNA-593 flow does NOT require these flags.

### 1.3 Circuit Breaker (Resilience4j)

**Protects calls to MSB from MAX:**

| Parameter | Value | Description |
|-----------|-------|-------------|
| `failureRateThreshold` | 30% | Opens circuit at 30% failure rate |
| `slowCallRateThreshold` | 80% | Opens circuit at 80% slow calls |
| `slowCallDurationThreshold` | 1000ms | Calls > 1s considered slow |
| `waitDurationInOpenState` | 20000ms | 20s wait before half-open |
| `slidingWindowSize` | 100 | Window of 100 calls |
| `minimumNumberOfCalls` | 50 | Min calls before calculating rate |
| `permittedNumberOfCallsInHalfOpenState` | 10 | Test calls in half-open |

**Location:** `max/clients/src/.../util/GlobalCircuitBreaker.java`
```java
CircuitBreakerConfig.custom()
    .failureRateThreshold(30)
    .slowCallRateThreshold(80)
    .slowCallDurationThreshold(Duration.ofMillis(1000))
    .permittedNumberOfCallsInHalfOpenState(10)
    .minimumNumberOfCalls(50)
    .slidingWindowSize(100)
    .waitDurationInOpenState(Duration.ofMillis(20000))
    .build();
```

**Exceptions Recorded:**
- `CougarServiceException`
- `SocketTimeoutException`

**JMX Configurable:** Yes - can adjust at runtime

### 1.4 Rate Limiting (Hazelcast)

**Per-path rate limiting:**

| Parameter | Description |
|-----------|-------------|
| `limit` | Requests per window |
| `duration` | ISO 8601 duration (e.g., "PT1M" = 1 minute) |

**Rate Limiter Types:**
- `IP_ADDRESS` - Rate limit by client IP
- `ACCOUNT_ID` - Rate limit by user account

**Location:** `max/application/src/.../util/PathRateLimit.java`
```java
public class PathRateLimit {
    private int limit;        // requests per window
    private String duration;  // ISO 8601 "PT1M" = 1 minute
}
```

**HTTP Response:** Returns `403 Forbidden` when limit exceeded

---

## 3. AMS Controls (Spend Budget Alert)

> **Applies to:** Spend Budget Alert only

### 2.1 What AMS Provides

**AMS is the source of the Spend Budget Alert (`budgetThresholdHit` banner). AMS calls MSB internally to get budget data.**

| Banner Type | Priority | Source | AMS Involved? |
|-------------|----------|--------|---------------|
| `kyc` | 1 | AMS | ✅ Yes |
| `news` | 2 | AMS | ✅ Yes |
| `rg` | 3 | AMS | ✅ Yes |
| `budget` | 4 | AMS | ✅ Yes |
| `budgetThresholdHit` | 5 | **AMS → MSB** | ✅ Yes - **Spend Budget Alert** |
| `phone` | 6 | AMS | ✅ Yes |
| `paypal` | 7 | AMS | ✅ Yes |

**BannerType.java (AMS):**
```java
public enum BannerType {
    KYC("kyc", 1),
    NEWS("news", 2),
    RG("rg", 3),
    BUDGET("budget", 4),
    BUDGETTHRESHOLDHIT("budgetThresholdHit", 5),  // ← Spend Budget Alert
    PHONE("phone", 6),
    PAYPAL("paypal", 7);
}
```

### 2.2 budgetThresholdHit Throttle

**Throttle Config (Okapi):**
```json
{
  "ams.banner.throttles.budgetThresholdHit": {
    "dimensions": [
      { "equalTo": "*" },                                           // brand
      { "oneOf": ["international", "pp_international", "sbg_international"] },  // jurisdiction
      { "oneOf": ["GB", "IE"] },                                    // country
      { "oneOf": ["MY_ACCOUNT_SUMMARY", "MY_ACCOUNT", "TBD"] }     // flow
    ],
    "value": "{\"dateBetween\": {\"min\": \"1\",\"max\": \"20\"},\"percentBetween\": {\"min\": \"50\",\"max\": \"90\"}}"
  }
}
```

**Throttle Conditions:**
| Condition | Values | Effect |
|-----------|--------|--------|
| `dateBetween.min/max` | 1-20 | Only show banner on days 1-20 of the month |
| `percentBetween.min/max` | 50-90 | Only show when usage is 50%-90% of limit |

### 2.3 BudgetThresholdHitBannerService

**BudgetThresholdHitBannerService.java (AMS):**
```java
@Component
public class BudgetThresholdHitBannerService implements BannerTypeService {

    private final List<String> AFFORDABILITY_FLAGS = ImmutableList.of(
            BESPOKE_NDL_SET,
            CDD_ACCOUNT_LIMITED, 
            CDD_INVESTIGATION_IN_PROGRESS, 
            CDD_INVESTIGATION_COMPLETE);

    @Override
    public List<BannerDetails> getBanners(...) {
        // 1. Check user has affordability flags
        if (!isAffordabilityFlow(customerInfo)) return empty;

        // 2. Check throttle exists and day is within range (e.g., 1-20)
        if (!isInThrottledPeriod(bannerThrottle)) return empty;

        // 3. Call MSB for budget status
        var budgetStatus = spendBudgetService.getBudgetStatus(brand, sessionToken);

        // 4. Check percentage is within range (e.g., 50-90%)
        if (!isInThrottledPercent(bannerThrottle, budgetStatus)) return empty;

        // 5. Return banner with interpolated values
        return buildBanner(budgetStatus, bannerThrottle);
    }
}
```

### 2.4 AMS → MSB Integration

**SpendBudgetService.java (AMS):**
```java
@Service
public class SpendBudgetService {

    @Value("$AMS_CLIENT{msb.url}")
    private String msbBaseUrl;  // e.g., https://msb-{brand}.example.com/budget-status

    public SpendBudgetInfo getBudgetStatus(String brand, String sessionToken) {
        var msbUrl = msbBaseUrl.replace("{brand}", brandMapping.get(brand));
        var response = restTemplate.exchange(
            msbUrl,
            HttpMethod.GET,
            new HttpEntity<>(buildHeaders(sessionToken)),  // x-api-key, ssoid cookie
            String.class
        );
        return mapPrimaryBudget(response);  // extracts primaryBudget.account[0]
    }

    // Brand mappings: PADDYPOWER→pp, BETFAIR→bf, SKYBET→sbg, POKERSTARS→ps
}
```

**Budget Percent Calculation:**
```java
private double calculateBudgetPercent(double remaining, double total) {
    if (remaining == 0) return 100d;  // Fully used
    return ((total - remaining) / total) * 100;
}
```

### 2.5 AMS Circuit Breaker

AMS calls are wrapped in circuit breaker (same config as MAX→MSB - see section 1.3).

---

## 4. MSB Backend (Data Provider)

> **Role:** MSB provides budget data to both MAX (for Deposit Limit Widget) and AMS (for Spend Budget Alert). MSB itself has limited controls.

### 3.1 What MSB Controls

| Control | Affects | Description |
|---------|---------|-------------|
| Platform Detection | Widget display | Native apps identified via User-Agent/cookies |
| Cache TTL | Both features | 10-600 second cache for PLS responses |
| Banner Expiry | Widget banners | 3-day auto-expire for MSB page-context banners |

### 3.2 Native App Detection

MSB detects native apps to set platform context:

```java
// CustomDeviceResolver.java - checks User-Agent + cookies
if (userAgent.contains("BetfairWrapper") || cookie.contains("IOS_BFCasino")) {
    device.setNative(true);
}
```

**Native App Whitelists:**
| Brand | User-Agents | Cookie Prefixes |
|-------|-------------|-----------------|
| Betfair | `BetfairWrapper, SMX, EMS` | `IOS_BF*, Android_BF*` |
| Paddy Power | `PPWrapper, SMS` | `IOS_PP*, Android_PP*` |
| Skybet | `SkybetWrapper` | - |

### 3.3 ⚠️ No Rate Limiting

**MSB has NO explicit rate limiting.** Only implicit controls:

| Control | Mechanism |
|---------|-----------|
| Cache TTL | 10-600 second cache durations |
| Connection Pool | Apache HttpClient max connections |
| Thread Pool | Spring Boot default limits |

**Risk:** DoS vulnerability - recommend adding Resilience4j rate limiter.

---

## 5. Native (TBD) Controls

### 4.1 GraphQL Catalogue

**Feature controlled by catalogue card availability:**
- `BudgetLimitsCard` → Deposit Limit Widget
- `AccountBannersCard` → Spend Budget Alert

If cards not returned in GraphQL response → features not shown.

### 4.2 Selector Filtering

**PDL Category Filtered Out:**
```typescript
// budget-limits-selectors.ts
if (limit.category === BudgetCategory.Pdl) {
    return acc;  // Skip PDL
}
```

### 4.3 Banner Type Filtering

**Regulatory Banners Filtered:**
```typescript
// account-banners-card-normalizer.ts
// Filters out bannerType === "rg" (regulatory)
```

---

## 6. Control Matrix by Feature

### Deposit Limit Widget

| Layer | Control | Effect |
|-------|---------|--------|
| **MAX** | `affordability` throttle | Legacy: enables widget |
| **MAX** | `HVNA-593_unified...` throttle | New: enables widget (overrides legacy) |
| **MAX** | `AFFORDABILITY_FLAGS` | Legacy: required user flags |
| **MAX** | Circuit Breaker | Disables on MSB failures |
| **MSB** | Platform detection | Sets native context |
| **PLS** | Account limits | No limits = no widget |
| **TBD** | `BudgetLimitsCard` presence | Catalogue must return card |
| **TBD** | PDL filter | Hides PDL category limits |

### Spend Budget Alert

| Layer | Control | Effect |
|-------|---------|--------|
| **AMS** | `ams.banner.throttles.budgetThresholdHit` | Master throttle for banner |
| **AMS** | `dateBetween` throttle | Only show on days 1-20 of month |
| **AMS** | `percentBetween` throttle | Only show when usage 50-90% |
| **AMS** | `AFFORDABILITY_FLAGS` | Required user flags (BESPOKE_NDL_SET, CDD_*) |
| **AMS** | Circuit Breaker | Disables on AMS failures |
| **MSB** | Budget status API | Provides current % usage to AMS |
| **PLS** | Account limits | Source of truth for limit data |
| **TBD** | `AccountBannersCard` presence | Catalogue must return card |
| **TBD** | Banner type filter | Filters regulatory banners |

---

## 7. How to Disable Features

### Disable Deposit Limit Widget

| Method | Layer | Action |
|--------|-------|--------|
| **Throttle** | MAX | Turn OFF both `affordability` and `HVNA-593_unified...` |
| **User Flag** | MAX | Remove all `AFFORDABILITY_FLAGS` from user (legacy only) |
| **Circuit Open** | MAX | Circuit breaker in OPEN state (automatic on failures) |
| **No Limits** | PLS | User has no active deposit limits |
| **Catalogue** | TBD | Don't include `BudgetLimitsCard` in catalogue |

### Disable Spend Budget Alert

| Method | Layer | Action |
|--------|-------|--------|
| **Throttle** | AMS | Remove/disable `ams.banner.throttles.budgetThresholdHit` from Okapi |
| **Day Range** | AMS | Set `dateBetween` to impossible range (e.g., 32-32) |
| **Percent Range** | AMS | Set `percentBetween` to 0-0 (never matches) |
| **User Flags** | AMS | Ensure user has no `AFFORDABILITY_FLAGS` (BESPOKE_NDL_SET, CDD_*) |
| **Circuit Open** | AMS | Circuit breaker in OPEN state |
| **Catalogue** | TBD | Don't include `AccountBannersCard` in catalogue |

---

## 8. Monitoring & Debugging

### Circuit Breaker State

**JMX Metrics:**
- `circuitbreaker.state` - CLOSED, OPEN, HALF_OPEN
- `circuitbreaker.failure.rate` - Current failure rate %
- `circuitbreaker.slow.call.rate` - Current slow call rate %

### Throttle Check

**Frontend (MAX):**
```javascript
platformConfig.getActiveThrottles().includes('affordability')
platformConfig.getActiveThrottles().includes('HVNA-593_unified_deposit_limits_widget_max')
```

### User Flag Check

**Backend (MAX):**
```java
userContext.getAccountContext().getActiveFlags()
    .stream()
    .map(Flag::getCode)
    .filter(AFFORDABILITY_FLAGS::contains)
    .collect(Collectors.toList());
```
