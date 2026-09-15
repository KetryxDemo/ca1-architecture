# Architecture model - Cardiac Ablation Platform

This repository holds the system architecture model. It is deliberately separate from the code
repository: the architecture drives the code, not the other way round. Nothing here is generated
from source. Source paths appear in the model as the realization of a modelled element, and the
code repository at `../code-repo/` is the thing that must agree with this model.

Everything in here is hypothetical, built for demonstration. No real product, decomposition or
device data.

## Format

One format, held rigidly: **YAML**, one entity kind per schema, `schema:` and `file:` as the first
two keys of every file. Markdown is used only for this README and for `STRUCTURE.md`, which are
navigation, not model content.

YAML rather than Markdown tables because the model is a graph with repeated, nested, multi-valued
relationships - a function is realized from many modules, a module offers many API groups, a
matrix cell carries a full usage record for each of its two sides. Tables flatten that and force
the reader to reconstruct the nesting; YAML keeps each fact addressable by key path, and it
parses, so the model can be checked mechanically rather than by eye.

Conventions that hold everywhere:

- Ids are uppercase, hyphen separated, no underscores. The prefix declares the entity kind; the
  full prefix list is in `model/platform.yaml` under `idPrefixes`.
- Every id has exactly **one definition site** - the single line where it appears as an `id:` key.
  Every other occurrence is a reference. `STRUCTURE.md` maps every id to its definition site, and
  is rebuilt by scanning the model for `id:` keys; if a line number and the file disagree, the
  file is right.
- Source file paths are quoted exactly as they appear in `../code-repo/`. Paths contain
  underscores; paths are not ids.
- Lists that could be empty are written as empty lists, never omitted, so that absence is explicit.
- Where a fact is not proven by source in `../code-repo/`, the entry carries an `evidence` or
  `...Evidence` field saying so.

## Layout

```
README.md                                  this file
STRUCTURE.md                               id index - every id to file and line
model/
  platform.yaml                            model root: layers, user needs, logical components,
                                           hardware blocks, process index, id conventions
  processes/
    PRC-HEALTHMON.yaml                     health-monitor      (decomposed here)
    PRC-GENERATORCTL.yaml                  generator-control   (block level only)
    PRC-MAPPINGENGINE.yaml                 mapping-engine      (block level only)
    PRC-CATHETERIO.yaml                    catheter-io         (block level only)
    PRC-CASERECORDER.yaml                  case-recorder       (block level only)
  functions/
    FN-HM-RAM.yaml                         memory integrity checking
    FN-HM-DISK.yaml                        storage health checking
    FN-HM-HB.yaml                          inter-process heartbeat / liveness monitoring
  modules/
    module-index.yaml                      realization layer: source modules, the API groups each
                                           offers with their activation conditions, construction
                                           sites and member read sites
  interfaces/
    topics.yaml                            message bus topics and the process-to-process
                                           interfaces built from them
  analysis/
    n2-matrix.yaml                         N-squared interaction matrices - one over the health
                                           monitor functions, one over the five processes
  allocation/
    function-to-control.yaml               function and source path to design control ids, plus
                                           the resolution procedure
    control-to-function.yaml               design control id to function and realization
```

The four peer processes are modelled at block level. `../code-repo/` contains the health-monitor
service alone, so their functions, modules and design control allocation are outside this slice.
Their names are the strings the platform uses on the bus: `generator-control`, `mapping-engine`,
`catheter-io`, `case-recorder`.

## Layers

| Layer | Id | Holds | Where |
| --- | --- | --- | --- |
| User needs | `LYR-NEED` | `UN-*` | `model/platform.yaml` |
| System functions | `LYR-FUNC` | `FN-*` | `model/functions/` |
| Logical components | `LYR-LOGICAL` | `LC-*` | `model/platform.yaml` |
| Software and hardware blocks | `LYR-BLOCK` | `PRC-*`, `HWB-*` | `model/processes/`, `model/platform.yaml` |
| Realization | `LYR-REAL` | `MOD-*`, `BHV-*` | `model/modules/module-index.yaml` |
| Requirements | `LYR-REQ` | `RQ-*`, `RC-*` | maintained outside this repo, indexed in `model/allocation/` |
| Safety risk | `LYR-SAFETY` | `RSK-*` | maintained outside this repo, indexed in `model/allocation/` |
| Cybersecurity | `LYR-CYBER` | none in this slice | - |
| Human factors | `LYR-HF` | none in this slice | - |
| Verification and validation | `LYR-VV` | `TC-*` | maintained outside this repo, indexed in `model/allocation/` |

## What this model does not hold

Design control **content** is not here. Requirement text, risk statements and test steps live in
the requirements management tool, which is the system of record for them. This repository holds the **allocation** - which
control id belongs to which function, and what that function is built from. An `RQ-`, `RSK-`,
`RC-` or `TC-` id appearing here is a pointer, never a copy.

`RC-HM-01` to `RC-HM-04` are `REQUIREMENT` items, not a separate type. Their risk control role is
carried by the built-in `IS_RISK_CONTROLLED_BY` relation, with the risk as source and the
requirement as target. `RC-` is a naming convention in this model, nothing more.

Design control allocation is populated for `PRC-HEALTHMON` only. That is a boundary of this model
slice; it is not a statement about coverage.

## How to read it

**From a function.** Open `model/functions/<id>.yaml`. It carries the owning process, the inputs
and outputs, its cadence and configuration keys, a `timerUsage` block recording every timer
instance it constructs and every member it reads, any clocks and thresholds it holds itself, the
source modules that realize it, and the design control ids allocated to it.

**From a source path.** `model/allocation/function-to-control.yaml` has a `moduleToControlIndex`.
Look the path up there. If `sharedAcrossFunctions` is false, the listed controls are the answer.
If it is true, the listed controls are the widest possible answer and `narrowWith` says where to
go next.

**From a design control id.** `model/allocation/control-to-function.yaml` has one entry per id,
giving the item type, the role, the function, the check id, the exclusive module and the shared
ones.

**From a change, to the set of elements it reaches.** The model records mechanisms and observed
facts. It does not record the consequence of any particular change, and no file contains a
precomputed answer. The steps are written out in full under `resolutionProcedure` in
`model/allocation/function-to-control.yaml`. In short: find the API group the changed members
belong to and read its activation condition, then compare that condition against each function's
recorded construction arguments and the members it actually reads, then check whether a threshold
that looks like it belongs to the shared module is in fact measured inside the function, and only
then carry the resulting function set into the allocation files.

A shared module is not a uniform dependency. Two functions can import the same module, construct
the same class and call the same method, and still depend on different parts of its contract,
because the part that is live on an instance can be decided by the arguments passed at
construction. `model/analysis/n2-matrix.yaml` records what each side of a pair does with a shared
module under `rowUsage` and `columnUsage`, and `model/modules/module-index.yaml` records the
condition under which each part of that module's contract is live. Neither file records the
intersection.
