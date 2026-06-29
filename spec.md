# Spec: AI Engineer World's Fair — Mobile Companion Site

**Status:** Draft for implementation
**Target:** A mobile-first web app that surfaces the World's Fair schedule from the
official open data API, prioritised around the topics I care about.
**Event:** AI Engineer World's Fair 2026, Jun 29 – Jul 2, Moscone West, San Francisco.

---

## 1. Goal

A fast, offline-tolerant, single-page site I can use on my phone on conference WiFi.
It pulls the open data, then ranks and groups sessions so my priority topics float to
the top. No login, no backend of my own — static hosting + client-side fetch.

### Priority topics (in order)
1. **Anthropic / model lab updates** (Anthropic first)
2. **OpenAI — coding capabilities**
3. **Leadership**
4. **Claude & personal assistants** (agents/assistants built on Claude; "Claws" read as Claude — confirm if I meant something else)
5. **Communities**

This ordering is the default sort key for the whole app (see §6).

---

## 2. Data sources (open data API)

Base: `https://www.ai.engineer/worldsfair/`

| Resource | URL | Use |
|---|---|---|
| Sessions | `https://www.ai.engineer/worldsfair/sessions.json` | Primary feed — talks, times, tracks, room, speaker refs |
| Speakers | `https://www.ai.engineer/worldsfair/speakers.json` | Names, bios, company, avatar, social |
| MCP server | `https://www.ai.engineer/worldsfair/mcp` | Agent tools: `get_conference_info`, `list_speakers`, `list_sessions`, `get_schedule` — optional, for a later "ask the schedule" feature |
| iCal | `https://www.ai.engineer/worldsfair/calendar.ics` | "Add to calendar" deep links |
| LLM overview | `https://www.ai.engineer/worldsfair/llms.md` / `llms-full.md` | Reference only |

**Note:** These hosts may be CORS-restricted or Cloudflare-gated from a browser
(`localhost` hit a 403/CONNECT block during research). The build step must handle this —
see §5 and §11 Open Questions.

---

## 3. Tech stack (defaults — change in implementation prompt if preferred)

- **Framework:** Vanilla TS + Vite, or React + Vite. Default to **React + Vite** for the
  component tree in §8. No SSR — it's a 4-day companion app, SPA is the right tradeoff.
- **Styling:** Tailwind CSS, mobile-first breakpoints (`sm` 640 / `md` 768). Design target
  is a **375px** viewport first.
- **State:** React Query (TanStack Query) for fetch + cache + retry, or plain `useState` +
  a tiny fetch wrapper if we stay vanilla.
- **Offline:** Service worker (Workbox or hand-rolled) + IndexedDB for the session/speaker
  payload. `localStorage` only for small UI prefs.
- **Deploy:** Static host — **Cloudflare Pages** or **GitHub Pages**. Push-to-deploy.

---

## 4. Data ingestion strategy

Two options; pick one in the implementation prompt:

**A. Build-time snapshot (recommended for reliability).**
A prebuild script fetches `sessions.json` + `speakers.json`, normalises them (§7), and
writes `public/data/sessions.json` + `speakers.json` into the bundle. Site loads instantly
and works fully offline from first paint. Re-run the build to refresh. Sidesteps CORS.

**B. Runtime fetch.**
Fetch the live endpoints on load, cache in IndexedDB, fall back to cache on failure.
Freshest data, but exposed to CORS / rate-limit / outage risk on conference WiFi.

Default: **A**, with B as a "refresh" button that tries the live endpoint and silently
keeps the snapshot if it fails.

---

## 5. Defensive parsing (the schema is not a contract)

Open data fields get renamed/removed between updates. Every field read must be guarded:

- Normalise into our own internal model (§7) at the ingestion boundary — the UI never
  touches raw API JSON.
- Treat every field as optional. Missing title → skip card; missing time → "Time TBA";
  missing track → "Untracked" bucket.
- Validate with a schema (Zod) at the boundary; log+drop malformed records rather than
  crashing the list.
- Speaker references in a session may be IDs *or* embedded objects — handle both.

---

## 6. Prioritisation & ranking (the core feature)

The app's signature behaviour: my priority topics bubble up.

### Topic model
Define topic groups as keyword/track matchers, each with a rank weight (lower = higher):

```
1  Anthropic        → track/title/speaker-company matches: "anthropic", "claude"
2  OpenAI coding    → "openai" AND ("code"|"coding"|"codex"|"swe"|"engineer")
3  Leadership       → track "Leadership" / titles with "leader", "scaling teams", "VP", "exec"
4  Claude+assistants→ "claude", "assistant", "agent", "personal assistant", "copilot"
5  Communities      → "community", "communities", "meetup", "open source", "ecosystem"
```

### Sort key
`(topicRank, startTime, title)`. Sessions matching no priority topic sort after all
priority ones, by time. A session matching multiple topics takes its **best** (lowest) rank.

