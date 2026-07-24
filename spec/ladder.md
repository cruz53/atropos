# Atropos Ladder Textual Representation — Working Draft

**Document status:** Working draft toward `spec/ladder.md`
**Revision:** 0.3 (2026-07-23)
**Supersedes:** ladder-exploration.md rev 0.1–0.2. The candidate-syntax
evaluation is complete; this document describes the adopted Atropos ladder
format. The rejected alternatives (instruction-stream, pure-expression, and
fully-structural forms) are of historical interest only and live in git
history.

---

## 0. Decision Record

| # | Question | Decision |
|---|----------|----------|
| D1 | Canonical ladder text format | **Adopted as specified in this document**: expression syntax for contact topology, element syntax for stateful/non-boolean/targeted constructs, joined by `->`. |
| D2 | Operator binding in contact expressions | **Power-flow binding: no operator precedence, strict left-to-right accumulation** (§4.1). A trailing series contact guards everything before it, matching graphical rung reading. No mandatory parenthesization. No whitespace significance; `;` terminates the rung. |
| D3 | Element naming | **IEC 61131-3 names** (`TON`, `R_TRIG`, `GT`, `MOV`, coil qualifiers `L`/`U`). Vendor mnemonics are not part of the language. A Logix-mnemonic *import* translator remains a possible future tool. |
| D4 | Rung structure | **One statement per rung** (§4.5). Possible future extensions noted there (`MULTIRUNG` grouping; a shortened, name-optional rung keyword) — recorded, not adopted. |
| D5 | Implementation language | **C++ from the start, kept for production** (§8). |

Still open, carried to normative `ladder.md`: multiple-coil-write policy
(§4.3), comparison element form, edge-contact sugar, rung comments,
action-element catalog (§7).

---

## 1. Design Principle

Ladder Diagram's information content is a two-dimensional topology: series
elements, parallel branches, power flow from left rail to right rail. Text
is one-dimensional, so a textual ladder format is an answer to the
question: *where does the topology go?*

The Atropos answer:

> **Encode topology (semantic). Refuse to encode layout (presentation).**

There is no layout information in Atropos ladder source — no grid
positions, no wire coordinates, no drawing metadata. The graphical
rendering is a pure, deterministic function of the parsed source: one
canonical picture per rung, no layout diffs, ever. The `gofmt` philosophy
applied to ladder.

Historical formats fail in one of two directions, and the Atropos format
is defined against both: *layout serialization* (the PLCopen XML failure —
unreadable, undiffable), and *grammar borrowing* (forcing ladder into an
instruction stream or a pure boolean-expression language, which either
hides the topology or cannot express timers, one-shots, and data moves
in-rung).

## 2. Format Rules

1. A **rung** is exactly one statement: a contact expression, one or more
   `->` targets, terminated by `;`. Whitespace and line breaks are
   insignificant everywhere; the semicolon ends the rung.

2. **Contact expression.** Series is `&`, parallel is `|`, a
   normally-closed contact is `!`, grouping is `(...)`. A bare identifier
   is a normally-open contact.

3. **Power-flow binding (no precedence).** The expression is read strictly
   left to right, exactly as a rung is read. Each operator combines the
   *entire structure built so far* with the next single term:
   - `<so-far> & X` — X in series with everything built so far.
   - `<so-far> | X` — X as a parallel branch across everything built so
     far.
   A "term" is one contact, one boolean-valued element, or one
   parenthesized group. **A parallel branch containing more than one
   element must be parenthesized.** Full treatment in §4.1.

4. **Targets** (right of `->`) are elements: coil `Var`, latching coil
   `Var(L)`, unlatching coil `Var(U)`, function block invocation
   `Inst:Type(params)`, and action elements (`MOV(Src, Dst)`-style).
   Multiple targets stack outputs on one power rail.

