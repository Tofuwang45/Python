# CLAUDE.md

TheAlgorithms/Python is an educational collection of about 1,600 standalone algorithm
implementations, mostly one algorithm per file. There is no app, package or CLI to build
or run. Every file's "product" is a readable implementation plus the doctests that prove
it works. PR rules (checkbox the bot enforces, one algorithm per PR) are in `AGENTS.md`.

## Layout

- Top-level dirs are topics (`sorts/`, `graphs/`, `maths/`, `data_structures/`, ...).
  Each one is a package with an `__init__.py` (ruff `INP` fails without it). Put new files
  in an existing topic dir. Never create a new top-level dir or a file at the root.
- Tests live **inside the algorithm file as doctests**. A few dirs also have a `tests/`
  subfolder or `test_*.py` beside the code (`graphs/tests/`, `matrix/tests/`,
  `data_structures/kd_tree/tests/`, root `tests/test_sorts.py`). Only add a pytest file
  where the dir already has them.
- `project_euler/problem_NNN/solN.py`: 3-digit zero-padded dir, and each file has a
  `solution()` whose default args are the problem's input. Answers are SHA-256 hashes in
  `scripts/project_euler_answers.json`, checked by `scripts/validate_solutions.py`.
- `scripts/` is repo tooling (directory builder, filename validator, PR-closing bots),
  not algorithms.

## Commands

Run everything from the repo root through `uv`. `.python-version` pins free-threaded
**3.15t**, and macOS `python3` is too old to import half the repo.

```bash
uv sync --group=test                           # once; add --group=cv for opencv files
uv run pytest strings/palindrome.py            # one file: doctests run via pyproject addopts
MPLBACKEND=Agg uv run pytest graphs/           # one topic
uvx ruff check . && uvx ruff format --check .  # same as CI ruff job
./scripts/validate_filenames.py                # lowercase, no spaces/dashes, inside a dir
uvx pre-commit run --all-files                 # everything CI and pre-commit.ci run
uv run pytest project_euler/problem_001/       # Euler is excluded from the main suite
```

Full suite, the way CI runs it (`build.yml`). Bare `uv run pytest` also collects the
image-processing and network files that CI skips:

```bash
MPLBACKEND=Agg uv run pytest --ignore=computer_vision/flip_augmentation.py \
  --ignore=computer_vision/harris_corner.py --ignore=computer_vision/mosaic_augmentation.py \
  --ignore=data_compression/peak_signal_to_noise_ratio.py --ignore=digital_image_processing/ \
  --ignore=docs/conf.py --ignore=project_euler/ --ignore=scripts/validate_solutions.py \
  --ignore=web_programming/current_stock_price.py \
  --ignore=web_programming/fetch_anime_and_play.py .
```

Keep the `MPLBACKEND=Agg` prefix on any multi-file run. Without it, doctests that call
`plt.show()` (e.g. `audio_filters/show_response.py`) open macOS windows and block until
they close, so the suite takes 70–90s instead of about 20s.

On a clean checkout, two tests fail on macOS and are not yours to fix:
`machine_learning/linear_discriminant_analysis.py::...gaussian_distribution` (seeded
`random` output) and `web_programming/instagram_crawler.py` (live network). `ty`
(`uv run --with ty ty check`) is advisory: CI runs it with `--exit-zero`.

## Conventions

- Every function gets a docstring with doctests, including the error path:
  `>>> f(-1)` / `Traceback (most recent call last):` / `ValueError: ...`. Raise
  `ValueError`/`TypeError` on bad input; don't print or return sentinel strings.
- End files with `if __name__ == "__main__":` / `import doctest` / `doctest.testmod()`.
  Keep `input()`, plotting and network calls under that guard, never at import time.
- New code needs full type hints even though `pyproject.toml` exempts older dirs
  (`sorts/*`, `graphs/*`, ...) from `ANN` rules. Those ignores are legacy, not permission.
- Cross-file imports are absolute from the repo root
  (`from data_structures.kd_tree.kd_node import KDNode`).
- Add a Wikipedia (or similar) URL in the module docstring for new algorithms.
- Names are descriptive snake_case (`is_palindrome`, not `check`). Filenames are lowercase
  with underscores.

## Guardrails

- `DIRECTORY.md` is regenerated and force-pushed by CI (`scripts/build_directory_md.py`).
  Don't hand-edit it or include it in a diff.
- Don't edit `uv.lock` by hand, and don't add a `requirements.txt` or `pip install`. Add
  dependencies in `pyproject.toml`, then run `uv lock`.
- Don't run `python path/to/file.py` to test a file. Files with package imports fail with
  `ModuleNotFoundError: No module named 'data_structures'`. Use `uv run pytest <file>`.
- Doctests must not hit the network or depend on randomness/time. CI runs each test 8x
  in parallel threads (`pytest-run-parallel`), so avoid mutable module-level state.
- Project Euler: no doctest that asserts the final answer, and don't touch
  `project_euler_answers.json`.
- Don't "fix" the `# FIX ME` / `# DO NOT FIX` rule ignores in `pyproject.toml`, or reformat
  untouched files. Keep diffs to the file you were asked about.
