# Mortgage Advisor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an AI-powered Hebrew mortgage simulator with a 4-step wizard, Shpitzer calculations, Claude API insights, bank recommendations, and inquiry letter generation.

**Architecture:** Next.js 14 App Router with TypeScript. Calculation logic lives in `lib/calculations.ts` (pure functions, no AI). Claude API is called from `app/api/ai-insights/route.ts` server-side. State flows through URL search params from wizard → results → letter pages.

**Tech Stack:** Next.js 14, TypeScript, Tailwind CSS, shadcn/ui, Anthropic SDK, Vercel

---

## File Map

| File | Responsibility |
|------|---------------|
| `lib/types.ts` | All shared TypeScript interfaces |
| `lib/calculations.ts` | Shpitzer formula, LTV, DTI, sensitivity |
| `lib/banks.ts` | Bank data, contact links, recommendation logic |
| `lib/ai-prompt.ts` | Claude prompt builder |
| `app/api/calculate/route.ts` | POST endpoint — runs calculations |
| `app/api/ai-insights/route.ts` | POST endpoint — calls Claude API |
| `app/api/ai-mix/route.ts` | POST endpoint — AI recommends תמהיל |
| `app/page.tsx` | Landing page |
| `app/simulator/page.tsx` | Wizard shell + state |
| `components/wizard/StepProfile.tsx` | Step 1 — borrower profile |
| `components/wizard/StepProperty.tsx` | Step 2 — property details |
| `components/wizard/StepPreferences.tsx` | Step 3 — preferences |
| `components/wizard/StepMix.tsx` | Step 4 — mortgage mix |
| `components/wizard/WizardProgress.tsx` | Progress bar + step nav |
| `app/results/page.tsx` | Results shell — fetches AI insights |
| `components/results/SummaryCard.tsx` | Monthly payment + total cost overview |
| `components/results/TrackTable.tsx` | Per-track breakdown table |
| `components/results/InsightsPanel.tsx` | AI insights display |
| `components/results/BankCards.tsx` | Bank recommendation cards |
| `app/letter/[bankId]/page.tsx` | Inquiry letter + copy + contact link |

---

## Task 1: Project Scaffold

**Files:**
- Create: `package.json`, `tsconfig.json`, `tailwind.config.ts`, `next.config.ts`
- Create: `.env.local.example`
- Create: `.gitignore`

- [ ] **Step 1: Scaffold Next.js project**

```bash
cd /c/Users/Itamar/MyProjects/mortgage-advisor
npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir=no --import-alias="@/*" --yes
```

Expected: project files created, `node_modules` installed.

- [ ] **Step 2: Install dependencies**

```bash
npm install @anthropic-ai/sdk
npm install class-variance-authority clsx tailwind-merge lucide-react
npx shadcn@latest init --yes --base-color slate
npx shadcn@latest add button input label select card progress badge separator
```

- [ ] **Step 3: Create `.env.local.example`**

```bash
cat > .env.local.example << 'EOF'
ANTHROPIC_API_KEY=your_key_here
EOF
cp .env.local.example .env.local
```

- [ ] **Step 4: Configure RTL in `app/layout.tsx`**

Replace the contents of `app/layout.tsx` with:

```tsx
import type { Metadata } from 'next'
import { Rubik } from 'next/font/google'
import './globals.css'

const rubik = Rubik({ subsets: ['hebrew', 'latin'] })

export const metadata: Metadata = {
  title: 'יועץ משכנתא AI',
  description: 'סימולטור משכנתא חכם עם המלצות AI',
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="he" dir="rtl">
      <body className={rubik.className}>{children}</body>
    </html>
  )
}
```

- [ ] **Step 5: Add RTL base styles to `app/globals.css`**

Add after the existing Tailwind directives:

```css
* {
  box-sizing: border-box;
}
body {
  direction: rtl;
  text-align: right;
}
```

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: scaffold Next.js project with RTL Hebrew setup"
```

---

## Task 2: TypeScript Types

**Files:**
- Create: `lib/types.ts`

- [ ] **Step 1: Write `lib/types.ts`**

```typescript
// lib/types.ts

export type EmploymentType = 'salaried' | 'self_employed' | 'mixed'
export type TransactionType = 'first_apartment' | 'investor' | 'mehir_lamishtaken' | 'upgrade'
export type RiskLevel = 'conservative' | 'balanced' | 'aggressive'
export type Priority = 'low_monthly' | 'low_total' | 'max_flexibility'
export type TrackType = 'prime' | 'fixed_cpi' | 'fixed_no_cpi' | 'variable_cpi' | 'variable_no_cpi' | 'zchaut' | 'grace'

export interface BorrowerProfile {
  age: number
  isCouple: boolean
  monthlyIncome: number       // net, combined if couple
  employmentType: EmploymentType
  isEligible: boolean         // זכאות
}

export interface PropertyDetails {
  propertyPrice: number
  downPayment: number         // absolute amount
  transactionType: TransactionType
  region: string              // e.g. "מרכז", "צפון", "דרום", "ירושלים"
}

export interface Preferences {
  loanTermYears: number       // 10–30
  riskLevel: RiskLevel
  priority: Priority
}

export interface MixTrack {
  type: TrackType
  amount: number              // absolute NIS
  rate: number                // annual %, e.g. 4.25
  isCpiLinked: boolean
  graceYears?: number         // only for grace track
}

export interface MortgageMix {
  tracks: MixTrack[]
  totalLoan: number
}

export interface WizardData {
  profile: BorrowerProfile
  property: PropertyDetails
  preferences: Preferences
  mix: MortgageMix
}

// Calculation outputs
export interface TrackResult {
  type: TrackType
  amount: number
  rate: number
  isCpiLinked: boolean
  monthlyPayment: number
  totalPayment: number
  totalInterest: number
}

export interface SensitivityResult {
  primePlusOne: number        // new total monthly if prime +1%
  primePlusTwo: number        // new total monthly if prime +2%
}

export interface ValidationResult {
  ltvValid: boolean
  ltvPercent: number
  ltvMax: number
  dtiPercent: number
  dtiWarning: boolean         // true if > 40%
  variableUnlinkedPercent: number
  variableUnlinkedWarning: boolean  // true if > 33%
}

export interface CalculationResult {
  tracks: TrackResult[]
  totalMonthlyPayment: number
  totalLoanCost: number
  sensitivity: SensitivityResult
  validation: ValidationResult
}

// AI outputs
export type InsightType = 'success' | 'warning' | 'info' | 'bank'

export interface Insight {
  type: InsightType
  text: string
}

export interface RecommendedBank {
  bankId: string
  reason: string
}

export interface AIInsightsResult {
  profileAnalysis: string
  mixReasoning: string
  insights: Insight[]         // exactly 3
  recommendedBanks: RecommendedBank[]  // 2–3 banks
}

// Bank data
export interface BankContactChannel {
  label: string               // e.g. "טופס אונליין"
  url: string
  type: 'form' | 'whatsapp' | 'chat' | 'phone'
}

export interface Bank {
  id: string
  name: string
  shortName: string
  channels: BankContactChannel[]
  primaryChannel: BankContactChannel
}
```

- [ ] **Step 2: Commit**

```bash
git add lib/types.ts
git commit -m "feat: add shared TypeScript types"
```

---

## Task 3: Calculation Engine

**Files:**
- Create: `lib/calculations.ts`
- Create: `lib/calculations.test.ts`

- [ ] **Step 1: Install test runner**

```bash
npm install --save-dev vitest @vitest/coverage-v8
```

Add to `package.json` scripts:
```json
"test": "vitest run",
"test:watch": "vitest"
```

Add `vitest.config.ts`:
```typescript
import { defineConfig } from 'vitest/config'
export default defineConfig({
  test: { environment: 'node' }
})
```

- [ ] **Step 2: Write failing tests**

Create `lib/calculations.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import {
  shpitzerMonthlyPayment,
  calculateTrack,
  calculateTotalMonthly,
  calculateSensitivity,
  validateMix,
} from './calculations'
import type { MixTrack, MortgageMix, BorrowerProfile, PropertyDetails } from './types'

describe('shpitzerMonthlyPayment', () => {
  it('calculates correct monthly payment for simple loan', () => {
    // 1,000,000 NIS, 5% annual, 20 years
    const result = shpitzerMonthlyPayment(1_000_000, 5, 20)
    expect(result).toBeCloseTo(6599.56, 0)
  })

  it('returns principal/term for 0% rate', () => {
    const result = shpitzerMonthlyPayment(1_200_000, 0, 20)
    expect(result).toBeCloseTo(5000, 0)
  })
})

describe('calculateTrack', () => {
  it('returns correct track result for fixed non-cpi track', () => {
    const track: MixTrack = {
      type: 'fixed_no_cpi',
      amount: 500_000,
      rate: 5.5,
      isCpiLinked: false,
    }
    const result = calculateTrack(track, 25)
    expect(result.monthlyPayment).toBeCloseTo(3064, 0)
    expect(result.totalPayment).toBeGreaterThan(500_000)
    expect(result.totalInterest).toBeCloseTo(result.totalPayment - 500_000, 0)
  })
})

