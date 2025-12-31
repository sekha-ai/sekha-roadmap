## Roadmap

VERSION ROADMAP:
- v0.1 (3mo): MVP - CRUD, MCP, Python SDK, Docker
- v0.2 (2mo): Summaries, scoring, VS Code plugin
- v0.3 (2mo): Postgres, K8s, audit logs, commercial licensing
- v1.0 (3mo): Federation, team features, SOC2
- v2.0 (future): Plugins, mobile, cloud-native embeddings


### Implementation additions:

- [ ] **Implement Homebrew deployment method
brew tap sekha-ai/core
brew install sekha-controller

- [ ] **7.4. Hybrid Deployment (Local Core + Cloud GPU)**

**Use case**: Keep data local, but use cloud LLMs for heavy tasks  
**Config:**
```toml
[bridge]
provider = "anthropic"  # or "openai"
api_key = "sk-ant-..."
fallback_to_local = true  # If cloud fails, use Ollama
```

- [ ] **7.5. Multi-Instance Federation (Advanced)**

**Use case**: Team shares memory across locations  
**Architecture:**
- Each user runs local Sekha core
- Bridge syncs encrypted summaries to shared S3/R2
- Uses CRDTs for conflict resolution (label renames, etc.)


### Code enhancements:


### Infrastructure Improvements:

- [ ] **Split deploy-cloud.sh by Provider** - Refactor `docker/deploy-cloud.sh` into separate provider files (`docker/providers/aws.sh`, `gcp.sh`, `azure.sh`) to improve maintainability as we add more cloud providers. Keep main script as orchestrator/dispatcher.

- [ ] **Rebuild embeddings endpoint (/api/v1/rebuild-embeddings) has TODO:
rust
// TODO: Implement actual rebuild logic in embedding service
