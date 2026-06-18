# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev      # Start development server
npm run build    # Production build
npm run lint     # Run ESLint
npm start        # Run production server
```

No test suite is configured.

## Environment

`.env.local` vars:

```
RESEND_API_KEY=...                 # required — pilot form emails + blog subscribe
RESEND_AUDIENCE_ID=...             # required for /api/subscribe — Resend Audiences ID
UPSTASH_REDIS_REST_URL=...         # optional — enables per-IP rate limit (5/hour)
UPSTASH_REDIS_REST_TOKEN=...       # required if UPSTASH_REDIS_REST_URL is set
```

Rate limiting in `app/api/send/route.ts` and `app/api/subscribe/route.ts` is gated on `UPSTASH_REDIS_REST_URL` — if unset, the limiter is skipped entirely (fine for local dev). `/api/subscribe` returns 503 if `RESEND_AUDIENCE_ID` is unset.

## Architecture

Multi-route marketing site. **Next.js 16 App Router**, **React 19**, **TypeScript**, **Tailwind CSS v4**.

### Routes

- `app/page.tsx` — Home. Composes section components from `components/sections/` directly (no client wrapper).
- `app/blog/page.tsx`, `app/blog/[slug]/page.tsx` — Build log index + post pages. Posts authored as TSX in `content/posts/`, registered in `content/posts/index.ts`.
- `app/pilot/page.tsx` — Pilot application landing.
- `app/investors/`, `app/our-story/`, `app/why-salency/`, `app/pricing/`, `app/changelog/`, `app/memory/` — Standalone marketing/content routes.
- `app/privacy/page.tsx`, `app/terms/page.tsx` — Legal.
- `app/sitemap.ts`, `app/robots.ts`, `app/opengraph-image.tsx` — SEO/social.

### Page composition

- `app/layout.tsx` — Root. Loads fonts, mounts `<JsonLd />` (Organization + WebSite schema), `<PilotModal />` (globally available), `<Analytics />`.
- `components/MarketingHeader.tsx` — Top nav, used on all marketing pages.
- `components/sections/*` — Page-level sections (HeroSection, HubSection, HowItWorksSection, NotDoSection, CtaSection, SiteFooter, ProblemSection, ThesisSection, MemorySection, MemoryStackSection, NotetakerSection, CompoundsSection, NotForYouSection, OurStorySection, InvestorsSection, PilotApplication, PilotCtaSection, ComingSoon). Most are `'use client'`.
- `components/HeroArtifact.tsx` — Animated hero visual. `components/FounderAvatar.tsx`, `lib/founders.tsx` — Team data.
- Blog chrome: `components/PostToc.tsx`, `components/ShareButtons.tsx`, `components/SubscribeForm.tsx`. Legal chrome: `components/LegalToc.tsx`.

When editing marketing copy: start in `components/sections/` — section names match home-page order (Hero, Hub, HowItWorks, NotDo, Cta).

### Pilot form flow

The pilot form is a **global modal**, not inline on the page.

- `components/PilotModal.tsx` exports a `<PilotModal />` mounted once in `app/layout.tsx`, plus an `openPilotModal(email?)` helper that dispatches a `CustomEvent('open-pilot-modal')`. Any CTA can open the modal; the optional `email` arg prefills the form.
- `components/EmailForm.tsx` — React Hook Form, rendered inside the modal. Hidden honeypot `website` field. POSTs to `/api/send`.
- `components/SubscribeForm.tsx` — Blog subscribe widget, POSTs to `/api/subscribe`.
- `app/api/send/route.ts` — Pilot endpoint. Flow: Upstash rate limit (if configured) → honeypot short-circuit returns `{success, status:'filtered'}` → Zod validate → send two emails via Resend (admin to `founders@salency.ai`, confirmation to user) with `Promise.allSettled` so admin failure 500s but user-confirmation failure is tolerated.
- `app/api/subscribe/route.ts` — Blog subscribe. Adds contact to Resend Audience (`RESEND_AUDIENCE_ID`); 503 if unset.
- HTML in emails uses a manual `escapeHtml` — do not concatenate user input into templates without it.

### Styling

- Tailwind v4 — no `tailwind.config.ts`. Theme tokens live in `@theme` blocks inside `app/globals.css`.
- Primary accent is **bright copper** `--copper: #FE8531`, with a full opacity ladder `--copper-a0` through `--copper-a60` for tints, hairlines, glows, and surface accents (used for CTAs, section eyebrows, checkmarks, founder badges, surfaces). The older cyan `--accent: #06b6d4` is still defined but rarely referenced — prefer `--copper` and the `--copper-aXX` ladder for all new work.
- Background uses `--bg: #0E0C11` with `--bg-2: #121015` and `--bg-3: #1A171E` for alternating bands, plus `bg-noise` and `bg-mesh` layers.
- Fonts loaded in `app/layout.tsx` via `next/font/google`: **Instrument Sans** (body + display, bound via `--font-body` and `--font-display` in `app/globals.css`), **Outfit** (large H1/H2 marks, `--font-outfit`), **Instrument Serif** (italic em accents, `--font-instrument-serif`), **Geist Sans** (selective utility text, `--font-geist-sans`), **Geist Mono** (eyebrows, dates, table labels, `--font-geist-mono`). The `--font-body` / `--font-display` tokens use literal font-family strings rather than `var(--font-instrument-sans)` because next/font sets its CSS variable on `<body>` via className, not `:root`.
- `components/ScrollReveal.tsx` wraps below-fold sections; uses IntersectionObserver and respects `prefers-reduced-motion`.
- `packages/brand-reveal/` — portable animated wordmark ("Sales + Saliency" → "Salency") and loading-screen splash, packaged for drop-in use in other Salency frontend apps. Not consumed by this repo — kept as a design primitive only.
- Prefer explicit `transition-property` lists over `transition-all`.

### Client/server split

Server by default. Most `components/sections/*` are `'use client'` (HeroSection, HubSection, etc.) along with `MarketingHeader`, `PilotModal`, `EmailForm`, `SubscribeForm`, `ScrollReveal`, `HeroArtifact`. Page shells in `app/*/page.tsx` stay server components and inject metadata + JsonLd schema. Posts in `content/posts/` are TSX modules imported by `app/blog/[slug]/page.tsx` for static rendering.

### Analytics + SEO

`@vercel/analytics` is mounted once in `app/layout.tsx` via `<Analytics />`. No other tracking. Schema.org JSON-LD is injected via `components/JsonLd.tsx` — Organization + WebSite live in the root layout; per-page schemas (SoftwareApplication, BlogPosting, etc.) are added in their own `page.tsx`. `app/sitemap.ts` and `app/robots.ts` generate `/sitemap.xml` and `/robots.txt`.

### Design system reference

- `DESIGN.md` — full proposed system with tokens, competitive research, SAFE/RISK classification.
- `SALENCY_LANDING_PAGE.md`, `TASKS.md` — product/positioning context and outstanding work.
- `.gstack/design-reports/design-preview.html` — live token preview (open in browser).

## Branching strategy

**Trunk = `main` (production). `test` is the staging integration branch.**

Flow: branch off `main` → PR feature branch to `test` → merge to `test` for staging review → promote feature branch to `main` once verified. `main` auto-deploys to production; `test` is the staging environment.

- **Always branch off `main`**, never off `test`. `test` is the integration target, not a base.
- **First PR targets `test`** so staging gets the change. After review on the staging deploy, open a second PR from the same feature branch to `main` to ship.
- **Branch names:** `content/*` for copy, `design/*` for visual, `feat/*` for features, `fix/*` for bugs, `refactor/*` for restructure.
- **One PR = one logical change.** Small, reviewable, bisectable.

**If you see an older PR targeting `test-revamp`:** retarget to `test` with `gh pr edit <N> --base test`.
