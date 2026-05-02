# DriverGuard AI — Feature Development Outline
**Version 2.0 Roadmap | Single-file PWA (index.html)**

> This document is the authoritative development reference. Each feature includes its goal, technical approach, data contracts, UI notes, and integration points with the existing codebase.

---

## Architecture Reminder

All features live inside `index.html`. State is persisted via `localStorage`. The app uses:
- **Claude API** (`claude-haiku-4-5`) for AI coaching
- **Web APIs**: `DeviceMotionEvent`, `Geolocation`, `Web Speech API`
- **EmailJS** for parent alerts
- **NHTSA API** for vehicle safety data

New features must not break the existing setup wizard, tuning panel, PIN lock, or incident log.

---

## PHASE 1 — Quick Wins

---

### Feature 1.1 — Trip Summary Cards

**Goal:** At session end (user taps "End Trip"), Claude auto-generates a structured summary card stored in the incident log and optionally emailed to parent.

#### Data Collected During Session
```
session = {
  startTime, endTime, durationMinutes,
  avgSpeedMPH, maxSpeedMPH,
  avgGForce, peakGForce,
  riskScoreTimeline: [...],   // array of {timestamp, score}
  avgRiskScore, peakRiskScore,
  incidentCount,
  incidentLog: [...],          // existing incident objects
  vehicleName,
  passengerCount              // added in Phase 2
}
```

#### Claude Prompt Contract
```
System: You are a driving safety coach. Be concise, specific, and encouraging.
        Respond with a JSON object only — no markdown.

User: Analyze this trip and return:
{
  "grade": "A/B/C/D/F",
  "headline": "one sentence verdict",
  "avgSpeed": "summary phrase",
  "gForceSummary": "summary phrase",
  "riskTrend": "improving | stable | worsening",
  "coachingTip": "one actionable tip under 20 words",
  "parentNote": "one sentence for parent report"
}
Trip data: [JSON.stringify(session)]
```

#### UI
- Card appears in incident log at top after trip ends
- Card has colored grade badge (A=green, B=teal, C=amber, D/F=red)
- "Share with Parent" button triggers EmailJS with `parentNote`
- Cards are exportable in the existing CSV export

#### Storage Key
`dg_trip_summaries` → array of summary objects (max 50, FIFO)

#### Integration Points
- Hook into existing "End Trip" / session-stop logic
- Reuse existing EmailJS `sendAlert()` function
- Append to existing `incidentLog` array with `type: "trip_summary"`

---

### Feature 1.2 — Drowsiness Detection

**Goal:** Detect driver fatigue via G-force jitter patterns. Flag when micro-corrections exceed a threshold over a rolling window.

#### Detection Algorithm
```
ROLLING WINDOW: 30 seconds of lateral G samples (sampled at ~10Hz = 300 points)

JITTER SCORE = StdDev(lateralG_window) × frequency_of_direction_changes

THRESHOLDS:
  jitterScore > 0.08  → DROWSY_WARNING  (amber alert)
  jitterScore > 0.14  → DROWSY_CRITICAL (red alert, voice alert)

COOLDOWN: 2 minutes between alerts to avoid alarm fatigue
RESET: Jitter score resets if speed drops below 15 MPH (stopped/slow)
```

#### Data Structure
```javascript
drowsinessState = {
  active: false,
  jitterScore: 0.0,
  samplesWindow: [],        // circular buffer, 300 max
  lastAlertTime: null,
  alertLevel: null,         // null | 'warning' | 'critical'
  sessionDrowsyEvents: []   // [{timestamp, jitterScore, speed}]
}
```

#### UI
- New drowsiness indicator icon in the HUD (eye icon, green/amber/red)
- Drowsy events logged in incident log with `type: "drowsiness"`
- Included in trip summary card (Feature 1.1) as `drowsyEventCount`
- Settings in Tune panel: enable/disable, threshold adjustments

#### Voice Alert (integrates with Feature 2.3)
- Warning: *"Stay alert. Signs of fatigue detected."*
- Critical: *"Pull over safely when possible. Fatigue detected."*

#### Integration Points
- Add to existing `DeviceMotionEvent` handler alongside G-force calculation
- Add drowsiness score to composite risk score (weight: configurable, default 15%)
- Add to parent alert email template as `drowsy_events`

---

