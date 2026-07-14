# SAssi — Full Audit & Apple/TestFlight Readiness Report

*Audited 2026-07-14. Facts about Apple programs, TestFlight, and iOS were researched and
adversarially verified against current (2025–2026) sources; load-bearing claims cite Apple's
own documentation.*

> **Update, same day:** the real SAssi app was located in `mgfv2r2mpv-hash/aba-dashboard`
> ("SAssi Cal", already Capacitor-wrapped with a committed Xcode project). See
> **`sassi-real-app-audit.md`** for the audit of the actual codebase — it supersedes §1–§3
> and the Phase 0–2 roadmap below. §4 (persistence options), §5 (TestFlight requirements),
> and the verified Apple facts remain the reference, with one correction: SAssi Cal uses
> AES-GCM beyond HTTPS, so its export-compliance answer is "uses encryption → standard
> exemption" (`ITSAppUsesNonExemptEncryption = true`), not `NO`.

---

## 1. What SAssi is in this repo — the critical framing

**This repo does not contain the SAssi app.** It contains a *design handoff bundle*
(`design_handoff_site_overhaul/`) whose `ui_kits/scheduler/` directory is a high-fidelity
**React prototype** of SAssi (~430 lines across 4 JSX files + mock data), explicitly labeled
a design reference:

> "the SAssi scheduling + compliance app (the product the core design system was
> reverse-engineered from), recreated in React from the design-system components" …
> "in the real app, mirror the structure in its own stack — the value here is the visual +
> interaction spec, not the bundle." — `README.md`

The bundle also confirms a **real, shipped SAssi exists elsewhere**:

- `tokens/fonts.css`: "The shipped iOS/web app currently renders in the platform system sans."
- `tokens/colors.css`: "Derived directly from the shipped SAssi scheduling app (inline-style
  palette, Tailwind-family hues)."
- `tokens/fonts.css` implies the real app has **API-key entry and a lock-screen PIN**.
- `tokens/typography.css`: the real app is "information-dense and mobile/iPad-first."

No repo path or URL for the real app appears anywhere in this repo. The prototype here
**cannot even render as shipped**: `ui_kits/scheduler/index.html` loads `../../_ds_bundle.js`
(the design-system component bundle), and that file **does not exist anywhere in the repo** —
`CalendarView.jsx.txt` destructures `window.NoOutcomeABADesignSystem_08ed95` at module top
level, so the page throws immediately.

**Consequence:** "publish SAssi to TestFlight" means either (a) locating the real SAssi
codebase (likely candidate: the `aba-dashboard` GitHub repo, not attached to this session), or
(b) building SAssi out from this prototype into a real app first. Section 3 scopes (b).

---

## 2. Feature rundown — what the prototype actually does

Status legend: **FUNCTIONAL** (works in the prototype) · **STUBBED** (control exists, no
behavior) · **FAKED** (hardcoded demo data).

### AppShell (frame + nav)
| Feature | Status |
|---|---|
| View switching (📅 Cal / 🔧 Comp / ✨ Wish) | FUNCTIONAL |
| In-memory appointment state + mutation | FUNCTIONAL |
| "+" Add appointment | STUBBED — placeholder view ("New appointment form would open here.") |
| Compliance issue badge | FAKED — `const issueCount = 4; // demo` |
| "AI key set" green dot | FAKED — static element |

### CalendarView (month grid + docked context pane)
| Feature | Status |
|---|---|
| Appointment chips: type accent bar, ✓ completed (green), canceled strikethrough (family = orange / staff = red) | FUNCTIONAL |
| Select chip → detail pane; Complete / Cancel / Reopen | FUNCTIONAL (in-memory only; Cancel always records `staff`) |
| Month grid | FAKED — hardcoded "March 2026", days 1–31 with **no weekday offset**, "today" hardcoded to the 12th |
| Prev/next month | STUBBED — no `onClick` |
| BCBA / BT / Client lens toggle | STUBBED — state is set but never read |
| Hours meter ("BT week") | FAKED — every appointment counted as 2h across the whole month |
| Agenda ("this week") | FAKED — hardcoded filter for days 9–13 |
| Edit button | STUBBED |

