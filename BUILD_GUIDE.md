# RGPV Exam Insights — Build Guide

End-to-end prompts. Copy-paste in order. Each step = one prompt to Claude Code.

## Stack

- **Frontend + API:** Next.js 16 (App Router, Turbopack, TypeScript) + React 19 + Tailwind v3 + GSAP + three.js (@react-three/fiber + drei) + Framer Motion + Recharts
- **Database:** Supabase Postgres — `tsvector` FTS + GIN index, RLS disabled
- **Deploy:** Vercel only (Next.js pages + `/api/*` route handlers in one deploy)
- **Ingestion (LOCAL ONLY — does not run on Vercel):** Python 3.10+ with PyMuPDF + rapidocr-onnxruntime + scikit-learn + supabase-py

> **Why ingest is local:** Vercel functions run Node, max ~10 s, no persistent disk. Scraper hits 1500+ URLs over minutes; extractor OCRs scanned PDFs at ~10 s/page; loader writes a 5 MB SQLite cache. None of that fits Vercel. Ingest runs on your laptop, pushes results to Supabase via service-role key, then Vercel reads Supabase directly.

---

## Step 0.0 — Generate `SPEC.md` from PDF

````
Read SPEC.pdf in this directory. Convert it to clean GitHub-flavored markdown and save as SPEC.md in repo root.

Rules:
- Preserve every section heading (1. Problem Statement … 14. After the Workshop) exactly.
- Preserve the SQL code block in section 5.4 verbatim — inside ```sql fences.
- Preserve the folder tree in section 8 verbatim — inside ``` fences.
- Preserve the tables in section 7 and section 12 as markdown tables.
- Use `- ` for bullets, `1.` for numbered lists.
- No paraphrasing. No summarizing.
- After writing, print the first 40 lines so I can verify.
````

Sanity check: 14 numbered sections, one ```sql block, one ``` folder tree, two markdown tables.

---

## Step 0.1 — Create `CLAUDE.md`

```
Create CLAUDE.md in repo root with:

PROJECT: RGPV Exam Insights — search individual RGPV B.Tech exam questions across years with frequency analysis. Read SPEC.md for full spec.

STACK (fixed):
- Frontend + API: Next.js 16 App Router (React 19, TypeScript, Tailwind v3, Turbopack)
- Animations: GSAP, three.js (@react-three/fiber + drei), Framer Motion, Recharts
- Database: Supabase Postgres (tsvector FTS, GIN index, RLS disabled)
- Ingest (local only): Python — PyMuPDF, rapidocr-onnxruntime, scikit-learn, supabase-py
- Deploy: Vercel only (root = web/)

UI THEME (non-negotiable): dark elegant glassmorphism. bg #0A0A0F. surface #12121A backdrop-blur.
violet→cyan gradient (#7C5CFF → #00D4FF). rose accent #FF6B9D. Instrument Serif display, Inter body, JetBrains Mono code. Hero = three.js distorted icosahedron + transmission halo + radial vignette. rounded-2xl everywhere. Cards: border rgba(255,255,255,0.08), inner glow on hover. Motion: GSAP stagger entries, hover scale 1.02 + glow.

BEFORE MARKING TASK COMPLETE:
(1) pytest if ingest changed
(2) npm run build if web/ changed
(3) curl affected /api/* route, verify JSON shape
(4) open affected page in browser, execute SPEC user flow
(5) screenshot result and report

DISCIPLINE:
- Plan BEFORE code. Wait for approval.
- Small diffs. One concern per commit.
- Never invent question samples — only parse real PDFs.
- Log parser failures, never crash.
- SUPABASE_SERVICE_ROLE_KEY is server-only. Never put in client component or NEXT_PUBLIC_* env. lib/supabase.ts starts with `import "server-only"`.
- Reference SPEC.md acceptance criteria in every PR description.
```

---

## Step 0.2 — Scaffold

