# Contributing

This repository is a course portfolio and reference implementation. Small,
well-scoped improvements are welcome.

## Before opening a change

1. Preserve the rate-monotonic priority order unless the change includes a new
   scheduling analysis.
2. Keep interrupt handlers nonblocking.
3. Use `vTaskDelayUntil()` for periodic tasks.
4. Do not replace measured evidence with estimates.
5. Update the requirement, hazard, and verification sections when behavior
   changes.

## Validation

- Build the `firmware/` project for ESP32-S3.
- Run nominal mode until WCET values stabilize.
- Exercise fault entry and recovery.
- Verify A/B heartbeats continue through degraded mode.
- Confirm the static site in `docs/` works at mobile and desktop widths.

## Commit style

Use a short imperative summary, for example:

```text
Add deadline-miss trace to fault console
```
