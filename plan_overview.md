
# Verification Plan/Roadmap (Skeleton)

This is a list of considerations for front-end verification. Priorities are P0 (most critical), P1 (must have), and P2 (should have).

## Verification Methodology

### P0: Unit Test Frameworks

- Unit tests can use SystemVerilog/UVM and/or cocotb.
- The choice depends on designer preference and test complexity. Both should run with VCS and Verilator.

### P0: Gold Reference Cosimulation Model

- Use `sw-sysemu` as the gold reference model for functional correctness.
- Provides functional verification through RTL cosimulation (multiple cosimulation models).
- P0: Two basic cosimulation setups: full SoC and CPU-subsystem. (SoC-noCPU uses scoreboards.)
- P1: ISS Multicore Cosimulation
  - Must support a multicore false-sharing testing option.
  - Enables verification of cache and consistency corner cases.
- P2: Checkpoint/Restore
  - `sw-sysemu` must be able to generate checkpoints.
  - Checkpoints must be loadable by both `sw-sysemu` and RTL simulation.
  - Helps break down overly large simulations/tests for debugging.

### P1: Constrained Random Generation

- SoC-noCPU configuration should use constrained-random generation + a scoreboard.
- Unit tests can use directed tests or constrained-random generation + a scoreboard.
- Options:
  - **SystemVerilog** constrained random with built-in constraint solvers
  - **cocotb + pyvsc** (Python Verification Stimulus and Coverage) for Python-based constraint solving

## Tracking and Reporting

### P1-P2: Daily Regression with Coverage/bug/performance Tracking

A web dashboard that summarizes nightly regressions. It tracks:

- Assertions, functional, and code coverage
- Coverage by SoC category (unit tests, full SoC, CPU-subsystem, SoC-noCPU)
- Build times per configuration
- Verification/simulation times per test and total
- Pass/fail tests
- Historical trends for build, test, and verification performance

### Test Plan: Track Cover/Directed

- Table/spreadsheet showing operations, corner cases, and sharing patterns, plus where they are covered.
- Some way to track/monitor coverage (functional and code).


## Verification Setups

### P1: Per-test/per-block unit tests

- The RTL designer (or close collaboration) may write simple but non-trivial unit tests for the module.
- Constrained-random is a great fit for these.

### Main Integration Verification Setup

#### P0: Full SoC (all-RTL cosimulation)

- Capable of running cosimulation at full-SoC level.

#### P0: CPU-subsystem

- One CPU-subsystem (no memory, no AXI4 NoC); single clock domain. Main advantage over full SoC is faster iteration.
- Keep UVM for existing scoreboards/tests. For P0, CPU-subsystem uses UVM and avoids cocotb initially (P2?).
- P0: Switch to `sw-sysemu`

#### P1: SoC-noCPU (no CPU-subsystem)

- Full SoC without CPU-subsystem (memory hierarchy + I/O) to generate patterns that are hard to reach with CPU-driven traffic.
- Synthetic tests (constrained-random generation using SystemVerilog or cocotb + pyvsc) to test SoC without CPUs.
- SystemVerilog or cocotb scoreboards
  - Includes checks for livelock/deadlock scenarios
  - Monitors outstanding transaction limits and timeouts
  - Validates interconnect behavior without CPU-generated traffic

### P1: Netlist Verification

- If using Synopsys synthesis, use Cadence Conformal to check netlist equivalence. If using Cadence synthesis, use Synopsys Formality.
- VCD-based power estimation on netlists for key benchmarks.

### P2: Emulation

- FPGA emulation as close to SoC as possible

## Tools/Infrastructure

### Simulation and Build Tools

#### P2: Front-end/Verilog Verification

- Verilator and VCS should be able to run all (P2) integration tests.
- Support incremental compilation when possible to reduce build times.
- Track compilation times and verification runtime (include in daily results).

#### P2: Linting/formatting

- Use a linting tool (Verilator or Verible), and optionally formatting (verible-format).

### P2: Formal Verification