describe('calculateTotalMonthly', () => {
  it('sums monthly payments across all tracks', () => {
    const tracks: MixTrack[] = [
      { type: 'prime', amount: 400_000, rate: 4.25, isCpiLinked: false },
      { type: 'fixed_no_cpi', amount: 600_000, rate: 5.5, isCpiLinked: false },
    ]
    const mix: MortgageMix = { tracks, totalLoan: 1_000_000 }
    const total = calculateTotalMonthly(mix, 25)
    expect(total).toBeGreaterThan(4000)
    expect(total).toBeLessThan(8000)
  })
})

describe('calculateSensitivity', () => {
  it('increases monthly payment when prime rises', () => {
    const tracks: MixTrack[] = [
      { type: 'prime', amount: 600_000, rate: 4.25, isCpiLinked: false },
      { type: 'fixed_no_cpi', amount: 400_000, rate: 5.5, isCpiLinked: false },
    ]
    const mix: MortgageMix = { tracks, totalLoan: 1_000_000 }
    const base = calculateTotalMonthly(mix, 25)
    const sensitivity = calculateSensitivity(mix, 25)
    expect(sensitivity.primePlusOne).toBeGreaterThan(base)
    expect(sensitivity.primePlusTwo).toBeGreaterThan(sensitivity.primePlusOne)
  })
})

describe('validateMix', () => {
  it('flags LTV violation for investor > 50%', () => {
    const property: PropertyDetails = {
      propertyPrice: 2_000_000,
      downPayment: 800_000,    // 60% LTV — exceeds 50% investor limit
      transactionType: 'investor',
      region: 'מרכז',
    }
    const mix: MortgageMix = {
      tracks: [{ type: 'fixed_no_cpi', amount: 1_200_000, rate: 5.5, isCpiLinked: false }],
      totalLoan: 1_200_000,
    }
    const profile: BorrowerProfile = {
      age: 40, isCouple: false, monthlyIncome: 25_000,
      employmentType: 'salaried', isEligible: false,
    }
    const result = validateMix(mix, property, profile)
    expect(result.ltvValid).toBe(false)
    expect(result.ltvPercent).toBeCloseTo(60, 0)
  })

  it('warns when DTI exceeds 40%', () => {
    const property: PropertyDetails = {
      propertyPrice: 2_000_000,
      downPayment: 500_000,
      transactionType: 'first_apartment',
      region: 'מרכז',
    }
    const mix: MortgageMix = {
      tracks: [{ type: 'fixed_no_cpi', amount: 1_500_000, rate: 5.5, isCpiLinked: false }],
      totalLoan: 1_500_000,
    }
    const profile: BorrowerProfile = {
      age: 30, isCouple: false, monthlyIncome: 10_000,
      employmentType: 'salaried', isEligible: false,
    }
    const result = validateMix(mix, property, profile)
    expect(result.dtiWarning).toBe(true)
  })
})
```

- [ ] **Step 3: Run tests — verify they fail**

```bash
npm test
```

Expected: multiple failures — `Cannot find module './calculations'`

- [ ] **Step 4: Implement `lib/calculations.ts`**

```typescript
// lib/calculations.ts
import type {
  MixTrack, MortgageMix, TrackResult, CalculationResult,
  SensitivityResult, ValidationResult, BorrowerProfile, PropertyDetails,
} from './types'

/** Shpitzer (annuity) monthly payment */
export function shpitzerMonthlyPayment(
  principal: number,
  annualRatePercent: number,
  termYears: number
): number {
  if (annualRatePercent === 0) return principal / (termYears * 12)
  const r = annualRatePercent / 100 / 12
  const n = termYears * 12
  return principal * (r * Math.pow(1 + r, n)) / (Math.pow(1 + r, n) - 1)
}

export function calculateTrack(track: MixTrack, termYears: number): TrackResult {
  const monthly = shpitzerMonthlyPayment(track.amount, track.rate, termYears)
  const totalPayment = monthly * termYears * 12
  const totalInterest = totalPayment - track.amount
  return {
    type: track.type,
    amount: track.amount,
    rate: track.rate,
    isCpiLinked: track.isCpiLinked,
    monthlyPayment: monthly,
    totalPayment,
    totalInterest,
  }
}

export function calculateTotalMonthly(mix: MortgageMix, termYears: number): number {
  return mix.tracks.reduce((sum, t) => sum + shpitzerMonthlyPayment(t.amount, t.rate, termYears), 0)
}

export function calculateSensitivity(mix: MortgageMix, termYears: number): SensitivityResult {
  const bumpedMix = (bump: number): MortgageMix => ({
    ...mix,
    tracks: mix.tracks.map(t =>
      t.type === 'prime' ? { ...t, rate: t.rate + bump } : t
    ),
  })
  return {
    primePlusOne: calculateTotalMonthly(bumpedMix(1), termYears),
    primePlusTwo: calculateTotalMonthly(bumpedMix(2), termYears),
  }
}

const LTV_LIMITS: Record<string, number> = {
  first_apartment: 75,
  investor: 50,
  mehir_lamishtaken: 90,
  upgrade: 70,
}

export function validateMix(
  mix: MortgageMix,
  property: PropertyDetails,
  profile: BorrowerProfile
): ValidationResult {
  const ltvPercent = (mix.totalLoan / property.propertyPrice) * 100
  const ltvMax = LTV_LIMITS[property.transactionType] ?? 75
  const ltvValid = ltvPercent <= ltvMax

  const monthlyPayment = calculateTotalMonthly(mix, 25) // use 25y for DTI check
  const dtiPercent = (monthlyPayment / profile.monthlyIncome) * 100
  const dtiWarning = dtiPercent > 40

  const variableUnlinkedAmount = mix.tracks
    .filter(t => t.type === 'variable_no_cpi')
    .reduce((sum, t) => sum + t.amount, 0)
  const variableUnlinkedPercent = (variableUnlinkedAmount / mix.totalLoan) * 100
  const variableUnlinkedWarning = variableUnlinkedPercent > 33.33

  return { ltvValid, ltvPercent, ltvMax, dtiPercent, dtiWarning, variableUnlinkedPercent, variableUnlinkedWarning }
}

export function calculateAll(
  mix: MortgageMix,
  termYears: number,
  property: PropertyDetails,
  profile: BorrowerProfile
): CalculationResult {
  const tracks = mix.tracks.map(t => calculateTrack(t, termYears))
  const totalMonthlyPayment = tracks.reduce((s, t) => s + t.monthlyPayment, 0)
  const totalLoanCost = tracks.reduce((s, t) => s + t.totalPayment, 0)
  const sensitivity = calculateSensitivity(mix, termYears)
  const validation = validateMix(mix, property, profile)
  return { tracks, totalMonthlyPayment, totalLoanCost, sensitivity, validation }
}
```

- [ ] **Step 5: Run tests — verify they pass**

```bash
npm test
```

Expected: all tests PASS

- [ ] **Step 6: Commit**

```bash
git add lib/calculations.ts lib/calculations.test.ts vitest.config.ts package.json
git commit -m "feat: add calculation engine with Shpitzer, LTV, DTI, sensitivity"
```

---

## Task 4: Bank Data + Prompt Builder

**Files:**
- Create: `lib/banks.ts`
- Create: `lib/ai-prompt.ts`

- [ ] **Step 1: Create `lib/banks.ts`**

```typescript
// lib/banks.ts
import type { Bank, WizardData } from './types'

export const BANKS: Bank[] = [
  {
    id: 'hapoalim',
    name: 'בנק הפועלים',
    shortName: 'הפועלים',
    channels: [
      { label: 'טופס אונליין', url: 'https://www.bankhapoalim.co.il/he/loans/mortgage', type: 'form' },
      { label: 'וואטסאפ', url: 'https://wa.me/972503005000', type: 'whatsapp' },
    ],
    primaryChannel: { label: 'טופס אונליין', url: 'https://www.bankhapoalim.co.il/he/loans/mortgage', type: 'form' },
  },
  {
    id: 'leumi',
    name: 'בנק לאומי',
    shortName: 'לאומי',
    channels: [
      { label: 'טופס אונליין', url: 'https://www.leumi.co.il/home/mortgage/', type: 'form' },
      { label: 'צ׳אט', url: 'https://www.leumi.co.il/home/mortgage/', type: 'chat' },
    ],
    primaryChannel: { label: 'טופס אונליין', url: 'https://www.leumi.co.il/home/mortgage/', type: 'form' },
  },
  {
    id: 'mizrahi',
    name: 'מזרחי-טפחות',
    shortName: 'מזרחי',
    channels: [
      { label: 'טופס משכנתא', url: 'https://www.mizrahi-tefahot.co.il/mortgages/', type: 'form' },
    ],
    primaryChannel: { label: 'טופס משכנתא', url: 'https://www.mizrahi-tefahot.co.il/mortgages/', type: 'form' },
  },
  {
    id: 'discount',
    name: 'בנק דיסקונט',
    shortName: 'דיסקונט',
    channels: [
      { label: 'טופס אונליין', url: 'https://www.discountbank.co.il/DB/private/loans/mortgage', type: 'form' },
    ],
    primaryChannel: { label: 'טופס אונליין', url: 'https://www.discountbank.co.il/DB/private/loans/mortgage', type: 'form' },
  },
  {
    id: 'beinleumi',
    name: 'הבנק הבינלאומי',
    shortName: 'בינלאומי',
    channels: [
      { label: 'תיאום שיחה', url: 'https://www.fibi.co.il/wps/portal/FibiMenu/Marketing/Private/Loans/Mortgage', type: 'form' },
    ],
    primaryChannel: { label: 'תיאום שיחה', url: 'https://www.fibi.co.il/wps/portal/FibiMenu/Marketing/Private/Loans/Mortgage', type: 'form' },
  },
]

