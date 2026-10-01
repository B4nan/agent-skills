# Denial of service

Reports in this family claim unbounded recursion, catastrophic regex backtracking, or unbounded allocation on attacker input.

## What to measure

- **Recursion:** find the input depth at which it fails and what it fails with. In JS, stack overflow is a `RangeError` that the caller can usually catch, so it fails one request, not the process. Check whether anything in the path turns it into a crash (an uncaught error in a callback or stream, a worker exit).
- **ReDoS:** time the regex on growing inputs (e.g. 1k, 10k, 100k characters of the pathological pattern) and report the numbers. Linear growth is not ReDoS. Note any length limit applied before the regex runs.
- **Allocation:** measure peak memory for realistic maximum input sizes, after any body-size limit the documented deployment applies.

## Telling severity

- Blocking the event loop or crashing the process affects every user: availability impact can be high.
- A catchable error on one malformed request affects only that request: usually low or none.
- An upstream request-size or depth limit that every realistic deployment has (HTTP body limits, JSON parser limits) bounds the input; say what bound applies.
- Gate 1 still applies: the input has to reach the code through a public entry point at runtime.

## Narrowest fix

A depth or length limit at the recursive entry point, an iterative rewrite of the hot path, or a regex rewritten to remove nested quantifiers. Make the limit generous enough that no legitimate input hits it, and have it throw a clear error.
