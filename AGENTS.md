# AGENTS.md

## Setup And Dependencies
- Use Python 3.11 for local work unless intentionally testing the 3.13 CI lane; `.python-version` is `3.11` and CI covers `3.11` plus `3.13`.
- `pyproject.toml` is the primary dependency manifest and `uv.lock` pins the resolved environment; run `uv sync --frozen` before `uv run --no-sync ...` commands.
- `requirements.txt` exists only for legacy pip installs and Docker cache compatibility; do not add new deps there without also updating `pyproject.toml` and the lockfile.
- Optional TwelveLabs support is installed with `uv sync --extra twelvelabs`; do not make it a base dependency unless every install path needs it.

## Verification Commands
- CI install: `uv sync --frozen --python 3.11` or `uv sync --frozen --python 3.13`.
- CI compile check: `uv run --no-sync python -m compileall app cli.py main.py webui test`.
- CI lint check, 3.11 only: `uv run --no-sync ruff check app cli.py main.py webui test`.
- Full test suite with CI UTF-8 mode: `uv run --no-sync python -X utf8 -m pytest -q test`.
- Coverage gate: `uv run --no-sync python -X utf8 -m coverage run -m pytest -q test` then `uv run --no-sync python -m coverage report`.
- Focused tests: `uv run --no-sync python -X utf8 -m pytest -q test/services/test_video.py` or append `::TestClass::test_method`.
- Live provider tests are skipped unless `MPT_RUN_INTEGRATION_TESTS=1` and the required credentials are provided.
- Redis-backed tests in CI use Redis 7 with `MPT_TEST_REDIS_HOST=127.0.0.1`, `MPT_TEST_REDIS_PORT=6379`, and `MPT_TEST_REDIS_DB=15`.

## Entrypoints
- API service starts from `main.py`, which runs `uvicorn` with `app.asgi:app`; API routes are wired in `app/router.py` and `app/controllers/v1/*`.
- WebUI starts from `webui/Main.py`; use `webui.bat` on Windows or `sh webui.sh` on macOS/Linux because those scripts set `PYTHONPATH`, choose ports, and prefer the project `.venv`/`uv`.
- CLI starts from `cli.py`; `uv run python cli.py --help` is the source of truth for pipeline stages, batch JSON/JSONL behavior, and argument validation.
- The shared video pipeline is `app/services/task.py`; API, CLI, and WebUI should stay aligned by reusing service-layer behavior instead of duplicating checks in one entrypoint.

## Config And Runtime Files
- `config.toml` is created from `config.example.toml` on first import of `app.config.config`; it can contain API keys and is ignored by git.
- Runtime outputs under `storage/`, `logs/`, `.coverage`, `coverage.xml`, `htmlcov/`, model downloads under `models/`, and Streamlit local config are ignored; avoid committing generated artifacts.
- `config.example.toml` documents public defaults only. Keep real credentials in local `config.toml` or environment variables and never print or commit them.
- `app.config.config.save_config()` uses atomic replacement with a Docker bind-mount fallback; preserve this behavior when touching config persistence.
- FFmpeg can be supplied with `[app].ffmpeg_path`; when valid, it sets `IMAGEIO_FFMPEG_EXE` during config import.

## Behavior Gotchas
- API key auth is optional: when `[app].api_key` is empty, `/api/v1` and `/tasks` are unauthenticated; when set, clients must send exactly one `x-api-key` header, and generated task files under `/tasks` are protected too.
- Browser CORS is same-origin by default. Only set `CORS_ALLOWED_ORIGINS` for trusted separate browser frontends; it does not affect curl/Postman/server-side clients.
- Redis task state assumes Redis is private to this app; do not expose it to untrusted writers without replacing the permissive compatibility parser in `app/services/state.py`.
- CLI charge-confirmation flags are required for paid AI video sources at material/video stages: `--confirm-seedance-charge`, `--confirm-ofox-charge`, and `--confirm-metaso-minimax-charge`.
- CLI batch manifests are UTF-8 JSON arrays or JSONL, max 100 tasks and 1 MiB; manifest-relative paths apply only to `custom_audio_file` and local `video_materials[].url` inside the manifest.
- `webui/Main.py` intentionally inserts the repo root at the front of `sys.path`; Ruff ignores `E402` only for this file.
- `docs/skill/SKILL.md` is a repo-local agent skill for creating finished videos; it uses adjacent `docs/skill/mpt_agent.py` and `uv run --no-project --python 3.11`.