export function getBankById(id: string): Bank | undefined {
  return BANKS.find(b => b.id === id)
}

/** Heuristic pre-filter before AI refines — returns all 5 banks ordered by fit */
export function rankBanksByProfile(data: WizardData): Bank[] {
  const scores: Record<string, number> = {
    hapoalim: 0, leumi: 0, mizrahi: 0, discount: 0, beinleumi: 0,
  }

  if (data.profile.employmentType === 'self_employed') scores.mizrahi += 3
  if (data.profile.isEligible) { scores.hapoalim += 2; scores.leumi += 2 }
  if (data.mix.totalLoan > 1_500_000) { scores.hapoalim += 2; scores.leumi += 2 }
  if (['צפון', 'דרום', 'נגב', 'גליל'].includes(data.property.region)) scores.mizrahi += 1
  if (data.property.transactionType === 'first_apartment') {
    scores.hapoalim += 1; scores.leumi += 1; scores.mizrahi += 1
  }

  return BANKS.sort((a, b) => (scores[b.id] ?? 0) - (scores[a.id] ?? 0))
}
```

- [ ] **Step 2: Create `lib/ai-prompt.ts`**

```typescript
// lib/ai-prompt.ts
import type { WizardData, CalculationResult } from './types'

export function buildInsightsPrompt(data: WizardData, calc: CalculationResult): string {
  const trackSummary = data.mix.tracks
    .map(t => `  - ${t.type}: ₪${t.amount.toLocaleString('he-IL')} @ ${t.rate}%`)
    .join('\n')

  return `אתה יועץ משכנתאות מומחה בשוק הישראלי. נתח את הנתונים הבאים והחזר JSON בלבד (ללא טקסט נוסף).

פרופיל הלווה:
- גיל: ${data.profile.age}
- זוג: ${data.profile.isCouple ? 'כן' : 'לא'}
- הכנסה חודשית נטו: ₪${data.profile.monthlyIncome.toLocaleString('he-IL')}
- תעסוקה: ${data.profile.employmentType}
- זכאות: ${data.profile.isEligible ? 'כן' : 'לא'}

פרטי הנכס:
- מחיר: ₪${data.property.propertyPrice.toLocaleString('he-IL')}
- הלוואה: ₪${data.mix.totalLoan.toLocaleString('he-IL')} (${calc.validation.ltvPercent.toFixed(1)}% מימון)
- סוג עסקה: ${data.property.transactionType}
- אזור: ${data.property.region}

תמהיל:
${trackSummary}

תוצאות חישוב:
- החזר חודשי כולל: ₪${calc.totalMonthlyPayment.toFixed(0)}
- עלות כוללת: ₪${calc.totalLoanCost.toFixed(0)}
- יחס החזר/הכנסה: ${calc.validation.dtiPercent.toFixed(1)}%
- אם פריים +1%: ₪${calc.sensitivity.primePlusOne.toFixed(0)} לחודש
- אם פריים +2%: ₪${calc.sensitivity.primePlusTwo.toFixed(0)} לחודש

החזר JSON בפורמט הבא בדיוק:
{
  "profileAnalysis": "משפט אחד עד שניים על פרופיל הלווה",
  "mixReasoning": "שני משפטים מדוע התמהיל הזה מתאים",
  "insights": [
    {"type": "success|warning|info|bank", "text": "טקסט"},
    {"type": "success|warning|info|bank", "text": "טקסט"},
    {"type": "success|warning|info|bank", "text": "טקסט"}
  ],
  "recommendedBanks": [
    {"bankId": "hapoalim|leumi|mizrahi|discount|beinleumi", "reason": "סיבה קצרה"},
    {"bankId": "...", "reason": "..."}
  ]
}`
}

export function buildMixPrompt(data: Omit<WizardData, 'mix'>): string {
  return `אתה יועץ משכנתאות מומחה. המלץ על תמהיל משכנתא מיטבי עבור הלווה הבא. החזר JSON בלבד.

פרופיל:
- גיל: ${data.profile.age}, הכנסה: ₪${data.profile.monthlyIncome.toLocaleString('he-IL')}
- תעסוקה: ${data.profile.employmentType}, זכאות: ${data.profile.isEligible ? 'כן' : 'לא'}
- סוג עסקה: ${data.property.transactionType}
- סכום הלוואה: ₪${(data.property.propertyPrice - data.property.downPayment).toLocaleString('he-IL')}
- תקופה: ${data.preferences.loanTermYears} שנים
- רמת סיכון: ${data.preferences.riskLevel}
- עדיפות: ${data.preferences.priority}

כללים: מסלול משתנה לא צמוד לא יעלה על 33% מהתמהיל.

החזר JSON:
{
  "tracks": [
    {"type": "prime|fixed_cpi|fixed_no_cpi|variable_cpi|variable_no_cpi|zchaut|grace", "percent": 30, "rate": 4.25, "isCpiLinked": false},
    ...
  ],
  "reasoning": "הסבר קצר בעברית"
}`
}
```

- [ ] **Step 3: Commit**

```bash
git add lib/banks.ts lib/ai-prompt.ts
git commit -m "feat: add bank data and AI prompt builder"
```

---

## Task 5: API Routes

**Files:**
- Create: `app/api/calculate/route.ts`
- Create: `app/api/ai-insights/route.ts`
- Create: `app/api/ai-mix/route.ts`

- [ ] **Step 1: Create `app/api/calculate/route.ts`**

```typescript
// app/api/calculate/route.ts
import { NextResponse } from 'next/server'
import { calculateAll } from '@/lib/calculations'
import type { WizardData } from '@/lib/types'

export async function POST(req: Request) {
  try {
    const body: WizardData = await req.json()
    const result = calculateAll(
      body.mix,
      body.preferences.loanTermYears,
      body.property,
      body.profile
    )
    return NextResponse.json(result)
  } catch (e) {
    return NextResponse.json({ error: 'חישוב נכשל' }, { status: 400 })
  }
}
```

- [ ] **Step 2: Create `app/api/ai-insights/route.ts`**

```typescript
// app/api/ai-insights/route.ts
import { NextResponse } from 'next/server'
import Anthropic from '@anthropic-ai/sdk'
import { buildInsightsPrompt } from '@/lib/ai-prompt'
import type { WizardData, CalculationResult, AIInsightsResult } from '@/lib/types'

const client = new Anthropic()

const STATIC_FALLBACK: AIInsightsResult = {
  profileAnalysis: 'הפרופיל שלך תקין לקבלת משכנתא.',
  mixReasoning: 'התמהיל שבחרת מאוזן ומתאים לרוב הלווים.',
  insights: [
    { type: 'info', text: 'מומלץ לפנות ל-3 בנקים לפחות לקבלת הצעות מתחרות.' },
    { type: 'warning', text: 'שימו לב לריביות הפריים — הן משתנות עם החלטות בנק ישראל.' },
    { type: 'success', text: 'השוו הצעות בין בנקים — ההפרש יכול להגיע לעשרות אלפי שקלים.' },
  ],
  recommendedBanks: [
    { bankId: 'mizrahi', reason: 'מתמחה במשכנתאות' },
    { bankId: 'hapoalim', reason: 'שירות דיגיטלי מוביל' },
  ],
}

