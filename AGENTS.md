# Repository guidance

Read `CLAUDE.md` before making changes. It contains the required privacy/security rules and design invariants; in particular, keep capture local, recordings/transcripts synthetic in tests, and the bounded-memory long-session pipeline intact.

## Project shape

- `backend/` is the FastAPI service (`python -m backend`); `extension/` is a Chrome MV3 extension loaded unpacked, with no build step. `extension/shared.js` holds pure helpers tested with Node.
- Audio is captured/played back by `extension/offscreen.js`, streamed to `/ws/audio`, incrementally decoded by one WebM demuxer per session into on-disk PCM, then transcribed from bounded windows. Whisper models are cached process-wide. Session segments are persisted as recognized; live and stored transcript responses must keep the same shape.
- Keep default backend binding on `127.0.0.1`. Docker listens on `0.0.0.0` internally but `docker-compose.yml` publishes only on host loopback; the unauthenticated API's CORS origins are deliberately scoped.

## Developer checks

```powershell
python -m venv .venv
./.venv/Scripts/python.exe -m pip install -r requirements-dev.txt
./.venv/Scripts/python.exe -m pytest tests/ -q
node --test tests/extension/shared.test.mjs
./.venv/Scripts/python.exe -m backend
```

- Focus Python tests with `./.venv/Scripts/python.exe -m pytest tests/test_decoder.py -q` (or another file / `-k name`). Tests run offline, must not download a Whisper model, and use generated synthetic Opus audio (`make_webm_opus` in `tests/test_decoder.py`).
- `pytest.ini` enables asyncio auto mode, strict markers, and treats warnings as errors. There is no configured lint, formatter, or typecheck command.
- `tests/extension/verify-capture.mjs` is an optional real-browser integration check: start the backend, install `playwright` and its bundled Chromium (`npm install playwright && npx playwright install chromium`), then run `node tests/extension/verify-capture.mjs`. Actual tab stream acquisition still requires a real toolbar click; verify capture manually in Chrome when changing that path.
- The backend creates `recordings/` at startup. Do not add its contents to source control.