### ComplianceView (traffic-light dashboard)
Entirely **FAKED**: renders hardcoded per-client supervision % and per-tech billable hours
from `data.js`; ignores the live `appts` prop; zero interactions. It *does* demonstrate the
full status model, including the **"over" audit-risk state** (e.g., 38% supervised vs 20%
target → "· audit risk").

### WishView (AI schedule rework)
Textarea is FUNCTIONAL (controlled input); **"Generate options" is FAKED** — it reveals two
hardcoded proposal cards regardless of what you type. No LLM call, no network. Accept /
Customize / Reject are STUBBED.

### Persistence
**None.** All state is React `useState` — a refresh loses everything. No `localStorage`, no
backend, no sync. (The bundle's localStorage keys — `noaba.*` — belong to the games/tools
kits, not the scheduler.)

### Dependencies to render at all
React 18.3.1 + ReactDOM UMD dev builds and `@babel/standalone` from unpkg (JSX transpiled in
the browser at runtime), the **missing** `_ds_bundle.js` (must export at least: `Button`,
`IconButton`, `SegmentedControl`, `StatusPill`, `Card`, `MetaChip`, `ProgressMeter`,
`Avatar`), `styles.css` → `tokens/*.css`, and two Google Fonts stylesheets (Atkinson
Hyperlegible + Mono).

### Brand / asset readiness for iOS
- `assets/app-icon.svg` exists — a purpose-built SAssi icon (rounded tile, N-as-motion-lines
  moving appointment blocks on an undated sage calendar). **SVG only** — an iOS build needs a
  1024×1024 PNG in the asset catalog (Xcode "Single Size" generates the rest).
- `logo-mark.svg` (ensō + mountain-N) and `wordmark.svg` ("SAssi — Schedule Assistant") exist.
- Font: Atkinson Hyperlegible (SIL OFL — fine to embed); the shipped app uses system sans.
- **Regulated terminology** (SKILL.md non-negotiable): user-facing copy must say
  "Supervising Behavior Analyst" (never "BCBA") and "Credentialed BT" (never "RBT") —
  **the prototype itself violates this** (the lens toggle is labeled "BCBA/BT"; the pane says
  "BT week"). Must be fixed before anything ships.

---

## 3. Gap list — prototype → launchable app

Roughly ordered by dependency:

1. **Design-system components** — recreate the 8+ missing `_ds_bundle.js` primitives (specs
   for all of them exist in the tokens/README).
2. **Real calendar math** — actual `Date` handling, weekday offsets, month navigation,
   "today" detection.
3. **Add / Edit appointment** — the core CRUD form (type, client, tech, time, recurrence?).
4. **Persistence** — the single biggest architectural decision (see §5 privacy):
   *local-first storage on device* (IndexedDB/SQLite) is strongly indicated; plaintext client
   names server-side are outside this project's documented compliance posture.
5. **Computed compliance** — derive supervision % and billable hours from real appointment
   data instead of static tables (the current meters are display-only).
6. **Wish It → real LLM call** — the site's `_worker.js` already has a production LLM proxy
   (`POST /api/llm-call` → Anthropic Messages API, default `claude-haiku-4-5`, Bearer-token
   auth with per-tool scopes, instant revocation via KV). SAssi would need its own tool scope,
   plus a schedule→prompt serializer, proposal parsing, and an apply/undo engine. Note: the
   proxy currently has **no model allowlist, no max-token cap, and no rate limiting** — fix
   before exposing it to a new client surface.
7. **Auth / PIN** — the real app reportedly has API-key entry + lock-screen PIN; decide
   whether the rebuilt app reuses the site's HMAC-token login or stays fully local.
8. **Terminology + copy pass** (§2 above).
9. **Icon rasterization** and (for iOS) splash/asset catalog.

Realistic effort: the wrap-and-ship steps are hours; **items 1–6 are the actual product build
— think small-number-of-weeks of focused work**, not days.

---

## 4. Why your app dies "after a few days" — and every fix, compared

The "few days" symptom matches Apple's **free-account 7-day provisioning limit** — officially
documented: "Provisioning profiles will expire 7 days from issuance, which may require you to
rebuild and re-install your app" (developer.apple.com/support/compare-memberships). Free
accounts also get max 10 App IDs (each 7-day) and, community-documented, 3 sideloaded apps at
once.

