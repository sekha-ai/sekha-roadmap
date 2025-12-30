## Roadmap


### Implementation additions

- [ ] **Implement Homebrew deployment method
brew tap sekha-ai/core
brew install sekha-controller


### Code enhancements:


### Infrastructure Improvements

- [ ] **Split deploy-cloud.sh by Provider** - Refactor `docker/deploy-cloud.sh` into separate provider files (`docker/providers/aws.sh`, `gcp.sh`, `azure.sh`) to improve maintainability as we add more cloud providers. Keep main script as orchestrator/dispatcher.

- [ ] **Rebuild embeddings endpoint (/api/v1/rebuild-embeddings) has TODO:
rust
// TODO: Implement actual rebuild logic in embedding service
