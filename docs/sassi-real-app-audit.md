# SAssi Cal (aba-dashboard) — Real-App Audit & TestFlight Punch List

*Audited 2026-07-14 against `mgfv2r2mpv-hash/aba-dashboard` `main` @ `e9e1886` (2026-07-13).
Companion to `sassi-testflight-audit.md`, which audited the design prototype in this repo and
the Apple publishing landscape. This document supersedes that report's Phase 0–2 roadmap:
the real app exists, is already Capacitor-wrapped with an Xcode project, and is far more
complete than its own documentation claims.*

---

## 1. Verdict

**SAssi Cal is a mature, healthy, near-shippable iOS app.** React 19 + Vite + Capacitor 8
(`appId` intent: `com.abascheduling`, name "SAssi Cal", v1.1.0), fully serverless on device
(all `/api/*` calls resolve in-memory via `src/nativeApi.ts`), with PIN + Face ID app lock,
AES-GCM at-rest encryption, an anonymizing boundary in front of the Anthropic API, and a
Cloudflare-Access-gated web portal companion.

**Quality evidence (all verified by actually running them in this session):**
- `tsc --noEmit`: **0 errors** · client + server builds pass (Vite 8, 513 ms)
- Vitest: **204/204 tests passing** (23 files, coverage thresholds configured)
- Verify harnesses: **181/181 checks passing** across compliance, wish, excel, builders,
  fixit, casemodel (24 domain harnesses exist in `scripts/`)
- E2E: 1 smoke spec only (boot + mount) — the one thin spot in test coverage
- Zero telemetry/analytics SDKs; only outbound calls are Anthropic and Google Maps routing

The 7-day expiry you experience is free-account (or dev-profile) sideloading. TestFlight
internal fixes that to a 90-day cycle with no review; an eventual unlisted App Store release
makes it permanent.

---

## 2. Feature rundown (verified in code, not docs)

**Shipped and wired into the UI — all confirmed DONE:**
- **Calendar** (hand-rolled, not a library): month/week/day + TimeGrid with drag-to-move and
  pinch-zoom; BCBA/BT/Case lenses; client-column day view with availability heat layer;
  session-flag markers (streaks, cancel-escalation, makeup, holiday).
- **Appointment lifecycle (Phase A):** complete (with actual-time confirmation), cancel with
  source + company-configurable reason codes, end-of-day DayReview sweep, ghost sessions.
- **Recurring series:** materialized-rows model with this/following/all edits, extension
  prompts ("series ending"), cadence profiling.
- **Draft tray:** every engine (build/tidy/wish/fix/undo) stages ops through a
  propose-preview-accept tray with solver grading (green/yellow/red).
- **Compliance dashboard (Phase B):** BCBA-confirmed supervision-credit model, per-client +
  per-tech, insurer-cap warning; the "Issues" tab.
- **Cases analytics:** per-case risk/utilization/cancel-pressure, home trends, ritual cards,
  to-dos that spawn prefilled sessions.
- **PTO/accruals** (4 accrual modes), **holidays**, **utilization**, **travel feasibility**
  (offline city-pair model + Google Routes refresh, no PHI sent).
- **Excel v2 workbook** import/export + **encrypted-JSON backup envelope** with import-diff
  preview (the newest feature, in the latest commit, shared with the web portal).
- **Schedule Tidy:** 7 rules + equivalence oracle (59/59 verify checks) — despite
  `docs/schedule-tidy-stub.md` calling it a stub.
- **Find-a-spot (Phase C):** ranked slot engine + FindTimeModal ("Move This / Replace This")
  — despite CLAUDE.md calling it "not started."
- **AI: Fix It / Wish It / sAssI chat** — all folded into the SAssiDock (issue queue,
  multi-turn chat with clarify chips, solution cards with impact summaries, meet-pace seeds).
  Models: claude-opus-4-8 / claude-sonnet-4-6 (default) / claude-haiku-4-5.
- **Deterministic builders:** month build (direct/supervision/PT/fill/occupancy/consolidate)
  plus an instant offline meet-pace solver — AI optional, not required.
- **CPR module:** standalone conditional-probability recording (Vollmer/lag-sequential
  stats, tap recorder, function-hypothesis report).
- **Security/native:** PIN lock (never stored; decryption success = verification), Face ID
  opt-in, password policy + dictionary, anonymizer with fail-closed raw-name guards.
- **Web portal** (`webportal/`): Cloudflare Pages behind Cloudflare Access SSO; decrypts the
  encrypted backup in-browser; no server-side storage, no KV/R2, no secrets.

