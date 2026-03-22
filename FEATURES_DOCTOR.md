# MediPulse — Doctor Features

## Route Structure
All doctor routes are under `/app/doctor/` and protected by middleware that checks `profile.role === 'doctor'`.

---

## Feature 1: Ward Overview Dashboard (`/doctor`)

The **home screen** for doctors — a real-time overview of all assigned patients.

### UI Layout
- Grid of `PatientCard` components (2 columns on mobile, 3–4 on desktop)
- Top bar shows: unread alert count, current time, doctor name
- Sticky alert banner for any `critical` severity alerts
- Filter bar: All | Critical | High Risk | Stable | My Ward

### PatientCard Component
Each card shows:
- Patient name, age, bed/ward number
- Risk score badge (color-coded: green/yellow/orange/red)
- Latest vitals mini-strip (4 most recent readings as sparkline)
- Active alerts count badge
- Status dot: `LIVE` (pulsing cyan) when receiving real-time data
- One-tap to full patient profile

### Data
- Load all assigned patients via `lib/supabase/queries/patients.ts`
- Subscribe to `risk_scores` realtime for live risk score updates
- Subscribe to `alerts` realtime for badge count updates

---

## Feature 2: Patient Profile (`/doctor/patients/[id]`)

Full patient detail view with tabbed navigation.

### Tabs

#### Tab 1: Overview
- Patient demographics card (photo, name, DOB, blood type, allergies, admission date)
- Primary diagnosis + notes
- Assigned doctor confirmation
- Emergency contact
- Quick action buttons: Message Patient | Add Note | View Alerts

#### Tab 2: Vitals (`/doctor/patients/[id]/vitals`)
- `VitalsGrid` — 2×3 grid of `VitalsCard` components (one per vital type)
- Each `VitalsCard` shows:
  - Current value (large, color-coded)
  - Trend arrow (↑↓→) vs last reading
  - Mini sparkline (last 20 readings)
  - Status badge: Normal / Warning / Critical
  - LIVE pulsing indicator
- `VitalsChart` — Recharts line chart with:
  - Time range selector: 1h | 6h | 24h | 7d
  - Toggle between vital types
  - Anomaly markers (red dots at detected anomaly points)
  - Reference range shading (green band = normal zone)
- AI Anomaly panel: lists detected anomalies with explanations

#### Tab 3: Lab Results (`/doctor/patients/[id]/labs`)
- Chronological list of lab reports
- Each report shows: test name, collection date, overall flag status
- Expandable: shows all parameters with values, reference ranges, H/L/HH/LL flags
- "AI Summary" toggle: shows Claude plain-language summary
- Upload new lab report (PDF → Supabase Storage)

#### Tab 4: Medications (`/doctor/patients/[id]/medications`)
- Active medications list with dosage, frequency, route
- Today's dose schedule with taken/pending status
- Add / discontinue medication form
- Drug interaction warnings (if AI detects conflicts)

#### Tab 5: Notes (`/doctor/patients/[id]/notes`)
- Timeline of all clinical notes (SOAP, progress, admission, discharge)
- "New Voice Note" button → opens `SOAPNoteEditor`
- Voice Note Flow:
  1. Doctor taps microphone button
  2. `VoiceRecorder` component activates (visual waveform, live transcript)
  3. Doctor speaks freely about patient assessment
  4. Stop recording → transcript displayed
  5. "Generate SOAP Note" → calls `/api/ai/soap-note`
  6. SOAP note rendered in editable form (doctor can edit before saving)
  7. Save → stored in `clinical_notes` table

---

## Feature 3: Alert Inbox (`/doctor/alerts`)

### Layout
- Three-tab header: All | Active | Acknowledged
- Sorted by: severity (critical first), then timestamp
- Critical alerts also appear in persistent banner on all doctor pages

### AlertItem Component
Each alert shows:
- Severity badge (Critical 🔴 / Warning 🟡 / Info 🔵)
- Patient name + ward/bed
- Alert title and short description
- Timestamp (relative: "3 min ago")
- AI reasoning excerpt (expandable)
- Action buttons: View Patient | Acknowledge | Escalate

### Alert Generation Logic
Located in `lib/alerts/alert-engine.ts`:
```typescript
// Triggers for alert creation:
// 1. Threshold breach: any vital outside VITAL_RANGES.warning
// 2. AI anomaly detection: severity = 'warning' or 'critical'
// 3. Risk score jump: score increases by 15+ points in one cycle
// 4. Missed critical medication dose: 30+ minutes overdue
```

### Push Notifications
- Browser push notifications via `lib/alerts/notification.ts`
- Request permission on first doctor login
- Critical alerts trigger push notification even when tab is in background

---

## Feature 4: AI Diagnosis Assistance (`/doctor/ai-assist`)

### Layout
- Patient selector (search/dropdown of assigned patients)
- Patient summary auto-populated on selection
- Free-text question field: "What's your clinical question?"
- Submit → calls `/api/ai/diagnosis`

### Results Panel
- **Differential Diagnoses**: Accordion list with likelihood badges
  - Each entry: diagnosis name, likelihood (High/Medium/Low), supporting evidence, workup suggestions
- **Urgent Flags**: Red banner list of items needing immediate attention
- **Medication Concerns**: Orange warning for any drug interaction flags
- **Suggested Orders**: Checklist-style suggested tests/interventions
- **Clinical Pearls**: Blue info cards with contextual clinical notes
- AI Disclaimer badge always visible

---

## Feature 5: Predictive Risk Score View

Accessible from both PatientCard and Patient Profile.

### `RiskScoreCard` Component
- Circular gauge (0–100 scale)
- Color zones: 0–30 (green), 31–60 (yellow), 61–80 (orange), 81–100 (red)
- Current score + trend (vs. 6 hours ago)
- Prediction window: "Next 24 hours"
- Top 3 contributing factors listed with weight bars
- Escalation threshold text: "Escalate if SpO2 drops below 91%"
- "Recalculate Now" button (on-demand refresh)

---

## Feature 6: Secure Messaging (`/doctor/messages`)

### Layout
- Left panel: conversation list (patient name, last message preview, timestamp, unread count)
- Right panel: message thread
- Supabase Realtime subscription on `messages` table

### MessageThread Component
- Bubbles: doctor messages right-aligned (navy), patient messages left-aligned (gray)
- Timestamps on hover
- Read receipts (✓ sent, ✓✓ read)
- Message input with Send button
- Optional: attach lab result or note reference

### Security
- Conversations only exist between a patient and their assigned doctor
- RLS policy enforces this at DB level — no cross-patient message access possible
