# Topic 1 Lab — Part C: Project Scoping Template

> **Save as:** `docs/week1_project_scope.md`
> **Student:** _[fill in your name and student ID]_
> **Repository:** https://github.com/thebalanceder/HarnessOrchestrationForCLM
> **Date:** 29 October 2026

---

## Project Name

**HarnessOrchestrationForCLM** — a datasheet-grounded geometry harness for a Vayo
Component Library Manager (CLM) instance.

## Domain

Electronic Design Automation (EDA) — component library management and PCB
footprint/3D-package data. Specifically: the boundary between a manufacturer
datasheet and the component library that a PCB design team draws from.

## Problem Statement

Before a component can be placed on a PCB, the design team needs a *package
model* for it — the body outline, pad geometry, pin count and pin numbering —
stored in the component library. In a Vayo CLM deployment that model is entered
by a human who opens the manufacturer datasheet, finds the mechanical drawing
table, reads the dimensions, and types them into the library's Package Editor.

That step is manual, unschema'd, and unverifiable. The library's native geometry
is stored as an opaque encoded blob, so a wrong dimension cannot be diffed, only
re-rendered and eyeballed. At the same time, large libraries accumulate
near-duplicate records, which means one bad entry propagates silently across
every part that reuses it.

AI engineering makes it possible to ground the extraction in the datasheet
itself, generate the library's text geometry from the drawing table, and place
deterministic validation gates plus a human review step between the generated
text and the library — turning a manual typing task into an auditable pipeline
that never writes to the library without review.

## Evidence of the Problem

### Evidence 1 — The library is large, duplicated, and mostly lacks pin metadata

Measured directly on a local Vayo CLM v6.0.1 instance (`VayoCompLib` database,
SQL Server, 29 October 2026):

| Measurement | Value |
| --- | --- |
| Package records (`PKG`) | 282,920 |
| Distinct package names | 28,591 |
| Mean records per distinct name | **9.9** |
| Packages carrying pin metadata (`PIN` rows) | 28,470 (**10.1%**) |
| `PIN` rows in total | 644,742 |
| Manufacturer part numbers (`MPN`) | 276,199 |
| MPN ↔ package links | 201,152 |
| Packages with **no** MPN link at all | 81,768 (**28.9%**) |
| Component types / subtypes in the catalogue | 36 / 749 |

Two facts follow. First, roughly ten records share every package name, so a
single incorrect geometry is not one bad row — it is a defect replicated across
an average of ten rows. Second, only one package in ten carries any pin-level
metadata, which is the data a reviewer would need in order to check that a
generated pin layout is right.

**Source:** first-party measurement, local Vayo CLM v6.0.1.1294 instance,
`VayoCompLib`, 29 Oct 2026. Query set retained in the project log. This is
first-party evidence and is not independently reproducible outside that
machine, so it is stated with its provenance rather than as a general industry
claim.

### Evidence 2 — The library exposes no machine-readable geometry format

The same instance stores package geometry in `PKG.GRAPHICS_OUTLINE`,
`GRAPHICS_PINS` and `GRAPHICS_POLARITY` as encoded `$V10…` blobs. Sampling a
record returns a string of the form:

```
$V10FDE13C88591D080602C100C85C39190C21020188B8670F8844A2B108C8B2111C…
```

There is no published schema for this encoding, and no text or JSON interchange
format is documented. The only human-writable representation is a plain-text
grammar reachable **only through the application's own Package Editor**
(`To Text` / `To Graphic`), whose keywords are `BODY:`, `BODYEXT:`,
`BODY_TOL:`, `COMP_DATA_EXT:`, `RECTCORNER:`, `PLANE:`, `PLANE_TOL:`,
`OUTLINE:`, `PADSYMBOL*` and `PINDEF:`. The vendor's bulk transfer tool
(`ComponentLibExportImport.exe`) accepts only its own `.clm` database-to-database
format, and its own documentation notes that if the part number and manufacturer
already exist the record will not be updated.

**Source:** first-party inspection of the installed application: `PKG` table
contents, the application's `Language\English.lib` resource file (INI-format
text listing the `TextGraphic_Form` caption `To Text`, the `Select File` button
and the `Package Editor` dialog), and the bundled `ComponentLibExportImport`
tool documentation.

### Why these two together matter

Evidence 1 says the defect surface is large and duplicated. Evidence 2 says the
defect is invisible: no schema, no diff, no bulk update path. Manual entry is
therefore both the bottleneck *and* the only audit surface. That combination —
high volume, no format contract, no automated write path — is the specific
engineering problem this project addresses.

## Initial Related Work (at least 3)

### 1. DocEDA — LLM extraction of parameters from circuit documents (2024)

