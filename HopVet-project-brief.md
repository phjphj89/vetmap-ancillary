# HopVet — Project Brief for Claude

**Purpose of this document:** this is a context-transfer brief for a rabbit-friendly vet clinic finder website called **HopVet**, built collaboratively across a long conversation with Claude. If you're a new Claude instance reading this, treat it as a condensed memory of that entire project — the decisions, the bugs we found and fixed, and the reasoning behind non-obvious choices. The goal is that you shouldn't have to rediscover any of this the hard way.

---

## 0. Read this first: working environment caveat

**Claude's sandboxed file-editing environment resets — sometimes even mid-conversation, not just between separate chats.** This has actually happened during this project: Claude's local copy of the codebase vanished partway through a session despite full memory of everything built.

**Practical consequence:** if you're a fresh Claude instance (or even a continuation of this same chat after time has passed), **ask the user for the current project zip before making any code changes.** Don't assume you have the files just because you remember building them. The user has been keeping timestamped copies of every zip specifically for this reason.

---

## 1. Project Overview

- **What it is:** a directory site helping people in the Czech Republic find veterinarians who are genuinely experienced with rabbits (and other small mammals). The core problem: rabbits are the 3rd most commonly kept pet in the Czech Republic, but most vet clinics have no one trained in their care. This site is a curated, hand-vetted list of clinics that do.
- **Owner:** Pepa (goes by Pepa in conversation), a real estate investment analyst in Prague, learning to code. Primary background language is Python. Genuinely wants to understand the code, not just have it work — prefers explanations, incremental changes, and being consulted on tradeoffs.
- **Site name:** HopVet. Tagline: "Rozumíme králíčkům" / "We understand rabbits" (deliberately short, moved from a longer descriptive tagline for space reasons on mobile).
- **Scope:** single-page-ish React app with client-side routing. No backend, no database, no auth — entirely static, hosted free on Netlify, deployed via manual GitHub web-UI uploads (not CLI/git push — Pepa drags files into GitHub's web interface).
- **Bilingual:** Czech (default) and English throughout, via a custom translation system (not a library).
- **Related but separate projects** (mentioned early on, not actively worked on recently): a multi-shelter rabbit adoption platform (Supabase-based, different repo) and an early-stage rabbit health-tracking app idea. Don't conflate these with HopVet.

---

## 2. Tech Stack

- **Frontend:** React + Vite, React Router (`react-router-dom`, real client-side routing with actual URLs)
- **Map:** React-Leaflet + Leaflet, OpenStreetMap tiles (no API key needed)
- **Hosting:** Netlify, free tier
- **Deployment method:** Pepa deletes and re-creates the GitHub repo, then drags the **entire unzipped project folder** into GitHub's web upload UI. Netlify auto-deploys from GitHub on push. This "delete and recreate" approach was adopted after repeated repo-sync issues.
- **No backend/database/auth** — all data lives in a single `clinics.json` file, edited via a standalone HTML tool (see §8).
- **Fonts:** self-hosted (not Google Fonts CDN) — Inter for body text, Fredoka for the "HopVet" title only (see §10 for a real bug we hit with font self-hosting).

---

## 3. Repository Structure (key files)

```
vet-map-cz/
  src/
    App.jsx                  # Shell: header (logo, language toggle, hamburger menu), routing
    App.css                  # All site styles (one big file)
    index.css                # CSS custom properties (color tokens), @font-face rules, global resets
    main.jsx                 # Entry point, font imports
    components/
      HomePage.jsx            # "/" route: map + filtered clinic list
      ClinicMap.jsx            # Leaflet map, custom pin rendering, pin color/icon logic
      ClinicList.jsx           # Filter row (kraj, distance, special filters) + result grid
      ClinicSummaryCard.jsx    # Lean card shown in the list/grid view
      ClinicCard.jsx           # Full-detail card, used ONLY on the detail page now
      ClinicDetailPage.jsx     # "/clinic/:id" route wrapper around ClinicCard
      OpeningHoursTable.jsx    # Renders the two-column (Regular/Emergency) hours table
      FriendsPage.jsx          # "/friends" route: partner orgs grid
      HamburgerMenu.jsx        # About/Contact/Friends link/map attribution dropdown
      LiveStatus.jsx            # "Currently Open/Closed/Emergency" pill, live-computed
      BackToTopButton.jsx      # Floating scroll-to-top button, all pages
      Badge.jsx                 # Generic colored pill component (icon + text)
      StatusIcons.jsx           # All small icon components (see §9)
      FlagIcons.jsx              # Czech/UK flag SVGs (hand-drawn, not emoji — see §10)
      BloodDonorButton.jsx      # Static link to external blood-donor rabbit charity
      SocialIcons.jsx            # Facebook/Instagram icons
    utils/
      liveStatus.js             # getLiveStatus() / getNextOpening() — real-time open/closed logic
      distance.js                # Haversine distance calc
      formatTime.js               # "9" -> "9:00" style formatting
      tempLocation.js              # TEMPORARY fixed Prague coordinates (see §11)
    i18n/
      LanguageContext.jsx          # CS/EN context provider
      translations.js               # ALL UI strings, both languages, plus weekday name maps
    data/
      clinics.json                   # The clinic database (see §4 for full schema)
      friends.json                    # Partner orgs (see §7)
      siteConfig.js                    # lastUpdated date, contact email
  public/
    fonts/                              # Self-hosted Inter + Fredoka .woff2 files
    images/friends/                      # Partner logo images (manually added, see §7)
    favicon.svg                           # Rabbit-ears mark
    _redirects                             # Netlify SPA redirect rule (critical for routing)
  netlify.toml
```

**Standalone tools (NOT part of the React project, separate single-file HTML apps):**
- `clinic-editor.html` — the primary data-editing tool (§8)
- `friends-editor.html` — simpler equivalent for `friends.json`

---

## 4. Data Model: `clinics.json` (full field reference)

Each clinic is one object in a flat array. **~88 clinics** as of last update. Field-by-field:

### Identity & location
| Field | Type | Notes |
|---|---|---|
| `id` | number | Unique, auto-incremented by the editor tool |
| `name` | string | |
| `region` | string | Free-text city label, e.g. "Brno" |
| `address` | string | |
| `lat`, `lng` | number | **Critical: must be real numbers, never null.** A null-coordinate clinic once crashed the entire site (Leaflet threw on marker creation with no error boundary to catch it) — there's now a defensive filter in `ClinicMap.jsx` that excludes any clinic with non-numeric coordinates before rendering markers, but don't rely on that as a substitute for entering real data |
| `kraj` | string | One of 14 Czech kraj names, computed originally via point-in-polygon geocoding |
| `district` | string | Informational only, not used in any app logic |
| `is_prague`, `is_brno`, `is_ostrava` | boolean | Independent flags layered on top of `kraj` for the region filter (Prague/Brno/Ostrava clinics are excluded from their parent kraj's filter bucket to avoid double-counting) |

### Contact
| Field | Type | Notes |
|---|---|---|
| `phone` | string | Raw/full text, may contain multiple numbers separated by `/` |
| `call_phone` | string | **The single number actually used for the Call button's `tel:` link.** Auto-migrated from `phone` (first number). The editor tool shows "quick pick" buttons when `phone` contains multiple numbers, letting Pepa choose which becomes `call_phone` without losing the others |
| `website` | string | |
| `google_maps_url` | string | **Must use Google's documented universal format**, not the shorthand. See §10 for why — the shorthand `?q=place_id:XXX` format is unreliable when a mobile browser hands off to the Maps app. Correct format: `https://www.google.com/maps/search/?api=1&query=<url-encoded-name>&query_place_id=<place_id>` |
| `facebook_url`, `instagram_url` | string | Empty string if none |
| `internal_contact_email` | string | **Never displayed publicly** — for Pepa's own outreach/newsletter use |
| `internal_notes` | string | **Never displayed publicly** — free-text notes for Pepa's own reference |

### Hours (see §6 for full explanation of the structure)
| Field | Type | Notes |
|---|---|---|
| `weekly_hours` | object | Keyed by `Mon`/`Tue`/.../`Sun`/`Holiday`, each a structured day-entry (see §6) |
| `hours` | string | Legacy raw text, kept as a fallback display and as a reference for manual re-entry |
| `hours_needs_review` | boolean | If true, the site shows `hours` (raw text) instead of the structured table — used for the ~19 clinics whose original hours text was too irregular to cleanly convert |

### Manual availability/emergency flags (all independent booleans, NOT derived from the hours grid — see §10 for why)
| Field | Meaning | Drives |
|---|---|---|
| `is_24_7` | Open 24/7 | Badge + red map pin |
| `open_weekends_bookable` | Open on weekends, normal bookable visit | "Otevřeno o víkendu" badge, calendar icon |
| `has_weekend_emergency` | Open on weekends but emergency-only, not bookable | "Víkendová pohotovost" badge, the user's custom e911-style icon |
| `after_hours_emergency` | Accepts emergencies outside normal opening hours | "Pohotovost po otevírací době" badge, moon icon |
| `emergency_on_phone` | Vet reachable by phone for emergencies, but clinic itself closed/no vet on site | "Pohotovost na telefonu" badge, phone+alert-dot icon |
| `hospitalization` | Offers overnight/stay-in care | Bed icon badge |
| `top_pick` | Pepa's personal recommendation | Crown icon badge, "Doporučeno"/"Top pick" — shown on both list and detail views, independently of other badges (not competing for a priority slot) |

**Important design note:** these flags used to be partially *derived* from the hours grid (e.g. "is this clinic open on Sat/Sun according to its listed hours?"). That was abandoned — Pepa explicitly said hours data is too ambiguous to reliably infer these real-world distinctions from (a Saturday "9:00–12:00" could mean a short normal day, an emergency-only window, or something else — no parsing logic can recover that). All four+ flags above are now 100% manual, editor-tool-controlled checkboxes. Filters, badges, and map pin coloring all read from these same flags, so nothing can show mismatched info.

### Visibility flags
| Field | Type | Notes |
|---|---|---|
| `published` | boolean | **If not `true`, the clinic is completely hidden** — from the map, the list, AND the detail page (even via direct URL, deliberately, so an "in progress" clinic can't be found via an old shared link). Existing clinics default `true`; new clinics added via "+Add new clinic" default `false` until Pepa finishes and confirms them |
| `has_rabbit_specialist` | boolean | Controls map pin appearance only (not visibility) — `true` shows the rabbit-head icon inside the pin, `false` shows a small plain filled circle (same size as the rabbit head, same pin color) instead. Existing clinics default `true` (this whole site is pre-curated for rabbit expertise); new clinics default `false` |
| `show_reviews` | boolean | Controls whether Google rating + review count are *displayed* on the site. The underlying `google_rating`/`google_review_count` data is always kept regardless — this is purely a display toggle so Pepa can track ratings privately without showing them publicly. Default `true` |
| `manually_checked` | boolean | Pepa's own "I've personally reviewed this clinic's data" tracker — shows a red ❗ next to the clinic in the editor's sidebar list if `false`. Completely independent of `hours_needs_review` (which has its own separate orange dot indicator) |

### Other
| Field | Type | Notes |
|---|---|---|
| `notes_en`, `notes_cs` | string | Bilingual free text shown on detail page |
| `emergency_en`, `emergency_cs` | string | Bilingual free text, emergency-specific notes |
| `recommended_vet` | string | **Plain text, deliberately not a structured list** — may contain multiple names comma-separated (e.g. "MVDr. Jekl, MVDr. Hauptman"). The detail/list pages split this by comma at *display time* to render each as its own boxed card, without needing a data-model change. Explicitly decided against restructuring this into an array — kept simple on purpose |
| `english_communication` | boolean | Shown as "Ano"/"Ne" ("Yes"/"No") on detail page |
| `google_rating`, `google_review_count` | number\|null | See `show_reviews` above for the display toggle |

---

## 5. Design System

### Colors (CSS custom properties in `src/index.css`)
- `--color-bg`: `#FBFAF5` (page background — was `#FFFFFF` at one point, changed manually by Pepa directly in the file)
- `--color-accent`: `#006662` (main brand green — buttons, active states, borders; has changed a couple of times through the project, currently this teal-green)
- `--color-card-bg`: `#FFFFFF` (card backgrounds)
- Header background: hardcoded `#FFFFFF` (deliberately *not* tied to `--color-bg`, so it stays visually distinct from the page background even if that changes)
- Badge/pill colors: several semantic pairs (bg + text) for each badge type — hospitalization (sage green), weekend-emergency (plum/purple), open-weekends (terracotta), emergency/24-7 (red), live-status pills (light green/red/purple backgrounds)
- Map pins: 3 colors — green (standard), terracotta (`#C97B4A`, "extended" — any emergency-type flag true, or closes after 19:00), red (`#A83B32`, 24/7)

### Fonts
- **Body/UI text:** Inter, self-hosted (weights 400/600/700)
- **"HopVet" title only:** Fredoka, self-hosted (weights 600/700) — deliberately *not* used for clinic names (tried it, diacritics didn't render well, reverted)
- **Critical gotcha:** see §10 — don't use `@fontsource/*` package's auto-generated CSS import for self-hosted fonts. Write `@font-face` rules by hand instead, referencing files copied into `public/fonts/`.

### Icons
All hand-built as small React SVG components in `StatusIcons.jsx` (13–14px, mostly stroke-based `fill="none" stroke="currentColor"`), matched to specific badges:
- `ClockIcon` — 24/7
- `CalendarIcon` — open on weekends
- `WeekendEmergencyIcon` — weekend emergency (this one is the user's own provided "e911-emergency-rounded" icon, filled style — kept exactly as given rather than converted to stroke style)
- `MoonIcon` — after-hours emergency
- `PhoneAlertIcon` — emergency on phone (phone handset + small filled alert dot)
- `BedIcon` — hospitalization (custom-designed after an iterative visual-preview process with the user — see git history/commit messages for the design iteration)
- `CrownIcon` — top pick / recommended (also user-provided reference)
- `PhoneIcon`, `MapPinIcon`, `GlobeIcon` — action buttons (Call/Maps/Website)

### Layout notes
- Desktop content capped at `max-width: 1200px`
- Map sits full-width at the top of the home page (not a sidebar — was restructured this way partway through)
- Filter row is "full-bleed" (borders span the full window width, not just the content column) using the `margin-left/right: calc(50% - 50vw)` CSS technique — **use this exact technique, not `left: 50%; right: 50%`**, which over-constrains the box and causes inconsistent centering (this was a real bug we hit and fixed)
- Mobile breakpoint: `900px` for most layout changes, `700px`/`480px` for a couple of narrower-specific tweaks

---

## 6. Opening Hours Data Structure

`weekly_hours` is an object with keys `Mon, Tue, Wed, Thu, Fri, Sat, Sun, Holiday`. Each value is:

```json
{
  "closed": false,
  "start1": "09:00", "end1": "12:00",
  "start2": "13:00", "end2": "18:00",
  "emergency_start1": "18:00", "emergency_end1": "22:00",
  "emergency_start2": "", "emergency_end2": "",
  "note_en": "", "note_cs": ""
}
```

- Up to 2 regular ranges per day (handles lunch-break splits) and up to 2 *separate* emergency ranges — added later specifically so a clinic can show "8:00–18:00 regular / 18:00–22:00 emergency" as two distinct columns rather than cramming it into a note
- The site renders this as a **two-column table**: "Ordinary Opening Hours" / "Emergency" (`t.regularHoursHeading` / `t.emergencyHoursHeading`), with today's row highlighted (light background) based on the visitor's actual current day
- `note_en`/`note_cs` are for genuinely irregular exceptions that don't fit clean time ranges (e.g. "by phone only", "1st Saturday of the month")
- Emergency-hour fields are **intentionally independent of the `closed` checkbox** — a day can be closed for regular business but still have emergency hours

---

## 7. Friends & Partners (`friends.json` + `/friends` route)

Much simpler schema, small list (not 80+ entries):
```json
{
  "id": 1,
  "name": "...",
  "logo_url": "/images/friends/example.png",
  "website_url": "...",
  "facebook_url": "...", "instagram_url": "...",
  "tags_en": ["Sanctuary", "Rabbits & Rodents"], "tags_cs": [...],
  "description_en": "...", "description_cs": "..."
}
```
- **`logo_url` gotcha:** must NOT include `/public/` in the path. Files in Vite's `public/` folder are served from the site root — a file at `public/images/foo.png` is referenced as `/images/foo.png`, not `/public/images/foo.png`. This tripped Pepa up once.
- Logo images use `object-fit: cover` (fills the banner completely, may crop) — was `contain` originally, changed because `contain` left visible letterboxing gaps when an image's aspect ratio didn't match the banner.
- Editor tool: `friends-editor.html`, same pattern as the clinic editor but much simpler (no day-by-day hours, no kraj filter needed for a small list).

---

## 8. The Editor Tools

Both are **single self-contained HTML files** — no build step, no server, no dependencies. Open directly in a browser. Pattern: load JSON file → edit via form → download updated JSON → manually upload to GitHub alongside the rest of the project.

### `clinic-editor.html` (primary tool)
Key features, roughly in the order they were built:
1. Load/download `clinics.json`, add new clinic (auto-incrementing ID)
2. Day-by-day hours grid: plain text `HH:MM` inputs (not native `<input type="time">` — see §10 for why), with auto-formatting on blur ("9" → "09:00")
3. Emergency-hours row per day, independent of the regular-hours closed checkbox
4. Phone "quick pick" buttons when `phone` contains multiple numbers (split on `/`)
5. Kraj filter dropdown + search box for the sidebar clinic list (built dynamically from whatever kraj values exist in the loaded data)
6. "Checked" checkbox (❗ red mark if unchecked) — Check All / Uncheck All bulk buttons
7. "Published" checkbox (🚫 mark + dimmed row if unpublished)
8. All the manual availability/emergency checkboxes (§4)
9. "Has rabbit/small mammal specialist" checkbox
10. "Show reviews" checkbox (next to the rating/review-count number fields)
11. Internal-use-only section: contact email + free-text notes, clearly separated with a "never shown publicly" label

**Testing approach used throughout:** no real browser/Puppeteer available in Claude's sandbox (network-blocked from downloading a Chromium binary). Testing was done via **jsdom** (a pure-JS DOM implementation) — load the real HTML file, dispatch real events, assert on the resulting DOM/data state. This caught real bugs (e.g. a bug where typing into the native `<input type="time">` widget didn't reliably fire `input` events across browsers — jsdom couldn't have caught that specific one since it doesn't replicate native widget quirks, which is why the fix was to switch to plain text inputs entirely, sidestepping the whole class of risk rather than patching around it).

### `friends-editor.html`
Same architecture, much simpler, no day-grid complexity.

---

## 9. Routing

React Router (`react-router-dom`), real URLs:
- `/` — home page (map + filtered list)
- `/clinic/:id` — detail page
- `/friends` — partners page

**Netlify requires a `public/_redirects` file** containing `/*    /index.html   200` — without this, direct navigation to `/clinic/123` (not clicked from within the app) 404s, because Netlify's static server doesn't know about client-side routes by default.

---

## 10. Known Bugs Found & Fixed (read before touching related code)

These are worth preserving because several took real debugging effort — don't reintroduce them.

1. **CSS cascade source-order bug:** a "mobile: don't stick" override for the filter row was placed *earlier* in `App.css` than the desktop `position: sticky` rule it was meant to beat. Since both were equal-specificity single-class selectors, CSS resolves ties by source order — the later (desktop) rule silently won even on mobile screens, for weeks, despite the media query correctly matching. **Fix/lesson:** all mobile media-query overrides now live in one block at the very end of `App.css`, guaranteeing they win against anything they're meant to override, regardless of which specific selector.

2. **Vite CSS minifier bug with self-hosted fonts:** using `@fontsource/*` package's auto-generated CSS (`import '@fontsource/fredoka/700.css'`) caused Vite's production minifier to silently *drop* the `latin-ext` unicode-range subset (needed for Czech diacritics: č, ř, ž) — worked fine in an unminified build, broken in production. Confirmed by comparing minified vs. unminified output. **Fix:** don't import the package's generated CSS at all. Copy the actual `.woff2` files into `public/fonts/`, and write `@font-face` rules by hand in `index.css`. This sidesteps the specific CSS structure that triggers the bug. Applied to all three self-hosted fonts (Nunito historically, now Inter + Fredoka).

3. **Leaflet marker crash on null coordinates → entire page blank:** a clinic with `lat: null, lng: null` (an incomplete "+Add new clinic" entry that got exported before being filled in) crashed Leaflet's marker creation with an uncaught exception. No error boundary existed, so React's whole render tree died — the entire site went blank, not just that one pin. **Fix:** (a) removed the bad entry from data, (b) added a permanent filter in `ClinicMap.jsx` that excludes any clinic with non-numeric lat/lng *before* attempting to render a marker for it, so this specific failure mode can't recur regardless of future data mistakes.

4. **z-index / stacking context gotcha:** the hamburger dropdown menu is nested *inside* the header in the DOM. Since the header itself is `position: fixed` with its own `z-index`, it creates a new stacking context — meaning the menu's own high z-index could only ever win *within* that context, not against things outside it (like the map, which has a lower effective stack level but was still winning). The real fix was raising the **header's** z-index (the actual boundary), not just the menu's. Current values: header `2000`, menu backdrop `2100`, menu panel `2200` — deliberately well clear of Leaflet's internal control/pane z-indexes (which default to values in the low thousands).

5. **Mobile pinch-zoom causing "detached" UI controls:** map's zoom controls and the header's language/menu buttons appeared visually misplaced on some phones. Root cause theory (never 100% confirmed, but the fix reportedly helped): allowing page-level pinch-zoom on a page containing an interactive map can let users accidentally zoom the *whole page* via the map, throwing off the relationship between rendered layout and native browser chrome. Fix: `<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">` — the map keeps its own separate zoom controls (relocated to bottom-right specifically for one-handed phone use), so disabling page-level pinch-zoom doesn't remove any map functionality.

6. **Google Maps `place_id:` shorthand unreliable on mobile:** `https://www.google.com/maps/place/?q=place_id:XXX` opens the Google Maps app on mobile but frequently fails to resolve to the actual pin (opens to a blank/generic map). This is a documented Google quirk with that shorthand specifically when a browser hands off to the native app. Fixed by batch-migrating all affected URLs (81 of them) to Google's officially documented "universal" format: `?api=1&query=<name>&query_place_id=<id>` — which reliably works in both web and app contexts. The place_id was already embedded in the old URLs, so this was a fully automated regex-based migration, no manual re-entry needed.

7. **Full-bleed CSS technique — use margin, not left/right:** `position: relative; left: 50%; right: 50%; margin-left: -50vw; margin-right: -50vw;` over-constrains the box model and produces inconsistent horizontal centering across browsers. The correct, standard technique is margin-only: `width: 100vw; margin-left: calc(50% - 50vw); margin-right: calc(50% - 50vw);` — no `left`/`right` properties at all, works regardless of `position` value (static/relative/sticky).

8. **Distance/rating filter mismatch caused by wrong file uploaded:** several rounds of "your fix didn't work" turned out to be Pepa accidentally re-uploading a stale local copy of `clinics.json` from much earlier in the project (missing months of migrations). Diagnosed by diffing the uploaded file's schema against expected current fields. **Lesson for future Claude:** if a reported bug doesn't reproduce in your own testing and the user says "still broken" after a redeploy, ask them to re-confirm they uploaded the *current* file, and consider asking for the actual file to diff rather than assuming your fix is correct or incorrect.

---

## 11. Known Temporary/Incomplete Pieces

- **`utils/tempLocation.js`** exports a hardcoded Prague coordinate (`{ lat: 50.0834140, lng: 14.4348084 }`), used as a stand-in "user location" for distance sorting/filtering everywhere. This is explicitly temporary — the real browser geolocation permission flow (`useUserLocation` hook, `LocationPrompt` component) was built early in the project but is currently **not wired up anywhere in the UI**. When reintroducing it, the rest of the distance-sorting code shouldn't need to change, since it just expects a `{lat, lng}` object.
- The Google Places API auto-rating-update idea (Python script + GitHub Actions cron) was discussed and architected early on but never implemented.
- A rabbit shelter adoption platform and a rabbit health-tracking app are separate, distinct projects mentioned early in the relationship — don't confuse their data models or goals with HopVet's.

---

## 12. Working Style / Preferences (for whoever picks this up)

- Pepa often shares **screenshots** (sometimes from an actual phone, with the URL bar visible) as bug reports — read the URL/context carefully, since more than once a reported "bug" turned out to be a misunderstanding about *which page* (detail vs. list) was being discussed.
- Prefers **being told the reasoning**, not just the fix — especially for CSS/positioning bugs, which have a habit of looking simple but having a real underlying mechanism worth explaining.
- Appreciates **honest confidence-calibration** — being told plainly when something is "my best guess, please verify" vs. "confirmed and tested" has come up as valued more than once.
- When uploading a `clinics.json` that differs from the last-known state, **diff it carefully before merging** — Pepa edits via the standalone tool independently between conversations, and diffing has caught both genuine new edits and (occasionally) accidentally-stale files.
- Rigorous by nature: has asked for icon options before implementation (e.g. bed icon — 3 proposals shown via inline SVG preview before picking), wants exact z-index/geometry math shown when diagnosing visual bugs, and values a real functional test (not just "the build succeeded") before calling something done.
- Single-file changes are preferred **when the change is genuinely isolated** to one file — but most CSS/data changes in practice ripple across at least 2 files, so default to full-project zips unless explicitly confirmed otherwise.

---

## 13. Quick-Start Checklist for a New Conversation

1. Ask for the current project zip (don't assume file state persists).
2. Extract, `npm install`, `npm run build` to confirm baseline works before changing anything.
3. If given a new `clinics.json`, diff it against the extracted project's copy before overwriting.
4. For any visual/CSS change, remember: mobile overrides go at the end of `App.css` (source-order matters), full-bleed elements use margin-only centering, and z-index conflicts are usually about the *ancestor's* stacking context, not just the element itself.
5. For icon/visual design requests, consider showing a quick preview (the Visualizer tool) before wiring into real code, especially for anything the user hasn't given exact reference art for.
6. Test with real data where possible — this project has a working jsdom-based testing pattern (see any of Claude's `jsdom-test.js` files bundled with the editor tool) that's proven effective for catching real bugs beyond "the build succeeded."
