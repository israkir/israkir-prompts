---
id: timestamp-data-source-audit
title: Timestamp & Data-Source UX Consistency Audit
category: ux
tags: [timestamp, provenance, audit, ux, consistency, data-source]
version: 1.0.0
---

# Prompt: Timestamp & Data-Source UX Consistency Audit

> **How to use:** Paste this document (or link it) as the full task brief for an LLM coding agent.  
> **Goal:** Find every user-visible timestamp, make timing semantics consistent, always pair time with source, respect user timezone, prioritize page-level provenance, and remove redundancy.

---

## Role

You are a senior product engineer performing a **timestamp and data-provenance UX audit**. You understand:

- The difference between **provider time**, **server/cache time**, **client receive time**, and **user activity time**
- Financial, IoT, analytics, and SaaS patterns for “as of”, “last synced”, and “live” labels
- Accessible, low-clutter UI for freshness indicators

Adapt terminology and placement to the product domain. Do not assume a specific framework unless the repo clearly uses one.

---

## Scope

### In scope

- All **user-visible** timestamps in UI (web, mobile, embedded widgets)
- Labels that imply freshness without a date (“Live”, “Real-time”, “Cached”, “Updated”)
- Data source attribution shown near or implied by timestamps
- Tooltips, popovers, aria-labels, and screen-reader text tied to time display

### Out of scope (unless they affect visible labels)

- Internal logs, metrics, debug panels not shipped to users
- Backend ingest/scheduling (only touch if UI lacks fields needed for honest labeling)

---

## Core principles (must satisfy all five)

### 1. Timestamp *kind* must be unambiguous

Every visible timestamp must answer: **“What moment does this refer to?”**

Use a consistent vocabulary across the product:

| Kind | Typical labels | Meaning |
|------|------------------|---------|
| **Data refresh** | “As of”, “Refreshed”, “Snapshot” | When upstream data was last fetched, computed, or published by the provider |
| **Server sync** | “Synced”, “Fetched” | When *your* backend/API last served or refreshed the payload (may differ from provider time) |
| **Live stream** | “Live”, “Streaming” | Real-time or near-real-time updates during an active session; show last tick in tooltip |
| **Domain event** | “Scheduled”, “Due”, “Occurred” | Business/calendar/event time (often domain-specific timezone, e.g. market hours) |
| **User activity** | “Sent”, “Edited”, “Created” | User or system actions unrelated to external data freshness |

**Rules:**

- Never show a bare relative time (“7m ago”) without stating *what* is 7 minutes old.
- Do not mix kinds in one label (e.g. “Live · 2h ago” without explaining live *what* vs stale *what*).
- When two kinds differ materially, show both in tooltip (e.g. “Provider snapshot: … · Served: …”).

---

### 2. Timestamp + source — always paired

**No orphan timestamps for data-backed UI.**

Every timestamp that reflects external or computed data must appear with **source attribution**:

- Provider name (e.g. “Acme API”, “OpenWeather”, “Stripe”, “Internal analytics”)
- Optional freshness/status indicator (`fresh` | `stale` | `unavailable` | `unknown`)
- Human-readable age where appropriate
- Detail on interaction (tooltip/popover) with exact datetime(s)

**Rules:**

- If source is unknown, show `"Source unknown"` or hide the timestamp until source is available — do not show time alone.
- User-activity timestamps (chat sent time, “edited 5m ago”) do **not** require an external data source.
- Prefer **one shared component** over ad-hoc `<span>Updated …</span>` patterns.

---

### 3. Timezone and formatting

**Display (default):** user’s local timezone via locale-aware formatting (`Intl`, platform date APIs, or established project utilities).

**Human-readable regimes (recommended defaults — align with product tone):**

| Age | Display |
|-----|---------|
| &lt; 60 seconds | “just now” |
| &lt; 1 hour | “Xm ago” |
| same local calendar day | time only (“3:42 PM”) |
| older | date + time (“Apr 15, 3:42 PM”) |

**Tooltip / popover (on hover, focus, or expand):**

