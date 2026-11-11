# Deployment

[← Project overview](../README.md) · [Backend setup](../backend/README.md)

## Local Docker Compose

Configure [backend](../backend/README.md#environment-variables) and [frontend](../frontend/README.md#local-setup) environment files, then run from the repository root:

```bash
docker compose up -d --build backend frontend
```

The frontend is available at `http://localhost:8080`; the backend at `http://localhost:8081`. Compose currently overrides `FASTAPI_BASE_URL` with `http://host.docker.internal:8081`. On hosts where that name is unavailable, update the Compose value to `http://backend:8081` for container-to-container access.

The backend image currently neither copies nor mounts the knowledge-base PDFs. Before running [ingestion](knowledge-base.md#updating-the-knowledge-base) inside the container, enable the commented knowledge-base mount in `docker-compose.yml` or supply the PDFs through another mount at `KNOWLEDGE_PATH`.

After changing environment variables, recreate the affected service:

```bash
docker compose up -d --build --force-recreate backend
```

## Azure Container Apps configuration — pending

Before deploying the updated backend image, configure the `talking-cv-backend` Container App with:

| Environment variable | Value |
| --- | --- |
| `QDRANT_URL` | HTTPS Qdrant Cloud cluster endpoint |
| `QDRANT_COLLECTION` | `cv` |
| `QDRANT_API_KEY` | Secret reference to a Container Apps secret, such as `qdrant-api-key` |

In the Azure Portal, add the database API key under the app's **Secrets**, then set `QDRANT_API_KEY` to reference that secret in the container's environment variables. Keep the existing Azure OpenAI settings and use the same embedding deployment as ingestion. The deployed app's retrieval key can use read-only permissions for the collection; ingestion needs write access.

The cloud collection must be populated before deployment. Local ingestion into that same cloud collection updates the knowledge base used by the deployed app without rebuilding its image. Docker Compose services, the local volume, and `backend/.env` settings are not automatically provisioned in Azure by a backend image deployment. Build and push a new backend image tag, update the Container App's image and environment configuration, and verify a CV question after deployment.

## Deployment workflows

The backend deployment workflow is manually triggered. It checks required runtime variable names, builds an image tagged with the Git commit SHA, deploys it, then verifies the image, health endpoint, authenticated metrics API, and chat stream. Configure the Azure, registry, and metrics secrets referenced by that workflow before running it.

Frontend deployment is defined in the web workflow. Environment files and local Docker volumes are not transferred automatically to Azure.