```
Read SPEC.md and CLAUDE.md. Create:

- src/ (Python ingest): __init__.py + empty scraper.py, extractor.py, parser.py, db.py, load.py, upload_supabase.py
- tests/: empty test_parser.py, test_extractor.py, test_db.py
- supabase/schema.sql placeholder
- web/ folder (Next.js)
- data/raw/.gitkeep + data/processed/.gitkeep
- requirements.txt: pymupdf, requests, beautifulsoup4, scikit-learn, pytest, rapidocr-onnxruntime, supabase
- web/package.json: next@latest (16+), react@latest (19+), react-dom@latest, typescript, tailwindcss@^3.4 (NOT v4 — different PostCSS), postcss, autoprefixer, gsap, three, @react-three/fiber@latest, @react-three/drei@latest, framer-motion, lucide-react, recharts, @supabase/supabase-js, @types/*
- .gitignore: python + node + .env* + data/raw/*+data/processed/* (keep .gitkeep) + *.sqlite +.next/ + node_modules/
- README.md skeleton
- .env.example: NEXT_PUBLIC_SUPABASE_URL=https://<project>.supabase.co + SUPABASE_SERVICE_ROLE_KEY=

Show tree first. Then create. No logic yet.
```

---

## Step 0.3 — `PLAN.md`

```
Read SPEC.md and CLAUDE.md. Write PLAN.md (under 400 lines):

1. Overview — 3 sentences.
2. Milestones table — Phase | Goal | Files touched | Acceptance check | Estimated time.
   Cover: scrape, extract+parse, supabase schema + upload, Next.js API routes, frontend (hero + search + insights), polish, Vercel deploy, demo.
3. File-by-file plan: src/* (Python ingest), supabase/schema.sql, web/app/api/*, web/lib/supabase.ts, web/components/* — purpose, public exports, deps, tests.
4. ASCII data flow: PDFs → extractor → parser → SQLite cache → upload_supabase → Supabase → Next.js /api/* → frontend.
5. Risk log: copy SPEC §12 + add (3D perf on low-end, GSAP hydration mismatch, Supabase free-tier row caps, leaked service-role key, Vercel function timeout if you accidentally fetch huge result sets server-side).
6. Definition of Done = SPEC §10 + UI gates (hero blob renders, search debounced, frequency chart <2 s, lighthouse ≥85 mobile).
7. End-of-phase commit messages.

No code. Reference SPEC by section number. Print milestone table.
Wait for approval before Phase 1.
```

---

## Phase 1 — Scrape PDFs (local)

### Step 1.1 — Inspect rgpvonline.com

```
Fetch listing pages for B.Tech sem 3–8 across all four branches:
- https://www.rgpvonline.com/btech-cse-question-papers.html
- https://www.rgpvonline.com/btech-it-question-papers.html
- https://www.rgpvonline.com/btech-ec-question-papers.html
- https://www.rgpvonline.com/btech-me-question-papers.html

Report:
- URL pattern (PDFs live at /be/<slug>-<session>-<year>.pdf, NOT site root)
- HTML structure of listing page
- That slugs carry multi-branch prefixes (e.g. ad-ai-al-cd-cs-..., au-me-..., ag-csit-it-...)
- robots.txt disallows all non-Google bots — proceed politely, document deviation

Output one markdown table covering every subject: branch | semester | subject_code | subject_name | url_slug. Expect ~98 entries.
```

### Step 1.2 — Write scraper

```
Based on inspection, write src/scraper.py:
- SUBJECTS dict: code → (slug, name, semester, branch). branch ∈ {CSE, IT, EC, ME}.
- download_subject(code, years=range(2018, 2026))
- Saves data/raw/<code>/<year>-<SESSION>.pdf (SESSION upper-case)
- URL: f"{BASE}/be/{slug}-{session.lower()}-{year}.pdf"
- Idempotent: skip if file exists and size > 0
- Polite: 0.4 s sleep, custom User-Agent
- Log OK / MISS / FAIL with status code
- CLI: python -m src.scraper --subject CS-502 | python -m src.scraper --all [--year-from --year-to]

Show plan first. Verify CS-502 (expect ~3-4 hits of 18 attempts). Then --all in background. Expect ~250-300 PDFs total.
```