**Important: TestFlight is *not* permanent either.** Every TestFlight build becomes
unavailable **90 days after upload** — installed copies stop launching; you must upload a new
build for a fresh 90 days. Verified against Apple's TestFlight documentation.

| Path | Cost | App keeps working for | Maintenance | Review? |
|---|---|---|---|---|
| Free Apple ID + Xcode sideload | $0 (needs Mac) | **7 days** | rebuild + reinstall weekly | none |
| AltStore / SideStore auto-resign | $0 | 7-day cycles, auto-refreshed | initial setup fiddly; refresh mostly automatic | none |
| Developer Program: development / ad-hoc signing | $99/yr | up to **12 months** per profile (capped by cert/membership expiry) | re-sign & reinstall yearly; 100 devices per product family | none |
| **TestFlight internal** | $99/yr | **90 days per build** | upload new build every <90 days; auto-update available | **no review** — live minutes after processing |
| TestFlight external | $99/yr | 90 days per build | same, plus test info | Beta App Review on first build of each version (~hours–1 day) |
| **App Store release (incl. Unlisted)** | $99/yr | **indefinitely** — Apple confirms installed apps "still function" even if your membership later expires | none after approval | full App Review once |
| PWA — Add to Home Screen | **$0** | **indefinitely** | none | none |

Key verified facts behind that table:

- **PWA persistence:** Home-screen web apps are **exempt from Safari's 7-day ITP storage
  eviction** — they run outside Safari with their own storage container and days-of-use
  counter; Apple/WebKit state that data deletion in an installed web app would be "a serious
  bug." Home-screen apps also get the full ~60%-of-disk quota, Web Push + app badges work
  (since iOS 16.4), and offline works via service workers. **iOS 26 (Sept 2025) made every
  site added to the Home Screen open as a web app by default.** No Background Sync and no
  App Store presence are the main limits.
- **Unlisted App Store distribution:** normal App Review + a request form; the app is
  link-only (no search/charts) — the natural terminal state for a private clinician tool.
  This is the **only zero-maintenance native path**: install once, works forever, even
  surviving a lapsed membership.
- TrollStore-style permanent signing is dead on modern iOS (patched since 17.0.1); US
  alternative app marketplaces still don't exist (EU/Japan/Brazil only).

---

## 5. TestFlight requirements checklist (as of mid-2026)

**Account & tooling**
- [ ] Apple Developer Program — **$99/year**, individual enrollment is fine for TestFlight
  (2FA Apple Account; confirmation typically ~24–48 h). ⚠️ For an eventual **App Store**
  release, Guideline 5.1.1(ix) requires apps "primarily for… healthcare-adjacent use handling
  sensitive user information" to be submitted by a **legal entity**, not an individual —
  worth structuring as the LLC/company account from the start if that's where this is headed.
- [ ] Builds must use **Xcode 26 / iOS 26 SDK** — mandatory for all App Store Connect uploads
  (including TestFlight) since **April 28, 2026**. Expect the floor to rise again ~April 2027.
- [ ] A Mac is needed to *compile* — but not necessarily *yours*:
  - **Codemagic**: 500 free macOS minutes/month, automatic code signing, publishes straight
    to TestFlight via an App Store Connect API key — lowest-friction no-Mac option.
  - **GitHub Actions**: macOS runners free/unlimited for public repos (private: 10×
    multiplier ≈ 200 real macOS min/month on the free plan); `macos-26` image ships Xcode 26.
  - **Xcode Cloud**: 25 free compute h/month with the program, but the first workflow must be
    created from Xcode on a Mac.
  - Certificates/profiles can be generated entirely from CI (fastlane match).

**App record & build**
- [ ] App Store Connect record: app name, bundle ID (immutable), SKU.
- [ ] 1024×1024 PNG icon in the asset catalog (rasterize `assets/app-icon.svg`).
- [ ] Export compliance: HTTPS-only apps are exempt — set
  `ITSAppUsesNonExemptEncryption = NO` in Info.plist so builds go live without a prompt.
- [ ] No screenshots needed for TestFlight (those come at App Store submission).

