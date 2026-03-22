# MediPulse — Project Structure

## Root Directory

```
medipulse/
├── .cursorrules                  # Cursor AI rules (global)
├── .env.local                    # Environment variables (never commit)
├── next.config.ts                # Next.js config + PWA setup
├── tailwind.config.ts            # Tailwind + custom design tokens
├── tsconfig.json
├── package.json
│
├── public/
│   ├── manifest.json             # PWA manifest
│   ├── icons/                    # PWA icons (192, 512px)
│   └── sounds/                   # Alert notification sounds
│
├── app/                          # Next.js App Router
│   ├── layout.tsx                # Root layout (fonts, providers)
│   ├── page.tsx                  # Landing / redirect to login
│   ├── globals.css               # Global styles + CSS variables
│   │
│   ├── (auth)/                   # Auth group (no sidebar)
│   │   ├── login/page.tsx
│   │   ├── signup/page.tsx
│   │   └── layout.tsx
│   │
│   ├── doctor/                   # Doctor-only routes
│   │   ├── layout.tsx            # Doctor sidebar + nav
│   │   ├── page.tsx              # Ward overview dashboard
│   │   ├── patients/
│   │   │   ├── page.tsx          # Patient list
│   │   │   └── [id]/
│   │   │       ├── page.tsx      # Individual patient profile
│   │   │       ├── vitals/page.tsx
│   │   │       ├── labs/page.tsx
│   │   │       ├── medications/page.tsx
│   │   │       └── notes/page.tsx
│   │   ├── alerts/page.tsx       # Alert inbox
│   │   ├── ai-assist/page.tsx    # Diagnosis assistance panel
│   │   └── messages/page.tsx     # Doctor messaging hub
│   │
│   ├── patient/                  # Patient-only routes
│   │   ├── layout.tsx            # Patient nav (minimal)
│   │   ├── page.tsx              # Health timeline home
│   │   ├── vitals/page.tsx       # My vitals history
│   │   ├── medications/page.tsx  # Medication tracker
│   │   ├── labs/page.tsx         # Lab results + AI summaries
│   │   ├── appointments/page.tsx
│   │   └── messages/page.tsx     # Message my doctor
│   │
│   └── api/                      # API Routes
│       ├── ai/
│       │   ├── anomaly/route.ts      # POST: analyze vitals for anomalies
│       │   ├── risk-score/route.ts   # POST: predict deterioration risk
│       │   ├── soap-note/route.ts    # POST: voice transcript → SOAP note
│       │   ├── diagnosis/route.ts    # POST: diagnosis assistance
│       │   └── lab-summary/route.ts  # POST: plain-language lab summary
│       ├── vitals/
│       │   ├── simulate/route.ts     # GET: WebSocket mock vitals stream
│       │   └── [patientId]/route.ts  # GET/POST: vitals CRUD
│       ├── alerts/
│       │   ├── route.ts              # GET: all alerts for doctor
│       │   └── [id]/route.ts         # PATCH: acknowledge alert
│       └── messages/
│           ├── route.ts              # GET/POST: conversations
│           └── [conversationId]/route.ts
│
├── components/
│   ├── ui/                       # shadcn/ui base components
│   │   ├── button.tsx
│   │   ├── card.tsx
│   │   ├── badge.tsx
│   │   ├── dialog.tsx
│   │   └── ... (all shadcn components)
│   │
│   ├── layout/
│   │   ├── DoctorSidebar.tsx
│   │   ├── PatientNav.tsx
│   │   ├── TopBar.tsx
│   │   └── AlertBadge.tsx
│   │
│   ├── vitals/
│   │   ├── VitalsCard.tsx         # Single metric card (HR, BP, etc.)
│   │   ├── VitalsChart.tsx        # Recharts time-series chart
│   │   ├── VitalsGrid.tsx         # 2x2 grid of VitalsCards
│   │   ├── VitalsStatusDot.tsx    # Color-coded status indicator
│   │   └── LiveVitalsBadge.tsx    # "LIVE" pulsing indicator
│   │
│   ├── alerts/
│   │   ├── AlertItem.tsx          # Single alert row
│   │   ├── AlertInbox.tsx         # Full alert list
│   │   ├── AlertBanner.tsx        # Critical alert banner (top of screen)
│   │   └── AlertSeverityBadge.tsx
│   │
│   ├── ai/
│   │   ├── RiskScoreCard.tsx      # Deterioration risk gauge
│   │   ├── AnomalyHighlight.tsx   # Highlighted anomaly in vitals
│   │   ├── DiagnosisPanel.tsx     # AI diagnosis assistance UI
│   │   ├── SOAPNoteEditor.tsx     # Voice → SOAP note component
│   │   ├── VoiceRecorder.tsx      # Mic button + transcript display
│   │   └── LabSummaryCard.tsx     # AI plain-language lab summary
│   │
│   ├── patient/
│   │   ├── HealthTimeline.tsx     # Visual health history timeline
│   │   ├── MedicationTracker.tsx  # Dose schedule + adherence
│   │   ├── LabReportViewer.tsx    # Lab report + AI summary toggle
│   │   └── PatientProfileCard.tsx
│   │
│   ├── doctor/
│   │   ├── WardOverview.tsx       # Grid of patient cards for ward
│   │   ├── PatientCard.tsx        # Compact patient status card
│   │   ├── PatientProfile.tsx     # Full patient detail view
│   │   └── AssignedPatientList.tsx
│   │
│   └── messages/
│       ├── ConversationList.tsx
│       ├── MessageThread.tsx
│       └── MessageInput.tsx
│
├── lib/
│   ├── supabase/
│   │   ├── client.ts             # Browser Supabase client
│   │   ├── server.ts             # Server Supabase client (RSC)
│   │   ├── middleware.ts         # Auth middleware helper
│   │   └── queries/
│   │       ├── patients.ts       # All patient DB queries
│   │       ├── vitals.ts         # Vitals queries
│   │       ├── labs.ts           # Lab result queries
│   │       ├── medications.ts    # Medication queries
│   │       ├── alerts.ts         # Alert queries
│   │       └── messages.ts       # Messaging queries
│   │
│   ├── ai/
│   │   ├── claude-client.ts      # Anthropic SDK setup
│   │   ├── anomaly-detection.ts  # Vitals anomaly analysis prompt + logic
│   │   ├── risk-scoring.ts       # Deterioration prediction prompt + logic
│   │   ├── soap-generator.ts     # Voice transcript → SOAP note
│   │   ├── diagnosis-assist.ts   # Differential diagnosis assistance
│   │   └── lab-summarizer.ts     # Medical → plain language
│   │
│   ├── vitals/
│   │   ├── mock-engine.ts        # WebSocket vitals simulator
│   │   ├── vitals-constants.ts   # Normal ranges, thresholds
│   │   └── vitals-types.ts       # TypeScript types for vitals
│   │
│   ├── alerts/
│   │   ├── alert-engine.ts       # Alert generation logic
│   │   ├── alert-types.ts        # Alert severity enums + types
│   │   └── notification.ts       # Browser push notification helper
│   │
│   └── utils/
│       ├── cn.ts                 # className utility
│       ├── format-date.ts        # Date formatting helpers
│       ├── format-vitals.ts      # Vitals display formatting
│       └── medical-units.ts      # Unit conversion helpers
│
├── hooks/
│   ├── useRealtimeVitals.ts      # Supabase realtime vitals subscription
│   ├── useAlerts.ts              # Alert polling + realtime
│   ├── useVoiceRecorder.ts       # Web Speech API hook
│   ├── usePatients.ts            # Doctor's patient list
│   └── useMessages.ts            # Messaging subscription
│
├── store/
│   ├── useAlertStore.ts          # Zustand: global alert state
│   ├── usePatientStore.ts        # Zustand: selected patient state
│   └── useUIStore.ts             # Zustand: sidebar, modals, theme
│
├── types/
│   ├── database.types.ts         # Auto-generated Supabase types
│   ├── patient.ts
│   ├── vitals.ts
│   ├── alert.ts
│   ├── medication.ts
│   ├── lab-result.ts
│   └── message.ts
│
└── docs/                         # Cursor prompt MD files
    ├── PROJECT_OVERVIEW.md       # ← This file
    ├── PROJECT_STRUCTURE.md
    ├── DATABASE_SCHEMA.md
    ├── AI_INTEGRATION.md
    ├── REALTIME_VITALS.md
    ├── FEATURES_DOCTOR.md
    ├── FEATURES_PATIENT.md
    ├── ALERTS_SYSTEM.md
    └── DESIGN_SYSTEM.md
```
