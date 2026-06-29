# Spec: AI Engineer World's Fair — Mobile Themed Feed

> A single-file, mobile-first web app that turns the AI Engineer World's Fair
> open data into a **curated, themed feed** of the sessions *you* care about,
> in *your* priority order. Built to open instantly on a phone on the expo
> floor — works offline, no build step, no backend.

---

## 1. Goal & context

You're attending the **AI Engineer World's Fair 2026** (June 29 – July 2,
Moscone West, SF; ~602 sessions across ~29 tracks). The official site exposes
**open data** (sessions, speakers, MCP server, iCal, embeddings) so attendees
can build their own schedule apps.

This app is not a full schedule browser. It's a **personal radar**: it surfaces
only the sessions matching a fixed set of themes you care about, grouped into
sections **in a fixed priority order**, optimized for reading on a phone while
walking between rooms.

### Decisions already made (do not re-litigate)

| Decision | Choice |
|---|---|
| **Implementation** | A single self-contained `index.html` — vanilla JS + CSS, **no build step, no framework, no npm**. Must open by double-clicking the file or via any static host. |
| **Data source** | A **cached JSON snapshot committed to the repo** (`data/sessions.json`, `data/speakers.json`). App reads these local files. No live network calls required at runtime. |
| **Content model** | **Curated themed feed.** Only sessions matching a theme are shown. Non-matching sessions are hidden. |
| **"Claws" theme** | Interpreted as **agents / tool-use broadly** (agentic tooling, function calling, MCP, autonomous agents), grouped with personal assistants. |

---

## 2. The themes (fixed order — this ordering is the core feature)

Sections render top-to-bottom in exactly this order:

1. **Anthropic** — model lab updates from Anthropic (Claude, releases, research).
2. **OpenAI** — model lab updates from OpenAI (GPT, releases, research).
3. **Coding capabilities** — code generation, SWE agents, dev tools, copilots, eval of coding.
4. **Leadership** — keynotes, eng leadership, org/strategy, "state of AI" talks.
5. **Agents & personal assistants** — agentic tooling, tool-use, MCP, autonomous agents, personal assistants ("Claws and personal assistants").
6. **Communities** — community tracks, meetups, open-source ecosystem, DevRel.

Each theme has: a stable `id`, a display `title`, a one-line `blurb`, an emoji
`icon`, and a `matcher` (see §4).

---

## 3. Data

### 3.1 Source endpoints (for refreshing the snapshot — NOT called at runtime)

> ⚠️ **Sandbox network note:** the build/agent environment's egress proxy
> **blocks `ai.engineer` (403 policy denial)**, so the snapshot **cannot be
> fetched from inside the sandbox**. Refresh it from a machine with open
> network access (your laptop) and commit the result.

```
https://www.ai.engineer/worldsfair/sessions.json
https://www.ai.engineer/worldsfair/speakers.json
```

Refresh command (run locally, then commit):

```bash
curl -fsSL https://www.ai.engineer/worldsfair/sessions.json  -o data/sessions.json
curl -fsSL https://www.ai.engineer/worldsfair/speakers.json -o data/speakers.json
```

### 3.2 Expected shape (Sessionize-style — VERIFY against the real file)

The data originates from Sessionize. **At implementation time, open the actual
`data/sessions.json` and confirm field names before coding the parser** — the
shape below is the expected/likely structure, not verified ground truth.

`sessions.json` is typically either an array of session objects or
`[{ groupName, sessions: [...] }]`. A session object likely has:

```jsonc
{
  "id": "string",
  "title": "string",
  "description": "string | null",
  "startsAt": "2026-06-30T10:00:00",   // ISO, local SF time
  "endsAt":   "2026-06-30T10:30:00",
  "isServiceSession": false,            // breaks/lunch — exclude these
  "isPlenumSession": false,             // keynotes/plenary
  "speakers": ["speakerId", ...],       // ids referencing speakers.json
  "categoryItems": [123, 456],          // ids referencing category options (tracks)
  "roomId": 1, "room": "Room name"
}
```

`speakers.json` likely:

```jsonc
{
  "id": "string",
  "fullName": "string",
  "tagLine": "string",        // e.g. "Member of Technical Staff, Anthropic"
  "bio": "string | null",
  "links": [{ "title": "...", "url": "...", "linkType": "Twitter" }],
  "sessions": ["sessionId", ...]
}
```

Categories/tracks may live in a top-level `categories` array (each with
`items: [{id, name}]`) or be embedded per session as `categories[].categoryItems[].name`.
**The parser must resolve `categoryItems` ids → human-readable track/tag names**,
because matching (§4) reads those names. Build a flat `categoryNameById` map at load.