- Full localized datetime + timezone abbreviation
- ISO 8601 or RFC-style exact string if useful for power users
- When domain requires it (markets, broadcasts, flights, SLAs): **second line in domain timezone** (e.g. “Market time (America/New_York): …”)

**Rules:**

- One formatter family project-wide — avoid mixing `toLocaleString()`, custom parsers, and library X in different screens.
- Relative labels should refresh on a sensible interval (e.g. 30–60s), not every second unless truly live.
- Past vs future: use clear wording (“in 2h” vs “2h ago”) and test edge cases around midnight and DST.

---

### 4. Placement hierarchy — page vs component

| Level | Use when | Placement |
|-------|----------|-----------|
| **Page / view-wide** | Single dataset, shared cache age, or one primary backend response | Header, toolbar, footer strip, or global status slot — **one primary chip** |
| **Section / component** | Different source, different refresh cadence, or materially different age | Inline on that section only |
| **Row / item** | Per-row provider or event time that differs from section default | Table cell, list meta, card footer |
| **Suppress** | Same source + same instant as a parent-level indicator | Remove duplicate |

**Priority:** Prefer **page-wide** provenance. Add lower-level timestamps only when age or source **materially differs** from the page default.

Document every exception with a one-line rationale.

---

### 5. Remove redundancy

Eliminate duplicate or conflicting:

- Timestamps (page chip + card header + sticky bar for the same payload)
- Source labels (icon + badge + footer all saying the same provider)
- Conflicting freshness signals (“Live” next to “Cached 1h ago” for the same field)

**Before removing:** confirm the duplicate truly refers to the same `(source, instant, kind)` triple.

---

## Phase A — Discovery (inventory first, code second)

### Step 1: Locate the codebase UI layer

Identify where UI lives (examples: `src/`, `app/`, `components/`, `pages/`, `features/`). Note framework (React, Vue, SwiftUI, etc.) and existing date utilities.

### Step 2: Search patterns

Run searches appropriate to the stack. Examples:

```text
# Relative / formatted time
formatDate|formatTime|timeAgo|relativeTime|fromNow|moment\(|dayjs|date-fns|Intl\.DateTimeFormat|toLocaleString

# Freshness & provenance
asOf|as_of|lastUpdated|lastSync|updatedAt|createdAt|generatedAt|refreshed|cached|live|stale|provenance|dataSource|provider

# UI copy
"As of"|"Last updated"|"Updated"|"Synced"|"Refreshed"|"Live"
```

### Step 3: Build an inventory table

For each finding, record:

| Location | Current label/copy | Kind (see §1) | Source shown? | Timezone handling | Redundant with? | Issue |
|----------|---------------------|---------------|---------------|-------------------|-----------------|-------|

Group rows by page/view. Flag orphan timestamps, mixed kinds, and duplicates.

### Step 4: Map existing shared utilities

Note any shared date/format components, hooks, or design-system tokens. Prefer extending these over adding parallel implementations.

---

## Phase B — Plan & implement

### Step 1: Propose a target model

Before editing, summarize:

- Canonical timestamp kinds and labels for this product
- Shared component(s) or utilities to centralize formatting and pairing
- Page-level vs section-level placement decisions per major view

### Step 2: Implement in priority order

1. Shared formatter / `TimestampWithSource` (or equivalent) if missing
2. Page-level provenance where a single backend response backs the view
3. Section- and row-level fixes where source or age differs materially
4. Redundancy removal and accessibility (tooltips, `aria-label`, focus)

Match existing code style. Do not refactor unrelated areas.

### Step 3: Backend gaps

If UI cannot show honest labels because API responses lack `providerUpdatedAt`, `servedAt`, or `source` metadata, list minimal API/schema changes needed — implement only when in scope and data is available.

---

## Phase C — Verify

Deliver a short summary with:

- Inventory of changes (file paths and what changed)
- Remaining exceptions with one-line rationale
- Manual test checklist: timezone edge cases, relative refresh, screen reader labels, duplicate suppression
- Any follow-ups blocked on backend or design

Do not mark complete while orphan data timestamps or conflicting “Live”/“Cached” pairs remain for the same payload.
