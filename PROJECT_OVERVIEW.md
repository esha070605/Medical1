# MediPulse — Project Overview

## What is MediPulse?
MediPulse is a **smart hospital health management system** that gives doctors real-time visibility into patient vitals, lab results, and AI-powered risk assessments — while giving patients a clear, understandable window into their own health journey.

It is built as a **Progressive Web App (PWA)** on Next.js 14, backed by Supabase for real-time data, and enhanced with Claude AI for clinical intelligence.

---

## Problem Statement
Hospital staff and patients face three core problems today:
1. **Fragmented data** — vitals, labs, medications, and notes live in separate systems
2. **Delayed alerts** — critical deterioration is caught reactively, not proactively
3. **Patient opacity** — patients receive medical records they cannot understand

MediPulse solves all three in a single, mobile-optimized interface.

---

## User Roles

### 👨‍⚕️ Doctor / Physician
- Monitors a ward of patients in real time
- Receives tiered AI alerts when vitals deviate
- Dictates clinical notes via voice → auto-structured as SOAP notes
- Sees predictive risk scores for each patient (next 6–24h deterioration probability)
- Messages patients securely within the app
- Views diagnosis assistance suggestions based on patient history + current data

### 🧑‍🦯 Patient (Self-Access)
- Views their own health timeline (vitals history, lab trends, medication schedule)
- Reads AI-generated plain-language summaries of lab reports
- Tracks medication doses and receives reminders
- Messages their assigned doctor
- Reviews visit history and upcoming appointments

---

## Key Differentiators

| Feature | Why It Matters |
|---|---|
| **AI Early Warning System** | Pattern-based anomaly detection beyond simple thresholds — catches deterioration 2–6 hours earlier |
| **Voice SOAP Notes** | Doctors speak naturally, Claude structures output into clinical format, saving 8–12 min per patient |
| **Patient Health Timeline** | Visual narrative of health journey — not just a record list but a story patients can follow |
| **Plain-language Lab Summaries** | Reduces patient anxiety and unnecessary follow-up calls to nurses |
| **Predictive Risk Scoring** | Proactive triage — surface the highest-risk patients before emergencies happen |
| **PWA Architecture** | No app store required; installable on any phone; works in low-connectivity hospital environments |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Next.js 14 (App Router)            │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │ Doctor UI   │  │  Patient UI  │  │  Auth Pages │ │
│  └──────┬──────┘  └──────┬───────┘  └──────┬──────┘ │
│         │                │                  │        │
│  ┌──────▼──────────────────────────────────▼──────┐  │
│  │              App Router API Routes              │  │
│  │   /api/ai/*   /api/vitals/*   /api/alerts/*    │  │
│  └──────┬──────────────────────────────────┬──────┘  │
└─────────┼──────────────────────────────────┼─────────┘
          │                                  │
┌─────────▼──────────┐          ┌────────────▼──────────┐
│   Supabase          │          │   Claude API           │
│   - Postgres (RLS)  │          │   - Anomaly detection  │
│   - Realtime subs   │          │   - SOAP note gen      │
│   - Auth (JWT)      │          │   - Risk scoring       │
│   - Storage (files) │          │   - Lab summaries      │
└────────────────────┘          └───────────────────────┘
          │
┌─────────▼──────────┐
│  WebSocket Mock     │
│  Vitals Engine      │
│  (simulates IoT     │
│   device feeds)     │
└────────────────────┘
```

---

## Module Summary

| Module | Location | Description |
|---|---|---|
| Auth & Roles | `/app/(auth)/` | Login, signup, role routing |
| Doctor Dashboard | `/app/doctor/` | Ward overview, alerts, patient list |
| Patient Dashboard | `/app/patient/` | Health timeline, meds, reports |
| Vitals Engine | `/lib/vitals/` | WebSocket mock + Supabase realtime |
| AI Layer | `/lib/ai/` | All Claude API integrations |
| Messaging | `/app/messages/` | Secure doctor-patient chat |
| Alerts System | `/lib/alerts/` | Tiered alert logic + notifications |
| Reports | `/app/reports/` | Lab result viewer + AI summaries |

---

## Development Phases

### Phase 1 — Foundation (Week 1–2)
- Supabase schema, RLS policies, auth setup
- Role-based routing (doctor vs patient)
- Base layout, design system, component library

### Phase 2 — Core Doctor Features (Week 3–4)
- Ward dashboard with patient list
- Real-time vitals with simulated WebSocket engine
- Vitals chart visualization (Recharts)
- Alert inbox with severity tiers

### Phase 3 — AI Integration (Week 5–6)
- Anomaly detection pipeline
- Predictive risk scoring
- Voice note → SOAP note generation
- Diagnosis assistance panel

### Phase 4 — Patient Features (Week 7)
- Health timeline UI
- Medication tracker
- Lab report viewer with AI summaries

### Phase 5 — Messaging & Polish (Week 8)
- Secure in-app messaging
- PWA configuration
- Performance optimization
- Accessibility audit
