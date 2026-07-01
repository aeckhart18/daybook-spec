# Daybook — Product Spec

*A conversational nutrition, supplement/medication, and activity tracker for iOS and Android.*

This document summarizes the product design worked out across planning and prototyping, as a handoff reference for development (whether built with Claude Code, by a hired developer, or in-house).

---

## 1. Core Concept

Most nutrition apps rely on manual entry or barcode scanning. Daybook's differentiator is **conversational logging** — the user talks or types naturally ("had eggs and toast, took my vitamin D, went for a walk"), and the app parses that into structured, confirmable data. The app also talks back: asking for clarification when something's ambiguous, and offering grounded insight when a log is solid.

**Platform:** Native iOS + Android (or React Native/Flutter with native modules for health/notifications). A browser can't deliver the core requirements — real pedometer access, native speech recognition, and background push notifications all require native platform APIs.

**Companion website:** A secondary, read-mostly dashboard, deferred until the app itself is built. It should support free-text "word dump" logging (typed instead of spoken) using the same parsing pipeline as the app.

---

## 2. Data Model

### Food Entry
```
{
  id, timestamp, raw_transcript, meal_type,
  food_name, quantity_label,          // e.g. "8oz" — never inherited from a sibling food in the same sentence
  calories, protein_g, carbs_g, fat_g,               // primary 4 — the only nutrients with user-set goals
  fiber_g, sugar_g, sodium_mg, saturated_fat_g,      // secondary — tracked, no goals, no red/green judgment styling
  vitamin_c_mg, vitamin_d_mcg, calcium_mg, iron_mg, potassium_mg,  // micronutrients — same treatment as secondary
  confidence: "high" | "low",
  needs_recipe: boolean               // true when this is an unmatched personal/named meal reference
}
```

### Substance Profile (supplements & medications)
```
{
  name, aliases,
  dose, unit,
  is_medication: boolean,             // medications and supplements share a shape but may warrant different handling
  reminderEnabled: boolean,
  reminderFrequency: "daily" | "weekly" | "monthly",
  reminderDay: number,                // weekday index (weekly) or day-of-month (monthly, clamped to month length)
  reminderTimes: [string],            // multiple times per day supported (e.g. twice-daily meds)
}
```

### Saved Meal (named recipe/template)
```
{
  name, aliases,
  totals: { calories, protein_g, carbs_g, fat_g, fiber_g, sugar_g, sodium_mg, saturated_fat_g,
            vitamin_c_mg, vitamin_d_mcg, calcium_mg, iron_mg, potassium_mg }
}
```
A saved meal is the *only* way a food reference skips restating quantities. A bare ingredient (e.g. "chicken thighs") never inherits a quantity from a prior log, no matter how many times it's been logged before.

