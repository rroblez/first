# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project state

This is a brand-new, minimal Python project. Currently `main.py` is the only source file and just prints `hi`; it imports `requests` and `pynetbox` but does not yet use them. There is no test suite, build tooling, linter config, or CI set up yet.

## Environment

- Python 3.9.6, managed via a `uv`-created virtualenv at `.venv/`.
- No `requirements.txt` or `pyproject.toml` exists yet — dependencies (`requests`, `pynetbox`) are currently only declared via the `import` statements in `main.py`.

## Commands

- Run the script: `.venv/bin/python main.py` (or activate the venv with `source .venv/bin/activate` first, then `python main.py`).
- Install a dependency into the venv: `uv pip install <package>` (uv is the package manager in use, per `.venv/pyvenv.cfg`).

## Notes for future work

- Since this project is essentially a skeleton, expect to be defining initial structure (dependency manifest, module layout, tests) rather than navigating existing architecture.
- `pynetbox` suggests this project is intended to interact with a NetBox instance (IPAM/DCIM); confirm intent with the user before assuming specific NetBox workflows, since none exist in the code yet.