### User-configurable order
- Default order = the list above.
- Persist a custom order in `localStorage` (`priorityOrder`) and accept a URL param
  `?priority=anthropic,openai-coding,leadership,...` for shareable/deep-linkable config.
- First visit, no preference → use the default order.

### UI affordances
- A reorderable chip list (drag or up/down) for priorities at the top of the schedule.
- A "⭐ My priorities" toggle to filter down to only matching sessions.
- Each card shows a colored topic tag so ranking is legible.

---

## 7. Internal data model

```ts
type Session = {
  id: string;
  title: string;
  description?: string;
  track?: string;
  startsAt?: string;      // ISO, source tz = America/Los_Angeles
  endsAt?: string;
  room?: string;
  speakers: SpeakerRef[]; // resolved to Speaker at render
  topicRank: number;      // computed (§6); 99 = no priority match
  topics: string[];       // matched topic ids
};

type Speaker = {
  id: string;
  name: string;
  company?: string;
  bio?: string;
  avatarUrl?: string;
  socials?: { type: string; url: string }[];
};
```

---

## 8. Component tree (React default)

```
<App>
  <Header/>                     // title, refresh button, online/offline badge
  <PriorityBar/>                // reorderable topic chips (state: priorityOrder)
  <Filters/>                    // day tabs, track filter, search, "My priorities" toggle
  <SessionList>                 // virtualised; consumes sorted+filtered sessions
    <DayGroup>
      <SessionCard/>            // time, title, topic tags, room, speaker avatars
        → <SessionDetail/>      // sheet/modal: full desc, speakers, add-to-calendar
    </DayGroup>
  </SessionList>
  <OfflineBanner/>
</App>
```

- **State location:** `priorityOrder`, active day, filters, search live in `<App>` (or a
  small store). Session data comes from React Query / IndexedDB cache.
- **Re-render triggers:** changing priority order, day, filter, or search recomputes the
  derived sorted list (memoised on those inputs).

---

## 9. Mobile UX requirements

- 375px-first; single column of cards. Tablet (`md`) → optional 2-column day grid.
- Sticky day tabs + priority bar; thumb-reachable filter controls.
- Tap target ≥ 44px. No horizontal scroll.
- Detail opens as a bottom sheet, not a route push (faster, keeps scroll position).
- Skeleton loaders on first paint; never a blank screen.

### Timezone
- Source times are **PT (America/Los_Angeles)**.
- Display in PT by default with a clear "PT" label; offer a toggle to "my device timezone".
- Conversion happens **client-side** with `Intl.DateTimeFormat` / `timeZone` option. Store
  ISO + source tz; never pre-format on the server.

---

## 10. Offline strategy

- Service worker precaches the app shell + the snapshot data (§4A).
- IndexedDB holds sessions + speakers; reads come from there, so the schedule works with
  zero connectivity.
- `localStorage`: only `priorityOrder`, tz preference, and starred session IDs.
- Online/offline badge in header; "refresh" tries live fetch, falls back silently.
- Prioritise persisting: today's + tomorrow's sessions, all priority-topic sessions,
  starred sessions.

---

## 11. Out of scope (v1)

- Live Claude "ask the schedule" demo via the MCP server — nice v2, needs an API key and
  abuse/cost controls; keep keys out of the static bundle.
- Auth / personal accounts.
- Push notifications for session start.
- A real "communities" dataset — the API has no communities endpoint, so v1 derives the
  Communities topic from keyword matching only (see §11 Open Questions).

---

## 12. Implementation milestones

1. Scaffold (Vite + React + Tailwind + TS), 375px layout shell.
2. Ingestion script (§4A) + Zod normaliser (§5) → internal model (§7).
3. Session list + day tabs + search + card/detail.
4. Prioritisation engine (§6) + reorderable PriorityBar + URL/localStorage persistence.
5. Timezone toggle + add-to-calendar (iCal deep link).
6. Service worker + IndexedDB offline (§10).
7. Deploy to Cloudflare/GitHub Pages; verify on a real phone.

---

## 13. Open questions (answer before/while implementing)

1. **"Claws"** — confirm this means **Claude** (assistants/agents). If it's a specific
   track or product, tell me and I'll remap topic #4.
2. Vanilla TS or React? Spec defaults to React.
3. Build-time snapshot (A) or live runtime fetch (B)? Spec defaults to A.
4. Is `sessions.json` actually CORS-open from a browser? Confirm during step 2; if not,
   commit to the snapshot approach.
5. Licence/attribution — check the open data licence and surface attribution + a link back
   to ai.engineer in the footer before going live.

---

## Sources
- [Schedule · AI Engineer World's Fair 2026](https://www.ai.engineer/worldsfair/schedule)
- [AI Engineer World's Fair 2026 (event page)](https://www.ai.engineer/worldsfair/2026)
- [llms.md overview](https://www.ai.engineer/worldsfair/2026/llms.md)
- [AI Engineer World's Fair (home)](https://www.ai.engineer/)
