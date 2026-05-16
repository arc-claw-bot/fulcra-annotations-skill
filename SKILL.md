---
name: fulcra-annotations
description: Create, list, update, and record Fulcra annotations through the Fulcra Life API. Use when a user asks to log an annotation, create an annotation button/definition, record a moment/boolean/numeric/scale annotation, inspect annotation IDs/source IDs, or build agent workflows that write Fulcra annotation events.
---

# Fulcra Annotations

Use this skill when the user wants an agent to create, record, or verify Fulcra annotations.

Fulcra gives agents and their humans scoped, secure access to read and write real-world context and shared human/agent memory: attention, events, location, calendar, health, wearables, and other streams. Use this skill for the write side of that loop: creating reusable annotation definitions and recording user-approved moments or values.

Agents should use the bundled script first. Do not hand-write curl calls unless the script is missing a required capability, because the script keeps tokens out of chat, builds the Fulcra ingest payload consistently, and performs readback verification.

## Core Concepts

- **Annotation definition**: the reusable button/metric definition, such as `Focus` or `Asked Agent to Do Something New`. Created once with `create`.
- **Annotation record**: one logged occurrence/value of a definition. Written with `record`.
- **Moment annotation**: an event with no value. Use for "this happened" logs.
- **Boolean/numeric/scale annotations**: metric-like records that require `--value`.
- **Definition tags**: reusable labels stored on the annotation definition. Add them when creating the definition with repeated `create --tag`.
- **Record tags**: labels stored on one specific record. Add them when recording with repeated `record --tag`.
- **Tag resolution**: Fulcra stores tag IDs. The script accepts tag names or UUIDs, resolves names to IDs, and creates missing tag names automatically.
- **Historical record**: any record whose event time is not now. Always pass `--recorded-at`.
- **Confirmed write**: a record is not confirmed until readback finds `verified_matches >= 1`.

## Safety Rules

- Never print access tokens, refresh tokens, raw private Fulcra records, credential files, or direct capability URLs in chat.
- Device auth URLs and user codes are allowed only when the intended user needs to approve Fulcra access from another device; send them only in the current trusted/private user channel.
- Ask before deleting or updating an existing annotation definition.
- For public demos, use synthetic annotation names/values unless the user explicitly approves real data.
- Do not claim a write succeeded from HTTP status alone. Check the script result and verify readback.
- Duration annotations are only partially supported by this skill; prefer moment/boolean/numeric/scale until Fulcra documents duration ingest shape more clearly.

## Auth

The script gets auth in this order:

1. `FULCRA_ACCESS_TOKEN` environment variable.
2. `fulcra-api auth print-access-token` from PATH, or the command in `FULCRA_CLI_COMMAND`.

Authenticate first:

```bash
fulcra-api auth login
```

For remote chat agents, keep the CLI running, send the printed device authorization URL and code to the intended user, and wait for approval. The user can open the URL from Discord, Telegram, Signal, mobile, or any browser. After approval, verify auth with a non-token command such as `fulcra-api user-info`; do not paste tokens into chat.

Set `FULCRA_HOME=/path/to/home` if credentials are not under the process `HOME`.

## Common Commands

List annotation definitions before creating a new one:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py list
```

Create a moment annotation definition:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py create \
  --type moment \
  --name "Asked Agent to Do Something New" \
  --description "Logged when the user asks the agent to try a new category of work" \
  --tag agent \
  --tag new-task
```

Use `create --tag` more than once to attach multiple tags to the definition. Definition tags should be short, stable labels such as `agent`, `health`, or `research`. The script resolves tag names to Fulcra tag IDs before sending the API payload.

Record a moment annotation now:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py record \
  --name "Asked Agent to Do Something New" \
  --note "User asked for a new annotation workflow" \
  --tag new-task
```

Use `record --tag` more than once to attach tags to the individual record. If `record --tag` is omitted, the record inherits the definition tags. If `record --tag` is present, those explicit record tags are resolved to Fulcra tag IDs and sent for that record.

Record a historical moment annotation:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py record \
  --name "Asked Agent to Do Something New" \
  --recorded-at "2026-05-15T10:00:00-04:00" \
  --note "User asked for a new annotation workflow"
```

Use a full ISO-8601 timestamp with timezone for historical writes. If the user says "yesterday at 10am", resolve it in the user's timezone and pass the offset explicitly, for example `2026-05-15T10:00:00-04:00`. Fulcra readback may show the equivalent UTC time, such as `2026-05-15T14:00:00+00:00`.

