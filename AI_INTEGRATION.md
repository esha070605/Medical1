# MediPulse — AI Integration Guide

## Overview
All AI features use the **Anthropic Claude API** (`claude-sonnet-4-20250514`). Claude is integrated across five distinct clinical intelligence modules. Every AI call is made server-side via Next.js API routes — never from the client directly.

## Setup

```typescript
// lib/ai/claude-client.ts
import Anthropic from '@anthropic-ai/sdk';

export const claude = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY!, // Server-side only
});

export const CLAUDE_MODEL = 'claude-sonnet-4-20250514';
export const MAX_TOKENS = 1024;
```

---

## Module 1: Vitals Anomaly Detection

**Route**: `POST /api/ai/anomaly`

**Trigger**: Called every 5 minutes per monitored patient, or immediately when a vitals reading falls outside threshold.

**Input**:
```typescript
interface AnomalyDetectionInput {
  patientId: string;
  patientAge: number;
  primaryDiagnosis: string;
  recentVitals: {          // Last 30 readings per vital type
    heartRate: number[];
    bloodPressureSys: number[];
    bloodPressureDia: number[];
    spo2: number[];
    temperature: number[];
    respiratoryRate: number[];
    timestamps: string[];
  };
  medications: string[];   // Active medications that affect vitals
}
```

**Prompt Template** (`lib/ai/anomaly-detection.ts`):
```typescript
const systemPrompt = `You are a clinical monitoring AI integrated into a hospital vitals system. 
Your role is to detect abnormal patterns in patient vitals data that may indicate deterioration.
You analyze trends, not just threshold breaches — identify patterns like gradual downward SpO2 trends, 
widening pulse pressure, or heart rate variability changes that precede clinical events.
Always respond in valid JSON only. No preamble or explanation outside the JSON structure.`;

const userPrompt = `Analyze the following vitals time-series for patient anomalies.

Patient context:
- Age: ${input.patientAge}
- Primary diagnosis: ${input.primaryDiagnosis}
- Active medications: ${input.medications.join(', ')}

Vitals data (most recent last):
${JSON.stringify(input.recentVitals, null, 2)}

Normal adult ranges for reference:
- Heart rate: 60-100 bpm
- Systolic BP: 90-140 mmHg
- Diastolic BP: 60-90 mmHg
- SpO2: 95-100%
- Temperature: 36.1-37.2°C
- Respiratory rate: 12-20 breaths/min

Respond with this exact JSON structure:
{
  "anomaliesDetected": boolean,
  "anomalies": [
    {
      "vitalType": string,
      "severity": "critical" | "warning" | "info",
      "pattern": string,
      "clinicalSignificance": string,
      "recommendedAction": string
    }
  ],
  "overallAssessment": string,
  "confidenceScore": number  // 0-1
}`;
```

**Response Handling**: Parse JSON response, store anomaly flags in `vitals` table, trigger alert creation if severity is `critical` or `warning`.

---

## Module 2: Predictive Risk Scoring

**Route**: `POST /api/ai/risk-score`

**Trigger**: Runs every 30 minutes per patient, or on-demand by doctor.

**Input**:
```typescript
interface RiskScoringInput {
  patientId: string;
  demographics: { age: number; gender: string; };
  primaryDiagnosis: string;
  comorbidities: string[];
  currentVitals: Record<string, number>;        // Latest reading per type
  vitalsLastHour: Record<string, number[]>;     // Trend data
  labResults: { name: string; value: number; flag: string; }[];
  medications: string[];
  admissionDaysCount: number;
  predictionWindowHours: 6 | 24;
}
```

**Prompt Template** (`lib/ai/risk-scoring.ts`):
```typescript
const systemPrompt = `You are a clinical deterioration prediction AI. You assess the probability 
that a hospitalized patient will experience significant clinical deterioration (requiring ICU transfer, 
emergency intervention, or rapid response team activation) within a specified time window.
Base your assessment on the pattern of vitals trends, lab results, medications, and clinical context.
Use established clinical scoring concepts (NEWS2, SOFA, MEWS) as guidance but provide your own holistic assessment.
Respond in valid JSON only.`;

const userPrompt = `Predict deterioration risk for this patient within the next ${input.predictionWindowHours} hours.

