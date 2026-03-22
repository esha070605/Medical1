# MediPulse — Real-Time Vitals Engine

## Overview
During development, vitals data is simulated by a **mock WebSocket engine** that generates physiologically realistic vital sign readings at configurable intervals. The engine injects data into Supabase, which then broadcasts changes via Supabase Realtime to all subscribed clients.

In production, this layer is replaced by a real device integration (HL7/FHIR adapter), with no UI code changes needed.

---

## Architecture

```
Mock Engine (Node.js server / Next.js API route)
      │
      │  Generates reading every 30s per patient
      ▼
Supabase INSERT → vitals table
      │
      │  Postgres → Realtime → WebSocket broadcast
      ▼
useRealtimeVitals hook (client)
      │
      ▼
VitalsGrid component → VitalsCard (live update)
```

---

## Mock Vitals Generator

**File**: `lib/vitals/mock-engine.ts`

```typescript
import { createClient } from '@/lib/supabase/server';

interface VitalsProfile {
  baseHeartRate: number;
  baseSystolic: number;
  baseDiastolic: number;
  baseSpO2: number;
  baseTemp: number;
  baseRespRate: number;
  condition: 'stable' | 'deteriorating' | 'critical' | 'recovering';
}

// Physiologically realistic variation ranges
const VARIATION = {
  heart_rate: { stable: 3, deteriorating: 8, critical: 15 },
  spo2: { stable: 0.5, deteriorating: 1.5, critical: 3 },
  temperature: { stable: 0.1, deteriorating: 0.3, critical: 0.5 },
  blood_pressure_sys: { stable: 5, deteriorating: 12, critical: 20 },
  blood_pressure_dia: { stable: 3, deteriorating: 8, critical: 14 },
  respiratory_rate: { stable: 1, deteriorating: 3, critical: 6 },
};

function generateReading(base: number, variation: number, trend: number = 0): number {
  const noise = (Math.random() - 0.5) * 2 * variation;
  return Math.round((base + noise + trend) * 10) / 10;
}

// Deteriorating patients gradually worsen over time
function applyTrend(profile: VitalsProfile, minutesElapsed: number): VitalsProfile {
  if (profile.condition !== 'deteriorating') return profile;
  const drift = minutesElapsed * 0.02; // Gradual drift
  return {
    ...profile,
    baseHeartRate: profile.baseHeartRate + drift * 2,
    baseSpO2: profile.baseSpO2 - drift * 0.5,
    baseSystolic: profile.baseSystolic + drift * 1.5,
  };
}

export async function generateMockVitals(
  patientId: string,
  profile: VitalsProfile,
  minutesElapsed: number = 0
): Promise<void> {
  const supabase = createClient();
  const adjusted = applyTrend(profile, minutesElapsed);
  const condition = profile.condition;

  const readings = [
    {
      patient_id: patientId,
      vital_type: 'heart_rate',
      value: generateReading(adjusted.baseHeartRate, VARIATION.heart_rate[condition]),
      unit: 'bpm',
    },
    {
      patient_id: patientId,
      vital_type: 'blood_pressure_sys',
      value: generateReading(adjusted.baseSystolic, VARIATION.blood_pressure_sys[condition]),
      unit: 'mmHg',
    },
    {
      patient_id: patientId,
      vital_type: 'blood_pressure_dia',
      value: generateReading(adjusted.baseDiastolic, VARIATION.blood_pressure_dia[condition]),
      unit: 'mmHg',
    },
    {
      patient_id: patientId,
      vital_type: 'spo2',
      value: generateReading(adjusted.baseSpO2, VARIATION.spo2[condition]),
      unit: '%',
    },
    {
      patient_id: patientId,
      vital_type: 'temperature',
      value: generateReading(adjusted.baseTemp, VARIATION.temperature[condition]),
      unit: '°C',
    },
    {
      patient_id: patientId,
      vital_type: 'respiratory_rate',
      value: generateReading(adjusted.baseRespRate, VARIATION.respiratory_rate[condition]),
      unit: 'breaths/min',
    },
  ];

  const { error } = await supabase.from('vitals').insert(readings);
  if (error) console.error('Mock vitals insert error:', error);
}
```

---

## Normal Ranges & Alert Thresholds

**File**: `lib/vitals/vitals-constants.ts`

