AntimatterSim v10 — Tier 0 Edge-Cloud Interlock

Software reference model, FMEA, clock-domain design, and hardware simulation for the beam-gating interlock between the FPGA gate, edge (Jetson) node, and cloud digital twin described in AntimatterSim v10.

This is a software design and simulation exercise, not hardware-qualified control software. Nothing here should be treated as ready for clinical or hardware deployment. Its purpose is to make the interlock design executable, testable, and falsifiable as cheaply as possible — in a Python process — before any of it becomes an expensive hardware-bench debugging session.

Status at a glance

61 automated tests passing across 4 modules (example-based, property-based/fuzzed, and hardware-simulated).

5 real defects found and fixed during this exercise (Section 3), each one invisible to the test layer that came before it found it.

4 items remain genuinely open and cannot be closed by further software work (Section 5) — clinical sign-off, hardware feasibility, real RTL timing, and realistic network modeling.

Document series

Read in this order — each stage found something the previous one couldn't see:

#DocumentWhat it covers1AMSv10_Risk_Prioritized_Roadmap.mdOriginal v10 feature set re-triaged by patient-safety impact and regulatory exposure rather than technical novelty2AMSv10_Edge_Cloud_Interlock_Spec.mdThe state-machine design itself: states, transitions, authority boundaries between FPGA/edge/cloud3AMSv10_Interlock_FMEA_and_Validated_Implementation.mdFormal FMEA on the design, plus the first executable reference implementation with example-based and property-based (Hypothesis) tests4AMSv10_Clock_Synchronization_Design.mdWhy "synchronize all the clocks" is the wrong instinct; shared-counter design for FPGA↔edge, deliberately-unsynchronized design for edge↔cloud5AMSv10_Hardware_Simulation_Full_Validated_Implementation.mdDiscrete-event simulation of the FPGA gate, edge node, and cloud interacting under crashes and network loss — found two permit-revocation-latency bugs6AMSv10_Choke_Point_Refactor_Full_Validated_Implementation.mdReplaces the per-call-site fix from #5 with a structural one (on_transition hook), and proves the replacement is more robust, not just differently shaped—This READMEMap of the whole series and the code, plus how to run it 

Code modules

interlock_state_machine.py Core FSM: states, transitions, watchdog, audit log, on_transition hook clock_sync.py GatePermit, FpgaGateValidator, clock-domain-separation design hardware_simulator.py Discrete-event simulation: SimClock, FpgaGateHardware, EdgeNode, CloudNode test_interlock_state_machine.py 33 example-based tests, one per transition-table row test_interlock_properties.py Hypothesis stateful fuzzer (500 examples x 80 steps) on the FSM alone test_clock_synchronization.py 16 tests proving safety decisions are unaffected by any other actor's clock test_hardware_simulation.py 11 tests (incl. 1000-example fuzz) on the full FPGA+edge+cloud interaction 

Running the tests

pip install pytest hypothesis python -m pytest test_interlock_state_machine.py test_interlock_properties.py \ test_clock_synchronization.py test_hardware_simulation.py -v 

Expected result: 61 passed. The property-based and fuzz tests use randomized search (Hypothesis), so runtime varies (roughly 10-15 seconds total) but pass/fail should not.

Governing principles baked into the design

These aren't decorative — every module's structure follows from them, and the tests exist specifically to catch violations of them:

Absence of a valid, current, in-tolerance signal is a stop condition, never a continue condition. The system fails to beam-off, never fails to beam-on.

The fastest loop has the least logic. The FPGA's permit check is a single integer comparison — no unit conversion, no skew compensation, no judgment calls.

No auto-reconciliation on ambiguity. A plan-hash mismatch on reconnect always faults and requires explicit operator recovery, even if the "new" plan might be more correct.

Every time-based safety decision uses only the local clock of the actor making it. Cross-actor timestamps are for audit-log correlation only, never a safety-decision input.

State-change side effects live in one place, not at every call site. The on_transition hook exists because relying on caller discipline at each call site produced the same class of bug twice, in two different places, during this exercise.

Findings timeline (what each stage caught that the last one missed)

Found byDefectFixed byFirst run of example testsFalsy-zero bug: if self._hold_started_at: treated a legitimate 0.0 timestamp as unsetis not None checkDesign review while writing testsTime-based budgets only re-checked reactively; could exceed budget with nothing to trigger the faultAdded tick() watchdog methodRe-reading the implementation for the FMEArecord_degraded_dose() accepted negative deltas, could mask a real budget overrunReject negative deltas with ValueErrorHardware simulation, tolerance-exceeded pathPhysical gate stayed open on a stale-but-valid permit for up to one edge-cycle after the FSM moved to HOLDManual immediate resync (later superseded)Hardware simulation, heartbeat path (found after raising fuzz budget)Same class of bug as above, on a different transition path (DEGRADED_EDGE_ONLY -> HOLD on reconnect)Manual immediate resync (later superseded)—(the two manual fixes above were themselves diagnosed as the real problem: fixes that depend on call-site discipline)on_transition choke-point refactor — moves the fix into the state machine's own transition path so it can't be missed at a new call site 

What remains genuinely open

These cannot be closed by more software work — they need a person or a piece of hardware this process doesn't have:

Clinical sign-off on the four safety-budget constants (hold_timeout_ms, heartbeat_miss_threshold, degraded_max_duration_s, degraded_max_dose_fraction_pct) — currently engineering placeholders.

Hardware feasibility of the shared FPGA-edge counter described in the clock synchronization design, or a specified PTP-over-dedicated-link fallback with its own sync-quality fault path.

Real FPGA/RTL timing — propagation delays, clock-domain-crossing metastability, register setup/hold times are not modeled anywhere in this software.

Realistic network behavior — UnreliableNetwork models independent packet drops; real WAN links have correlated bursts and variable latency, neither modeled here.

Independent hardware E-stop confirmed to bypass this software entirely — the single most safety-critical open item, unchanged since the original spec.

Who should look at what

Clinical physics: the four budget constants (section above), and the FMEA document's severity/occurrence/detection scoring.

Hardware/RTL engineering: the shared-counter design, the FPGA permit-validator's assumed timing, and eventually validating this state machine's transition table as the RTL's behavioral spec.

Software/safety review: the FMEA document itself should get an independent second pass — it was produced by the same process that wrote the code, which the FMEA document itself flags as a conflict of interest worth resolving before this goes further.

