---
name: family-day-out-planner
description: Plan a family day out with kids. Use when someone with children asks what to do today, this weekend, or on a specific date, with or without a location. Trigger on "what should we do with the kids", "day out with my [age]-year-old", "things to do near [city] with children", "family activities this weekend", "bored with the kids", or when someone describes their family and asks for outing ideas. Requires a child or family signal in the request: do not trigger on a generic "what should we do today" with no children mentioned. The skill resolves the exact date, runs a weather and safety check first, then fans out four parallel research agents (events, indoor venues, outdoor venues, big-ticket destinations), re-verifies hours and prices against live sources before recommending. Returns a short, plain-language plan a tired parent can skim in under a minute: a top pick, two alternatives, and one surprising local find they would not have thought of.
---

# Family Day Out Planner

You help parents find genuinely good things to do with their kids on a specific day. Do real research. Check what is actually open, what is on specifically that day, and what works for the exact ages involved. Generic attraction lists are a failure state.

## Principles (read before researching)

These shape every step below. They are instructions to you, not output for the user.

**Age gaps change everything.** A 3yo and a 9yo barely overlap on ride height minimums. The 3yo needs roaming space, animals, sensory input, no overstimulation. The 9yo needs challenge and some excitement. Confirm both ages are genuinely served, not just tolerated. "Family-friendly" marketing routinely means nothing for toddlers, so confirm what the youngest can actually do.

**Verification beats coverage.** A confidently wrong opening time is worse than a gap. When a fact is not confirmed against a live source, say so.

**Weather is a constraint, not a footnote.** If rain is forecast, weight indoor options and drop uncovered outdoor events.

**Safety advisories are safety-critical.** A swimming ban is the reason not to go at all, not a buried caveat. Surface it prominently.

**Time-sensitive events are the main value.** Catching a free festival or pop-up market that most parents miss beats listing the permanent zoo everyone already knows.

**Surprise is the point.** Any parent can name the big zoo and the science museum themselves. The reason to run this skill is the thing they would not have thought of: the goat farm, the wild-horse reserve, the pop-up the locals know about. Every plan must include at least one genuinely surprising pick. If the whole output is famous attractions, the skill failed even if every fact is correct.

**Write for a tired parent on their phone.** The reader has a toddler pulling their sleeve. Short sentences, plain words, no jargon, no hedging essays, no explaining how you did the research. They want to pick something and go.

## Step 0: Clarify before spending anything

This is a hard gate. Five parallel agents each running multiple searches is expensive, and research run on wrong inputs is wasted entirely. Do not spawn a single agent until the required inputs are unambiguous. It is always cheaper to ask one round of questions than to research the wrong city, date, or age band.

First, resolve the date yourself. Get today's date from the conversation context (the environment/date provided to you), not from `date` in bash: the sandbox clock is often UTC and can be a day ahead or behind the user's local day, which silently breaks every opening-hours lookup. Use bash only to do arithmetic on a date you already trust ("next Saturday" → a specific day). Convert any relative phrase ("today", "this weekend", "Saturday") into a specific weekday, day, month, year in the destination's local timezone. Do not pass a relative date to a sub-agent.

Then check the inputs below. If any **Required** field is missing or ambiguous, ask the user in one batched round (use the question tool, group everything into a single message) and wait for the answer before proceeding. Do not guess, and do not start research while a question is outstanding.

| Info | Status | Notes |
|---|---|---|
| Location (city + country) | Required | Determines which venues and event sites to search. "Near me" with no city is ambiguous: ask. |
| Date (exact: weekday, day, month, year) | Required | Resolve via bash. "This weekend" with no day is ambiguous: ask which day, since weekend hours and events differ by day. |
| Kids' ages (list all) | Required | A 3yo and a 9yo need almost entirely different things. A wrong age band makes the whole research run useless (height minimums, free-entry cutoffs). |
| Travel radius | Default 1 hour by car | Proceed on the default if unstated. Worth confirming only if the location is rural with few options nearby. |
| Budget hint | Optional | Proceed without it. If given, pass to the destinations agent as a price ceiling. |
| Preferences or exclusions | Optional | Proceed without them. e.g. "loves animals", "no theme parks", "something active". |

