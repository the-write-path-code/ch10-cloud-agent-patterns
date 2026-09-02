# Chapter 10: From Local Triumph to Cloud Failure

Companion code for *Building Safe Agentic AI for Enterprise Systems* by Mohit Aggarwal.

This repository shows why an agent can appear correct on a laptop and fail after deployment to a stateless, multi-instance cloud service. A process-local cache works when every request reaches the same Python process. It stops being shared state when Cloud Run, Uvicorn workers, or any other scaling mechanism routes requests to separate processes.

The fixed architecture moves retrieval out of the agent process. A stateless agent service asks a dedicated FastAPI retrieval service for evidence, and that service reads from a shared Vertex AI RAG corpus. The system no longer depends on what one container happened to remember.

## What You Will Run

| Chapter section | Demonstration | What it shows |
| --- | --- | --- |
| 10.1 | Local success and cloud failure | A module-level retrieval cache behaves correctly in one Python process and loses its meaning when requests reach separate Cloud Run instances. |
| 10.2 | Concurrent worker isolation | Concurrent requests expose that separate workers do not share a heap, a module-level dictionary, or a cache counter. |
| 10.3 | Stateless agent design | The agent removes process-local retrieval state and asks for evidence through a service boundary. |
| 10.4 | FastAPI retrieval boundary | A dedicated retrieval service queries Vertex AI RAG Engine and returns structured contexts. |
| 10.5 | End-to-end validation | The broken service, retrieval service, and fixed agent service are deployed and checked through health and query endpoints. |

## Production Warning

A module-level cache, Python global, in-memory lock, or process-local session store is not shared state across Cloud Run instances or Uvicorn workers. It may look reliable in local development because a local test often uses one process. That is a runtime accident, not an architecture.

This repository demonstrates a cloud deployment pattern. It does not make a deployed RAG service safe by itself. Authentication, authorization, API rate limits, data retention, network egress, observability, cost limits, and write-path safety require their own controls.

## Prerequisites

### Local development and validation