```typescript
export const VITAL_RANGES = {
  heart_rate: {
    unit: 'bpm',
    normal: { min: 60, max: 100 },
    warning: { min: 50, max: 120 },
    critical: { min: 40, max: 150 },
  },
  blood_pressure_sys: {
    unit: 'mmHg',
    normal: { min: 90, max: 140 },
    warning: { min: 80, max: 160 },
    critical: { min: 70, max: 180 },
  },
  blood_pressure_dia: {
    unit: 'mmHg',
    normal: { min: 60, max: 90 },
    warning: { min: 50, max: 100 },
    critical: { min: 40, max: 110 },
  },
  spo2: {
    unit: '%',
    normal: { min: 95, max: 100 },
    warning: { min: 90, max: 100 },
    critical: { min: 0, max: 90 },
  },
  temperature: {
    unit: '°C',
    normal: { min: 36.1, max: 37.2 },
    warning: { min: 35.5, max: 38.5 },
    critical: { min: 0, max: 39.5 },
  },
  respiratory_rate: {
    unit: 'breaths/min',
    normal: { min: 12, max: 20 },
    warning: { min: 10, max: 25 },
    critical: { min: 0, max: 30 },
  },
};

export type VitalStatus = 'normal' | 'warning' | 'critical';

export function getVitalStatus(type: keyof typeof VITAL_RANGES, value: number): VitalStatus {
  const ranges = VITAL_RANGES[type];
  if (value < ranges.critical.min || value > ranges.critical.max) return 'critical';
  if (value < ranges.warning.min || value > ranges.warning.max) return 'warning';
  return 'normal';
}
```

---

## Realtime Subscription Hook

**File**: `hooks/useRealtimeVitals.ts`

```typescript
import { useEffect, useState } from 'react';
import { createClient } from '@/lib/supabase/client';
import type { VitalReading } from '@/types/vitals';

export function useRealtimeVitals(patientId: string) {
  const [vitals, setVitals] = useState<Record<string, VitalReading[]>>({});
  const [latestVitals, setLatestVitals] = useState<Record<string, VitalReading>>({});
  const [isConnected, setIsConnected] = useState(false);
  const supabase = createClient();

  useEffect(() => {
    // Load initial vitals (last 60 readings per type)
    loadInitialVitals();

    const channel = supabase
      .channel(`vitals:${patientId}`)
      .on('postgres_changes', {
        event: 'INSERT',
        schema: 'public',
        table: 'vitals',
        filter: `patient_id=eq.${patientId}`,
      }, (payload) => {
        const newReading = payload.new as VitalReading;
        setLatestVitals(prev => ({
          ...prev,
          [newReading.vital_type]: newReading,
        }));
        setVitals(prev => ({
          ...prev,
          [newReading.vital_type]: [
            ...(prev[newReading.vital_type] || []).slice(-59),
            newReading,
          ],
        }));
      })
      .subscribe((status) => {
        setIsConnected(status === 'SUBSCRIBED');
      });

    return () => {
      supabase.removeChannel(channel);
    };
  }, [patientId]);

  return { vitals, latestVitals, isConnected };
}
```

---

## Simulating Different Patient Conditions

Define patient profiles in the mock engine. Each profile represents a clinical scenario:

```typescript
export const PATIENT_PROFILES: Record<string, VitalsProfile> = {
  stable_recovery: {
    baseHeartRate: 72, baseSystolic: 120, baseDiastolic: 80,
    baseSpO2: 98, baseTemp: 36.8, baseRespRate: 16,
    condition: 'stable',
  },
  early_sepsis: {
    baseHeartRate: 105, baseSystolic: 100, baseDiastolic: 65,
    baseSpO2: 94, baseTemp: 38.6, baseRespRate: 22,
    condition: 'deteriorating',
  },
  cardiac_event: {
    baseHeartRate: 135, baseSystolic: 85, baseDiastolic: 55,
    baseSpO2: 91, baseTemp: 37.1, baseRespRate: 26,
    condition: 'critical',
  },
  post_surgery: {
    baseHeartRate: 88, baseSystolic: 115, baseDiastolic: 74,
    baseSpO2: 97, baseTemp: 37.4, baseRespRate: 18,
    condition: 'recovering',
  },
};
```

These are assigned to mock patients in seed data, enabling doctors to see diverse clinical scenarios during development and demos.

---

## API Route for Simulation Control

**File**: `app/api/vitals/simulate/route.ts`

```typescript
// GET /api/vitals/simulate?action=start|stop|status
// Only available in development environment

export async function GET(request: Request) {
  if (process.env.NODE_ENV !== 'development') {
    return Response.json({ error: 'Not available in production' }, { status: 403 });
  }
  // Start/stop the mock vitals generation interval
}
```
