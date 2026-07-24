# Atropos Language Specification

**Document status:** Draft
**Specification version:** 0.2
**Project status:** Pre-alpha

## 1. Scope

This document defines the top-level language model of the Atropos programming
language.

Atropos is a text-first programming language for deterministic industrial
control systems. Its programming model is based on IEC 61131-3 and is intended
to support multiple industrial control programming languages within a common
textual source and project model.

The Atropos language specification defines the structure and semantics of
Atropos application source.

This specification does not define:

* A specific processor architecture
* A specific operating system
* A specific PLC vendor implementation
* A compiler implementation language
* A runtime implementation language
* A graphical programming environment
* A fieldbus implementation
* A hardware I/O implementation

Target-specific behavior shall be defined separately from the core language
specification.

## 2. Design Principles

The Atropos language is designed according to the following principles.

### 2.1 Text Is Canonical

Plain-text source files are the canonical representation of an Atropos
application.

Graphical representations of Ladder Diagram, Function Block Diagram, and
Sequential Function Chart logic are views of the canonical source.

A graphical representation shall not contain program behavior that cannot be
represented in canonical Atropos source.

An Atropos application shall be capable of being created, inspected, compiled,
and maintained without a graphical development environment.

Atropos source encodes program topology and semantics. Atropos source shall
not encode graphical layout. Grid positions, wire coordinates, element
placement, and other presentation metadata are not part of the language and
shall not appear in canonical source.

### 2.2 Industrial Control Is the Primary Domain

Atropos is a domain-specific programming language for industrial control.

Language design shall prioritize:

* Deterministic execution
* Predictable timing
* Explicit state
* Observable behavior
* Bounded resource use
* Long-term maintainability
* Online inspection
* Fault diagnosis
* Portability between supported control targets

General-purpose programming language features shall not be added solely
because they are common in general-purpose software development.

### 2.3 IEC 61131-3 Alignment

Atropos shall build toward conformance with applicable IEC 61131-3 requirements.

Where Atropos implements an IEC 61131-3 language element or semantic concept,
its behavior should conform to the applicable standard unless an intentional
deviation is documented.

Atropos uses IEC 61131-3 vocabulary for standard language elements, including
standard function blocks (such as `TON`, `CTU`, and `R_TRIG`) and standard
functions (such as `GT` and `MOV`-class operations). Vendor-specific
instruction mnemonics are not part of the Atropos language. Translation of
vendor-specific textual formats into Atropos source is a tooling concern, not
a language concern.

Atropos does not currently claim IEC 61131-3 conformance.

All future conformance claims shall be supported by documented requirements
and conformance testing.

### 2.4 Language Is Independent of Target

Atropos application source shall not be defined by the implementation details
of a specific PLC, processor, operating system, or vendor development
environment.

The language specification defines application behavior.

Compiler backends and runtimes are responsible for implementing that behavior
on supported targets.

Target-specific configuration will be required for hardware resources,
communications, task assignment, or I/O mapping.

Such configuration should remain separate from application logic wherever
practical.

### 2.5 Multiple Control Languages Are First-Class

Atropos shall support multiple industrial control programming languages.

The initial language model is intended to support:

* Ladder Diagram (LD)
* Structured Text (ST)
* Function Block Diagram (FBD)
* Sequential Function Chart (SFC)

Each supported control language is a first-class Atropos source language.

Atropos shall not require all control logic to be translated into a common
user-facing syntax.

Each source body shall explicitly identify the control language in which it is
authored.

### 2.6 Shared Semantic Environment

Atropos is designed to provide a common semantic environment across all
supported control languages.

The exact degree to which language-specific semantics differ shall be defined
as each control language specification matures. Language-specific syntax and
execution rules may differ, but shared language constructs should use common
definitions wherever practical.

The common semantic environment includes:

* Types
* Variables
* Constants
* Functions
* Function blocks
* Programs
* Symbol resolution
* Scope
* Task relationships
* Source diagnostics

A symbol declared according to the Atropos language rules may be referenced
from any control language in which that symbol and its type are valid.

Language boundaries shall not implicitly create isolated variable or type
systems.

