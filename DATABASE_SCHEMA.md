# MediPulse — Database Schema

## Supabase (PostgreSQL) with Row Level Security

All tables have RLS enabled. No client-side query bypasses RLS — all access is through authenticated Supabase clients with JWT-based role detection.

---

## Enums

```sql
CREATE TYPE user_role AS ENUM ('doctor', 'patient');
CREATE TYPE alert_severity AS ENUM ('critical', 'warning', 'info');
CREATE TYPE alert_status AS ENUM ('active', 'acknowledged', 'resolved');
CREATE TYPE vital_type AS ENUM ('heart_rate', 'blood_pressure_sys', 'blood_pressure_dia', 'spo2', 'temperature', 'respiratory_rate');
CREATE TYPE medication_status AS ENUM ('active', 'discontinued', 'completed');
CREATE TYPE note_type AS ENUM ('soap', 'progress', 'admission', 'discharge');
```

---

## Tables

### `profiles`
Extends Supabase `auth.users`. Created via trigger on user signup.

```sql
CREATE TABLE profiles (
  id           UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  role         user_role NOT NULL,
  full_name    TEXT NOT NULL,
  email        TEXT NOT NULL,
  phone        TEXT,
  avatar_url   TEXT,
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  updated_at   TIMESTAMPTZ DEFAULT NOW()
);
```

### `doctors`
Extended profile for doctor users.

```sql
CREATE TABLE doctors (
  id               UUID PRIMARY KEY REFERENCES profiles(id) ON DELETE CASCADE,
  specialization   TEXT NOT NULL,
  department       TEXT NOT NULL,
  license_number   TEXT UNIQUE NOT NULL,
  ward_assignment  TEXT
);
```

### `patients`
Extended profile for patient users.

```sql
CREATE TABLE patients (
  id                  UUID PRIMARY KEY REFERENCES profiles(id) ON DELETE CASCADE,
  date_of_birth       DATE NOT NULL,
  blood_type          TEXT,
  allergies           TEXT[],
  emergency_contact   JSONB,          -- { name, phone, relation }
  admission_date      DATE,
  discharge_date      DATE,
  ward                TEXT,
  bed_number          TEXT,
  assigned_doctor_id  UUID REFERENCES doctors(id),
  diagnosis_primary   TEXT,
  diagnosis_notes     TEXT,
  insurance_info      JSONB
);
```

### `vitals`
Individual vital sign readings. Populated by WebSocket mock engine during development.

```sql
CREATE TABLE vitals (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  patient_id   UUID NOT NULL REFERENCES patients(id) ON DELETE CASCADE,
  recorded_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  vital_type   vital_type NOT NULL,
  value        NUMERIC NOT NULL,
  unit         TEXT NOT NULL,
  is_anomaly   BOOLEAN DEFAULT FALSE,
  anomaly_note TEXT,               -- AI-generated explanation if anomaly
  source       TEXT DEFAULT 'mock' -- 'mock' | 'manual' | 'device'
);

CREATE INDEX idx_vitals_patient_time ON vitals(patient_id, recorded_at DESC);
CREATE INDEX idx_vitals_anomaly ON vitals(is_anomaly) WHERE is_anomaly = TRUE;
```

### `lab_results`
Lab reports and individual test values.

```sql
CREATE TABLE lab_results (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  patient_id       UUID NOT NULL REFERENCES patients(id) ON DELETE CASCADE,
  ordered_by       UUID REFERENCES doctors(id),
  test_name        TEXT NOT NULL,
  test_category    TEXT NOT NULL,  -- 'CBC', 'Metabolic Panel', 'Lipid Panel', etc.
  results          JSONB NOT NULL, -- Array of { parameter, value, unit, reference_range, flag }
  collected_at     TIMESTAMPTZ NOT NULL,
  resulted_at      TIMESTAMPTZ,
  report_file_url  TEXT,           -- Supabase Storage URL
  ai_summary       TEXT,           -- Claude plain-language summary (cached)
  summary_generated_at TIMESTAMPTZ,
  notes            TEXT
);

CREATE INDEX idx_labs_patient ON lab_results(patient_id, collected_at DESC);
```

### `medications`
Medication prescriptions and schedules.

```sql
CREATE TABLE medications (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  patient_id      UUID NOT NULL REFERENCES patients(id) ON DELETE CASCADE,
  prescribed_by   UUID REFERENCES doctors(id),
  drug_name       TEXT NOT NULL,
  generic_name    TEXT,
  dosage          TEXT NOT NULL,       -- e.g. "500mg"
  frequency       TEXT NOT NULL,       -- e.g. "twice daily"
  route           TEXT NOT NULL,       -- e.g. "oral", "IV", "topical"
  schedule        JSONB NOT NULL,      -- Array of { time: "08:00", dose: "500mg" }
  start_date      DATE NOT NULL,
  end_date        DATE,
  status          medication_status DEFAULT 'active',
  instructions    TEXT,
  side_effects    TEXT[]
);
```

### `medication_doses`
Individual dose tracking for adherence.

```sql
CREATE TABLE medication_doses (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  medication_id   UUID NOT NULL REFERENCES medications(id) ON DELETE CASCADE,
  patient_id      UUID NOT NULL REFERENCES patients(id),
  scheduled_at    TIMESTAMPTZ NOT NULL,
  taken_at        TIMESTAMPTZ,
  status          TEXT NOT NULL DEFAULT 'pending',  -- 'pending' | 'taken' | 'missed' | 'skipped'
  notes           TEXT
);

CREATE INDEX idx_doses_patient_scheduled ON medication_doses(patient_id, scheduled_at);
```

