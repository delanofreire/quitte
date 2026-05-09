# Quitte Design System

## Overview

**Quitte** is a financial recovery and personal budget management platform, available on web and mobile. The product helps users take control of their finances — tracking debts, managing budgets, setting recovery goals, and monitoring progress over time. The brand communicates trust, clarity, and forward momentum.

**Sources:** Logo provided by client (`assets/quitte-logo.png`). Design system built from brand analysis.

---

## Content Fundamentals

- **Tone:** Calm, empowering, non-judgmental. Quitte meets users where they are and guides them forward — never shaming, always encouraging.
- **Voice:** Direct and clear. Short sentences. Action-oriented ("Veja seu progresso", "Adicionar meta", "Pagar agora").
- **Casing:** Title case for headings, sentence case for body, ALL CAPS for labels and badges only.
- **Person:** Second person (você / you). Avoid first person.
- **Emoji:** Not used. Iconography handles visual emphasis.
- **Numbers:** Always formatted with proper locale (R$ 1.200,00 for PT-BR).
- **Vibe:** Professional fintech meets personal coach.

---

## Visual Foundations

### Colors
Primary color extracted from logo: **Deep Teal (#1C6278)**. See `colors_and_type.css` for full palette.

- **Primary scale:** Deep teal family — trustworthy, stable, sophisticated.
- **Semantic:** Green for positive/success, Amber for warnings, Red for danger/debt.
- **Neutrals:** Cool gray family.
- **Background:** Light blue-gray (#EEF3F6) — lifts the teal without competing.

### Typography
- **Display / Headings:** Barlow — geometric, bold, condensed. Mirrors the logo's industrial sans character.
- **Body:** Inter — highly legible at small sizes, optimized for screen reading.
- **Mono:** JetBrains Mono — for financial figures, account numbers, code.

### Spacing
8px base grid. Tokens: 4, 8, 12, 16, 24, 32, 48, 64, 96px.

### Radii
- `sm`: 4px — inline elements, badges
- `md`: 8px — inputs, chips
- `lg`: 12px — cards
- `xl`: 16px — modals, panels
- `full`: 9999px — pills, avatars

### Shadows
Subtle elevation system. Cards use soft `box-shadow` with teal tint. No harsh drop shadows.

### Backgrounds
- Light mode: #EEF3F6 background, #FFFFFF card surfaces
- Teal gradient header bands for dashboard hero areas
- No textures or patterns — clean, flat with selective elevation

### Animation
- Easing: `cubic-bezier(0.4, 0, 0.2, 1)` (Material standard ease)
- Duration: 150ms for micro-interactions, 300ms for transitions, 500ms for page-level
- Hover: background color shift + slight shadow lift
- Press: scale(0.98)
- No bounces — financial context calls for composed, settled motion

### Cards
Rounded corners (12px), white background, subtle shadow (`0 2px 8px rgba(28,98,120,0.08)`), no border. Active state adds a teal border.

### Imagery
Cool-toned photography when used. Financial charts use the brand teal palette. Illustrations are flat and geometric (not hand-drawn).

### Icons
Lucide Icons (CDN) — 1.5px stroke weight, rounded caps. Always at 16, 20, or 24px.

---

## Iconography

**System:** [Lucide Icons](https://lucide.dev/) via CDN — consistent stroke weight (1.5px), minimal, modern.
**Sizes:** 16px (inline), 20px (UI elements), 24px (navigation/headers).
**Color:** Inherits text color or uses semantic color when indicating state.
**Never:** Filled icons, emoji as icons, decorative SVGs.

---

## File Index

```
README.md                  ← This file
SKILL.md                   ← Agent skill definition
colors_and_type.css        ← CSS variables (colors + type)
assets/
  quitte-logo.png          ← Primary logo
preview/
  colors-primary.html      ← Primary teal scale
  colors-neutrals.html     ← Neutral gray scale
  colors-semantic.html     ← Success/Warning/Danger
  type-scale.html          ← Heading + body type specimens
  type-specimens.html      ← Font weight + style matrix
  spacing-tokens.html      ← Spacing + radii + shadows
  comp-buttons.html        ← Button variants + states
  comp-inputs.html         ← Form inputs + selects
  comp-badges.html         ← Badges + chips + tags
  comp-cards.html          ← Card variants
  comp-data.html           ← Data display (charts, progress, stats)
ui_kits/
  web/index.html           ← Web dashboard UI kit
  mobile/index.html        ← Mobile app UI kit
```
