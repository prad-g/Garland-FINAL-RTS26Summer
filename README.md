# Aegis Flight Computer

**Real-Time Systems Final Portfolio — Summer 2026**  
**Author:** William Garland  
**Target role:** Avionics / Embedded Software Engineer  
**Platform:** ESP32-S3, ESP-IDF, FreeRTOS, Wokwi

> Aegis is a mixed-criticality avionics scheduler that protects attitude and
> control tasks when an injected CPU-overload fault forces the system outside
> its nominal timing assumptions.

## Portfolio links

- GitHub Pages: https://prad-g.github.io/Garland-FINAL-RTS26Summer/
- Wokwi: https://wokwi.com/projects/471017682208757761
- Demo video: https://youtu.be/WcYueR0EY3Y

## 1. Project overview

Aegis models a small flight-control computer on an ESP32-S3. Four periodic
FreeRTOS tasks are pinned to Core 1 and assigned fixed priorities using
rate-monotonic scheduling. Task A samples attitude at 100 Hz, Task B updates an
inner control loop at 50 Hz, Task C assembles a telemetry packet at 20 Hz, and
Task D performs health monitoring and housekeeping at 10 Hz. A low-priority
mission monitor runs on Core 0 so serial output does not distort the Core 1
timing experiment.

The original App 2 scheduler supplied the periodic architecture, heartbeat
counters, preemption evidence, and measured worst-case execution time. The
portfolio enhancement adds an App 3-style GPIO interrupt on GPIO 18. The ISR
uses a direct-to-task notification to wake a sporadic fault manager without
performing nontrivial work in interrupt context.

The first accepted button press injects an eight-times attitude-computation
workload. The health path watches deadline counters and enters a degraded mode
when timing stress is detected or when the 700 ms confirmation window expires.
Degraded mode preserves Tasks A and B, sends one telemetry packet for every five
nominal telemetry releases, and skips the nonessential insertion-sort
housekeeping workload. A second button press clears the fault and restores all
nominal services.

This is a classroom demonstrator rather than certifiable airborne software. It
uses an assurance-oriented structure—explicit timing requirements, traceable
hazards, measured evidence, and deterministic fault behavior—without claiming
DO-178C compliance or FAA approval.

## 2. Integrated system

### Application lineage

| Course application | Reused element | Final integration |
|---|---|---|
| App 2 — Multi-task scheduling | Four periodic tasks, RMS priorities, WCET, heartbeats, preemption | Main flight workload and schedulability baseline |
| App 3 — Event latency | GPIO 18 ISR and direct-to-task notification | Low-latency fault-injection command path |
| Final portfolio enhancement | Criticality-aware service shedding | Degraded mode protects flight-critical tasks |

### Requirements

| ID | Requirement | Verification |
|---|---|---|
| RTS-01 | Attitude task shall release every 10 ms with priority 15 on Core 1. | Source inspection + heartbeat ratio |
| RTS-02 | Control task shall release every 20 ms with priority 10 on Core 1. | Source inspection + heartbeat ratio |
| RTS-03 | Telemetry shall release every 50 ms nominally. | Source inspection + heartbeat ratio |
| RTS-04 | Health monitoring shall release every 100 ms. | Source inspection + heartbeat ratio |
| RTS-05 | Periodic releases shall use absolute-time delay. | `vTaskDelayUntil()` inspection |
| RTS-06 | Each task shall record mean and maximum execution time. | Serial WCET table |
| RTS-07 | GPIO 18 shall notify the fault manager from an ISR. | Button event and log |
| RTS-08 | Degraded mode shall preserve Tasks A/B and reduce C/D workload. | Injected-fault run |
| RTS-09 | A second accepted button press shall restore nominal service. | Recovery run |

## 3. System architecture