---

## Phase 2 — Extract + Parse (local)

### Step 2.1 — Extractor with OCR fallback

```
Write src/extractor.py:
- extract_text(pdf_path) → str using PyMuPDF
- Layout-aware: page.get_text("blocks") sorted by x-bucket then y (handles two-column papers)
- Many RGPV PDFs from 2022+ are scanned images. Add OCR fallback:
  when a page yields <50 chars, render at 200 DPI and run rapidocr-onnxruntime (pure-python, no tesseract binary).
- Skip page only if BOTH native text + OCR yield <50 chars

Write tests/test_extractor.py: text PDF returns non-empty, missing file raises FileNotFoundError, synth-scanned PDF triggers OCR. Run pytest.
```

### Step 2.2 — Parser

```
First, dump 3 actual extracted texts from data/raw/CS-502/. Identify the RGPV pattern:
- Main header: "N." or "N. a)" alone on a line (N = 1..10)
- Sub header: "a)" / "b)" / "c)" alone on a line
- Marks: bare integer line (1..20), OR trailing-int on text line
- Page numbers: "[1]" with literal brackets
- Bilingual: English then Hindi (Hindi arrives as ASCII garble — custom Devanagari font)

Then write src/parser.py:
- Question dataclass: unit_number, question_number, marks, question_text
- Separate regexes for RE_MAIN (allows inline body), RE_SUB, RE_BARE_MARKS, RE_TRAILING_MARKS, RE_PAGE_NUM
- Track last_main_no so sub-headers after a finalize restore parent number
- Page-number regex MUST require literal brackets — bare `7` is marks
- Hindi-junk filter: token-level garble check (not unicode) — drop line if ≥50% tokens contain $ ¶ § | ° « ñ © ® < > or are mostly `?`. English-letter ratio alone misclassifies RGPV's font-encoded Devanagari as English.
- _clean_text strips garble tokens
- When all parsed questions have marks=None, infer per-main from "Maximum Marks : N" + "Attempt any K" → per_main = N/K
- unit_number = min(((qno-1)//2)+1, 5)
- Log skipped lines to logs/parser.log; never raise
- Target: ≥80% extraction rate (SPEC §5.3)

Write tests/test_parser.py: golden samples (bare-marks paper, inferred-marks paper, garbage input), helper tests (clean, hindi, unit map, marks inference). Run pytest. Report per-paper parse rate on real CS-502 PDFs.
```

### Step 2.3 — Iterate

```
Run parser on every CS-502 PDF. Print per-paper Q count + with_marks + units.
If any paper <80%, show raw vs parsed side-by-side, propose regex fix, wait for approval.
```

---

## Phase 3 — Storage (Supabase) (45 min)

### Step 3.1 — Schema

```
Create supabase/schema.sql (idempotent — wrap every CREATE in IF NOT EXISTS):

papers (id BIGSERIAL PK, subject_code, subject_name, year INT, session TEXT, branch, semester INT, source_pdf_path, source_pdf_url, UNIQUE(subject_code, year, session))

questions (id BIGSERIAL PK, paper_id BIGINT REFERENCES papers ON DELETE CASCADE, unit_number, question_number, marks, question_text TEXT NOT NULL,
           tsv tsvector GENERATED ALWAYS AS (to_tsvector('english', question_text)) STORED)

Indexes: idx_questions_paper, idx_questions_unit, idx_questions_marks, idx_questions_tsv (GIN on tsv)

subject_frequencies (subject_code, rank INT, term, score DOUBLE PRECISION, PRIMARY KEY (subject_code, rank))

ALTER TABLE … DISABLE ROW LEVEL SECURITY for all three tables.

VIEW subject_stats: per-subject papers + questions counts.
VIEW branch_stats: per-branch subjects + papers + questions counts.

Tell user: open Supabase dashboard → SQL editor → paste schema → Run.
```

### Step 3.2 — Local SQLite cache + loader

