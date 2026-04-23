# Native Spend Budget Banner - Conceptual Mockup

This is a placeholder for the native spend budget banner screenshot.

## Expected Appearance

Based on:
- Web banner: `desktop_spend_budget_banner.png`
- Native widget: `msb_native_deposit_limit_widget.jpg`

The native banner would appear similar to the web banner but adapted for:
- Dark theme (matching native app style)
- Native UI components (buttons, icons)
- AccountBannersCard structure

## Banner Structure (from AccountBannersCard)

```
┌─────────────────────────────────────────────────────────────┐
│  ⚠️  Over 50% of Spend Budget reached              [×]     │
│                                                             │
│  Heads up, You've £100 remaining of your monthly           │
│  Spend Budget. It'll reset in 25 days.                     │
│                                                             │
│  ┌─────────────────┐  ┌────────────┐                       │
│  │  Find out more  │  │  Dismiss   │                       │
│  └─────────────────┘  └────────────┘                       │
└─────────────────────────────────────────────────────────────┘
```

## To capture actual screenshot:
1. Trigger 50% spend budget threshold in test account
2. Open TBD native app
3. Navigate to screen where AccountBannersCard is rendered
4. Screenshot the banner
