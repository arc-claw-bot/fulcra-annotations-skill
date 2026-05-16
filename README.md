# Fulcra Annotations

Create Fulcra annotation definitions and record annotation events from an agent workflow.

Fulcra gives agents and their humans scoped, secure access to read and write real-world context and shared human/agent memory: attention, events, location, calendar, health, wearables, and other streams. This skill is the write path: agents can create reusable annotation definitions and record moments, booleans, numeric values, and scale ratings after user approval.

## What It Does

- Lists existing Fulcra annotation definitions.
- Creates annotation definitions, including definition-level tags.
- Records annotation events, including historical timestamps.
- Supports moment, boolean, numeric, and scale annotations.
- Supports record-level tags for individual logged events.
- Verifies writes by reading the event back after ingest.

## Requirements

- Python 3.11 or newer.
- Fulcra API access for the target user.
- `fulcra-api auth login` completed, or `FULCRA_ACCESS_TOKEN` supplied by a trusted secret manager.

For remote agents, run `fulcra-api auth login` on the agent host, keep it polling, and send only the printed device authorization URL and user code to the intended user through the active trusted user channel. The user can approve from any browser on any device. Never send access tokens or credential files.

If credentials live outside the process home, set `FULCRA_HOME` to the home directory that contains the Fulcra CLI credentials. If the CLI binary is not `fulcra-api`, set `FULCRA_CLI_COMMAND`.

## Quick Start

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

## Tags

Tags can apply to either the definition or an individual record.

Use definition tags for stable categories that describe the annotation itself:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py create \
  --type moment \
  --name "Medication Taken" \
  --tag health \
  --tag adherence
```

Use record tags for context that only applies to one event:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py record \
  --name "Medication Taken" \
  --note "Backfilled after travel" \
  --tag backfill \
  --tag travel
```

Use short, reusable, lowercase tags such as `health`, `agent`, `research`, `new-task`, `manual-test`, or `backfill`. Do not put timestamps, private notes, or one-off sentences in tags.

## Safety

- Do not print access tokens, Magic Links, or private Fulcra records in chat or logs.
- Use dry-run mode before risky writes.
- Ask before deleting or changing an existing annotation definition.