```
Cache layer so reruns don't re-extract every PDF.

Write src/db.py:
- connect(path, check_same_thread=False), row_factory=Row, PRAGMA foreign_keys ON
- init_db with SQLite mirror of Supabase schema (FTS5 + triggers — fine for SQLite, just a buffer)
- insert_paper idempotent on UNIQUE(subject_code, year, session) — wipes old questions on rerun
- insert_questions bulk executemany

Write src/load.py CLI:
- Walks data/raw/, runs extractor → parser, derives meta from path + SUBJECTS
- Inserts via db.py
- Reports loaded / total_questions / skipped (with reason)

Write tests/test_db.py: schema columns match SPEC §5.4, insert+query roundtrip, idempotent rerun, FTS5 match, FTS rebuild.
Run pytest. Run loader. Expect ~150-200 papers + ~1500-2500 questions.
```

### Step 3.3 — Upload to Supabase

```
Write src/upload_supabase.py:
- env: NEXT_PUBLIC_SUPABASE_URL + SUPABASE_SERVICE_ROLE_KEY (service-role bypasses RLS)
- Wipe target tables first (idempotent reload):
    DELETE subject_frequencies, DELETE questions, DELETE papers
- Read SQLite via sqlite3
- Push papers in batches of 500; capture returned ids for paper_id remap
- Push questions in batches of 500; apply _clean_text BEFORE upload to strip garble (cheaper than re-parsing PDFs)
- Compute per-subject TF-IDF with sklearn (stop_words='english', ngram_range=(1,2), max_features=500)
- Insert top-20 terms per subject into subject_frequencies (rank, term, score)

Run: NEXT_PUBLIC_SUPABASE_URL=... SUPABASE_SERVICE_ROLE_KEY=... python -m src.upload_supabase
Print final Supabase counts.
```

---

## Phase 4 — Frontend + API (Next.js) (90 min)

### Step 4.1 — Next.js API routes (Supabase queries)

```
Install @supabase/supabase-js in web/.

Create web/lib/supabase.ts:
- FIRST LINE: import "server-only"; (makes accidental client import a build-time error)
- Singleton createClient(NEXT_PUBLIC_SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, { auth: { persistSession: false } })
- Helpers exported:
    cleanText(text): drops tokens containing $ ¶ § | ° « ñ © ® < > or majority non-ASCII letters
    sanitizeFts(q): strip metachars, join 2+-char tokens with " | " for OR semantics

Create web/app/api/*/route.ts for:
- GET /api/health           → Supabase ping, return {status:"ok"}
- GET /api/stats            → {papers, questions, subjects, year_from, year_to}
- GET /api/branches         → from branch_stats view
- GET /api/subjects?branch= → from subject_stats view
- GET /api/search           → q, subject, unit, year_from, year_to, marks, branch, limit
                              Use .textSearch("tsv", sanitizeFts(q), { type:"websearch", config:"english" })
- GET /api/frequency?subject=&top_n=  → from subject_frequencies
- GET /api/question/[id]    → with paper join; 404 on miss
- GET /api/random?subject=&branch=    → count then offset random pick

Each route: `export const dynamic = "force-dynamic"`.
For /search and /frequency, soft-fail (return [] on Supabase error) — malformed FTS shouldn't 500.
Run cleanText on every returned question_text.

Run npm run build. curl every endpoint against your Supabase project.
```

### Step 4.2 — Frontend scaffold + theme

```
In web/:
- Pin tailwindcss@^3.4 (v4 needs @tailwindcss/postcss + different config — don't migrate now)
- postcss.config.js: { plugins: { tailwindcss: {}, autoprefixer: {} } }
- tailwind.config.ts colors:
    bg #0A0A0F, surface #12121A, border rgba(255,255,255,0.08), text #F5F5FA, muted #9CA3AF,
    violet #7C5CFF, cyan #00D4FF, rose #FF6B9D
    backgroundImage.accent-gradient = linear-gradient(135deg, #7C5CFF, #00D4FF)
- app/globals.css: @tailwind directives + body radial-gradient (#1a1530 → #0A0A0F) + subtle SVG noise overlay (opacity 0.03)
- app/layout.tsx: next/font/google for Instrument Serif (display), Inter (body), JetBrains Mono (mono).
  Expose --font-display, --font-body, --font-mono. Mount <NavBar /> here from day one.
- components/ui/: GlassCard, GradientButton (Link or button via prop), Input, Select, Tag, NavBar (sticky, active-route highlight via usePathname)
- lib/api.ts: same-origin fetcher → relative "/api" (no NEXT_PUBLIC_API_URL needed; no CORS)

Build. Screenshot homepage.
```