```text
CORE 0 — OBSERVATION                         CORE 1 — REAL-TIME CONTROL

+-----------------------+                    +----------------------------+
| Mission monitor, 1 Hz |<-- atomic words ---| A  Attitude sample         |
| heartbeats / WCET /   |                    |    10 ms, priority 15       |
| misses / fault state  |                    +-------------+--------------+
+-----------------------+                                  |
                                                           v
GPIO 18                                                    +----------------------------+
   |                                                       | B  Inner control loop      |
   v                                                       |    20 ms, priority 10      |
+-----------------------+                                  +-------------+--------------+
| Minimal button ISR    |                                                |
| vTaskNotifyGiveFromISR|                                                v
+-----------+-----------+                                  +----------------------------+
            |                                              | C  Telemetry packet        |
            v                                              |    50 ms, priority 5       |
 +----------------------+                                  +-------------+--------------+
 | Fault manager, P14   |                                                |
 | debounce / inject /  |                                                v
 | degrade / recover    |                                  +----------------------------+
 +----------------------+                                  | D  Health / housekeeping   |
                                                           |    100 ms, priority 2      |
                                                           +----------------------------+
```

All four periodic tasks are pinned to Core 1. The priority order is
`A > B > C > D`. A can preempt B, C, and D; B can preempt C and D; C can
preempt D. The sporadic fault manager uses priority 14, below attitude and above
the remaining periodic tasks, because the event is urgent but must not displace
the highest-criticality 10 ms sample.

## 4. Task table and measured WCET

The completed App 2 Wokwi run executed long enough to record 2,399 / 1,200 /
480 / 240 heartbeats. These counts match the expected period ratio:

```text
A:B:C:D = 2399:1200:480:240 ≈ 10:20:50:100 ms
```

| Task | Function | Period | Mean WCET | Max WCET | Max + 30% | Deadline | Priority | Core |
|---|---|---:|---:|---:|---:|---:|---:|---:|
| A | Attitude sample | 10 ms | 583 µs | 584 µs | 760 µs | 10 ms | 15 | 1 |
| B | Inner control loop | 20 ms | 243 µs | 244 µs | 318 µs | 20 ms | 10 | 1 |
| C | Telemetry packet + CRC | 50 ms | 8,236 µs | 8,249 µs | 10,724 µs | 50 ms | 5 | 1 |
| D | Health / housekeeping | 100 ms | 26,441 µs | 26,819 µs | 34,865 µs | 100 ms | 2 | 1 |

The task bodies exclude dynamic allocation. `esp_timer_get_time()` brackets the
deterministic work, and the maximum recorded time is multiplied by 1.30 and
rounded up. The margin values are used as computation times in the analytical
defense.

### WCET evidence excerpt

```text
Task  Function           Period   Priority Heartbeats  Mean(us) Max(us) Max+30%(us)
A     Attitude sample    10 ms    15       2399        583      584     760
B     Control loop       20 ms    10       1200        243      244     318
C     Telemetry packet   50 ms    5        480         8236     8249    10724
D     Health check       100 ms   2        240         26441    26819   34865
```

## 5. Schedulability defense

For four independent periodic tasks with deadline equal to period, the
Liu-Layland rate-monotonic sufficient utilization bound is:

```text
U_RM(4) = 4(2^(1/4) - 1) = 0.75683
```

Using measured maximum WCET plus 30% margin:

```text
U = CA/TA + CB/TB + CC/TC + CD/TD

  = 760/10,000
  + 318/20,000
  + 10,724/50,000
  + 34,865/100,000

  = 0.07600 + 0.01590 + 0.21448 + 0.34865
  = 0.65503
```

Because `0.65503 < 0.75683`, the nominal task set passes the sufficient
rate-monotonic test. The result does not prove certification or cover blocking,
I/O, cache, multicore interference, or interrupts; it does provide a
conservative course-level scheduling defense for the measured workload.

### Required 3× Task A stress case

```text
U_3xA = U + 2(CA/TA)
       = 0.65503 + 2(760/10,000)
       = 0.80703
```

`0.80703 > 0.75683`, so the stressed set no longer passes the sufficient RM
bound. A utilization above the bound does not by itself prove deadline failure,
but it removes the simple schedulability guarantee and motivates response-time
analysis or mitigation. The final demonstration makes that engineering
decision visible through fault injection and service shedding.

## 6. Preemption evidence

The following adjacent lines came from the completed App 2 run:

```text
PREEMPT: task_d health_check start t=1060261
PREEMPT: task_a attitude_sample tick t=1164569
PREEMPT: task_d health_check finish t=1331435
```

The timestamps satisfy:

