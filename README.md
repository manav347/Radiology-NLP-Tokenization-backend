## Report Analysis Backend (FastAPI + MinIO + DynamoDB)

A FastAPI backend for uploading TXT/PDF reports to S3-compatible storage (MinIO), auto-processing them via an OpenAI-powered analyzer, and persisting results in DynamoDB Local. Includes simple user signup/login with hashed passwords and a webhook integration from MinIO to trigger processing.

### Features

- **User**: signup and login with `passlib[bcrypt]` password hashing.
- **Documents**: upload `.txt`/`.pdf` files to MinIO S3 with rich object metadata.
- **Processing**: MinIO webhook calls the API, which downloads the file, analyzes the content using OpenAI Chat Completions, and stores a structured summary.
- **Storage**: DynamoDB Local tables for `users`, `file`, and `document_results`.
- **API Docs**: Interactive Swagger UI at `/docs` when the server is running.

### Tech Stack

- Python 3.11, FastAPI, Uvicorn
- MinIO (S3-compatible) for object storage
- DynamoDB Local for metadata/results
- OpenAI Chat Completions API
- Boto3, Pydantic v2, pdfplumber (PDF parsing)

### Repository Layout

```
backend/
  app/
    auth.py                  # Password hashing and verification
    config.py                # App settings (S3/MinIO, DynamoDB, CORS)
    create_table.py          # Creates DynamoDB Local tables
    document/
      download.py            # Helper to read object bytes from S3
      upload.py              # (placeholder)
    Dockerfile               # Optional: containerizing the API
    docker-compose.yml       # Local infra: DynamoDB Local + Admin, MinIO
    entrypoint.sh            # (unused in current compose; retained)
    main.py                  # FastAPI app, CORS, router registration, S3 bucket ensure
    models.py                # Pydantic models for users/files/results
    openai.py                # OpenAI call wrapper (loads .env)
    requirements.txt         # Python dependencies
    routers/
      document.py            # Upload, list, process webhook, fetch result
      user.py                # Signup/login endpoints
    storage.py               # Boto3 clients and table handles
```

### Prerequisites

- Python 3.11+
- Docker + Docker Compose
- An OpenAI API key if you want to enable analysis

### Environment Variables

- **OPENAI_API_KEY**: required for `/document/process` analysis.
- Optional overrides (defaults work for local):
  - `DYNAMODB_ENDPOINT` (default `http://localhost:8000`)
  - `AWS_ACCESS_KEY_ID` (default `dummy`)
  - `AWS_SECRET_ACCESS_KEY` (default `dummy`)

Note: S3/MinIO settings are defined in `app/config.py` and default to local MinIO: endpoint `http://localhost:9000`, bucket `uploads`, access `minio`, secret `minio123`.

To provide `OPENAI_API_KEY`, you can set it in your shell environment or place a `.env` file in the repo root with:

```
OPENAI_API_KEY=sk-...
```

### Quickstart (Local Dev)

1. Create and activate a virtualenv, then install deps:

```bash
python -m venv .venv
. .venv/Scripts/activate  # on Windows (Git Bash/PowerShell)
# source .venv/bin/activate  # on macOS/Linux
pip install -r app/requirements.txt
```

2. Launch local infrastructure (DynamoDB Local + Admin UI, MinIO):

```bash
docker compose -f app/docker-compose.yml up -d
```

3. Create DynamoDB tables:

```bash
python app/create_table.py
```

4. Run the API (port 8080) and open docs at `http://localhost:8080/docs`:

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8080 --reload
```

5. Confirm services:

- MinIO Console: `http://localhost:9001` (user `minio` / pass `minio123`)
- DynamoDB Admin UI: `http://localhost:8001`

### How Processing Works

1. POST `/document/upload` saves the file to MinIO with object metadata `{uuid, uploader, filename}` and records the upload into DynamoDB `file` table.
2. MinIO, via docker-compose config, sends a webhook to `http://host.docker.internal:8080/document/process` on object PUT events.
3. The API downloads the file, calls OpenAI for a structured medical summary, writes the result to `document_results`, and marks the file record as `processed`.
4. Clients fetch the structured result via `GET /document/result?uuid=...`.

### API Reference

- Base URL: `http://localhost:8080`

#### POST `/signup`

Create a new user.

```bash
curl -X POST http://localhost:8080/signup \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","email":"alice@example.com","password":"secretpw"}'
```

#### POST `/login`

Login with username and password.

```bash
curl -X POST http://localhost:8080/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"secretpw"}'
```

#### POST `/document/upload`

Upload a TXT or PDF file. Optional `username` query parameter (defaults to `anonymous`).

```bash
curl -X POST "http://localhost:8080/document/upload?username=alice" \
  -H "Accept: application/json" \
  -F "file=@path/to/report.txt"
```

Response contains the file record including its generated `uuid`.

#### GET `/document/`

List uploaded files.

```bash
curl http://localhost:8080/document/
```

#### POST `/document/process`

MinIO webhook target. You normally do not call this manually. To simulate locally:

```bash
curl -X POST http://localhost:8080/document/process \
  -H "Content-Type: application/json" \
  -d '{
    "Records": [
      {
        "s3": {
          "bucket": {"name": "uploads"},
          "object": {
            "key": "report.txt",
            "userMetadata": {
              "X-Amz-Meta-Uuid": "<uuid-from-upload-response>",
              "X-Amz-Meta-Uploader": "alice",
              "X-Amz-Meta-Filename": "report.txt"
            }
          }
        }
      }
    ]
  }'
```

#### GET `/document/result?uuid=...`

Fetch the processed summary by UUID.

```bash
curl "http://localhost:8080/document/result?uuid=<uuid>"
```

### Running Everything in Docker (optional)

There is a `fastapi` service scaffold commented out in `app/docker-compose.yml`. To run the API in Docker as well:

1. Uncomment the `fastapi` service and set environment variables, e.g. `OPENAI_API_KEY`.
2. Adjust the MinIO webhook endpoint to call the service name instead of the host, for example:
   - Replace `MINIO_NOTIFY_WEBHOOK_ENDPOINT_PRIMARY: "http://host.docker.internal:8080/document/process"`
   - With `MINIO_NOTIFY_WEBHOOK_ENDPOINT_PRIMARY: "http://fastapi:8080/document/process"`
3. Build and start:

```bash
docker compose -f app/docker-compose.yml up -d --build
```

### Notes and Limitations

- PDF parsing: The project includes `pdfplumber`, but the current implementation in `document/download.py` only decodes object bytes as UTF-8. Add PDF handling if you plan to upload PDFs.
- Test file `app/test_main.http` references endpoints (`/`, `/hello/{name}`) that are not present in the current app; use `/docs` to explore available routes.
- CORS origins are configured in `app/config.py` (`ALLOWED_ORIGINS`). Update for your frontend URL if needed.
- S3 bucket `uploads` is ensured at app startup via `lifespan` in `app/main.py`.

### Troubleshooting

- Webhook not firing: Ensure MinIO is running and the compose `MINIO_NOTIFY_*` variables are present; check MinIO Console Events. If the API runs outside Docker, using `host.docker.internal` enables the container to reach your host on Windows/macOS.
- DynamoDB tables missing: Re-run `python app/create_table.py` and refresh DynamoDB Admin UI (`http://localhost:8001`).
- OpenAI errors: Verify `OPENAI_API_KEY` is set; network egress allowed; check console output for exceptions from `app/openai.py`.
- Permissions/bucket errors: Confirm bucket `uploads` exists (it should be auto-created) and object keys match in webhook payloads.

### License

No license file is provided. Add one if you plan to distribute.
