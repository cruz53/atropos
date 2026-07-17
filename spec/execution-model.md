# Atropos Execution Model Specification

**Document status:** Draft  
**Specification version:** 0.1  
**Project status:** Pre-alpha

## 1. Scope

This document defines the execution model of an Atropos application.

It specifies how compiled Atropos application logic is initialized, scheduled,
invoked, executed, stopped, restarted, and faulted.

This specification defines execution semantics shared across supported targets.
It is intended to ensure that an Atropos application has consistent observable
behavior regardless of the compiler backend or runtime used to execute it.

This specification defines:

* Application lifecycle states
* Task types and task scheduling concepts
* Assignment of Programs to tasks
* Cyclic execution
* Program invocation
* Routine and control body invocation
* Function execution
* Function Block execution and persistent instance state
* I/O process image behavior
* Variable initialization and retention
* Task overruns
* Runtime faults
* Cross-task data access
* Determinism requirements

This specification does not define:

* A specific operating system scheduler
* A specific processor architecture
* A specific runtime implementation language
* A specific compiler implementation
* A specific hardware I/O system
* A specific fieldbus protocol
* Target-specific task limits
* Target-specific timing resolution
* Target-specific fault recovery mechanisms
* Safety certification requirements

Target-specific implementations may impose additional limits but shall preserve
the semantics defined by this specification unless an intentional target
restriction is documented.

## 2. Execution Principles

The Atropos execution model is designed according to the following principles.

### 2.1 Deterministic Behavior

Application execution shall be deterministic within the limits defined by the
Atropos language, task configuration, runtime configuration, and target
capabilities.

Given the same:

* Application source
* Compiler configuration
* Runtime configuration
* Initial application state
* Input values
* Input timing
* Task activation timing

an Atropos application should produce the same defined sequence of observable
results.

Behavior that is intentionally implementation-defined or target-dependent
shall be explicitly documented.

### 2.2 Explicit Execution

A Program Organization Unit shall not execute solely because it is declared.

Execution shall occur only when established by:

* Task assignment
* Program invocation
* Routine invocation
* Function invocation
* Function Block invocation
* Language-specific execution rules

Unreferenced declarations shall not execute.

### 2.3 Bounded Execution

The execution model shall support bounded and predictable use of runtime
resources.

Language features that may introduce unbounded execution time, unbounded memory
growth, or uncontrolled recursion shall be restricted or prohibited by the
applicable language specification.

### 2.4 Observable State

Runtime state required for operation, diagnosis, and online inspection should
remain observable through defined runtime interfaces.

Inspection of runtime state shall not alter application semantics unless the
user performs an explicit write, force, or control action.

Online inspection, modification, and forcing behavior shall be defined in a
separate online operations specification.

### 2.5 Target Independence

The execution semantics of an Atropos application shall not depend upon a
specific operating system, processor, or vendor runtime.

Compiler backends and runtimes may use different implementation mechanisms
provided they preserve the observable behavior required by this specification.

## 3. Application Lifecycle

An Atropos application proceeds through defined lifecycle states.

The initial lifecycle states are:

* Unloaded
* Loaded
* Initializing
* Stopped
* Running
* Faulted

Additional transitional states may be defined by the runtime specification.

### 3.1 Unloaded

In the Unloaded state, the application is not available for execution.

Application Programs, Functions, Function Blocks, tasks, and runtime data do
not have active execution state.

### 3.2 Loaded

In the Loaded state, the runtime has accepted the compiled application and its
configuration.

The runtime shall validate that required application resources are available
before the application may enter the Initializing state.

Loading an application shall not by itself begin Program execution.

### 3.3 Initializing

In the Initializing state, the runtime prepares application state for
execution.

Initialization may include:

* Allocating statically defined application storage
* Creating Function Block instance state
* Applying initial variable values
* Restoring retained values when permitted
* Initializing the I/O process image
* Validating task configuration
* Validating target resource mappings
* Running defined initialization procedures

Application task execution shall not begin until required initialization has
completed successfully.