### Feature 1.3 — Speed Limit Overlay

**Goal:** Fetch the posted speed limit for the driver's current road using OpenStreetMap's Overpass API. Warn when exceeding it.

#### API Strategy
```
Provider: Overpass API (free, no key required)
  https://overpass-api.de/api/interpreter

Query: Find nearest highway with maxspeed tag within 30m of GPS coords
  [out:json];
  way(around:30,{lat},{lon})[highway][maxspeed];
  out;

Fallback: Nominatim reverse geocode to get road class, estimate limit:
  residential → 25 MPH
  secondary → 35 MPH
  primary / trunk → 55 MPH
  motorway → 65 MPH

Cache: Store last fetched limit by road segment (rounded lat/lon to 4dp)
  key: dg_speedlimit_{lat4}_{lon4}
  TTL: 30 days (roads don't change often)

Poll Rate: Fetch new limit when GPS moves >100m from last fetch point
```

#### Violation Logic
```
OVER_LIMIT_BUFFER: +5 MPH grace (configurable in Tune panel)
ALERT_LEVELS:
  speed > limit + buffer      → visual warning (amber)
  speed > limit + buffer + 10 → visual + voice warning (red)
  speed > limit + buffer + 20 → parent alert trigger added to composite
```

#### UI
- Speed limit badge displayed next to current speed in HUD
- Badge turns amber/red when exceeding limit
- "Speed Limit Unknown" state shown as "--" badge
- Tune panel: toggle feature on/off, set MPH buffer

#### Data Added to Incident Log
```
incident.speedLimit = 45    // posted limit at time of incident
incident.overLimitBy = 12   // MPH over limit
```

#### Integration Points
- Hook into existing GPS position update loop
- Add speed limit violation to composite risk score (weight: configurable, default 20%)
- Add `speed_limit` and `over_limit_by` to EmailJS template

---

## PHASE 2 — Medium Effort

---

### Feature 2.1 — Heatmap Route Replay

**Goal:** After a trip, show the driven route on a map with segments color-coded by risk score (green → yellow → red).

#### Libraries
```
Leaflet.js (via CDN, ~42KB gzip) — free, no API key required
  https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.min.js
  https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.min.css

Tile provider: OpenStreetMap (free, no key)
```

#### Data Collection (runs during trip)
```javascript
routeTrack = [
  { lat, lon, timestamp, speedMPH, riskScore, gForce },
  ...  // recorded every 2 seconds while moving (speed > 5 MPH)
]
// Stored in sessionStorage during trip, moved to localStorage on trip end
// Storage key: dg_route_{tripId}
// Max 1800 points per trip (60 min at 2s intervals)
// Keep last 10 trips. FIFO eviction.
```

#### Rendering Logic
```
Split routeTrack into segments between consecutive points.
Color each segment by riskScore:
  score 0–30   → #22c55e (green)
  score 30–55  → #eab308 (amber)
  score 55–75  → #f97316 (orange)
  score 75–100 → #ef4444 (red)

Polyline weight: 4px
Click on segment → popup shows: speed, risk score, G-force at that point
```

#### UI
- "View Route" button on trip summary card opens map modal
- Map fills 90% viewport, close button top-right
- Trip selector dropdown to replay past 10 trips
- Legend bar at bottom (green → red scale)
- Incident markers on map (triangle icon) at incident GPS coords

#### Integration Points
- Add GPS tracking array (`routeTrack`) to session state
- Add "View Route" action to trip summary card (Feature 1.1)
- Route data included in CSV export as separate sheet/section

---

### Feature 2.2 — Teen Driver Score Card (Weekly Report)

**Goal:** Generate a weekly PDF/HTML report summarizing driving trends, suitable for parents.

#### Report Contents
```
HEADER
  Driver name, vehicle, reporting period (Mon–Sun)

SUMMARY METRICS (pulled from last 7 days of trip summaries)
  Total trips, total miles, total drive time
  Average trip grade, best grade, worst grade
  Average risk score trend (chart: 7 bars, one per day)

HIGHLIGHT STATS
  Highest speed recorded
  Peak G-force recorded
  Drowsy events count (Feature 1.2)
  Speed limit violations count (Feature 1.3)

TOP INCIDENTS TABLE
  Date | Time | Type | Risk Score | Speed | Details

AI COACHING SUMMARY
  Claude prompt: Summarize this week's driving in 3 bullet points.
  Frame positively. Identify the #1 area to improve.

TREND COMPARISON
  vs. prior week (if data available): better/worse/same per metric
```

