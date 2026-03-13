# create-event

An [OpenClaw](https://github.com/anthropics/openclaw) skill that autonomously creates [Kyndlo](https://kyndlo.com) events from campaign tasks. It claims tasks, researches venues via Google Places, creates events, uploads photos, and marks tasks complete — all without human intervention.

## How it works

```
/create-event colorado-2027 5
```

The agent picks up where you left off:

1. **Claims** the next pending task from the campaign (high priority first, then medium, then low)
2. **Searches** Google Places for matching venues in the task's county
3. **Validates** that the venue has published operating hours (the "hours gate")
4. **Creates** the event on the Kyndlo platform with full location, recurrence, and pricing data
5. **Uploads** a venue photo from Google Places
6. **Completes** the task and moves to the next one

Repeat up to `batch-size` times per invocation.

## Prerequisites

### CLI tools

| Tool | Purpose | Install |
|------|---------|---------|
| [gokyn](https://github.com/kyndlo/gokyn-cli) | Kyndlo platform CLI (events, tasks, activities) | `cd apps/gokyn-cli && make install` |
| [goplaces](https://github.com/kyndlo/goplaces) | Google Places API CLI (venue search & details) | `go install github.com/kyndlo/goplaces@latest` |
| curl | Download venue photos | Pre-installed on macOS/Linux |

### Environment variables

```bash
export KYNDLO_API_TOKEN="kyndlo_..."        # Required — Kyndlo API token
export GOOGLE_PLACES_API_KEY="AIza..."      # Required — Google Places API key
```

Verify setup:
```bash
gokyn whoami
goplaces search "coffee in Denver, Colorado" --limit 1
```

## Usage

```
/create-event <campaign-name> [batch-size]
```

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `campaign-name` | Yes | — | Campaign identifier (e.g. `colorado-2027`, `florida-2027`) |
| `batch-size` | No | 1 | Number of tasks to process in this run |

### Examples

```bash
# Process one task from the Colorado campaign
/create-event colorado-2027

# Process 10 tasks
/create-event colorado-2027 10

# Work on Florida
/create-event florida-2027 5
```

## Workflow overview

```
Pre-flight                          The Loop (per task)
─────────────                       ───────────────────
 1. Verify API token                 1. Claim next task
 2. Check campaign progress          2. Map activity category
 3. Fetch creation rules             3. Find activity ID
 4. Report status                    4. Search venues (5 candidates)
                                     5. Hours gate check
                                     6. Extract venue data
                                     7. Parse open days → recurrence
                                     8. Create event (always inactive)
                                     9. Upload venue photo
                                    10. Complete task
                                    11. Report progress
```

## Safety guardrails

Events created by agents are **always** inactive and private:

```
--is-active=false    # Never goes live automatically
--is-public=false    # Not visible to users
--is-premium-only    # Restricted access
```

A human must review and activate each event before it becomes visible to users.

## Dynamic rules

Event creation rules are **not** embedded in the skill. The agent fetches them fresh at the start of every run:

```bash
gokyn task rules
```

This means rules can be updated in the database without modifying the skill file. The agent always operates with the latest guidelines for venue selection, description writing, pricing, and quality standards.

## Priority cascade

Tasks are organized into three priority tiers based on county population:

| Priority | Meaning |
|----------|---------|
| `high` | High-population counties — process first |
| `medium` | Mid-population counties |
| `low` | Low-population/rural counties |

The agent tries high-priority tasks first. If none remain, it cascades to medium, then low. If no tasks exist at any priority, the loop stops.

## Category mapping

Some task categories don't exactly match activity names in the database. The skill includes a mapping table that handles these 8 known mismatches automatically:

| Task Category | Maps To |
|---|---|
| Specialty Niche Museums | Specialty & Niche Museums |
| Yoga Classes | Wellness centers |
| Restaurant/Cafes with Animal Encounters | Coffee with animal encounters |
| Community Theaters | Theater lounges |
| Craft Cafes | Craft cafes |
| Beach/Boardwalks | Beach Boardwalks |
| Food Trucks | Food truck parks |
| Planetariums | Planetarium |

## Error handling

The skill is designed to keep going when individual steps fail:

| Failure | What happens |
|---------|-------------|
| No venues found | Tries broader search query, then skips task |
| No venues with hours | Skips task with reason logged |
| Activity ID not found | Tries partial search, then skips task |
| Event creation fails | Logs error, does NOT complete task, moves on |
| Photo upload fails | Completes task without photo |

At the end of a run, the agent prints a summary showing created events, skipped tasks, and failures.

## Supported campaigns

| Campaign | State | Timezone |
|----------|-------|----------|
| `colorado-2027` | Colorado | America/Denver |
| `florida-2027` | Florida | America/New_York |
| `newyork-2027` | New York | America/New_York |

New campaigns can be created with `gokyn task seed`. See the [gokyn CLI docs](https://github.com/kyndlo/gokyn-cli) for details.

## File structure

```
create-event/
  SKILL.md    # The skill definition — all agent instructions
  README.md   # This file
```

## Related

- [gokyn-cli](https://github.com/kyndlo/gokyn-cli) — Kyndlo platform CLI
- [goplaces](https://github.com/kyndlo/goplaces) — Google Places API CLI
- [OpenClaw](https://github.com/anthropics/openclaw) — Skill framework for AI agents

## License

Private — Kyndlo Inc.