If initialization fails, the application shall enter the Faulted state or
remain Stopped with a diagnostic, according to the severity of the failure.

### 3.4 Stopped

In the Stopped state, normal task execution does not occur.

Application state may remain available for:

* Inspection
* Diagnostics
* Controlled modification
* Download or replacement
* Restart preparation

The runtime shall define whether physical outputs are:

* Held at their last commanded values
* Set to configured safe values
* Released to target-specific control
* Handled according to another explicitly configured stop policy

The selected stop behavior shall not be implicit.

### 3.5 Running

In the Running state, configured tasks may activate and execute their assigned
Programs.

Task execution shall follow the rules defined by this specification and the
application task configuration.

### 3.6 Faulted

The application shall enter the Faulted state when a runtime condition prevents
continued execution according to the configured fault policy.

In the Faulted state:

* Normal task execution shall stop unless a limited recovery task is explicitly
  permitted
* The fault condition shall be recorded
* Relevant diagnostic information should be retained
* Output handling shall follow the configured fault-output policy
* Restart shall require an explicit runtime or user action unless automatic
  recovery is specifically configured

A fault shall not be silently ignored.

## 4. Startup and Restart Classes

Atropos shall distinguish between restart classes because initialization and
retained state behavior may differ.

The initial restart classes are:

* Cold start
* Warm start
* Controlled restart

The final names and exact IEC alignment remain subject to further specification
review.

### 4.1 Cold Start

A cold start initializes application state without relying upon retained values
from a previous execution session, except where a separately defined
nonvolatile configuration requires otherwise.

On a cold start:

* Non-retained variables shall receive their defined initial values
* Function Block instance state shall receive its defined initial values
* Retained variables may be reset according to the retention specification
* Tasks shall begin from their initial scheduling state
* Outputs shall remain under startup policy until initialization is complete

### 4.2 Warm Start

A warm start may restore variables declared as retained.

On a warm start:

* Retained variables shall be restored when valid retained data is available
* Non-retained variables shall receive their defined initial values
* Restored values shall be validated against the current application definition
* Incompatible retained data shall produce a diagnostic
* Function Block state shall be restored only where the applicable state is
  explicitly retained

The exact compatibility rules for retained data shall be defined separately.

### 4.3 Controlled Restart

A controlled restart occurs when the runtime stops and restarts an application
without treating the event as a complete loss of runtime state.

The retained and non-retained behavior of a controlled restart shall be
explicitly defined by the runtime configuration.

A controlled restart shall not silently preserve state that the application
declares non-retained.

## 5. Task Model

A task is a scheduling entity that causes one or more assigned Programs to
execute.

A task definition includes, as applicable:

* Task name
* Task type
* Activation condition
* Period
* Priority
* Assigned Programs
* Overrun policy
* Watchdog or execution-time limit
* Enable state

The core execution specification shall define task semantics independently of
the mechanism used by a target runtime to implement scheduling.

### 5.1 Task Types

The initial task types are:

* Cyclic task
* Periodic task
* Event task

Whether cyclic and periodic tasks remain distinct in the final language model
is an open design question.

### 5.2 Cyclic Task

A cyclic task executes repeatedly whenever the application is Running.

After completing its assigned Programs, the task becomes eligible to begin its
next cycle according to runtime scheduling rules.

A cyclic task does not require a fixed configured period.

The runtime shall prevent uncontrolled task execution from starving required
system services or higher-priority application tasks.

### 5.3 Periodic Task

A periodic task is activated at a configured time interval.

A periodic task definition shall include a period.

The runtime shall attempt to activate the task according to the configured
period and supported timing resolution.

The configured period represents the desired interval between task
activations, not a guarantee that execution completes within that interval.

### 5.4 Event Task

An event task is activated by a defined event source.

Potential event sources include:

* A change in a mapped input
* A communications event
* A runtime event
* A software-generated event
* A target-specific hardware event

Event sources and event qualification rules shall be explicit.

Event tasks shall not be triggered by undefined or hidden runtime behavior.

The initial Atropos implementation may defer support for event tasks.