**Genuinely not built:** fieldwork-hours tracking (Phase D — a dormant flag only).
**Legacy, not shipped:** the Express dev server (`server.ts`) + its weak AES-CBC helper —
dev-mode only; the device never runs it.

**Your repo docs are behind your code** (worth fixing so future agent sessions don't
mis-plan): CLAUDE.md says Phase C isn't started (it is), says the biometric SPM package is
unlinked (it's linked at HEAD, v10.0.0), references a WishComposer.tsx that no longer
exists, and lists authorization-utilization as deferred (it's computed in CasesHome).
HANDOFF.md and SETUP.md describe the pre-serverless era and v1 Excel. IMPLEMENTATION_PLAN.md
matches reality.

---

## 3. TestFlight punch list — upload blockers, in order

These stand between the current repo and a build on your phone via TestFlight internal
(no review; live minutes after processing; 90-day per-build expiry):

1. **Membership:** TestFlight requires the paid Apple Developer Program ($99/yr). A
   `DEVELOPMENT_TEAM` (47SMSR879V) is already committed with Automatic signing — if that's a
   paid team, this is done; if it's a free personal team, enroll first.
2. **Resolve the bundle-ID split-brain.** The Xcode project ships
   `com.abascheduler` while `capacitor.config.ts` declares `com.abascheduling` (a stale
   `capacitor.config.json` at the root still carries the old ID and should be deleted).
   Pick one, set it in the pbxproj, and create the App Store Connect app record for it.
3. **Fix `UIRequiredDeviceCapabilities`** in `ios/App/App/Info.plist` — it contains an empty
   `<string/>` alongside `armv7`, an invalid value likely to fail upload validation. Best
   practice: remove the key entirely (modern default) or leave a single valid value.
4. **Add the export-compliance key.** The app uses AES-GCM via WebCrypto for at-rest and
   file encryption — encryption beyond HTTPS, but *standard, OS-provided algorithms*, which
   are **exempt**. The plist key asks specifically about NON-exempt encryption, so the
   correct value is `ITSAppUsesNonExemptEncryption = false` (in App Store Connect terms:
   "uses encryption → qualifies for exemption"). Without the key, every build sits in
   "Missing Compliance" until answered by hand. *[Fixed on `dev` 2026-07-14.]*
5. **Version discipline:** `MARKETING_VERSION 1.0` / `CURRENT_PROJECT_VERSION 1` are
   hardcoded in the pbxproj and disconnected from package.json's 1.1.0. Bump
   `CURRENT_PROJECT_VERSION` on every upload (agvtool or a build script), or the second
   upload collides.
6. **Produce the web assets on a Mac or CI:** `dist-client/` and `ios/App/App/public/` are
   gitignored, so a fresh clone can't archive until `npm install && npm run cap:ios` runs.
   There is currently **no iOS CI** (the five GitHub workflows are all ubuntu; no fastlane).
   Options: your Mac + Xcode Organizer, or Codemagic (500 free macOS min/month, publishes to
   TestFlight via an App Store Connect API key), or GitHub Actions `macos-26`.
7. **Xcode 26 required** for all uploads since 2026-04-28 — Capacitor 8 already mandates it,
   so any current toolchain works; just don't archive from an old Xcode.

Cleanups worth doing while in there: committed `xcuserdata/` and a Finder-duplicate
`config 2.xml` in the Xcode project; `patch-spm-biometric.cjs` clobbers the v10 biometric
plugin's shipped Package.swift with a v9-era template on every `cap:ios` run (benign today,
fragile tomorrow — gate it on plugin version or delete it); `npm install` depends on
`cdn.sheetjs.com` and sharp's GitHub downloads (fine on normal CI, but vendor the xlsx
tarball if you want reproducibility).

**Already done — the things people usually forget:** 1024×1024 App Store icon with alpha
pre-stripped (the classic rejection, pre-avoided), full splash set including dark mode,
`NSFaceIDUsageDescription` present, biometric plugin linked, deployment target 15.0 uniform,
Package.resolved committed, no entitlements/background modes to justify.

---

## 4. Security & privacy findings (ranked; file:line cites in the session record)

These do **not** block TestFlight internal, but items 1–4 matter for external
TestFlight/App Store review, and several are worth fixing regardless because this is real
client data:

1. **Plaintext `.xlsx` export can carry real client + technician names.** The schedule
   password is optional; when unset, the workbook leaves the device via the share sheet
   with real names in four sheets, plus the obfuscated API key. Fix: force encryption (or
   strip names) whenever the export contains PHI.