export async function POST(req: Request) {
  try {
    const { wizardData, calcResult }: { wizardData: WizardData; calcResult: CalculationResult } = await req.json()
    const prompt = buildInsightsPrompt(wizardData, calcResult)

    const message = await client.messages.create({
      model: 'claude-opus-4-5',
      max_tokens: 1024,
      messages: [{ role: 'user', content: prompt }],
    })

    const text = message.content[0].type === 'text' ? message.content[0].text : ''
    const jsonMatch = text.match(/\{[\s\S]*\}/)
    if (!jsonMatch) throw new Error('No JSON in response')
    const result: AIInsightsResult = JSON.parse(jsonMatch[0])
    return NextResponse.json(result)
  } catch (e) {
    // Graceful degradation — return static fallback
    return NextResponse.json(STATIC_FALLBACK)
  }
}
```

- [ ] **Step 3: Create `app/api/ai-mix/route.ts`**

```typescript
// app/api/ai-mix/route.ts
import { NextResponse } from 'next/server'
import Anthropic from '@anthropic-ai/sdk'
import { buildMixPrompt } from '@/lib/ai-prompt'
import type { WizardData, MortgageMix, MixTrack } from '@/lib/types'

const client = new Anthropic()

export async function POST(req: Request) {
  try {
    const data: Omit<WizardData, 'mix'> = await req.json()
    const totalLoan = data.property.propertyPrice - data.property.downPayment
    const prompt = buildMixPrompt(data)

    const message = await client.messages.create({
      model: 'claude-opus-4-5',
      max_tokens: 512,
      messages: [{ role: 'user', content: prompt }],
    })

    const text = message.content[0].type === 'text' ? message.content[0].text : ''
    const jsonMatch = text.match(/\{[\s\S]*\}/)
    if (!jsonMatch) throw new Error('No JSON')
    const parsed = JSON.parse(jsonMatch[0])

    const tracks: MixTrack[] = parsed.tracks.map((t: { type: string; percent: number; rate: number; isCpiLinked: boolean }) => ({
      type: t.type,
      amount: Math.round((t.percent / 100) * totalLoan),
      rate: t.rate,
      isCpiLinked: t.isCpiLinked,
    }))

    const mix: MortgageMix = { tracks, totalLoan }
    return NextResponse.json({ mix, reasoning: parsed.reasoning })
  } catch (e) {
    // Fallback: balanced default mix
    const totalLoan = 0 // will be recalculated from body — handled client side
    return NextResponse.json({
      mix: { tracks: [], totalLoan },
      reasoning: 'לא ניתן לקבל המלצת AI כרגע. נא לבחור תמהיל ידנית.',
      error: true,
    })
  }
}
```

- [ ] **Step 4: Commit**

```bash
git add app/api/
git commit -m "feat: add calculate, ai-insights, and ai-mix API routes"
```

---

## Task 6: Wizard State + Shell

**Files:**
- Create: `app/simulator/page.tsx`
- Create: `components/wizard/WizardProgress.tsx`

- [ ] **Step 1: Create `components/wizard/WizardProgress.tsx`**

```tsx
// components/wizard/WizardProgress.tsx
'use client'
import { cn } from '@/lib/utils'

const STEPS = ['פרופיל לווה', 'פרטי נכס', 'העדפות', 'תמהיל']

interface Props {
  currentStep: number  // 0-indexed
}