Record by annotation ID when names are ambiguous:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py record \
  --id "<annotation-id>" \
  --note "Logged from automation"
```

Create a 1-5 scale annotation definition:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py create \
  --type scale \
  --name "Focus" \
  --description "How focused do I feel right now?" \
  --scale-labels "1=Scattered,2=Low,3=Neutral,4=Focused,5=Locked In" \
  --default-value 3
```

Record a scale/numeric/boolean value:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py record \
  --name "Focus" \
  --value 4 \
  --note "Deep work block started well"
```

Read back recent records for verification:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py recent \
  --name "Asked Agent to Do Something New" \
  --hours 72 \
  --limit 20
```

Dry-run any write before sending it:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py record \
  --name "Focus" --value 4 --dry-run
```

## Workflow

1. Determine the annotation type: moment, boolean, numeric, or scale. Prefer moment for "this happened".
2. Decide whether tags belong on the definition, the individual record, or both.
3. Run `list` and check whether the definition already exists.
4. If the definition is missing and the user asked to create/log that annotation, run `create`.
5. Record with `record --name ...` or `record --id ...`.
6. For historical writes, always include `--recorded-at "<ISO-8601 timestamp with timezone>"`.
7. Trust only a successful script result with `verified_matches >= 1` for confirmed writes.
8. If verification fails after ingest returns `204`, wait briefly and rerun `recent --id <annotation-id>` or `recent --name "<name>" --hours <window>`.
9. Report the exact timestamp written and the readback timestamp. Do not include tokens, Magic Links, or unnecessary private record data.

## Tag Rules

- Use definition tags for stable classification of the annotation itself, such as `health`, `agent`, `research`, or `workflow`.
- Use record tags for context that applies only to one logged occurrence, such as `new-task`, `manual-test`, `backfill`, or `user-requested`.
- Add definition tags with `create --tag <tag>`. Repeat `--tag` for multiple definition tags.
- Add record tags with `record --tag <tag>`. Repeat `--tag` for multiple record tags.
- `--tag` accepts either a Fulcra tag name or an existing tag UUID.
- Fulcra stores tags as UUIDs. The script resolves tag names to UUIDs and creates a missing tag name automatically.
- If `record --tag` is omitted, the record uses the definition tags.
- If `record --tag` is provided, the explicit record tags are sent for that record.
- Use lowercase, short, reusable tags. Prefer `new-task` over a full sentence.
- Do not use tags for timestamps, detailed notes, values, people names, or private context. Use `--recorded-at`, `--note`, `--value`, and `--source` for those.
- Existing definitions cannot be retagged by this script yet. Ask before using direct API update paths.

## Timestamp Rules

- If `--recorded-at` is omitted, the script records the annotation at the current time.
- If the user asks for a historical or scheduled-looking time, pass `--recorded-at`.
- Use the user's local timezone when resolving relative phrases like "yesterday at 10am".
- Include the timezone offset in the timestamp. Do not pass naive local times.
- UTC readback is expected. Compare instants, not string equality.

Example: on 2026-05-16 in New York, "yesterday at 10am" means `2026-05-15T10:00:00-04:00`, which readback may show as `2026-05-15T14:00:00+00:00`.

## Verification Rules

- `record` returns `recorded_at` and `verified_matches`.
- `verified_matches >= 1` means the script found the written record in Fulcra after ingest.
- For historical writes, use `recent --hours` with a window large enough to include the target time if a second verification is needed.
- When inspecting readback, use only minimal fields needed for confirmation: annotation name/id, `recorded_at`, value if relevant, and note if relevant.

## API Notes

Core REST endpoints:

- `GET /user/v1alpha1/annotation` lists annotation definitions.
- `POST /user/v1alpha1/annotation` creates a definition.
- `PUT /user/v1alpha1/annotation/{annotation_id}` updates a definition.
- `DELETE /user/v1alpha1/annotation/{annotation_id}` deletes a definition.
- `POST /ingest/v1/record` records annotation events.
- Readback uses `/data/v1alpha1/event/MomentAnnotation` for moment/duration and `/data/v1alpha1/metric/{BooleanAnnotation|NumericAnnotation|ScaleAnnotation}` for metric annotation values.

For schema details or upstream gaps, read `references/api-notes.md`.
