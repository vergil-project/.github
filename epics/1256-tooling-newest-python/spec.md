# Narrow vergil-tooling to Python 3.15 — design

- **Epic:** vergil-project/.github#1256
- **Planning issue:** vergil-project/.github#958 (review history is in its
  comments)
- **Status:** approved

## Intent

vergil-tooling tests every Python series in the window, 3.12 to 3.15
(`requires-python = ">=3.12"`), and its CI builds all four in parallel. It is
a set of command-line tools that no other repo imports. Since epic
vergil-project/.github#649 every Vergil image and VM runs it on the core
Python, 3.15, and the language fixtures cover the window for the projects
Vergil builds. vergil-tooling no longer needs to test the range itself.

This epic sets vergil-tooling's floor to 3.15 and makes sure that every place
vergil-tooling is installed or run uses 3.15, within today's install
mechanisms (packaged installs, and uv everywhere else).

## Scope

- **The floor.** `requires-python = ">=3.15"`, ruff `target-version =
  "py315"`, mypy `python_version = "3.15"`, `uv.lock` re-locked, and the ruff
  `UP` rewrites the new target asks for. vergil-tooling's
  `[project].repository-type` is `tooling`, so the CI matrix and `vrg-validate
  --all-versions` resolve it as an app: one series, the newest window version
  with a published runtime, 3.15 today. When a newer series gets a published
  runtime, CI moves to it on the next build instead of testing it next to
  3.15.

  > **Amended after implementation** (2026-11-17;
  > vergil-project/vergil-tooling#4608, PR
  > vergil-project/vergil-tooling#4647). The original text said the CI
  > matrix and `vrg-validate --all-versions` follow `requires-python`, and
  > that a new series joining the window is tested next to 3.15 until a later
  > PR raises the floor. Neither holds: the version resolver ignores
  > `requires-python` and narrows only by `repository-type`. By the human's
  > ruling on vergil-tooling#4608's merge review, the single series comes
  > from changing vergil-tooling's `repository-type` from `library` to
  > `tooling`, so CI follows the newest window series with a published
  > runtime.
- **Every install names 3.15.** Each place that installs or syncs
  vergil-tooling without naming an interpreter gets `--python 3.15`:

  | Place | Repo | Today |
  | --- | --- | --- |
  | `install-tooling.sh` self-repo `uv sync` on a host runner (vergil-tooling's own CI host jobs) | vergil-actions | `only-system`, so the runner's 3.12 |
  | `install-tooling.sh` dev-pin and `install-source: git` installs on a host runner | vergil-actions | no `--python` |
  | `install-tooling.sh` and `container_cache.py` installs in an image without the vergil apt source or a core Python | vergil-actions, vergil-tooling | no `--python` |
  | `vm_packages.dev_install` (`vrg-vm update <dev ref>`) | vergil-tooling | no `--python` |
  | `scripts/vrg-vm-init.sh`, `tests/test_vergil.sh` | vergil-vm | no `--python` |
  | the dev-tree `uv sync --group dev` and the developer-host install in `consuming-repo-setup.md`, `releasing.md`, `CLAUDE.md` and `README.md` | vergil-tooling | no `--python` |

  On a host runner the self-repo sync drops `UV_PYTHON_PREFERENCE=only-system`
  and lets uv provide 3.15. That setting stays for container jobs, where it
  pins uv to the image's interpreter (issue #1002).
- **Already 3.15, unchanged:** the packaged installs (images, VMs, host
  runners on a release pin), `package/setup`'s driver install, the
  `vergil.toml` consumer-refresh, the documented host installs, and the docs
  builds on `prod-base`.

## Order

The vergil-actions change must be released, moving `v2.1`, before the floor
rises. Without it, vergil-tooling's own CI host jobs sync on 3.12 and fail.
The vergil-tooling and vergil-vm install changes land before the floor too.

## Out of scope

- Reclassifying vergil-tooling as an application (vergil-project/.github#1260).

  > **Amended after implementation** (2026-11-17;
  > vergil-project/vergil-tooling#4608). The `repository-type` change from
  > `library` to `tooling` (see the floor, above) already makes vergil-tooling
  > resolve as an app for version resolution. The rest of #1260, its release
  > and branching model and the other follow-on effects, stays out of scope.
- Detecting a new series and prompting a floor raise (vergil-project/.github#1261).
- `vrg-host` handling of vergil-tooling (vergil-project/vergil-tooling#4603;
  corrected in the retrospective from a dead `vergil-project/.github#1266`).
- Unifying the definitions of the core series (vergil-project/.github#1267).
- Changing how any install path gets its interpreter beyond naming 3.15.