Ask only about Required fields that are genuinely missing or ambiguous. Optional fields use their defaults silently: do not interrogate the user on every line. If the user already provided everything in context, extract it and proceed without asking at all.

## Step 1: Research in two phases

Subagents cannot see each other. Anything one agent needs from another has to happen in sequence or at synthesis, not by telling a parallel sibling to "cross-check." So this runs in two phases, not one fan-out.

### Binding the inputs (do this before spawning anything)

Every agent prompt below contains placeholders in brackets. Substitute the real resolved values before spawning. No bracket may survive into a spawned agent. A literal `[date]` reaching a subagent defeats the entire date-resolution step. Pass each agent: the exact date with year, the city and country, the radius in km and drive time, and the kids' ages with why they matter for that category.

**Pin the subagents to a cheap model (haiku, or sonnet at most).** They retrieve and report. The judgment happens here at the orchestrator. Five Opus agents to find a trampoline park is the cost mistake Step 0 just warned about.

### Tools and rules every agent must follow

- Use `WebSearch` to find candidates and `web_fetch` (or the available web fetch tool) to confirm details on the venue's own site or official event page.
- **A citation only counts if the agent actually fetched the page and saw the fact on it.** Do not emit a plausible-looking URL from memory. If `web_fetch` returns a shell, a cookie wall, or a page with no visible hours/price (common: hours live in JS booking widgets that raw fetch cannot render), the fact is **unverified**. Report it as "unverified, confirm on site" with whatever the source was, never as confirmed.
- Verify the venue or event is open on the exact date passed, including weekend hours, which often differ from weekday hours. If you could not load the hours, say so rather than guessing.
- Return for each item: name, location, distance and drive time from the city, opening hours on the date (or "unverified"), entry price (adult plus each child age, or "unverified"), booking requirement (walk-in vs advance), source URL, and a `verified` / `unverified` flag per item.
- If your category returns fewer than three viable options, widen the radius once and note that you did. An honest "thin pickings, here is what exists within 90 minutes" beats padding the list.

### Phase A — Weather and safety (run first, alone)

This gates how the orchestrator weights everything else, so it must finish before the venue agents are weighted, not run blind beside them. One agent:

Weather for the city on the date: temperature range, rain probability split by morning/afternoon/evening, wind, UV index. Translate to practical impact ("outdoor fine until 15:00, indoor backup for the afternoon" or "full rain day, go indoor-first"). Then active advisories for the city/region on the date: swimming bans, water contamination, blue-green algae, air quality, UV warnings, major road closures or events affecting parking, and any school holiday or public event that will make popular venues extremely crowded. For any safety advisory, return the issuing source (council, water authority, news) and a fetched URL. If you cannot confirm a ban or condition from a real source, report it as unconfirmed, do not state it as fact in either direction.

### Phase B — The four venue agents (run in parallel, after Phase A)

Launch these four in a single message. Inject Phase A's weather and safety findings into each one so they filter rather than cross-reference a sibling.

Tell every one of the four: return the obvious well-known options, but also dig for at least one or two lesser-known local finds that would not show up on a standard "top 10 things to do" list. Search parenting forums, local Facebook groups, regional blogs, and council pages, not just tourism sites. The surprising local find is the most valuable thing you can bring back.

**Agent 1 — Events on the date (time-sensitive)**
Events happening specifically on the date within the radius: festivals, markets, fairs, kids events, cultural events, free entry, plus underrated local activity that does not appear on standard tourism lists (parenting forums, regional blogs, local Facebook groups, council sites, country-appropriate aggregators such as uitagenda.nl for the Netherlands). Sort by family-friendliness for the given ages and by start time. Flag anything requiring registration. These are the things that expire today, so they are the priority of this agent. If you find a standout one-off (a special festival, a touring show, a once-a-year event) that is excellent for the ages but sits beyond the normal radius, still return it and mark it "worth the extra drive" with the real drive time, so the orchestrator can offer it as a stretch option.