**Testing tiers**
- [ ] **Internal** (recommended for you): up to 100 App Store Connect users, 30 devices each,
  **no Beta App Review** — builds are installable minutes after upload. This is the fastest
  legitimate way to get SAssi on your phone.
- [ ] **External**: up to 10,000 testers via public link; first build of each version passes
  Beta App Review; needs beta description, feedback email, and demo credentials if there's a
  login.

**Review-facing product requirements** (Beta Review is lighter than App Store review, but the
guidelines nominally apply; all of these bind at App Store release)
- [ ] **Guideline 4.2 (minimum functionality):** a thin wrapper that just loads
  tools.nooutco.me remotely risks rejection. A **Capacitor 8** build with locally bundled
  assets, offline handling (reviewers use airplane mode), and native touches (push, haptics)
  is routinely approved.
- [ ] **Guideline 5.1.2(i) (updated Nov 2025):** apps must clearly disclose and get **explicit
  permission before personal data is shared with third-party AI** — this directly covers
  "Wish It" sending schedule/client data to an LLM, even through your own Worker proxy. Build
  a consent step (and scrub names — see below).
- [ ] **Guideline 5.1.3 (health):** ABA appointment data with client names is identifiable
  therapy information — safest posture is to treat it as health data: declare it in the
  privacy nutrition label (App Store stage), keep it **out of iCloud**, no third-party
  disclosure.
- [ ] **5.1.1(v):** if the app grows accounts, in-app account deletion is required.
- [ ] Age-rating questionnaire (new 2026 tiers) — a constrained scheduling assistant should
  rate normally; unfiltered generative chat would force 18+.

**Privacy architecture constraint (from this repo's own policy)**
`docs/hipaa-baa-zdr.md` establishes the working posture: **no BAA, no ZDR → PHI must never
reach the LLM API**; client-side de-identification before send; the name→token map is itself
PHI and must never be stored/transmitted in plaintext. For SAssi this means:
- Store schedule data **locally on device** (or encrypted client-side if synced) — plaintext
  client names in Cloudflare KV would be outside the documented posture.
- "Wish It" must scrub names to role tokens before the prompt leaves the device, mirroring
  the NoteDrafter scrub pipeline.
- This local-first design *also* satisfies Apple's health-data and third-party-AI rules —
  the compliance and review incentives point the same direction.

---

## 6. Recommended roadmap

**Phase 0 — locate the real SAssi (decision needed).** If the shipped app referenced by the
design bundle lives in `aba-dashboard` (or another repo), audit *that* codebase — its stack
decides the wrap path (already-native? React web? something else?). Everything below assumes
building from this repo's prototype.

**Phase 1 — ship the PWA now ($0, ~immediately after Phase 2 basics).** You already run
Cloudflare Pages on this domain with an auth system and an LLM proxy. A `manifest.json` +
service worker + the existing static-hosting setup makes SAssi installable from Safari with
**no 7-day expiry, no Apple account, offline support, and push**. This solves "persists on my
devices" this month, while the Apple track proceeds. (One repo-specific caveat: Bot Fight
Mode challenges API fetches — the notes tools' `.js`-suffix workaround, or the OOS.md WAF
skip-rule, applies to any SAssi API calls too.)

**Phase 2 — make the product real (the actual work).** Items 1–6 of §3: components, calendar
math, add/edit, local-first persistence, computed compliance, Wish It → `/api/llm-call` with
consent + name-scrubbing. Weeks, not days.

**Phase 3 — Apple track.** Enroll in the Developer Program ($99) → Capacitor 8 wrap with
bundled assets → Codemagic (or GH Actions) signing + upload → **TestFlight internal** on your
phone (no review, 90-day re-upload cadence — automatable from CI on a monthly cron).

**Phase 4 — terminal state.** When stable, submit an **Unlisted App Store release**: one full
App Review (privacy labels, AI-consent flow, health-data posture, entity account), then the
app is on your devices permanently with zero re-signing forever after.

### Cost summary
| | One-time | Recurring |
|---|---|---|
| PWA route | $0 | $0 |
| TestFlight route | — | $99/yr + a re-upload every <90 days (CI-automatable) |
| Unlisted App Store | one App Review effort | $99/yr (and installed apps survive even a lapse) |
