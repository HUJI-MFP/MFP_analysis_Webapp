# MFP Analysis App

A web app for analysing lab data: LC-MS (mass spectrometry), FTIR spectra and plate-reader MIC assays, plus an AI assistant that can see your open sessions. The backend reuses the pure-Python analysis code of the original desktop app, so both give the same results.

What changed and when: [`CHANGELOG.md`](CHANGELOG.md).

**Local development URLs**

| Service | URL |
| --- | --- |
| Frontend (Vite) | http://127.0.0.1:5173 |
| Backend API | http://127.0.0.1:8000 |
| Health check | http://127.0.0.1:8000/api/health |
| OpenAPI docs | http://127.0.0.1:8000/docs |

The frontend proxies every `/api/*` request to the backend on port 8000.

---

## What the app does

### Home

- Start page with a card per module, example data you can open in one click, guided tours, and short walkthrough animations.
- Guided workflows that take you through a task step by step: **Analyse a MIC plate**, **Find my product in an LC-MS run**, **Identify FTIR peaks** and **Prepare figures and SI for a paper**.

### LC-MS

- Load **mzML** (or `.mzML.gz`) files; browse the TIC, UV traces and individual MS1 spectra.
- Extract ion chromatograms (EICs) by *m/z*, integrate features, overlay runs, Kendrick mass-defect plots, spectrum labelling and region summation.
- **Polymer Studio**: search for expected products of a polymerisation (monomers, end groups, mixed dimers, a pGlu preset), with a **Checks** tab that flags doubtful assignments. The MS1 y-axis can show absolute intensity or % relative intensity (Design panel).
- **One-click pipeline** (Analyse whole run): finds the UV peaks (at most 35), aligns UV to MS (falling back to the usual delay when alignment is unclear), and labels each peak from its raw spectrum:
  - [M+H]⁺ with a Na⁺ fallback, a chemistry limit on which products are allowed, series neighbours and isomer clusters;
  - a "distinct peak" rule for overlapping peaks (rise, sharpness or a clear edge), an artifact check, and an EIC-vs-UV shape score;
  - noncovalent dimers ([2M+H]⁺ and [A+B+H]⁺), shown with both readings when either is possible.
- **Review panel** for pipeline labels: approve or reject one by one or in bulk, a "To review" filter, sorting, and keyboard keys (arrows, A, R, Enter).
- **Past runs**: reopen a run's review, download its figure or workbook, or delete it. Runs older than 5 days are cleaned up automatically (`MFP_PIPELINE_RETENTION_DAYS`).
- Exports: SVG, PNG, a 600 dpi print PNG (180 × 100 mm) and `peaks.xlsx`; publication figure export with journal width presets.

### FTIR

- Load spectra from common text/CSV formats.
- Baseline correction, smoothing, atmospheric masking, peak picking (incl. second-derivative mode), peak assignment against a built-in library, reference matching, integration, normalisation and peak fitting.

### Plate Reader

- Import BioTek Gen5 exports or any Excel/CSV sheet with an 8 × 12 plate block; Gen5 notes pre-fill the layout.
- Plate map: dilution series, compound rows, growth control and blank; paint wells, exclude bad wells, shared layout templates.
- MIC results: heatmap, dose–response, % growth, stick plot, optional 4PL IC₅₀ and a results table, each with a "How is this calculated?" panel and automatic checks.
- A calculation workbook (Excel, live formulas) per plate, and an Experiment view that combines plates with the same tag, with a MIC reading grid and figure export.

### AI Assistant