**Expression operator binding is language-specific and is a documented,
intentional divergence.** The Ladder Diagram contact expression uses
power-flow binding (no operator precedence; strict left-to-right
accumulation, as a rung is read), defined in `ladder.md`. Structured Text
uses conventional operator precedence per IEC 61131-3, to be defined in
`structured-text.md`. To prevent confusion between the two rules, the
boolean operator surfaces are deliberately disjoint: Ladder Diagram uses the
symbols `&`, `|`, and `!`; Structured Text uses the keywords `AND`, `OR`,
and `NOT`. A control language shall not accept the other language's boolean
operator surface as an alias.

## 3. Source Model

An Atropos application consists of one or more plain-text source files.

The initial source file extension is:

```text
.atp
```

The `.atp` extension is an initial project convention and may be revised as the
language evolves.

Canonical Atropos applications shall be represented by Atropos source files.

Alternative authoring environments, including graphical tools, shall generate,
load, and save canonical Atropos source files directly rather than maintaining
a separate proprietary source representation.

Users shall be able to inspect and work with the canonical source files even
when an application was initially created through a graphical authoring tool.

### 3.1 Compilation Unit

Each Atropos source file is a compilation unit.

A compilation unit may contain one or more top-level declarations permitted by
the language specification.

Top-level declarations may include:

* Type declarations
* Global variable declarations
* Constant declarations
* Function declarations
* Function block declarations
* Program declarations

The exact grammar for top-level declarations shall be defined in later
revisions of this specification.

### 3.2 Source Ordering

Source declaration order shall not define execution order unless explicitly
stated by the applicable language or execution specification.

The compiler shall determine symbol relationships through semantic analysis.

Task configuration and program execution relationships shall be explicitly
defined.

The physical ordering of source files in a filesystem shall not define
application execution order.

Although declaration order shall not alter application semantics except where
explicitly defined, source should be organized consistently and logically to
support readability, review, troubleshooting, and long-term maintenance.

### 3.3 Source Encoding

Atropos source files shall use UTF-8 encoding.

Compilers shall accept both LF and CRLF line endings.

Line-ending style shall be semantically insignificant.

## 4. Program Organization

Atropos adopts the IEC 61131-3 concept of Program Organization Units.

The core Program Organization Units are:

* Program
* Function Block
* Function

Additional organizational constructs may be defined where required by the
Atropos source or execution model.

### 4.1 Program

A Program is an executable Program Organization Unit associated with
application execution through task configuration.

A Program may contain:

* Local variables
* Function calls
* Function block instances
* Control language routines or bodies
* References to visible global symbols

A Program shall not execute solely because it exists in source.

Program execution shall be established by the Atropos execution and task model.

### 4.2 Function Block

A Function Block is a stateful Program Organization Unit.

A Function Block represents reusable control behavior whose output may depend
upon both its current inputs and its persistent internal state.

Each Function Block instance has its own independent instance state.

Function Block behavior shall be defined by one or more supported Atropos
control language bodies as permitted by the Function Block specification.

Function Block state shall persist between invocations according to the
Atropos execution model.

Function Blocks are appropriate for control components whose behavior develops
over time, including timers, counters, equipment modules, sequences, and other
stateful control objects.

### 4.3 Function

A Function is a stateless Program Organization Unit that produces a result from
its defined inputs.

A Function shall not retain execution state between invocations.

Given the same input values and the same applicable constant values, a Function
shall produce the same result.

Functions are appropriate for calculations, conversions, comparisons, and
other deterministic operations that do not require persistent internal state.

Functions shall follow the state and side-effect requirements defined by the
Atropos function specification and applicable IEC 61131-3 semantics.

The compiler shall reject Function behavior that violates required Function
semantics.

## 5. Control Language Bodies

Executable control logic is contained in a control language body.

Each control language body explicitly declares its source language.

Conceptually:

```text
PROGRAM Example

    LADDER MotorControl

        RUNG MotorSeal
            (StartPB | MotorRun) & !StopPB -> MotorRun;

    END_LADDER


    ST Calculation

        ...

    END_ST

END_PROGRAM
```

The Ladder Diagram body above reflects the current working draft of the
Atropos ladder format (`ladder.md`). The PROGRAM-level grammar, the ST body,
and the exact top-level block delimiters are not yet normative.

### 5.1 Language Identity

A control language body has a defined language identity.

The language identity determines:

* The parser used for the body
* Valid syntax
* Language-specific semantic rules
* Source formatting rules
* Applicable diagnostics
* Graphical representation rules, where applicable

The compiler shall not infer the language of a control body from its contents.

Language identity shall be explicit.

### 5.2 Cross-Language References

