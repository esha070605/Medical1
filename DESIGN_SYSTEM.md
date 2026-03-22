# MediPulse — Design System

## Aesthetic Direction
**Dark clinical precision** — MediPulse uses a deep navy dark theme that evokes trust, professionalism, and calm. Electric cyan accents provide high-contrast data highlights. The interface feels like a premium medical instrument, not a generic SaaS dashboard.

Patient-facing views shift to a softer, lighter variant — more approachable, less clinical.

---

## Color Palette

```css
:root {
  /* Base */
  --bg-primary: #0A1628;        /* Deep navy — main backgrounds */
  --bg-secondary: #0F1F3D;      /* Slightly lighter navy — cards */
  --bg-tertiary: #162040;       /* Elevated surfaces */
  --bg-glass: rgba(255,255,255,0.05); /* Glassmorphism card bg */

  /* Accent */
  --accent-primary: #00D4FF;    /* Electric cyan — live data, CTAs */
  --accent-secondary: #0094CC;  /* Deeper cyan — hover states */
  --accent-glow: rgba(0,212,255,0.15); /* Glow effect for live indicators */

  /* Text */
  --text-primary: #F0F4FF;      /* Near white — primary text */
  --text-secondary: #8B9CC8;    /* Muted blue-gray — secondary text */
  --text-tertiary: #4A5A80;     /* Faint — placeholders, disabled */

  /* Vitals Status */
  --status-normal: #00C853;     /* Green */
  --status-warning: #FFB800;    /* Amber */
  --status-critical: #FF4444;   /* Red */
  --status-info: #00D4FF;       /* Cyan */

  /* Borders */
  --border-subtle: rgba(255,255,255,0.08);
  --border-active: rgba(0,212,255,0.4);

  /* Patient theme overrides */
  --patient-bg: #F4F7FF;
  --patient-card: #FFFFFF;
  --patient-accent: #0061FF;
  --patient-text: #1A2340;
}
```

---

## Typography

```typescript
// next/font setup in app/layout.tsx
import { DM_Sans, JetBrains_Mono } from 'next/font/google';

const dmSans = DM_Sans({
  subsets: ['latin'],
  variable: '--font-sans',
  weight: ['300', '400', '500', '600', '700'],
});

const jetbrainsMono = JetBrains_Mono({
  subsets: ['latin'],
  variable: '--font-mono',
  weight: ['400', '500', '600'],
});
```

### Usage Rules
| Use | Font | Weight | Size |
|---|---|---|---|
| Headings (H1) | DM Sans | 700 | 2rem |
| Headings (H2) | DM Sans | 600 | 1.5rem |
| Body text | DM Sans | 400 | 0.875rem |
| Labels/captions | DM Sans | 500 | 0.75rem |
| Vital numbers (live) | JetBrains Mono | 600 | 1.75–2.5rem |
| Data tables | JetBrains Mono | 400 | 0.8rem |
| Code/technical | JetBrains Mono | 400 | 0.875rem |

---

## Component Patterns

### Glass Card
Used for all primary content cards in doctor UI.
```css
.glass-card {
  background: var(--bg-glass);
  backdrop-filter: blur(12px);
  border: 1px solid var(--border-subtle);
  border-radius: 12px;
  padding: 1.25rem;
}
```

```typescript
// components/ui/glass-card.tsx
export function GlassCard({ children, className, ...props }) {
  return (
    <div
      className={cn(
        "bg-white/5 backdrop-blur-md border border-white/10 rounded-xl p-5",
        className
      )}
      {...props}
    >
      {children}
    </div>
  );
}
```

### LIVE Indicator
Used on any component receiving real-time data.
```typescript
// components/vitals/LiveVitalsBadge.tsx
export function LiveVitalsBadge({ isConnected }: { isConnected: boolean }) {
  if (!isConnected) return <span className="text-xs text-white/30">Offline</span>;
  return (
    <span className="flex items-center gap-1.5 text-xs font-mono text-cyan-400">
      <span className="relative flex h-2 w-2">
        <span className="animate-ping absolute h-full w-full rounded-full bg-cyan-400 opacity-75" />
        <span className="relative rounded-full h-2 w-2 bg-cyan-400" />
      </span>
      LIVE
    </span>
  );
}
```

### Vital Card
```typescript
// components/vitals/VitalsCard.tsx
// Props: vitalType, currentValue, unit, trend, status, sparklineData, isLive
// Displays large monospace number with status color
// Trend arrow: ↑ (red for HR/Temp/RR, green for SpO2) | ↓ reverse | → neutral
// Mini sparkline (Recharts ResponsiveContainer, no axes, just the line)
```

