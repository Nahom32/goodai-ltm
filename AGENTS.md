# Repository Guidelines

## Project Structure & Module Organization
`goodai/` contains the Python package. Core long-term-memory code lives in `goodai/ltm/`, with subpackages for `mem/`, `embeddings/`, `reranking/`, `persistence/`, `eval/`, and `training/`. Shared utilities live in `goodai/helpers/` and general-purpose modules in `goodai/modules/` and `goodai/text_gen/`. Tests are colocated with the code in `*/tests/` directories such as `goodai/ltm/mem/tests/`. Example integrations live in `examples/`, training and evaluation entry points in `scripts/`, and benchmark notes in `evaluations/`.

## Build, Test, and Development Commands
Install dependencies with `pip install -r requirements.txt`, then install the package in editable mode with `pip install -e .`. Run the full test suite with `pytest`. Target a narrower area while iterating, for example `pytest goodai/ltm/mem/tests -q` or `pytest goodai/helpers/tests/test_json_helper.py`. Package metadata and distribution wiring are defined in `setup.py`; use `python setup.py sdist bdist_wheel` only when preparing a release build.

## Coding Style & Naming Conventions
Follow existing Python style: 4-space indentation, `snake_case` for functions and modules, `PascalCase` for classes, and explicit imports grouped by standard library, third-party, and local modules. Keep type hints where the surrounding code uses them and prefer small, focused helpers over deeply nested logic. There is no dedicated formatter config in this repository, so match the local file style and keep comments sparse and technical.

## Testing Guidelines
Tests run under `pytest`, but many suites use `unittest.TestCase`; both patterns are already accepted here. Name test files `test_*.py` and test methods `test_*`. Add regression coverage alongside the affected module, for example new memory behavior should usually be covered under `goodai/ltm/mem/tests/`. Prefer deterministic assertions and avoid network-dependent tests unless explicitly guarded.

## Commit & Pull Request Guidelines
Recent history favors short, imperative commit subjects such as `Added support for FlagEmbedding models.` or `Mutable field not working with Python 3.11+`. Keep the first line concise, explain behavioral changes in the body when needed, and reference issue or PR numbers when applicable. Pull requests should describe the user-visible impact, list test coverage added or updated, and include example output or screenshots when changing examples, evaluations, or generated artifacts.

## Configuration & Secrets
Some features depend on external providers through packages such as `openai`, `litellm`, and `cohere`. Load credentials from environment variables or a local `.env` file via `python-dotenv`; never commit secrets, tokens, or machine-specific paths.