5. **Stateful and boolean-valued elements may appear in the series
   chain.** When reached, they execute; their boolean output (a function
   block contributes its `Q`) continues power flow. Combined with the
   no-short-circuit rule (§4.2), an in-rung timer executes every scan
   regardless of upstream contact state — the behavior a timer requires.

6. The contact-expression grammar is **closed**: contacts, boolean-valued
   elements, `& | !`, and parentheses only. No arithmetic, no assignment,
   no general expressions. Pressure to open it is pressure to write
   Structured Text, and ST exists.

7. **Rendering rule.** The graphical view is derived from the parse tree:
   left-operand accumulation renders as the upper/left structure, each
   `|` term renders as a branch below it, series order is left-to-right.

## 3. The Format Against Representative Logic

Eight canonical rungs covering the constructs that break naive textual
formats. Each is shown as the graphical rung and its Atropos source.
Graphical notation:

```text
--] [--    normally-open contact (examine if closed)
--]/[--    normally-closed contact (examine if open)
--( )--    output coil
--(L)--    latching coil (set)
--(U)--    unlatching coil (reset)
+------+
| BLOK |   function block / instruction block
+------+
```

### R1 — Seal-in motor start (classic 3-wire control)

```text
     StartPB      StopPB               MotorRun
----+---] [---+----]/[-------------------( )------
    |         |
    | MotorRun|
    +---] [---+
```

```text
RUNG MotorSeal
    (StartPB | MotorRun) & !StopPB -> MotorRun;
```

The parenthesized group is the parallel start/seal pair; `& !StopPB` puts
the stop contact in series after it; `->` drives the coil. The most common
rung in industry is one line, and the mapping is line-of-sight.

### R2 — Parallel branch with a series sub-condition

```text
     HighLevel                          FillValve
----+---] [------------------+------------( )------
    |                        |
    | ManualFill  FillEnable |
    +---] [---------] [------+
```

```text
RUNG FillControl
    HighLevel | (ManualFill & FillEnable) -> FillValve;
```

`| (…)` parallels the group across `HighLevel`. The branch keeps its
parentheses because it contains two elements (rule 3: the thing after `|`
is one term).

### R3 — Nested branch, series guard, two outputs

The `Fault` contact guards the **entire** parallel structure; two coils
are driven from the same power rail.

```text
     Auto      Sensor1                  Fault      Conveyor
----+--] [--+---] [---+--+---------------]/[---+-----( )------
    |       |         |  |                     |
    |       | Sensor2 |  |                     |  AlarmReset
    |       +---] [---+  |                     +-----( )------
    |                    |
    | Manual  Override   |
    +--] [------] [------+
```

```text
RUNG ConveyorPermissive
    Auto & (Sensor1 | Sensor2) | (Manual & Override) & !Fault
        -> Conveyor
        -> AlarmReset;
```

Read as power flow, left to right:

| Step | Source consumed | Structure built |
|---|---|---|
| 1 | `Auto` | contact |
| 2 | `& (Sensor1 \| Sensor2)` | sensor branch in series after Auto |
| 3 | `\| (Manual & Override)` | Manual·Override as a parallel branch across ALL of the above |
| 4 | `& !Fault` | Fault NC in series after the whole parallel structure |

No outer parentheses required: unadorned left-to-right reading places the
guard on the main rung, blocking everything — exactly as the picture
reads. This rung is the motivating case for decision D2.

### R4 — Comparison in-rung (non-boolean data entering ladder)

```text
    +--------------+     PumpRun        HighTempAlarm
----| GT           |-------] [--------------( )------
    | TankTemp     |
    | 80.0         |
    +--------------+
```

```text
RUNG HighTemp
    GT(TankTemp, 80.0) & PumpRun -> HighTempAlarm;
```

A boolean-valued IEC comparison element is a contact-like term in the
expression. (Function style vs an operator-island sugar such as
`[TankTemp > 80.0]` is open — §7.)

### R5 — Function block mid-rung (power flows through the block)