Control language bodies may reference symbols declared in the shared Atropos
semantic environment.

For example, a variable written by Ladder Diagram logic may be read by
Structured Text logic if:

* The variable is globally visible to both control bodies
* The variable type is valid for both operations
* The access does not violate language or execution rules

Variables not declared as global shall remain local to the Program Organization
Unit in which they are declared.

All supported control languages shall use the common Atropos type system unless
a language-specific restriction is explicitly defined and justified by the
applicable control language specification.

The compiler shall resolve cross-language symbol references through the common
Atropos symbol table.

### 5.3 Cross-Language Execution

Execution shall begin with a Program invoked by the configured task model.

Within that Program, execution proceeds in defined source order through the
main program body and through any routines, Functions, or Function Blocks that
are explicitly invoked.

A declaration that is not reached through the active invocation path shall not
execute solely because it exists in source.

Each control language shall follow its own defined execution semantics within
its body.

A compiler shall not reorder control bodies or invocations in a manner that
changes defined application behavior.

## 6. Types

Atropos shall define a common type system shared by all supported control
languages.

The type system shall include IEC-aligned elementary types and user-defined
types.

The planned type categories include:

* Boolean types
* Integer types
* Unsigned integer types
* Real types
* Time and duration types
* Character and string types
* Enumerated types
* Array types
* Structured user-defined types

The exact supported types, ranges, conversion rules, and storage semantics
shall be defined in `type-system.md`.

All control languages shall use the common Atropos type system.

A language-specific parser may provide language-appropriate syntax for an
operation, but the resulting semantic type shall be represented consistently
by the compiler.

## 7. Variables and Scope

Atropos shall provide explicit variable declaration and scope rules.

The planned scope categories include:

* Local scope
* Global scope

Local scope applies to variables declared within a Program, Function Block,
Function, routine, or other locally scoped language construct.

Additional scope rules may be defined where required by IEC 61131-3 alignment
or the Atropos execution model.

### 7.1 Local Scope

Local scope should be preferred for state that does not require external
visibility.

A locally scoped variable shall not be accessible outside the Program
Organization Unit or language construct in which it is declared.

Atropos shall not provide a mechanism that exposes an otherwise local variable
outside its declared scope. Data that must be shared shall be declared through
an appropriate global interface.

### 7.2 Global Scope

Atropos shall support globally visible variables.

Global variables are required for practical industrial control applications
and interoperability between application components.

Variables that are not used outside their local Program Organization Unit
should remain locally scoped.

Compiler diagnostics and static analysis tools may issue warnings for excessive
global state, ambiguous ownership, or unsafe access patterns.

Such warnings shall report the condition to the programmer but shall not alter
the defined semantics of otherwise valid application source.

### 7.3 Symbol Identity

Each declared symbol has an identity within the Atropos semantic model.

Symbol resolution shall not depend on graphical object identity,
editor-generated identifiers, or filesystem ordering.

The compiler may generate internal identifiers.

Generated internal identifiers are implementation details and shall not
replace, alter, or obscure user-defined source names.

User-defined identifiers shall remain the canonical names used for source,
diagnostics, debugging, and external inspection unless an explicit aliasing
feature is later defined by the language specification.

## 8. User-Defined Types

Atropos shall support structured user-defined data types.

User-defined types provide named structures composed of defined members.

Conceptually:

```text
TYPE MotorStatus
    Running : BOOL;
    Faulted : BOOL;
    Speed   : REAL;
END_TYPE
```

This syntax is illustrative and is not yet normative.

User-defined types shall be part of the common Atropos type system and shall
be usable by all supported control languages.

User-defined types are native Atropos language constructs and shall be
defined independently of any target-specific implementation.

Compiler backends may map Atropos user-defined types to equivalent
target-specific constructs.

## 9. Function Blocks and Reusable Control Components

Reusable stateful control behavior shall be represented using Function Blocks.

Function Blocks provide the core language mechanism for encapsulating:

* State
* Inputs
* Outputs
* Internal variables
* Control behavior

A compiler backend may map an Atropos Function Block to an equivalent
target-specific reusable control construct.

Function Block semantics shall be defined by the Atropos language
specification rather than by the terminology or implementation model of any
specific target.

Target mappings are backend responsibilities.

## 10. Execution Model

Atropos application execution is deterministic according to the rules of the
Atropos execution specification.

The language specification recognizes the following execution concepts:

