# Atropos Language Specification

**Document status:** Draft
**Specification version:** 0.1
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

All supported Atropos control languages participate in a common semantic
environment. (not sure about this it is a good goal to work towards but idk if
there will be differences or not)

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

Language boundaries shall not implicitly create isolated variable systems.

## 3. Source Model

An Atropos application consists of one or more plain-text source files.

The initial source file extension is: (good start but we are not married to
this.)

```text
.atp
```

The file extension is part of the Atropos project convention and does not
define a compiler implementation.

An Atropos compiler may accept source from other input mechanisms provided the
resulting program is semantically equivalent to canonical Atropos source. (no
the other sources should create .atp files directly and load those. i want to
encourage the user to notice and use the atp files even if they started with a
graphical interface)

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
stated by the applicable language or execution specification. (yes but i do not
want this to allow for sloppy code)

The compiler shall determine symbol relationships through semantic analysis.

Task configuration and program execution relationships shall be explicitly
defined.

The physical ordering of source files in a filesystem shall not define
application execution order.

### 3.3 Source Encoding

Atropos source files shall use UTF-8 encoding.

Canonical Atropos source shall use LF line endings. (should work with either
LF or CRLF, controls engineers are often on windows machines)

Compilers may accept other platform line endings as input but shall treat
line-ending differences as semantically insignificant. (yes just said this)

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

Each Function Block instance has its own instance state.

Function Block behavior shall be defined by one or more supported Atropos
control language bodies as permitted by the Function Block specification.

Function Block state shall persist between invocations according to the
Atropos execution model.

### 4.3 Function

A Function is a Program Organization Unit that produces a result from its
jjdefined inputs.

Functions shall follow the state and side-effect requirements defined by the
Atropos function specification and applicable IEC 61131-3 semantics.

The compiler shall reject Function behavior that violates required Function
semantics.

(please make the difference between function blocks and functions more clear)

## 5. Control Language Bodies

Executable control logic is contained in a control language body.

Each control language body explicitly declares its source language.

Conceptually:

```text
PROGRAM Example

    LADDER MotorControl

        ...

    END_LADDER


    ST Calculation

        ...

    END_ST

END_PROGRAM
```

The example above is illustrative.

The exact block delimiters, declaration syntax, and grammar are not yet
normative. (Yes we will need to work on this more)

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

* The variable is visible in both scopes (global scope, all other scopes will
be local)
* The variable type is valid for both operations (variable types should be
common across all languages unless you can give me a convincing argument as to
why not)
* The access does not violate language or execution rules

The compiler shall resolve cross-language symbol references through the common
Atropos symbol table.

### 5.3 Cross-Language Execution

The use of multiple source languages does not independently define execution
order. (youve said this a lot)

Execution order is determined by:

* Task configuration
* Program organization
* Routine or body invocation
* Language-specific execution semantics

(Code execution will scan line by line through the main program unit and any
subroutines called by that program unit anything outside of that would be
declared but not called)

A compiler shall not reorder control bodies in a manner that changes defined
application behavior.

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
* Program scope
* Function Block instance scope (to me this is just local scope said 3 times)
* Global scope

Additional scope categories may be defined where required by IEC 61131-3
alignment or the Atropos execution model.

### 7.1 Local Scope

Local scope should be preferred for state that does not require external
visibility.

A locally scoped variable shall not be accessible outside its defined scope
unless explicitly exposed through a defined language mechanism.
(No there will be no such mechanisms)

### 7.2 Global Scope

Atropos shall support globally visible variables.

Global variables are required for practical industrial control applications
and interoperability between application components.

The existence of global scope shall not imply that unrestricted global state
is preferred. (any variables that are not used out of the local program should
be local)

Compiler diagnostics and static analysis tools may identify excessive global
state, ambiguous ownership, or unsafe access patterns. (yes i like this)

Such diagnostics shall not change the defined semantics of valid application
source unless the relevant rule is defined as a compilation error.
(no it should just report on the issues, it is on the programmer to change
issues. these sound like compiler warnings to me actually)

### 7.3 Symbol Identity

Each declared symbol has an identity within the Atropos semantic model.

Symbol resolution shall not depend on graphical object identity,
editor-generated identifiers, or filesystem ordering.

The compiler may generate internal identifiers.

Generated internal identifiers are implementation details and shall not
replace canonical source names in the language specification. (yes but i 
would like the user generated function to work in the same way though you are
right they should not alter existing symbols)

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
(good syntax)

This syntax is illustrative and is not yet normative.

User-defined types shall be part of the common Atropos type system and shall
be usable by all supported control languages.

User-defined type semantics shall not depend on a vendor-specific construct
such as a Rockwell Automation UDT. (a little too on the nose, we need to say
this without literally saying it)

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

A compiler backend may map an Atropos Function Block to a target-specific
reusable control construct.

The Atropos language specification shall not define Function Blocks in terms
of vendor-specific constructs such as Add-On Instructions. (again too on
the nose)

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

The compiler shall retain sufficient source mapping information to associate
executable behavior with canonical source locations.

(i think you mean temporary files i am fine with that)

## 12. Graphical Representation

Atropos may provide graphical representations of graphical IEC control
languages. (in the form of bolt on applications.)

A graphical representation is derived from canonical Atropos source and its
semantic model.

The initial graphical language targets are:

* Ladder Diagram
* Function Block Diagram
* Sequential Function Chart

A graphical renderer shall not invent executable behavior.

If source cannot be represented by a defined graphical language view,
the compiler or rendering tool shall report that limitation explicitly. (this
will not be an issue, the compiler will be text based purely and the rendering
tool will be part of the bolt on gui interface. this is going to be a viewer
only. translate the text into pretty pictures for the dummy engineers)

Atropos does not require every supported control language to be interchangeable
with every other control language.

Structured Text authored as Structured Text remains Structured Text.

Ladder Diagram authored as Ladder Diagram remains Ladder Diagram.

The common semantic model exists to support compilation and analysis, not to
imply lossless automatic conversion between fundamentally different source
languages. (not sure exactly what this means but there will be no loss because
there will be no conversion. the gui is a viewer)

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

* Exact top-level source grammar (we will work on this soon)
* Whether one source file may contain multiple Programs 
(i think potentially yes)
* Exact control language block delimiters (soon, all of these)
* Routine and control body invocation syntax
* Namespace or module support
* Global variable declaration organization
* Variable initialization syntax
* Retained and non-retained variable semantics
* Reference and pointer support (probably not, per IEC)
* Dynamic memory restrictions (per IEC)
* Function side-effect restrictions
* Cross-task variable access rules
* Function Block body organization
* Target-specific language extensions
* Source-level attributes and metadata
* Standard library organization
* Standard Function Block organization
(all of these need to conform to IEC)

These questions shall be resolved through specification development and
documented design decisions.

## 16. Normative Status

This document is an early draft.

Examples are non-normative unless explicitly identified otherwise.

The terms **shall**, **shall not**, **should**, **should not**, and **may**
are intended to develop normative meaning as the specification matures.

Atropos currently makes no claim of IEC 61131-3 conformance or certification.