### Step 4.3 — Hero + GSAP + home

```
components/three/HeroScene.tsx (client):
- <Canvas> camera fov 38, position [0,0,5], dpr [1,2]
- Group offset position={[1.7, -0.1, 0]} so blob sits right of headline
- CoreBlob: <Float> wrapping <icosahedronGeometry args={[1, 64]} /> + MeshDistortMaterial
    color #7C5CFF, emissive #5B3FE0 (intensity 0.45), distort 0.55, speed 2.2, metalness 0.85
- GlassShell: scale 1.55, MeshTransmissionMaterial transmission 1, chromaticAberration 0.06,
    attenuationColor #FF6B9D (drei needs ≥3 geometry subdivisions for this to render right)
- Lights: directional violet, directional cyan, rose point, small white point for spec
- Radial vignette div over Canvas: background radial-gradient(ellipse 80% 70% at 18% 50%, rgba(10,10,15,0.92) 0%, transparent 70%)

components/AnimatedCount.tsx: GSAP tween 0 → value, power2.out, 1.6 s.
components/StatusPill.tsx: animate-ping cyan dot + "<subjects> subjects · <papers> papers · <questions> questions" from /api/stats.

app/page.tsx (client):
- Hero with HeroScene background, foreground grid md:[1.2fr_1fr]
- StatusPill
- H1 clamp(3rem,7vw,6.5rem), Instrument Serif, three lines:
    "Every question" / "RGPV ever asked." (muted) / "Searchable." (bg-accent-gradient bg-clip-text)
- Two CTAs: GradientButton href="/search" + ghost "/insights"
- 5 inline "Try '<topic>'" chips → /search?q=<topic>
- 4-tile stats band with AnimatedCount fed by /api/stats
- 4-tile branches band from /api/branches → /insights?branch=<branch>
- ~25 popular-topic chips
- Top-9 subjects grid from /api/subjects → /insights/<code> ; "View all →" → /insights
- CTA footer

app/insights/page.tsx: subject grid filtered by branch query param. Wrap export in <Suspense> for useSearchParams.

Use stale-flag pattern (let active = true ; cleanup sets false) instead of AbortController so devtools network panel stays clean.

Install three @react-three/fiber @react-three/drei gsap framer-motion lucide-react recharts. Build. Browser-verify hero spins, stats count up.
```

### Step 4.4 — Search page

```
Create app/search/page.tsx (client, page export wrapped in <Suspense>):
- Init q, subject, unit, marks, year_from, year_to from useSearchParams so /search?q=BCNF&subject=CS-502 lands with input pre-filled
- Sticky top bar: subject dropdown (from /api/subjects), search input (debounced 300 ms via lib/useDebounced.ts), unit/marks selects, year_from/year_to number inputs, Reset
- Each result card: Tag(subject_code), rose Tag(marks), mono "<SESSION> <year>", "Unit N", body, links to /insights/<subject_code> + external PDF
- Empty state, loading skeletons, no-match state, error state (all on-theme)
- Framer Motion AnimatePresence stagger-fade on results change
- Stale-flag pattern, not AbortController

Build. Verify clicking a home chip lands pre-populated. Single-word query matches OR-style.
```

### Step 4.5 — Frequency view