* Application lifecycle
* Tasks
* Programs
* Program invocation
* Periodic execution
* Event-driven execution
* I/O process image
* Persistent control state
* Runtime faults

Detailed execution behavior shall be defined in `execution-model.md`.

The compiler and runtime shall preserve defined execution semantics across
supported targets.

## 11. Compiler Semantic Model

All supported source languages shall be translated into a common compiler
semantic model.

The common semantic model is not a fifth Atropos programming language.

It is an internal representation of application meaning.

Conceptually:

```text
Ladder Source --------+
                      |
Structured Text ------+
                      |
FBD Source -----------+----> Semantic Model ----> Atropos IR
                      |
SFC Source -----------+
```

Language-specific information required for source diagnostics, debugging,
or graphical representation shall not be discarded during compilation.

The compiler shall retain sufficient internal source mapping information to
associate executable behavior with its originating source language, canonical
source location, and Program Organization Unit.

The form and storage duration of this internal compiler information are
implementation details.

## 12. Graphical Representation

Graphical representations of graphical IEC control languages may be provided
by optional bolt-on viewer applications.

The compiler itself shall remain text based and shall not require a graphical
programming environment.

A graphical viewer shall derive its display from canonical Atropos source and,
where useful, compiler-provided semantic information.

The graphical rendering of a control language body shall be a deterministic
function of the parsed canonical source. Two renderings of the same source by
conforming viewers shall present the same structure. A viewer shall not read,
write, or require layout metadata, because no such metadata exists in
canonical source (§2.1).

The initial graphical language targets are:

* Ladder Diagram
* Function Block Diagram
* Sequential Function Chart

A graphical viewer shall present the control language in a familiar graphical
form without defining, modifying, or extending executable behavior.

The graphical representation is a view of the existing canonical source, not a
conversion into a second programming language or a separate project format.

Structured Text authored as Structured Text remains Structured Text.

Ladder Diagram authored as Ladder Diagram remains Ladder Diagram.

Function Block Diagram authored as Function Block Diagram remains Function
Block Diagram.

Sequential Function Chart authored as Sequential Function Chart remains
Sequential Function Chart.

## 13. Diagnostics

The Atropos compiler shall provide source-oriented diagnostics.

Diagnostics should include:

* Source file
* Source location
* Diagnostic severity
* Diagnostic identifier
* Human-readable description

Where practical, diagnostics should identify the originating control language
and Program Organization Unit.

Compiler diagnostics shall be deterministic for equivalent source and compiler
configuration.

The diagnostic identifier system shall be defined separately.

## 14. Language Evolution

The Atropos language shall use explicit specification versions.

Language behavior shall not silently change between specification versions.

Breaking semantic changes require:

* Documentation
* A specification version change
* Compiler diagnostics where practical
* Migration guidance where practical

Experimental language features shall be explicitly identified as experimental.

Compiler-specific extensions shall not be represented as standard Atropos
language features.

## 15. Open Questions

The following language design questions remain unresolved:

* Exact top-level source grammar
* Final source file extension
* Whether one source file may contain multiple Programs
* Exact control language block delimiters *(Ladder Diagram: working draft in
  `ladder.md`; ST, FBD, SFC: open)*
* Routine and control body invocation syntax
* Namespace or module support
* Global variable declaration organization
* Variable initialization syntax
* Retained and non-retained variable semantics
* Reference and pointer support
* Dynamic memory restrictions
* Function side-effect restrictions
* Cross-task variable access rules
* Function Block body organization
* Target-specific language extensions
* Source-level attributes and metadata
* Standard library organization
* Standard Function Block organization

The following previously open questions have working-draft resolutions
recorded in `ladder.md` and `docs/decisions/`:

* Ladder Diagram textual representation and rung structure
* Ladder contact-expression operator binding (power-flow binding)
* Standard element vocabulary (IEC 61131-3 names; no vendor mnemonics)

These questions shall be resolved through specification development and
documented design decisions.

Where an unresolved question corresponds to an IEC 61131-3 requirement or
restriction, the applicable IEC behavior shall be the starting point for the
Atropos design unless an intentional deviation is documented.

## 16. Normative Status

This document is an early draft.

Examples are non-normative unless explicitly identified otherwise.

The terms **shall**, **shall not**, **should**, **should not**, and **may**
are intended to develop normative meaning as the specification matures.

Atropos currently makes no claim of IEC 61131-3 conformance or certification.
