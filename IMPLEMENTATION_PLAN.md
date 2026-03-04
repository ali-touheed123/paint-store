# Paint Store — Implementation Plan

## Overview

This document is the single source of truth for building the Paint Store e-commerce application. It incorporates the premium **Paint Calculator** feature with coverage-based calculations, tin size breakdown, and WhatsApp integration.

---

## Table of Contents

1. [Tech Stack](#tech-stack)
2. [Color System & Design Tokens](#color-system--design-tokens)
3. [Project Structure](#project-structure)
4. [Environment Variables](#environment-variables)
5. [Database Schema](#database-schema)
6. [Phase 1 — Foundation Setup](#phase-1--foundation-setup)
7. [Phase 2 — Paint Calculator Feature](#phase-2--paint-calculator-feature)
8. [Phase 3 — Core E-Commerce Pages](#phase-3--core-e-commerce-pages)
9. [Phase 4 — Admin Dashboard](#phase-4--admin-dashboard)
10. [Phase 5 — Integration & Polish](#phase-5--integration--polish)
11. [Phase 6 — Testing & SEO](#phase-6--testing--seo)
12. [Calculator Business Logic Reference](#calculator-business-logic-reference)
13. [Considerations & Edge Cases](#considerations--edge-cases)

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 14 (App Router, TypeScript) |
| Styling | Tailwind CSS |
| UI Components | shadcn/ui |
| Animations | Framer Motion |
| Icons | Lucide React |
| Database | Supabase (PostgreSQL) |
| Auth | Supabase Auth |
| Image Hosting | Supabase Storage (or direct `/images` path) |
| Deployment | Vercel |

---

## Color System & Design Tokens

```typescript
// tailwind.config.ts
colors: {
  navy:      '#1a365d',   // Primary brand colour
  gold:      '#d4af37',   // Accent / CTA highlight
  'gold-pale': '#f9f5e7', // Subtle background tint
  emerald:   '#10b981',   // Success / WhatsApp green
}
```

---

## Project Structure

```
paint-store-app/
├── app/
│   ├── (public)/
│   │   ├── page.tsx                        # Home — hero, featured, calculator CTA
│   │   ├── products/
│   │   │   ├── page.tsx                    # Product listing with filters
│   │   │   └── [slug]/page.tsx             # Product detail + calculator CTA
│   │   ├── brands/
│   │   │   └── [brand]/page.tsx            # Brand product listing
│   │   ├── calculator/
│   │   │   └── page.tsx                    # Standalone calculator page
│   │   └── visualizer/
│   │       └── page.tsx                    # Colour visualizer
│   ├── admin/
│   │   ├── layout.tsx
│   │   ├── page.tsx                        # Admin dashboard
│   │   ├── products/page.tsx
│   │   ├── brands/page.tsx
│   │   └── calculator/page.tsx             # Manage paint types & adjustments
│   └── api/
│       ├── calculator/
│       │   ├── paint-types/route.ts        # GET paint types
│       │   └── inquiries/route.ts          # POST save calculation
│       └── products/
│           └── route.ts
├── components/
│   ├── calculator/
│   │   ├── index.ts
│   │   ├── paint-calculator.tsx            # Main container + state
│   │   ├── dimension-inputs.tsx            # Room L × W × H inputs
│   │   ├── direct-area-input.tsx           # Manual sq ft entry
│   │   ├── paint-type-selector.tsx         # Paint type dropdown
│   │   ├── surface-adjustments.tsx         # Doors & windows counters
│   │   ├── calculation-result.tsx          # Results card
│   │   ├── tin-breakdown-display.tsx       # Visual drum/gallon/quarter display
│   │   └── whatsapp-button.tsx             # Pre-filled WhatsApp CTA
│   ├── ui/                                 # shadcn/ui base + custom
│   │   ├── counter-input.tsx               # ±1 increment input
│   │   └── calculator-card.tsx             # Styled card wrapper
│   ├── layout/
│   │   ├── navbar.tsx
│   │   └── footer.tsx
│   └── products/
│       ├── product-card.tsx
│       └── product-grid.tsx
├── lib/
│   ├── calculator/
│   │   ├── utils.ts                        # Core calculation functions
│   │   └── constants.ts                    # Paint types, tin sizes, factors
│   ├── supabase.ts
│   └── utils.ts
├── types/
│   ├── calculator.ts                       # Calculator-specific interfaces
│   └── index.ts                            # Re-exports
├── supabase/
│   └── migrations/
│       ├── 001_products.sql
│       ├── 002_brands.sql
│       └── 003_calculator_tables.sql
├── __tests__/
│   └── calculator/
│       └── utils.test.ts
├── public/
│   └── images/ -> (symlink or copy of /images asset repo)
├── .env.local.example
└── next.config.js
```

---

## Environment Variables

```bash
# .env.local.example

# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key

# WhatsApp
NEXT_PUBLIC_WHATSAPP_NUMBER=923475658761
NEXT_PUBLIC_WHATSAPP_MESSAGE_TEMPLATE="Hi! The calculator says I need {litres} litres of {paintType} for {totalArea} sq/ft. Please help me choose the right product."

# Branding
NEXT_PUBLIC_COMPANY_NAME="The Paint Hub"
NEXT_PUBLIC_SITE_URL=https://thepainthouse.pk
```

---

## Database Schema

### `supabase/migrations/003_calculator_tables.sql`

```sql
-- Paint types configurable by admin
CREATE TABLE paint_types (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug        TEXT UNIQUE NOT NULL,
  label       TEXT NOT NULL,
  category    TEXT NOT NULL,             -- 'wall' | 'wood' | 'metal' | 'specialty'
  unit        TEXT NOT NULL DEFAULT 'litre',  -- 'litre' | 'kg'
  coverage    NUMERIC(8,2) NOT NULL,     -- sq ft per unit
  is_active   BOOLEAN NOT NULL DEFAULT TRUE,
  sort_order  INT NOT NULL DEFAULT 0,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Surface adjustments (doors / windows add to paintable area)
CREATE TABLE surface_adjustments (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug        TEXT UNIQUE NOT NULL,
  label       TEXT NOT NULL,
  area_sq_ft  NUMERIC(6,2) NOT NULL,
  is_active   BOOLEAN NOT NULL DEFAULT TRUE
);

-- Track user calculations for analytics / lead gen
CREATE TABLE calculator_inquiries (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  paint_type_slug TEXT NOT NULL,
  input_mode      TEXT NOT NULL,   -- 'dimensions' | 'direct'
  room_length     NUMERIC,
  room_width      NUMERIC,
  room_height     NUMERIC,
  direct_area     NUMERIC,
  doors           INT DEFAULT 0,
  windows         INT DEFAULT 0,
  coats           INT DEFAULT 2,
  total_area      NUMERIC NOT NULL,
  net_area        NUMERIC NOT NULL,
  litres_needed   NUMERIC NOT NULL,
  drums           INT NOT NULL,
  gallons         INT NOT NULL,
  quarters        INT NOT NULL,
  whatsapp_opened BOOLEAN DEFAULT FALSE,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Seed default paint types
INSERT INTO paint_types (slug, label, category, unit, coverage, sort_order) VALUES
  ('primer',        'Primer / Sealer',   'wall',      'litre', 400, 1),
  ('emulsion',      'Emulsion',          'wall',      'litre', 350, 2),
  ('weather-shield','Weather Shield',    'wall',      'litre', 300, 3),
  ('enamel',        'Synthetic Enamel',  'metal',     'litre', 400, 4),
  ('putty',         'Wall Putty',        'wall',      'kg',     20, 5),
  ('texture',       'Texture Paint',     'wall',      'litre',  25, 6);

-- Seed default surface adjustments
INSERT INTO surface_adjustments (slug, label, area_sq_ft) VALUES
  ('door',   'Door',   21),
  ('window', 'Window', 15);
```

---

## Phase 1 — Foundation Setup

### Step 1 — Type Definitions (`types/calculator.ts`)

```typescript
export interface PaintType {
  id: string;
  slug: string;
  label: string;
  category: 'wall' | 'wood' | 'metal' | 'specialty';
  unit: 'litre' | 'kg';
  coverage: number;      // sq ft per unit
  isActive: boolean;
  sortOrder: number;
}

export interface TinBreakdown {
  drums: number;         // 16 L (or 16 kg)
  gallons: number;       // 3.64 L
  quarters: number;      // 0.91 L
  totalUnits: number;
  exactUnits: number;    // raw calculated value before rounding up
}

export interface SurfaceAdjustment {
  slug: string;
  label: string;
  areaSqFt: number;
  count: number;
}

export type InputMode = 'dimensions' | 'direct';

export interface CalculationParams {
  inputMode: InputMode;
  roomLength?: number;
  roomWidth?: number;
  roomHeight?: number;
  directArea?: number;
  paintType: PaintType;
  doors: number;
  windows: number;
  coats: number;
}

export interface PaintCalculation {
  wallArea: number;        // raw 2×(L+W)×H or direct input
  adjustmentArea: number;  // doors × 21 + windows × 15
  totalArea: number;       // wallArea + adjustmentArea
  netArea: number;         // totalArea × coats × wastage factor
  unitsNeeded: number;     // netArea / coverage (exact)
  tinBreakdown: TinBreakdown;
  unit: 'litre' | 'kg';
}
```

### Step 2 — Calculator Constants (`lib/calculator/constants.ts`)

```typescript
export const WASTAGE_FACTOR = 1.1;

export const TIN_SIZES = {
  DRUM:    { label: 'Drum',    litres: 16,   kg: 16   },
  GALLON:  { label: 'Gallon',  litres: 3.64, kg: 3.64 },
  QUARTER: { label: 'Quarter', litres: 0.91, kg: 0.91 },
} as const;

export const SURFACE_ADJUSTMENTS = {
  door:   { label: 'Door',   areaSqFt: 21 },
  window: { label: 'Window', areaSqFt: 15 },
} as const;

export const DEFAULT_PAINT_TYPES: PaintType[] = [
  { id: '1', slug: 'primer',         label: 'Primer / Sealer',  category: 'wall',  unit: 'litre', coverage: 400, isActive: true, sortOrder: 1 },
  { id: '2', slug: 'emulsion',       label: 'Emulsion',         category: 'wall',  unit: 'litre', coverage: 350, isActive: true, sortOrder: 2 },
  { id: '3', slug: 'weather-shield', label: 'Weather Shield',   category: 'wall',  unit: 'litre', coverage: 300, isActive: true, sortOrder: 3 },
  { id: '4', slug: 'enamel',         label: 'Synthetic Enamel', category: 'metal', unit: 'litre', coverage: 400, isActive: true, sortOrder: 4 },
  { id: '5', slug: 'putty',          label: 'Wall Putty',       category: 'wall',  unit: 'kg',    coverage: 20,  isActive: true, sortOrder: 5 },
  { id: '6', slug: 'texture',        label: 'Texture Paint',    category: 'wall',  unit: 'litre', coverage: 25,  isActive: true, sortOrder: 6 },
];
```

### Step 3 — Core Calculation Utilities (`lib/calculator/utils.ts`)

```typescript
import { WASTAGE_FACTOR, TIN_SIZES } from './constants';
import type { CalculationParams, PaintCalculation, TinBreakdown } from '@/types/calculator';

export function calculateRoomArea(length: number, width: number, height: number): number {
  return 2 * (length + width) * height;
}

export function calculateTinBreakdown(units: number, tinSizes: typeof TIN_SIZES): TinBreakdown {
  const size = tinSizes.DRUM.litres;
  const drums = Math.floor(units / tinSizes.DRUM.litres);
  const afterDrums = units % tinSizes.DRUM.litres;
  const gallons = Math.floor(afterDrums / tinSizes.GALLON.litres);
  const afterGallons = afterDrums % tinSizes.GALLON.litres;
  const quarters = Math.ceil(afterGallons / tinSizes.QUARTER.litres);
  return { drums, gallons, quarters, totalUnits: drums + gallons + quarters, exactUnits: units };
}

export function calculatePaintRequirements(params: CalculationParams): PaintCalculation {
  const { inputMode, roomLength, roomWidth, roomHeight, directArea, paintType, doors, windows, coats } = params;

  const wallArea = inputMode === 'dimensions'
    ? calculateRoomArea(roomLength!, roomWidth!, roomHeight!)
    : directArea!;

  const adjustmentArea = doors * 21 + windows * 15;
  const totalArea = wallArea + adjustmentArea;
  const netArea = totalArea * coats * WASTAGE_FACTOR;
  const unitsNeeded = netArea / paintType.coverage;

  return {
    wallArea,
    adjustmentArea,
    totalArea,
    netArea,
    unitsNeeded,
    tinBreakdown: calculateTinBreakdown(unitsNeeded, TIN_SIZES),
    unit: paintType.unit,
  };
}

export function formatTinBreakdown(breakdown: TinBreakdown): string {
  const parts: string[] = [];
  if (breakdown.drums)   parts.push(`${breakdown.drums} Drum${breakdown.drums > 1 ? 's' : ''} (16L)`);
  if (breakdown.gallons) parts.push(`${breakdown.gallons} Gallon${breakdown.gallons > 1 ? 's' : ''} (3.64L)`);
  if (breakdown.quarters) parts.push(`${breakdown.quarters} Quarter${breakdown.quarters > 1 ? 's' : ''} (0.91L)`);
  return parts.length ? parts.join(' + ') : 'Less than 1 quarter';
}

export function generateWhatsAppMessage(
  calc: PaintCalculation,
  paintTypeLabel: string,
  whatsappNumber: string,
  template: string,
): string {
  const message = template
    .replace('{litres}', calc.unitsNeeded.toFixed(2))
    .replace('{paintType}', paintTypeLabel)
    .replace('{totalArea}', calc.totalArea.toFixed(0));
  return `https://wa.me/${whatsappNumber}?text=${encodeURIComponent(message)}`;
}
```

---

## Phase 2 — Paint Calculator Feature

### Step 4 — Component Architecture

Each component has a single responsibility. State lives in the parent `paint-calculator.tsx`.

#### `components/calculator/paint-calculator.tsx` (main container)

- Manages `inputMode`, `params`, and `result` state
- Renders tabs: **"Room Dimensions"** | **"Direct Area"**
- Calls `calculatePaintRequirements()` on every param change (debounced 300 ms)
- Renders `<CalculationResult />` in a sticky right column on desktop, collapsed below on mobile

#### `components/calculator/dimension-inputs.tsx`

- Three number inputs: Length, Width, Height (all in feet)
- Shows live area preview: `2 × (L+W) × H = {area} sq ft`

#### `components/calculator/direct-area-input.tsx`

- Single number input for total wall area in sq ft
- Helper text: "Measure each wall and add them together"

#### `components/calculator/paint-type-selector.tsx`

- `<Select>` component populated from `PAINT_TYPES` (or API)
- Shows unit alongside each option: "Emulsion — 350 sq ft / litre"
- Groups by `category`: Wall Paints | Metal & Wood | Specialty

#### `components/calculator/surface-adjustments.tsx`

- `<CounterInput>` for number of doors (default 0)
- `<CounterInput>` for number of windows (default 0)
- `<CounterInput>` for number of coats (default 2, min 1, max 5)
- Shows running area total: "+{area} sq ft added"

#### `components/calculator/calculation-result.tsx`

- Shows: Total Area, Net Area (after wastage), Units Needed
- Renders `<TinBreakdownDisplay />`
- Renders `<WhatsAppButton />`
- Animates in with Framer Motion `fadeInUp` on result change

#### `components/calculator/tin-breakdown-display.tsx`

- Visual icon + count row for each tin size (drum, gallon, quarter)
- Uses navy/gold colour tokens
- Greyed out when count is 0

#### `components/calculator/whatsapp-button.tsx`

- Renders a large emerald green button with WhatsApp icon
- Opens `generateWhatsAppMessage()` URL in new tab
- Fires POST to `/api/calculator/inquiries` (fire-and-forget) to track the lead
- Shows "Chat on WhatsApp" label

#### `components/ui/counter-input.tsx`

- `−` button / number display / `+` button
- Accepts `min`, `max`, `value`, `onChange`
- Accessible: uses `aria-label="Decrease doors"` etc.

### Step 5 — Calculator Page (`app/calculator/page.tsx`)

- Full-page layout with two-column grid (inputs left, result right) on `lg+`
- SEO title: "Paint Calculator — How Much Paint Do I Need? | The Paint Hub"
- Breadcrumb: Home › Calculator
- FAQ accordion below the calculator for SEO

### Step 6 — API Routes

#### `app/api/calculator/paint-types/route.ts`
```typescript
// GET — returns active paint types ordered by sort_order
// Falls back to DEFAULT_PAINT_TYPES if Supabase is unavailable
```

#### `app/api/calculator/inquiries/route.ts`
```typescript
// POST — saves a calculator inquiry row
// Body: CalculationParams + PaintCalculation result
// Returns 201 Created with inquiry ID
```

---

## Phase 3 — Core E-Commerce Pages

### Home Page (`app/page.tsx`)

Sections (top to bottom):
1. Hero — full-width banner with CTA "Shop Paint" and "Try Calculator"
2. Brand Logos strip — 10 brand logos in a horizontal scroll
3. Featured Products grid — 8 bestsellers
4. **Calculator CTA section** — "Not sure how much paint you need?" with `<PaintCalculator />` embedded or a link to `/calculator`
5. Category cards — Decorative / Industrial / Auto
6. Colour Visualizer teaser

### Product Listing (`app/products/page.tsx`)

- Filter sidebar: brand, category, paint type
- Responsive grid: 2 cols mobile, 3 cols tablet, 4 cols desktop
- Each card: product image, name, brand badge, "Calculate Needs" shortcut icon

### Product Detail (`app/products/[slug]/page.tsx`)

- Image gallery (product images from `/images/products/{brand}/`)
- Product description, brand, category, available sizes
- **"Calculate Paint Needs" section** — renders `<PaintCalculator />` with `defaultPaintType` pre-selected from product category
- Related products carousel

---

## Phase 4 — Admin Dashboard

### Calculator Settings Page (`app/admin/calculator/page.tsx`)

Features:
- **Paint Types table** — edit `label`, `coverage`, `unit`, `isActive`, `sortOrder`
- **Surface Adjustments table** — edit door/window `areaSqFt` values
- **Inquiry Stats** — total calculations, WhatsApp clicks, most popular paint type
- All edits persist to Supabase via server actions

---

## Phase 5 — Integration & Polish

### WhatsApp Number Configuration

The WhatsApp number `923475658761` is read from `NEXT_PUBLIC_WHATSAPP_NUMBER`. The admin can update it in `.env.local` without touching code. For runtime configurability, add a `settings` table to Supabase and fetch it in `whatsapp-button.tsx`.

### Pre-selecting Paint Type from Product Page

Pass `?paintType=emulsion` (or the product's `paintTypeSlug`) as a query param to `/calculator`. The calculator reads `useSearchParams()` and sets the default selection accordingly.

### Mobile Optimisations

- Collapsible "Advanced Options" section (coats, surface adjustments) — hidden by default on mobile
- Sticky result card at bottom of screen on mobile using `position: sticky`
- `inputMode="numeric"` on all number inputs to trigger numeric keypad on iOS/Android

---

## Phase 6 — Testing & SEO

### Unit Tests (`__tests__/calculator/utils.test.ts`)

```typescript
describe('calculateRoomArea', () => {
  it('computes 2×(L+W)×H correctly', ...)
  it('handles zero dimensions', ...)
})

describe('calculateTinBreakdown', () => {
  it('returns only drums for large quantities', ...)
  it('fills drums first, then gallons, then quarters', ...)
  it('rounds quarters up (never short-ship)', ...)
  it('handles exact multiples without remainder', ...)
  it('handles less than one quarter', ...)
})

describe('calculatePaintRequirements', () => {
  it('applies wastage factor of 1.1', ...)
  it('multiplies by coats correctly', ...)
  it('adds door and window areas', ...)
  it('uses direct area when inputMode is "direct"', ...)
})

describe('generateWhatsAppMessage', () => {
  it('encodes special characters in message', ...)
  it('substitutes all template placeholders', ...)
})
```

### SEO

- `/calculator` page: `<title>Paint Calculator — How Much Paint Do I Need?</title>`
- FAQ schema (`application/ld+json`) with questions like "How do I calculate how much paint I need?"
- Open Graph image showing the calculator UI

---

## Calculator Business Logic Reference

### Formulas

```
wallArea        = 2 × (length + width) × height           [room dimensions mode]
wallArea        = directArea                                [direct mode]

adjustmentArea  = (doors × 21) + (windows × 15)            [sq ft]

totalArea       = wallArea + adjustmentArea
netArea         = totalArea × coats × 1.1                  [1.1 = 10% wastage]
unitsNeeded     = netArea ÷ coverage                       [litres or kg, exact]

drums           = floor(unitsNeeded ÷ 16)
remaining       = unitsNeeded mod 16
gallons         = floor(remaining ÷ 3.64)
remaining       = remaining mod 3.64
quarters        = ceil(remaining ÷ 0.91)
```

### Coverage Rates

| Paint Type | Coverage | Unit |
|---|---|---|
| Primer / Sealer | 400 | sq ft / litre |
| Emulsion | 350 | sq ft / litre |
| Weather Shield | 300 | sq ft / litre |
| Synthetic Enamel | 400 | sq ft / litre |
| Wall Putty | 20 | sq ft / kg |
| Texture Paint | 25 | sq ft / litre |

### Tin Sizes

| Size | Volume |
|---|---|
| Drum | 16 L |
| Gallon | 3.64 L |
| Quarter | 0.91 L |

### Surface Area Add-ons

| Surface | Area Added |
|---|---|
| Door | +21 sq ft |
| Window | +15 sq ft |

> **Note:** Doors and windows **add** to the painted area (they represent paintable frame/surround surfaces, not unpainted voids). If a future requirement changes this to subtract unpainted glass/panel areas, update `adjustmentArea` to use subtraction and update the UI label accordingly.

---

## Considerations & Edge Cases

1. **Putty is in kg, not litres** — The UI must show "kg" as the unit when putty is selected. The tin size labels should read "Bag (20kg)", "Bag (5kg)" etc. for putty, not drums/gallons. Implement a `PaintType.tinSizeOverrides` field to support this.

2. **Admin configurability** — Coverage rates should be editable in the admin without a redeploy. The `paint_types` Supabase table feeds the calculator. The Next.js API route falls back to `DEFAULT_PAINT_TYPES` constants if the DB is unreachable.

3. **Quarters must never short-ship** — Always `Math.ceil` the quarter calculation. Never `Math.floor` or round to nearest.

4. **Zero result guard** — If `unitsNeeded <= 0` (e.g. all dimensions are 0), show an empty state with "Please enter valid dimensions."

5. **Coats minimum** — Coats counter should enforce `min=1`. Do not allow 0 coats.

6. **Performance** — Calculator logic is synchronous and trivial. Use `useMemo` on `calculatePaintRequirements(params)` keyed to `params` to prevent recalculations during unrelated re-renders.

7. **Accessibility** — Counter inputs need explicit `aria-label` on each button. Live region (`aria-live="polite"`) on the results card so screen readers announce new values.

8. **WhatsApp deeplink** — The URL format `https://wa.me/{number}?text={encoded}` works for both mobile app and WhatsApp Web. The number must include country code with no `+` prefix (e.g. `923475658761`).

9. **Analytics** — Track these events: `calculator_calculation_performed`, `calculator_whatsapp_clicked`, `calculator_paint_type_selected`. Fire them via a thin analytics wrapper so the provider can be swapped later.

10. **Tin size availability** — Not all products come in all tin sizes. Add an optional `availableTinSizes: ('drum' | 'gallon' | 'quarter')[]` field to `PaintType` for future UX improvements (e.g. greying out unavailable sizes in the breakdown display).