- **AXI4 Protocol Formal Verification**:
  - Commercial tools: Cadence JasperGold, Synopsys VC Formal
  - Open-source option: [https://github.com/pulp-platform/axi](https://github.com/pulp-platform/axi) (formal properties for AXI protocol)
- Formal property verification for critical blocks (arbiters, FIFOs, cache controllers)
- Formal model checking for FSM exhaustive state coverage

### Verification Checking

#### P1: X-prop (VCS only)

- Policy for what is considered valid X-prop behavior.
- No X values should reach cosimulation (verification checks) unless reset is asserted.
- X-propagation checking enabled to catch uninitialized signals.

##### P1: Reset Testing Strategy

**Note**: Strategy still TBD, options under consideration:

- P1: SDF+Netlist for a few hunder cycles to boot (no clock generator block)
- P2: Possible approach: run a SPICE simulation for a few cycles to validate reset behavior.
- Verify all state elements properly initialize during reset
- Ensure clean reset propagation across all clock domains

#### P1: Assertions (SVA)

- Designer-provided assertions (or help) for conditions that should never happen in the module/block.
- Track SVA coverage on the regression dashboard alongside other coverage metrics.
- Protocol checkers (SVA-based) if possible for standard interfaces

##### SVA Verification Tools

- **JasperGold** is the typical industry-standard tool for formal assertion verification
  - Try to explore options for open-source/cost-aware flows (and runtime SVA instead of formal?).
- **Alternative **: use Yosys-based flows for assertion checking
  - Yosys can check some assertions during simulation
  - Lower cost but less comprehensive than JasperGold formal verification

## Regression/Tests

#### P0: Regression Requirements

Divide performance/verification into three groups:

- **3-day full validation** (tape-out quality): full regression runs all required tests
  - Runs weekly
  - Leverages Verilator to scale
  - Also run with VCS (may take longer; license-dependent)

- **Overnight suite:** most important tests (integration + performance + power)
  - Gate merges on the overnight suite passing
  - May batch multiple pull requests for testing (not always)

- **1h "critical" suite:** must-run/pass before each pull request

- Someone (verification lead?) owns moving tests between full/overnight/critical suites and deprecations (based on coverage and manual inspection).

Enable GitHub merge queues or some form of test batching across PRs if there are lots of PRs for nightly regressions.

Regression is deterministic: tests may use a random seed, but log it to allow exact re-execution.

### P1: Performance simple kernels

- memcpy, triad, matrix multiply...
- Dhrystone

### P2: Performance small apps or App checkpoints

- Key apps or checkpoints from apps that can load in integration tests

### P0: Random Program Generators (RPG)

- Coverage-driven: RPG and constrained random generation driven by both code coverage and functional coverage
- In-house MTG (custom ISA)
- Use riescue ([https://github.com/tenstorrent/riescue](https://github.com/tenstorrent/riescue))
- Consider FORCE-RISCV ([https://github.com/openhwgroup/force-riscv](https://github.com/openhwgroup/force-riscv))

### P1: Directed Tests

- Create a repository of handwritten (or AI-assisted) tests that target corner cases or missing coverpoints (ideally integrated with RPG randomness).
- Atomics, `cache_flush`, and custom assembly are typical directed tests.

### P1: Past Bugs or Corrective Regression

- Any test (RPG or manual) that finds a real RTL bug gets added to Directed Tests to keep it in regression.

### P1: RISC-V Compliance and Open Tests

- RISC-V official test suites (riscv-tests, riscv-arch-test) should be added to regression as manual tests
- Other publicly available test suites integrated as directed tests
- Ensures ISA compliance and provides additional test coverage

### P1: ECC Injection

- Able to run regressions with full-SoC and random ECC failures

## Coverage Plan

- Define flow/setup to gather code/functional coverage (Verilator)

### P0: Code Coverage

- Goal is 100% toggle coverage for 1-bit wires, or a waiver per wire explaining why coverage is not required.
- All FSM states covered.

### P0: Functional Coverage

- Coverpoints per block/module for what the designer thinks a difficult-but-possible test should do (e.g., FIFO full and push+pop in the same cycle).
- Coverage-driven verification: RPG and constrained random generation driven by both code coverage and functional coverage goals


### Coverage Requirements

- P1: 100% single-bit signals toggle coverage (with formal waivers for uncovered signals)
- P1: All FSM states covered
- P0: All designer-specified coverpoints hit

### P2: Bug Tracking and Closure

- Track bug discovery rate over time
- Require a stabilization period with no new critical/high-severity bugs discovered (e.g., 1 week clean run)
- Maximum outstanding bug counts by severity level (TBD)

### P2: Defeature Options (Chicken-bits)

- Maintain "chicken-bits" or defeature options for risky/new features
- Allows disabling problematic features in silicon if issues found post-tapeout
- Document all chicken-bit controls and their impact on functionality

### P1: Additional Criteria (TBD)

- Regression pass rate requirements
- Performance target achievement
- Waiver approval process

## Specialized Verification

### P3: RISC-V Debug (trace/OpenOCD/... support)

- Consider something like [https://github.com/pulp-platform/riscv-dbg](https://github.com/pulp-platform/riscv-dbg)

### P1: Cross Clock or Power Domains

- Synopsys or Cadence cross-clock checks to avoid issues.
- Power domain (UPF) verification (often supported in the same tools).

# Issues Outside Front-end Verification

## P1: DFT Plan

- What DFT features exist: scan, MBIST/LBIST, boundary scan/JTAG, compression, repair
- Verification approach for DFT modes (entry/exit, isolation, clocks, resets, safety)
- Testcases for scan enable/disable behavior, ATPG assumptions, and DFT constraints sanity

## P1: Backend

- Run at least once a week for timing/area/power.
- Fully automated back-end (DRC and hold/setup fixes).
- Post-layout parasitic extraction (RC extraction)
- Post-layout timing verification (static timing analysis with extracted parasitics)
- LVS (Layout vs Schematic) verification

## Sign-off Criteria

- Many specifics still to be decided and agreed upon by the team.

### P2: Power Management (TBD)

**Note**: Requirements depend on architecture decisions regarding DVFS and power domains.

If applicable, verification needed for:

- Dynamic Voltage and Frequency Scaling (DVFS) transitions
- Power domain entry/exit sequences
- Clock gating verification
- Power gating verification
- Voltage/frequency scaling corner cases
- Retention register verification
- Power domain isolation checks