```text
1060261 < 1164569 < 1331435
```

Task D began, Task A executed before Task D finished, and both tasks were pinned
to Core 1. Therefore the priority-15 attitude task preempted the priority-2
health task.

## 7. Fault injection and graceful degradation

### Normal mode

- A: attitude sample every 10 ms
- B: control update every 20 ms
- C: full 1,024-byte CRC workload every 50 ms
- D: essential checks plus worst-case insertion-sort housekeeping every 100 ms
- Status LED: slow blink

### Injected fault

Pressing the red Wokwi button produces a falling edge on GPIO 18. The ISR calls
`vTaskNotifyGiveFromISR()` and yields only when the awakened task has higher
priority than the interrupted task. The fault manager debounces the command,
enables an eight-times Task A computation load, and observes deadline counters.

### Degraded mode

After timing stress is confirmed:

- Tasks A and B retain their periods and priorities.
- Task C sends one full telemetry packet for every five nominal releases,
  changing the effective packet interval from 50 ms to 250 ms.
- Task D continues essential monitoring but skips the nonessential insertion
  sort.
- Dropped telemetry releases and deadline misses remain visible.
- The status LED is steady on.

This is mixed-criticality load shedding: flight-critical work remains
available, while mission and background work are reduced. Pressing the button
again clears the injected workload and restores the nominal services.

### Expected serial sequence

```text
W (...) AEGIS: [FAULT] 8x attitude workload enabled; monitoring deadline counters
E (...) AEGIS: [DEGRADED] timing stress confirmed; telemetry=1/5, nonessential housekeeping=OFF
...
I (...) AEGIS: [RECOVERY] injected workload cleared; nominal services restored
```

## 8. Hazard analysis

| ID | Failure condition | Effect | Risk | Design control | Verification |
|---|---|---|---|---|---|
| HZ-01 | Attitude task misses 10 ms deadline | Stale state reaches controller | High | Highest RM priority, deadline counter, low-criticality shedding | WCET + overload run |
| HZ-02 | CRC workload starves control | Control output delayed by telemetry | High | Telemetry priority 5, decimation in degraded mode | Heartbeat + recovery trace |
| HZ-03 | Mixed-frame shared state | Telemetry inconsistency | Medium | Atomic single-word handoffs; protected snapshot planned for production | Interface review |
| HZ-04 | Button bounce/repeat | Nondeterministic fault command | Medium | 200 ms debounce + direct notification | One transition per press |

The fuller analysis is in [HAZARD_ANALYSIS.md](HAZARD_ANALYSIS.md).

## 9. Industry-standard mapping

The FAA's AC 20-115D recognizes DO-178C as an acceptable means—but not the only
means—of showing compliance for airborne software. It describes life-cycle
planning, development, verification, configuration management, quality
assurance, and certification-liaison evidence. NASA-STD-8739.8B describes
software safety analysis methods used to identify software hazards and controls.

This portfolio is **aligned in mindset only**:

| Assurance concept | Portfolio artifact |
|---|---|
| Explicit software requirements | RTS-01 through RTS-09 |
| Architecture and interfaces | Core/task diagram and shared-state definition |
| Verification evidence | WCET table, heartbeat ratios, utilization, preemption |
| Robustness / off-nominal test | GPIO-triggered overload and recovery |
| Software safety analysis | Hazard table with controls and verification |
| Configuration management | Repository, license, versioned source, ZIP backup |

Sources:

- [FAA AC 20-115D](https://www.faa.gov/documentLibrary/media/Advisory_Circular/AC_20-115D.pdf)
- [NASA-STD-8739.8B](https://soma.larc.nasa.gov/stp/dynamic/pdf_files/NASA-STD-87398B%20SoftwareAssuranceSafety.pdf)

No compliance, certification, DAL, or approval claim is made.

## 10. AI use disclosure

ChatGPT was used to help integrate the previous applications, organize the
avionics narrative, draft the portfolio site and documentation, and review the
schedulability calculations. The WCET values, heartbeat counts, and preemption
timestamps used in the portfolio came from William Garland's completed Wokwi
run. The student remains responsible for understanding, running, explaining,
and validating the final submission.

## License

MIT License. See [LICENSE](LICENSE).