The single most important test case: **an in-rung TON must execute every
scan** or its timing state corrupts.

```text
    RunCmd     +--------------+           Motor
------] [------| TON  Timer1  |-------------( )------
               | PT:  T#3s    |
               +--------------+
```

```text
RUNG DelayedMotor
    RunCmd & Timer1:TON(PT := T#3s) -> Motor;
```

The FB invocation is a term in the series chain. When the scan reaches
this rung, `Timer1` executes with `IN` = power flow at its position; its
`Q` continues the chain. Per §4.2, `Timer1` executes every scan even while
`RunCmd` is false.

Under one-statement-per-rung the split idiom is two named rungs, each
independently addressable in diagnostics:

```text
RUNG AuxTimer
    MotorRun -> AuxDelay:TON(PT := T#3s);

RUNG AuxOut
    AuxDelay.Q -> AuxContactor;
```

### R6 — One-shot / rising-edge

```text
    FloatSw    Cycle_ONS              CycleStart
------] [------[R_TRIG]-----------------( )------
```

```text
RUNG CycleTrigger
    FloatSw & Cycle_ONS:R_TRIG() -> CycleStart;
```

Edge detection uses the IEC blocks `R_TRIG` (rising) and `F_TRIG`
(falling) as ordinary in-rung FB instances — no special grammar. A
contact-level sugar (e.g., `^Var`) remains open (§7).

### R7 — Latch / unlatch pair

```text
    OverTemp                           HeaterFault
------] [--------------------------------(L)------

    ResetPB     OverTemp               HeaterFault
------] [---------]/[--------------------(U)------
```

```text
RUNG FaultSet
    OverTemp -> HeaterFault(L);

RUNG FaultClear
    ResetPB & !OverTemp -> HeaterFault(U);
```

`(L)` sets on true power flow; `(U)` clears on true power flow; neither
acts on false — aligned to IEC set/reset coil semantics.

### R8 — Data move (no boolean output at all)

```text
    CalcDone   +------------------+
------] [------| MOV              |
               | Src: FlowTotal   |
               | Dst: HMI_Display |
               +------------------+
```

```text
RUNG PublishTotal
    CalcDone -> MOV(FlowTotal, HMI_Display);
```

A non-boolean action is a terminal element: it executes when power flow at
its `->` is true. It cannot appear in the series chain — nothing flows out
of it — and the grammar enforces that by construction.

---

## 4. Semantic Rules

### 4.1 Operator binding — power-flow binding (decided, D2)

**The hazard this rule eliminates.** Under conventional programming-language
precedence (`&` binding tighter than `|`), source like R3 written without
outer parentheses would attach a trailing `& !Fault` to only the last
branch, while its layout suggests it guards everything. Indentation that
lies about the parse is a wrong-motor-starts class of bug: a reviewer
reads the indentation, not a precedence table.

**The rule.**

> No operator precedence. The expression is consumed strictly left to
> right. Each `&` places the next term in series with the entire structure
> built so far; each `|` places the next term in parallel across the
> entire structure built so far. Parentheses create a nested sub-structure
> that is then treated as one term. Whitespace is insignificant; the rung
> terminates at `;`.

Both possible readings of the R3 shape, drawn — the format's rule selects
the first:

**Power-flow binding (Atropos) — `Fault` guards everything:**

```text
     Auto      Sensor1                  Fault      Conveyor
----+--] [--+---] [---+--+---------------]/[---------( )------
    |       |         |  |
    |       | Sensor2 |  |
    |       +---] [---+  |
    | Manual  Override   |
    +--] [------] [------+
```

**Conventional precedence (rejected) — `Fault` guards only one branch:**

```text
     Auto      Sensor1                             Conveyor
----+--] [--+---] [---+--+--------------------------( )------
    |       |         |  |
    |       | Sensor2 |  |
    |       +---] [---+  |
    |                    |
    | Manual  Override  Fault
    +--] [------] [------]/[--+
```

**Documented consequences, accepted:**