### Risk Score Gauge
```typescript
// components/ai/RiskScoreCard.tsx
// Uses SVG arc for circular gauge
// Score zones: 0-30 (green), 31-60 (yellow), 61-80 (orange), 81-100 (red)
// Animated: score animates from previous value on update
// Shows top 3 contributing factors below gauge
```

---

## Spacing Scale (Tailwind)
Follow Tailwind's default scale. Additional conventions:
- Card padding: `p-5` (1.25rem)
- Section gaps: `gap-4` or `gap-6`
- Page padding (mobile): `px-4 py-5`
- Page padding (desktop): `px-8 py-6`
- Between related elements: `gap-2` or `gap-3`
- Between sections: `gap-6` or `gap-8`

---

## Animation Patterns

All animations use Tailwind + Framer Motion:

```typescript
// Page entrance: staggered fade-up
const containerVariants = {
  hidden: {},
  show: { transition: { staggerChildren: 0.08 } }
};
const itemVariants = {
  hidden: { opacity: 0, y: 16 },
  show: { opacity: 1, y: 0, transition: { duration: 0.3, ease: 'easeOut' } }
};

// Live data update flash
// When a vitals card value changes: brief cyan background flash
// CSS: transition-colors duration-300, flash class applied then removed
```

### Specific Animation Rules
| Element | Animation |
|---|---|
| Page load | Staggered fade-up (0.08s between items) |
| New vitals reading | Value crossfade + card edge glow |
| Critical alert arrival | Alert banner slides down, card pulses red |
| Risk score change | Gauge needle animates smoothly to new value |
| SOAP note generation | Skeleton loader → content reveal with typewriter effect |
| Message received | Bubble slides in from left with bounce easing |

---

## Mobile-First Rules

1. **Touch targets**: Minimum 44×44px for all interactive elements
2. **Bottom navigation**: Doctor has 4-tab bottom nav on mobile (Dashboard, Patients, Alerts, Messages)
3. **Swipe gestures**: Patient health timeline supports swipe-up for more events
4. **Thumb zone**: Primary actions always in bottom 40% of screen on mobile
5. **Font sizes**: Minimum 14px body text, 16px inputs (prevents iOS zoom)
6. **Overflow**: Vitals sparklines scroll horizontally on mobile (hidden scrollbar)

---

## Accessibility

- All color-coded status (normal/warning/critical) must ALSO use icon + text label (not color alone)
- Focus visible styles: `ring-2 ring-cyan-400 ring-offset-2 ring-offset-navy-900`
- ARIA labels on all icon-only buttons
- Vitals chart must include `aria-label` with current value
- Alert severity announced to screen readers via `role="alert"` on critical banners

---

## Tailwind Config Extensions

```typescript
// tailwind.config.ts
export default {
  theme: {
    extend: {
      colors: {
        navy: {
          900: '#0A1628',
          800: '#0F1F3D',
          700: '#162040',
        },
        cyan: {
          400: '#00D4FF',
          500: '#0094CC',
        },
      },
      fontFamily: {
        sans: ['var(--font-sans)', 'system-ui'],
        mono: ['var(--font-mono)', 'monospace'],
      },
      animation: {
        'pulse-cyan': 'pulse-cyan 2s ease-in-out infinite',
        'slide-down': 'slide-down 0.3s ease-out',
      },
      keyframes: {
        'pulse-cyan': {
          '0%, 100%': { boxShadow: '0 0 0 0 rgba(0, 212, 255, 0)' },
          '50%': { boxShadow: '0 0 0 8px rgba(0, 212, 255, 0.15)' },
        },
        'slide-down': {
          from: { transform: 'translateY(-100%)', opacity: '0' },
          to: { transform: 'translateY(0)', opacity: '1' },
        },
      },
    },
  },
};
```

---

## shadcn/ui Component Overrides

All shadcn components are installed with `--style=default` and then overridden in `components/ui/` to match the dark navy theme. Key overrides:

- `Button`: Primary variant uses `bg-cyan-400 text-navy-900 hover:bg-cyan-500`
- `Card`: Replaced with `GlassCard` pattern
- `Badge`: Custom severity variants (critical/warning/info/normal)
- `Input`: `bg-white/5 border-white/10 text-white placeholder:text-white/30`
- `Dialog`: `bg-navy-800 border border-white/10`
