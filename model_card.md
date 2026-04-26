# Model Card — PawPal+ AI Pet Care Advisor

## Model Details

| Field | Value |
|---|---|
| Model used | `claude-sonnet-4-6` (Anthropic) |
| Access method | Anthropic Python SDK via REST API |
| Task type | Retrieval-Augmented Generation (RAG) — task suggestion |
| Input | Pet profile (species, age, special needs, existing tasks) + retrieved care guidelines |
| Output | Structured JSON task suggestions + self-reported confidence score |

---

## Intended Use

PawPal+ is a scheduling assistant for pet owners managing multiple pets. The AI component identifies **care tasks that are missing** from the owner's current schedule by:

1. Retrieving relevant veterinary guidelines from a curated 16-entry knowledge base (scored by species, age group, and special-needs keywords)
2. Prompting Claude with the pet's profile plus that context, asking for 3–5 missing tasks as structured JSON
3. Parsing the response into `Task` objects that flow directly into the schedule engine

The system is a **scheduling aid**, not a diagnostic tool. It is not intended to replace veterinary advice.

---

## AI Collaboration — How Claude Was Used in Development

### Design and architecture
Claude was used as a brainstorming partner during class design. The most useful exchange was the discussion around how `Schedule` should relate to `Owner` — Claude suggested passing the full `Owner` object rather than just `available_time`, which made `Schedule.available_time` derivable as a `@property` rather than a duplicated value. This was adopted as-is after evaluating it against the requirements.

### Suggestions accepted with modification
Claude initially suggested using string literals (`"high"`, `"medium"`, `"low"`) for task priority. This was evaluated against the actual scheduler requirements — sort and compare priorities numerically — and rejected in favor of `Priority(Enum)` with integer values (`HIGH=3`, `MEDIUM=2`, `LOW=1`). The enum approach made comparisons reliable, eliminated case-mismatch bugs, and enabled the stable descending sort in `sort_tasks_by_priority()`.

### Suggestions rejected
Early in development, Claude suggested using sentence-transformers embeddings and cosine similarity for the knowledge base retriever. This would have added a large ML dependency (model download, embedding API) to search 16 entries. The heuristic scoring approach (species → age group → special-needs keywords) was chosen instead: it is transparent, fast, fully testable, and equally accurate at this scale.

### Claude integrated into the system (runtime)
The production AI call in `ai_advisor.get_ai_suggestions()` uses a carefully constrained prompt:
- Provides explicit rules ("do not repeat existing tasks," "only suggest tasks appropriate for this species and age")
- Requires one JSON object per line (no markdown fences, no wrapper arrays)
- Requests `CONFIDENCE: <float>` on a final line for a parseable signal
- Retrieved context is injected before the instruction so the model's suggestions are grounded in the knowledge base

The constraint to output bare JSON per line — rather than a wrapper JSON object — came from an AI suggestion that was adopted after testing, because the wrapper approach produced inconsistent formatting that the parser could not reliably handle.

---

## Known Biases and Limitations

### Knowledge base coverage
The 16-entry knowledge base was written by hand and reflects general veterinary consensus for common domestic species. **Dogs and cats dominate.** There are no entries for rabbits, fish, reptiles, birds, or exotic pets. Owners of non-dog/cat pets receive only the 2–3 universal entries (hydration, feeding, vet checkups). This is a data gap, not a model limitation.

### Duration estimation
In manual testing, Claude consistently underestimated task durations for large-breed dogs (suggested 5-minute grooming sessions that would realistically take 20 minutes). This reflects the absence of breed-specific data in the knowledge base; the model has no way to calibrate duration without that signal.

### Self-reported confidence
The `CONFIDENCE: <float>` value is produced by Claude's own assessment, not an external metric. In manual runs it averaged ~0.82 regardless of how sparse the knowledge base context was. It is a useful UI signal but should not be treated as a calibrated probability.

### Species matching
Early versions of the retriever could return dog-specific entries for unknown species because age-group scoring alone could push them into the top-k. A species-must-match guard was added after a test caught this. The fix is covered by `test_unknown_species_returns_universal_entries_only`.

### Misuse potential
The system suggests tasks users might follow without consulting a vet. Mitigations in place: the UI frames suggestions as coming from general guidelines (not diagnosis); each suggestion requires the user to explicitly click "Add"; the app's subtitle reads "scheduling assistant," not "diagnostic tool."

---

## Testing Results

```
38 tests total — 38 passed (100%)

tests/test_pawpal.py      19 tests  — scheduling logic, conflict detection, recurrence, filtering
tests/test_ai_advisor.py  19 tests  — knowledge base loading, retriever scoring, parser, full pipeline
```

All Anthropic API calls in the test suite are mocked (`@patch("ai_advisor.anthropic.Anthropic")`), so tests run offline without an API key.

### Key test findings

| Area | Finding |
|---|---|
| Retriever species guard | Without the guard, dog entries appeared for unknown species. One test caught this; the guard was added. |
| Claude output format | Early runs returned suggestions in markdown code fences. Adding "no markdown code fences" to the prompt resolved this in subsequent runs; the parser logs and skips any line it cannot parse rather than crashing. |
| Duplicate avoidance | Claude reliably avoided suggesting tasks equivalent to existing ones, even when the existing task used informal names ("morning walk" vs. "daily walk"). |
| Duration calibration | Suggested durations for large-breed grooming tasks were systematically short (5 min instead of ~20 min). No fix applied — would require breed-specific knowledge base entries. |
| Confidence score | Manually observed range: 0.74–0.91. Scores were higher when the pet had special-needs keywords that matched knowledge base entries, which is the expected behavior. |

### Confidence level: ⭐⭐⭐⭐ (4 / 5)

All 38 automated tests pass. One star is held back because the Streamlit UI layer (`app.py`) and the plan formatter (`explain_plan`) have no automated tests — correctness of those paths can only be verified manually through the browser. The AI advisor's output quality also cannot be fully captured by unit tests; meaningful evaluation would require a labeled benchmark of pet profiles with known correct suggestions.

---

## Portfolio Reflection

> **What this project says about me as an AI engineer:**
>
> PawPal+ reflects how I think about integrating AI into real systems: the hardest part is never the API call — it's the interface between the model and the rest of the code. I spent more time designing the prompt schema, the retrieval scoring heuristic, and the point in the data flow where AI output becomes a first-class `Task` object than I did on the API integration itself. Throughout the project I treated every AI suggestion — both the Claude suggestions during development and the ones generated at runtime — as a first draft to be evaluated against actual requirements, not instructions to follow wholesale. The switch from string priorities to `Priority(Enum)`, the rejection of sentence-transformers for 16 entries, and the decision to make suggestions actionable rather than just informational are all examples of that habit. I'm most proud that the AI's output doesn't exist in a separate advisory box — it flows through the same conflict detection, priority sorting, and plan generation as every other task in the system.