### 3.3 Normalized internal model

After loading + joining, normalize every session to:

```js
{
  id, title, description,
  start: Date, end: Date,
  day: "Mon" | "Tue" | "Wed" | "Thu",   // derived from start
  timeLabel: "10:00–10:30",
  room,
  speakers: [{ id, name, org, tagLine }],   // org parsed from tagLine, see §4.2
  tracks: ["Track name", ...],              // resolved category names
  searchText: "lowercased title+desc+tracks+speaker names+orgs",
  themes: ["anthropic", "agents", ...]      // assigned in §4
}
```

Exclude `isServiceSession` (breaks, lunch, registration) and any session with
no `start`.

---

## 4. Theme matching (the classification logic)

Each session is tested against every theme's matcher. **A session is assigned to
the single highest-priority theme it matches** (priority = the §2 order). This
guarantees no duplicates across sections and that, e.g., an Anthropic agents
talk shows under **Anthropic**, not **Agents**. Sessions matching **no** theme
are dropped from the feed.

> Make single-vs-multi assignment a one-line config flag
> `ASSIGN_TO_ALL_MATCHING_THEMES = false` so it's trivial to flip to
> show-in-every-matching-section later.

### 4.1 Matching inputs

Match case-insensitively against the session's `searchText` (title +
description + track names + speaker names + speaker orgs). Use **word-boundary**
regexes to avoid false hits (e.g. `\bgpt\b`, not substring `gpt`).

### 4.2 Speaker org extraction

Parse the org from `speaker.tagLine` (commonly `"Role, Company"` or
`"Role @ Company"` or `"Role at Company"`). Take the substring after the last
`,` / `@` / ` at `, trim it. Org-based matching is the **strongest** signal for
the Anthropic / OpenAI themes — a talk by an Anthropic employee is an Anthropic
talk even if the title doesn't say "Anthropic".

### 4.3 Matcher keyword sets (initial — tune after seeing real data)

| Theme | Matches if ANY of … |
|---|---|
| `anthropic` | org/text contains `anthropic`, `claude`, `\bmcp\b` *(only if also Anthropic-attributed)* |
| `openai` | org/text contains `openai`, `\bgpt\b`, `chatgpt`, `\bo1\b`/`\bo3\b`, `dall-e`, `sora` |
| `coding` | `coding`, `code generation`, `codegen`, `\bswe\b`, `software engineer(ing)? agent`, `copilot`, `developer tools`, `ide`, `pull request`, `code review` |
| `leadership` | `keynote`, `leadership`, `\bcto\b`, `\bvp\b`, `state of`, `strategy`, `org(aniz|anis)`, `hiring`, `team`, `roadmap` |
| `agents` | `agent`, `agentic`, `tool use`/`tool-use`, `function calling`, `\bmcp\b`, `assistant`, `autonomous`, `orchestrat`, `multi-agent` |
| `communities` | `community`, `communities`, `meetup`, `open source`/`open-source`, `\boss\b`, `devrel`, `developer relations`, `ecosystem`, `hackathon` |

Keep the keyword sets in a single editable `THEMES` config object at the top of
the script so they can be tuned in one place after inspecting the real snapshot.

---

## 5. UI / UX (mobile-first)

Design target: a phone held one-handed, possibly on bad conference wifi (hence
offline). Default to **dark theme** (expo halls are dim; saves battery).

### 5.1 Layout

```
┌─────────────────────────────┐
│  AI Engineer WF · My Feed   │  ← sticky header
│  [ All ][Mon][Tue][Wed][Thu]│  ← day filter chips (horizontal scroll)
│  🔎 search…                  │  ← live filter over searchText
├─────────────────────────────┤
│  🟧 Anthropic            (n) │  ← theme section header (collapsible)
│   ┌───────────────────────┐ │
│   │ 10:00–10:30 · Room 2  │ │  ← session card
│   │ Title of the talk     │ │
│   │ Jane Doe · Anthropic  │ │
│   │ #track  #track        │ │
│   └───────────────────────┘ │
│   … more cards …            │
│  🟢 OpenAI               (n) │
│  … sections 3–6 …           │
└─────────────────────────────┘
```

### 5.2 Components & behavior

- **Sticky top bar**: title + day chips + search box. Stays pinned on scroll.
- **Day chips**: `All / Mon / Tue / Wed / Thu` (derive actual days from data).
  Selecting one filters all sections to that day. Horizontal-scroll if narrow.
