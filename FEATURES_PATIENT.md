# MediPulse — Patient Features

## Route Structure
All patient routes are under `/app/patient/` and protected by middleware that checks `profile.role === 'patient'`.

Patient UI is intentionally **simpler and more friendly** than the doctor UI. No clinical jargon, no dense data tables. Everything is visual, scannable, and designed for someone without a medical background.

---

## Feature 1: Health Timeline (`/patient`)

The **home screen** for patients — a chronological, visual narrative of their health journey.

### Layout
- Top: personalized greeting ("Good morning, Riya 👋") + today's date
- Summary card: Current status | Ward & Bed | Assigned Doctor
- Vertical timeline feed below (most recent at top)

### Timeline Events
Each event is a card on the timeline with an icon, date, and summary:

| Event Type | Icon | Content |
|---|---|---|
| Vital Reading | 💓 | "Heart rate normal — 78 bpm" |
| Lab Result | 🧪 | "CBC results available — tap to view" |
| Medication Taken | 💊 | "Amoxicillin 500mg taken at 8:00 AM" |
| Doctor Note | 📋 | "Dr. Sharma added a progress note" |
| Doctor Message | 💬 | "New message from Dr. Sharma" |
| Appointment | 📅 | "Review appointment scheduled" |
| AI Insight | 🤖 | "Your SpO2 has been stable for 24 hours" |

### Implementation
```typescript
// Health timeline is assembled from multiple data sources
// lib/supabase/queries/patients.ts → getHealthTimeline()
// Merges: vitals, lab_results, medication_doses, clinical_notes, messages
// Sorted by timestamp, paginated (20 items/page)
```

---

## Feature 2: My Vitals (`/patient/vitals`)

Patient's own vitals history in a friendly, non-clinical presentation.

### Layout
- Four cards (HR, BP, SpO2, Temp) with:
  - Current value in large font
  - Plain-language status: "Normal ✅" / "Slightly elevated ⚠️" / "Being monitored 🔴"
  - Trend sentence: "Your heart rate has been stable over the last 24 hours."
- Interactive Recharts line chart:
  - Time selector: Today | This Week
  - Toggle between vital types
  - Normal range shown as shaded green band
  - No anomaly markers shown to patient (clinical detail kept for doctors)
- "What does this mean?" expandable info section for each vital

### Language Rules for Patient Vitals
- NEVER show raw clinical flags or anomaly data to patients
- Status must use one of: "Normal", "Slightly elevated", "A bit low", "Being closely monitored"
- If value is critical, show: "Your care team has been notified and is monitoring this closely"

---

## Feature 3: Medication Tracker (`/patient/medications`)

### Layout
- **Today's Schedule** (top section)
  - Timeline of doses for today: 8 AM | 12 PM | 6 PM | 10 PM
  - Each dose: medication name, dosage, status (Taken ✅ / Due 🕐 / Missed ⚠️)
  - "Mark as Taken" button for pending doses
- **All Active Medications** (list section)
  - Card per medication: name, dosage, frequency, end date (if set)
  - "Learn more" → AI plain-language explanation of the drug's purpose
- **Adherence Meter** — visual ring showing % of doses taken this week

### Reminders
- Browser push notification at each scheduled dose time
- Configurable: patient can set reminder 15 min before dose

### MedicationTracker Component
```typescript
// Dose status logic
function getDoseStatus(scheduledAt: Date, takenAt: Date | null): DoseStatus {
  if (takenAt) return 'taken';
  const now = new Date();
  const minutesOverdue = differenceInMinutes(now, scheduledAt);
  if (minutesOverdue < 0) return 'upcoming';
  if (minutesOverdue < 60) return 'due';
  return 'missed';
}
```

---

## Feature 4: Lab Results (`/patient/labs`)

### Layout
- Chronological list of lab reports (newest first)
- Each report card:
  - Test name + date collected
  - Overall status badge: "All Normal ✅" / "Some Results Need Attention ⚠️"
  - "View Details" → expandable panel

### Lab Detail View
- Two modes toggled by patient:

  **Simple View (default)**
  - AI plain-language summary (from `/api/ai/lab-summary`)
  - List of abnormal findings in plain language: "Your hemoglobin is slightly low. This means your blood may be carrying a little less oxygen than ideal."
  - "What should I do?" → followup note from AI

  **Detailed View (optional)**
  - Full parameter table with values, units, reference ranges
  - H/L flags visible (but explained: "H means higher than normal range")
  - Download PDF report button (from Supabase Storage)

### AI Summary Generation
- Generated on first patient view of a lab result
- Cached in `lab_results.ai_summary` column
- Regeneration not needed unless result is amended

---

## Feature 5: Appointments & Visit History (`/patient/appointments`)

### Upcoming Appointments
- Card list: date, time, doctor name, type (Follow-up / Review / Procedure)
- "Add to Calendar" button (generates .ics file)

### Visit History
- Past visits with discharge summaries (doctor-uploaded)
- Admission → discharge date shown per visit
- Primary diagnosis listed per visit

---

## Feature 6: Secure Messaging (`/patient/messages`)

### Layout
- Single conversation (with assigned doctor only)
- Chat bubble interface:
  - Patient messages: right-aligned, soft teal
  - Doctor messages: left-aligned, navy
- Doctor availability indicator: "Dr. Sharma usually responds within 2 hours"
- Message input at bottom

### What Patients Can Send
- Text messages
- Questions about their medications or test results
- Symptom updates between visits

### Messaging Rules
- Patient can only message their assigned doctor
- No direct messaging to nurses, admin, or other doctors
- Message history is part of the patient's medical record

---

## Patient UI Design Principles

1. **No jargon** — replace all clinical terms with plain English
2. **Calm color palette** — avoid red/warning colors unless truly urgent; use blue and green primarily
3. **Actionable every time** — every screen ends with a clear next step or reassurance
4. **Mobile-first** — designed for one-thumb use on a phone in a hospital bed
5. **Low cognitive load** — never show more than one primary action per screen
6. **Empowering** — patients should feel informed and in control, not overwhelmed

---

## Patient Onboarding Flow

On first login after account creation:

1. Welcome screen with doctor's name and photo
2. Quick tour of the three main tabs (Timeline, Medications, Labs)
3. Request notification permission for medication reminders
4. "Your care team is here for you" confirmation screen
