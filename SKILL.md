---
name: fulcra-annotations
description: Create, list, update, and record Fulcra annotations through the Fulcra Life API. Use when a user asks to log an annotation, create an annotation button/definition, record a moment/boolean/numeric/scale annotation, inspect annotation IDs/source IDs, or build agent workflows that write Fulcra annotation events.
---

# Fulcra Annotations

Use this skill when the user wants an agent to create or record Fulcra annotations. Prefer the bundled script over hand-written curl so tokens stay out of chat and payloads stay consistent.

## Safety Rules

- Never print access tokens, Magic Links, raw private Fulcra records, or capability URLs in chat.
- Ask before deleting or updating an existing annotation definition.
- For public demos, use synthetic annotation names/values unless the user explicitly approves real data.
- Duration annotations are only partially supported by this skill; prefer moment/boolean/numeric/scale until Fulcra documents duration ingest shape more clearly.

## Auth

The script gets auth in this order:

1. `FULCRA_ACCESS_TOKEN` environment variable.
2. `fulcra-api auth print-access-token` from PATH.
3. `uv tool run 'git+https://git@github.com/fulcradynamics/fulcra-api-python.git@add-cli' auth print-access-token`.

If using the beta CLI, authenticate first:

```bash
uv tool run 'git+https://git@github.com/fulcradynamics/fulcra-api-python.git@add-cli' auth login
```

Set `FULCRA_HOME=/path/to/home` if credentials are not under the process `HOME`.

## Common Commands

List annotation definitions:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py list
```

Create a Moment annotation definition:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py create \
  --type moment \
  --name "Asked Agent to Do Something New" \
  --description "Logged when the user asks the agent to try a new category of work"
```

Record an annotation event by name:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py record \
  --name "Asked Agent to Do Something New" \
  --note "User asked for a new annotation workflow"
```

Record by annotation ID when names are ambiguous:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py record \
  --id "<annotation-id>" \
  --note "Logged from automation"
```

Create a 1-5 scale:

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

Dry-run any write:

```bash
python3 skills/fulcra-annotations/scripts/fulcra_annotations.py record \
  --name "Focus" --value 4 --dry-run
```

## Workflow

1. Run `list` and check whether the definition already exists.
2. If missing and the user asked for creation, run `create`.
3. Record with `record --name ...` or `record --id ...`.
4. Trust only a successful script result with `verified_matches >= 1` for confirmed writes.
5. If verification fails after ingest returns `204`, wait briefly and rerun `recent --id <annotation-id>`.

## API Notes

Core REST endpoints:

- `GET /user/v1alpha1/annotation` lists annotation definitions.
- `POST /user/v1alpha1/annotation` creates a definition.
- `PUT /user/v1alpha1/annotation/{annotation_id}` updates a definition.
- `DELETE /user/v1alpha1/annotation/{annotation_id}` deletes a definition.
- `POST /ingest/v1/record` records annotation events.
- Readback uses `/data/v1alpha1/event/MomentAnnotation` for moment/duration and `/data/v1alpha1/metric/{BooleanAnnotation|NumericAnnotation|ScaleAnnotation}` for metric annotation values.

For schema details or upstream gaps, read `references/api-notes.md`.
