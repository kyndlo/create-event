---
name: create-event
description: "Autonomously create Kyndlo events from campaign tasks using gokyn CLI and goplaces for venue research. Usage: /create-event <campaign> [batch-size]"
user-invocable: true
metadata:
  { "openclaw": { "requires": { "bins": ["gokyn", "goplaces", "curl"], "env": ["KYNDLO_API_TOKEN", "GOOGLE_PLACES_API_KEY"] }, "primaryEnv": "KYNDLO_API_TOKEN", "os": ["darwin", "linux"] } }
---

# create-event — Autonomous Event Creator

Create Kyndlo events from campaign tasks end-to-end. Claims tasks, researches venues via Google Places, creates events, uploads photos, and marks tasks complete.

## Invocation

```
/create-event <campaign-name> [batch-size]
```

- **campaign-name** (required): The campaign to pull tasks from (e.g. `colorado-2027`)
- **batch-size** (optional, default 1): Number of tasks to process in this run

When running autonomously, identify yourself as `openclaw-<campaign-name>` (e.g. `openclaw-colorado-2027`).

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
   gokyn task context --campaign <campaign> --json
   ```
   Confirm the campaign exists. Report progress (counties done, tasks remaining) to the user.

3. **Fetch event creation rules:**
   ```bash
   gokyn task rules
   ```
   Read the full markdown output and internalize every rule. These rules govern venue selection, event data formatting, image requirements, and quality standards. **Follow every rule strictly during event creation.** Rules are maintained in the database and may change between runs — always fetch fresh.

4. **Report status** to the user: campaign name, progress percentage, tasks remaining, and confirm rules loaded.

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

## Day-of-Week Parsing

When converting Google Places `weekdayDescriptions` to `--recurrence-days` numbers:

| Day | Number |
|-----|--------|
| Sunday | 0 |
| Monday | 1 |
| Tuesday | 2 |
| Wednesday | 3 |
| Thursday | 4 |
| Friday | 5 |
| Saturday | 6 |

**Rule:** Include a day number ONLY if its weekdayDescription does NOT contain "Closed". Omit closed days entirely.

---

## State Timezone Reference

| State | Timezone |
|-------|----------|
| Colorado | America/Denver |
| Florida | America/New_York |
| New York | America/New_York |

---

## The Autonomous Loop

Repeat for each task up to the batch size. Track progress as `[Task X/N]`.

### Step 1: Claim a task

```bash
gokyn task next --campaign <campaign> --assign --name "openclaw-<campaign>" --json
```

Do NOT pass `--priority` — the server automatically returns tasks in priority order (high first, then medium, then low). This is simpler and more reliable than cascading through priorities manually.

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

### Step 4: Search for venues

```bash
goplaces search "<category> in <county> County, <state>" --json --limit 5 --region US
```

This returns up to 5 venue candidates. If no results, try a broader query:
```bash
goplaces search "<category> near <county>, <state>" --json --limit 5 --region US
```

If still no results, skip the task with reason "No venues found for <category> in <county> County".

### Step 5: Hours gate

For each venue candidate (in order), fetch details:
```bash
goplaces details <place_id> --json
```

Check for `regularOpeningHours.weekdayDescriptions` in the response. A venue **passes** the hours gate if:
- `regularOpeningHours` exists AND
- `weekdayDescriptions` is present and non-empty AND
- At least one day is NOT "Closed"

Use the first venue that passes. If no venue passes the hours gate, skip the task with reason "No venues with published hours for <category> in <county> County".

### Step 6: Gather venue data

From the passing venue's details response, extract:
- **name** — `displayName.text`
- **address** — `formattedAddress`
- **latitude** — `location.latitude`
- **longitude** — `location.longitude`
- **hours** — `regularOpeningHours.weekdayDescriptions`
- **website** — `websiteUri` (use as `--booking-url` if present)
- **price** — `priceLevel` (map: `PRICE_LEVEL_FREE`=0, `PRICE_LEVEL_INEXPENSIVE`=10, `PRICE_LEVEL_MODERATE`=25, `PRICE_LEVEL_EXPENSIVE`=50, `PRICE_LEVEL_VERY_EXPENSIVE`=100, otherwise 0)

### Step 7: Parse open days

Convert `weekdayDescriptions` to recurrence-days numbers using the Day-of-Week Parsing table. For each entry:
1. Read the day name (e.g. "Monday: 9:00 AM – 5:00 PM")
2. If the entry contains "Closed", skip it
3. Otherwise, include the corresponding number

Join the numbers with commas for `--recurrence-days` (e.g. `1,2,3,4,5`).

### Step 8: Create the event

Look up the timezone from the State Timezone Reference table.

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
  --recurrence-days "<parsed days>" \
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

### Step 9: Upload a photo

```bash
goplaces details <place_id> --json --photos
```

Extract the first photo name from `photos[0].name`, then:

```bash
goplaces photo "<photo_name>" --json --max-width 1200
```

Extract the `photoUri` from the response, then download and upload:

```bash
curl -L -o /tmp/venue-photo.jpg "<photoUri>"
gokyn image upload --file /tmp/venue-photo.jpg --event-id <eventId>
```

If any photo step fails, continue without a photo — the event is still valid.

### Step 10: Complete the task

```bash
gokyn task complete <taskId> --event-id <eventId> --venue "<venue name>"
```

### Step 11: Report progress

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
| No search results from goplaces | Try broader query (drop "County", use "near"). If still none, skip task. |
| No venues pass hours gate | Skip task with reason "No venues with published hours". |
| Activity ID not found | Try partial/shorter search terms. If still not found, skip task. |
| Event creation fails | Log the error. Do NOT mark task complete. Continue to next task. |
| Photo download/upload fails | Continue without photo. Still complete the task. |
| gokyn task complete fails | Log the error. Report the event ID so it can be linked manually. |

---

## Summary Report

After all tasks are processed (or the loop ends early), print a summary:

```
=== Create-Event Summary ===
Campaign:   <campaign-name>
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