1. **Branch grouping.** `A | B & C` means `(A | B) & C` — C in series
   after the branch — not `A | (B & C)`. To put a multi-element series
   inside a branch, parenthesize the branch: `A | (B & C)`. Teaching rule:
   *the thing after `|` is one branch; if the branch has more than one
   element, wrap it.* The R3 walk-through table (§3) is the teaching
   example.
2. **Cross-language divergence.** The same character sequence would parse
   differently in Structured Text, which follows conventional precedence
   per IEC. Mitigation is deliberate surface separation: ladder uses the
   symbols `& | !`; ST uses the keywords `AND OR NOT`. `ladder.md` and
   `structured-text.md` shall each state this divergence explicitly, and
   the formatter/linter shall flag `AND`/`OR`/`NOT` keywords in a ladder
   contact expression as errors — never accept them as aliases.
3. **Formatter obligation.** The canonical formatter shall indent
   continuation lines to reflect power-flow accumulation so that layout
   and parse can never disagree in formatted source.

**Parser note.** Power-flow binding is *simpler* to implement than
conventional precedence — one flat left-associative loop instead of
stratified precedence levels (§6 grammar).

### 4.2 Short-circuit evaluation — no short-circuit (adopted)

Whether `Sensor2` is evaluated when `Sensor1` is already true is
unobservable for plain contacts. With a function block in a branch it is
the whole ballgame: a skipped TON accumulates wrong state. Rule:

> **No short-circuit.** Every element in a rung's expression is
> evaluated/executed on every scan in which the rung is reached.

Optimizers may skip evaluation only where skipping is provably
unobservable (pure contacts).

### 4.3 Multiple writes to one coil — OPEN

`Out` driven unqualified from two rungs: last-write-wins (mainstream OTE
behavior), OR-accumulate, or compile error? Real ladder bugs live here.
Working proposal: **last-write-wins with a mandatory compiler warning**,
suppressible per-coil only by an explicit source annotation so the intent
is visible in the diff. Decision deferred to normative `ladder.md`.

### 4.4 Element naming — IEC 61131-3 vocabulary (decided, D3)

Element vocabulary follows IEC 61131-3: standard function blocks (`TON`,
`TOF`, `TP`, `CTU`, `CTD`, `CTUD`, `R_TRIG`, `F_TRIG`, `RS`, `SR`),
standard functions for boolean-valued elements (`GT`, `GE`, `EQ`, `NE`,
`LE`, `LT`, …), `MOV`-class action elements enumerated against the IEC
standard function set, and coil qualifiers `(L)`/`(U)` aligned to IEC
set/reset coil semantics. Vendor mnemonics are not part of the language:
the concern they addressed was vendor IP exposure, which IEC vocabulary
resolves while strengthening the conformance path (language.md §2.3). A
Logix-mnemonic *import* translator for migrating existing programs remains
a possible future tool and is unaffected by this decision.

### 4.5 Rung structure — one statement per rung (decided, D4; revisit noted)

A `RUNG` contains exactly one statement. Multi-step logic is written as
multiple named rungs (R5 split form, §3). Rationale: several complex
multi-line statements under one rung name, differentiated only by
semicolons, would blur diagnostics, review, and the one-rung-one-picture
graphical mapping. `END_RUNG` is unnecessary and absent from the grammar;
`;` terminates the rung. Every drawn rung has a name, so diagnostics and
cross-references address rungs unambiguously.

**Recorded for possible revisit — not adopted:**

* **`MULTIRUNG` grouping.** An explicit construct for a named group of
  related rungs, preserving one-statement-per-rung inside it while giving
  the group a shared name/comment for organization and review:

  ```text
  MULTIRUNG AuxContactorControl
      RUNG AuxTimer
          MotorRun -> AuxDelay:TON(PT := T#3s);
      RUNG AuxOut
          AuxDelay.Q -> AuxContactor;
  END_MULTIRUNG
  ```

  Open sub-questions if pursued: does the group name appear in
  diagnostics alongside the rung name; does the graphical view render it
  as a titled region; may inner rungs be anonymous (below)?

