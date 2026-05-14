# Ex6 — Rasa structured half

## Your answer

RasaStructuredHalf.run() implements a three-phase pipeline: normalise,
POST, interpret. The input_payload arrives from the loop half (or bridge)
as a dict with a "data" key containing raw booking fields. Phase one calls
normalise_booking_payload from validator.py, which canonicalises each field
(parse_currency_gbp strips "GBP"/"pounds" prefixes, parse_time_24h converts
to HH:MM, canonicalise_venue_id lowercases and underscores), then assembles
a Rasa-shaped message with sender, message text, and metadata.booking dict.

Phase two POSTs this JSON to the Rasa REST webhook endpoint via
urllib.request. Three failure modes are handled without raising: HTTPError
and URLError return HalfResult with error_code SA_EXT_SERVICE_UNAVAILABLE,
TimeoutError returns SA_EXT_TIMEOUT. All three set next_action="escalate"
so the bridge can retry or fail gracefully.

Phase three parses the JSON response array. For each message it checks
custom.action for "committed" or "rejected", and also scans the text
for fallback patterns ("booking confirmed", "can't accept"). A confirmed
booking extracts the booking_reference from either custom.booking_reference
or a regex on the text. Rejection extracts the reason string.

In my offline run (sess_b0abba45a4a7), the stdlib mock server on port
5905 received the normalised payload for haymarket_tap with party_size=6
and deposit_gbp=200. Since party <= 8 and deposit <= 300, the mock
confirmed with reference BK-7D401E9E. The structured half returned
HalfResult(success=True, next_action="complete") with the full booking
and reference in output.

The mock server encodes the same business rules as Alasdair's persona:
party > 8 triggers action="rejected" with reason "party_too_large",
deposit > 300 triggers "deposit_too_high". This ensures offline tests
exercise both confirmation and rejection paths deterministically.

## Citations

- sessions/sess_b0abba45a4a7 — mock Rasa booking confirmed, ref BK-7D401E9E
- starter/rasa_half/structured_half.py:75-213 — RasaStructuredHalf.run()
- starter/rasa_half/structured_half.py:432-484 — mock server rules
- starter/rasa_half/validator.py — normalise_booking_payload + parse helpers
