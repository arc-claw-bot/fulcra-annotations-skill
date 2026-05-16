# Fulcra Annotations Skill

Create Fulcra annotation definitions and record annotation events from an agent workflow.

This repo contains an OpenClaw/Codex-compatible skill plus a small Python helper script for the Fulcra Life API. It is intended for bots that need to log user-approved events such as moments, boolean values, numeric values, or scale ratings.

## What It Does

- Lists existing Fulcra annotation definitions.
- Creates annotation definitions, including definition-level tags.
- Records annotation events, including historical timestamps.
- Supports moment, boolean, numeric, and scale annotations.
- Supports record-level tags for individual logged events.
- Verifies writes by reading the event back after ingest.

## Repository Layout

- `SKILL.md` - agent instructions and operating rules.
- `scripts/fulcra_annotations.py` - CLI helper used by the skill.
- `references/api-notes.md` - Fulcra API endpoint notes and caveats.
- `agents/openai.yaml` - skill metadata for OpenAI/Codex-style agents.

## Requirements

- Python 3.11 or newer.
- Fulcra API access for the target user.
- One of these auth paths:
  - `FULCRA_ACCESS_TOKEN` in the environment.
  - `fulcra-api auth print-access-token` available on `PATH`.
  - Fulcra beta CLI via `uv tool run 'git+https://git@github.com/fulcradynamics/fulcra-api-python.git@add-cli'`.

If credentials live outside the process home, set `FULCRA_HOME` to the home directory that contains the Fulcra CLI credentials.

## Quick Start

From an agent workspace that has this skill installed:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py list
```

Create a reusable moment annotation definition:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py create \
  --type moment \
  --name "Asked Agent to Do Something New" \
  --description "Logged when the user asks the agent to try a new category of work" \
  --tag agent \
  --tag new-task
```

Record a moment now:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py record \
  --name "Asked Agent to Do Something New" \
  --note "User asked for a new annotation workflow" \
  --tag new-task
```

Record a historical moment:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py record \
  --name "Asked Agent to Do Something New" \
  --recorded-at "2026-05-15T10:00:00-04:00" \
  --note "Backfilled from user request" \
  --tag backfill
```

The script returns JSON. Treat the write as confirmed only when `verified_matches` is at least `1`.

## Annotation Definitions vs Records

Fulcra uses two related concepts:

- **Annotation definition**: the reusable annotation itself, like a button or metric named `Focus`.
- **Annotation record**: one logged occurrence or value for that definition.

Create definitions once. Record events each time something happens.

Use `--id <annotation-id>` when multiple annotations share a name or when automation needs a stable target.

## Tags

Tags can apply to either the definition or an individual record.

Use **definition tags** for stable categories that describe the annotation itself:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py create \
  --type moment \
  --name "Medication Taken" \
  --tag health \
  --tag adherence
```

Use **record tags** for context that only applies to one event:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py record \
  --name "Medication Taken" \
  --note "Backfilled after travel" \
  --tag backfill \
  --tag travel
```

Tag behavior:

- Repeat `--tag` to send multiple tags.
- `--tag` accepts a Fulcra tag name or an existing tag UUID.
- Tag names are resolved to Fulcra tag IDs before sending.
- Missing tag names are created automatically.
- If `record --tag` is omitted, the record inherits the definition tags.
- If `record --tag` is present, those explicit record tags are used for that record.

Use short, reusable, lowercase tags such as `health`, `agent`, `research`, `new-task`, `manual-test`, or `backfill`. Do not put timestamps, private notes, or one-off sentences in tags.

## Historical Timestamps

Use `--recorded-at` for any event that did not happen right now.

Always pass a full ISO-8601 timestamp with timezone:

```bash
--recorded-at "2026-05-15T10:00:00-04:00"
```

If a user says "yesterday at 10am", resolve it in that user's timezone before writing. Fulcra readback may return the equivalent UTC instant, such as `2026-05-15T14:00:00+00:00`.

## Scale, Numeric, and Boolean Values

Create a scale definition:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py create \
  --type scale \
  --name "Focus" \
  --description "How focused do I feel right now?" \
  --scale-labels "1=Scattered,2=Low,3=Neutral,4=Focused,5=Locked In" \
  --default-value 3
```

Record a scale value:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py record \
  --name "Focus" \
  --value 4 \
  --note "Deep work block started well"
```

Boolean, numeric, and scale annotations require `--value`.

## Verification

After every write, the helper script attempts readback verification and returns `verified_matches`.

If verification is delayed, inspect recent records:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py recent \
  --name "Asked Agent to Do Something New" \
  --hours 72 \
  --limit 20
```

For historical writes, make sure the `--hours` window includes the target timestamp.

## Safety

- Do not print access tokens, Magic Links, or private Fulcra records in chat or logs.
- Use dry-run mode before risky writes:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py record \
  --name "Focus" \
  --value 4 \
  --dry-run
```

- Ask before updating or deleting an existing annotation definition.
- For demos, use synthetic names and values unless the user explicitly approves real data.

## License

No license is declared yet. Treat this as source-available until a license file is added.