```
Create app/insights/[subject]/page.tsx:
- Next 15+ async params: type Props = { params: Promise<{ subject: string }> } ; const { subject } = use(params)
- Header: subject name (Instrument Serif), AnimatedCount for question total
- Recharts BarChart layout="vertical", bars filled with linearGradient violet→cyan
- Click handler on per-<Cell> (NOT <Bar> — type-safety differs). Click → router.push(`/search?q=<term>&subject=<code>`)
- Tooltip formatter: narrow ValueType | undefined via typeof check
- Below: inline SVG word cloud — sized <button> text, font-size scaled by score, color by quartile (rose/cyan/violet/muted). Zero extra deps — react-wordcloud is unmaintained for React 18/19.

Data from /api/frequency. Build. Verify nav from chart + word cloud.
```

### Step 4.6 — Polish

```
Across pages:
- Consistent spacing (py-24 sections, gap-6 grids)
- Page transitions via Framer Motion AnimatePresence
- app/not-found.tsx + app/error.tsx with on-theme blob
- Lighthouse run, target ≥85 mobile perf
- Screenshot every page, tick SPEC §10 checklist
```

---

## Phase 5 — Deploy (Vercel)

```
Vercel project root = web/. Two env vars only:
  NEXT_PUBLIC_SUPABASE_URL
  SUPABASE_SERVICE_ROLE_KEY (do NOT prefix with NEXT_PUBLIC)

CLI:
  cd web
  npm i -g vercel
  vercel link
  vercel env add NEXT_PUBLIC_SUPABASE_URL production
  vercel env add SUPABASE_SERVICE_ROLE_KEY production
  vercel --prod

Or dashboard: import GitHub repo → Root Directory = web/ → paste env vars → Deploy.

Verify: https://<app>.vercel.app/api/health → {"status":"ok"}.

NOTE: ingest (scraper, extractor, parser, upload_supabase) runs LOCALLY only. Vercel deploys web/ only. To refresh data, re-run the local ingest pipeline and let upload_supabase push to Supabase.

For automated refresh later: GitHub Actions cron running the same Python pipeline on a schedule, with the two Supabase env vars set in repo secrets.
```

---

## Final README

```
Write README.md:
- Hero screenshot
- 1-paragraph what + why
- Live URL placeholder
- Stack table (Next.js 16 + Supabase + Vercel)
- Local setup: ingest pipeline + npm run dev
- Ingestion runbook (scraper → load → upload_supabase)
- Env vars table
- SPEC §10 acceptance checklist
- Notes (robots.txt, OCR latency, RLS off)
- Credits

Verify GitHub preview renders.
```

---

## Demo script (last 30 min, mandatory)

```
Write DEMO.md, 3-minute walkthrough:
1. Open hero — blob renders, stats count up from /api/stats
2. Click "Search questions"
3. Search "BCNF" in DBMS — results across years from Supabase FTS
4. Apply unit + marks filter
5. Switch to Insights — frequency chart loads
6. Click top term → back to filtered search

Practice twice. Time it.
```

---

## Lessons learned (apply BEFORE re-running prompts)

### Data

1. **rgpvonline PDF URLs live under `/be/<slug>.pdf`**, not site root. Naive `f"{base}/{code}.pdf"` returns 404 every time.
2. **URL slug ≠ subject code** — multi-branch prefixes. Store full slug per subject.
3. **Coverage is sparse** — ~3 of ~18 (year × session) per subject. Scrape every subject across CSE+IT+EC+ME sem 3–8 to clear SPEC §10.1 (≥30 PDFs).
4. **Most 2022+ papers are scanned images.** Use **rapidocr-onnxruntime** as fallback when PyMuPDF returns <50 chars/page.
5. **robots.txt disallows all non-Google bots.** Workshop scale, low volume. Custom UA + 0.4 s sleep, document deviation.

### Parser

1. Page-number regex `^\[?\d+\]?$` will eat bare `7` marks lines. Require literal brackets: `^\[\s*\d+\s*\]$`.
2. After `finalize()` resets main_no, a following `b)` loses parent number. Track `last_main_no` separately.
3. Two paper flavours: bare-marks integer per sub-part vs no per-question marks at all. For the latter, infer per-main = `Maximum Marks : N` / `Attempt any K`.
4. Hindi blocks come through as ASCII garble (custom Devanagari font). English-letter ratio alone misclassifies them as English. Token-level garble check (`$ ¶ § | ° « ñ © ® < >` or majority `?`) is the only reliable filter.

