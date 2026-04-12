# Mortgage Advisor — Design Spec
**Date:** 2026-04-13  
**Project:** mortgage-advisor  
**Status:** Approved

---

## Overview

An AI-powered mortgage simulator and advisory tool for the Israeli market. Enables any person — from first-time homebuyer to experienced investor — to understand their mortgage options, receive personalized AI insights, get bank recommendations, and generate a ready-to-send inquiry letter.

---

## Goals

1. Simulate mortgage scenarios with accurate Israeli market calculations
2. Recommend a tailored mortgage mix (תמהיל) with AI-backed reasoning
3. Compare which banks are most suitable for the user's profile
4. Generate a personalized inquiry text + direct contact link for each bank
5. Present everything in a friendly, accessible Hebrew RTL UI

---

## Stack

| Layer | Technology |
|-------|------------|
| Framework | Next.js 14 (App Router) |
| Language | TypeScript |
| Styling | Tailwind CSS + shadcn/ui |
| AI | Claude API (Anthropic) via API Routes |
| Deployment | Vercel |
| Language/Direction | Hebrew, RTL |

---

## Pages & Routing

```
/                → Landing page + quick entry to simulator
/simulator       → Main simulator (4-step wizard)
/results         → Results: calculations + AI insights + bank recommendations
/letter/[bankId] → Generated inquiry letter + bank contact link
```

---

## Simulator Wizard (4 Steps)

### Step 1 — Borrower Profile
- Age
- Marital status
- Monthly net income (individual or couple)
- Employment type: salaried / self-employed / mixed
- Government eligibility (זכאות): first apartment, discharged soldier, etc.

### Step 2 — Property Details
- Property price
- Down payment (amount + auto-calculated percentage)
- Transaction type: first apartment / investor / מחיר למשתכן / upgrade (שיפור דיור)
- Geographic region (affects bank recommendations)

### Step 3 — Preferences
- Loan term: 10–30 years
- Risk level: conservative / balanced / aggressive
- Priority: low monthly payment / low total cost / maximum flexibility

### Step 4 — Mortgage Mix (תמהיל)
- Manual selection per track (amounts + percentages)
- OR "Choose for me (AI)" button → returns recommended mix with reasoning

---

## Supported Mortgage Tracks

