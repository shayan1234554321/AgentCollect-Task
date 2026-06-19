# UX Bug Detection from PostHog Sessions

## Goal
Identify UX bugs on payment and dashboard pages using PostHog session data.

---

## Phase 1: Establish Baseline (Expected Behavior)

Before detecting bugs, define what "correct" looks like:

### 1.1 Payment Page Flow
- User navigates to payment page
- User fills payment form fields
- User clicks submit
- Success confirmation OR error message displayed
- Response time: target < 3s for form load, < 5s for submission

### 1.2 Dashboard Page Flow
- User navigates to dashboard
- Dashboard loads with data/widgets
- User interacts with filters/controls
- Response time: target < 2s for initial load

### 1.3 Expected PostHog Events
| Event Name | Trigger | Expected Frequency |
|------------|---------|-------------------|
| `page_viewed` | Page load | Once per session |
| `form_started` | First field interaction | Once per form |
| `form_field_changed` | Field input | Per field touched |
| `button_clicked` | CTA clicks | Per click |
| `payment_submitted` | Submit click | Once per attempt |
| `payment_success` | Successful payment | Once per success |
| `payment_error` | Failed payment | Per failure |
| `api_request` | Network call | Per request |
| `api_response` | Network response | Per response |

### 1.4 Expected Network Patterns
| Pattern | Indicator | Acceptable Threshold |
|---------|-----------|---------------------|
| Success | 2xx response | < 3s latency |
| Redirect | 3xx response | < 1s |
| Client error | 4xx response | Validation errors only |
| Server error | 5xx response | 0 occurrences |
| Timeout | Request > 30s | < 1% of requests |

### 1.5 DOM Stability
- No layout shift (CLS < 0.1) during user interaction
- No elements disappearing during fill
- Autocomplete/autofill should not break fields

---

## Phase 2: Collect Signals from PostHog

Query PostHog for session recordings and events:

### 2.1 Event Collection Query
```
filters:
  - event in ['click', 'change', 'pageview', 'api_request', 'api_response']
  - page_url contains '/payment' OR '/dashboard'
  - timestamp within last 7 days
```

### 2.2 Key Signals to Extract
| Signal | PostHog Field | What It Reveals |
|--------|---------------|-----------------|
| Click coordinates | `pos_x`, `pos_y` | Dead clicks (no interactive element) |
| Click velocity | `time_since_last_click` | Rage clicks (< 300ms intervals) |
| Click target | `target_element` | Which element is being clicked |
| Network failures | `$network_request` with failure status | 500 errors, timeouts |
| Form abandonment | `form_started` without `form_completed` | Drop-off points |
| Session duration | `session_duration` | Abnormal exits |
| DOM mutations | `$dom_changed` events | Unexpected UI changes |

---

## Phase 3: Detect UX Issues

### 3.1 Dead Clicks
**Definition:** Click on non-interactive element or element with `pointer-events: none`
**Detection:**
- Click events on `<div>`, `<span>`, `<p>` without click handlers
- Click on elements with `cursor: default`
- Click coordinate lands on empty space

### 3.2 Rage Clicks
**Definition:** Rapid repeated clicks on same element (> 3 clicks in < 1s)
**Detection:**
- Same `target_element` clicked 4+ times
- Time between clicks < 300ms
- Indicates: button not responding, loading state missing

### 3.3 Form/Cart Abandonment
**Detection:**
- `form_started` event exists
- No `form_completed` or `payment_success` within same session
- Last event is `click` on a different page or `page_hidden`

### 3.4 Network 500 Errors
**Detection:**
- `api_response` with `status_code >= 500`
- Associated with user action (click → request → 500)
- Error not gracefully handled (no error message displayed)

### 3.5 Additional Issues to Flag
- **Empty state clicks:** User clicked expecting content, got none
- **Redirect loops:** Multiple page views of same URL in short span
- **Stuck loading:** Click triggered, but no response in > 10s

---

## Phase 4: Scoring System

Apply scores to prioritize issues:

### 4.1 Base Scores
| Factor | Points | Rationale |
|--------|--------|-----------|
| Payment page | +2 | Higher business impact |
| Dashboard page | +0 | Lower urgency |
| 500 error | +3 | Direct failure |
| Dead click | +1 | Annoyance, not blocking |
| Rage click | +2 | Indicates blocking issue |

### 4.2 Repetition Multiplier
| Occurrence | Points | Rationale |
|------------|--------|-----------|
| Single occurrence | +0 | Could be edge case |
| 2-5 occurrences | +1 | Pattern emerging |
| 6+ occurrences | +3 | Systematic issue |

### 4.3 Priority Tiers
| Score | Priority | Action |
|-------|----------|--------|
| 8+ | P0 - Critical | Fix within 24h |
| 5-7 | P1 - High | Fix within 1 week |
| 2-4 | P2 - Medium | Fix within sprint |
| 0-1 | P3 - Low | Backlog |

---

## Phase 5: Output

### 5.1 Bug Report Template
```
## UX Bug: [Title]

**Location:** [Payment/Dashboard] → [Page/Component]
**Detected:** [Date range]
**Occurrences:** [Count]

**Evidence:**
- [Session recording link]
- [Screenshots of dead click / error state]

**Signal Summary:**
- Click count: X
- Average click velocity: Xms
- Network errors: X

**Score:** [Total] (P[Priority])

**Recommended Fix:**
[Description]
```

### 5.2 Weekly Summary
- Total bugs found
- Breakdown by page and type
- Trend vs previous week
- Top 3 priorities for sprint

---

## Questions to Resolve

1. **Normal network logs:** What is the typical 2xx/3xx/4xx ratio? (baseline needed)
2. **Expected flows:** Are there alternative payment paths (Guest checkout vs logged-in)?
3. **PostHog setup:** Which events are currently being captured? (need to verify instrumentation)
4. **Session sampling:** What % of sessions are recorded? (affects confidence in findings)