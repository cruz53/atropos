# Atropos

**An open-source, text-first IEC 61131-3 programming language and toolchain
for deterministic industrial control.**

> **Project status: Pre-alpha / specification and architecture development**

Atropos is an effort to build a modern, text-first programming environment
for industrial control systems based on the IEC 61131-3 programming model.

The project begins with a simple premise:

**PLC source code should be source code.**

Industrial control software is still commonly developed inside vendor-specific
graphical environments and stored in project formats that are difficult to
inspect, diff, review, test, automate, and integrate with modern software
development tools.

Atropos aims to preserve the languages, execution concepts, and deterministic
behavior familiar to PLC engineers while making plain-text source files the
canonical source of truth.

The goal is not to replace PLC programming with Python, C, Rust, or another
general-purpose programming language.

The goal is to build a PLC programming toolchain designed around industrial
control from the beginning.

## Project Goals

Atropos is being designed around the following goals:

* Text-first PLC application development
* Alignment with IEC 61131-3 semantics and a path toward formal conformance
* Multiple IEC programming languages within a common source and project model
* Deterministic industrial control execution
* Git-native version control and code review
* Portable compilation targets
* Hardware-independent application source
* A defined runtime interface and hardware abstraction model
* Terminal-based online monitoring and debugging
* Machine-readable compiler and semantic interfaces
* Compatibility with modern static analysis, testing, automation, and
AI-assisted engineering tools
* An open specification independent of any single automation vendor

## Text Is the Source of Truth

Atropos source files are plain text.

Graphical representations of Ladder Diagram, Function Block Diagram, and
Sequential Function Chart programs are considered **views of source code**,
not the canonical project format.

The intended model is:

```text
Plain-Text Source
        |
        v
 Atropos Compiler
        |
        v
 Semantic Model / IR
        |
        +------> Ladder View
        |
        +------> FBD View
        |
        +------> SFC View
        |
        +------> Structured Text
        |
        v
  Target Backend
        |
        v
 Executable Control Application
```

A graphical editor may eventually be built for Atropos, but the use of a
graphical editor must never be required to create, inspect, compile, deploy,
or debug an Atropos application.

## Multi-Language Source Model

IEC 61131-3 languages represent different control problems well.

Atropos does not intend to force every control problem into a single syntax.

Instead, source explicitly identifies the language used by each routine or
code block.

A project may contain Ladder Diagram, Structured Text, Function Block Diagram,
and Sequential Function Chart logic while sharing a common type and variable
system.

The following is a conceptual example only. The Atropos grammar has not yet
been defined:

```text
PROGRAM BatchSystem

LADDER MotorControl

    XIC(StartPB) XIO(StopPB) OTE(MotorRun);

END_LADDER


ST FlowCalculation

    FlowTotal := FlowTotal + FlowRate;

END_ST


SFC BatchSequence

    STEP Idle INITIAL;

    TRANSITION StartBatch
        WHEN StartCommand;

    STEP Filling;

END_SFC

END_PROGRAM
```

This syntax is illustrative and should not be interpreted as a stable language
specification.

## Architecture

Atropos is currently envisioned as four coordinated architectural components.

### Language Specification

The language specification defines:

* Source structure
* Type system
* Variables and scope
* User-defined data types
* Functions
* Function blocks
* Program organization units
* Tasks
* Ladder Diagram textual representation
* Structured Text
* Function Block Diagram textual representation
* Sequential Function Chart textual representation
* Execution semantics
* Cross-language symbol behavior
* Error and fault semantics

The language specification defines **behavior**, not a specific runtime
implementation.

### Compiler

The Atropos compiler parses all supported source languages into a common
semantic representation.

The planned compilation pipeline is conceptually:

```text
Source
  |
  v
Language Parser
  |
  v
Semantic Analysis
  |
  v
Atropos Intermediate Representation
  |
  v
Target Backend
```

Initial development is expected to use C as the first native code generation
target.

A C backend allows an early Atropos compiler to use mature existing toolchains
such as Clang or GCC for native code generation.

C is considered an implementation backend, not part of the Atropos language
specification.

Future backends may target other native toolchains, intermediate
representations, industrial runtimes, or vendor-supported formats.

### Runtime

The Atropos runtime provides the control execution environment required by
compiled applications.

Expected runtime responsibilities include:

* Deterministic task scheduling
* Periodic and event-driven task execution
* I/O process image management
* Hardware abstraction
* Timer and counter services
* Runtime fault handling
* Application lifecycle management
* Communications services
* Symbol and variable inspection
* Online value modification
* Debug services
* Runtime diagnostics
* Application image validation
* Simulation support

Compiled Atropos applications are intended to interact with the runtime
through a defined and versioned runtime ABI.