### 5.5 Task Enable State

A task may be enabled or disabled according to application or runtime
configuration.

A disabled task shall not activate.

Disabling a task shall not automatically destroy its Program or Function Block
state unless explicitly defined by a reset operation.

### 5.6 Task Priority

Tasks may be assigned priorities.

The meaning and supported range of priorities may be target-specific, but the
relative ordering of configured application priorities shall be preserved where
the target supports preemptive scheduling.

The final priority and preemption model remains an open design question.

A target unable to preserve required task priority semantics shall reject the
application or report the unsupported configuration.

## 6. Program Assignment

A Program becomes executable through assignment to a task.

A task may contain one or more assigned Programs.

The order of Program execution within a task shall be explicitly defined by the
task configuration.

Unless otherwise specified, assigned Programs shall execute sequentially in
their configured order.

A Program shall not execute merely because it is declared in source.

### 6.1 Program Assignment Rules

The compiler or runtime shall validate that each executable Program has a valid
task assignment.

Whether one Program may be assigned to more than one task remains an open
design question.

If multiple task assignment is permitted, each assignment shall have clearly
defined state, reentrancy, and concurrency semantics.

Until those semantics are defined, a Program should be assigned to no more than
one task.

### 6.2 Program Invocation

When a task invokes a Program:

1. The Program begins at its defined entry body.
2. The Program executes its control language bodies and explicit invocations in
   defined order.
3. Called routines, Functions, and Function Blocks execute when reached.
4. Control returns to the calling location after a synchronous call completes.
5. Program execution completes when the end of the active invocation path is
   reached.

Declarations outside the active invocation path shall not execute.

### 6.3 Program Completion

A Program invocation completes when its main body and all synchronous calls
made from that body have completed.

Program completion does not destroy local persistent state or Function Block
instance state.

Temporary evaluation state may be released or reused after completion according
to the compiler and runtime implementation.

## 7. Task Execution Cycle

A task activation consists of a defined sequence of execution phases.

The default conceptual sequence is:

1. Activate the task.
2. Make the applicable input process image available.
3. Execute assigned Programs in configured order.
4. Commit task-visible output results according to the output policy.
5. Record task timing and diagnostic information.
6. Complete the task activation.

This conceptual sequence does not require a particular runtime implementation.

### 7.1 Input Availability

At the beginning of a task activation, the task shall observe input values
according to the configured I/O process image policy.

The default model should provide a stable input snapshot for the duration of a
task activation.

Immediate or direct input access may be supported separately and shall be
explicit in source or configuration.

### 7.2 Program Order

Programs assigned to the same task shall execute in configured order.

A later Program in the same task activation shall observe writes made by an
earlier Program to shared application variables unless an applicable buffering
rule explicitly states otherwise.

### 7.3 Output Publication

Output values written during Program execution shall be published according to
the configured output process image policy.

The default model should publish output process image values after the
applicable task or application scan completes.

Immediate or direct output access may be supported separately and shall be
explicit.

### 7.4 Task Completion

A task activation completes after:

* All assigned Programs complete
* Required output publication completes
* Required runtime accounting completes
* No unhandled execution fault remains

A periodic task that completes before its next activation shall remain inactive
until the next scheduled activation.

## 8. Invocation and Control Flow

Atropos control flow is based upon explicit execution paths.

Control flow may include:

* Sequential execution
* Conditional execution
* Iteration
* Routine calls
* Function calls
* Function Block calls
* Language-specific branch or network behavior
* SFC step and transition behavior

### 8.1 Sequential Execution

Within a sequential control body, executable elements shall be evaluated in
defined source order unless the language-specific specification defines another
ordering rule.

The physical location of unrelated declarations shall not affect execution.

### 8.2 Routine Invocation

A routine is an executable body associated with a Program or another permitted
Program Organization Unit.

A routine shall execute only when explicitly invoked or when designated as the
entry body of an invoked Program Organization Unit.

Routine invocation is synchronous unless an asynchronous execution mechanism
is explicitly defined by a later specification.

