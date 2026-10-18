Yes. Since your organization already uses **`infra` to refer to cloud infrastructure and deployment configurations**, I'd avoid using `infrastructure/` for your application integrations. It could confuse developers about whether a module contains application code or Azure resources.

For your code generation agent, I'd recommend these alternatives:

| Name            | What it communicates                                                     | My assessment                                             |
| --------------- | ------------------------------------------------------------------------ | --------------------------------------------------------- |
| `integrations/` | Connections to external systems and SDKs                                 | **Best fit** for your use case                            |
| `adapters/`     | Wrappers that translate external APIs into your application's interfaces | Best for a clean, ports-and-adapters architecture         |
| `platform/`     | Shared technical capabilities used across the application                | Good, but potentially too broad                           |
| `connectors/`   | Clients connecting to GitHub, Azure Blob Storage, LLM providers, etc.    | Good, but less suitable for internal technical components |
| `providers/`    | Implementations of interchangeable services, such as LLM providers       | Too narrow for your entire module                         |
| `runtime/`      | Execution environments, workers and agent runtimes                       | Best reserved for runtime-specific components             |

## My recommendation: `integrations/`

Your application will interact with several external systems:

* GitHub and its APIs
* GitHub Copilot SDK
* Azure OpenAI and other LLM providers
* CodeGraph and Graphify
* Azure Blob Storage
* LangSmith
* PostgreSQL

These are primarily external integrations, so the name is intuitive.

I'd structure it like this:

```text
src/codegen/
│
├── api/
├── core/
├── schemas/
├── models/
├── services/
│
├── integrations/
│   ├── llm/
│   │   ├── copilot_client.py
│   │   ├── azure_openai_client.py
│   │   ├── model_factory.py
│   │   └── model_router.py
│   │
│   ├── github/
│   │   ├── client.py
│   │   ├── repository.py
│   │   ├── branches.py
│   │   └── pull_requests.py
│   │
│   ├── indexing/
│   │   ├── base.py
│   │   ├── codegraph.py
│   │   ├── graphify.py
│   │   └── symbol_search.py
│   │
│   ├── storage/
│   │   ├── blob_storage.py
│   │   └── artifact_store.py
│   │
│   ├── observability/
│   │   ├── langsmith.py
│   │   └── metrics.py
│   │
│   └── database/
│       └── postgres.py
│
├── agents/
├── workflows/
├── prompts/
└── validation/
```

Your cloud infrastructure and deployment configurations can remain in the organization's existing `infra/` directory.

### One architectural distinction worth considering

If you want to follow clean architecture principles, there is an even better option:

* **`ports/`** — interfaces your application depends on, such as `LLMClient`, `RepositoryIndexer`, and `ArtifactStore`.
* **`adapters/`** — implementations of those interfaces using Copilot SDK, Azure OpenAI, GitHub APIs, CodeGraph, and Blob Storage.

This makes it easier to replace a provider without changing your agents or business services.

However, it introduces more abstraction and files. For your initial implementation, I'd choose **`integrations/`** because it's straightforward, understandable, and doesn't overengineer the project.

**Final choice: `integrations/`** for the external SDKs and services, with `infra/` reserved for cloud infrastructure and deployment.
