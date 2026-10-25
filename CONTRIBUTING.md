# Contributing to spinf-prompts

This repository holds files only: the official packs, their fixtures, samples and the format specification. The code is
spinf-serve's `packs` module.

- **A published version never changes.** Thresholds and calibration are refit as a new patch version; changed or added
  questions, reads or categories are a new minor; the manifest lists the newest first.
- **Validate** with `spinf-serve packs validate .` (`pip install spinf-serve`) and list what a server would enable with
  `spinf-serve packs list --packs . --no-official --allow-namespace spinf --model <model>`. CI runs both.
- **Fixtures**: every version ships `packs/<name>/fixtures/<version>.json`, the samples scored on the server with the
  expected output in every mode; spinf-serve's tests recompute it. A new version needs a new fixture.
- **Your own packs** belong in your own repository and namespace (`docs/pack-format.md`, README "Publish your own packs");
  `spinf/` is reserved for this one. To have your packs listed in spinf-serve's PACKS.md, open a "List my pack" issue
  there: https://github.com/spinfinc/spinf-serve/issues/new?template=pack.yml.
- Data a pack was calibrated or checked on is credited in `ACKNOWLEDGEMENTS.md` with its terms, and is not bundled
  unless its licence allows it. Apache 2.0 applies to every contribution.