- Git
- [uv](https://docs.astral.sh/uv/)
- Python 3.12, as required by `pyproject.toml`
- Docker, optional, for local image builds

### Cloud deployment

- Google Cloud CLI (`gcloud`), authenticated to the intended project
- A Google Cloud project with Vertex AI, Artifact Registry, Cloud Build, Cloud Run, and related APIs enabled
- A Vertex AI RAG corpus containing the approved policy documents used by the example
- IAM permissions for the deployment account and Cloud Run service identities

The repository contains local failure demonstrations, but the fixed end-to-end path depends on deployed Cloud Run services and Vertex AI RAG Engine.

## Quick Start

### 1. Install uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 2. Clone and synchronize the repository

```bash
git clone https://github.com/the-write-path-code/ch10-cloud-agent-patterns.git
cd ch10-cloud-agent-patterns
uv sync
```

The repository currently has no committed `uv.lock` or `.python-version` file. `uv sync` resolves dependencies from `pyproject.toml`, which requires Python 3.12. Before public release, generate and commit a lock file and Python-version pin.

### 3. Create local configuration

```bash
cp .env.example .env
```

At minimum, configure the project, region, RAG corpus, and model:

```dotenv
GOOGLE_CLOUD_PROJECT=your-project-id
GOOGLE_CLOUD_LOCATION=us-central1
RAG_CORPUS=projects/your-project-id/locations/us-central1/ragCorpora/your-corpus-id
AGENT_MODEL=gemini-2.0-flash-001
```

For local Google Cloud authentication, use an approved credential method for your environment. The `.env.example` supports `GOOGLE_APPLICATION_CREDENTIALS` for a local service-account file, but do not commit a credential file or store one in the repository directory.

## Configuration

| Variable | Used by | Purpose |
| --- | --- | --- |
| `GOOGLE_CLOUD_PROJECT` | All cloud resources | Target Google Cloud project ID |
| `GOOGLE_CLOUD_LOCATION` | Vertex AI and Cloud Run | Region for the RAG corpus and services |
| `GOOGLE_APPLICATION_CREDENTIALS` | Local development only | Path to a service-account credential file when that authentication method is approved |
| `RAG_CORPUS` | Retrieval service | Full Vertex AI RAG corpus resource name |
| `RETRIEVAL_SERVICE_URL` | Fixed and broken agent services | URL for Service A, the retrieval boundary |
| `AGENT_SERVICE_URL` | Client validation | URL for Service B, the fixed stateless agent |
| `BROKEN_AGENT_URL` | Failure demonstration | URL for the intentionally stateful broken agent |
| `AGENT_MODEL` | Agent services | Gemini model used by the ADK agent |

All resource names and service URLs belong in deployment-time configuration. Do not hard-code them in application code.

> **Tip**
>
> Run the broken-state demonstration before deploying the fixed service. The contrast is the chapter: one process remembers because it happens to stay alive; the fixed design retrieves from shared infrastructure every time.

## Run the Chapter Demonstrations

### 1. Demonstrate Process-Local State Loss, Sections 10.1 and 10.2

Run the intentionally broken agent demonstration:

```bash
uv run python broken_agent/run_stateless_failure.py
```

The broken implementation stores retrieval results and cache counters in module-level variables. The demonstration makes worker isolation observable by sending requests that reach more than one instance or process.

The expected signal is not an exception. Each request can still return a valid response. The diagnostic fields show that the requests reached different containers and did not share cached state.

### 2. Deploy the Retrieval Boundary, Section 10.4

Service A is the FastAPI retrieval service. It receives a query, asks Vertex AI RAG Engine for contexts, and returns structured JSON.

Before deployment, validate local configuration and Google Cloud access with the repository scripts:

```bash
uv run python scripts/validate_env.py
uv run python scripts/validate_gcp.py
uv run python scripts/validate_rag.py
```

Deploy Service A using the repository deployment script:

```bash
bash scripts/deploy_service_a.sh
```

After deployment, save the service URL in `.env`:

```dotenv
RETRIEVAL_SERVICE_URL=https://your-retrieval-service-url
```

### 3. Deploy the Fixed Stateless Agent, Sections 10.3 through 10.5

Service B receives the user query and asks Service A for evidence. It does not maintain an in-process retrieval cache.

Deploy it after `RETRIEVAL_SERVICE_URL` is set:

```bash
bash scripts/deploy_service_b.sh
```

Save the deployed URL:

```dotenv
AGENT_SERVICE_URL=https://your-agent-service-url
```

The fixed agent discovers the retrieval service through `RETRIEVAL_SERVICE_URL` injected at deployment time. It does not require a hard-coded URL or memory shared between containers.

### 4. Validate the Fixed Service, Section 10.5

Call the fixed agent:

```bash
curl -X POST "$AGENT_SERVICE_URL/query" \
  -H "Content-Type: application/json" \
  -d '{"query":"What is the remote work policy?"}'
```

Call the retrieval service directly:

```bash
curl -X POST "$RETRIEVAL_SERVICE_URL/retrieve" \
  -H "Content-Type: application/json" \
  -d '{"query":"What is the vacation policy?"}'
```

Then run the end-to-end validation script:

```bash
uv run python scripts/validate_services.py
```

### 5. Compare Broken and Fixed Deployments

When the intentionally broken Cloud Run service is deployed, set its URL:

```dotenv
BROKEN_AGENT_URL=https://your-broken-agent-service-url
```

Use the failure script and workflow diagrams to compare the two designs:

```text
Broken path:
request -> container A -> in-process cache
next request -> container B -> empty in-process cache

Fixed path:
request -> stateless agent -> retrieval service -> Vertex AI RAG corpus
```

The fixed path can be slower than a warm in-process cache, but it has a defined shared source of evidence that every worker can reach.

## Expected Results

### Broken service

The broken demonstration should show that requests can land on distinct workers or Cloud Run instances. The service-local cache counter does not represent system-wide state, and a value stored in one process is unavailable in another.

A useful diagnostic result includes:

```text
unique_containers: 2
shared_state: false
```

The exact container identifiers vary by deployment.

### Fixed service

The fixed agent should return a grounded response based on structured contexts from the retrieval service. The absence of a local cache hit is correct by design. The agent does not need to remember a prior retrieval result because it can ask the shared retrieval boundary for current evidence.

### Retrieval service

A direct `/retrieve` request should return a structured `contexts` array. The chapter example expects five retrieved contexts for its sample policy query, subject to the configured retrieval settings and corpus contents.

## Run the Tests

```bash
uv run pytest
```

Run the test suite before changing service contracts, environment-variable names, RAG corpus integration, request routing, or deployment scripts. The tests and validation scripts should establish that:

- The broken agent depends on process-local state.
- The fixed agent does not retain retrieval state in its own process.
- The retrieval service returns a documented structured response.
- Service B discovers Service A through configuration.
- Health checks and end-to-end validation distinguish a failed service dependency from an empty successful retrieval.

## Repository Layout

```text
.
├── README.md
├── pyproject.toml
├── .env.example
├── broken_agent/
│   ├── stateful_agent.py             # Module-level cache anti-pattern
│   ├── app.py                        # Broken-agent FastAPI wrapper
│   ├── Dockerfile
│   └── run_stateless_failure.py       # Sections 10.1 and 10.2 demonstration
├── fixed_agent/                       # Stateless-agent implementation, if retained separately
├── retrieval_service/
│   ├── app.py                        # Service A FastAPI retrieval boundary
│   └── Dockerfile
├── agent_service/
│   ├── app.py                        # Service B ADK agent backend
│   └── Dockerfile
├── test_client/                       # Request and validation helpers
├── scripts/
│   ├── deploy_service_a.sh
│   ├── deploy_service_b.sh
│   ├── validate_env.py
│   ├── validate_gcp.py
│   ├── validate_rag.py
│   └── validate_services.py
├── workflow/
│   ├── 01_local_failure.md
│   ├── 02_concurrent_workers.md
│   ├── 03_stateless_fix.md
│   ├── 04_retrieval_service.md
│   ├── 05_end_to_end_cloud_run.md
│   ├── 06_env_wiring.md
│   └── 07_broken_vs_fixed.md
└── tests/
```

## Architecture Diagrams and Supporting Documents

The `workflow/` directory contains the Chapter 10 diagrams:

- A local cache succeeding in one process and failing across Cloud Run instances.
- Concurrent worker memory isolation.
- The stateless-agent fix.
- The FastAPI retrieval boundary.
- The end-to-end Cloud Run request path.
- Deployment-time environment wiring.
- A side-by-side broken-versus-fixed architecture comparison.

Start with `01_local_failure.md`, then read `03_stateless_fix.md`. The rest of the chapter follows from that contrast: memory in a container is not shared system state.

## Safety and Operational Limits

- This architecture externalizes retrieval state. It does not make the RAG corpus correct, current, access-controlled, or safe to expose to every caller.
- Vertex AI RAG corpora can contain sensitive documents. Apply IAM, service-to-service authentication, document-ingestion review, data retention, and logging controls before using real material.
- The supplied deployment scripts use `--allow-unauthenticated`. Do not use that setting for an enterprise knowledge service without an explicit access-control decision. Prefer authenticated invocation and least-privilege service identities.
- Service URLs, RAG corpus names, project IDs, and model names are deployment configuration. Keep them out of source code and do not place secrets in `.env.example`.
- A service account key file is a high-value credential. Prefer workload identity, application-default credentials, or managed service identities where possible. If a local key file is required, store it outside the repository and rotate it according to policy.
- Cloud Run scaling proves that local memory cannot coordinate workers. It does not solve duplicate writes, stale state, or transaction safety. Chapters 11 through 13 address those boundaries.

## Troubleshooting

### `uv sync` uses the wrong Python version

The project requires Python 3.12. Check available interpreters:

```bash
uv python list
```

Then rerun `uv sync`. Before public release, add `.python-version` and `uv.lock` so the supported version and dependency set are explicit.

### Local validation cannot authenticate to Google Cloud

Confirm that `gcloud` is authenticated to the intended project and that the local credential method matches your organization's policy. Check `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION`, and `GOOGLE_APPLICATION_CREDENTIALS` if your local path uses a service-account file.

### The retrieval service cannot find the RAG corpus

Verify the full `RAG_CORPUS` resource name, project, region, enabled Vertex AI API, and service identity permissions. Run:

```bash
uv run python scripts/validate_rag.py
```

before redeploying a service.

### Service B returns 502 or cannot retrieve evidence

Check that `RETRIEVAL_SERVICE_URL` was injected into the deployed Service B revision. Test Service A's `/health` and `/retrieve` endpoints directly before changing agent code.

### The broken-state demonstration shows one container only

The failure needs more than one worker or Cloud Run instance. Check the broken-service deployment configuration and send concurrent requests. A one-instance run can hide the architecture flaw.

### Docker build fails

The repository currently has a version mismatch to resolve: `pyproject.toml` requires Python 3.12 while the exported service Dockerfiles use `python:3.11-slim`. Update the Docker base images to Python 3.12 before treating the container path as release-ready.

## Related Chapters

- Chapter 2 separates deterministic stages from model-driven work and explains why one monolithic loop is difficult to inspect.
- Chapters 3 through 5 establish retrieval, grounding, and corrective-routing controls before cloud deployment.
- Chapter 9 introduces Cloud Run deployment for a multimodal application.
- Chapters 11 through 13 add persistence boundaries, idempotency, and optimistic concurrency control for agent actions that write state.
- Chapter 15 adds continuous evaluation, drift detection, and CI gates for systems that keep changing after deployment.

## License and Errata

See `LICENSE` for licensing terms. Report documentation or code issues through this repository's GitHub issue tracker.
