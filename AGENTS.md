# AGENTS.md — adb-mcp

A structured-output MCP server wrapping the Android Debug Bridge. 90 tools in 9 groups (device, apps, input, ui, logs, media, files, network, system), every one returning JSON instead of terminal text, so an agent can drive a phone without copy/pasting `adb shell` output. Built for mobile reverse-engineering and QA automation. PyPI package is `adbmcp`; the import package and `python -m` target are `adb_mcp`; the console script is `adbmcp`.

**This checkout is UPSTREAM-OWNED** — `origin` is `https://github.com/FlashZ/adb-mcp`. Local commits are fine; do not push, do not open PRs.

## Hard rules

- **Never weaken the shell gate.** `run_shell` exists only when `ADB_MCP_ALLOW_SHELL=1`; with it unset it must return an error and run nothing. Default is off.
- **All device-side quoting goes through `core.quote_argv` / `core.shell` / `core.shell_rc`.** `adb shell` concatenates its args through the device's `sh`, so any tool that passes a user string (package name, `dumpsys` arg, settings value) must use a list-form call. Hand-built strings bypass the injection guard.
- **Keep `delete_file`'s protected-root refusal** (`/`, `/system`, `/data`, `/sdcard`, `/vendor`, `/product`) and `dumpsys`'s service-name-only validation. These are the documented scope guarantees in `SECURITY.md`.
- **`README.md` is the public contract** — the 90-tool list, env vars and callback names are asserted there. Changing tool names or counts means updating README, and `tests/test_smoke.py` asserts registration.
- **Do not push to `origin`.** If a change belongs upstream, report it; Joash decides.
- Line length 110, `target-version = "py310"`, ruff selects `E,F,I,UP,B` (see `pyproject.toml`).

## Install and run

```bash
pip install adbmcp                       # from PyPI
pip install -e .                         # from a clone
pip install -e ".[dev]"                  # + pytest, ruff
```

```bash
adbmcp                                   # starts on stdio, waits; Ctrl-C to exit. No output = healthy.
python -m adb_mcp.server                 # equivalent
```

Requires Python ≥ 3.10 and platform-tools `adb` on the machine. `adb` is resolved by `core.adb_bin()` from `ADB_PATH`, then `ANDROID_HOME` / `ANDROID_SDK_ROOT`, the default SDK paths, scoop, then `PATH` (result is `lru_cache`d).

```bash
pytest -q        # device-free: imports, tool registration, parsers, quoting regressions
ruff check .
```

CI (`.github/workflows/ci.yml`) runs exactly those two on Python 3.10/3.11/3.12 for every push to `main` and every PR. `.github/workflows/release.yml` publishes to PyPI via Trusted Publishing on a GitHub Release or a `v*` tag.

## Configuration

| env var | meaning |
|---|---|
| `ADB_PATH` | Path to `adb` (auto-detected if unset) |
| `ANDROID_HOME` / `ANDROID_SDK_ROOT` | SDK root for adb auto-detection |
| `ADB_SERIAL` | Default device serial; overridden at runtime by `set_default_device` |
| `ADB_MCP_ALLOW_SHELL` | `1`/`true` enables the opt-in `run_shell` escape hatch |

Every device tool also takes a per-call `serial`.

## Where things are

| Area | File | Read it when |
|---|---|---|
| Security model, what is and is not guarded | `SECURITY.md` | Touching any tool that shells out or mutates the device |
| Public contract: install, client registration, all 90 tools, config | `README.md` | Before renaming a tool or changing behaviour |
| adb discovery, `run`, `quote_argv`, `shell`/`shell_rc`, `ok`/`err`, `@tool`, `shell_allowed` | `adb_mcp/core.py` | Any plumbing change |
| stdio entry point (`import adb_mcp.tools` then `mcp.run()`) | `adb_mcp/server.py` | Startup problems |
| Tool registration order | `adb_mcp/tools/__init__.py` | A tool not appearing |
| The 90 tools, one module per group | `adb_mcp/tools/{device,apps,input,ui,logs,media,files,network,system}.py` | Changing or adding a tool |
| Device-free tests incl. injection regressions | `tests/test_smoke.py`, `tests/test_regressions.py` | You changed quoting or a parser |
| Ready-made client config | `examples/mcp.json` | Wiring a new MCP client |

## Gotchas

- The `adb_mcp/server.py` docstring still says the entry point is `adb-mcp`; the real console script declared in `pyproject.toml` is `adbmcp`.
- Windows clients that don't inherit your PATH need the absolute-interpreter + `python -m adb_mcp.server` form, plus an explicit `ADB_PATH`.
- Tests must stay device-free — CI has no attached phone. Anything that needs a real device belongs in a note, not a test.

## Documentation hygiene

All project knowledge lives in this folder. Do not create or update notes for this project in Obsidian.
