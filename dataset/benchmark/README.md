# Benchmark Items

This file will contain 40–50 labeled benchmark items after Phase 4 is complete.

## Format

Each item follows the schema defined in `docs/blueprint.md` Section 4.1:

```json
{
  "id": "Q001",
  "question": "...",
  "category": "answerable|unanswerable|ambiguous|conflicting",
  "expected_answer": "...",
  "expected_tool": "search|calculator|lookup",
  "expected_behavior": "answer|abstain|clarify",
  "relevant_doc_ids": [1, 2],
  "notes": "..."
}
```

## Labeling Process

See `labeling_notes.md` for the full calibration round documentation.

**Calibration round:** Two independent annotators label items Q001–Q015 independently, then reconcile.
**Full labeling:** Items Q016–Q050 labeled consistently after calibration.
