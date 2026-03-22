# MediPulse — Alerts System

## Overview
The alerts system is the **clinical safety backbone** of MediPulse. It generates, prioritizes, delivers, and tracks clinical alerts for doctors based on patient vitals, AI analysis, and medication events.

---

## Alert Severity Tiers

| Tier | Color | Sound | Delivery | Response Time |
|---|---|---|---|---|
| **Critical** | `#FF4444` Red | Urgent tone | Banner + Push + Inbox | Immediate |
| **Warning** | `#FFB800` Amber | Alert chime | Push + Inbox | < 15 min |
| **Info** | `#00D4FF` Cyan | Silent | Inbox only | Routine |

---

## Alert Sources

### 1. Threshold Breach Alerts (Automated)
Generated in `lib/alerts/alert-engine.ts` whenever a new vitals reading is inserted.

```typescript
export async function checkVitalsThreshold(reading: VitalReading): Promise<void> {
  const status = getVitalStatus(reading.vital_type, reading.value);
  if (status === 'normal') return;

  const severity = status === 'critical' ? 'critical' : 'warning';

  // Deduplicate: don't create a new alert if one already exists for this patient/vital
  const existing = await getActiveAlertForVital(reading.patient_id, reading.vital_type);
  if (existing) return;

  await createAlert({
    patientId: reading.patient_id,
    severity,
    title: `${formatVitalName(reading.vital_type)} ${status === 'critical' ? 'Critical' : 'Abnormal'}`,
    description: `${formatVitalName(reading.vital_type)} reading of ${reading.value}${reading.unit} is outside ${status} range.`,
    vitalType: reading.vital_type,
    triggerData: reading,
  });
}
```

### 2. AI Anomaly Alerts
Generated when `POST /api/ai/anomaly` returns anomalies. The AI can detect pattern-based issues that threshold checks miss (e.g., gradual SpO2 downtrend that hasn't breached threshold yet).

```typescript
// Each anomaly in the AI response creates an alert if severity >= 'warning'
for (const anomaly of aiResponse.anomalies) {
  if (anomaly.severity === 'info') continue;
  await createAlert({
    ...baseAlertData,
    title: `AI Detection: ${anomaly.pattern}`,
    description: anomaly.clinicalSignificance,
    aiExplanation: anomaly.recommendedAction,
    severity: anomaly.severity,
  });
}
```

### 3. Risk Score Jump Alerts
Generated when a patient's risk score increases by 15+ points in a 30-minute window.

```typescript
export async function checkRiskScoreChange(
  patientId: string,
  newScore: number,
  previousScore: number
): Promise<void> {
  const delta = newScore - previousScore;
  if (delta < 15) return;

  const severity = newScore > 75 ? 'critical' : 'warning';
  await createAlert({
    patientId,
    severity,
    title: `Rapid Risk Score Increase`,
    description: `Patient deterioration risk jumped from ${previousScore} to ${newScore} in the last 30 minutes.`,
    riskScore: newScore,
  });
}
```

### 4. Medication Alerts
Generated when a critical medication dose is missed.

```typescript
// Checked every 15 minutes via scheduled job
// A dose is flagged if: status = 'pending' AND scheduled_at is > 30 min ago
// Only for medications tagged as 'critical' (e.g. insulin, cardiac meds)
```

---

## Alert Lifecycle

```
CREATED (active)
    │
    ├── Doctor views alert → status stays 'active'
    │
    ├── Doctor clicks Acknowledge → status = 'acknowledged', acknowledged_at set
    │
    └── Patient condition improves → status = 'resolved', resolved_at set
         (resolved automatically when vitals return to normal range for 3 consecutive readings)
```

---

## Alert Deduplication Rules

- One active alert per patient + vital_type combination (threshold alerts)
- One active alert per patient + AI anomaly pattern combination
- Risk score alerts: max 1 per patient per 30-minute window
- Medication alerts: 1 per missed dose per patient

---

## UI Components

### `AlertBanner` — Top of all doctor pages
```typescript
// Shows when any critical alert exists for any assigned patient
// Fixed position, cannot be dismissed (only resolves when alert resolves)
// Click → navigates to alert inbox filtered to critical
```

### `AlertInbox` — `/doctor/alerts`
```typescript
// Full list with tabs: All | Active | Acknowledged
// Sorted: critical first, then by timestamp DESC
// Real-time updates via Supabase subscription
```

### `AlertItem`
```typescript
// Single alert row with:
// - Severity icon + color-coded left border
// - Patient name → clickable link to patient profile
// - Alert title + truncated description
// - Timestamp (relative)
// - Expandable: full description + AI explanation
// - Action buttons: View Patient | Acknowledge
```

### `AlertBadge` — In sidebar nav
```typescript
// Red circle with count of active critical alerts
// Yellow circle with count of active warning alerts
// Updates in real time
```

---

## Push Notifications

```typescript
// lib/alerts/notification.ts
export async function sendBrowserNotification(alert: Alert): Promise<void> {
  if (!('Notification' in window)) return;
  if (Notification.permission !== 'granted') return;

  const notification = new Notification(`MediPulse Alert — ${alert.severity.toUpperCase()}`, {
    body: `${alert.patientName}: ${alert.title}`,
    icon: '/icons/alert-icon.png',
    badge: '/icons/badge.png',
    tag: alert.id,         // Deduplicates: same tag = replace existing notification
    requireInteraction: alert.severity === 'critical', // Critical stays until dismissed
  });

  notification.onclick = () => {
    window.focus();
    router.push(`/doctor/patients/${alert.patientId}`);
  };
}
```

Permission is requested on doctor's first login. If denied, show a persistent in-app banner suggesting they enable notifications for patient safety.