### Exercise Entry
```
{ exercise_type, duration_min, calories_burned }
```
Kept intentionally simple — a plain AI estimate from the description, no device-workout matching. (Device reconciliation was designed and prototyped, then deliberately cut as scope creep relative to the app's core differentiator — see §7.)

### Weight Entry
```
{ value, unit: "lb" | "kg", t }
```
Loggable both conversationally and via a quick manual-entry field (a scale reading isn't really something people narrate).

### Day Log
```
{ food: [...], supplements: [...], exercise: [...], weight: WeightEntry | null, steps: number }
```
One per calendar day, keyed by date. `steps` is sourced passively from HealthKit/Health Connect — never manually logged.

---

## 3. Conversational Parsing Pipeline

Both voice (phone, native STT) and typed text (website) feed the **same parser** — a free-text "word dump" can always contain multiple mixed entries ("had eggs, took my pills, went for a walk"), so the parser always segments first rather than assuming single-entry input.

**Pipeline:**
1. **Transcribe** (voice) or take raw text directly (website)
2. **Segment** into individual entries
3. **Classify** each segment: food / supplement / exercise / weight
4. **Extract** structured fields per segment (LLM call against a strict JSON schema)
5. **Reconcile client-side** against known saved meals / substance profiles / prior pending entries
6. **Confidence-flag** anything incomplete
7. **Confirm card** — user reviews/edits before anything is saved; nothing logs silently

### Key parsing rules established during design
- **Never let one food's stated quantity bleed onto another food in the same sentence.** "8oz of chicken with rice" is two entries, only one with a quantity. The parser must explain *why* it split them, in the same reply that raises any other clarification needed — not as two separate, competing responses.
- **Unmatched personal/named meal references trigger a recipe request, not a guess.** "My protein oatmeal" (no saved match) doesn't get a placeholder card — the reply asks for ingredients and quantities, and the resulting combined entry auto-suggests saving under that original name. A stale, unanswered recipe request should expire after one turn rather than incorrectly attaching to an unrelated later message.
- **Corrections replace, not duplicate.** If a user corrects a pending (unconfirmed) entry in a later message, match by name/type and replace the card rather than stacking both. (Prototype note: this used string-overlap matching as a heuristic — a production version should have the model reference an explicit entry ID rather than inferring intent from text similarity.)
- **Dose inheritance for substances, never for food quantities.** Once a substance is known, its dose can be inferred from context ("took my lisinopril" → known dose). Modifier language ("extra," "half," "skipped") should flag for confirmation rather than silently overriding the default.
- **Reply tone:** grounded and specific, never generic praise. When everything parses cleanly, offer a real insight tied to the actual numbers (e.g., protein-forward, fiber-light) rather than empty encouragement.

---

## 4. Screens

### Home
- Date navigation (arrows/label) to browse history — **browsing never changes where new logs are written**; new entries always go to the actual current day regardless of what's being viewed
- Calorie ring + macro bars (primary 4, goal-tracked)
- "Also tracked" grid — secondary nutrients + micronutrients, no goals, no color judgment
- Steps card (device-synced)
- Reminders banner (only on today, only when something's due — see §5)
- Chronological entry list for the viewed day

### Log
- Chat-thread interface, not a form — conversation history persists across turns so corrections and recipe follow-ups have context
- Confirm cards render inline under the AI's reply, editable before saving
- Quantity field on food cards is always editable, even when the parser didn't catch one

### Meals
- Saved meal library: name, totals, edit (name + macros) or delete via a per-item menu

### Meds
- Substance list: dose, type (medication/supplement), quick "Log dose" button
- Reminder controls per substance: on/off, frequency (daily/weekly/monthly), day picker for weekly/monthly, and **multiple times per day** (chips, add/remove)
- Per-dose tracking: logging one dose only clears the next due time slot, not the whole day — critical for twice-daily+ medications
- Notification permission toggle (real OS-level notifications in the native app; browser Notification API was used as a prototype stand-in)

### Trends
- Body weight — line chart, 7-day window, quick manual-entry row
- Steps — bar chart, 7-day window, dashed reference line at a common goal (10,000, should likely be user-configurable)
- Calories — bar chart, 7-day window, dashed line at daily target
- (An earlier "average macro split" card was removed as redundant with what Home already shows)

---

## 5. Reminders & Notifications

- Each substance profile supports **daily / weekly / monthly** frequency, with **multiple times per day** for the daily case (the common twice- or three-times-daily medication pattern)
- Due-reminder logic compares scheduled times already passed today against doses actually logged today — not just a single "taken today" flag — so a second daily dose still reminds after the first is logged
- **Known simplification:** doses are treated as interchangeable within a day, not tied to a specific time slot. Logging any dose clears the earliest unfulfilled slot; the system doesn't verify *which* scheduled dose was actually taken. Worth revisiting if stricter per-slot tracking becomes a requirement.
- Real app: OS-native scheduled notifications (fire even if the app is closed/phone locked) — this is a materially better mechanism than what any browser-based demo can achieve, and shouldn't be underestimated as an implementation gap.

---

## 6. Wearable & Sensor Integration

| Source | Integration path |
|---|---|
| Native pedometer | HealthKit (iOS) / Health Connect (Android) |
| Apple Watch | Flows through HealthKit automatically |
| Hume Band | Flows through HealthKit/Health Connect via the Hume Health app (confirmed: not a separate direct API) |
| WHOOP | Separate OAuth-based developer API — doesn't flow through Apple Health by default; some users may optionally sync it there |

One HealthKit/Health Connect integration covers steps, heart rate, and most wearables. WHOOP is the one outlier needing dedicated work.

---

## 7. Deliberately Cut / Simplified Scope

Worth preserving *why* these were cut, not just that they were:

- **Device-workout reconciliation for exercise** (matching a voice-logged "went for a walk" against an automatically-synced Apple Watch workout, preferring the device's more accurate calorie/duration data, with an ambiguous-match picker when multiple candidates exist) — this was fully designed and prototyped, but cut as scope creep. It solved a real problem (double-counted or conflicting calorie-burn numbers) but added significant surface area for something that isn't core to what makes this app different. **Worth reconsidering later**, but shouldn't block an initial launch.
- **"Average macro split"** trend card — removed for being redundant with Home's existing macro display.
- **Monthly reminders by day-of-number** — simplest possible version of "monthly." Doesn't match how people actually think about monthly prescriptions (often "every 4 weeks" or relative to last dose, not a fixed calendar day). Flagged as a likely revisit.

---

## 8. Onboarding (designed, not yet prototyped)

Six-step sequence: **Welcome → Create account → Grant permissions (mic, health, notifications — requested contextually, not all at once) → Baseline profile (age/weight/goals, needed for macro targets) → First guided log → Home dashboard.**

The "first guided log" step is the most important — it should be a real, working log (not a mockup), with worked examples shown for both food ("two scrambled eggs, a tablespoon of butter, one slice of toast, a black coffee") and medications/supplements ("500mg vitamin C and my lisinopril, 10mg, around 8am"), each shown as a before/after (raw sentence → parsed result) so the mental model is visual, not just explained in text.

---

## 9. Technical Notes for Implementation

- **Nutrition data source:** the prototype uses AI-estimated values for demonstration. Production needs a real food database (USDA FoodData Central, Nutritionix, or similar) for the primary lookup, with the LLM parser handling segmentation/extraction and quantity interpretation, not nutrient values themselves where a database match exists.
- **Backend required for:** cross-device sync, the website dashboard, and holding the LLM API key securely (the prototype calls the Anthropic API directly from the client, which is fine for a demo but not for production).
- **Local-first on the phone:** voice capture, parsing (where feasible), and logging should work offline, syncing when connectivity returns.
- **Confidence flagging should extend per-nutrient, not just per-item**, once a real food database is in place — a dish might be well-estimated on calories/protein but genuinely unknown on sodium; showing "—" is more honest than a false 0.

---

*This document reflects the design and prototype as of this conversation. A working (browser-based) interactive prototype demonstrating this logic is available as a separate file.*

