# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

PLPR v2: Iranian license plate recognition (GPL-3.0). One two-stage model pipeline runs in four places: a browser app (`web/`), a PySide6 desktop app (`home-yolo.py`), a FastAPI server (`server/app.py`) and an edge agent (`server/agent.py`). `PLPR-modernization-plan.md` has the audit and rationale behind v2; `docs/` covers benchmark, training and research.

## Commands

Python (3.10+, uv or pip). CI uses 3.12:

```bash
uv venv && uv pip install -e ".[server,desktop,dev]"   # add ,train for export/training
uv run python scripts/export_onnx.py                   # REQUIRED once: model/*.pt -> model/onnx/ (needs "train" extra, only step that needs PyTorch)
uv run ruff check plpr scripts server tests trainer    # lint (line-length 140; yolov5/ excluded)
QT_QPA_PLATFORM=offscreen uv run pytest -q             # all tests
QT_QPA_PLATFORM=offscreen uv run pytest tests/test_core.py::test_tracker_votes_and_emits_once -q   # single test
```

Engine tests are skipped when `model/onnx/plate.onnx` is missing, so export first.

Web (`cd web`, Node 22):

```bash
npm install && npm run models   # copies ONNX models from ../model/onnx into the app
npm run dev                     # vite dev server
npm test                        # vitest (src/**/*.test.ts); single file: npx vitest run src/lib/core.test.ts
npm run build                   # tsc --noEmit + vite build
npm run e2e                     # Playwright (read.spec.ts, ui.spec.ts); needs `npx playwright install chromium`
```

Run server: `PLPR_ADMIN_TOKEN=... uv run uvicorn server.app:app`. Agent: `uv run python -m server.agent --source ... --gate-id 1 --server URL --token T`. Desktop: copy `config.example.ini` to `config.ini` (git-ignored; holds camera credentials), then `uv run python home-yolo.py`. Server env vars, auth/roles and webhooks are documented in `server/README.md`.

## Architecture

Pipeline: frame → plate detector (YOLOv5s ONNX) → crop → character detector (43 classes) → sort chars left to right → grammar check (2 digits, letter, 3 digits, 2-digit region) → `PlateTracker` votes per character across frames so one car yields one entry → resident lookup (allowed / not allowed / unknown).

- `plpr/` is the core library: `engine.py` (ONNX Runtime inference, no torch), `plate.py` (Persian plate grammar/parsing), `labels.py`, `tracking.py`, `plate_type.py` (colour classifier for special plates), `synth.py` (synthetic plate generator). Models load from `model/onnx/` (override with `PLPR_MODEL_DIR` on the server).
- **`web/src/lib/` is a hand-maintained TypeScript port of `plpr/`** (`pipeline.ts` mirrors `engine.py`, `plate.ts`, `tracker.ts`, `labels.ts`). Change behaviour in both and keep them in step; browser tests check both read the same plates. The web app runs inference in `worker.ts` (WebGPU/WASM) and keeps residents/log in IndexedDB (`db.ts`); `i18n.ts` handles Persian (RTL)/English.
- `server/`: `app.py` (endpoints), `models.py` (SQLModel), `settings.py` (pydantic-settings, `PLPR_*` env), `webhooks.py` (HMAC-signed), `agent.py` (edge agent with disk buffer while offline), `import_v1.py` (migrate v1 SQLite DBs). Tokens are stored only as SHA-256 hashes.
- `scripts/`: `export_onnx.py`, `make_dataset.py` (synthetic data), `train.py`, `eval.py`, `quantize.py` (FP16/INT8 variants).
- `docker/`: API + agent + go2rtc + optional Postgres.

Legacy v1 code still in the tree: `home-keras.py` + `hdf5/`, `streamlit_app.py`, `ai/`, `database/`, `helper/`, `gui/*.ui`, `enteries_window.py`, `resident*.py`, `services/`, `configParams.py`. The desktop app (`home-yolo.py`) uses these for the UI/DB but the new `plpr` engine for inference. `yolov5/` is vendored upstream code (AGPL) used only for export/training — don't lint or edit it.

## Notes

- SQL must use parameters; sort columns are whitelisted (a v2 security fix).
- Model weights (`model/*.pt`, `hdf5/`) are tracked in git; generated ONNX files in `model/onnx/` are not, and CI regenerates them in a `models` job that the `python` and `web` jobs depend on.
