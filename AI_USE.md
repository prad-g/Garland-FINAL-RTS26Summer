# AI Use Summary

ChatGPT was used to assist with:

- integrating concepts from the earlier scheduling and event-latency
  applications;
- organizing the final avionics architecture and degraded-mode behavior;
- drafting and reviewing C source, documentation, portfolio copy, and layout;
- checking utilization arithmetic and presenting the scheduling defense; and
- structuring the hazard analysis, demo script, and final reflection.

The measured WCET values, heartbeat counts, and preemption timestamps came from
William Garland's completed Wokwi/ESP32 runs:

- A: 583 µs mean, 584 µs max, 760 µs margin
- B: 243 µs mean, 244 µs max, 318 µs margin
- C: 8,236 µs mean, 8,249 µs max, 10,724 µs margin
- D: 26,441 µs mean, 26,819 µs max, 34,865 µs margin
- Heartbeats: 2,399 / 1,200 / 480 / 240
- Preemption timestamps: 1,060,261 / 1,164,569 / 1,331,435 µs

The student is responsible for running, validating, understanding, explaining,
and submitting the final work.