### `alerts`
AI-generated and threshold-based clinical alerts for doctors.

```sql
CREATE TABLE alerts (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  patient_id      UUID NOT NULL REFERENCES patients(id) ON DELETE CASCADE,
  doctor_id       UUID NOT NULL REFERENCES doctors(id),
  severity        alert_severity NOT NULL,
  status          alert_status DEFAULT 'active',
  title           TEXT NOT NULL,
  description     TEXT NOT NULL,
  ai_explanation  TEXT,            -- Claude's detailed reasoning
  trigger_data    JSONB,           -- Raw data that triggered the alert
  vital_type      vital_type,      -- Which vital triggered (if applicable)
  risk_score      NUMERIC,         -- 0–100 deterioration risk score
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  acknowledged_at TIMESTAMPTZ,
  acknowledged_by UUID REFERENCES doctors(id),
  resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_alerts_doctor_status ON alerts(doctor_id, status, created_at DESC);
CREATE INDEX idx_alerts_patient ON alerts(patient_id, created_at DESC);
```

### `clinical_notes`
SOAP notes and other doctor documentation.

```sql
CREATE TABLE clinical_notes (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  patient_id      UUID NOT NULL REFERENCES patients(id) ON DELETE CASCADE,
  author_id       UUID NOT NULL REFERENCES doctors(id),
  note_type       note_type NOT NULL,
  voice_transcript TEXT,          -- Raw voice transcript (pre-AI)
  subjective      TEXT,           -- SOAP: S
  objective       TEXT,           -- SOAP: O
  assessment      TEXT,           -- SOAP: A
  plan            TEXT,           -- SOAP: P
  raw_content     TEXT,           -- For non-SOAP note types
  ai_generated    BOOLEAN DEFAULT FALSE,
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### `risk_scores`
Periodic AI-generated deterioration risk scores per patient.

```sql
CREATE TABLE risk_scores (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  patient_id      UUID NOT NULL REFERENCES patients(id) ON DELETE CASCADE,
  score           NUMERIC NOT NULL CHECK (score >= 0 AND score <= 100),
  risk_level      TEXT NOT NULL,  -- 'low' | 'medium' | 'high' | 'critical'
  contributing_factors JSONB,     -- Array of { factor, weight, description }
  ai_reasoning    TEXT,
  calculated_at   TIMESTAMPTZ DEFAULT NOW(),
  window_hours    INTEGER DEFAULT 24  -- Prediction window
);

CREATE INDEX idx_risk_patient_time ON risk_scores(patient_id, calculated_at DESC);
```

### `messages`
Secure doctor-patient messaging.

```sql
CREATE TABLE conversations (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  doctor_id       UUID NOT NULL REFERENCES doctors(id),
  patient_id      UUID NOT NULL REFERENCES patients(id),
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  last_message_at TIMESTAMPTZ,
  UNIQUE(doctor_id, patient_id)
);

CREATE TABLE messages (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id  UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
  sender_id        UUID NOT NULL REFERENCES profiles(id),
  content          TEXT NOT NULL,
  is_read          BOOLEAN DEFAULT FALSE,
  created_at       TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_messages_conversation ON messages(conversation_id, created_at);
```

---

## Row Level Security Policies

```sql
-- Profiles: users see only their own
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users can view own profile" ON profiles
  FOR SELECT USING (auth.uid() = id);

-- Vitals: doctors see assigned patients; patients see own
ALTER TABLE vitals ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Doctors see assigned patient vitals" ON vitals
  FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM patients p
      WHERE p.id = vitals.patient_id
      AND p.assigned_doctor_id = auth.uid()
    )
  );
CREATE POLICY "Patients see own vitals" ON vitals
  FOR SELECT USING (patient_id = auth.uid());

-- Alerts: only the assigned doctor
ALTER TABLE alerts ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Doctors see own alerts" ON alerts
  FOR ALL USING (doctor_id = auth.uid());

-- Messages: both parties in conversation
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Conversation participants see messages" ON messages
  FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM conversations c
      WHERE c.id = messages.conversation_id
      AND (c.doctor_id = auth.uid() OR c.patient_id = auth.uid())
    )
  );
```

---

## Supabase Realtime

Enable realtime for these tables in Supabase dashboard:
- `vitals` — doctors subscribe per patient for live vitals
- `alerts` — doctors subscribe to their alert feed
- `messages` — both parties subscribe to conversation updates
- `risk_scores` — doctors subscribe per patient

```typescript
// Example realtime subscription (in useRealtimeVitals.ts)
const channel = supabase
  .channel(`vitals:${patientId}`)
  .on('postgres_changes', {
    event: 'INSERT',
    schema: 'public',
    table: 'vitals',
    filter: `patient_id=eq.${patientId}`
  }, (payload) => {
    // Handle new vital reading
  })
  .subscribe();
```

---

## Supabase Storage Buckets

| Bucket | Purpose | Access |
|---|---|---|
| `lab-reports` | PDF lab reports | Authenticated users only |
| `avatars` | Profile photos | Public |
| `voice-transcripts` | Raw voice recordings (temp) | Authenticated only, auto-expire 7 days |
