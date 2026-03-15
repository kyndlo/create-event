---
name: create-event
description: "Autonomously create Kyndlo events from campaign tasks for a specific city. Researches venues, creates events, uploads photos, and marks tasks complete. Usage: /create-event <campaign> <city>"
user-invocable: true
metadata:
  { "openclaw": { "requires": { "bins": ["gokyn", "curl"], "env": ["KYNDLO_API_TOKEN"] }, "primaryEnv": "KYNDLO_API_TOKEN", "os": ["darwin", "linux"] } }
---

# create-event — Autonomous Event Creator

Create Kyndlo events from campaign tasks end-to-end. Claims tasks, researches venues, creates events, uploads photos, and marks tasks complete.

## Invocation

```
/create-event <campaign-name> <city-name>
```

- **campaign-name** (required): The campaign to pull tasks from (e.g. `colorado-2027`)
- **city-name** (required): The target city scope (e.g. `Denver`, `Orlando`, `Rural`). Only tasks in counties belonging to that city are claimed.

Default behavior: once invoked, keep processing that city sequentially until there are no more eligible tasks, safe progress is blocked, or the user stops the run.

When running autonomously, identify yourself using the agent name only in `--name` (for example `Sugar`), not a campaign-specific combination.

---

## Pre-flight Checks

Before entering the task loop, run these checks in order. Stop if any fail.

1. **Verify token:**
   ```bash
   gokyn whoami --json
   ```
   Confirm the token is valid and has the required permissions.

2. **Get campaign context:**
   ```bash
   gokyn task context --campaign <campaign> --city <city> --json
   ```
   Confirm the campaign exists. Report city-level progress (counties done, tasks remaining) to the user.

3. **List available cities** (optional, useful for discovery):
   ```bash
   gokyn task cities --state <state>
   ```
   Shows all cities in the state with their constituent counties and populations. Use this to decide which city to target.

4. **Fetch event creation rules:**
   ```bash
   gokyn task rules
   ```
   Read the full markdown output and internalize every rule. These rules govern venue selection, event data formatting, image requirements, and quality standards. **Follow every rule strictly during event creation.** Rules are maintained in the database and may change between runs — always fetch fresh.

5. **Report status** to the user: campaign name, city, progress percentage, tasks remaining, and confirm rules loaded.

---

## Category Name Mapping

Task `activityCategory` values sometimes differ from activity titles in the database. Use this table when searching for the activity ID:

| Task activityCategory | Search Term for `gokyn activity list --search` |
|---|---|
| Specialty Niche Museums | Specialty & Niche Museums |
| Yoga Classes | Wellness centers |
| Restaurant/Cafes with Animal Encounters | Coffee with animal encounters |
| Community Theaters | Theater lounges |
| Craft Cafes | Craft cafés |
| Beach/Boardwalks | Beach Boardwalks |
| Food Trucks | Food truck parks |
| Planetariums | Planetarium |

If the task category is not in this table, use it as-is for the search.

---

## Day-of-Week Reference

When setting `--recurrence-days`, use these day numbers:

| Day | Number |
|-----|--------|
| Sunday | 0 |
| Monday | 1 |
| Tuesday | 2 |
| Wednesday | 3 |
| Thursday | 4 |
| Friday | 5 |
| Saturday | 6 |

Include a day ONLY if the venue is open that day. Omit closed days entirely.

---

## State Timezone Reference

| State | Timezone |
|-------|----------|
| Colorado | America/Denver |
| Florida | America/New_York |
| New York | America/New_York |

---

## Venue Research

Each task requires finding a real venue that matches the `activityCategory` in the task's `county` and `state`. You need to collect these data points for each venue:

| Data Point | Required | Used For |
|---|---|---|
| Venue name | yes | `--title`, `--location-place` |
| Street address | yes | `--location-address` |
| Latitude | yes | `--location-lat` |
| Longitude | yes | `--location-lng` |
| Open days (which days of the week) | yes | `--recurrence-days` |
| Website URL | no | `--booking-url` |
| Price estimate | no | `--price` (default 0) |
| Photo URL | no | `gokyn image upload` |

