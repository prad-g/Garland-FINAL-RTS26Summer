# Aegis Software Hazard Analysis

## Scope

This analysis covers the classroom ESP32-S3/Wokwi demonstrator. It does not
assign an aircraft development assurance level and does not claim compliance
with FAA regulations, RTCA DO-178C, or a production safety process.

The analysis follows an industry-aligned structure:

```text
Failure condition → local effect → system effect → control → evidence
```

## Hazard log

| ID | Failure condition | Local effect | System effect | Initial risk | Design control | Verification evidence | Residual risk |
|---|---|---|---|---|---|---|---|
| HZ-01 | Attitude task misses 10 ms deadline | New attitude sample is late | Control task may use stale flight state | High | Highest periodic priority; absolute-time releases; deadline counter; degraded mode sheds lower-criticality work | WCET table, heartbeat ratio, injected-overload run | Medium |
| HZ-02 | Control task misses 20 ms deadline | Control output is late | Reduced closed-loop stability margin | High | Priority 10; telemetry and housekeeping below control; overload response preserves B | Fault run shows A/B continuing while C/D are reduced | Medium |
| HZ-03 | Telemetry CRC consumes excessive CPU | Lower-priority workload interferes with D and may increase response time of later releases | Loss of health visibility or schedule collapse | High | C remains below A/B; packet generation decimated 5:1 in degraded mode | Dropped-packet counter and effective 250 ms period | Low |
| HZ-04 | Health housekeeping consumes excessive CPU | Nonessential insertion sort delays low-priority execution | Less timing margin for mission services | Medium | Housekeeping is priority 2 and disabled in degraded mode while essential checks remain | D WCET comparison before/after fault | Low |
| HZ-05 | Shared flight data contains inconsistent fields | Telemetry combines values from different frames | Incorrect ground interpretation | Medium | Demonstrator uses atomic single-word handoffs; production requirement calls for protected versioned snapshot | Code inspection; production roadmap item | Medium |
| HZ-06 | Button bounce creates repeated fault commands | Fault enters and clears rapidly | Nondeterministic demonstration state | Medium | Falling-edge interrupt plus 200 ms task-context debounce | One state transition per physical press | Low |
| HZ-07 | Fault event is lost | Stress cannot be injected or cleared | Test result is invalid; not a flight hazard in this demonstrator | Low | Direct-to-task notification with latched count | Serial `[FAULT]` / `[RECOVERY]` trace | Low |
| HZ-08 | Monitor output interferes with timing | Serial formatting extends task execution | WCET or deadlines no longer represent flight workload | Medium | Monitor pinned to Core 0; periodic tasks on Core 1; preemption logs bounded | Core assignment inspection | Low |
| HZ-09 | Simulator behavior differs from hardware | WCET evidence is not portable | Incorrect production scheduling assumption | High | Evidence labeled Wokwi-specific; production plan requires target-hardware measurement | Documentation review | Medium |

## Safety requirements derived from hazards

| Safety requirement | Source hazards | Implementation |
|---|---|---|
| SR-01: A and B shall retain nominal periods during degraded mode. | HZ-01, HZ-02 | Degraded mode does not change A/B periods or priorities. |
| SR-02: Telemetry workload shall be reducible without blocking A or B. | HZ-02, HZ-03 | C is priority 5 and sends one packet per five releases in degraded mode. |
| SR-03: Nonessential housekeeping shall be removable during timing stress. | HZ-04 | D bypasses insertion sort in degraded mode. |
| SR-04: ISR shall not block or perform application logging. | HZ-06, HZ-07 | ISR sends a direct notification only. |
| SR-05: Deadline misses and shed work shall remain observable. | HZ-01–HZ-04 | Per-task miss counters and telemetry-dropped counter. |
| SR-06: Simulator timing shall not be represented as hardware qualification evidence. | HZ-09 | Explicit scope and production roadmap. |

## Verification matrix

| Test | Procedure | Pass criterion |
|---|---|---|
| V-01 Nominal heartbeat | Run at least 20 seconds without button input. | Ratios approach 100:50:20:10 releases per second; no unexpected stop. |
| V-02 WCET capture | Observe monitor after at least 100 A releases. | Mean, max, and margin are nonzero and max never decreases. |
| V-03 Preemption | Capture D start, A tick, D finish on Core 1. | `D_start < A_tick < D_finish`. |
| V-04 Fault entry | Press GPIO 18 button once. | One `[FAULT]` line followed by one `[DEGRADED]` line. |
| V-05 Critical service | Observe A/B after degraded entry. | A/B heartbeats continue at nominal ratio. |
| V-06 Load shedding | Observe C and dropped counter. | C advances approximately once per 250 ms; dropped counter increases. |
| V-07 Recovery | Press GPIO 18 button again. | `[RECOVERY]` appears, C returns to 50 ms, D resumes full housekeeping. |
| V-08 Debounce | Hold or rapidly click button inside 200 ms. | No more than one accepted transition in debounce interval. |

## Assurance reference

- [FAA AC 20-115D](https://www.faa.gov/documentLibrary/media/Advisory_Circular/AC_20-115D.pdf)
- [NASA-STD-8739.8B](https://soma.larc.nasa.gov/stp/dynamic/pdf_files/NASA-STD-87398B%20SoftwareAssuranceSafety.pdf)

These references motivate evidence organization only; they do not make the
project compliant or approved.
