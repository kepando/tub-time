# Tub Time 🦆

A personal, installable PWA for managing Ken's hot tub: log tests and soaks, generate
Claude-powered treatment plans, and walk through them with an interactive checklist.
Data lives in OneDrive; auth is Microsoft (MSAL); the plan engine is the Claude API.

Built mobile-first for iPhone (Add to Home Screen). Single-file app (`index.html`) in the
kepando-dev house style — vanilla JS, OneDrive JSON storage, GitHub Pages.

- **Live:** `https://kepando.github.io/tub-time/`
- **Reference patterns:** ForgeTrack (MSAL + Graph + PWA), JobScout (browser-direct Claude key).

---

## 1. Entra app registration (one-time)

Create a **new, dedicated** registration — do not reuse another app's client ID.

1. [portal.azure.com](https://portal.azure.com) → **Microsoft Entra ID → App registrations → New registration**.
2. **Name:** `Tub Time`
3. **Supported account types:** *Personal Microsoft accounts only* (authority `.../consumers`).
4. **Platform:** *Single-page application (SPA)*.
5. **Redirect URIs** (SPA type):
   - `https://kepando.github.io/tub-time/` (production)
   - `http://localhost:5500/` (or whatever port you serve locally)
6. **API permissions** → *Microsoft Graph → Delegated*:
   - `User.Read`
   - `Files.ReadWrite`
   - `Calendars.ReadWrite` — **Phase 2** (add when calendar scheduling ships)
7. Copy the **Application (client) ID**.
8. In `index.html`, set `CONFIG.MSAL_CLIENT_ID` to that value.

Until the client ID is set, the app shows a config hint on the sign-in screen and won't sign in.

---

## 2. Claude API key

**The key is never in the code or the repo.** This is a public GitHub Pages site, so the
JobScout pattern is used: the key is entered once in-app (**Settings → Anthropic API key**)
and stored in your OneDrive at `settings.json`, synced across your devices. Calls go
browser-direct to `api.anthropic.com` with the `anthropic-dangerous-direct-browser-access`
header — appropriate for a single-user personal app on your own devices.

> Tradeoff: the key is present in your browser at runtime. If that's ever not acceptable,
> the alternative is a small Azure Function that validates your Entra token and holds the
> key server-side (proposed in the spec, not built in Phase 1).

Model: `claude-opus-4-8` (held in `CONFIG.CLAUDE_MODEL`; supports image input for Phase 2
label photos). Plans are requested as structured JSON, validated, and retried once before
falling back to raw text.

---

## 3. Where the data lives

Everything is under **`/zApplications/TubTime/`** in your OneDrive — nothing is ever written
to the OneDrive root. Path is in one constant (`CONFIG.BASE_PATH`).

```
/zApplications/TubTime/
  settings.json        tub profile, targets, intervals, scheduling windows, API key
  products.json        product catalog + strip profiles (pad order & chart values)
  tests.json           test sessions: readings → plan → steps done/changed → retest
  soaks.json           soaks: date, duration, people, notes
  maintenance.json     filter, drain/refill, enzyme, repairs (Phase 2 UI)
  events.json          app-created Outlook events (Phase 2)
  knowledge/
    rules.md           chemistry rules — sent as the system prompt every request
  products/images/     label photos (Phase 2)
```

Each JSON file carries a `schemaVersion`. Writes use eTag `If-Match` conflict handling, a
1.2 s debounce, and a local mirror + retry queue in `localStorage` — a failed save is
queued and retried (on interval, on reconnect), so a log entry is never silently lost.
A JSON **Export / backup** button is on the Settings page.

---

## 4. Deploy to GitHub Pages

1. Create a **public** repo named `tub-time` under the `kepando` account.
2. Push these files (repo root): `index.html`, `manifest.json`, `service-worker.js`,
   `duck.svg`, the five icon PNGs, `README.md`.
3. **Settings → Pages → Source:** Deploy from branch → `main` / root.
4. Visit `https://kepando.github.io/tub-time/`, sign in, add the API key in Settings.
5. On iPhone Safari: **Share → Add to Home Screen** to install.

The `build/` folder (icon render scaffolding) is gitignored and not deployed.

### Icons / branding
The rubber-ducky mark is an original SVG (`duck.svg`). Icons are generated from it:
`apple-touch-icon` (180), manifest icons (192 / 512), a maskable 512 with safe padding,
and a 32px favicon — all on a water-blue background so the iPhone home-screen icon isn't
clipped or shown on black. To regenerate after editing the duck, re-run the headless-Chrome
render in `build/` (`render-icon.html` / `render-maskable.html`).

---

## 5. Secrets check before each push

No secrets belong in this repo. Quick check:

```sh
grep -rniE 'sk-ant|x-api-key.*sk|ANTHROPIC_API_KEY=.+' . --exclude-dir=.git
```

The MSAL client ID is a public SPA identifier (safe to commit). The Anthropic key lives only
in OneDrive.

---

## 6. Phases

- **Phase 1 (this build):** auth, OneDrive storage, settings + rules editor, seeded products
  with a strip profile driving the test form, test → Claude plan → interactive checklist
  (timers that survive app close, retest loop), soak log, treatment log, dashboard, backup.
  *Confirm it works as an iPhone home-screen app before Phase 2.*
- **Phase 2:** label-photo extraction (products + strip profiles), maintenance/repair log
  with intervals, Outlook calendar scheduling (`Calendars.ReadWrite`, `calendarView`,
  propose-before-create).
- **Phase 3:** trend charts, smarter dashboard insights, fresh-fill startup plan on refill.

---

## 7. Notes on seeded data to confirm

- The seeded strip profile lists **7 pads** in the Apple Notes order (Total Hardness, Total
  Chlorine, Free Chlorine, Bromine, Total Alkalinity, Cyanuric Acid, pH). The spec also said
  "6 pads" — confirm against the actual bottle and trim/adjust in Products.
- Strip **chart values** per pad are common-strip placeholders — confirm them against your
  bottle's color chart (Phase 2 photo extraction will set these automatically).
