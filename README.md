# dd-lib-stub

Single source of truth for all `dd*` Python libraries ([ddutils](https://github.com/davyddd/ddutils),
[dddesign](https://github.com/davyddd/dddesign), [ddsql](https://github.com/davyddd/ddsql), ...).

It solves three problems in one place:

1. **Reusable GitHub Actions workflows** (`.github/workflows/`) — CI logic (linters, test matrix, coverage,
   PyPI publishing) lives here; each library only keeps thin caller stubs.
2. **Copier template** (`copier.yml` + `template/`) — the starting point for a new library and the mechanism
   for propagating boilerplate updates (configs, Dockerfile, fabfile, workflow stubs) to existing ones.
3. **Renovate preset** (`default.json`) — shared dependency-update rules; libraries reference it from their
   `renovate.json`.

Tooling standard: [uv](https://docs.astral.sh/uv/) (packaging, lock, build), ruff (lint + format),
[ty](https://docs.astral.sh/ty/) (type checking), [complexipy](https://github.com/rohaquinlop/complexipy)
(cognitive complexity), Docker + Fabric for the local dev loop.

## Creating a new library

```bash
uvx copier copy gh:davyddd/dd-lib-stub <path-to-new-lib>
cd <path-to-new-lib>
git init && uv lock
fab build && fab tests && fab linters
```

Answers are stored in `.copier-answers.yml`. Per-library specifics (supported Python range, extra CI
test matrix) are template variables; the local Docker image always runs on the newest supported Python — they are never overwritten by updates.

### CI test matrix

The tests job of the reusable workflow takes a single `test-matrix` input: a JSON object used verbatim as the
GitHub `strategy.matrix`. The `python-version` axis is generated from `min_python_version`/`max_python_version`;
every other axis is a list of pip install specs that get installed on top of the locked environment, so a
library can be tested against any number of dependency versions at once. `include`/`exclude` work as in
GitHub Actions. The extra axes come from the `test_matrix` copier answer, e.g. for dddesign:

```json
{
  "pydantic": ["pydantic[email]==2.1", "pydantic[email]==2.11", "pydantic[email]==2.12", "pydantic[email]==2.13"],
  "exclude": [{"python-version": "3.13", "pydantic": "pydantic[email]==2.1"}]
}
```

After pushing to GitHub:
- add a PyPI [trusted publisher](https://docs.pypi.org/trusted-publishers/): repository `davyddd/<name>`,
  workflow `publish_python_package.yml`, environment `pypi`;
- create the `pypi` environment in the repo settings;
- add the `CODECOV_TOKEN` secret.

## Propagating updates to existing libraries

Changes are delivered through two independent channels:

- **CI logic** (steps, action versions, uv version in CI): edit the reusable workflows here and release a new
  tag. Caller stubs pin the exact template version they were generated from (`@vX.Y.Z`), so the libraries pick
  up the change either via `copier update` or through the Renovate PR that bumps the `uses:` reference.
- **Files living inside each repo** (lint configs, Dockerfile, fabfile, pyproject skeleton, workflow stubs):
  edit `template/`, commit, tag, then in each library run:

  ```bash
  uvx copier update
  ```

  Copier does a three-way merge against the answers in `.copier-answers.yml`, preserving local changes
  (dependencies in `pyproject.toml`, the library's Python range, etc.). `README.md` and package skeletons
  are generated once and never touched by updates (`_skip_if_exists`).

- **Dependency versions** (actions, uv in Dockerfiles, pinned linters/pytest): Renovate opens PRs in each
  library automatically using the shared preset from this repo.

## Releasing a library

Bump `version` in `pyproject.toml` (e.g. `uv version --bump minor`), merge to `main`, then push a `v*.*.*`
tag matching the version. The publish pipeline runs the full QA matrix, checks that the tag is on `main`,
matches the project version and is not already on PyPI, builds the distributions (`build_python_package.yml`),
uploads them to PyPI and creates a GitHub release with the artifacts (`github_release.yml`).

The PyPI upload step itself lives in the library's workflow stub, not in a reusable workflow: PyPI trusted
publishing verifies the workflow file of the publishing job and rejects reusable workflows.

## Versioning of this repo

Tag every release as `vX.Y.Z`. The same tag serves both consumers: copier records it in
`.copier-answers.yml` (`_commit`) and the workflow stubs reference it in `uses: davyddd/dd-lib-stub/...@vX.Y.Z`,
so a library is always on one well-defined template version. Tags are never moved.
