# AI Engineers World's Fair — Interview Prep Grill

**Topic:** Building a mobile-friendly site using the World's Fair open data API  
**Focus areas:** Anthropic, OpenAI coding capabilities, leadership, Claude & personal assistants, communities

---

## Round 1: API & Data Fundamentals

**Q1.** The World's Fair open data API exposes sessions, speakers, and tracks. What's your first API call when the page loads, and how do you structure the response to support filtering by track priority (Anthropic first, then OpenAI, etc.)?

**Q2.** The API returns 200+ sessions. What do you render on first load vs. lazy-load? How do you cache data for a mobile user on spotty conference WiFi?

**Q3.** Open data APIs often have unstable schemas — fields get renamed or removed between updates. How do you defensively parse the JSON so your UI doesn't silently break?

**Q4.** The API likely doesn't have a "communities" endpoint — Discord links, GitHub orgs, and Slack groups are semi-structured or scraped. How do you blend that alongside clean API data in your UI?

---

## Round 2: Mobile-Friendly Site Architecture

**Q5.** What's your CSS approach for a conference schedule on a 375px screen — framework (Tailwind, Bootstrap) or vanilla? How does your schedule grid adapt from mobile to tablet?

**Q6.** Attendees lose signal mid-conference. What's your offline strategy — service worker, localStorage, or IndexedDB? What data do you prioritise to persist?

**Q7.** Sessions are in SF (PT) but attendees' phones may be set to their home timezone. How do you handle time display, and where does the conversion happen — server, client, or both?

**Q8.** SPA vs. server-rendered — which do you pick for a 2-day conference companion app, and why? What's the tradeoff you're explicitly accepting?

---

## Round 3: Content & Prioritisation

**Q9.** You want Anthropic model lab updates to appear first. How do you implement user-configurable track priority — localStorage preference, URL param (`?priority=anthropic,openai`), or something else? What happens on first visit with no preference set?

**Q10.** For "Claude & personal assistants" content, you want to embed a live Claude demo on the page. What API call do you make, what model do you choose for a public-facing low-cost demo, and how do you prevent abuse (prompt injection, runaway costs)?

**Q11.** "Leadership" is a track at the fair. Sessions from OpenAI and Anthropic leadership may cover conflicting narratives. How do you present both neutrally in your UI without editorialising?

**Q12.** You're pulling model lab updates for Anthropic and OpenAI. Name two concrete things announced or released by each in the last 6 months that you'd highlight as a card on your site.

---

## Round 4: Architecture Deep-Dive

**Q13.** Walk me through your component tree for the sessions list page: what are the top-level components, what state lives where, and what triggers a re-render?

**Q14.** You want the site live before the fair starts. What's your deploy target — Vercel, Netlify, GitHub Pages, CF Pages — and why? What's your CI/CD pipeline from push to live?

**Q15.** A journalist screenshots your site and posts it. The World's Fair open data has a licence. What do you check before going live, and where do you surface attribution in the UI?

---

## Rapid Fire (30 seconds each)

- What HTTP status code means the API is rate-limiting you, and what do you do?
- What's the difference between `preload`, `prefetch`, and `preconnect` on mobile?
- You get a CORS error hitting the API from localhost. Name two ways to fix it.
- What Anthropic model would you use for a summarisation feature on session descriptions, and why not a larger one?
- One sentence: what is Claude's constitutional AI approach, and why does it matter to an enterprise buyer?

---

## How to Use This File

Work through each round out loud or in writing. For each answer:
1. State your decision
2. Name the tradeoff you're accepting
3. Give a one-line example (endpoint, component name, code snippet)

A strong answer takes 60–90 seconds. A great answer ends with "and the thing I'd watch out for is..."
