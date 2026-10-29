# HarnessOrchestrationForCLM

> A datasheet-grounded harness that generates and validates component package
> geometry for a Vayo Component Library Manager (CLM) instance, with a mandatory
> human review gate.

**Student:** CHEK CHEE HIM (A24AI0021)
**Course:** AI Engineering — Topic 1 Lab
**Repository role:** this repository is the engineering record for the semester
project. Requirements, architecture, data documentation, source code, prompts,
evaluation evidence and governance notes accumulate here week by week.

---

## Problem Statement

Before a component can be used in a PCB design, its package model — body
outline, pad geometry, pin count and pin numbering — must exist in the component
library. In a Vayo CLM deployment that model is entered by a human who reads the
manufacturer datasheet's mechanical drawing table and types the dimensions into
the library's Package Editor. That step is manual and unverifiable: the library
stores geometry as an opaque encoded blob with no published schema, no diff and
no bulk update path, while the library itself holds an average of ~10 records
per distinct package name, so one wrong entry is replicated silently across
every part that reuses it.

AI engineering makes it possible to ground the extraction in the datasheet
itself, generate the library's text geometry deterministically, and place
validation gates plus a human review step between the generated text and the
library — turning a manual typing task into an auditable pipeline.

## Evidence of the Problem

- **The defect surface is large and duplicated.** Measured on a local Vayo CLM
  v6.0.1 instance (`VayoCompLib`, 29 Oct 2026): 282,920 package records but only
  28,591 distinct package names (9.9 records per name); only 28,470 packages
  (10.1%) carry any pin metadata across 644,742 `PIN` rows; 81,768 packages
  (28.9%) have no manufacturer part number linked at all. *(Source: first-party
  measurement of the local instance; query set retained in the project log.)*

- **The geometry format is closed, so a defect cannot be diffed.** Package
  geometry is stored as `$V10…` encoded blobs in `PKG.GRAPHICS_OUTLINE` /
  `GRAPHICS_PINS` / `GRAPHICS_POLARITY` with no published schema. The only
  human-writable form is a plain-text grammar (`BODY:`, `PINDEF:`,
  `PADSYMBOL*`, …) reachable only through the vendor's Package Editor, and the
  vendor's bulk tool (`ComponentLibExportImport.exe`) accepts only its own
  `.clm` database format and will not update an existing part. *(Source:
  first-party inspection of the installed application and its bundled
  documentation.)*

## Initial Related Work / Existing Approaches

- **DocEDA (Chen et al., 2024)** — LLM extraction of parameters from circuit
  documents to drive analog design. *Key limitation:* targets electrical
  parameters for netlist generation, not package mechanical geometry, and emits
  to an open format rather than a closed vendor encoding.

- **LLM table understanding in scientific documents (2024)** — *How Well Do
  Large Language Models Understand Tables in Materials Science?*, Integrating
  Materials and Manufacturing Innovation.
  https://link.springer.com/article/10.1007/s40192-024-00362-6
  *Key limitation:* accuracy is sensitive to table structure and degrades on
  complex tables — datasheet mechanical tables (merged headers, min/typ/max
  columns, separate unit row) are a worst case — and the work proposes no
  validation layer to catch that in production.

- **Commercial CAD-model services (SnapEDA, Ultra Librarian)** —
  https://www.snapeda.com/ · https://www.ultralibrarian.com/
  *Key limitation:* coverage businesses supplying pre-built models in their own
  supported formats, not the target library's grammar; they cannot repair a
  wrong record already inside a customer library, and per-part access does not
  scale to 282,920 records.

- **KiCad official footprint library** —
  https://gitlab.com/kicad/libraries/kicad-footprints
  *Key limitation:* proves the goal (geometry as reviewable text under version
  control) but is human-authored against KiCad conventions, with no
  datasheet-to-footprint automation.

- **Vayo `ComponentLibExportImport.exe`** (vendor baseline)
  *Key limitation:* moves records between CLM databases and refuses to update
  existing part/manufacturer pairs, so it can neither create geometry from a
  datasheet nor correct geometry in place.

## Project Objectives

- **O1: To design** a datasheet-grounded extraction and generation harness that
  converts a datasheet's mechanical drawing table into the target library's
  plain-text geometry grammar, with explicit unit normalisation and explicit
  disambiguation of similar dimension symbols (e.g. `E` overall span vs `E1`
  body width).
- **O2: To develop** the orchestration layer — worklist intake, deterministic
  datasheet-to-row binding, agent execution, validation gates that run outside
  the model, a human review step, and an audit trail of what was generated,
  checked and approved.
- **O3: To evaluate** the harness on geometric agreement with a human-verified
  reference package, round-trip validity through the vendor application, and
  reviewer effort / automation-bias risk.

## Target Users

PCB library engineers and component librarians who currently key datasheet
dimensions into CLM by hand and are accountable for library correctness;
hardware design engineers who bear the cost of a bad footprint at assembly; and
the reviewer/approver role who needs an audit trail proving a generated package
was checked against its source document.

## Data Sources

- **STMicroelectronics M24M02-A125 datasheet**, DocID027947 Rev 3 (37 pp.),
  §9.1 SO8N outline and mechanical table — reference case. Public vendor
  document; a local copy is held for course use only and is **not** committed
  here.
- **KiCad official footprint library** —
  https://gitlab.com/kicad/libraries/kicad-footprints (open reference geometry)
- **Vayo CLM documentation** — https://www.vayoinfo.com/
- **Local Vayo CLM v6.0.1 catalogue export** — the worklist of components
  needing geometry. Internal; not committed.

## Repository Structure

```
.
├── README.md
├── docs/                     # week-by-week written deliverables
│   ├── week1_observation_sheet.md
│   ├── week1_system_map.png
│   ├── week1_project_scope.md
│   └── source/               # editable diagram source + notes
├── data/                     # dataset notes and manifests (no raw vendor PDFs)
├── src/                      # harness and orchestration code
├── notebooks/                # exploratory and evaluation notebooks
├── evaluation/               # test cases, metrics, results
└── prompts/                  # versioned prompt and instruction artefacts
```

## Weekly Progress

| Week | Focus | Deliverable | Status |
|------|-------|-------------|--------|
| W1 | System map + project scope | README + docs | ✅ |