Chen, H., Cai, H., Wu, L., & Gao, M. (2024). *DocEDA: Automated Extraction and
Design of Analog Circuits from Documents with Large Language Model.*

**Key limitation:** DocEDA targets *electrical* parameters (device values,
connectivity) in order to generate a circuit netlist. It does not address
mechanical/package geometry, does not handle the ambiguity between similarly
named drawing dimensions, and produces output for an open simulation format
rather than a closed vendor encoding.

### 2. Evaluation of LLM table understanding in scientific documents (2024)

*How Well Do Large Language Models Understand Tables in Materials Science?*
(2024). Integrating Materials and Manufacturing Innovation.
https://link.springer.com/article/10.1007/s40192-024-00362-6

**Key limitation:** the study shows LLM extraction accuracy is sensitive to
table structure and input format, and degrades on complex tables. Datasheet
mechanical tables are a worst case for this: merged header cells, a
`min / typ / max` column triad, and a separate unit row. The paper measures the
weakness but does not propose a validation layer that would catch it in
production.

### 3. Commercial CAD-model services — SnapEDA, Ultra Librarian

https://www.snapeda.com/ · https://www.ultralibrarian.com/

**Key limitation:** these are coverage businesses. They supply pre-built models
in *their* supported output formats (Altium, KiCad, Allegro, …), not in a Vayo
CLM text grammar, and they cannot repair a wrong record that already exists
inside a customer's library. Access is per-part or subscription-based, so a
library with 282,920 package records cannot be systematically re-derived this
way.

### 4. KiCad official footprint library (open, version-controlled)

https://gitlab.com/kicad/libraries/kicad-footprints

**Key limitation:** this is the closest existing example of "geometry as
reviewable text under version control", and it demonstrates that the goal is
achievable. But it is human-authored against KiCad's own conventions, contains
no automated datasheet-to-footprint path, and its naming and pad conventions do
not match the target library's.

### 5. The vendor's own bulk transfer tool (baseline / negative result)

Vayo `ComponentLibExportImport.exe`.

**Key limitation:** it moves records between two CLM databases and refuses to
update an existing part-number/manufacturer pair. It therefore cannot *create*
geometry from a datasheet, and it cannot *correct* geometry already in place —
which is precisely the two operations this project needs.

## Engineering Gap / Limitation

The gap is not "can an LLM read a datasheet". Current work already shows it can,
with caveats. The gap is that **no existing approach closes the loop into a
closed, schema-less, vendor-specific geometry format with a correctness gate
that is independent of the model.**

Three specific unresolved limitations:

1. **No format contract.** The target grammar has no published schema and no
   validator. Correctness cannot be established by construction, only by
   round-tripping through the vendor GUI and visually comparing the rendered
   package. Any harness must therefore treat the vendor application as the
   oracle and design its validation around that.
2. **Symbol ambiguity inside the source document.** Datasheet mechanical tables
   routinely define several similar symbols — `E` (overall span including
   leads) versus `E1` (body width) is the canonical trap. Both are plausible
   "width" values, both are in millimetres, and choosing the wrong one produces
   a structurally valid but physically wrong package. A confidence score does
   not catch this; only an explicit disambiguation step does.
3. **Silent propagation.** Because ~10 records share each package name, a
   generated-but-wrong geometry does not fail loudly. It fails at assembly.

## Initial Project Objectives (2–3)

**O1: To design** a datasheet-grounded extraction and generation harness that
converts a datasheet's mechanical drawing table into the target library's
plain-text geometry grammar, with explicit handling of unit normalisation and
of ambiguous dimension symbols.

**O2: To develop** the orchestration layer around that harness — worklist
intake, deterministic binding of each datasheet to its component row, agent
execution, validation gates that run *outside* the model, a human review step,
and an audit trail recording what was generated, what was checked, and who
approved it.

**O3: To evaluate** the harness on three axes: (a) geometric agreement with a
human-verified reference package, (b) validity of the generated text on
round-trip through the vendor application, and (c) reviewer effort and
automation-bias risk, measured as the proportion of critical defects a reviewer
actually catches when presented with plausible-looking output.

## Target Users

- **Primary:** PCB library engineers / component librarians who currently key
  datasheet dimensions into CLM by hand, and who are accountable for library
  correctness.
- **Secondary:** hardware design engineers who consume the library and bear the
  cost of a bad footprint at assembly or in the field.
- **Tertiary:** the reviewer/approver role in the quality process, who needs an
  audit trail showing that a generated package was checked against its source
  document rather than merely accepted.

## Data Sources (at least 2)