#### Generation Options
1. **In-browser HTML** — open in new tab, print to PDF via browser
2. **EmailJS delivery** — send HTML email to parent on Sunday 8 PM (configurable)

#### Data Sources
```
Pull from: dg_trip_summaries (Feature 1.1)
Filter: last 7 completed trip summaries
Aggregate: calculate weekly stats from individual trip data
```

#### UI
- "Weekly Report" button in Tune panel (parent section)
- Manual "Generate Now" + auto-schedule toggle
- Preview renders in-app before sending

#### Storage Key
`dg_weekly_reports` → array of last 12 weekly report objects

---

### Feature 2.3 — Voice Coaching Mode

**Goal:** Use Web Speech API to speak Claude's driving feedback aloud, keeping the driver's eyes on the road.

#### Speech Output (Text-to-Speech)
```javascript
// Check support
const ttsSupported = 'speechSynthesis' in window;

// Speak function
function speak(text, priority = 'normal') {
  if (!settings.voiceCoaching) return;
  const u = new SpeechSynthesisUtterance(text);
  u.rate = 1.0;      // configurable: 0.8–1.3
  u.pitch = 1.0;
  u.volume = 1.0;
  // For urgent alerts, cancel current speech first
  if (priority === 'urgent') speechSynthesis.cancel();
  speechSynthesis.speak(u);
}
```

#### What Gets Spoken
```
PRIORITY LEVELS:
  urgent  → drowsiness critical, risk score > threshold, speed limit critical
  normal  → Claude AI coach response, trip summary grade announcement
  low     → speed limit warnings, routine status updates

NEVER SPEAK:
  - Raw numbers (say "your speed is high" not "87 miles per hour")
  - More than 15 words in a single utterance
  - When car is stopped (speed < 5 MPH) unless it's a summary
```

#### Voice Selection
```javascript
// Prefer a female en-US voice (research shows higher acceptance for safety apps)
function getBestVoice() {
  const voices = speechSynthesis.getVoices();
  return voices.find(v => v.lang === 'en-US' && v.name.includes('Female'))
      || voices.find(v => v.lang === 'en-US')
      || voices[0];
}
```

#### Optional: Voice Input (Speech-to-Text)
```javascript
// Allow hands-free chat with AI coach
const recognition = new (window.SpeechRecognition || window.webkitSpeechRecognition)();
recognition.continuous = false;
recognition.lang = 'en-US';
// Triggered by: shake gesture OR tap mic button in chat
```

#### UI
- Voice toggle in HUD (microphone icon)
- Voice settings in Tune panel: rate, volume, voice selection, enable STT
- Visual "speaking" indicator in HUD when TTS is active

#### Integration Points
- Wrap existing Claude AI coach `sendMessage()` output through `speak()`
- Add `speak()` calls to drowsiness alerts (Feature 1.2)
- Add `speak()` calls to speed limit violations (Feature 1.3)
- Announce trip grade at end of trip (Feature 1.1): *"Trip complete. Your grade is B."*

---

### Feature 2.4 — Passenger Count Awareness

**Goal:** Adjust risk thresholds based on passenger count. More passengers = statistically higher risk for teen drivers.

#### Risk Multiplier Table
```
Passengers  | Risk Multiplier | Threshold Tightening
------------|-----------------|---------------------
0 (solo)    | 1.0×            | baseline
1           | 1.1×            | +5 to all alert thresholds
2           | 1.2×            | +10 to all alert thresholds
3           | 1.35×           | +15, voice coaching more frequent
4+          | 1.5×            | +20, parent alert threshold lowered -10
```

#### Source
CDC data: Teen crash risk doubles with 1 passenger, quadruples with 2+. Risk multipliers are based on relative risk ratios from CDC teen driver research.

#### UI
- Passenger count selector in HUD (0–5+, large touch targets)
- Count selector appears during trip setup / wizard step 5
- Current passenger count shown as small badge on HUD
- Tune panel: toggle feature, customize multipliers

#### Data Added to Trip Summary
```
tripSummary.passengerCount = 2
tripSummary.riskMultiplierApplied = 1.2
```