### How to research

Use whatever tools and methods are available to you. The goal is to find a real, currently operating venue. Good approaches include:

- **Google Places** (`goplaces` CLI if available): `goplaces search "<category> in <county> County, <state>" --json --limit 5 --region US`, then `goplaces details <place_id> --json` for hours/address/coordinates.
- **Web search**: Search for `<category> in <county> County, <state>` and visit venue websites or review sites to gather address, hours, and coordinates.
- **Map services**: Use any mapping tool or API to find venues and extract coordinates.
- **Direct knowledge**: If you know a real venue that fits, verify it's still operating and gather the required data.

### Venue selection criteria

1. The venue must be a real, currently operating business or location
2. It must be in (or very near) the correct county
3. It must match the activity category
4. It must have known open days (at least 1 day per week)
5. Prefer venues with published hours, a website, and photos

### Hours gate

A venue is only usable if you can determine which days it is open. If you cannot determine opening days from any source, skip the venue and try the next candidate. If no candidates have determinable hours, skip the task with reason "No venues with published hours for \<category\> in \<county\> County".

Search for **up to 5 candidates** and use the first one that passes the hours gate.

---

## The Autonomous Loop

Repeat sequentially for the requested city until there are no more eligible tasks, safe progress is blocked, or the user asks you to stop. Track progress as `[Task X]`.

**Important:** If the loop exits early for any reason, run the Cleanup step to release any uncompleted tasks.

### Step 1: Claim a task

```bash
gokyn task next --campaign <campaign> --city <city> --assign --name "<agent-name>" --json
```

Use the agent name only for `--name` (for example `Sugar`). Do NOT combine the agent name with the campaign.

Do NOT pass `--priority` — the server automatically returns tasks in priority order (high first, then medium, then low). This is simpler and more reliable than cascading through priorities manually.

Because this skill is city-scoped by default, only tasks whose `cities` array includes that city name should be claimed. This lets agents focus on a metro area (e.g. `Denver`) or all rural counties (`Rural`).

If the response contains `"count": 0` (no tasks), stop the loop. If the response includes a `"diagnostic"` field, report it to the user — it indicates a server-side issue worth investigating.

Extract from the JSON response (`tasks[0]`):
- `taskId` — the `_id` field
- `county` — the county name
- `activityCategory` — the activity type
- `cluster` — the spirit cluster
- `state` — the state name

### Step 2: Map the category

Check the Category Name Mapping table above. If the `activityCategory` matches a left-column entry, use the corresponding right-column search term. Otherwise use the category as-is.

### Step 3: Find the activity ID

```bash
gokyn activity list --search "<mapped-category>" --json
```

Extract the activity `_id` from the first matching result. If no results, try shorter/partial search terms (e.g. "Specialty" instead of "Specialty & Niche Museums"). If still no match, skip the task with reason "Activity not found for category: <category>".

### Step 4: Research a venue

Using the approach described in **Venue Research** above, find a real venue matching the task's `activityCategory` in `county`, `state`.

Collect:
- **name** — the venue's display name
- **address** — full street address
- **latitude** and **longitude** — GPS coordinates
- **open days** — which days of the week the venue is open (as numbers 0-6)
- **website** — venue URL (optional, used as `--booking-url`)
- **price** — estimated cost in USD (optional, default 0)

If no venue can be found after trying multiple candidates, skip the task with reason "No venues found for \<category\> in \<county\> County".

If no venue has determinable open days, skip the task with reason "No venues with published hours for \<category\> in \<county\> County".

### Step 5: Create the event

Look up the timezone from the State Timezone Reference table.

Format the open days as a comma-separated list of day numbers for `--recurrence-days` (e.g. `1,2,3,4,5`).