| # | Source | URL / identifier |
| --- | --- | --- |
| 1 | STMicroelectronics **M24M02-A125** datasheet, DocID027947 Rev 3, 37 pp. — used as the reference case; §9.1 (p. 34) carries the SO8N outline and mechanical table | `st.com` product page for M24M02-A125; document ID DocID027947 Rev 3. A local copy is held for course use only and is **not** committed to this repository. |
| 2 | **KiCad official footprint library** — open, version-controlled reference geometry for cross-checking land-pattern conventions | https://gitlab.com/kicad/libraries/kicad-footprints |
| 3 | **Vayo CLM documentation** — vendor documentation for the library application and its geometry text grammar | https://www.vayoinfo.com/ |
| 4 | Local **Vayo CLM v6.0.1 catalogue export** — the worklist of components needing geometry (internal, not public; not committed) | internal instance, `VayoCompLib` |

The datasheet is the primary ground truth. It is deliberately kept out of the
repository: it is a copyrighted vendor document that cannot be redistributed,
and the lab's GitHub safety rule prohibits committing such material.

## Similar to EduAdvisor

- Same architecture class: a **document-grounded retrieval and generation
  system** over a bounded corpus of authoritative PDFs.
- Same failure family: **hallucination** is the headline risk — an invented
  value is more dangerous than a missing one, because it looks like an answer.
- Same evaluation shape: quality is judged by comparison against an
  authoritative source, not by whether the output "looks reasonable".
- Same need for **provenance**: the user must be able to see which document and
  which table a value came from.

## Different from EduAdvisor

- **The output is not prose, it is geometry.** A wrong answer here does not
  mislead a reader; it produces a physical part that does not fit. The cost of
  error is shifted from "user is misinformed" to "board is scrap".
- **The output format is closed and schema-less.** EduAdvisor answers in natural
  language. This project must emit a vendor-specific text grammar with no
  published validator, so correctness has to be established by round-tripping
  through the vendor application.
- **Human-in-the-loop is mandatory, not optional.** EduAdvisor can answer
  directly. Here the agent is explicitly forbidden from writing to the library;
  a reviewer places the final artefact. The design problem is therefore partly
  a *review-workflow* problem.
- **Ambiguity is structural, not conversational.** EduAdvisor's ambiguity is in
  the user's question. Here it is inside the source document — two similarly
  named symbols, both plausible, both in the same unit.
- **Evaluation is deterministic where it can be.** Because a human-verified
  reference package exists, part of the evaluation is a byte-level diff rather
  than a rubric.

## Responsible AI Risks (at least 2)

**Risk 1 — Safety: hallucinated or misread dimension produces a wrong package.**
Who is harmed: the design team (rework, scrap), and downstream the end user of
the product if the defect escapes to production. A single transposed value
(`E` read where `E1` was meant) yields a package that is structurally valid,
renders plausibly, and is physically wrong. This is the highest-severity risk
in the project and the reason the agent is barred from writing to the library.

**Risk 2 — Automation bias: the human gate becomes theatre.**
Who is harmed: everyone downstream. If generated output is well-formatted and
confident, a reviewer under time pressure will approve it without re-deriving
the numbers from the datasheet. The control that is supposed to catch Risk 1
degrades into a rubber stamp. Mitigation to be designed and *measured* in O3(c):
force the reviewer to confirm specific values rather than a single "looks fine"
checkbox, and record which values were confirmed.

**Risk 3 — Data provenance and IP.**
Who is harmed: the institution and the student, legally; the vendor, commercially.
Datasheets are copyrighted and cannot be redistributed; customer bills of
materials and library contents may be confidential. Committing a datasheet or an
internal catalogue export to a public repository would breach both the vendor's
licence and the lab's GitHub safety rule. Mitigation: the repository stores
documents' *identifiers and derived measurements*, never the documents
themselves; `.gitignore` is configured accordingly.

**Risk 4 — Transparency: an opaque format defeats review.**
Who is harmed: the reviewer, and by extension the quality process. Because
native geometry is an encoded blob, a reviewer cannot see *what changed* between
two versions — only two rendered pictures. Mitigation: review the generated
plain-text geometry, which is diffable, rather than the encoded record; and keep
the generated text in the audit trail.

---

## Scope Boundary for Week 1

Stated explicitly so the scope does not drift:

**In scope for this project:** datasheet → package geometry text for
lead-frame packages (SOIC/SOP-class first), with a review gate and an audit
trail.

**Out of scope:** writing directly to the vendor database; automatic approval;
symbol/footprint pairing beyond the package body; 3D model mesh generation;
any high-risk domain named in the lab's avoid-list.

**Not claimed:** that the target library is missing 3D data. Measured on the
same instance, **all 282,920 package records already carry populated geometry
blobs** (0 empty outlines, 0 empty pin graphics), so the existing 3D viewer
already renders every record. The problem addressed here is the *creation and
correction* of that geometry from source documents — not the absence of a
renderer. Stating this up front prevents the project from being sold as
something it is not.