When a routine completes, execution returns to the caller.

### 8.3 Recursion

Direct and indirect recursion should be prohibited in the initial Atropos
language.

If recursion is introduced later, it shall require explicitly bounded stack and
execution behavior suitable for deterministic control systems.

The compiler should diagnose recursive call cycles.

### 8.4 Iteration

Loop constructs shall follow the syntax and semantics of the applicable control
language.

Loops whose maximum execution count cannot be statically or operationally
bounded may be restricted by compiler diagnostics, watchdog limits, or runtime
policy.

A loop shall not prevent task completion indefinitely without producing a
detectable overrun or watchdog condition.

## 9. Language-Specific Execution

Each supported control language shall define execution rules compatible with
the common Atropos execution model.

This section defines initial principles. Detailed language behavior shall be
specified in separate language documents.

### 9.1 Structured Text

Structured Text statements shall execute sequentially in source order except
where control flow explicitly changes that order.

Function and Function Block calls shall occur when their invocation expressions
are evaluated.

The final expression evaluation order shall be defined by the Structured Text
specification.

### 9.2 Ladder Diagram

Ladder Diagram logic shall execute in defined rung order.

Within a rung, evaluation shall follow the Ladder Diagram execution rules
defined by the Ladder specification.

Branch evaluation, short-circuit behavior, coil writes, and multiple writes to
the same variable shall be explicitly specified.

A Ladder routine shall complete after all reachable rungs in the routine have
been evaluated.

### 9.3 Function Block Diagram

Function Block Diagram execution order shall be determined from explicitly
defined network ordering and data dependencies.

A Function Block Diagram shall not rely upon an ambiguous visual arrangement
to determine execution semantics.

Where multiple valid evaluation orders exist, the language shall define a
deterministic ordering rule or require the source to provide explicit ordering.

### 9.4 Sequential Function Chart

Sequential Function Chart execution shall be based upon active steps,
transitions, and actions.

The SFC specification shall define:

* Initial steps
* Transition evaluation
* Simultaneous transition behavior
* Step activation and deactivation
* Action qualifiers
* Action execution order
* Conflict resolution
* Interaction with the task scan

An SFC body shall execute as part of the task and Program invocation that
contains it.

## 10. Functions

A Function is a stateless synchronous computation.

A Function invocation shall:

1. Evaluate its input arguments according to defined expression-order rules.
2. Create or assign invocation-local variables.
3. Execute the Function body.
4. Produce the defined return value.
5. Discard invocation-local execution state after completion.

A Function shall not retain internal state between invocations.

A Function shall not depend upon hidden mutable state.

The exact restrictions on access to global variables and side effects remain to
be defined by the Function specification and IEC alignment review.

A Function invocation shall complete before execution continues at the calling
location.

## 11. Function Blocks

A Function Block is a stateful synchronous Program Organization Unit.

Each declared Function Block instance shall have independent instance state.

### 11.1 Instance Identity

Function Block state belongs to a specific declared instance.

Two instances of the same Function Block type shall not share instance state
unless they explicitly reference shared global data.

Compiler-generated internal identifiers shall not replace user-defined instance
names in diagnostics or online inspection.

### 11.2 Invocation

When a Function Block instance is invoked:

1. Input parameters are made available to the instance.
2. The Function Block body executes.
3. Internal instance state may be read and modified.
4. Output parameters are updated.
5. Instance state remains available for the next invocation.

The precise timing of input assignment and output visibility shall be defined by
the applicable control language.

### 11.3 Persistent State

Function Block instance state shall persist between invocations while the
application remains loaded.

State retention across stops, restarts, downloads, or power loss shall depend
upon explicit retention declarations and restart policy.

### 11.4 Multiple Invocations

A Function Block instance may be invoked more than once during a task
activation unless prohibited by the applicable language specification.

Each invocation operates on the same instance state.

Programmers shall be responsible for the intentional effects of multiple
invocations of the same instance within one scan.

Compiler diagnostics may warn about multiple invocations where behavior is
likely to be unintended.