* **Shortened, name-optional rung keyword.** For trivial rungs where
  naming is friction, a short form — e.g. `R` with an optional name —
  with the compiler assigning a stable auto-number for diagnostics:

  ```text
  R  AuxDelay.Q -> AuxContactor;          // anonymous; diagnostics: R#7
  R AuxOut: AuxDelay.Q -> AuxContactor;   // short form, still named
  ```

  Tension to resolve before adopting: auto-numbers are not stable across
  insertions, which degrades diffs and cross-references — the exact
  problems named rungs solve. A hybrid (anonymous allowed only inside a
  named `MULTIRUNG`) might bound the damage.

If field experience shows rung-name fatigue, these are the starting
points; multi-statement rungs under a single plain `RUNG` remain rejected.

---

## 5. (Reserved)

Section retained to keep cross-references from earlier revisions stable;
the candidate comparison matrix that occupied it is superseded and lives
in git history.

## 6. Draft Grammar (non-normative)

```ebnf
routine   = "LADDER" ident { rung } "END_LADDER" ;
rung      = "RUNG" ident stmt ;
stmt      = expr target { target } ";" ;
target    = "->" element ;
element   = fb_call | action | coil ;
coil      = ref [ "(" ("L" | "U") ")" ] ;
fb_call   = ident ":" ident "(" [ param { "," param } ] ")" ;
action    = ident "(" [ arg { "," arg } ] ")" ;      (* MOV etc.; catalog: §7 *)
param     = ident ":=" value ;
value     = time_lit | number | ref ;

(* power-flow binding: one flat left-associative chain, & and | equal *)
expr      = term { ("&" | "|") term } ;
term      = "!" term
          | "(" expr ")"
          | fb_call
          | bool_elem
          | ref ;
bool_elem = ident "(" [ arg { "," arg } ] ")" ;      (* GT(x, y) etc. *)
ref       = ident { "." ident } ;
time_lit  = ("T"|"t") "#" duration ;                 (* T#3s, T#500ms, T#1m30s *)
```

Known gaps for normative `ladder.md`: distinguishing `action` vs
`bool_elem` vs `fb_call` without whole-catalog lookahead (likely a
semantic rather than syntactic split — the standard-element catalog is in
the symbol table before parse-tree finalization); rung comments;
edge-contact sugar; catalog enumeration.

## 7. Remaining Decisions for Normative `ladder.md`

1. Multiple-coil-write policy (§4.3) — working proposal on the table.
2. Comparison element form: function style `GT(TankTemp, 80.0)` only, or
   an operator-island sugar `[TankTemp > 80.0]`.
3. Edge-contact sugar (e.g., `^Var` as an anonymous `R_TRIG`) — or always
   require explicit named instances.
4. Rung comment syntax, and whether comments attach to the rung as
   metadata (surviving round-trip to graphics) or remain plain source
   comments.
5. Action-element catalog scope for v0.1 (minimum viable: `MOV`; then
   `ADD`/`SUB`-class, or defer all arithmetic to ST).
6. `MULTIRUNG` and short-form rung keyword (§4.5) — parked unless
   rung-name fatigue materializes.

## 8. Implementation Note (D5)

The compiler is implemented in **C++ from the start** and that codebase is
kept for production; no intermediate prototype in another language.
Recorded rationale: maintainer C++ fluency is developing, and a
Python-first step would increase, not decrease, the difficulty of owning
the production codebase — a later rewrite would land on the maintainer
least served by it. Consequence accepted: slower early iteration while
semantics are fluid, offset by the eight canonical rungs in this document
serving as the fixed semantic target and the seed of the conformance test
suite (`tests/conformance/`). This decision and rationale shall be copied
to `docs/decisions/` when that directory is populated.