#### Integration Points
- Modify composite risk score calculation: `adjustedScore = rawScore × multiplier`
- Add passenger count to EmailJS parent alert template
- Include in weekly score card (Feature 2.2)
- Pass to Claude AI coach context: *"There are 2 passengers in the vehicle."*

---

## Cross-Feature Integration Map

```
Feature 1.1 (Trip Summary)
  ← consumes: 1.2 (drowsy events), 1.3 (speed violations), 2.4 (passengers)
  → feeds: 2.2 (weekly report), EmailJS

Feature 1.2 (Drowsiness)
  → triggers: 2.3 (voice alert)
  → adds to: composite risk score, 1.1 trip summary

Feature 1.3 (Speed Limit)
  → triggers: 2.3 (voice alert)
  → adds to: composite risk score, 1.1 trip summary

Feature 2.1 (Heatmap)
  ← consumes: GPS track + risk score timeline (collected from session start)

Feature 2.2 (Score Card)
  ← consumes: 1.1 trip summaries (last 7 days)
  → delivers: EmailJS weekly report

Feature 2.3 (Voice)
  ← triggered by: 1.1, 1.2, 1.3, Claude chat responses

Feature 2.4 (Passengers)
  → modifies: composite risk score multiplier
  → adds context to: Claude AI coach prompt
```

---

## localStorage Key Registry

| Key | Feature | Format | Max Size |
|-----|---------|--------|----------|
| `dg_trip_summaries` | 1.1 | Array[50] of summary objects | ~500KB |
| `dg_drowsiness_settings` | 1.2 | Settings object | <1KB |
| `dg_speedlimit_{lat}_{lon}` | 1.3 | `{limit, fetchedAt}` | <1KB each |
| `dg_route_{tripId}` | 2.1 | Array[1800] of track points | ~200KB each |
| `dg_weekly_reports` | 2.2 | Array[12] of report objects | ~300KB |
| `dg_voice_settings` | 2.3 | Settings object | <1KB |
| `dg_passenger_config` | 2.4 | `{count, multipliers}` | <1KB |

---

## Development Sequence (Recommended Build Order)

```
SPRINT 1 (Foundation)
  [ ] 2.3 Voice Coaching Mode  ← needed by 1.2 and 1.3 alerts
  [ ] 2.4 Passenger Count      ← modifies risk score early

SPRINT 2 (Sensors)
  [ ] 1.2 Drowsiness Detection ← builds on existing G-force pipeline
  [ ] 1.3 Speed Limit Overlay  ← builds on existing GPS pipeline

SPRINT 3 (Summaries)
  [ ] 1.1 Trip Summary Cards   ← consumes all sensor data

SPRINT 4 (Reports & Map)
  [ ] 2.2 Weekly Score Card    ← consumes trip summaries
  [ ] 2.1 Heatmap Route Replay ← requires route track data (add collection in Sprint 1)
```

> **Note:** Add `routeTrack[]` GPS collection in Sprint 1 even though the map (2.1) is Sprint 4 — you can't replay a trip that wasn't recorded.

---

## Tune Panel Additions Per Feature

| Feature | New Setting | Type | Default |
|---------|------------|------|---------|
| 1.1 | Auto-email trip summary | Toggle | OFF |
| 1.2 | Drowsiness detection | Toggle | ON |
| 1.2 | Jitter warning threshold | Slider 0.05–0.20 | 0.08 |
| 1.2 | Jitter critical threshold | Slider 0.10–0.25 | 0.14 |
| 1.3 | Speed limit overlay | Toggle | ON |
| 1.3 | Over-limit buffer (MPH) | Slider 0–15 | 5 |
| 2.1 | Route recording | Toggle | ON |
| 2.1 | Keep route history | Select 1–30 trips | 10 |
| 2.2 | Weekly report schedule | Toggle + time picker | OFF |
| 2.2 | Auto-email weekly report | Toggle | OFF |
| 2.3 | Voice coaching | Toggle | ON |
| 2.3 | Speech rate | Slider 0.8–1.3 | 1.0 |
| 2.3 | Voice input (STT) | Toggle | OFF |
| 2.4 | Passenger awareness | Toggle | ON |
| 2.4 | Passenger count (quick set) | 0–5+ selector | 0 |

---

*Last updated: DriverGuard AI v2.0 planning phase*