- Chat with context from your open LC-MS / FTIR / Plate Reader sessions.
- Providers: **Demo** (canned answers, no network), **OpenAI** (`OPENAI_API_KEY`), **Ollama** (local LLM at `OLLAMA_BASE_URL`; install [Ollama](https://ollama.com/) separately).
- Can run backend **automation actions** (EICs, integration, exports, …) from the same action catalog the MCP server uses.

### Publication

- Builds a supplementary-information (SI) package from recorded results (`/api/publication`).

### Automation and MCP (optional)

- The backend exposes an **automation API** (`/api/automation/*`) with preview/confirm steps for destructive actions.
- An **MCP server** in `MFP_analysis_app/mcp_server/` lets Claude Code, Cursor or other MCP clients drive LC-MS analysis through the backend. See [`MFP_analysis_app/mcp_server/README.md`](MFP_analysis_app/mcp_server/README.md).

### Desktop app (legacy)

The original Tkinter + Matplotlib GUI lives in `MFP_analysis_app/lab_gui/`. The web backend imports only its **analysis modules**, not its UI. Running the desktop app also needs `matplotlib` and `tkinter` (on some Linux distros: `python3-tk`); start it from `lab_gui/app.py`.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  Browser  →  React 18 + TypeScript + Vite + Tailwind + Plotly│
│              http://127.0.0.1:5173                           │
└──────────────────────────┬───────────────────────────────────┘
                           │  /api/*  (Vite proxy in dev; same origin in production)
┌──────────────────────────▼───────────────────────────────────┐
│  FastAPI backend  →  uvicorn on :8000                        │
│  Routers: lcms | lcms/pipeline | ftir | plate-reader | ai    │
│           workspaces | experiments | examples | publication  │
│           automation                                         │
└──────────────────────────┬───────────────────────────────────┘
                           │  imports
┌──────────────────────────▼───────────────────────────────────┐
│  MFP_analysis_app/lab_gui/  (shared pure-Python analysis)    │
│  lcms_io, lcms_model, ftir_analysis, plate_reader_io, …      │
└──────────────────────────────────────────────────────────────┘
```

| Layer | Stack |
| --- | --- |
| Frontend | React 18, TypeScript (strict), Vite, Tailwind CSS, Plotly (`react-plotly.js`), React Router, Redux Toolkit, Vitest |
| Backend | Python 3.12, FastAPI, Uvicorn, Pydantic v2, SQLite |
| Analysis | NumPy, Pandas, SciPy, Pyteomics, openpyxl |
| AI | OpenAI SDK, httpx (Ollama) |

In production a single container serves both: the Dockerfile builds the frontend (Node 20) and FastAPI serves the built app alongside the API.

---

## Repository layout

```
.
├── README.md, CHANGELOG.md, CLAUDE.md
├── Dockerfile, railway.toml, Procfile, start-backend.sh   ← deployment
├── requirements.txt                   ← includes the backend requirements
├── .github/workflows/ci.yml           ← CI
├── docs/                              ← design specs and plans
└── MFP_analysis_app/
    ├── lab_gui/                       ← shared analysis code + desktop Tk app
    ├── mcp_server/                    ← optional MCP bridge (separate install)
    └── web/
        ├── frontend/                  ← React app (Vite)
        │   └── src/
        │       ├── views/             ← Home, LCMS, FTIR, PlateReader, AI
        │       ├── components/, layout/, theme/, workflows/
        │       └── api.ts             ← typed wrapper for all backend calls
        ├── backend/
        │   ├── app/                   ← main.py, routers/, services/, automation/
        │   ├── benchmarks/            ← LC-MS pipeline benchmark
        │   ├── tests/                 ← pytest suite
        │   ├── requirements.txt       ← backend dependencies
        │   ├── requirements.lock      ← exact versions (Docker and CI)
        │   └── requirements-dev.txt   ← + pytest
        ├── shared_fixtures/           ← sample data
        ├── AI_AUTOMATION_ROADMAP.md
        └── PUBLICATION_PLOT_EXPORT_ROADMAP.md
```

> The backend adds `MFP_analysis_app/` to `sys.path` and imports `lab_gui.*`, so `lab_gui/` must stay in the same checkout.

---

## Requirements

| Requirement | Version |
| --- | --- |
| Python | 3.12 recommended (used by Docker and CI) |
| Node.js | 20 or newer (CI uses 24, which matches the npm that wrote `package-lock.json`) |
| OS | Windows, macOS or Linux (examples below use PowerShell) |

Ollama and the MCP server are separate installs and are not needed for the web app.

---

## Quick start

### 1. Python environment (from the repo root)

```powershell
git clone <repo-url> MFP_Analysis_app
cd MFP_Analysis_app
python -m venv .venv
.\.venv\Scripts\Activate.ps1          # macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
```

### 2. Frontend dependencies

```powershell
cd MFP_analysis_app\web\frontend
npm install
```

### 3. Start the app

**One command** (the Python venv must be active, since it also starts the backend):

```powershell
cd MFP_analysis_app\web\frontend
npm run dev
```

**Or two terminals:**

```powershell
# Terminal 1 — backend
cd MFP_analysis_app\web\backend
python -m uvicorn app.main:app --reload --port 8000

# Terminal 2 — frontend
cd MFP_analysis_app\web\frontend
npm run dev:frontend
```

### 4. Check it works

- http://127.0.0.1:5173 opens on the Home page.
- http://127.0.0.1:8000/api/health returns `{"status":"ok"}`.

Sample data: the example cards on the Home page, or `MFP_analysis_app/web/shared_fixtures/`.

---

## Tests and checks

```powershell
# Frontend: type-check and unit tests
cd MFP_analysis_app\web\frontend
npm run lint
npm run test:run

# Backend tests
cd MFP_analysis_app\web\backend
pip install -r requirements-dev.txt
python -m pytest tests/ -q

# LC-MS pipeline benchmark (scores the pipeline against hand-labelled runs)
python -m benchmarks.lcms_pipeline.run_benchmark
```

GitHub Actions (`.github/workflows/ci.yml`) runs on every pull request and push to `main`: backend tests on the locked dependencies and on pandas 3, the frontend type-check and tests, and a Docker build.

---

## Environment variables

None are needed for the analysis modules.

| Variable | Used for | Default |
| --- | --- | --- |
| `OPENAI_API_KEY` | OpenAI provider in the AI tab | unset (demo provider) |
| `OLLAMA_BASE_URL` | Local Ollama LLM | `http://127.0.0.1:11434` |
| `MFP_DATA_DIR` | Uploads, SQLite database, caches, backups, automation log | `/data` in Docker; locally `/data` if it exists, else `.data` |
| `MFP_MAX_UPLOAD_MB` | Largest accepted upload (HTTP 413 above it) | `2048` |
| `MFP_MAX_DECOMPRESSED_MB` | Largest size an uploaded `.mzML.gz` may expand to | `8192` |
| `MFP_PIPELINE_RETENTION_DAYS` | Age after which LC-MS pipeline runs are deleted | `5` |
| `MFP_CORS_ORIGINS` | Extra origins allowed to call the API from a browser (only if the frontend is hosted elsewhere) | none (same origin only) |
| `MFP_GIT_SHA` | Commit recorded as the app version in results (Railway's `RAILWAY_GIT_COMMIT_SHA` is used automatically) | `git rev-parse HEAD` if available |

---

## Themes

Two themes, **day** and **night**, set via `data-theme` on `<html>`. Plotly charts take their colours from the theme tokens; exported figures keep their print colours.

---

## API overview

| Prefix | Module |
| --- | --- |
| `/api/lcms` | LC-MS sessions, TIC, EIC, spectra, Polymer Studio, exports |
| `/api/lcms/pipeline` | One-click pipeline: start a run, review labels, past runs, figure and workbook downloads |
| `/api/ftir` | FTIR load, preprocessing, peaks, assignment, matching, fitting |
| `/api/plate-reader` | Plate import, layouts and templates, MIC analysis, calculation workbooks |
| `/api/experiments` | Tag sessions across modules and fetch an experiment bundle by tag |
| `/api/workspaces` | Workspaces and lab member profiles |
| `/api/examples` | Example data for the Home page |
| `/api/publication` | SI package from recorded results |
| `/api/ai` | Chat, provider status, context snapshot |
| `/api/automation` | Action catalog, preview/execute, browser WebSocket bridge |

Full interactive docs: http://127.0.0.1:8000/docs

---

## Deployment and operations (Railway)

The app is deployed from the `Dockerfile` (see `railway.toml`) and is **public**: anyone with the Railway link can use it and see all lab data. There is no login by design.

- **Volume.** Attach a Railway volume at `/data`. The image sets `MFP_DATA_DIR=/data`, so the database (`mfp_database.db`), uploads, LC-MS index caches, pipeline runs, the automation log and backups all survive redeploys. Without a volume, everything is lost on each deploy.
- **Nightly database snapshots.** The server copies every SQLite database to `/data/backups/db-YYYYMMDD-HHMMSS/` once a day (and at startup if the last copy is older than a day), keeping the newest 7. Copies are consistent even while the app is writing. They protect against a bad write or accidental deletion, not against losing the volume.
- **Off-site copy.** Uploads are never rewritten, so they need a copy elsewhere rather than versions. Make a full archive (databases + uploads, no caches) from a shell on the service (`railway ssh`), then copy it off Railway:

  ```bash
  cd /app/MFP_analysis_app/web/backend
  python -m app.backup /data/mfp-full.tar.gz
  ```

  To restore, stop the service, extract the archive into `/data`, and start it again.
- **Pipeline cleanup.** LC-MS pipeline runs (and LC-MS files they uploaded themselves) older than `MFP_PIPELINE_RETENTION_DAYS` are deleted at startup and, at most once an hour, when the run list is opened, so the volume doesn't keep growing.
- **Logs.** Server messages are single timestamped lines (`time LEVEL logger: message`) on stdout, visible in Railway's log view. Unexpected errors are shown to users with a reference code; search the logs for that code to find the traceback.
- **AI assistant.** If `OPENAI_API_KEY` is set, every visitor uses it. Set a monthly usage limit on the key's project in the OpenAI dashboard.
- **Health check.** `GET /api/health`.

To try the production image locally:

```powershell
docker build -t mfp-analysis .
docker run -p 8000:8000 -v mfp-data:/data mfp-analysis
```

Then open http://127.0.0.1:8000.

---

## Setup notes

- **Large mzML files:** the first load builds an MS1 index and can take a while; later operations use the cached index.
- **Paths with spaces (Windows):** quote them when you `cd`.

## Contact

Moran's lab.