```bash
gokyn event create \
  --title "<venue name>" \
  --description "<2-3 sentence description of the venue, what visitors can expect, and why it's good for social connection. Follow the rules fetched in pre-flight.>" \
  --activity <activityId>:1 \
  --start-date-time "2027-01-01T09:00:00Z" \
  --end-date-time "2028-01-01T23:59:00Z" \
  --timezone "<timezone>" \
  --location-type physical \
  --location-place "<venue name>" \
  --location-address "<formatted address>" \
  --location-lat <latitude> \
  --location-lng="<longitude>" \
  --recurring \
  --recurrence-frequency weekly \
  --recurrence-interval 1 \
  --recurrence-days "<open days>" \
  --is-public=false \
  --is-premium-only \
  --is-active=false \
  --price <price> \
  --price-currency USD \
  --booking-url "<website or omit flag>" \
  --json
```

**Critical reminders:**
- ALWAYS use `--is-active=false`, `--is-public=false`, and `--is-premium-only` — agents must never activate events
- Negative longitude values MUST use `=` syntax: `--location-lng="-104.99"` (not `--location-lng -104.99`)
- End date is always `2028-01-01T23:59:00Z` (one year window for recurring events)
- Start date is always `2027-01-01T09:00:00Z`

Extract the `eventId` from the JSON response. If event creation fails, log the error, do NOT mark the task complete, and continue to the next task.

### Step 6: Upload a photo

If you obtained a photo URL during venue research, download and upload it:

```bash
curl -L -o /tmp/venue-photo.jpg "<photo-url>"
gokyn image upload --file /tmp/venue-photo.jpg --event-id <eventId>
```

Photo sources (use whichever is available):
- Google Places photos via `goplaces details <place_id> --json --photos` then `goplaces photo "<name>" --json --max-width 1200`
- Venue website images
- Any publicly available photo of the venue

If no photo is available or the upload fails, continue without one — the event is still valid.

### Step 7: Complete the task

```bash
gokyn task complete <taskId> --event-id <eventId> --venue "<venue name>"
```

### Step 8: Report progress

Print a status line:
```
[Task X/N] COMPLETED: <venue name> in <county> County (<cluster>/<category>) — Event ID: <eventId>
```

If the task was skipped, print:
```
[Task X/N] SKIPPED: <county> County / <category> — Reason: <reason>
```

---

## Error Recovery

| Error | Recovery |
|---|---|
| No tasks returned (count: 0) | If `diagnostic` field present, report it to user. Otherwise stop the loop — campaign may be complete. |
| No venues found | Try broader search terms (drop "County", use nearby areas). If still none, skip task. |
| No venues with determinable hours | Skip task with reason "No venues with published hours". |
| Activity ID not found | Try partial/shorter search terms. If still not found, skip task. |
| Event creation fails | Log the error. Release the task: `gokyn task release <taskId>`. Continue to next task. |
| Photo download/upload fails | Continue without photo. Still complete the task. |
| gokyn task complete fails | Log the error. Release the task: `gokyn task release <taskId>`. Report the event ID so it can be linked manually. |
| Agent interrupted or unexpected error | Any claimed task not yet completed must be released. See Cleanup section. |

---

## Cleanup

Before exiting — whether normally after the loop or due to an error — release any task that was claimed but not completed or skipped. This prevents tasks from being stuck in `in_progress` indefinitely.

For each claimed task that was NOT completed or skipped:
```bash
gokyn task release <taskId>
```

**Rule:** A task must NEVER be left in `in_progress` status when the agent exits. Every claimed task must end in one of three states:
- **completed** — event was created successfully
- **skipped** — no viable venue or unrecoverable error (task won't be retried)
- **released** (back to pending) — temporary failure, another agent can retry later

Use `skip` for permanent failures (no venues exist, category not found). Use `release` for transient failures (API timeout, event creation error, network issue).

---

## Summary Report

After all tasks are processed (or the loop ends early), print a summary:

```
=== Create-Event Summary ===
Campaign:   <campaign-name>
City:       <city-name>
Processed:  <total> tasks
Created:    <count>
Skipped:    <count>
Failed:     <count>

Created Events:
  - <venue name> in <county> (<category>) — Event ID: <id>
  - ...

Skipped Tasks:
  - <county> / <category> — <reason>
  - ...

Failed Tasks:
  - <county> / <category> — <error>
  - ...
```
