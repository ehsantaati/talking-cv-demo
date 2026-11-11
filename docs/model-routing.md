# Model routing and provider fallback

[← Project overview](../README.md) · [Backend setup](../backend/README.md)

The scope checker classifies each request using the question and supplied conversation history. Its `ScopeDecision` sets a route and scoped query. LangGraph conditional edges use that route to choose the workflow; the answer node then maps the same route to a model provider. There is one intent classification step, followed by a deterministic provider lookup, rather than a second LLM classifier.

| Scope route | Workflow | Answer provider | Model setting |
| --- | --- | --- | --- |
| `cv` | Retrieve CV evidence, then generate and validate an answer | Azure OpenAI | `AZURE_OPENAI_LLM_DEPLOYMENT` |
| `github` | Require fresh GitHub tool evidence, then generate and validate an answer | OpenAI | `OPENAI_MODEL` |
| `both` | Retrieve CV evidence and require GitHub evidence before answering | Azure OpenAI | `AZURE_OPENAI_LLM_DEPLOYMENT` |
| `conversation` | Answer from session history and validate against that history | OpenAI | `OPENAI_MODEL` |
| `unrelated` / `restricted` | Return a fixed scope or access-limit response | No answer-model call | — |

Failed checks and insufficient evidence can also produce a fixed fallback. Conversation requests without history return a fixed fallback without invoking the answer model. The `both` route uses one Azure answer model for the combined workflow, including its GitHub tool calls; it does not switch providers for each tool.

Scope classification, CV retrieval checks, and answer validation always select Azure as their primary provider through the same gateway; they do not use intent-based provider selection. They can fall back to OpenAI if Azure fails. For CV requests, initial retrieval and its evidence check happen before the answer model is selected.

### Model configuration

Configure providers in the untracked `backend/.env`:

```env
# Azure answer model and primary classification/validation model
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_API_KEY=your-azure-api-key
AZURE_OPENAI_API_VERSION=2024-02-01
AZURE_OPENAI_LLM_DEPLOYMENT=your-azure-deployment-name

# OpenAI answer model for GitHub and conversation routes
OPENAI_API_KEY=your-openai-api-key
OPENAI_MODEL=gpt-4.1
```

`OPENAI_MODEL` takes a plain model name without the `openai/` prefix; the provider configuration adds that prefix. If omitted, it defaults to `gpt-4.1-mini`. Azure uses the deployment name configured in your Azure resource. These LLM settings are separate from `AZURE_OPENAI_DEPLOYMENT_NAME`, which configures embeddings.

The backend route mapping selects the provider. Provider configuration supplies its model and credentials, and the gateway caches model instances per provider. The answer nodes invoke the selected model, while graph assembly defines the workflow.

After changing models or credentials, recreate the backend to reload its environment and cached models:

```bash
docker compose up -d --build --force-recreate backend
```

## Provider fallback

The gateway uses LiteLLM's native `Router` and `ChatLiteLLMRouter` integration with a one-way fallback:

```python
fallbacks=[{"azure": ["openai"]}]
```

Intent routing still selects the primary provider. If an Azure call fails, LiteLLM can attempt the configured OpenAI model with the same messages, tool definitions, tool choice, and structured-output schema. This applies to answer generation and the Azure-primary scope and evidence checks, avoiding a dependency on Azure succeeding before failover is available.

The router uses `num_retries=2` and `max_fallbacks=1`: it allows up to two additional attempts within a model group, then at most one fallback. LiteLLM may skip retries depending on the error and deployment availability. Calls have a 30-second timeout, and the shared router's deployment cooldown duration is configured to 30 seconds. OpenAI has no reverse fallback to Azure. If both providers fail, the existing scope/evidence checks withhold answers; an answer-generation failure propagates to the caller.

This uses LiteLLM's generic provider-failure fallback, not a rate-limit-only policy. Automated tests simulate Azure HTTP `429` and `503`, healthy primary calls, structured scope output, and exhausted providers without making live API calls. The fallback is intended for failed provider calls; seamless continuation of an already partially consumed stream is not enabled.

No extra credentials are required beyond the Azure and OpenAI environment settings above. Rebuild the backend after applying the gateway change. A successful routed answer fallback logs `LLM fallback:` with the route, primary, actual provider, and model. The response's `model_id` also identifies fallback deployments for classification and validation calls. See [LiteLLM's retry and fallback documentation](https://docs.litellm.ai/docs/router/learn/reliability) for the underlying behavior.