2. **No third-party-AI consent gate — Apple 5.1.2(i) (Nov 2025) requires one.** No screen
   discloses that (anonymized) schedule data goes to Anthropic, and nothing asks permission
   before the first call. Add an explicit opt-in naming the third party and what is/isn't
   sent. (The anonymizer itself is genuinely good: roster names → tokens, notes/titles
   dropped, per-field de-anonymization, fail-closed raw-name guards.)
3. **Free-text reaches Anthropic name-scrubbed but not PHI-scrubbed.** Wish notes, Fix-It
   guidance, and chat turns only strip *roster* names — an address, diagnosis, parent name,
   or member ID typed into free text passes through verbatim. Extend scrubbing or gate
   free-text behind the consent copy.
4. **At-rest blobs are iCloud-backup-eligible and Files-app-visible.** `schedule.enc`,
   `aiconfig.enc`, `pin.verifier`, and `pin.stash` live in `Directory.Data` (iOS Documents:
   backed up by default) with `UIFileSharingEnabled=true`. Move them to
   `Directory.LibraryNoCloud` (or set the exclude-from-backup flag). Apple 5.1.3 frowns on
   personal health info in iCloud.
5. **The Face ID convenience stash defeats the encryption.** Enabling biometrics writes the
   PIN to `pin.stash` under `obfuscateKey` — AES with a *bundled, public* passphrase — so
   anyone with the file (e.g., from a backup) recovers the PIN, which unlocks the schedule
   and API key. The code comments admit the tradeoff; it's off by default, but combined
   with (4) it's the weakest link. Bind the stash to the Secure Enclave (keychain item with
   biometry access control) instead.
6. **Numeric PIN + PBKDF2 100k iterations is brute-forceable offline** given `pin.verifier`
   is a known-plaintext oracle. Raise iterations substantially and/or allow alphanumeric
   passphrases; treat the PIN gate as UX, not cryptography, until then.
7. **CPR sessions sit unencrypted in `localStorage`** (with a free-text client label) —
   outside the PIN scheme entirely, and backup-eligible. Bring them under `secureStore`.
8. Minor: API key in plaintext `sessionStorage` during a session; the obfuscated `_Config`
   key in exported workbooks is recoverable by anyone with the file; the dev Express server
   has wildcard CORS and a plaintext password-disclosure endpoint (never expose it beyond
   localhost); webportal pins `@anthropic-ai/sdk` ^0.53 vs the app's ^0.91.

**Solid ground worth stating:** the core at-rest crypto (AES-GCM-256, per-blob salt+IV,
PBKDF2) is correctly constructed; the device never talks to any server except Anthropic and
Google Maps (city centroids only, no names); there's no telemetry; the web portal stores
nothing server-side and fails closed on missing Access config.

---

## 5. What applies at each publishing tier

| Tier | Review? | What must be true |
|---|---|---|
| **TestFlight internal** (you + up to 100 ASC users) | None — live in minutes | §3 punch list only. Security items are your own risk tolerance. 90-day re-upload cadence. |
| TestFlight external (up to 10k testers) | Beta App Review, first build per version | §3 + consent gate (§4.2) + beta description/feedback email. Privacy labels not yet required. |
| App Store (incl. **Unlisted** — the install-once-forever endgame) | Full App Review | Everything: §4 items 1–4, privacy nutrition labels ("Health" category), account-deletion rule if accounts appear, and note Apple requires healthcare-adjacent apps handling sensitive data to be submitted by a **legal entity**, not an individual account. |

## 6. Recommended sequence

1. **This week (gets it on your phone):** fix §3 items 2–5 (about an hour of edits), run
   `npm install && npm run cap:ios` + archive on a Mac (or 30 min of Codemagic setup with an
   ASC API key), upload, add yourself as an internal tester. Done — 90-day builds, no review.
2. **Automate the treadmill:** a monthly CI job (Codemagic cron or GH Actions `macos-26`)
   that bumps `CURRENT_PROJECT_VERSION`, builds, and uploads — TestFlight auto-updates your
   phone and the 90-day expiry never bites.
3. **When ready for permanence:** knock out §4 items 1–5, add privacy labels, then submit an
   **Unlisted App Store** release — installed apps then work indefinitely, even if the
   membership later lapses.
4. **Housekeeping:** refresh CLAUDE.md/HANDOFF/SETUP to match the code (§2), delete the
   stale `capacitor.config.json`, un-commit `xcuserdata/`.