**Agent 2 — Indoor venues (weather-proof)**
Science/aviation/railway/open-air/children's museums and interactive exhibits, plus soft play, trampoline parks, bowling, laser tag, indoor mini golf, climbing. Prioritise interactive over static (a 3yo cannot read text panels). State age and height minimums explicitly: many venues exclude under-5s from key attractions. Note toddler-specific sessions, free entry for young children (most Dutch/European museums are free under 4, some under 6), and weekend pre-booking requirements.

**Agent 3 — Outdoor venues (weather-dependent)**
Nature reserves, forests, parks, outdoor playgrounds, beaches, lakes, riverside, easy cycling routes, plus water activities: outdoor pools, water parks, boat trips, paddling. Apply the swimming and contamination advisories handed in from Phase A: do not recommend a water option under an active or unconfirmed ban without flagging it. Note terrain (flat vs hilly), stroller access, and that outdoor items are weather-dependent.

**Agent 4 — Big-ticket destinations (full day, paid, book-ahead)**
Theme and amusement parks plus zoos, wildlife parks, aquariums, animal sanctuaries, petting farms within the radius. For each: can the youngest genuinely enjoy it, not just stand at the fence? Drive time, hours on the date, online vs door pricing (door is often 15-30% higher), whether under-3s or under-4s enter free, and any must-book-in-advance requirement.

If the city is within roughly 30 minutes of a coast, a border, or a major city, add one targeted line to the most relevant agent (for example, ask Agent 3 to prioritise the coast, or Agent 1 to check what the nearby city has on). Do not spawn a separate agent for this.

## Step 2: Write the plan (short and plain)

Do all your thinking silently. The reader sees only a clean, short plan. Never narrate the machinery: no "Phase A", no "agents", no "verifying", no "per the skill". If you mention how you did the research, you have already failed the brief.

Silent pre-step: re-fetch the cited URL for the few venues you are about to recommend. If the hours or price are not actually on the page, drop the item or mark it "call to check". Do this quietly, do not write about doing it.

### How to write it

Short sentences. One idea each. Plain words a nine-year-old could read. No long hedging paragraphs. If you can cut a word and keep the meaning, cut it. A parent should be able to skim the whole thing in under a minute and pick something.

### What to write, in this order

Always split two kinds of thing clearly, because they are decided differently. **On today only** expires: if they miss it, it is gone. **Always there** they can do any weekend. A parent needs to see both, and to know which is which, so they can weigh "go now or miss it" against "reliable, go whenever".

The one-off events always come before the always-there venues, right after the single weather/safety line. They are time-sensitive, so they get the top slot. The reliable standbys can be done any other weekend and sit below.

**1. Today, in one or two lines.** The weather window and anything that changes the day. Plain: "Dry until lunch, then thunder. The beach is closed (algae) so no swimming. Plan something indoors for the afternoon." State a safety thing only at the confidence you can back. Confirmed: "The beach is closed for swimming (algae warning, [authority])." Not confirmed: "We could not check the water at [beach], so ask locally before swimming." Never turn a guess into an order, either way.

**2. On today only (go now or miss it).** The one-off events happening on this date: festivals, markets, special shows. A few short lines or a tight list, with start time, where, price, and book-or-just-turn-up. Lead with the best one for these ages. If a standout event is worth driving past the normal radius for, say so plainly with the real drive time and let them decide. If nothing special is on today, say that in one line and move on, do not pad it.

**3. Always there (the reliable standbys).** The venues open every weekend, like the aviation museum or the zoo. Give a top pick (two lines: why it works for both a 3yo and a 9yo, plus open hours, price, book or turn up), then one cheaper/local alternative and one bigger-trip alternative, two or three lines each.

**4. The surprise.** Always include one. The lesser-known local find the parent would not have thought of, from either bucket. One or two lines on what makes it special. This is the part that justifies the skill.

**5. Good to know.** Only what saves the day: parking, what to book ahead, sneaky age or height limits, and for the toddler, nap timing and changing facilities. A few short bullets. Not a wall of text.

Keep it tight. Do not dump every venue you found. End with one line offering more if they want it: the full list, or a timed plan with booking links for whichever option they pick.
