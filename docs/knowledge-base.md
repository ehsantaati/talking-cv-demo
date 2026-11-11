# Knowledge base and retrieval

[← Project overview](../README.md) · [Backend setup](../backend/README.md)

The backend searches the `cv` collection in Qdrant Cloud through LangChain's `QdrantVectorStore`, using an HTTPS cluster endpoint and a database API key. Cloud ingestion and retrieval from the local application have been verified; deploying this integration to Azure Container Apps is pending. A separate Docker Compose Qdrant service remains available for local development.

The ingestion pipeline loads PDF pages from the knowledge-base directory, splits their extracted text into chunks, generates Azure OpenAI embeddings, and stores vectors, chunk text, and metadata in Qdrant. Metadata includes source filename, document ID, document type (currently `document`), a SHA-256 document version, zero-based page number, start index in extracted page text, and a stable chunk UUID. These fields identify the source text; they do not contain PDF bounding boxes.

The retrieval service connects to the existing collection and embeds each question to search for similar document vectors. It does not re-ingest PDFs or store questions in the knowledge base. Ingestion and retrieval use the same embedding deployment.

### Qdrant Cloud setup

Create a cluster in [Qdrant Cloud](https://cloud.qdrant.io/), copy its cluster endpoint, and create a database API key with write access for ingestion. See the [official quickstart](https://qdrant.tech/documentation/cloud-quickstart/). Keep the key in your untracked environment file; do not commit it.

Set these values in `backend/.env` alongside the existing Azure configuration:

```env
QDRANT_URL=https://your-cluster-endpoint:6333
QDRANT_COLLECTION=cv
QDRANT_API_KEY=your-database-api-key
KNOWLEDGE_PATH=/app/knowledge-base
```

From the project root, start the services and run ingestion before asking CV questions:

```bash
docker compose up -d --build backend frontend
docker compose exec backend uv run python -m backend.ingest
```

Inspect the collection and point payloads in the Qdrant Cloud console. After changing environment values, recreate the backend so it receives the new configuration:

```bash
docker compose up -d --build --force-recreate backend
```

### Updating the knowledge base

Add or update PDFs in `knowledge-base/`, then run:

```bash
docker compose exec backend uv run python -m backend.ingest
```

This writes to the endpoint configured in the running backend's `QDRANT_URL`. Stable IDs update existing points at the same document, page, and chunk offset, avoiding duplicates on unchanged re-ingestion. Removed or shifted chunks are not automatically cleaned up yet, and re-ingestion still generates document embeddings. Backend restarts reuse the stored collection without ingestion.

### Optional local Qdrant

To use the Docker database instead of Qdrant Cloud, set `QDRANT_URL=http://qdrant:6333` and `QDRANT_API_KEY=` in `backend/.env`, then run:

```bash
docker compose up -d qdrant
docker compose up -d --build --force-recreate backend
docker compose exec backend uv run python -m backend.ingest
```

Inspect local points at [the local dashboard](http://localhost:6333/dashboard). Local data persists in the `qdrant_data` named volume; deleting that volume deletes the local database. Cloud and local collections are separate and must each be populated. When using the cloud endpoint, the local service can be stopped with `docker compose stop qdrant`.

For cloud deployment requirements, see [Azure configuration](deployment.md#azure-container-apps-configuration--pending).