${JSON.stringify(input, null, 2)}

Respond with:
{
  "riskScore": number,          // 0-100 (100 = highest risk)
  "riskLevel": "low" | "medium" | "high" | "critical",
  "contributingFactors": [
    {
      "factor": string,
      "weight": number,         // 0-1, contribution to overall score
      "description": string
    }
  ],
  "protectiveFactors": [string],
  "recommendedMonitoring": string,
  "escalationThreshold": string, // "If X happens, escalate immediately"
  "reasoning": string
}`;
```

---

## Module 3: Voice → SOAP Note Generator

**Route**: `POST /api/ai/soap-note`

**Trigger**: Doctor clicks "Generate SOAP Note" after voice recording completes.

**Input**:
```typescript
interface SOAPNoteInput {
  transcript: string;        // Raw voice transcript from Web Speech API
  patientContext: {
    name: string;
    age: number;
    primaryDiagnosis: string;
    currentMedications: string[];
    recentVitals: Record<string, number>;
    lastLabSummary?: string;
  };
  noteType: 'soap' | 'progress' | 'admission';
}
```

**Prompt Template** (`lib/ai/soap-generator.ts`):
```typescript
const systemPrompt = `You are a clinical documentation AI that converts physician voice transcripts 
into structured medical notes. You extract and organize clinical information from natural, conversational 
speech into the appropriate medical documentation format. 
Be precise with medical terminology. Do not add clinical information not present in the transcript. 
If information for a SOAP section is absent from the transcript, note "[Not documented in this visit]".
Respond in valid JSON only.`;

const userPrompt = `Convert this physician voice transcript into a structured SOAP note.

Patient context:
${JSON.stringify(input.patientContext, null, 2)}

Voice transcript:
"${input.transcript}"

Respond with:
{
  "noteType": "soap",
  "subjective": string,     // Patient's reported symptoms, complaints, history from transcript
  "objective": string,      // Examination findings, vitals referenced, lab values mentioned
  "assessment": string,     // Clinical assessment and diagnosis discussion
  "plan": string,           // Treatment plan, orders, follow-up
  "suggestedTitle": string, // Brief note title e.g. "Day 3 Progress Note"
  "missingElements": [string], // Elements a complete note should have but weren't mentioned
  "confidenceScore": number  // 0-1, how complete the transcript was
}`;
```

**Voice Recording Hook** (`hooks/useVoiceRecorder.ts`):
```typescript
// Uses Web Speech API for browser-native transcription
const recognition = new (window.SpeechRecognition || window.webkitSpeechRecognition)();
recognition.continuous = true;
recognition.interimResults = true;
recognition.lang = 'en-US';
```

---

## Module 4: Diagnosis Assistance

**Route**: `POST /api/ai/diagnosis`

**Trigger**: Doctor opens AI Assist panel and submits patient context.

**Input**:
```typescript
interface DiagnosisAssistInput {
  chiefComplaint: string;
  symptoms: string[];
  vitals: Record<string, number>;
  labResults: { name: string; value: string; flag: string; }[];
  currentMedications: string[];
  pastMedicalHistory: string;
  patientAge: number;
  patientGender: string;
  doctorQuestion?: string;  // Optional specific question from doctor
}
```

**Prompt Template** (`lib/ai/diagnosis-assist.ts`):
```typescript
const systemPrompt = `You are a clinical decision support AI assisting a licensed physician. 
Your role is to surface relevant differential diagnoses, flag potential drug interactions, 
highlight concerning lab patterns, and suggest evidence-based next steps.
You do NOT make final diagnoses — you assist and inform the physician's clinical judgment.
Always include a disclaimer that this is AI-assisted support, not a diagnosis.
Respond in valid JSON only.`;

const userPrompt = `Provide clinical decision support for this case.
${input.doctorQuestion ? `Doctor's specific question: "${input.doctorQuestion}"` : ''}