- **Search box**: live, debounced (~150ms), filters across `searchText` within
  the current day filter. Empty = show all.
- **Theme sections**: rendered in §2 order. Each header shows the theme icon,
  title, and a **count** of currently-visible sessions. Tap header to
  collapse/expand (persist collapsed state in `localStorage`).
- **Session card** (tap to expand inline, or open a detail sheet):
  - Time range + day + room (always visible).
  - Title (prominent).
  - Speakers with org.
  - Track tags as chips.
  - Expanded: full description + **"Add to calendar"** (generate an `.ics`
    blob client-side from start/end/title/room) + a **★ favorite** toggle
    (persist ids in `localStorage`).
- **Empty section**: if a theme has 0 matches under the current filter, show a
  muted "No matching sessions" row (don't hide the section — the ordering is
  the point and the user should see it considered the theme).
- **Favorites**: a `★ My picks` chip at top that filters to favorited sessions
  across all themes.
- **"Now / Next" affordance** (nice-to-have): since the snapshot has real
  times and the conference is live (today is within the event), subtly badge
  sessions happening **now** or **next** relative to `new Date()` (SF time).

### 5.3 Mobile/perf requirements

- Single `index.html`, **everything inlined** (CSS in `<style>`, JS in
  `<script>`, data either inlined or `fetch()`ed from `./data/*.json`).
  Prefer `fetch('./data/sessions.json')` so the data files stay diffable; fall
  back gracefully if opened via `file://` blocks fetch (then inline a build
  step or instruct to serve with `python3 -m http.server`).
- `<meta name="viewport" content="width=device-width, initial-scale=1">`.
- Tap targets ≥ 44px; system font stack; no external fonts/CDNs (offline).
- Responsive: single column ≤640px; max-width ~720px centered on larger screens.
- No layout shift on load; render skeleton or "Loading…" then hydrate.

---

## 6. File structure

```
/
├── index.html          # the entire app (HTML + inlined CSS + inlined JS)
├── data/
│   ├── sessions.json   # committed snapshot
│   └── speakers.json   # committed snapshot
├── scripts/
│   └── refresh-data.sh # the curl commands from §3.1 (run locally)
└── README.md           # how to refresh data + how to open/serve
```

If `file://` fetch restrictions are a problem, the implementer may instead
**inline the JSON** into `index.html` as `<script type="application/json">`
blocks during a tiny refresh step — document whichever path is chosen.

---

## 7. Acceptance criteria

- [ ] Opening the app on a phone-width viewport shows six theme sections in the
      exact order: Anthropic → OpenAI → Coding → Leadership → Agents/Assistants → Communities.
- [ ] Only sessions matching a theme appear; service sessions (breaks/lunch) never appear.
- [ ] Each session shows time, room, speaker(s) + org, and track tags.
- [ ] No session appears in more than one section (with default config).
- [ ] Day chips and search filter all sections live; counts update.
- [ ] Collapse state and favorites persist across reloads (`localStorage`).
- [ ] "Add to calendar" produces a valid `.ics` for a session.
- [ ] Works with **no network** after first load (data is local; no CDNs).
- [ ] Single `index.html` runs with at most `python3 -m http.server` — no build, no npm.
- [ ] Parser is resilient: missing description, missing speaker org, empty
      tracks, or alternate top-level JSON shape (array vs grouped) don't crash it.

---

## 8. Build order (suggested for the next prompt)

1. Add `data/sessions.json` + `data/speakers.json` (real snapshot, fetched
   locally — remember the sandbox can't reach `ai.engineer`). If unavailable at
   implementation time, scaffold with a small representative fixture and wire
   the real file in later.
2. **Inspect the real JSON** and confirm field names / shape against §3.2.
3. Implement load + normalize + category-id resolution (§3.3).
4. Implement `THEMES` config + matcher with priority assignment (§4).
5. Build the static shell + sticky header + day chips + search (§5.1–5.2).
6. Render sections + cards; wire filters, collapse, favorites, `.ics`.
7. Polish mobile styling, dark theme, "now/next" badge.
8. Verify against §7; write `README.md` + `scripts/refresh-data.sh`.

---

## 9. Open questions to resolve during implementation

- Exact `sessions.json` shape (array vs grouped; where tracks live) — **verify against the real file**.
- Whether the snapshot's times are SF-local (assumed) — confirm before computing day/now-next.
- Keyword tuning: after seeing real tracks, adjust §4.3 (the official ~29 track
  names may let you match by track directly instead of keyword guessing).