### 11.5 Concurrent Invocation

Concurrent invocation of the same Function Block instance by multiple tasks
shall be prohibited unless explicit concurrency semantics are later defined.

The compiler or runtime should reject an application that permits unsafe
concurrent invocation of one instance.

## 12. Variables and Runtime State

Runtime variables shall have defined storage duration, initialization behavior,
scope, and retention behavior.

The type system specification shall define representation and value ranges.

### 12.1 Local Variables

Local variables belong to their declaring Program Organization Unit or language
construct.

The storage duration of a local variable depends upon its declaration category.

The language shall distinguish between:

* Invocation-local temporary variables
* Program-local persistent variables
* Function Block instance variables

The exact declaration syntax shall be defined separately.

### 12.2 Global Variables

Global variables are available according to the scope rules of the language
specification.

Writes to global variables take effect according to the execution ordering and
cross-task access rules of this specification.

A global variable shall not imply atomic access for every type or target.

### 12.3 Initial Values

A variable may have a defined initial value.

If no explicit initial value is provided, the applicable type or declaration
specification shall define the default value.

Initialization shall occur before normal task execution begins.

### 12.4 Retained Variables

A retained variable is eligible to preserve its value across defined restart
conditions.

Retention shall be explicit.

A variable shall not be treated as retained solely because a target runtime
happens to preserve its memory.

The retention specification shall define:

* Eligible variable categories
* Storage and restoration behavior
* Data validation
* Application version compatibility
* Failure handling
* Reset behavior

### 12.5 Non-Retained Variables

A non-retained variable shall receive its defined initial value whenever the
applicable restart class requires reinitialization.

Runtime implementations shall not expose accidental memory persistence as
defined application behavior.

## 13. I/O Process Image

Atropos shall support a process image model that separates application logic
from target-specific physical I/O access.

The process image consists conceptually of:

* Input process image
* Output process image

### 13.1 Input Process Image

The input process image contains values sampled from configured input sources.

The default model shall provide stable values during the applicable task or
application scan.

The exact sampling boundary remains an open design question where multiple
tasks execute at different periods.

### 13.2 Output Process Image

The output process image contains application-commanded output values.

Application writes shall update the output process image according to defined
execution order.

Physical output publication shall occur at a defined boundary.

### 13.3 Direct I/O Access

Atropos may support direct or immediate I/O access for applications that
require it.

Direct I/O access shall be explicit and distinguishable from normal process
image access.

The timing, atomicity, failure behavior, and portability limitations of direct
I/O access shall be documented.

### 13.4 I/O Mapping

Mapping between Atropos variables and physical or communications I/O is
target-specific configuration.

I/O mapping shall not change the core semantic type or scope of the mapped
variable.

Invalid or unavailable mappings shall produce diagnostics.

### 13.5 I/O Failure

The runtime shall define behavior for:

* Missing I/O devices
* Stale communications data
* Invalid input quality
* Output write failure
* Partial I/O update
* Communications timeout

Quality and fault information should be available to application logic through
defined mechanisms.

The core language shall not silently substitute valid-looking values for failed
I/O without an explicit policy.

## 14. Task Timing

The runtime shall measure or otherwise determine task timing sufficiently to
detect configured timing violations.

Relevant timing values may include:

* Activation time
* Start time
* Completion time
* Execution duration
* Jitter
* Missed activation count
* Overrun count

The supported timing resolution may be target-specific.

### 14.1 Period

A periodic task period defines its requested activation interval.

The period shall be greater than zero and representable by the target.

A task configuration that requests unsupported timing shall be rejected or
require an explicit target-specific adjustment.

### 14.2 Deadline

A task may define an execution deadline or watchdog time.

If task execution exceeds the configured limit, the runtime shall invoke the
configured overrun or watchdog policy.

### 14.3 Jitter

Task activation jitter is the difference between requested and actual
activation timing.

Targets may differ in achievable jitter.

A runtime should make measured jitter available for diagnostics where
practical.

### 14.4 Task Overrun

