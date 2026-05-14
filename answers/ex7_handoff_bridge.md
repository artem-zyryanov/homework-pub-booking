# Ex7 — Handoff bridge

## Your answer

HandoffBridge.run() implements a bounded loop (max_rounds=3) that
alternates between the loop half and structured half with explicit
handoff files as the coordination mechanism.

Each round starts by emitting a bridge.round_start trace event, then
calling loop_half.run(). If the loop returns next_action="complete",
the session is marked complete and the bridge exits. If it returns
"handoff_to_structured", the bridge calls build_forward_handoff() to
construct a Handoff object containing the loop's output data and
return_instructions telling the structured half to respond with
next_action="escalate" plus a reason if it cannot confirm. This
handoff is written to ipc/handoff_to_structured.json via write_handoff(),
and a state-change trace event records the loop-to-structured transition.

The structured half then runs. On confirmation (next_action="complete"),
the session is marked complete. On rejection (next_action="escalate"),
the bridge enters the reverse-handoff path: build_reverse_task()
constructs a new task dict containing prior_result, rejection_reason,
and retry=True. The old forward handoff file is archived to
logs/handoffs/round_N_forward.json rather than deleted, preserving the
audit trail. The bridge then loops back with the reverse task as
current_input for the loop half.

In my run (sess_9ba10d49e4d2), the bridge completed in 2 rounds. The
trace shows bridge.round_start events at each round boundary and
session.state_changed events at each half transition (loop to structured,
then structured to complete). The integrity check in
starter/handoff_bridge/integrity.py verifies that the trace contains
at least one round_start, at least one state_changed, and at least
one tool call — catching the case where the bridge reports success
without performing real work.

The max_rounds=3 bound prevents infinite retry loops. If the structured
half keeps rejecting, the bridge eventually returns
BridgeResult(outcome="max_rounds_exceeded") and marks the session failed,
forcing the caller to handle the unresolvable conflict explicitly.

## Citations

- sessions/sess_9ba10d49e4d2 — bridge completed in 2 rounds
- starter/handoff_bridge/bridge.py:56-171 — HandoffBridge.run() main loop
- starter/handoff_bridge/bridge.py:177-208 — build_forward_handoff + build_reverse_task
- starter/handoff_bridge/integrity.py — trace-based verification
