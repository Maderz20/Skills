# Debugging

Read this only when investigating a non-trivial bug.

## First moves (do these before anything else)

1. Reproduce the bug. If you can't reproduce, the rest is guessing.
2. Identify the exact failing assertion / error / unexpected output.
3. Locate the code that produces it — `rg` for the error string or function name.
4. Read the immediate function plus its direct callers. Don't go deeper yet.

## Diagnostic order

1. Read the failing code carefully — most bugs are visible on inspection
2. Check recent changes to the area (`git log --oneline -- path/to/file`)
3. Add targeted logging or use a debugger — don't guess
4. Form one hypothesis, test it, discard or confirm
5. If three hypotheses fail, stop and escalate — your mental model of the code is wrong

## Don't

- Don't shotgun fix — change one thing at a time
- Don't add try/except to "fix" a bug — that hides it
- Don't refactor while debugging — separate concerns
- Don't change tests to match buggy output — fix the bug
- Don't speculate about causes you haven't verified

## Escalate when

- Bug spans >3 modules
- Reproduction requires production-like data
- Timing / concurrency is involved
- The code "shouldn't be doing that" — likely a deeper invariant is broken
