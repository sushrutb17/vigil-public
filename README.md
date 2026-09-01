# VIGIL — Aviation Safety Signal Triage

VIGIL turns batches of public NASA Aviation Safety Reporting System (ASRS)
reports into a ranked list of emerging hazards and source-cited investigator
drafts. It drafts and recommends; a human is the only terminal approval gate.

**Live demo:** [vigil-ui-715230861973.us-central1.run.app](https://vigil-ui-715230861973.us-central1.run.app)

**Demo video:** [VIGIL — All Things Agentic Hackathon](https://youtu.be/kGiqvn-vrv0)

This is the code-focused public companion repository for VIGIL. It contains the
runnable application, tests, configuration, and deployment surface without the
project's internal planning and production-material directories.

## The problem

NASA's Aviation Safety Reporting System receives a large volume of confidential
incident reports that are individually useful but especially valuable in the
aggregate. Several separately filed reports can describe the same emerging
hazard even when no single report contains the complete pattern. VIGIL compresses
those signals into named hazards and source-cited draft briefs for human review.

## How it works

1. Public ASRS reports are normalized and embedded.
2. HDBSCAN groups related reports; severe noise reports remain visible as
   singletons.
3. Deterministic risk scoring ranks clusters against frozen thresholds.
4. Google ADK agents research precedents, draft a brief, and critique its
   citations in parallel.
5. Streamlit presents the evidence and requires a human to edit, approve, or
   reject the draft.

Clustering and risk scoring contain no model calls. The Analyst names hazards but
does not compute risk, and a deterministic citation gate removes factual claims
that do not cite a real ACN from the source reports.

## Run locally

### No credentials required

```bash
uv sync --all-groups
make demo     # end-to-end pipeline on a bundled six-report fixture
make ui       # Streamlit review UI
make check    # ruff + pytest
```

`make ui` serves `artifacts/demo_run.json` when present — a committed snapshot of
a real live run over real ASRS data with 23 hazard clusters, 4 escalations, 1,328
severe singletons, and 5,000 reports triaged. It falls back to the bundled
fixture if that file is absent.

### With real data

```bash
make download    # Hugging Face Parquet export
make run-real    # seeded 5,000-report slice, deterministic stages only
make eval-offline
```

The holdout split is created as a read-only local file and is excluded from
container images.

### With live Gemini agents

Put a Google AI Studio key in a gitignored `.env` file:

```bash
GOOGLE_API_KEY=your-key
```

Then run:

```bash
set -a; source .env; set +a
make run-live   # live Analyst, Coordinator, Precedent, Risk, Writer, and Critic
make artifact   # save a run for the UI
```

Only naming, prose, and brief drafting come from the model. Deterministic
clustering and risk scoring are byte-identical with and without `--live`.

### Offline self-improvement

```bash
make improve
```

The improvement loop is extractor-only and offline. It can revise the Extractor
instruction, never the frozen risk policy or other agents, and evaluates a
candidate against a locked holdout before promotion.

## Measured results

The committed evaluation runs report the following extractor results:

| Extractor prompt | Dev macro-F1 | Dev accuracy | Holdout macro-F1 | Holdout accuracy |
|---|---:|---:|---:|---:|
| Majority-class + keyword baseline | 0.0515 | 0.395 | — | — |
| v1 | 0.0056 | 0.105 | 0.0081 | 0.080 |
| v2, promoted by the loop | **0.4099** | **0.600** | **0.4219** | **0.680** |

The citation gate achieved a 1.000 catch rate for seeded uncited or fabricated
claims while retaining correctly cited claims. Clustering purity against the
coarse `Events_Anomaly` taxonomy was 0.301, versus a 0.219 majority-class
baseline; adjusted Rand was 0.0018 and the noise fraction was 0.837. That weak
clustering result is intentionally visible rather than tuned away immediately.

## Failure tolerance

Coordinator sub-agents fail independently. To exercise the fallback path:

```bash
set -a; source .env; set +a
uv run python -m pipeline.run_batch --demo --live --fail-agent risk
```

One failed sub-agent yields a `DEGRADED` brief whose missing section falls back
to a cited deterministic line. Multiple failures fall back to a fully
deterministic template instead of dropping the cluster.

## Safety boundaries

- Clustering is embeddings plus HDBSCAN only; there are no LLM calls in that
  stage.
- `config/frozen.yaml` is read-only at runtime. Agents cannot retune escalation
  thresholds.
- Brief claims must cite source ACNs, and citations are checked against source
  reports rather than accepted by formatting alone.
- The self-improvement loop can revise only the Extractor instruction.
- The holdout is isolated from prompt revision.
- The human approval gate is terminal. VIGIL never sends, files, or executes a
  recommended action.

## Repository layout

```text
agents/       Google ADK definitions and brief-generation agents
pipeline/     ingestion, embeddings, clustering, risk, and orchestration
eval/         metrics, safety guards, and offline improvement loop
ui/           Streamlit review interface
config/       frozen policy, model IDs, and prompt registry
infra/        Cloud Run deployment script
tests/        unit, integration, and safety-guard tests
```

## Cloud deployment

```bash
gcloud auth login
gcloud config set project your-project
gcloud services enable run.googleapis.com cloudbuild.googleapis.com \
    artifactregistry.googleapis.com secretmanager.googleapis.com

set -a; source .env; set +a
export GOOGLE_CLOUD_PROJECT=your-project
make deploy
```

Deployment creates separate Cloud Run surfaces for the public Streamlit UI and
the batch job. The UI can persist human decisions to Firestore but cannot access
the Gemini secret; the batch job owns model access.

## Data credit

VIGIL uses the public NASA ASRS corpus as packaged by the Hugging Face dataset
`elihoole/asrs-aviation-reports`. The historical data is demonstrated as a
replay, never represented as a live feed. Do not place confidential reports,
credentials, or production secrets in this repository.