Patient data:
${JSON.stringify(input, null, 2)}

Respond with:
{
  "differentialDiagnoses": [
    {
      "diagnosis": string,
      "likelihood": "high" | "medium" | "low",
      "supportingEvidence": [string],
      "againstEvidence": [string],
      "suggestedWorkup": [string]
    }
  ],
  "urgentFlags": [string],          // Anything requiring immediate attention
  "medicationConcerns": [string],   // Interactions, contraindications
  "suggestedOrders": [string],      // Recommended tests or interventions
  "clinicalPearls": [string],       // Relevant clinical notes
  "disclaimer": string              // Always include AI limitation disclaimer
}`;
```

---

## Module 5: Lab Report Plain-Language Summary

**Route**: `POST /api/ai/lab-summary`

**Trigger**: On-demand when patient views lab results, or when doctor first views a new lab result.

**Input**:
```typescript
interface LabSummaryInput {
  testName: string;
  testCategory: string;
  results: {
    parameter: string;
    value: string;
    unit: string;
    referenceRange: string;
    flag: 'H' | 'L' | 'HH' | 'LL' | 'normal';
  }[];
  patientAge: number;
  forAudience: 'patient' | 'doctor';  // Changes language complexity
}
```

**Prompt Template** (`lib/ai/lab-summarizer.ts`):
```typescript
const patientSystemPrompt = `You summarize medical lab results in plain, reassuring language 
for patients with no medical background. Avoid jargon. Use analogies where helpful. 
Be factual and accurate — do not minimize genuinely concerning results, but frame them calmly. 
Always note that the patient should discuss results with their doctor. Respond in valid JSON only.`;

const userPrompt = `Summarize these lab results for a ${input.forAudience === 'patient' ? 'patient (no medical background)' : 'physician'}.

${JSON.stringify(input.results, null, 2)}

Respond with:
{
  "overallStatus": "normal" | "attention_needed" | "concerning",
  "summary": string,              // 2-3 sentence plain language overview
  "abnormalFindings": [
    {
      "parameter": string,
      "plainLanguageExplanation": string,
      "whatItMeans": string
    }
  ],
  "normalFindings": string,       // Brief mention of what's in range
  "followUpNote": string,         // e.g. "Discuss with your doctor at your next visit"
  "urgency": "routine" | "soon" | "urgent"
}`;
```

---

## AI Response Caching Strategy

| Module | Cache Duration | Cache Key |
|---|---|---|
| Anomaly Detection | No cache (always fresh) | — |
| Risk Score | 30 minutes | `risk:${patientId}:${windowHours}` |
| SOAP Note | No cache (one-time generation) | — |
| Diagnosis Assist | No cache | — |
| Lab Summary | Until new lab results | `lab-summary:${labResultId}` |

Store cached AI responses in Supabase (`ai_summary` column on `lab_results`, `ai_reasoning` on `risk_scores`).

---

## Error Handling

```typescript
// lib/ai/claude-client.ts
export async function callClaude(
  systemPrompt: string,
  userPrompt: string,
  maxTokens = MAX_TOKENS
): Promise<string> {
  try {
    const response = await claude.messages.create({
      model: CLAUDE_MODEL,
      max_tokens: maxTokens,
      system: systemPrompt,
      messages: [{ role: 'user', content: userPrompt }],
    });

    const text = response.content
      .filter(block => block.type === 'text')
      .map(block => block.text)
      .join('');

    return text;
  } catch (error) {
    console.error('Claude API error:', error);
    throw new Error('AI service temporarily unavailable');
  }
}
```

## AI Disclaimer UI Rule

Every AI-generated output in the UI **must** display:
```
⚠️ AI-assisted — for clinical support only. Not a substitute for physician judgment.
```
Use the `<AIDisclaimer />` component which renders this badge consistently.