A task overrun occurs when a task has not completed before a timing boundary
defined by its configuration.

The application shall define or inherit an explicit overrun policy.

Potential policies include:

* Record a warning and continue
* Skip the next activation
* Queue one pending activation
* Prevent overlapping activations
* Stop the affected task
* Fault the application

The initial Atropos model should prohibit overlapping activations of the same
task.

An overrun shall not silently create concurrent execution of a Program or
Function Block instance.

## 15. Task Scheduling and Concurrency

The final Atropos scheduling model remains under development.

The execution model shall nevertheless preserve deterministic application
semantics.

### 15.1 Non-Preemptive Scheduling

A runtime may execute tasks non-preemptively.

Under non-preemptive scheduling, an active task runs until completion or fault
before another application task begins.

Task priority determines which ready task executes next.

### 15.2 Preemptive Scheduling

A runtime may support preemptive task scheduling where required.

Under preemptive scheduling, a higher-priority task may interrupt a
lower-priority task.

Preemption points, shared-data visibility, and atomicity requirements shall be
defined before preemptive scheduling becomes normative.

### 15.3 Cross-Task Variable Access

Multiple tasks may access the same global variable only according to defined
cross-task access rules.

Potential hazards include:

* Read-write races
* Write-write races
* Partial updates
* Inconsistent multi-variable state
* Priority-dependent behavior

The compiler and static analysis tools should identify potentially unsafe
cross-task access.

The final design shall define whether unsafe access is:

* Prohibited
* Allowed with warnings
* Allowed only through synchronization constructs
* Defined through a deterministic scheduling model

### 15.4 Atomicity

The language specification shall define which variable operations are atomic.

Atomicity may depend upon:

* Data type
* Data width
* Target architecture
* Runtime implementation

Target-dependent atomicity shall not be assumed by portable application source.

Explicit atomic or synchronized access mechanisms may be defined later.

## 16. Runtime Faults

A runtime fault is a condition that prevents correct continuation of defined
application execution.

Potential runtime faults include:

* Division by zero
* Invalid array index
* Invalid reference access
* Arithmetic overflow where trapping is required
* Invalid type conversion
* Task watchdog expiration
* Task overrun
* Stack exhaustion
* Internal runtime failure
* Required I/O mapping failure
* Corrupt retained data
* Unrecoverable communications failure
* Explicit application fault request

### 16.1 Fault Detection

The compiler should detect faults statically where possible.

The runtime shall detect defined runtime faults that cannot be resolved during
compilation.

### 16.2 Fault Severity

Faults may be classified by severity.

Potential severity classes include:

* Diagnostic
* Warning
* Recoverable task fault
* Application fault
* Runtime system fault

The final diagnostic and fault classification system shall be defined
separately.

### 16.3 Fault Response

Each runtime fault category shall have a defined response.

A response may include:

* Record a diagnostic
* Substitute a defined result
* Abort the current operation
* Stop the current Program
* Stop the current task
* Fault the application
* Enter a target-defined safe state

Undefined continuation after a detected fault shall not be permitted.

### 16.4 Output Behavior During Faults

Output handling during a fault shall follow an explicit configured policy.

Potential policies include:

* Hold last commanded value
* Apply configured safe values
* Clear output process image
* Transfer output authority to a separate safety or supervisory system

The core execution model shall not assume that one output policy is safe for
all industrial applications.

### 16.5 Fault Recovery

Recovery from a fault shall require a defined action.

Potential actions include:

* Automatic retry for a specifically recoverable condition
* Task reset
* Application reset
* Warm restart
* Cold restart
* Application download
* Runtime restart

Fault recovery shall not erase diagnostic evidence before it can be inspected,
unless required by resource limitations or an explicit reset action.

## 17. Stop and Shutdown Behavior

Stopping an Atropos application shall be an explicit runtime transition.

When a stop is requested:

1. The runtime prevents new normal task activations.
2. The runtime handles any currently executing task according to the configured
   stop policy.
3. Output values are handled according to the configured stop-output policy.
4. Required retained state is committed.
5. The application enters the Stopped state.

