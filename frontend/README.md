# Talking CV frontend

[← Project overview](../README.md) · [Backend](../backend/README.md)

The Laravel frontend gives recruiters a conversational interface with starter questions, session history, conversation reset, streamed answers, and visible agent activity.

## Local setup

Requires PHP 8.3 or later, Composer, and Node.js/npm. From `frontend/`:

```bash
composer install
cp .env.example .env
php artisan key:generate
```

Set these values in the untracked `.env` for a locally running backend:

```env
APP_URL=http://localhost:8080
FASTAPI_BASE_URL=http://localhost:8081
SESSION_DRIVER=file
CACHE_STORE=file
QUEUE_CONNECTION=sync
```

File sessions/cache and a synchronous queue avoid needing a local database for those services. To retain the example's database-backed settings instead, configure the database and run `php artisan migrate`.

Build assets and start Laravel:

```bash
npm ci
npm run build
php artisan serve --host=0.0.0.0 --port=8080
```

Start the [backend](../backend/README.md#local-setup) separately and open `http://localhost:8080`. For containers, use the [Docker guide](../docs/deployment.md#local-docker-compose); Compose overrides the backend URL.

## Streaming and live architecture

The browser sends questions through Laravel's `/ask/stream` route, which forwards requests to the backend's `/asks/stream/` endpoint. The backend returns newline-delimited JSON events for answers and agent activity.

The Live Agent Architecture chart shows the classified route beneath Router and the selected provider/model beneath AI Agent or Conversation. Labels reset for each question and remain empty while idle or when no answer model is invoked. Start events identify the primary selection; end events identify the actual serving provider after successful fallback.

See [model routing](../docs/model-routing.md) for route decisions and [model-selection events](../docs/observability.md#observing-model-selection) for backend diagnostics. Rebuild both services when changes affect the chart and backend stream:

```bash
docker compose up -d --build backend frontend
```

Run that Compose command from the repository root.

## Tests

From `frontend/`:

```bash
composer test
```

Production configuration and image deployment are covered in the [deployment guide](../docs/deployment.md#deployment-workflows).
