**From COBOL to F\*: Extracting Verified Behavior from Legacy Systems via OpenTelemetry**

---

### Abstract

This document outlines a methodology for migrating legacy COBOL systems into formally verified F\* code. Instead of attempting direct translation or refactoring, it proposes a process of behavioral observation using OpenTelemetry (OTel), extracting side-effect trails, inferring state transitions and contracts, and reconstructing verified F\* equivalents from empirical behavioral models.

---

### Motivation

COBOL remains a cornerstone of critical financial, insurance, and governmental systems. Despite its age and arcane structure, organizations continue to rely on it because it "just works." However, its rigidity, lack of encapsulation, and opaque procedural structure make it nearly impossible to safely modify, verify, or modernize.

Refactoring COBOL directly is a recipe for disaster: undocumented business logic, global mutable state, and a complete absence of type systems or contracts make correctness guarantees impossible.

The solution? Treat COBOL as a **black box**, observe its behavior exhaustively, and construct a **verified, observable clone**.

---

### Overview of the Methodology

The process is divided into several phases:

1. **Instrumentation via OpenTelemetry**
2. **Behavioral Trace Collection**
3. **State Transition Modeling**
4. **Contract Inference**
5. **Formal Reimplementation in F\***
6. **Behavioral Diffing and Test Oracles**

---

### Phase 1: Instrumentation via OpenTelemetry

Enhance the original COBOL system with minimal, non-intrusive instrumentation:

* Identify side-effecting operations: `MOVE`, `ADD`, `SUBTRACT`, `WRITE`, `READ`, `CALL`, `PERFORM`, `IF`, etc.
* Insert `PERFORM` calls to logging paragraphs just before or after side-effecting lines.

  * Example: `PERFORM LOG_MOVE OF LOGGING-SECTION` where `LOG_MOVE` appends a structured trace to a file or sends a UDP packet.
* Use COBOL's `CALL` mechanism to invoke native routines:

  * Define C-shared libraries with simple `void log_move(const char* from, const char* to, double value)` signatures.
  * Use `CALL "log_move" USING VAR1, VAR2, VALUE` directly from COBOL.
* Minimize changes by using conditional compilation (if supported) or preprocessing macros to toggle between instrumented and raw builds.
* Optionally: use dynamic analysis on mainframe environments (e.g., z/OS SMF exits or user exits) to passively tap into file and DB I/O without modifying source code.

#### Example Telemetry Span:

```json
{
  "operation": "MOVE",
  "from": "TRANSACTION-AMOUNT",
  "to": "BALANCE",
  "value": 100.50,
  "line": 4730,
  "context": "PROCESS-INTEREST",
  "timestamp": "2025-07-22T23:47:11Z"
}
```

These logs are exported to a centralized collector for analysis.

---

### Phase 2: Behavioral Trace Collection

Aggregate traces across:

* Multiple runs
* Real-world and edge-case data sets
* Controlled environments (test/mainframe clone, not production)

The result is a high-fidelity record of the system's **actual behavior**, including:

* Variable lifetimes and value evolution
* Execution paths
* Conditionals and decisions
* Data dependencies and correlations

---

### Phase 3: State Transition Modeling

Using the telemetry logs, construct a behavioral model:

* Build **finite-state machines** or graph-based flow models of how values propagate and mutate.
* Identify common sequences, transitions, and dependency chains.
* Capture control flow with a mapping of `PERFORM`, `GOTO`, `IF`, and `EVALUATE` constructs.

This model is the foundation of understanding the system's **implicit semantics**.

---

### Phase 4: Contract Inference

From the observed traces:

* Infer **preconditions**: *Variable X is always >= 0 when this operation executes.*
* Infer **postconditions**: *After operation Y, Z is always true.*
* Discover **invariants**: *Balance is always positive, account IDs are unique, etc.*

These contracts are translated into F\* annotations:

```fstar
val update_balance : amt:float -> bal:float -> Tot float
  (requires (amt >= 0.0))
  (ensures (fun r -> r >= bal))
```

---

### Phase 5: Formal Reimplementation in F\*

Using the inferred model and contracts:

* Rebuild the business logic in F\*, using total functions, monads for I/O, and dependent types.
* Mirror the original behavior **functionally**, not syntactically.
* Make side effects explicit.
* Integrate the inferred contracts for verification.

---

### Phase 6: Behavioral Diffing and Test Oracles

* Re-run the same COBOL batch job and F\* equivalent side-by-side.
* Compare telemetry output: state transitions, results, logs.
* Differences flag semantic divergence — used to refine contracts or detect missed behavior.

Over time, test cases from COBOL runs become **test oracles** for the verified implementation.