### Supabase

1. **Service-role key bypasses RLS** but cannot run raw DDL via REST. Apply `schema.sql` via Supabase SQL editor (one paste, 5 s).
2. **`tsvector` as generated column + GIN index** is the Postgres equivalent of SQLite FTS5. Use `.textSearch("tsv", tsq, { type: "websearch" })` to honour `|` (OR).
3. **`sanitizeFts` joins tokens with ` | `** so casual multi-word queries match any term, not all.
4. **Explicitly `DISABLE ROW LEVEL SECURITY`** at end of schema.sql.
5. **Precompute TF-IDF at upload time**, store top-20 per subject in `subject_frequencies`. Avoids shipping sklearn to Vercel.
6. **Aggregate views** (`subject_stats`, `branch_stats`) keep API routes simple.

### Frontend

1. With Next.js API routes everything is same-origin — no CORS needed at all.
2. Search button on home must be a Next `<Link>`, not a bare `<button>`.
3. Mount NavBar in `app/layout.tsx` from day one.
4. `react-wordcloud` is unmaintained for React 18/19. Build a 30-line SVG-text cloud instead.
5. Recharts strict types: `Tooltip.formatter` value is `ValueType | undefined`; `<Bar onClick>` shape differs from `<Cell onClick>`. Use Cell-level handlers.
6. Next 15+ async route params: `type Props = { params: Promise<{ subject: string }> }` ; unwrap with `const { subject } = use(params)`.
7. `useSearchParams()` requires a `<Suspense>` boundary. Wrap page exports in `<Suspense fallback={…}>`.
8. After upgrading Next majors, `rm -rf .next` before next build — stale artefacts cause cryptic "Cannot find module for page: /_not-found".
9. `MeshTransmissionMaterial` needs ≥3 geometry subdivisions and sits AFTER a separate inner mesh for the glass halo to render.
10. **Pin `tailwindcss@^3.4`.** v4 needs `@tailwindcss/postcss` and CSS-first config — don't migrate now.
11. **AbortController + React StrictMode = red rows in devtools** (first-mount fetches get aborted on cleanup). Use `let active = true` stale-flag pattern instead.

### Hero / 3D

1. Full-viewport `<Canvas>` overpowers text. Offset the group (`position={[1.7,-0.1,0]}`) AND overlay a radial vignette (rgba(10,10,15,0.92) → transparent at 70%) anchored to the text side.
2. `MeshDistortMaterial` alone reads flat. Layer a `MeshTransmissionMaterial` shell (scale ~1.55) for the chromatic glass halo.
3. Camera fov 38 + z 5 frames the blob at "logo size", not "wallpaper size".

### API (Next.js routes)

1. `export const dynamic = "force-dynamic"` on every route — otherwise Next caches and stats stay stale.
2. `cleanText` on the server keeps every consumer consistent.
3. Soft-fail (`return NextResponse.json([], { status: 200 })`) on Supabase errors for list endpoints. Malformed FTS queries → empty results, not 500.
4. `import "server-only"` at top of `lib/supabase.ts` so accidental client import is a build-time error. Service-role key NEVER in `NEXT_PUBLIC_*`.

### Vercel

1. Project root = `web/` (NOT repo root). Vercel dashboard → Settings → Root Directory.
2. Two env vars — paste in Production + Preview + Development.
3. Python ingest does NOT run on Vercel. Run locally. To automate later, use GitHub Actions cron with Supabase secrets in repo settings.

---

## Failure protocol

1. Stop. Don't push forward.
2. Paste error + last 20 lines of relevant code back.
3. Ask: "Diagnose root cause. Propose fix. Don't change code yet."
4. Approve fix → apply → re-run that step.

Never `--no-verify`. Never silence errors.
