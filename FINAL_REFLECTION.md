# Final Reflection

## Real-Time Systems — Aegis Flight Computer

The most important change I would make if I restarted this course would be to
design each application around a single evolving system from the beginning.
The avionics theme was present in my earlier work, but the final portfolio made
me see how much stronger the applications become when they share one technical
narrative. A periodic scheduler, interrupt-latency experiment, synchronization
exercise, and task-notification comparison can look like isolated laboratory
assignments. When they are treated as parts of a flight computer, each one
answers a specific engineering question: when must work execute, how quickly
can an external event reach a task, how is shared state protected, and what
happens when the processor cannot provide every service at its nominal rate?

I would also begin collecting evidence earlier and in a more structured format.
The final result depended on measured heartbeat counts, maximum execution
times, margin calculations, and serial timestamps. Capturing those values only
at the end creates a risk that a correct result is difficult to reproduce or
explain. In a future project, I would define an evidence table with the
requirement, test configuration, expected result, actual result, and artifact
name before running the first experiment. I would also keep Wokwi screenshots
and serial logs with configuration flags in their filenames. That would make it
easier to compare nominal, loaded, synchronized, and faulted runs without
depending on memory.

The part that was harder than expected was understanding the difference between
a program that runs correctly and a system whose timing behavior can be
defended. At first it was easy to look at a responsive serial monitor and assume
that every task was operating properly. The heartbeat counters showed why that
is not enough. A task may continue to execute while slowly drifting, missing a
deadline, or running at the wrong ratio relative to other tasks. Using
`vTaskDelayUntil()` made the release pattern explicit, while the heartbeat
counts gave a simple long-duration check that the intended 10, 20, 50, and 100
millisecond periods were preserved.

Worst-case execution time was another challenge. A mean time describes typical
behavior, but real-time design must account for the longest observed path and
for uncertainty beyond that observation. My measured maximums were 584
microseconds for attitude sampling, 244 microseconds for the control loop, 8,249
microseconds for telemetry, and 26,819 microseconds for health and housekeeping.
Adding a 30 percent margin produced a nominal utilization of 0.65503. Because
that value is below the four-task rate-monotonic sufficient bound of 0.75683, I
could make a conservative schedulability argument for the measured task set.
The analysis also exposed the design's sensitivity. Tripling Task A increased
the utilization to 0.80703, which no longer passed the sufficient test.

The fault-injection enhancement was the most valuable part of the portfolio
because it changed the analysis from a static calculation into an engineering
response. The system does not treat every workload as equally important.
Attitude sampling and control are flight-critical; telemetry is mission-useful;
the insertion-sort housekeeping workload is nonessential. Under injected CPU
stress, the final design maintains the high-criticality tasks, reduces
telemetry to one packet every five releases, and removes the expensive
housekeeping operation while retaining essential health monitoring. This is a
small demonstration of graceful degradation. It shows that a real-time system
should have a planned off-nominal mode rather than simply failing when the
nominal schedule is no longer defensible.

The interrupt path also clarified the importance of choosing the correct
communication primitive. The GPIO interrupt service routine performs no
substantial processing. It uses a direct-to-task notification to wake a
sporadic fault-manager task, then leaves the decision and logging work in task
context. This keeps interrupt latency bounded and avoids using a mutex or other
blocking primitive inside the ISR. The design is simpler than a general queue
because the event carries only one meaning: toggle the injected fault state.

The most valuable lesson I learned is that real-time software is defined by
evidence and controlled behavior, not merely speed. A fast task can still be
incorrect if it runs after its deadline, reads inconsistent state, or prevents
a more critical task from executing. Priorities need a policy, periods need an
absolute reference, shared data needs an ownership strategy, and failures need
a response that can be observed and verified. The preemption timestamps made
the scheduler behavior concrete: Task D started at 1,060,261 microseconds, Task
A executed at 1,164,569 microseconds, and Task D finished at 1,331,435
microseconds. Since the Task A event occurred between Task D's start and finish
on the same core, the log directly demonstrated fixed-priority preemption.

This project also changed how I would discuss embedded work in an interview.
Instead of listing that I used an ESP32 and FreeRTOS, I can explain a complete
engineering decision. I assigned rate-monotonic priorities, measured execution
times, calculated 65.503 percent nominal utilization, proved preemption from
timestamps, injected an overload through an ISR notification path, and designed
a degraded mode that protected the critical control functions. That narrative
shows the relationship between code, timing analysis, safety thinking, and
verification.

For production, I would replace synthetic sensor data with a timestamped IMU
interface and replace separate atomic variables with a protected multi-field
flight-state snapshot. I would extend the analysis to include blocking,
interrupt interference, cache behavior, multicore interference, and actual
hardware measurements. I would add watchdog and brownout recovery, unit and
integration tests, hardware-in-the-loop fault injection, requirements
traceability, configuration records, and structural coverage appropriate to
the required assurance level. The current implementation is a classroom
demonstrator, but it established the habits I would carry into a professional
avionics or embedded-systems role: define the timing contract, measure it,
challenge it, and make the response to failure deliberate.