The stop policy shall define whether an active task:

* Completes its current activation
* Stops at a defined safe boundary
* Is aborted immediately

Immediate abortion may leave partially updated application state and therefore
shall not be the default unless required by the target or fault condition.

## 18. Downloads and Application Replacement

Replacing a loaded application shall follow a defined download and activation
procedure.

The runtime shall validate the new application before execution.

The replacement procedure shall define:

* Whether the current application must stop
* Whether online replacement is supported
* Which values may be preserved
* Function Block instance compatibility
* Retained data compatibility
* Task configuration changes
* I/O mapping changes
* Rollback behavior after activation failure

The initial Atropos implementation may require a full application stop and
restart for every download.

Online edits and partial downloads shall be specified separately and shall not
be assumed by this document.

## 19. Debugging and Online Inspection

Debugging and online inspection shall observe the execution model defined by
this specification.

A debugger may provide:

* Task state
* Program state
* Current variable values
* Function Block instance state
* Active SFC steps
* Task timing
* Fault history
* Source-level execution location

Breakpoints, single stepping, forcing, and online writes may intentionally alter
normal timing or execution.

Such operations shall be explicitly indicated by the runtime.

A debugging operation shall not be represented as normal deterministic
application behavior.

Detailed debugging semantics shall be defined in a separate specification.

## 20. Determinism and Portability Requirements

A conforming compiler backend and runtime shall preserve the defined observable
behavior of an Atropos application.

Observable behavior includes:

* Program invocation order
* Routine invocation order
* Variable read and write ordering
* Function results
* Function Block state transitions
* Task activation semantics
* Process image behavior
* Defined fault behavior
* Retention and initialization behavior

A backend shall not silently:

* Reorder dependent operations
* Remove observable writes
* Combine task activations
* Introduce concurrent execution
* Change initialization values
* Change retention behavior
* Change fault response
* Replace process image access with direct I/O access

Optimizations are permitted only when they preserve defined observable
behavior.

Target restrictions that prevent semantic preservation shall be reported
before application execution.

## 21. Implementation Freedom

This specification defines observable execution semantics rather than internal
runtime architecture.

A conforming implementation may use:

* Native machine code
* Generated C or another intermediate language
* An interpreter
* A virtual machine
* Operating system threads
* A single-threaded scheduler
* Hardware timer facilities
* Software timer facilities

The selected implementation shall preserve the behavior required by this
specification.

Internal implementation details shall not become accidental language semantics.

## 22. Open Questions

The following execution-model questions remain unresolved:

* Final lifecycle state names and transitions
* Exact IEC alignment of cold, warm, and controlled restarts
* Whether cyclic and periodic tasks are distinct task types
* Whether event tasks are included in the first implementation
* Whether task scheduling is initially preemptive or non-preemptive
* Task priority representation
* Whether a Program may be assigned to more than one task
* Exact process image sampling boundary with multiple tasks
* Exact output publication boundary with multiple tasks
* Whether shared global writes become visible immediately across tasks
* Atomicity guarantees for elementary and structured types
* Required synchronization mechanisms
* Exact Function restrictions on global reads and writes
* Loop-bounding requirements
* Multiple invocation diagnostics for Function Block instances
* Overrun policy defaults
* Fault severity classes
* Default stop-output policy
* Default fault-output policy
* Retained data compatibility across application versions
* Online download and online-edit semantics
* Debugger interaction with deterministic task timing
* Runtime behavior when physical I/O becomes unavailable

These questions shall be resolved through specification development and
documented design decisions.

Where an unresolved question corresponds to an IEC 61131-3 requirement or
restriction, the applicable IEC behavior shall be the starting point for the
Atropos design unless an intentional deviation is documented.

## 23. Normative Status

This document is an early draft.

Examples and conceptual sequences are non-normative unless explicitly
identified otherwise.

The terms **shall**, **shall not**, **should**, **should not**, and **may** are
intended to develop normative meaning as the specification matures.

Atropos currently makes no claim of IEC 61131-3 conformance or certification.

