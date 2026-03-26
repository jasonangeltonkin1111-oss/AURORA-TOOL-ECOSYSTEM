# Strategy Builder Blueprint

## 1. Tool identity
- **Name:** Strategy Builder
- **Suite:** Education Suite
- **Public tier vs private tier:** Public = Template storage; Private = System integration
- **Status (Idea/Planned/In Build/Released):** Planned

## 2. User-facing definition
- **Problem statement:** Build templates with predictable behavior that reduces manual errors.
- **Target user:** New users configuring AURORA tools.
- **Primary use cases:** Use Strategy Builder to onboard users and codify repeatable operating templates.
- **Non-goals:** No hidden execution intelligence, no autonomous strategy generation, and no cross-account optimization in public mode.

## 3. Functional specification
- **Inputs:** Manual settings, account metrics (balance/equity/P&L), broker symbol constraints, and market/session context when available.
- **Core logic rules:** Deterministic rule evaluation; public logic follows **Template storage** behavior, while private roadmap variant extends to **System integration**.
- **Outputs/actions:** Panel state updates, actionable controls, guard/confirmation events, and structured logs (`tool_action_submitted`, `tool_action_blocked`, `tool_action_applied`).
- **Config parameters:** Risk/profile presets, per-symbol overrides, alert severity, and persistence scope (session-only vs saved profile).
- **Edge cases:** Missing broker metadata, stale quotes, latency timeouts, min-lot/step rounding, and conflicting rules (strictest rule wins).

## 4. UX specification
- **Panel layout:** Header (status + symbol/account), central control group for build templates, footer with recent actions/log snippets.
- **Controls:** Enable toggle, numeric inputs/sliders, preset dropdowns, confirm/apply button, and reset-to-default profile action.
- **Alerts/messages:** Inline warnings for soft limits, modal block for hard violations, and success toast with applied values.
- **Error states:** Read-only fallback on API failure, explicit retry CTA, and lock badge when required inputs are invalid.

## 5. Technical specification
- **Dependencies/shared modules:** Risk engine, rule engine, session service, alert bus, telemetry client, and profile manager.
- **Data model/state:** Local UI state + normalized snapshot (`account`, `symbol`, `tool_config`, `rule_results`, `last_action`).
- **Integration points:** Broker API adapter, shared event bus, suite dashboard widgets, and journaling/export hooks.
- **Performance constraints:** Local rule decision target <200 ms; non-blocking UI under quote/update bursts; idempotent action replay after reconnect.

## 6. Compliance/safety
- **Prop constraints handled:** Enforce daily loss, max exposure, lot precision, and execution locks where applicable before action confirmation.
- **Risk controls:** Hard-stop blockers for violations, soft-warning thresholds, deterministic fallbacks, and complete audit logs.
- **Session/news/time restrictions:** No additional temporal restrictions beyond global platform guardrails.

## 7. QA checklist
- **Scenario tests:** Valid action path, blocked path, degraded data path, reconnect path, and rule-conflict path.
- **Regression checks:** Settings persistence, symbol/account switch behavior, telemetry schema validity, and UI lock/unlock transitions.
- **Manual verification steps:** Configure profile, simulate account/market state changes, trigger alerts/blocks, confirm log payloads, and verify recovery after disconnection.

## 8. Release metadata
- **Version:** 0.1.0-strategy-builder
- **Changelog notes:** Initial blueprint populated from roadmap table with explicit public/private behavior and implementation constraints.
- **Monetization tier (single/bundle/suite):** Bundle