| Track | Description | Default Rate (editable) |
|-------|-------------|------------------------|
| פריים (Prime) | Bank of Israel rate + margin, variable | P−0.5% (P=4.75% → 4.25%) |
| קבועה צמודה (ק"צ) | Fixed rate + CPI-linked | 3.2% + CPI |
| קבועה לא צמודה (קל"צ) | Fixed rate, not CPI-linked | 5.5% |
| משתנה צמודה | Variable every 5 years + CPI | 2.8% + CPI |
| משתנה לא צמודה | Variable every 5 years, no CPI | 5.1% |
| זכאות | Bank of Israel track for eligible borrowers | 3.0% fixed (BOI rate) |
| גרייס | Principal deferred for defined period | Same as underlying track |

> All default rates are editable by the user in the wizard (advanced mode). Rates reflect approximate market rates as of early 2026 and are used for simulation only — not financial advice.

---

## Calculation Engine (TypeScript, client-free)

All financial calculations run server-side in TypeScript without AI dependency:

- **Monthly payment** per track — Shpitzer (שפיצר) amortization formula
- **Total cost** over full loan period (principal + interest + CPI projections)
- **Interest rate sensitivity** — impact of +1% / +2% prime rate change
- **LTV (Loan-to-Value)** validation against Bank of Israel regulations:
  - First apartment: max 75%
  - Investor: max 50%
  - מחיר למשתכן: max 90%
- **Debt-to-income ratio** — warning if monthly payment exceeds 40% of net income
- **Recommended mix validation** — ensures compliance with Bank of Israel track limits (max 1/3 variable unlinked)

---

## AI Engine (Claude API)

Called from a Next.js API Route after calculations complete. Receives full borrower profile + calculated results. Returns structured JSON with:

1. **Borrower profile analysis** — personalized observations based on age, income, employment
2. **Mix reasoning** — why this specific תמהיל suits this borrower's situation
3. **3 key insights** — risks, opportunities, important considerations (formatted as bullets)
4. **Bank comparison** — which 2–3 banks are most suitable and why

### Prompt strategy
- System prompt establishes role as Israeli mortgage expert
- User message contains structured borrower data + calculated results
- Response requested as structured JSON to enable clean UI rendering
- Fallback: if API call fails, show calculation results only with static tips

### AI insights display
```
┌─────────────────────────────────────────┐
│ 💡 מה כדאי שתדע על התמהיל שלך         │
│                                         │
│ ✅ התמהיל מאוזן ומתאים לפרופיל שלך    │
│ ⚠️  43% מהכנסתך → גבול עליון סביר    │
│ 📈 אם הפריים יעלה ב-1%, ההחזר יעלה... │
│ 🏦 לפרופיל שלך, מזרחי-טפחות עשוי...  │
└─────────────────────────────────────────┘
```

---

## Bank Recommendations

### Banks included

| Bank | Primary contact channel |
|------|------------------------|
| בנק הפועלים | Online form + WhatsApp |
| בנק לאומי | Live chat + form |
| מזרחי-טפחות | Online form (mortgage specialist) |
| בנק דיסקונט | Form + mortgage hotline |
| הבנק הבינלאומי | Form + call scheduling |

### Recommendation logic
AI selects 2–3 banks based on:
- **Self-employed** → מזרחי-טפחות (known flexibility)
- **Eligible borrowers (זכאות)** → הפועלים / לאומי
- **Large loans** → large banks (הפועלים, לאומי)
- **Periphery regions** → banks with local presence
- **First-time buyers** → banks with digital onboarding

---

## Inquiry Letter Generator

Triggered from results page. Generates a personalized Hebrew inquiry text:

```
שלום,

אני [שם] בן/בת [גיל], שכיר/ה עם הכנסה חודשית של ₪[X],
ומעוניין/ת לקבל הצעה למשכנתא לרכישת דירה [ראשונה / להשקעה].

פרטי הבקשה:
• מחיר נכס: ₪[X]
• הלוואה מבוקשת: ₪[X] ([Y]% מימון)
• תקופה: [Z] שנים
• תמהיל מבוקש:
  - פריים: [X]%
  - קל"צ: [X]%
  - ק"צ: [X]%

יחס החזר להכנסה: [X]%

אשמח לקבל הצעה מותאמת.
תודה
```

Below the letter:
- Copy-to-clipboard button
- Direct link to bank's fastest contact channel (form / WhatsApp / chat)

---

## UI/UX Principles

- **RTL Hebrew throughout** — all layouts, inputs, text direction
- **Mobile-first** — wizard works well on phone
- **Progressive disclosure** — show complexity only when needed (advanced options collapsed by default)
- **Color-coded risk indicators** — green/yellow/red for key metrics
- **Loading states** — skeleton screens while AI processes
- **Graceful degradation** — if Claude API is unavailable, show calculations + static tips

---

## Project Structure

```
mortgage-advisor/
├── app/
│   ├── page.tsx                  # Landing page
│   ├── simulator/
│   │   └── page.tsx              # Wizard
│   ├── results/
│   │   └── page.tsx              # Results + AI insights
│   ├── letter/
│   │   └── [bankId]/page.tsx     # Inquiry letter
│   └── api/
│       ├── ai-insights/route.ts  # Claude API call
│       └── calculate/route.ts    # Calculation engine
├── lib/
│   ├── calculations.ts           # All mortgage math
│   ├── banks.ts                  # Bank data + contact links
│   ├── ai-prompt.ts              # Claude prompt builder
│   └── types.ts                  # Shared TypeScript types
├── components/
│   ├── wizard/                   # Step components
│   ├── results/                  # Charts, insights cards
│   └── ui/                       # shadcn/ui components
└── docs/
    └── superpowers/specs/
        └── 2026-04-13-mortgage-advisor-design.md
```

---

## Data Flow

```
User fills wizard
      ↓
POST /api/calculate
  → runs Shpitzer + LTV + DTI calculations
  → returns structured CalculationResult
      ↓
POST /api/ai-insights
  → sends borrower profile + CalculationResult to Claude
  → returns InsightsResult (JSON)
      ↓
Results page renders:
  → amortization table + charts
  → AI insights panel
  → bank recommendation cards (2–3 banks)
  → "Generate inquiry" button per bank
      ↓
/letter/[bankId]
  → renders personalized inquiry text
  → copy button + direct bank contact link
```

---

## Error Handling

| Scenario | Behavior |
|----------|----------|
| Claude API timeout / failure | Show calculations + static tips, no error shown to user |
| LTV exceeds regulation limit | Red warning inline in wizard, blocks progression |
| DTI > 40% | Yellow warning, user can proceed with acknowledgment |
| Invalid input (negative values, etc.) | Inline field validation with Hebrew error messages |

---

## Out of Scope (v1)

- User accounts / saved simulations
- Real-time rate scraping from banks
- Actual loan application submission
- PDF export (plain text copy is sufficient for v1)
- English language support
