# Ex9 — Reflection

## Q1 — Planner handoff decision

### Your answer

In my Ex7 run (session sess_9ba10d49e4d2), the DefaultPlanner split the
task into two subgoals: one for open-ended research (assigned to loop)
and one for committing the booking under policy constraints (assigned to
structured). The signal that drove the structured assignment was the
subgoal description referencing deterministic rules — party size limits,
deposit thresholds, confirmation protocol. The planner is prompted with
descriptions of both halves: loop is for "open-ended tool use where the
LLM decides what to do next," structured is for "rule-governed dialog
with known constraints." When a subgoal's success criterion involves
enforcing a policy rather than discovering information, the planner
routes it to structured.

This matters because the structured half enforces rules in Python code
(the mock Rasa server checks party <= 8, deposit <= 300), not in prose.
If the booking subgoal were assigned to the loop half, the LLM would
need to "know" the rules and could hallucinate acceptance of an invalid
booking. The structured half makes the decision mechanically: party_size
> 8 produces action="rejected" regardless of what the LLM thinks.

The broader design lesson is that the planner's routing decision is
advisory — it maps prose descriptions to half types. The real safety
comes from the structured half's code enforcing constraints that the
LLM cannot override. Put the invariants in code, not in prompts.

### Citation

- sessions/sess_9ba10d49e4d2/logs/trace.jsonl — bridge.round_start and state_changed events
- starter/handoff_bridge/bridge.py:56-171 — HandoffBridge.run() dispatching on next_action
- starter/rasa_half/structured_half.py:432-484 — mock server enforcing party/deposit rules

---

## Q2 — Dataflow integrity catch

### Your answer

The integrity check exposed a real fabrication problem in my Ex5 real-LLM
runs. In session sess_8ce1fe808d80, the Qwen3-32B executor produced a
flyer naming "The Edinburgh Grand" as the venue — a name that appears
nowhere in venues.json — with empty cost and weather fields. The
venue_search tool had been called twice with non-matching parameters
(near="Edinburgh City Centre" party=10, near="Old Town" party=15),
returning 0 and 1 results respectively. The executor ignored the actual
results and fabricated a venue name wholesale.

verify_dataflow returned ok=True but only vacuously ("no extractable
facts in flyer") because the LLM left every money and temperature field
blank. This is arguably worse than an explicit failure: the flyer has
correct HTML structure, proper data-testid tags, and a plausible-sounding
venue name. A human reviewer scanning the output would see what looks
like a complete document and might not notice that every fact field is
empty or fabricated.

Compare this to the offline run (sess_23134e2dac2b) where verify_dataflow
confirmed 4 concrete facts against _TOOL_CALL_LOG. The contrast
demonstrates what the integrity check is designed to catch: the LLM
produces confident-looking output regardless of whether it grounded its
facts in actual tool returns. A stricter implementation could treat
vacuous verification (zero facts extracted) as a warning rather than
a pass.

### Citation

- sessions/sess_8ce1fe808d80/workspace/flyer.html — hallucinated venue, empty fact fields
- sessions/sess_23134e2dac2b — offline run with 4 verified facts
- starter/edinburgh_research/integrity.py:118-164 — verify_dataflow extraction and matching

---

## Q3 — Removing one framework primitive

### Your answer

If forced to remove all but one sovereign-agent primitive, I would keep
session directories. The critical production failure mode they prevent
is concurrent booking sessions corrupting each other's state.

In my Ex7 run (sess_9ba10d49e4d2), the handoff bridge wrote a forward
handoff file to session.ipc_input_dir and later archived it to
logs/handoffs/round_1_forward.json. Both paths are scoped to a single
session directory. If two concurrent bookings shared a flat filesystem,
the second bridge round would overwrite the first session's
handoff_to_structured.json, causing the first booking to read the wrong
venue data and potentially confirm a reservation the customer never
requested. Session directories prevent this by giving each booking its
own filesystem namespace — sess_9ba10d49e4d2/ipc/ can never collide
with another session's ipc/.

The same isolation protects the integrity check. verify_dataflow
compares flyer facts against _TOOL_CALL_LOG entries from the current
session. Without per-session directories, tool call records from
concurrent runs would intermingle, and the integrity check could
falsely verify a fabricated fact that happened to match a different
session's tool output. In production with dozens of simultaneous
bookings, this would silently pass hallucinated venue names or costs.

The remaining primitives can each be reimplemented in roughly twenty
lines of code on top of session directories — they are conveniences,
not foundations. Session directories are the atomic unit of truth that
makes every higher-level feature possible.

### Citation

- sessions/sess_9ba10d49e4d2/ — bridge handoff files in ipc/ and logs/handoffs/
- starter/handoff_bridge/bridge.py:103-111 — write_handoff scoped to session paths
- starter/edinburgh_research/integrity.py:118-164 — verify_dataflow reading session-scoped log
