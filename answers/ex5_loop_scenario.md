# Ex5 — Edinburgh research loop scenario

## Your answer

The DefaultPlanner decomposed the task into two subgoals: sg_1 ("research
Edinburgh venues near Haymarket for a party of 6") and sg_2 ("produce an
HTML flyer with the chosen venue, weather, and cost"). Both were assigned
to the loop half because they require open-ended tool use rather than
deterministic dialog.

The executor ran sg_1 first. It issued venue_search, get_weather, and
calculate_cost as parallel tool calls in a single turn — this is safe
because all three are registered with parallel_safe=True (they only read
JSON fixtures and perform pure computation). venue_search filtered
venues.json by open_now, area substring match on "Haymarket", party
size >= 6, and budget constraint, returning Haymarket Tap. get_weather
looked up Edinburgh on 2026-04-25 and returned cloudy/12C. calculate_cost
applied the formula: base_per_head(18) * venue_modifier(1.0) * 6 * 3 =
324 subtotal, 32 service charge, plus 200 min_spend = 556 total, with a
20% deposit of 111.

For sg_2 the executor called generate_flyer with event_details containing
all facts from the prior tools. generate_flyer is parallel_safe=False
because it writes workspace/flyer.html. The HTML uses data-testid
attributes on every key fact so verify_dataflow can parse them
structurally via extract_testid_facts.

After the flyer was written, verify_dataflow extracted four facts
(two money values, one temperature, one weather condition) and confirmed
each appeared in _TOOL_CALL_LOG. The integrity check passed: "dataflow
OK: verified 4 fact(s) against tool outputs."

In the real-LLM runs (Qwen3-32B executor), the model frequently ignored
the explicit tool arguments in the task prompt — searching for party
sizes of 10-50 and areas like "Edinburgh City Centre" that do not exist
in the fixture. This illustrates exactly why the integrity check matters:
when the LLM did produce a flyer, it hallucinated venue names and omitted
cost data entirely, passing verification only vacuously ("no extractable
facts"). The FakeLLM scripted trajectory is deterministic and always
produces a correct flyer, demonstrating the gap between offline testing
and real model behavior.

## Citations

- sessions/sess_23134e2dac2b (offline) — 4 verified facts, correct flyer
- sessions/sess_8ce1fe808d80 (real) — vacuous verification, hallucinated venue
- starter/edinburgh_research/tools.py — tool implementations
- starter/edinburgh_research/integrity.py — verify_dataflow + fact extraction
