# SIH26106

An AI-assisted email threat and forensic intelligence platform. Upload an RFC 5322 `.eml` file to preserve it as evidence, extract forensic indicators, assess authentication and relay signals, and produce an explainable phishing-risk assessment.

> This is an MVP investigative aid. AI conclusions are probabilistic and should be reviewed alongside the preserved evidence.

## What it does

- Preserves uploaded `.eml` files as original evidence
- Parses email headers, MIME structure, body text, attachments, URLs, domains, and IP addresses
- Calculates SHA-256 attachment hashes
- Interprets reported SPF, DKIM, and DMARC results and reconstructs `Received` relay chains
- Builds case timelines and entity graphs, and generates forensic reports
- Separates observed, parsed, inferred, and AI-assessed findings in the investigation workflow
- Optionally uses Google Gemini for structured threat-intent assessment; attachment bytes are never sent to the model
- Optionally enriches public IP indicators through a configured enrichment provider

## Architecture

```text
React + TypeScript (Vite)
           |
        REST API
           |
   Go + Chi modular monolith
     |       |       |
 Parser  Forensics  AI/enrichment
           |
      PostgreSQL     Redis (local infrastructure)
```

The backend deliberately uses a modular monolith: parsing, deterministic forensic analysis, AI adapters, persistence, evidence, reporting, and HTTP transport remain separated internally without adding deployment complexity.

## Quick start

### Prerequisites

- Go (the version declared in [`backend/go.mod`](backend/go.mod))
- Node.js and npm
- Docker and Docker Compose for PostgreSQL/Redis

### 1. Configure local environment

```bash
cp .env.example .env
cp frontend/.env.example frontend/.env
```

Edit `.env` only if you need to change defaults or enable optional integrations. Keep real API keys in your local environment or deployment configuration; never commit them.

### 2. Start local infrastructure

```bash
docker compose up -d
```

This starts PostgreSQL on `localhost:5432` and Redis on `localhost:6379` by default.

### 3. Run the backend

```bash
cd backend
DATABASE_URL='postgres://sih26106:change-me-for-local-development@localhost:5432/sih26106?sslmode=disable' go run ./cmd/api
```

The API listens at `http://localhost:8080`. Confirm it is available with:

```bash
curl http://localhost:8080/api/health
```

### 4. Run the frontend

In another terminal:

```bash
cd frontend
npm install
npm run dev
```

Open the Vite URL shown in the terminal (normally `http://localhost:5173`) and upload an `.eml` file.

## Optional AI and enrichment configuration

The application works without API credentials. In that state, its responses explicitly report that AI assessment or IP enrichment is unavailable rather than fabricating a result.

To enable Google Gemini, set the following in your local environment before starting the backend:

```bash
GEMINI_API_KEY=your_key
GEMINI_MODEL=gemini-2.0-flash
```

The frontend can also configure the supported AI provider for the lifetime of the backend process via `POST /api/settings/ai`. Keys are not returned, logged, or persisted.

For optional passive IP enrichment, configure `IP_ENRICHMENT_API_KEY` and the associated provider URL/settings described in [`.env.example`](.env.example).

## API overview

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `GET` | `/api/health` | Service health check |
| `GET`, `POST` | `/api/settings/ai` | Inspect or configure runtime AI-provider status |
| `POST` | `/api/emails` | Upload an `.eml` file and create a case |
| `POST` | `/api/emails/{email_id}/analysis` | Start an analysis |
| `GET` | `/api/emails/{email_id}` | Retrieve parsed email data |
| `GET` | `/api/emails/{email_id}/analysis` | Retrieve analysis results |
| `GET` | `/api/emails/{email_id}/evidence` | Retrieve recorded evidence |
| `GET` | `/api/cases/{case_id}/graph` | Retrieve the entity graph |
| `GET` | `/api/cases/{case_id}/timeline` | Retrieve the case timeline |
| `POST`, `GET` | `/api/cases/{case_id}/report` | Create or retrieve a report |
| `GET` | `/api/cases/{case_id}/report.pdf` | Download the forensic PDF report |

The upload endpoint accepts multipart form data with a `file` field. Only `.eml` files are accepted, and the current maximum file size is 50 MB. See the full [API contract](docs/api_contract.md).

## Development commands

```bash
# Backend tests
cd backend && go test ./...

# Frontend tests
cd frontend && npm test

# Frontend production build
cd frontend && npm run build
```

## Repository layout

```text
backend/             Go API and forensic-analysis modules
frontend/            React investigation workspace
docs/                Architecture, API contract, data model, and project notes
docker-compose.yml   Local PostgreSQL and Redis services
.env.example         Backend and infrastructure configuration template
```

## Safety and evidence handling

- Uploaded raw emails are treated as preserved evidence; normal API responses do not return the complete raw artifact.
- Attachments are inspected for metadata and hashes, but are not executed or sent as bytes to the AI provider.
- The service does not automatically browse extracted URLs.
- AI confidence reflects confidence in a classification, not confirmation of attacker identity or location.

## Further documentation

- [Architecture](docs/architecture.md)
- [Analysis pipeline](docs/analysis-pipeline.md)
- [Data model](docs/data-model.md)
- [Forensic UI/UX](docs/forensic_ui_ux.md)
- [API contract](docs/api_contract.md)