export function WizardProgress({ currentStep }: Props) {
  return (
    <div className="mb-8">
      <div className="flex justify-between mb-2">
        {STEPS.map((label, i) => (
          <div key={i} className="flex flex-col items-center gap-1">
            <div className={cn(
              'w-8 h-8 rounded-full flex items-center justify-center text-sm font-bold',
              i < currentStep ? 'bg-green-500 text-white' :
              i === currentStep ? 'bg-blue-600 text-white' :
              'bg-gray-200 text-gray-500'
            )}>
              {i < currentStep ? '✓' : i + 1}
            </div>
            <span className={cn(
              'text-xs hidden sm:block',
              i === currentStep ? 'text-blue-600 font-semibold' : 'text-gray-400'
            )}>
              {label}
            </span>
          </div>
        ))}
      </div>
      <div className="w-full bg-gray-200 rounded-full h-2">
        <div
          className="bg-blue-600 h-2 rounded-full transition-all duration-300"
          style={{ width: `${((currentStep) / (STEPS.length - 1)) * 100}%` }}
        />
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Create `app/simulator/page.tsx`**

```tsx
// app/simulator/page.tsx
'use client'
import { useState } from 'react'
import { useRouter } from 'next/navigation'
import { WizardProgress } from '@/components/wizard/WizardProgress'
import { StepProfile } from '@/components/wizard/StepProfile'
import { StepProperty } from '@/components/wizard/StepProperty'
import { StepPreferences } from '@/components/wizard/StepPreferences'
import { StepMix } from '@/components/wizard/StepMix'
import type { BorrowerProfile, PropertyDetails, Preferences, MortgageMix, WizardData } from '@/lib/types'

const DEFAULT_PROFILE: BorrowerProfile = {
  age: 35, isCouple: true, monthlyIncome: 20000,
  employmentType: 'salaried', isEligible: false,
}
const DEFAULT_PROPERTY: PropertyDetails = {
  propertyPrice: 1800000, downPayment: 450000,
  transactionType: 'first_apartment', region: 'מרכז',
}
const DEFAULT_PREFS: Preferences = {
  loanTermYears: 25, riskLevel: 'balanced', priority: 'low_monthly',
}

export default function SimulatorPage() {
  const router = useRouter()
  const [step, setStep] = useState(0)
  const [profile, setProfile] = useState<BorrowerProfile>(DEFAULT_PROFILE)
  const [property, setProperty] = useState<PropertyDetails>(DEFAULT_PROPERTY)
  const [preferences, setPreferences] = useState<Preferences>(DEFAULT_PREFS)
  const [mix, setMix] = useState<MortgageMix | null>(null)

  const handleFinish = (finalMix: MortgageMix) => {
    const data: WizardData = { profile, property, preferences, mix: finalMix }
    const encoded = encodeURIComponent(JSON.stringify(data))
    router.push(`/results?data=${encoded}`)
  }

  return (
    <main className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-4">
      <div className="max-w-2xl mx-auto">
        <h1 className="text-3xl font-bold text-center text-blue-900 mb-2">יועץ משכנתא AI</h1>
        <p className="text-center text-gray-500 mb-8">סימולטור חכם לתמהיל מותאם אישית</p>
        <WizardProgress currentStep={step} />
        <div className="bg-white rounded-2xl shadow-lg p-6">
          {step === 0 && (
            <StepProfile value={profile} onChange={setProfile} onNext={() => setStep(1)} />
          )}
          {step === 1 && (
            <StepProperty value={property} onChange={setProperty}
              onNext={() => setStep(2)} onBack={() => setStep(0)} />
          )}
          {step === 2 && (
            <StepPreferences value={preferences} onChange={setPreferences}
              onNext={() => setStep(3)} onBack={() => setStep(1)} />
          )}
          {step === 3 && (
            <StepMix
              profile={profile} property={property} preferences={preferences}
              onFinish={handleFinish} onBack={() => setStep(2)} />
          )}
        </div>
      </div>
    </main>
  )
}
```

- [ ] **Step 3: Commit**

```bash
git add app/simulator/ components/wizard/WizardProgress.tsx
git commit -m "feat: add wizard shell with step state and progress bar"
```

---

## Task 7: Wizard Steps 1–3

**Files:**
- Create: `components/wizard/StepProfile.tsx`
- Create: `components/wizard/StepProperty.tsx`
- Create: `components/wizard/StepPreferences.tsx`

- [ ] **Step 1: Create `components/wizard/StepProfile.tsx`**

```tsx
// components/wizard/StepProfile.tsx
'use client'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'
import type { BorrowerProfile, EmploymentType } from '@/lib/types'

interface Props {
  value: BorrowerProfile
  onChange: (v: BorrowerProfile) => void
  onNext: () => void
}

export function StepProfile({ value, onChange, onNext }: Props) {
  const update = (partial: Partial<BorrowerProfile>) => onChange({ ...value, ...partial })

  return (
    <div className="space-y-5">
      <h2 className="text-xl font-bold text-gray-800">פרופיל הלווה</h2>

      <div className="grid grid-cols-2 gap-4">
        <div>
          <Label>גיל</Label>
          <Input type="number" min={18} max={80} value={value.age}
            onChange={e => update({ age: +e.target.value })} />
        </div>
        <div>
          <Label>הכנסה חודשית נטו (₪)</Label>
          <Input type="number" min={0} value={value.monthlyIncome}
            onChange={e => update({ monthlyIncome: +e.target.value })} />
        </div>
      </div>

      <div>
        <Label>מצב משפחתי</Label>
        <div className="flex gap-3 mt-2">
          {[{ v: false, l: 'יחיד/ה' }, { v: true, l: 'זוג' }].map(({ v, l }) => (
            <button key={l} onClick={() => update({ isCouple: v })}
              className={`flex-1 py-2 rounded-lg border-2 text-sm font-medium transition-colors ${
                value.isCouple === v ? 'border-blue-600 bg-blue-50 text-blue-700' : 'border-gray-200 text-gray-600'
              }`}>
              {l}
            </button>
          ))}
        </div>
      </div>

      <div>
        <Label>סוג תעסוקה</Label>
        <div className="flex gap-2 mt-2 flex-wrap">
          {([['salaried', 'שכיר/ה'], ['self_employed', 'עצמאי/ת'], ['mixed', 'מעורב']] as [EmploymentType, string][]).map(([v, l]) => (
            <button key={v} onClick={() => update({ employmentType: v })}
              className={`px-4 py-2 rounded-lg border-2 text-sm font-medium transition-colors ${
                value.employmentType === v ? 'border-blue-600 bg-blue-50 text-blue-700' : 'border-gray-200 text-gray-600'
              }`}>
              {l}
            </button>
          ))}
        </div>
      </div>

      <div className="flex items-center gap-3 p-3 bg-amber-50 rounded-lg border border-amber-200">
        <input type="checkbox" id="eligible" checked={value.isEligible}
          onChange={e => update({ isEligible: e.target.checked })} className="w-4 h-4" />
        <label htmlFor="eligible" className="text-sm text-amber-800 cursor-pointer">
          זכאי/ת למשכנתא מסובסדת (דירה ראשונה, חייל משוחרר וכו׳)
        </label>
      </div>

      <Button onClick={onNext} className="w-full bg-blue-600 hover:bg-blue-700">
        המשך →
      </Button>
    </div>
  )
}
```

- [ ] **Step 2: Create `components/wizard/StepProperty.tsx`**

```tsx
// components/wizard/StepProperty.tsx
'use client'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'
import type { PropertyDetails, TransactionType } from '@/lib/types'

interface Props {
  value: PropertyDetails
  onChange: (v: PropertyDetails) => void
  onNext: () => void
  onBack: () => void
}

const TRANSACTION_LABELS: [TransactionType, string][] = [
  ['first_apartment', 'דירה ראשונה'],
  ['investor', 'להשקעה'],
  ['mehir_lamishtaken', 'מחיר למשתכן'],
  ['upgrade', 'שיפור דיור'],
]

const REGIONS = ['מרכז', 'תל אביב', 'ירושלים', 'חיפה והצפון', 'דרום', 'שרון']

const LTV_LIMITS: Record<TransactionType, number> = {
  first_apartment: 75, investor: 50, mehir_lamishtaken: 90, upgrade: 70,
}

export function StepProperty({ value, onChange, onNext, onBack }: Props) {
  const update = (partial: Partial<PropertyDetails>) => onChange({ ...value, ...partial })
  const loanAmount = value.propertyPrice - value.downPayment
  const ltvPercent = value.propertyPrice > 0 ? (loanAmount / value.propertyPrice) * 100 : 0
  const ltvMax = LTV_LIMITS[value.transactionType]
  const ltvViolation = ltvPercent > ltvMax

  return (
    <div className="space-y-5">
      <h2 className="text-xl font-bold text-gray-800">פרטי הנכס</h2>

      <div className="grid grid-cols-2 gap-4">
        <div>
          <Label>מחיר הנכס (₪)</Label>
          <Input type="number" min={0} value={value.propertyPrice}
            onChange={e => update({ propertyPrice: +e.target.value })} />
        </div>
        <div>
          <Label>הון עצמי (₪)</Label>
          <Input type="number" min={0} value={value.downPayment}
            onChange={e => update({ downPayment: +e.target.value })} />
        </div>
      </div>

      {value.propertyPrice > 0 && (
        <div className={`p-3 rounded-lg text-sm ${ltvViolation ? 'bg-red-50 border border-red-300 text-red-700' : 'bg-green-50 border border-green-200 text-green-700'}`}>
          {ltvViolation
            ? `⚠️ מימון ${ltvPercent.toFixed(1)}% חורג מהמותר (${ltvMax}%) עבור ${TRANSACTION_LABELS.find(([v]) => v === value.transactionType)?.[1]}`
            : `✅ מימון ${ltvPercent.toFixed(1)}% — הלוואה: ₪${loanAmount.toLocaleString('he-IL')}`
          }
        </div>
      )}

      <div>
        <Label>סוג עסקה</Label>
        <div className="grid grid-cols-2 gap-2 mt-2">
          {TRANSACTION_LABELS.map(([v, l]) => (
            <button key={v} onClick={() => update({ transactionType: v })}
              className={`py-2 rounded-lg border-2 text-sm font-medium transition-colors ${
                value.transactionType === v ? 'border-blue-600 bg-blue-50 text-blue-700' : 'border-gray-200 text-gray-600'
              }`}>
              {l}
            </button>
          ))}
        </div>
      </div>

      <div>
        <Label>אזור גיאוגרפי</Label>
        <div className="flex gap-2 mt-2 flex-wrap">
          {REGIONS.map(r => (
            <button key={r} onClick={() => update({ region: r })}
              className={`px-3 py-1.5 rounded-full border text-sm transition-colors ${
                value.region === r ? 'border-blue-600 bg-blue-50 text-blue-700' : 'border-gray-200 text-gray-500'
              }`}>
              {r}
            </button>
          ))}
        </div>
      </div>

      <div className="flex gap-3">
        <Button variant="outline" onClick={onBack} className="flex-1">← חזרה</Button>
        <Button onClick={onNext} disabled={ltvViolation} className="flex-1 bg-blue-600 hover:bg-blue-700">
          המשך →
        </Button>
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Create `components/wizard/StepPreferences.tsx`**

```tsx
// components/wizard/StepPreferences.tsx
'use client'
import { Button } from '@/components/ui/button'
import { Label } from '@/components/ui/label'
import type { Preferences, RiskLevel, Priority } from '@/lib/types'

interface Props {
  value: Preferences
  onChange: (v: Preferences) => void
  onNext: () => void
  onBack: () => void
}

export function StepPreferences({ value, onChange, onNext, onBack }: Props) {
  const update = (partial: Partial<Preferences>) => onChange({ ...value, ...partial })

  return (
    <div className="space-y-6">
      <h2 className="text-xl font-bold text-gray-800">העדפות</h2>

      <div>
        <Label>תקופת ההלוואה: <span className="text-blue-600 font-bold">{value.loanTermYears} שנים</span></Label>
        <input type="range" min={10} max={30} step={1} value={value.loanTermYears}
          onChange={e => update({ loanTermYears: +e.target.value })}
          className="w-full mt-2 accent-blue-600" />
        <div className="flex justify-between text-xs text-gray-400 mt-1">
          <span>10 שנים</span><span>30 שנים</span>
        </div>
      </div>

      <div>
        <Label>רמת סיכון</Label>
        <div className="grid grid-cols-3 gap-2 mt-2">
          {([['conservative', '🛡️ שמרן'], ['balanced', '⚖️ מאוזן'], ['aggressive', '🚀 אגרסיבי']] as [RiskLevel, string][]).map(([v, l]) => (
            <button key={v} onClick={() => update({ riskLevel: v })}
              className={`py-3 rounded-lg border-2 text-sm font-medium transition-colors ${
                value.riskLevel === v ? 'border-blue-600 bg-blue-50 text-blue-700' : 'border-gray-200 text-gray-600'
              }`}>
              {l}
            </button>
          ))}
        </div>
      </div>

      <div>
        <Label>מה הכי חשוב לך?</Label>
        <div className="space-y-2 mt-2">
          {([
            ['low_monthly', '💰 החזר חודשי נמוך'],
            ['low_total', '📉 עלות כוללת נמוכה'],
            ['max_flexibility', '🔓 גמישות מקסימלית'],
          ] as [Priority, string][]).map(([v, l]) => (
            <button key={v} onClick={() => update({ priority: v })}
              className={`w-full py-3 px-4 rounded-lg border-2 text-sm font-medium text-right transition-colors ${
                value.priority === v ? 'border-blue-600 bg-blue-50 text-blue-700' : 'border-gray-200 text-gray-600'
              }`}>
              {l}
            </button>
          ))}
        </div>
      </div>

      <div className="flex gap-3">
        <Button variant="outline" onClick={onBack} className="flex-1">← חזרה</Button>
        <Button onClick={onNext} className="flex-1 bg-blue-600 hover:bg-blue-700">המשך →</Button>
      </div>
    </div>
  )
}
```

- [ ] **Step 4: Commit**

```bash
git add components/wizard/StepProfile.tsx components/wizard/StepProperty.tsx components/wizard/StepPreferences.tsx
git commit -m "feat: add wizard steps 1-3 (profile, property, preferences)"
```

---

## Task 8: Wizard Step 4 — Mix

**Files:**
- Create: `components/wizard/StepMix.tsx`

- [ ] **Step 1: Create `components/wizard/StepMix.tsx`**

```tsx
// components/wizard/StepMix.tsx
'use client'
import { useState } from 'react'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import type { BorrowerProfile, PropertyDetails, Preferences, MortgageMix, MixTrack, TrackType, WizardData } from '@/lib/types'

const TRACK_LABELS: Record<TrackType, string> = {
  prime: 'פריים',
  fixed_cpi: 'קבועה צמודה (ק"צ)',
  fixed_no_cpi: 'קבועה לא צמודה (קל"צ)',
  variable_cpi: 'משתנה צמודה',
  variable_no_cpi: 'משתנה לא צמודה',
  zchaut: 'זכאות',
  grace: 'גרייס',
}

const DEFAULT_RATES: Record<TrackType, number> = {
  prime: 4.25, fixed_cpi: 3.2, fixed_no_cpi: 5.5,
  variable_cpi: 2.8, variable_no_cpi: 5.1, zchaut: 3.0, grace: 4.25,
}

interface Props {
  profile: BorrowerProfile
  property: PropertyDetails
  preferences: Preferences
  onFinish: (mix: MortgageMix) => void
  onBack: () => void
}

const totalLoanFrom = (property: PropertyDetails) => property.propertyPrice - property.downPayment

export function StepMix({ profile, property, preferences, onFinish, onBack }: Props) {
  const totalLoan = totalLoanFrom(property)
  const [tracks, setTracks] = useState<MixTrack[]>([
    { type: 'prime', amount: Math.round(totalLoan * 0.3), rate: DEFAULT_RATES.prime, isCpiLinked: false },
    { type: 'fixed_no_cpi', amount: Math.round(totalLoan * 0.4), rate: DEFAULT_RATES.fixed_no_cpi, isCpiLinked: false },
    { type: 'fixed_cpi', amount: Math.round(totalLoan * 0.3), rate: DEFAULT_RATES.fixed_cpi, isCpiLinked: true },
  ])
  const [aiReasoning, setAiReasoning] = useState<string | null>(null)
  const [loadingAI, setLoadingAI] = useState(false)

  const allocatedTotal = tracks.reduce((s, t) => s + t.amount, 0)
  const remaining = totalLoan - allocatedTotal
  const allAllocated = Math.abs(remaining) < 100

  const updateTrack = (i: number, partial: Partial<MixTrack>) => {
    setTracks(prev => prev.map((t, idx) => idx === i ? { ...t, ...partial } : t))
  }

  const removeTrack = (i: number) => setTracks(prev => prev.filter((_, idx) => idx !== i))

  const addTrack = (type: TrackType) => {
    if (tracks.find(t => t.type === type)) return
    setTracks(prev => [...prev, {
      type, amount: 0, rate: DEFAULT_RATES[type],
      isCpiLinked: ['fixed_cpi', 'variable_cpi'].includes(type),
    }])
  }

  const handleAIMix = async () => {
    setLoadingAI(true)
    try {
      const body: Omit<WizardData, 'mix'> = { profile, property, preferences }
      const res = await fetch('/api/ai-mix', { method: 'POST', body: JSON.stringify(body), headers: { 'Content-Type': 'application/json' } })
      const data = await res.json()
      if (!data.error && data.mix?.tracks?.length) {
        setTracks(data.mix.tracks)
        setAiReasoning(data.reasoning)
      }
    } finally {
      setLoadingAI(false)
    }
  }

  const availableToAdd = (Object.keys(TRACK_LABELS) as TrackType[]).filter(t => !tracks.find(tr => tr.type === t))

  return (
    <div className="space-y-5">
      <h2 className="text-xl font-bold text-gray-800">תמהיל משכנתא</h2>
      <p className="text-sm text-gray-500">סכום הלוואה: <span className="font-bold text-blue-700">₪{totalLoan.toLocaleString('he-IL')}</span></p>

      <Button onClick={handleAIMix} disabled={loadingAI} variant="outline"
        className="w-full border-purple-400 text-purple-700 hover:bg-purple-50">
        {loadingAI ? '⏳ מחשב...' : '🤖 בחר לי תמהיל (AI)'}
      </Button>

      {aiReasoning && (
        <div className="p-3 bg-purple-50 border border-purple-200 rounded-lg text-sm text-purple-800">
          💡 {aiReasoning}
        </div>
      )}

      <div className="space-y-3">
        {tracks.map((track, i) => (
          <div key={track.type} className="p-3 bg-gray-50 rounded-lg border">
            <div className="flex justify-between items-center mb-2">
              <span className="font-medium text-sm">{TRACK_LABELS[track.type]}</span>
              <button onClick={() => removeTrack(i)} className="text-red-400 text-xs hover:text-red-600">✕ הסר</button>
            </div>
            <div className="grid grid-cols-2 gap-2">
              <div>
                <label className="text-xs text-gray-500">סכום (₪)</label>
                <Input type="number" value={track.amount}
                  onChange={e => updateTrack(i, { amount: +e.target.value })} className="text-sm" />
              </div>
              <div>
                <label className="text-xs text-gray-500">ריבית (%)</label>
                <Input type="number" step="0.01" value={track.rate}
                  onChange={e => updateTrack(i, { rate: +e.target.value })} className="text-sm" />
              </div>
            </div>
            <div className="text-xs text-gray-400 mt-1">
              {((track.amount / totalLoan) * 100).toFixed(1)}% מהתמהיל
            </div>
          </div>
        ))}
      </div>

      <div className={`text-sm font-medium ${allAllocated ? 'text-green-600' : 'text-orange-500'}`}>
        {allAllocated ? '✅ כל הסכום חולק' : `נותר לחלק: ₪${remaining.toLocaleString('he-IL')}`}
      </div>

      {availableToAdd.length > 0 && (
        <div>
          <p className="text-xs text-gray-400 mb-2">הוסף מסלול:</p>
          <div className="flex flex-wrap gap-2">
            {availableToAdd.map(t => (
              <button key={t} onClick={() => addTrack(t)}
                className="px-3 py-1 rounded-full border border-dashed border-blue-300 text-blue-600 text-xs hover:bg-blue-50">
                + {TRACK_LABELS[t]}
              </button>
            ))}
          </div>
        </div>
      )}

      <div className="flex gap-3">
        <Button variant="outline" onClick={onBack} className="flex-1">← חזרה</Button>
        <Button onClick={() => onFinish({ tracks, totalLoan })} disabled={!allAllocated}
          className="flex-1 bg-blue-600 hover:bg-blue-700">
          לתוצאות →
        </Button>
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Commit**

```bash
git add components/wizard/StepMix.tsx
git commit -m "feat: add wizard step 4 - mortgage mix with AI recommendation"
```

---

## Task 9: Results Page

**Files:**
- Create: `app/results/page.tsx`
- Create: `components/results/SummaryCard.tsx`
- Create: `components/results/TrackTable.tsx`
- Create: `components/results/InsightsPanel.tsx`
- Create: `components/results/BankCards.tsx`

- [ ] **Step 1: Create `components/results/SummaryCard.tsx`**

```tsx
// components/results/SummaryCard.tsx
import type { CalculationResult } from '@/lib/types'

interface Props { calc: CalculationResult; loanTermYears: number }

export function SummaryCard({ calc, loanTermYears }: Props) {
  const { totalMonthlyPayment, totalLoanCost, validation, sensitivity } = calc
  return (
    <div className="grid grid-cols-2 md:grid-cols-4 gap-4 mb-6">
      {[
        { label: 'החזר חודשי', value: `₪${totalMonthlyPayment.toFixed(0).replace(/\B(?=(\d{3})+(?!\d))/g, ',')}`, color: 'blue' },
        { label: 'עלות כוללת', value: `₪${(totalLoanCost / 1_000_000).toFixed(2)}M`, color: 'indigo' },
        { label: 'יחס החזר/הכנסה', value: `${validation.dtiPercent.toFixed(1)}%`, color: validation.dtiWarning ? 'orange' : 'green' },
        { label: 'מימון (LTV)', value: `${validation.ltvPercent.toFixed(1)}%`, color: validation.ltvValid ? 'green' : 'red' },
      ].map(({ label, value, color }) => (
        <div key={label} className={`bg-white rounded-xl p-4 shadow-sm border-t-4 border-${color}-500`}>
          <p className="text-xs text-gray-500 mb-1">{label}</p>
          <p className={`text-xl font-bold text-${color}-700`}>{value}</p>
        </div>
      ))}
    </div>
  )
}
```

- [ ] **Step 2: Create `components/results/TrackTable.tsx`**

```tsx
// components/results/TrackTable.tsx
import type { TrackResult } from '@/lib/types'

const LABELS: Record<string, string> = {
  prime: 'פריים', fixed_cpi: 'ק"צ', fixed_no_cpi: 'קל"צ',
  variable_cpi: 'משתנה צמודה', variable_no_cpi: 'משתנה לא צמודה',
  zchaut: 'זכאות', grace: 'גרייס',
}

export function TrackTable({ tracks }: { tracks: TrackResult[] }) {
  const fmt = (n: number) => `₪${Math.round(n).toLocaleString('he-IL')}`
  return (
    <div className="bg-white rounded-xl shadow-sm overflow-hidden mb-6">
      <div className="p-4 border-b bg-gray-50">
        <h3 className="font-bold text-gray-700">פירוט לפי מסלול</h3>
      </div>
      <div className="overflow-x-auto">
        <table className="w-full text-sm">
          <thead className="bg-gray-100 text-gray-600 text-xs">
            <tr>
              <th className="p-3 text-right">מסלול</th>
              <th className="p-3 text-right">סכום</th>
              <th className="p-3 text-right">ריבית</th>
              <th className="p-3 text-right">החזר חודשי</th>
              <th className="p-3 text-right">עלות כוללת</th>
            </tr>
          </thead>
          <tbody>
            {tracks.map((t, i) => (
              <tr key={i} className="border-t hover:bg-gray-50">
                <td className="p-3 font-medium">{LABELS[t.type] ?? t.type}</td>
                <td className="p-3">{fmt(t.amount)}</td>
                <td className="p-3">{t.rate}%{t.isCpiLinked ? ' + מדד' : ''}</td>
                <td className="p-3 font-bold text-blue-700">{fmt(t.monthlyPayment)}</td>
                <td className="p-3 text-gray-600">{fmt(t.totalPayment)}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Create `components/results/InsightsPanel.tsx`**

```tsx
// components/results/InsightsPanel.tsx
import type { AIInsightsResult, InsightType } from '@/lib/types'

const ICONS: Record<InsightType, string> = {
  success: '✅', warning: '⚠️', info: '📈', bank: '🏦',
}

interface Props { insights: AIInsightsResult; isLoading?: boolean }

export function InsightsPanel({ insights, isLoading }: Props) {
  if (isLoading) {
    return (
      <div className="bg-gradient-to-r from-purple-50 to-blue-50 rounded-xl p-6 mb-6 border border-purple-100">
        <div className="animate-pulse space-y-3">
          <div className="h-4 bg-purple-200 rounded w-1/2" />
          <div className="h-3 bg-purple-100 rounded w-full" />
          <div className="h-3 bg-purple-100 rounded w-3/4" />
        </div>
      </div>
    )
  }

  return (
    <div className="bg-gradient-to-r from-purple-50 to-blue-50 rounded-xl p-6 mb-6 border border-purple-100">
      <h3 className="font-bold text-purple-900 mb-3">💡 מה כדאי שתדע על התמהיל שלך</h3>
      <p className="text-sm text-gray-700 mb-3">{insights.profileAnalysis}</p>
      <p className="text-sm text-blue-800 bg-blue-50 p-2 rounded mb-4">{insights.mixReasoning}</p>
      <div className="space-y-2">
        {insights.insights.map((ins, i) => (
          <div key={i} className="flex gap-2 text-sm">
            <span>{ICONS[ins.type]}</span>
            <span className="text-gray-700">{ins.text}</span>
          </div>
        ))}
      </div>
    </div>
  )
}
```

- [ ] **Step 4: Create `components/results/BankCards.tsx`**

```tsx
// components/results/BankCards.tsx
import Link from 'next/link'
import { Button } from '@/components/ui/button'
import { BANKS } from '@/lib/banks'
import type { RecommendedBank } from '@/lib/types'

interface Props {
  recommendedBanks: RecommendedBank[]
  encodedData: string
}

const CHANNEL_ICONS: Record<string, string> = {
  form: '📋', whatsapp: '💬', chat: '🗨️', phone: '📞',
}

export function BankCards({ recommendedBanks, encodedData }: Props) {
  return (
    <div className="mb-6">
      <h3 className="font-bold text-gray-700 mb-3">🏦 הבנקים המומלצים עבורך</h3>
      <div className="grid gap-4 md:grid-cols-2">
        {recommendedBanks.map(({ bankId, reason }) => {
          const bank = BANKS.find(b => b.id === bankId)
          if (!bank) return null
          return (
            <div key={bankId} className="bg-white rounded-xl p-4 shadow-sm border hover:shadow-md transition-shadow">
              <h4 className="font-bold text-gray-800 mb-1">{bank.name}</h4>
              <p className="text-sm text-gray-500 mb-3">{reason}</p>
              <div className="flex gap-2 flex-wrap">
                <Link href={`/letter/${bankId}?data=${encodedData}`}>
                  <Button size="sm" className="bg-blue-600 hover:bg-blue-700 text-xs">
                    ✉️ צור פנייה
                  </Button>
                </Link>
                <a href={bank.primaryChannel.url} target="_blank" rel="noopener noreferrer">
                  <Button size="sm" variant="outline" className="text-xs">
                    {CHANNEL_ICONS[bank.primaryChannel.type]} {bank.primaryChannel.label}
                  </Button>
                </a>
              </div>
            </div>
          )
        })}
      </div>
    </div>
  )
}
```

- [ ] **Step 5: Create `app/results/page.tsx`**

```tsx
// app/results/page.tsx
'use client'
import { useSearchParams, useRouter } from 'next/navigation'
import { useEffect, useState, Suspense } from 'react'
import { SummaryCard } from '@/components/results/SummaryCard'
import { TrackTable } from '@/components/results/TrackTable'
import { InsightsPanel } from '@/components/results/InsightsPanel'
import { BankCards } from '@/components/results/BankCards'
import { Button } from '@/components/ui/button'
import type { WizardData, CalculationResult, AIInsightsResult } from '@/lib/types'

function ResultsContent() {
  const params = useSearchParams()
  const router = useRouter()
  const encodedData = params.get('data') ?? ''
  const wizardData: WizardData | null = encodedData ? JSON.parse(decodeURIComponent(encodedData)) : null

  const [calc, setCalc] = useState<CalculationResult | null>(null)
  const [insights, setInsights] = useState<AIInsightsResult | null>(null)
  const [loadingCalc, setLoadingCalc] = useState(true)
  const [loadingAI, setLoadingAI] = useState(true)

  useEffect(() => {
    if (!wizardData) return
    fetch('/api/calculate', {
      method: 'POST',
      body: JSON.stringify(wizardData),
      headers: { 'Content-Type': 'application/json' },
    })
      .then(r => r.json())
      .then((result: CalculationResult) => {
        setCalc(result)
        setLoadingCalc(false)
        return fetch('/api/ai-insights', {
          method: 'POST',
          body: JSON.stringify({ wizardData, calcResult: result }),
          headers: { 'Content-Type': 'application/json' },
        })
      })
      .then(r => r?.json())
      .then((ai: AIInsightsResult) => {
        setInsights(ai)
        setLoadingAI(false)
      })
      .catch(() => setLoadingAI(false))
  }, [])

  if (!wizardData) return <div className="text-center p-8">נתונים חסרים. <button onClick={() => router.push('/simulator')} className="text-blue-600">חזרה לסימולטור</button></div>
  if (loadingCalc) return <div className="text-center p-8 text-gray-500">מחשב...</div>
  if (!calc) return <div className="text-center p-8 text-red-500">שגיאה בחישוב</div>

  return (
    <main className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-4">
      <div className="max-w-3xl mx-auto">
        <div className="flex justify-between items-center mb-6">
          <h1 className="text-2xl font-bold text-blue-900">תוצאות הסימולציה</h1>
          <Button variant="outline" size="sm" onClick={() => router.push('/simulator')}>עריכה</Button>
        </div>
        <SummaryCard calc={calc} loanTermYears={wizardData.preferences.loanTermYears} />
        <TrackTable tracks={calc.tracks} />
        <InsightsPanel insights={insights!} isLoading={loadingAI || !insights} />
        {insights && <BankCards recommendedBanks={insights.recommendedBanks} encodedData={encodedData} />}
      </div>
    </main>
  )
}

export default function ResultsPage() {
  return <Suspense fallback={<div className="text-center p-8">טוען...</div>}><ResultsContent /></Suspense>
}
```

- [ ] **Step 6: Commit**

```bash
git add app/results/ components/results/
git commit -m "feat: add results page with summary, track table, AI insights, bank cards"
```

---

## Task 10: Letter Page + Landing Page

**Files:**
- Create: `app/letter/[bankId]/page.tsx`
- Create: `app/page.tsx`

- [ ] **Step 1: Create `app/letter/[bankId]/page.tsx`**

```tsx
// app/letter/[bankId]/page.tsx
'use client'
import { useSearchParams, useParams } from 'next/navigation'
import { useState, Suspense } from 'react'
import { Button } from '@/components/ui/button'
import { getBankById } from '@/lib/banks'
import type { WizardData } from '@/lib/types'

const EMPLOYMENT_HE: Record<string, string> = {
  salaried: 'שכיר/ה', self_employed: 'עצמאי/ת', mixed: 'שכיר/ה ועצמאי/ת',
}
const TRANSACTION_HE: Record<string, string> = {
  first_apartment: 'ראשונה', investor: 'להשקעה',
  mehir_lamishtaken: 'במסגרת מחיר למשתכן', upgrade: 'שיפור דיור',
}

function LetterContent() {
  const { bankId } = useParams<{ bankId: string }>()
  const params = useSearchParams()
  const encodedData = params.get('data') ?? ''
  const data: WizardData | null = encodedData ? JSON.parse(decodeURIComponent(encodedData)) : null
  const bank = getBankById(bankId)
  const [copied, setCopied] = useState(false)

  if (!data || !bank) return <div className="text-center p-8 text-gray-500">נתונים חסרים</div>

  const loanAmount = data.property.propertyPrice - data.property.downPayment
  const ltvPercent = ((loanAmount / data.property.propertyPrice) * 100).toFixed(1)
  const trackLines = data.mix.tracks
    .map(t => {
      const label: Record<string, string> = {
        prime: 'פריים', fixed_cpi: 'קבועה צמודה', fixed_no_cpi: 'קבועה לא צמודה',
        variable_cpi: 'משתנה צמודה', variable_no_cpi: 'משתנה לא צמודה', zchaut: 'זכאות', grace: 'גרייס',
      }
      const pct = ((t.amount / loanAmount) * 100).toFixed(0)
      return `  - ${label[t.type] ?? t.type}: ₪${t.amount.toLocaleString('he-IL')} (${pct}%) @ ${t.rate}%`
    })
    .join('\n')

  const letterText = `שלום,

אני ${EMPLOYMENT_HE[data.profile.employmentType]}, בן/בת ${data.profile.age}, ${data.profile.isCouple ? 'זוג' : 'יחיד/ה'}.
הכנסה חודשית נטו: ₪${data.profile.monthlyIncome.toLocaleString('he-IL')}
${data.profile.isEligible ? 'זכאי/ת למשכנתא מסובסדת.\n' : ''}
מעוניין/ת לקבל הצעה למשכנתא לרכישת דירה ${TRANSACTION_HE[data.property.transactionType] ?? ''}.

פרטי הבקשה:
• מחיר נכס: ₪${data.property.propertyPrice.toLocaleString('he-IL')}
• הון עצמי: ₪${data.property.downPayment.toLocaleString('he-IL')}
• הלוואה מבוקשת: ₪${loanAmount.toLocaleString('he-IL')} (${ltvPercent}% מימון)
• תקופה: ${data.preferences.loanTermYears} שנים
• אזור: ${data.property.region}

תמהיל מבוקש:
${trackLines}

אשמח לקבל הצעת מחיר מפורטת.
תודה רבה`

  const handleCopy = () => {
    navigator.clipboard.writeText(letterText)
    setCopied(true)
    setTimeout(() => setCopied(false), 2000)
  }

  return (
    <main className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-4">
      <div className="max-w-2xl mx-auto">
        <h1 className="text-2xl font-bold text-blue-900 mb-2">פנייה ל{bank.name}</h1>
        <p className="text-gray-500 text-sm mb-6">העתק את הטקסט ושלח דרך הערוץ המועדף</p>

        <div className="bg-white rounded-xl shadow p-6 mb-4 font-mono text-sm leading-7 whitespace-pre-wrap border-r-4 border-blue-500">
          {letterText}
        </div>

        <div className="flex gap-3 mb-4">
          <Button onClick={handleCopy} className="flex-1 bg-blue-600 hover:bg-blue-700">
            {copied ? '✅ הועתק!' : '📋 העתק טקסט'}
          </Button>
          <a href={bank.primaryChannel.url} target="_blank" rel="noopener noreferrer" className="flex-1">
            <Button variant="outline" className="w-full border-green-400 text-green-700 hover:bg-green-50">
              🔗 {bank.primaryChannel.label} ←
            </Button>
          </a>
        </div>

        <div className="text-center">
          <a href={`/results?data=${encodedData}`} className="text-sm text-blue-600 hover:underline">← חזרה לתוצאות</a>
        </div>
      </div>
    </main>
  )
}

export default function LetterPage() {
  return <Suspense fallback={<div className="p-8 text-center">טוען...</div>}><LetterContent /></Suspense>
}
```

- [ ] **Step 2: Create `app/page.tsx` (Landing)**

```tsx
// app/page.tsx
import Link from 'next/link'
import { Button } from '@/components/ui/button'

export default function LandingPage() {
  return (
    <main className="min-h-screen bg-gradient-to-br from-blue-900 via-blue-800 to-indigo-900 flex flex-col items-center justify-center p-6 text-white">
      <div className="text-center max-w-xl">
        <div className="text-6xl mb-4">🏠</div>
        <h1 className="text-4xl font-bold mb-3">יועץ משכנתא AI</h1>
        <p className="text-blue-200 text-lg mb-8">
          קבל תמהיל מותאם אישית, insights חכמים, והמלצות איפה לפנות — בחינם ובדקות
        </p>

        <div className="grid grid-cols-3 gap-4 mb-10 text-center">
          {[
            { icon: '🧮', title: 'חישוב מדויק', desc: 'שפיצר + LTV + DTI' },
            { icon: '🤖', title: 'AI Insights', desc: 'ניתוח מותאם אישית' },
            { icon: '✉️', title: 'פנייה לבנקים', desc: 'טקסט מוכן + קישור' },
          ].map(({ icon, title, desc }) => (
            <div key={title} className="bg-white/10 rounded-xl p-4 backdrop-blur">
              <div className="text-3xl mb-1">{icon}</div>
              <div className="font-bold text-sm">{title}</div>
              <div className="text-blue-300 text-xs">{desc}</div>
            </div>
          ))}
        </div>

        <Link href="/simulator">
          <Button size="lg" className="bg-white text-blue-900 hover:bg-blue-50 font-bold text-lg px-10 py-4 rounded-xl shadow-lg">
            התחל סימולציה →
          </Button>
        </Link>

        <p className="text-blue-300 text-xs mt-6">
          * הכלי מיועד לסימולציה בלבד ואינו מהווה ייעוץ פיננסי מוסמך
        </p>
      </div>
    </main>
  )
}
```

- [ ] **Step 3: Commit**

```bash
git add app/letter/ app/page.tsx
git commit -m "feat: add letter page and landing page"
```

---

## Task 11: Environment, Deploy, Final Polish

**Files:**
- Create: `vercel.json`
- Modify: `.gitignore`

- [ ] **Step 1: Set ANTHROPIC_API_KEY in `.env.local`**

```bash
# Edit .env.local and add your real key:
# ANTHROPIC_API_KEY=sk-ant-...
echo "Ensure ANTHROPIC_API_KEY is set in .env.local"
```

- [ ] **Step 2: Add `.env.local` to `.gitignore`**

Verify `.gitignore` contains:
```
.env.local
.env*.local
```

If not, add it:
```bash
echo ".env.local" >> .gitignore
```

- [ ] **Step 3: Run full test suite**

```bash
npm test
```

Expected: all tests PASS

- [ ] **Step 4: Run dev server and smoke test**

```bash
npm run dev
```

Open http://localhost:3000 and verify:
- Landing page renders in Hebrew RTL
- Wizard navigates through all 4 steps
- Results page shows calculations
- Letter page renders with copy button

- [ ] **Step 5: Build for production**

```bash
npm run build
```

Expected: `✓ Compiled successfully` — no TypeScript errors, no build errors.

- [ ] **Step 6: Deploy to Vercel**

```bash
npx vercel --prod
```

When prompted, set environment variable: `ANTHROPIC_API_KEY`

- [ ] **Step 7: Final commit**

```bash
git add -A
git commit -m "feat: complete mortgage advisor v1 — wizard, calculations, AI insights, bank letters"
git push origin master
```

---

## Self-Review Checklist

**Spec coverage:**
- ✅ 4-step wizard (Tasks 7–8)
- ✅ All 7 mortgage tracks with default rates (Task 2 types, Task 3 calculations)
- ✅ Shpitzer formula (Task 3)
- ✅ LTV validation per transaction type (Task 3)
- ✅ DTI warning at 40% (Task 3)
- ✅ Sensitivity analysis +1%/+2% prime (Task 3)
- ✅ Variable unlinked max 33% validation (Task 3)
- ✅ Claude API for insights (Task 5)
- ✅ Claude API for AI mix recommendation (Tasks 5, 8)
- ✅ Graceful degradation on AI failure (Task 5)
- ✅ Bank data + recommendation logic (Task 4)
- ✅ Inquiry letter generator (Task 10)
- ✅ Copy to clipboard (Task 10)
- ✅ Bank contact links (Tasks 4, 10)
- ✅ RTL Hebrew throughout (Tasks 1, 7–10)
- ✅ Landing page (Task 10)
- ✅ Results page (Task 9)
- ✅ Vercel deploy (Task 11)

**Type consistency:** All types defined in Task 2 (`lib/types.ts`) and used consistently throughout. `MixTrack`, `MortgageMix`, `WizardData`, `CalculationResult`, `AIInsightsResult` — all referenced by exact same names across tasks.

**No placeholders:** All code blocks are complete. No TBDs.
