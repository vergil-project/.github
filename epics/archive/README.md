# Archived epics

The folders here are **abandoned design attempts**. They are kept for the
record only. They will not be implemented.

**Agents: do not read anything under this directory.** Don't use it as input,
context, precedent or a source of decisions for any current work, and don't
follow links into it. A fresh attempt at the same problem must start from its
own planning issue and the current code.

## Contents

- `443-multi-runtime-lifecycle/`: the first attempt at multi-series Python
  runtimes. It was abandoned on 2026-10-25 for two reasons:
  - it bundled a second feature, the unattended lifecycle;
  - its repeated pushback rounds built on each earlier round's decisions
    without re-examining them, which compounded into a design too complex to
    implement.