The compiler must not depend on the internal implementation of the runtime.

### Tooling

Atropos tooling is expected to include:

* Command-line compiler tools
* Project management commands
* Source formatting
* Static analysis
* Language server support
* Terminal-based online debugging
* Variable monitoring
* Runtime diagnostics
* Application deployment
* Test and simulation tools

The planned online debugger is terminal-first and may use a TUI interface
suitable for local terminals or remote SSH sessions.

Graphical tooling may be developed in the future as a client of the same
compiler and runtime interfaces.

## Portable Targets

Atropos source is intended to remain independent of a specific processor,
operating system, PLC vendor, or hardware platform.

The language front end and semantic analysis should be shared across targets.

Target-specific behavior belongs in compiler backends, runtime ports, and
hardware adaptation layers.

```text
                    +--> C / Native Backend
                    |
Atropos Source --> IR +--> Simulation Backend
                    |
                    +--> Future Industrial Target
                    |
                    +--> Future Vendor Backend
```

A backend must preserve the behavior defined by the Atropos language and
execution specifications.

The long-term goal is for the same control application source to be portable
across multiple supported industrial targets with target-specific
configuration isolated from application logic wherever practical.

## Industrial Control Is the Primary Use Case

Atropos is not intended to be a general-purpose application language.

Its design should favor:

* Deterministic behavior
* Explicit state
* Predictable execution
* Bounded resource use
* Observable control flow
* Long-lived industrial applications
* Online diagnostics
* Safe failure behavior
* Reviewability by controls engineers

Language features that conflict with these priorities may be rejected even
when they are common in general-purpose programming languages.

## Git-Native Development

Atropos projects should work naturally with standard source control workflows.

The project intends to support:

* Meaningful line-based diffs
* Branching
* Pull requests
* Code review
* Blame and history
* Automated builds
* Automated tests
* Static analysis
* Reproducible compilation

A binary or opaque graphical project database must not be required as the
authoritative source for an Atropos application.

## Machine-Readable by Design

The compiler should provide structured access to its understanding of a project.

Tools should not need to reverse engineer source text or graphical project
files to understand an Atropos application.

Compiler interfaces may expose:

* Abstract syntax trees
* Semantic information
* Symbol tables
* Type information
* Cross references
* Task relationships
* Program organization
* Source locations
* Diagnostics

These interfaces are intended to support editors, analysis tools, test
systems, documentation generators, and AI-assisted engineering tools.

AI is not the definition of Atropos.

However, a well-defined textual language and semantic toolchain should allow
AI systems to inspect, generate, review, and modify industrial control code
more reliably than workflows based on screenshots or opaque vendor project
formats.

## IEC 61131-3

Atropos intends to align its control programming model and semantics with IEC
61131-3 and to build toward formal language conformance where practical.

The project is currently in the specification phase and **does not claim IEC
61131-3 conformance at this time**.

Conformance claims must be based on documented behavior, conformance testing,
and the applicable requirements of the standard.

Atropos is an independent open-source project and is not affiliated with or
endorsed by Rockwell Automation, CODESYS, PLCopen, the IEC, or any other
automation vendor or standards organization.

## Current Status

Atropos is currently at the beginning of specification and architecture
development.

There is no production compiler or runtime.

The immediate priorities are:

1. Define the project charter and architectural boundaries.
2. Define the source and project model.
3. Define the execution model.
4. Define the common type and variable system.
5. Define the initial textual Ladder Diagram representation.
6. Define the Structured Text language profile.
7. Define the Atropos intermediate representation.
8. Define the compiler/runtime ABI boundary.
9. Build the first parser and semantic analysis prototype.
10. Execute a minimal mixed-language control application in simulation.

The project is intentionally beginning with specification work before
implementation.

## Contributing

Atropos is in an early design phase.

Contributions, criticism, and industrial control experience are welcome. Early
discussion should focus on semantics, execution behavior, portability, and
practical PLC engineering workflows rather than adding features for their own
sake.

Before substantial implementation contributions are accepted, the project
intends to define a contributor and licensing policy that preserves both the
open-source nature of Atropos and the project's ability to support sustainable
commercial development.

## License

Atropos is currently licensed under the GNU General Public License v3.0.

The licensing architecture of the compiler, runtime, generated application
code, and future SDK components is under active design.

A core requirement is that PLC applications authored by Atropos users remain
the property of their authors and are not made subject to Atropos licensing
solely because they were compiled using the Atropos toolchain.

See `LICENSE` and future licensing architecture documentation for details.

---

**Atropos is an experiment in what PLC development could look like if
industrial control software were designed for plain text, open tooling,
modern version control, and portable execution from the beginning.**

