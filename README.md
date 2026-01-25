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

- [ ] **Future Tool: memory_analyze**

*Description:* AI-powered conversation analysis that generates insights, summaries, and identifies patterns across your memory.

*Value Proposition:*
- Automatically generates executive summaries of long conversations
- Identifies recurring themes and topics across multiple conversations
- Extracts key decisions, action items, and open questions
- Provides sentiment analysis and interaction patterns
- Enables "conversation analytics" for power users

*Controller Requirements:*
- New endpoint: `POST /api/v1/conversations/analyze`
- Requires LLM Bridge integration (calls Ollama/OpenAI/Anthropic)
- Needs orchestrator service for multi-conversation analysis
- New repository method: `find_conversations_by_theme()`

*Implementation Complexity:* High
- Requires new `analyze.rs` service in controller
- Needs embedding-based theme clustering
- LLM prompt engineering for summaries
- Cost management for LLM API calls

*Priority:* Medium (after core tools are stable)
*Estimated Effort:* 2-3 days (controller) + 1 day (MCP) + 1 day (tests)

- [ ] **Auto-naming: Folders/labels** - Future Feature, Not Core

User creates:
  /work/project-alpha
  /personal/recipes
  
User adds labels:
  #urgent, #reference, #completed

AI suggests (optional, future):
  "This looks like a project planning conversation. 
   Suggested folder: /work/project-alpha 
   Suggested labels: #planning, #technical"
  
  [Accept] [Ignore] [Never suggest again]

- [ ] **Pattern Recognition** - new function to analyze all conversations/messages for patterns (that can be displayed to the user) to understand things that are not immediately obvious and be more of a 30,000 ft. view of your 'memory'.


### Code enhancements:


### Infrastructure Improvements:

- [ ] **Split deploy-cloud.sh by Provider** - Refactor `docker/deploy-cloud.sh` into separate provider files (`docker/providers/aws.sh`, `gcp.sh`, `azure.sh`) to improve maintainability as we add more cloud providers. Keep main script as orchestrator/dispatcher.

- [ ] **Rebuild embeddings endpoint (/api/v1/rebuild-embeddings) has TODO:
rust
// TODO: Implement actual rebuild logic in embedding service



## Considerations:

2. Requires further discussion.
4. I will make note to implement
6. I will make note to implement
8. Yes, we may allow specific import formats in the future.
9. I will make note to implement
10. I will consider for the future based on user input/feedback
11. Not a fan of either of those suggestions. This requires further discussion.
12. I will make note to implement
13. I believe this require further discussion.
14. This will require a bit more discussion and planning.
15. I will make note to implement in the future
16. I will make note to implement in the future
17. I will make note to implement in the future
18. I will make note to implement in the future
19. I will make note to implement in the future
21. I will make note to implement in the future
23. I will make note to implement in the future
24. I will make note to implement in the future
25. I will make note to implement in the future
26-30. I will make note to implement in the future
    
## **2.Chroma connection handling**

From `chroma_client.rs`, the error propagation:
```rust
pub async fn search(&self, query_embedding: Vec<f32>) -> Result<Vec<ChromaResult>, ChromaError> {
    let response = self.client.post(url).json(&body).send().await?;
    // The ? operator propagates connection errors up
}
```

Then in the repository layer, there might be fallback logic I missed. Let me check if the controller has a hybrid search strategy where FTS5 kicks in when Chroma fails. I should examine the `semantic_search` implementation in the repository trait to see if it has fallback behavior.

***

## **4. Tokenizer Library and Better Token Budget Math**

**Is a tokenizer library required?** Technically no, but accuracy matters for context assembly.

**Current Math Problems**:
```rust
let msg_tokens = message.content.len() / 4;
```

This assumes:
- All characters are ASCII (1 byte = 0.25 tokens)
- No language variance (Chinese/Japanese use more tokens per char)
- No special token overhead (system messages, role tags)

**Better Implementation Without External Library**:

```rust
/// Improved token estimation using Unicode-aware heuristics
fn estimate_tokens(text: &str) -> usize {
    let mut tokens = 0;
    
    // Count by Unicode grapheme clusters
    let graphemes: Vec<&str> = text.graphemes(true).collect();
    
    for grapheme in graphemes {
        let char_code = grapheme.chars().next().unwrap() as u32;
        
        tokens += match char_code {
            // ASCII: ~0.25 tokens per char (4 chars per token)
            0x0000..=0x007F => 1,
            
            // Latin Extended, Cyrillic: ~0.5 tokens per char
            0x0080..=0x024F => 2,
            
            // CJK (Chinese, Japanese, Korean): ~1.5 tokens per char
            0x4E00..=0x9FFF | 0x3040..=0x309F | 0x30A0..=0x30FF => 6,
            
            // Arabic, Hebrew, Thai: ~0.75 tokens per char
            0x0600..=0x06FF | 0x0590..=0x05FF | 0x0E00..=0x0E7F => 3,
            
            // Emoji and symbols: ~1 token each
            0x1F300..=0x1F9FF => 4,
            
            // Everything else: conservative estimate
            _ => 2,
        };
    }
    
    // Account for special tokens and formatting
    let overhead = match text {
        t if t.contains("```") => 10,  // Code blocks add tokens
        t if t.starts_with("##") => 5,  // Markdown headers
        _ => 2,  // Base overhead for role/separator tokens
    };
    
    (tokens / 4) + overhead
}

/// Production version with external tokenizer (optional dependency)
#[cfg(feature = "accurate-tokenization")]
fn estimate_tokens_accurate(text: &str, model: &str) -> usize {
    use tiktoken_rs::get_bpe_from_model;
    
    let bpe = get_bpe_from_model(model).expect("Model tokenizer not found");
    bpe.encode_with_special_tokens(text).len()
}
```

**Why this is better**:
- Unicode-aware (handles international text)
- Language-specific heuristics (CJK gets proper weight)
- Overhead accounting (special tokens, formatting)
- Optional accurate mode with real tokenizer behind feature flag

**Feature flag approach**:
```toml
[dependencies]
tiktoken-rs = { version = "0.5", optional = true }

[features]
accurate-tokenization = ["tiktoken-rs"]
```

This lets users choose: fast heuristic (default) or accurate external library (opt-in).

***

## **5. Race condition in JS SDK vs Python SDK**

**My claim**: JS SDK has race condition, Python doesn't.

**JavaScript SDK** (from `client.ts`):
```typescript
async acquire(): Promise<void> {
    const now = Date.now();
    this.requests = this.requests.filter(time => now - time < this.windowMs);
    
    if (this.requests.length >= this.limit) {
        await new Promise(resolve => setTimeout(resolve, waitMs));
        return this.acquire();  // RECURSIVE RETRY
    }
    
    this.requests.push(now);  // ❌ NO LOCK
}
```

**The Race Condition**:
```typescript
// Two concurrent calls:
Call A: checks length (99) < 100 ✅
Call B: checks length (99) < 100 ✅
Call A: pushes to array (now 100)
Call B: pushes to array (now 101) ❌ OVER LIMIT
```

**BUT** - you're right about the use case. A single developer using the JS SDK (browser or Node.js) isn't making 100 concurrent requests. The race condition exists in theory but is **unlikely in practice** for the intended single-user scenario.

**Fixed Implementation** (if we care about correctness):

```typescript
class RateLimiter {
  private requests: number[] = [];
  private acquireQueue: Promise<void> = Promise.resolve();
  
  async acquire(): Promise<void> {
    // Chain acquisitions to serialize access
    const previousAcquire = this.acquireQueue;
    
    let resolver: () => void;
    this.acquireQueue = new Promise(resolve => {
      resolver = resolve;
    });
    
    await previousAcquire;
    
    try {
      const now = Date.now();
      this.requests = this.requests.filter(time => now - time < this.windowMs);
      
      if (this.requests.length >= this.limit) {
        const waitMs = this.windowMs - (now - this.requests);
        await new Promise(resolve => setTimeout(resolve, waitMs));
        return this.acquire();
      }
      
      this.requests.push(now);
    } finally {
      resolver!();
    }
  }
}
```

**Verdict**: Theoretical issue, not practical concern for single-user. But correct implementation costs nothing.

***

## **6. Embedding Dimension Validation - World-Class Design**

### **The Problem**
```rust
// Today: No validation
let embedding = llm_bridge.embed_text(text).await?;
chroma.add(id, embedding, metadata).await?;  // YOLO
```

If Ollama model changes from `nomic-embed-text` (768D) to `mxbai-embed-large` (1024D), Chroma rejects the insert but the message is already in SQLite.

### **World-Class Solution**

**Architecture**:
1. **Collection Versioning**: Track embedding model + dimensions
2. **Validation Layer**: Check dimensions before storage
3. **Migration Support**: Re-embed when model changes
4. **Graceful Degradation**: Fall back to keyword search on mismatch

**Implementation**:

```rust
// src/storage/embedding_config.rs
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EmbeddingConfig {
    pub id: Uuid,
    pub model_name: String,
    pub dimensions: usize,
    pub distance_metric: DistanceMetric,
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub is_active: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum DistanceMetric {
    Cosine,
    L2,
    InnerProduct,
}

// Storage layer
impl EmbeddingConfigRepository {
    pub async fn get_active_config(&self) -> Result<EmbeddingConfig, RepositoryError> {
        use crate::storage::entities::embedding_configs;
        
        embedding_configs::Entity::find()
            .filter(embedding_configs::Column::IsActive.eq(true))
            .one(self.db)
            .await?
            .ok_or_else(|| RepositoryError::NotFound("No active embedding config".into()))
    }
    
    pub async fn create_config(&self, model: String, dims: usize) -> Result<EmbeddingConfig, RepositoryError> {
        // Deactivate old config
        embedding_configs::Entity::update_many()
            .col_expr(embedding_configs::Column::IsActive, Expr::value(false))
            .exec(self.db)
            .await?;
        
        // Create new config
        let config = EmbeddingConfig {
            id: Uuid::new_v4(),
            model_name: model,
            dimensions: dims,
            distance_metric: DistanceMetric::Cosine,
            created_at: chrono::Utc::now(),
            is_active: true,
        };
        
        embedding_configs::ActiveModel {
            id: Set(config.id),
            model_name: Set(config.model_name.clone()),
            dimensions: Set(config.dimensions as i32),
            distance_metric: Set(serde_json::to_string(&config.distance_metric)?),
            created_at: Set(config.created_at.naive_utc()),
            is_active: Set(true),
        }
        .insert(self.db)
        .await?;
        
        Ok(config)
    }
}

// Validation layer
pub struct EmbeddingValidator {
    config: Arc<RwLock<EmbeddingConfig>>,
    repo: Arc<dyn EmbeddingConfigRepository>,
}

impl EmbeddingValidator {
    pub async fn validate(&self, embedding: &[f32]) -> Result<(), ValidationError> {
        let config = self.config.read().await;
        
        if embedding.len() != config.dimensions {
            return Err(ValidationError::DimensionMismatch {
                expected: config.dimensions,
                got: embedding.len(),
                model: config.model_name.clone(),
            });
        }
        
        Ok(())
    }
    
    pub async fn validate_or_refresh(&mut self, embedding: &[f32]) -> Result<(), ValidationError> {
        match self.validate(embedding).await {
            Ok(()) => Ok(()),
            Err(ValidationError::DimensionMismatch { got, .. }) => {
                // Auto-detect model change
                warn!("Embedding dimension mismatch, refreshing config");
                let new_model = self.detect_model_from_dimensions(got).await?;
                self.update_config(new_model, got).await?;
                Ok(())
            }
            Err(e) => Err(e),
        }
    }
    
    async fn detect_model_from_dimensions(&self, dims: usize) -> Result<String, ValidationError> {
        // Query LLM bridge for current model
        let models = self.llm_bridge.list_models().await?;
        
        for model in models {
            // Test embed to get dimensions
            let test_embedding = self.llm_bridge.embed_text("test", Some(&model)).await?;
            if test_embedding.len() == dims {
                return Ok(model);
            }
        }
        
        Err(ValidationError::UnknownModelDimensions(dims))
    }
    
    async fn update_config(&mut self, model: String, dims: usize) -> Result<(), ValidationError> {
        let new_config = self.repo.create_config(model, dims).await?;
        *self.config.write().await = new_config;
        Ok(())
    }
}

// Migration 008
CREATE TABLE IF NOT EXISTS embedding_configs (
    id UUID PRIMARY KEY,
    model_name TEXT NOT NULL,
    dimensions INTEGER NOT NULL,
    distance_metric TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT false
);

CREATE INDEX idx_embedding_configs_active 
ON embedding_configs(is_active) 
WHERE is_active = true;

// Store embedding_config_id with each message
ALTER TABLE messages ADD COLUMN embedding_config_id UUID REFERENCES embedding_configs(id);
CREATE INDEX idx_messages_embedding_config ON messages(embedding_config_id);

// Chroma collection versioning
pub async fn create_or_get_collection(&self, config: &EmbeddingConfig) -> Result<String, ChromaError> {
    let collection_name = format!("sekha_{}_{}", config.model_name, config.dimensions);
    
    // Create if doesn't exist
    if !self.collection_exists(&collection_name).await? {
        self.create_collection(&collection_name, config.dimensions, &config.distance_metric).await?;
    }
    
    Ok(collection_name)
}
```

**Migration Command**:
```bash
sekha migrate-embeddings --from nomic-embed-text --to mxbai-embed-large

# Re-embeds all messages with new model
# Creates new Chroma collection
# Updates embedding_config
# Background job processes 1000 messages at a time
```

**Benefits**:
- Automatic model detection
- Graceful handling of dimension changes
- Full audit trail of which embeddings use which model
- Safe migration path

***

## **7. Single Point of Failure - Reconsidered with Deployment Model**

You're right - I was thinking cloud HA. **For single-user deployment**:

**Actual Usage**:
- Developer runs Sekha on their laptop/workstation
- Or deploys to personal VPS/homelab
- Controller crash = restart container (Docker auto-restart)
- Data persists in SQLite + Chroma volumes

**This is fine**. Personal tools don't need five-nines uptime.

**However**, if a user WANTS HA (e.g., team deployment), here's a world-class design:

### **Optional HA Deployment (Kubernetes)**

```yaml
# kubernetes/sekha-ha.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sekha-controller
spec:
  replicas: 3
  serviceName: sekha-controller
  selector:
    matchLabels:
      app: sekha-controller
  template:
    spec:
      containers:
      - name: controller
        image: sekha/controller:latest
        env:
        - name: DATABASE_URL
          value: "postgresql://sekha:pass@postgres:5432/sekha"  # Shared Postgres instead of SQLite
        - name: CHROMA_URL
          value: "http://chroma-lb:8000"  # Load-balanced Chroma cluster
        - name: REDIS_URL
          value: "redis://redis-sentinel:26379"  # Redis Sentinel for HA
        volumeMounts:
        - name: config
          mountPath: /etc/sekha
      volumes:
      - name: config
        configMap:
          name: sekha-config
---
apiVersion: v1
kind: Service
metadata:
  name: sekha-controller
spec:
  type: LoadBalancer
  selector:
    app: sekha-controller
  ports:
  - port: 8080
    targetPort: 8080
```

**But this is overkill for 99% of users**. Single-container deployment is correct default.

***

## **8. Batch Operations - Revisited**

- Proxy/MCP captures conversations in real-time, one at a time
- Not a batch import system
- Human generates conversations sequentially
- No need for `create_many()`

**However**, there IS one batch scenario: **initial setup**.

When a user first deploys Sekha and wants to import their last month of Claude conversations from Anthropic's dashboard export, they'd have JSON files.

**But you said**: "If we decide to implement import, it would be from another sekha.db."

**That makes perfect sense**. Sekha-to-Sekha import, not arbitrary JSON import.

### **World-Class sekha.db Import Design**

```rust
// POST /api/v1/admin/import
pub async fn import_from_sekha_db(
    Path(source_db_path): Path<String>,
    Query(params): Query<ImportParams>,
) -> Result<Json<ImportResult>, ApiError> {
    let source_conn = SqlitePool::connect(&format!("sqlite://{}", source_db_path)).await?;
    
    // Validate schema compatibility
    let source_version = get_schema_version(&source_conn).await?;
    let target_version = get_schema_version(&state.db).await?;
    
    if source_version != target_version {
        return Err(ApiError::IncompatibleSchema {
            source: source_version,
            target: target_version,
        });
    }
    
    // Import with conflict resolution
    let import_job = ImportJob::new(source_conn, state.db.clone(), params);
    let result = import_job.execute().await?;
    
    Ok(Json(result))
}

pub struct ImportParams {
    pub conflict_strategy: ConflictStrategy,
    pub label_prefix: Option<String>,  // Prefix imported labels with "imported:"
    pub folder_mapping: HashMap<String, String>,  // Remap folders
}

pub enum ConflictStrategy {
    Skip,          // Skip if conversation with same ID exists
    Rename,        // Rename imported conversation (append _imported_N)
    Merge,         // Merge messages into existing conversation
    Overwrite,     // Replace existing (dangerous)
}

impl ImportJob {
    async fn execute(&self) -> Result<ImportResult, ImportError> {
        let mut result = ImportResult::default();
        
        // 1. Import conversations
        let conversations = self.fetch_source_conversations().await?;
        for conv in conversations {
            match self.import_conversation(conv).await {
                Ok(id) => result.imported.push(id),
                Err(e) if matches!(e, ImportError::Conflict) => {
                    match self.params.conflict_strategy {
                        ConflictStrategy::Skip => result.skipped.push(conv.id),
                        ConflictStrategy::Rename => {
                            let renamed = self.rename_and_import(conv).await?;
                            result.renamed.push(renamed);
                        }
                        _ => result.errors.push((conv.id, e)),
                    }
                }
                Err(e) => result.errors.push((conv.id, e)),
            }
        }
        
        // 2. Import embeddings (if Chroma data available)
        if let Some(chroma_path) = self.params.chroma_data_path {
            self.import_chroma_data(chroma_path).await?;
        }
        
        // 3. Rebuild FTS index for imported messages
        self.rebuild_fts_index(&result.imported).await?;
        
        Ok(result)
    }
}
```

**CLI Usage**:
```bash
sekha import ~/backups/sekha-2026-01-01.db \
  --conflict=rename \
  --label-prefix="backup:" \
  --remap-folder="/work=/archive/old-work"
```

**Embedding costs**: If importing from another sekha.db, embeddings already exist. Just copy the Chroma collection. No re-embedding needed.

***

## **9. Pagination**

### **Current Problem**
```python
conversations = await memory.list_conversations(label="Work")
# Returns ALL Work conversations (could be 10,000)
```

### **World-Class Cursor-Based Pagination**

**Why cursor over offset**:
- Offset pagination breaks when data changes during iteration
- Cursor-based is stable and efficient

**Implementation**:

```rust
// src/models/api.rs
#[derive(Debug, Serialize, Deserialize)]
pub struct PaginatedRequest {
    pub cursor: Option<String>,  // Base64-encoded (timestamp, id)
    pub limit: u32,              // Max: 100, default: 20
    pub sort_by: SortField,
    pub sort_order: SortOrder,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct PaginatedResponse<T> {
    pub items: Vec<T>,
    pub next_cursor: Option<String>,
    pub has_more: bool,
    pub total_count: Option<u64>,  // Expensive, only if requested
}

#[derive(Debug, Serialize, Deserialize)]
struct Cursor {
    timestamp: chrono::NaiveDateTime,
    id: Uuid,
}

impl Cursor {
    fn encode(&self) -> String {
        let json = serde_json::to_string(self).unwrap();
        base64::encode(&json)
    }
    
    fn decode(s: &str) -> Result<Self, CursorError> {
        let json = base64::decode(s)?;
        Ok(serde_json::from_slice(&json)?)
    }
}

// API endpoint
pub async fn list_conversations(
    Query(params): Query<PaginatedRequest>,
) -> Result<Json<PaginatedResponse<Conversation>>, ApiError> {
    let cursor = params.cursor
        .map(|c| Cursor::decode(&c))
        .transpose()?;
    
    let mut query = conversations::Entity::find()
        .filter(conversations::Column::Status.eq("active"));
    
    // Apply cursor
    if let Some(cursor) = cursor {
        match params.sort_order {
            SortOrder::Asc => {
                query = query.filter(
                    conversations::Column::UpdatedAt.gt(cursor.timestamp)
                        .or(conversations::Column::UpdatedAt.eq(cursor.timestamp)
                            .and(conversations::Column::Id.gt(cursor.id)))
                );
            }
            SortOrder::Desc => {
                query = query.filter(
                    conversations::Column::UpdatedAt.lt(cursor.timestamp)
                        .or(conversations::Column::UpdatedAt.eq(cursor.timestamp)
                            .and(conversations::Column::Id.lt(cursor.id)))
                );
            }
        }
    }
    
    // Sort and limit
    let limit = params.limit.min(100) as u64;
    query = match params.sort_order {
        SortOrder::Asc => query.order_by_asc(conversations::Column::UpdatedAt),
        SortOrder::Desc => query.order_by_desc(conversations::Column::UpdatedAt),
    };
    
    let items = query.limit(limit + 1).all(db).await?;
    
    let has_more = items.len() > limit as usize;
    let items: Vec<_> = items.into_iter().take(limit as usize).collect();
    
    let next_cursor = if has_more {
        items.last().map(|last| Cursor {
            timestamp: last.updated_at,
            id: last.id,
        }.encode())
    } else {
        None
    };
    
    Ok(Json(PaginatedResponse {
        items: items.into_iter().map(Conversation::from).collect(),
        next_cursor,
        has_more,
        total_count: None,  // Could add optional count query
    }))
}
```

**SDK Usage**:
```python
# Python SDK
cursor = None
while True:
    page = await memory.list_conversations(label="Work", cursor=cursor, limit=50)
    
    for conv in page.items:
        print(conv.label)
    
    if not page.has_more:
        break
    
    cursor = page.next_cursor

# JavaScript SDK
const iterator = memory.listConversationsIterator({ label: "Work", limit: 50 });

for await (const conversation of iterator) {
    console.log(conversation.label);
}
```

**Benefits**:
- Stable iteration even if data changes
- Efficient (indexed on updated_at + id)
- Standard REST pattern
- Async iterator support

***

## **10. Background Importance Scoring - Analysis**

**Your concern**: "What if background jobs fail?"

**Valid points**:
- Adds complexity (job queue, worker processes)
- Failure scenarios need handling
- Additional monitoring required

**Trade-off Analysis**:

| Aspect | Synchronous (current) | Asynchronous (proposed) |
|--------|----------------------|------------------------|
| Store latency | 500-1000ms (LLM call) | <50ms (immediate return) |
| Failure handling | User sees error immediately | Silent failure possible |
| Complexity | Low | Medium (job queue) |
| User experience | Slower stores | Fast stores, eventual consistency |

**My recommendation**: **Start synchronous, add async later if needed**.

**Reasoning**:
- Single-user = low write volume
- 500ms store latency is acceptable for human interaction
- No job queue infrastructure to manage
- Simpler failure model

**However**, if you DO want async scoring:

### **World-Class Background Job Design**

```rust
// Use existing Redis as job queue (it's already running!)
use redis::aio::ConnectionManager;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
pub struct ImportanceScoreJob {
    pub message_id: Uuid,
    pub conversation_id: Uuid,
    pub content_preview: String,  // First 200 chars for debugging
    pub retry_count: u32,
    pub max_retries: u32,
}

pub struct JobQueue {
    redis: ConnectionManager,
    queue_name: String,
}

impl JobQueue {
    pub async fn enqueue(&self, job: ImportanceScoreJob) -> Result<(), JobError> {
        let job_json = serde_json::to_string(&job)?;
        
        // RPUSH to Redis list (FIFO queue)
        self.redis
            .rpush(&self.queue_name, job_json)
            .await?;
        
        // Set job metadata for monitoring
        let job_key = format!("job:{}:metadata", job.message_id);
        self.redis
            .hset_multiple(&job_key, &[
                ("status", "queued"),
                ("queued_at", &chrono::Utc::now().to_rfc3339()),
            ])
            .await?;
        
        Ok(())
    }
    
    pub async fn dequeue(&self) -> Result<Option<ImportanceScoreJob>, JobError> {
        // BLPOP with timeout (blocking pop, returns None if empty)
        let result: Option<(String, String)> = self.redis
            .blpop(&self.queue_name, 1.0)
            .await?;
        
        match result {
            Some((_, job_json)) => {
                let job = serde_json::from_str(&job_json)?;
                Ok(Some(job))
            }
            None => Ok(None),
        }
    }
}

// Worker process
pub async fn importance_scoring_worker(
    queue: Arc<JobQueue>,
    engine: Arc<ImportanceEngine>,
) -> Result<(), WorkerError> {
    info!("Starting importance scoring worker");
    
    loop {
        match queue.dequeue().await {
            Ok(Some(job)) => {
                let job_key = format!("job:{}:metadata", job.message_id);
                
                // Update status
                queue.redis.hset(&job_key, "status", "processing").await?;
                queue.redis.hset(&job_key, "started_at", &chrono::Utc::now().to_rfc3339()).await?;
                
                // Process job
                match engine.calculate_score(job.message_id).await {
                    Ok(score) => {
                        // Update message with score
                        engine.repo.update_importance_score(job.message_id, score).await?;
                        
                        // Mark job complete
                        queue.redis.hset(&job_key, "status", "completed").await?;
                        queue.redis.hset(&job_key, "completed_at", &chrono::Utc::now().to_rfc3339()).await?;
                        queue.redis.hset(&job_key, "score", &score.to_string()).await?;
                        
                        info!("Scored message {} with {}", job.message_id, score);
                    }
                    Err(e) => {
                        error!("Failed to score message {}: {}", job.message_id, e);
                        
                        // Retry logic
                        if job.retry_count < job.max_retries {
                            let retry_job = ImportanceScoreJob {
                                retry_count: job.retry_count + 1,
                                ..job
                            };
                            
                            // Exponential backoff
                            let delay_secs = 2u64.pow(retry_job.retry_count);
                            tokio::time::sleep(tokio::time::Duration::from_secs(delay_secs)).await;
                            
                            queue.enqueue(retry_job).await?;
                            queue.redis.hset(&job_key, "status", "retrying").await?;
                        } else {
                            // Max retries exceeded, mark failed
                            queue.redis.hset(&job_key, "status", "failed").await?;
                            queue.redis.hset(&job_key, "error", &e.to_string()).await?;
                            
                            // Fall back to heuristic score
                            let heuristic = engine.heuristic_score_only(job.message_id).await?;
                            engine.repo.update_importance_score(job.message_id, heuristic).await?;
                            
                            warn!("Fell back to heuristic score {} for message {}", heuristic, job.message_id);
                        }
                    }
                }
            }
            Ok(None) => {
                // Queue empty, wait
                tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
            }
            Err(e) => {
                error!("Queue error: {}", e);
                tokio::time::sleep(tokio::time::Duration::from_secs(5)).await;
            }
        }
    }
}

// Start worker on controller boot
#[tokio::main]
async fn main() {
    // ... existing setup ...
    
    // Spawn worker task
    let worker_queue = job_queue.clone();
    let worker_engine = importance_engine.clone();
    tokio::spawn(async move {
        if let Err(e) = importance_scoring_worker(worker_queue, worker_engine).await {
            error!("Worker crashed: {}", e);
        }
    });
    
    // ... start HTTP server ...
}
```

**Monitoring Endpoint**:
```rust
// GET /api/v1/admin/jobs/stats
pub async fn job_stats(State(redis): State<ConnectionManager>) -> Json<JobStats> {
    let queue_length: usize = redis.llen("importance_jobs").await.unwrap_or(0);
    
    let statuses = vec!["queued", "processing", "completed", "failed"];
    let mut counts = HashMap::new();
    
    for status in statuses {
        let pattern = format!("job:*:metadata");
        let keys: Vec<String> = redis.keys(&pattern).await.unwrap_or_default();
        
        let mut count = 0;
        for key in keys {
            let job_status: String = redis.hget(&key, "status").await.unwrap_or_default();
            if job_status == status {
                count += 1;
            }
        }
        counts.insert(status.to_string(), count);
    }
    
    Json(JobStats {
        queue_length,
        status_counts: counts,
    })
}
```

**Benefits**:
- Uses existing Redis (no new dependency)
- Automatic retries with exponential backoff
- Falls back to heuristic on failure
- Observable via `/admin/jobs/stats`
- Worker runs in same process (no separate deployment)

**When to use**: If user reports slow conversation stores, enable async mode via config flag.

***

## **11. Transaction Boundaries**

**The Issue**:
```rust
// Phase 1: Query Chroma (external service)
let semantic = repo.semantic_search(query, 200).await?;

// Phase 2: Query SQLite
let pinned = conversations::find().all(db).await?;

// Phase 3: More SQLite
let recent = messages::find().all(db).await?;

// If someone writes to SQLite between Phase 2 and 3, inconsistent results
```

**Solution**: Not traditional ACID transactions (can't include Chroma), but **snapshot consistency**.

```rust
use tokio::sync::RwLock;
use std::sync::Arc;

pub struct SnapshotContext {
    pub timestamp: chrono::NaiveDateTime,
    pub snapshot_id: Uuid,
}

pub struct ContextAssembler {
    repo: Arc<dyn ConversationRepository>,
    snapshot_lock: Arc<RwLock<()>>,  // Read lock for consistency
}

impl ContextAssembler {
    pub async fn assemble(
        &self,
        query: &str,
        preferred_labels: Vec<String>,
        context_budget: usize,
    ) -> Result<Vec<Message>, RepositoryError> {
        // Acquire read lock (allows concurrent reads, blocks writes)
        let _read_guard = self.snapshot_lock.read().await;
        
        // All queries happen under this lock
        let snapshot = SnapshotContext {
            timestamp: chrono::Utc::now().naive_utc(),
            snapshot_id: Uuid::new_v4(),
        };
        
        // Phase 1: Semantic search (Chroma)
        // Note: Chroma is eventually consistent anyway
        let candidates = self.recall_candidates(query, &preferred_labels, &snapshot).await?;
        
        // Phase 2-4: All SQLite queries use snapshot timestamp
        let ranked = self.rank_candidates(candidates, query, &preferred_labels).await?;
        let context = self.assemble_context(&mut ranked, context_budget, &snapshot).await?;
        let enhanced = self.enhance_context(context, &snapshot).await?;
        
        Ok(enhanced)
    }
    
    async fn recall_candidates(
        &self,
        query: &str,
        preferred_labels: &[String],
        snapshot: &SnapshotContext,
    ) -> Result<Vec<CandidateMessage>, RepositoryError> {
        // All queries filtered by snapshot timestamp
        let mut query_builder = conversations::Entity::find()
            .filter(conversations::Column::UpdatedAt.lte(snapshot.timestamp));
        
        // ... rest of query ...
    }
}

// Write operations acquire write lock
impl ConversationRepository for SqliteRepository {
    async fn create_conversation(&self, conv: Conversation) -> Result<Uuid, RepositoryError> {
        let _write_guard = self.snapshot_lock.write().await;
        
        // Write operations block all reads temporarily
        // This ensures consistent snapshots
        
        // ... insert conversation ...
        
        Ok(conv.id)
    }
}
```

**Alternative: Optimistic Concurrency**

```rust
// Add version to conversations table
ALTER TABLE conversations ADD COLUMN version INTEGER NOT NULL DEFAULT 1;

pub async fn update_conversation_with_version(
    &self,
    id: Uuid,
    expected_version: i32,
    updates: ConversationUpdate,
) -> Result<(), RepositoryError> {
    let result = conversations::Entity::update_many()
        .filter(conversations::Column::Id.eq(id))
        .filter(conversations::Column::Version.eq(expected_version))
        .col_expr(conversations::Column::Label, Expr::value(updates.label))
        .col_expr(conversations::Column::Version, Expr::value(expected_version + 1))
        .exec(db)
        .await?;
    
    if result.rows_affected == 0 {
        return Err(RepositoryError::VersionConflict {
            id,
            expected: expected_version,
        });
    }
    
    Ok(())
}
```

**Recommendation**: 
- Use **RwLock approach** for read-heavy workloads (context assembly)
- SQLite's WAL already provides write isolation
- Single-user deployment makes locking lightweight

***

## **12. Embedding ID Tracking**

**Problem**: When message deleted from SQLite, orphaned embedding in Chroma.

**Solution**: Track Chroma IDs and cascade deletes.

```rust
// Migration 009: Add embedding tracking
CREATE TABLE IF NOT EXISTS message_embeddings (
    message_id UUID PRIMARY KEY REFERENCES messages(id) ON DELETE CASCADE,
    chroma_id TEXT NOT NULL,
    chroma_collection TEXT NOT NULL,
    embedding_config_id UUID REFERENCES embedding_configs(id),
    created_at TIMESTAMP NOT NULL
);

CREATE INDEX idx_message_embeddings_chroma ON message_embeddings(chroma_id, chroma_collection);

// Repository implementation
impl MessageRepository for SqliteRepository {
    async fn delete_message(&self, id: Uuid) -> Result<(), RepositoryError> {
        // Start transaction
        let txn = self.db.begin().await?;
        
        // 1. Get embedding info BEFORE deleting message
        let embedding_info = message_embeddings::Entity::find()
            .filter(message_embeddings::Column::MessageId.eq(id))
            .one(&txn)
            .await?;
        
        // 2. Delete from SQLite (cascade deletes embedding record)
        messages::Entity::delete_by_id(id)
            .exec(&txn)
            .await?;
        
        // 3. Delete from Chroma (asynchronously, failure is non-critical)
        if let Some(info) = embedding_info {
            let chroma = self.chroma_client.clone();
            let chroma_id = info.chroma_id;
            let collection = info.chroma_collection;
            
            tokio::spawn(async move {
                if let Err(e) = chroma.delete(&collection, &chroma_id).await {
                    warn!("Failed to delete embedding {} from Chroma: {}", chroma_id, e);
                    // Non-fatal: Chroma orphans will be cleaned by periodic job
                }
            });
        }
        
        txn.commit().await?;
        Ok(())
    }
}

// Periodic cleanup job (runs daily)
pub async fn cleanup_orphaned_embeddings(
    db: &DatabaseConnection,
    chroma: &ChromaClient,
) -> Result<CleanupStats, CleanupError> {
    let mut stats = CleanupStats::default();
    
    // Get all Chroma IDs
    let chroma_ids: HashSet<String> = chroma.list_all_ids("sekha_messages").await?;
    
    // Get all tracked embedding IDs
    let tracked_ids: HashSet<String> = message_embeddings::Entity::find()
        .all(db)
        .await?
        .into_iter()
        .map(|e| e.chroma_id)
        .collect();
    
    // Find orphans
    let orphans: Vec<String> = chroma_ids.difference(&tracked_ids).cloned().collect();
    
    info!("Found {} orphaned embeddings", orphans.len());
    
    // Delete in batches
    for chunk in orphans.chunks(100) {
        chroma.delete_batch("sekha_messages", chunk).await?;
        stats.deleted += chunk.len();
    }
    
    Ok(stats)
}
```

**Chroma Client Enhancement**:
```rust
impl ChromaClient {
    pub async fn add_with_tracking(
        &self,
        collection: &str,
        embedding: Vec<f32>,
        metadata: HashMap<String, Value>,
        db: &DatabaseConnection,
        message_id: Uuid,
        config_id: Uuid,
    ) -> Result<String, ChromaError> {
        // 1. Add to Chroma
        let chroma_id = Uuid::new_v4().to_string();
        self.add(collection, &chroma_id, embedding, metadata).await?;
        
        // 2. Track in SQLite
        let tracking = message_embeddings::ActiveModel {
            message_id: Set(message_id),
            chroma_id: Set(chroma_id.clone()),
            chroma_collection: Set(collection.to_string()),
            embedding_config_id: Set(config_id),
            created_at: Set(chrono::Utc::now().naive_utc()),
        };
        
        tracking.insert(db).await?;
        
        Ok(chroma_id)
    }
}
```

**Benefits**:
- Automatic cascade delete
- Orphan detection and cleanup
- Audit trail of embeddings
- Graceful handling of Chroma failures

***

## **13. Schema Versioning**

```rust
// Migration 000 (always first)
CREATE TABLE IF NOT EXISTS schema_version (
    version INTEGER PRIMARY KEY,
    applied_at TIMESTAMP NOT NULL,
    migration_file TEXT NOT NULL,
    checksum TEXT NOT NULL
);

// Migration runner
use sha2::{Sha256, Digest};

pub struct MigrationRunner {
    db: DatabaseConnection,
    migrations_path: PathBuf,
}

#[derive(Debug)]
pub struct Migration {
    pub version: i32,
    pub name: String,
    pub up_sql: String,
    pub down_sql: String,
    pub checksum: String,
}

impl MigrationRunner {
    pub async fn run_pending_migrations(&self) -> Result<Vec<i32>, MigrationError> {
        // 1. Get current version
        let current_version = self.get_current_version().await?;
        
        // 2. Load all migrations
        let all_migrations = self.load_migrations()?;
        
        // 3. Verify existing migrations haven't changed
        self.verify_applied_migrations(&all_migrations).await?;
        
        // 4. Apply pending migrations
        let mut applied = Vec::new();
        for migration in all_migrations {
            if migration.version > current_version {
                self.apply_migration(&migration).await?;
                applied.push(migration.version);
                info!("Applied migration {}: {}", migration.version, migration.name);
            }
        }
        
        Ok(applied)
    }
    
    async fn get_current_version(&self) -> Result<i32, MigrationError> {
        let result = schema_version::Entity::find()
            .order_by_desc(schema_version::Column::Version)
            .one(&self.db)
            .await?;
        
        Ok(result.map(|r| r.version).unwrap_or(0))
    }
    
    async fn verify_applied_migrations(
        &self,
        migrations: &[Migration],
    ) -> Result<(), MigrationError> {
        let applied = schema_version::Entity::find()
            .all(&self.db)
            .await?;
        
        for applied_migration in applied {
            let migration = migrations
                .iter()
                .find(|m| m.version == applied_migration.version)
                .ok_or_else(|| MigrationError::MissingMigration(applied_migration.version))?;
            
            if migration.checksum != applied_migration.checksum {
                return Err(MigrationError::ChecksumMismatch {
                    version: migration.version,
                    expected: applied_migration.checksum,
                    got: migration.checksum.clone(),
                });
            }
        }
        
        Ok(())
    }
    
    async fn apply_migration(&self, migration: &Migration) -> Result<(), MigrationError> {
        let txn = self.db.begin().await?;
        
        // Execute migration SQL
        for statement in migration.up_sql.split(";").filter(|s| !s.trim().is_empty()) {
            txn.execute(Statement::from_string(
                DatabaseBackend::Sqlite,
                statement.to_string(),
            ))
            .await?;
        }
        
        // Record migration
        let record = schema_version::ActiveModel {
            version: Set(migration.version),
            applied_at: Set(chrono::Utc::now().naive_utc()),
            migration_file: Set(migration.name.clone()),
            checksum: Set(migration.checksum.clone()),
        };
        
        record.insert(&txn).await?;
        
        txn.commit().await?;
        Ok(())
    }
    
    fn load_migrations(&self) -> Result<Vec<Migration>, MigrationError> {
        let mut migrations = Vec::new();
        
        for entry in std::fs::read_dir(&self.migrations_path)? {
            let entry = entry?;
            let path = entry.path();
            
            if path.extension().map_or(false, |ext| ext == "sql") {
                let filename = path.file_stem().unwrap().to_string_lossy();
                
                // Parse "001_create_conversations.sql"
                let parts: Vec<&str> = filename.splitn(2, '_').collect();
                let version: i32 = parts.parse()?;
                let name = parts.to_string(); [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/collection_46e813f6-88e3-4b2f-8753-db87de0f7859/9d4a066c-555c-45e5-bce4-854d17f49a9b/Repo-Lint-Unit-Integration-Coverage-CI.csv)
                
                let content = std::fs::read_to_string(&path)?;
                
                // Split on -- DOWN marker
                let (up_sql, down_sql) = if content.contains("-- DOWN") {
                    let parts: Vec<&str> = content.splitn(2, "-- DOWN").collect();
                    (parts.to_string(), parts.to_string()) [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/collection_46e813f6-88e3-4b2f-8753-db87de0f7859/9d4a066c-555c-45e5-bce4-854d17f49a9b/Repo-Lint-Unit-Integration-Coverage-CI.csv)
                } else {
                    (content.clone(), String::new())
                };
                
                // Calculate checksum
                let mut hasher = Sha256::new();
                hasher.update(up_sql.as_bytes());
                let checksum = format!("{:x}", hasher.finalize());
                
                migrations.push(Migration {
                    version,
                    name,
                    up_sql,
                    down_sql,
                    checksum,
                });
            }
        }
        
        migrations.sort_by_key(|m| m.version);
        Ok(migrations)
    }
    
    pub async fn rollback_to_version(&self, target: i32) -> Result<(), MigrationError> {
        let current_version = self.get_current_version().await?;
        
        if target >= current_version {
            return Err(MigrationError::InvalidRollback { target, current: current_version });
        }
        
        let migrations = self.load_migrations()?;
        
        // Apply DOWN migrations in reverse order
        for migration in migrations.iter().rev() {
            if migration.version > target && migration.version <= current_version {
                self.rollback_migration(migration).await?;
                info!("Rolled back migration {}: {}", migration.version, migration.name);
            }
        }
        
        Ok(())
    }
    
    async fn rollback_migration(&self, migration: &Migration) -> Result<(), MigrationError> {
        if migration.down_sql.is_empty() {
            return Err(MigrationError::NoRollbackSQL(migration.version));
        }
        
        let txn = self.db.begin().await?;
        
        for statement in migration.down_sql.split(";").filter(|s| !s.trim().is_empty()) {
            txn.execute(Statement::from_string(
                DatabaseBackend::Sqlite,
                statement.to_string(),
            ))
            .await?;
        }
        
        schema_version::Entity::delete_by_id(migration.version)
            .exec(&txn)
            .await?;
        
        txn.commit().await?;
        Ok(())
    }
}

// Controller startup
#[tokio::main]
async fn main() {
    let db = Database::connect(config.database_url).await.unwrap();
    
    // Run migrations on startup
    let runner = MigrationRunner::new(db.clone(), Path::new("./migrations"));
    
    match runner.run_pending_migrations().await {
        Ok(applied) if !applied.is_empty() => {
            info!("Applied {} migrations: {:?}", applied.len(), applied);
        }
        Ok(_) => {
            info!("Database schema up to date");
        }
        Err(e) => {
            error!("Migration failed: {}", e);
            std::process::exit(1);
        }
    }
    
    // ... start server ...
}
```

**Migration File Format**:
```sql
-- 008_add_embedding_tracking.sql

-- UP
CREATE TABLE IF NOT EXISTS message_embeddings (
    message_id UUID PRIMARY KEY REFERENCES messages(id) ON DELETE CASCADE,
    chroma_id TEXT NOT NULL,
    chroma_collection TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL
);

CREATE INDEX idx_message_embeddings_chroma 
ON message_embeddings(chroma_id, chroma_collection);

-- DOWN
DROP INDEX IF EXISTS idx_message_embeddings_chroma;
DROP TABLE IF EXISTS message_embeddings;
```

**CLI Commands**:
```bash
sekha migrate status                    # Show current version and pending
sekha migrate up                        # Apply all pending
sekha migrate up --to 10               # Apply up to version 10
sekha migrate down --to 5              # Rollback to version 5
sekha migrate create "add_user_prefs"  # Create new migration file
```

***

## **14. API Key Validation**

Planning OAuth future. Here's a design that transitions smoothly:

```rust
// src/auth/mod.rs
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum AuthMethod {
    ApiKey(ApiKeyAuth),
    OAuth(OAuthAuth),  // Future
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ApiKeyAuth {
    pub key: String,
    pub key_type: ApiKeyType,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ApiKeyType {
    Test,       // sk-test-*
    Production, // sk-sekha-*
}

impl ApiKeyAuth {
    pub fn validate(&self) -> Result<(), AuthError> {
        match self.key_type {
            ApiKeyType::Test => {
                if !self.key.starts_with("sk-test-") {
                    return Err(AuthError::InvalidKeyFormat("Test key must start with sk-test-"));
                }
                if self.key.len() < 20 {
                    return Err(AuthError::InvalidKeyFormat("Test key must be at least 20 characters"));
                }
            }
            ApiKeyType::Production => {
                if !self.key.starts_with("sk-sekha-") {
                    return Err(AuthError::InvalidKeyFormat("Production key must start with sk-sekha-"));
                }
                if self.key.len() < 40 {
                    return Err(AuthError::InvalidKeyFormat("Production key must be at least 40 characters"));
                }
            }
        }
        
        // Additional validation: character set
        if !self.key.chars().all(|c| c.is_ascii_alphanumeric() || c == '-' || c == '_') {
            return Err(AuthError::InvalidKeyFormat("Key contains invalid characters"));
        }
        
        Ok(())
    }
    
    pub fn parse(key: &str) -> Result<Self, AuthError> {
        let key_type = if key.starts_with("sk-test-") {
            ApiKeyType::Test
        } else if key.starts_with("sk-sekha-") {
            ApiKeyType::Production
        } else {
            return Err(AuthError::InvalidKeyFormat("Key must start with sk-test- or sk-sekha-"));
        };
        
        let auth = ApiKeyAuth {
            key: key.to_string(),
            key_type,
        };
        
        auth.validate()?;
        Ok(auth)
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OAuthAuth {
    pub access_token: String,
    pub token_type: String,
    pub expires_at: chrono::DateTime<chrono::Utc>,
    pub scopes: Vec<String>,
}

// Future OAuth implementation
impl OAuthAuth {
    pub async fn validate(&self) -> Result<(), AuthError> {
        // Validate with OAuth provider
        todo!("OAuth validation")
    }
}
```

**Middleware**:
```rust
use axum::{
    extract::Request,
    http::StatusCode,
    middleware::Next,
    response::Response,
};

pub async fn auth_middleware(
    req: Request,
    next: Next,
) -> Result<Response, (StatusCode, String)> {
    let auth_header = req.headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .ok_or_else(|| (StatusCode::UNAUTHORIZED, "Missing Authorization header".to_string()))?;
    
    let auth_method = if auth_header.starts_with("Bearer sk-") {
        // API Key
        let key = auth_header.strip_prefix("Bearer ").unwrap();
        let api_key = ApiKeyAuth::parse(key)
            .map_err(|e| (StatusCode::UNAUTHORIZED, e.to_string()))?;
        
        AuthMethod::ApiKey(api_key)
    } else if auth_header.starts_with("Bearer ey") {
        // JWT (OAuth)
        let token = auth_header.strip_prefix("Bearer ").unwrap();
        let oauth = OAuthAuth::from_jwt(token).await
            .map_err(|e| (StatusCode::UNAUTHORIZED, e.to_string()))?;
        
        oauth.validate().await
            .map_err(|e| (StatusCode::UNAUTHORIZED, e.to_string()))?;
        
        AuthMethod::OAuth(oauth)
    } else {
        return Err((StatusCode::UNAUTHORIZED, "Invalid Authorization format".to_string()));
    };
    
    // Attach auth context to request
    let mut req = req;
    req.extensions_mut().insert(auth_method);
    
    Ok(next.run(req).await)
}
```

**SDK Implementations**:

**Python**:
```python
# sekha/auth.py
from typing import Literal
import re

class ApiKeyAuth:
    def __init__(self, key: str):
        self.key = key
        self.key_type = self._detect_type()
        self._validate()
    
    def _detect_type(self) -> Literal["test", "production"]:
        if self.key.startswith("sk-test-"):
            return "test"
        elif self.key.startswith("sk-sekha-"):
            return "production"
        else:
            raise ValueError("API key must start with sk-test- or sk-sekha-")
    
    def _validate(self):
        if self.key_type == "test":
            if len(self.key) < 20:
                raise ValueError("Test key must be at least 20 characters")
        else:
            if len(self.key) < 40:
                raise ValueError("Production key must be at least 40 characters")
        
        # Character validation
        if not re.match(r'^[a-zA-Z0-9_-]+$', self.key):
            raise ValueError("API key contains invalid characters")
```

**JavaScript**:
```typescript
// auth.ts
export class ApiKeyAuth {
  constructor(private key: string) {
    this.validate();
  }
  
  private validate(): void {
    const keyType = this.detectType();
    
    if (keyType === 'test' && this.key.length < 20) {
      throw new Error('Test key must be at least 20 characters');
    }
    
    if (keyType === 'production' && this.key.length < 40) {
      throw new Error('Production key must be at least 40 characters');
    }
    
    if (!/^[a-zA-Z0-9_-]+$/.test(this.key)) {
      throw new Error('API key contains invalid characters');
    }
  }
  
  private detectType(): 'test' | 'production' {
    if (this.key.startsWith('sk-test-')) return 'test';
    if (this.key.startswith('sk-sekha-')) return 'production';
    throw new Error('API key must start with sk-test- or sk-sekha-');
  }
}
```

**Global Specification Document** (`docs/auth-spec.md`):
```markdown
# Sekha Authentication Specification v1.0

## API Key Format

### Test Keys
- **Prefix**: `sk-test-`
- **Minimum Length**: 20 characters
- **Character Set**: `[a-zA-Z0-9_-]`
- **Example**: `sk-test-1234567890abcdef`
- **Usage**: Development and testing only

### Production Keys
- **Prefix**: `sk-sekha-`
- **Minimum Length**: 40 characters
- **Character Set**: `[a-zA-Z0-9_-]`
- **Example**: `sk-sekha-1234567890abcdef1234567890abcdef`
- **Usage**: Production deployments

## HTTP Header Format
```
Authorization: Bearer <api_key>
```

## OAuth 2.0 (Future)
- **Token Type**: JWT
- **Header Format**: `Authorization: Bearer <jwt_token>`
- **Supported Flows**: Authorization Code, Client Credentials
- **Scopes**: TBD

## Error Responses
- `401 Unauthorized`: Missing or invalid credentials
- `403 Forbidden`: Valid credentials but insufficient permissions

## SDK Implementation Requirements
All SDKs MUST:
1. Validate key format on construction
2. Reject keys with invalid prefix
3. Reject keys below minimum length
4. Validate character set
5. Provide clear error messages
```

**Enforcement**:
```bash
# CI test that runs against all SDKs
./tests/auth-compliance-test.sh

# Tests:
# - Valid test key accepted
# - Valid production key accepted
# - Short keys rejected
# - Invalid prefix rejected
# - Invalid characters rejected
# - Error messages match spec
```

***

## **15. API Key Scoping - Reconsidered**

You're right - single-user deployment means compromised key only affects that user's local DB, not a global system.

**However**, there are still valid scenarios for scoping:

**Scenario 1**: Developer shares read-only access with teammate
**Scenario 2**: CI/CD pipeline needs write-only access to store build logs
**Scenario 3**: Analytics dashboard needs read-only metrics access

### **Optional Scoped Keys Design**

```rust
// migrations/009_api_key_scopes.sql
CREATE TABLE IF NOT EXISTS api_keys (
    id UUID PRIMARY KEY,
    key_hash TEXT NOT NULL UNIQUE,
    key_prefix TEXT NOT NULL,  -- First 8 chars for identification
    permissions TEXT NOT NULL,  -- JSON array of permissions
    created_at TIMESTAMP NOT NULL,
    expires_at TIMESTAMP,
    last_used_at TIMESTAMP,
    is_active BOOLEAN NOT NULL DEFAULT true
);

CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
CREATE INDEX idx_api_keys_active ON api_keys(is_active) WHERE is_active = true;

// Permission system
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ApiKey {
    pub id: Uuid,
    pub key_hash: String,
    pub key_prefix: String,
    pub permissions: Vec<Permission>,
    pub expires_at: Option<chrono::DateTime<chrono::Utc>>,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum Permission {
    // Conversation permissions
    ConversationRead,
    ConversationWrite,
    ConversationDelete,
    
    // Search permissions
    SearchQuery,
    
    // Export permissions
    Export,
    
    // Admin permissions
    AdminStats,
    AdminConfig,
    
    // Wildcard
    All,
}

impl ApiKey {
    pub fn has_permission(&self, required: &Permission) -> bool {
        self.permissions.contains(&Permission::All) || 
        self.permissions.contains(required)
    }
    
    pub fn is_expired(&self) -> bool {
        self.expires_at.map_or(false, |exp| exp < chrono::Utc::now())
    }
}

// Middleware check
pub async fn check_permission(
    Extension(api_key): Extension<ApiKey>,
    required: Permission,
) -> Result<(), (StatusCode, String)> {
    if !api_key.is_active {
        return Err((StatusCode::FORBIDDEN, "API key is revoked".to_string()));
    }
    
    if api_key.is_expired() {
        return Err((StatusCode::FORBIDDEN, "API key has expired".to_string()));
    }
    
    if !api_key.has_permission(&required) {
        return Err((StatusCode::FORBIDDEN, format!("Missing permission: {:?}", required)));
    }
    
    Ok(())
}

// Route protection
pub async fn create_conversation(
    Extension(api_key): Extension<ApiKey>,
    Json(conversation): Json<NewConversation>,
) -> Result<Json<Conversation>, ApiError> {
    check_permission(Extension(api_key.clone()), Permission::ConversationWrite).await?;
    
    // ... rest of handler ...
}
```

**CLI for Key Management**:
```bash
# Create read-only key
sekha keys create \
  --name "Teammate Read Access" \
  --permissions conversation:read,search:query \
  --expires-in 30d

# Output:
# Created key: sk-sekha-readonly-a1b2c3...
# Permissions: conversation:read, search:query
# Expires: 2026-02-23

# List keys
sekha keys list

# Revoke key
sekha keys revoke sk-sekha-readonly-a1b2c3...
```

**Default Behavior**: If no scoped keys exist, master key has all permissions (backward compatible).

***

## **16. Folder Path Sanitization**

```rust
// src/validation/folder.rs
use std::path::{Path, PathBuf};
use regex::Regex;

pub struct FolderValidator {
    max_depth: usize,
    max_length: usize,
    allowed_chars: Regex,
}

impl Default for FolderValidator {
    fn default() -> Self {
        Self {
            max_depth: 10,
            max_length: 255,
            allowed_chars: Regex::new(r"^[a-zA-Z0-9_\-/]+$").unwrap(),
        }
    }
}

impl FolderValidator {
    pub fn validate(&self, folder: &str) -> Result<String, ValidationError> {
        // 1. Must start with /
        if !folder.starts_with('/') {
            return Err(ValidationError::InvalidFolder("Folder must start with /"));
        }
        
        // 2. No trailing slash (except root)
        let folder = if folder != "/" && folder.ends_with('/') {
            folder.trim_end_matches('/')
        } else {
            folder
        };
        
        // 3. Length check
        if folder.len() > self.max_length {
            return Err(ValidationError::InvalidFolder("Folder path too long"));
        }
        
        // 4. Character validation
        if !self.allowed_chars.is_match(folder) {
            return Err(ValidationError::InvalidFolder("Folder contains invalid characters"));
        }
        
        // 5. No path traversal
        if folder.contains("..") {
            return Err(ValidationError::InvalidFolder("Path traversal not allowed"));
        }
        
        if folder.contains("//") {
            return Err(ValidationError::InvalidFolder("Double slashes not allowed"));
        }
        
        // 6. No hidden folders
        let parts: Vec<&str> = folder.split('/').collect();
        for part in parts.iter().skip(1) {  // Skip first empty part
            if part.starts_with('.') {
                return Err(ValidationError::InvalidFolder("Hidden folders not allowed"));
            }
        }
        
        // 7. Depth check
        let depth = parts.len() - 1;  // -1 for leading /
        if depth > self.max_depth {
            return Err(ValidationError::InvalidFolder(
                format!("Folder depth {} exceeds maximum {}", depth, self.max_depth)
            ));
        }
        
        // 8. No reserved names
        let reserved = ["con", "prn", "aux", "nul", "com1", "lpt1"];  // Windows reserved
        for part in parts.iter().skip(1) {
            if reserved.contains(&part.to_lowercase().as_str()) {
                return Err(ValidationError::InvalidFolder("Reserved folder name"));
            }
        }
        
        Ok(folder.to_string())
    }
    
    pub fn normalize(&self, folder: &str) -> Result<String, ValidationError> {
        // 1. Trim whitespace
        let folder = folder.trim();
        
        // 2. Ensure leading slash
        let folder = if !folder.starts_with('/') {
            format!("/{}", folder)
        } else {
            folder.to_string()
        };
        
        // 3. Remove trailing slash (except root)
        let folder = if folder != "/" && folder.ends_with('/') {
            folder.trim_end_matches('/').to_string()
        } else {
            folder
        };
        
        // 4. Collapse multiple slashes
        let re = Regex::new(r"/+").unwrap();
        let folder = re.replace_all(&folder, "/").to_string();
        
        // 5. Validate normalized path
        self.validate(&folder)?;
        
        Ok(folder)
    }
}

// Integration
impl ConversationRepository for SqliteRepository {
    async fn create_conversation(&self, conv: NewConversation) -> Result<Conversation, RepositoryError> {
        // Normalize and validate folder
        let folder = self.folder_validator
            .normalize(&conv.folder)
            .map_err(|e| RepositoryError::ValidationError(e.to_string()))?;
        
        let conversation = Conversation {
            folder,  // Use normalized folder
            ..conv.into()
        };
        
        // ... rest of creation ...
    }
}
```

**Test Cases**:
```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_valid_folders() {
        let validator = FolderValidator::default();
        
        assert!(validator.validate("/work").is_ok());
        assert!(validator.validate("/projects/ai").is_ok());
        assert!(validator.validate("/deep/folder/structure/is/ok").is_ok());
    }
    
    #[test]
    fn test_path_traversal() {
        let validator = FolderValidator::default();
        
        assert!(validator.validate("/../etc/passwd").is_err());
        assert!(validator.validate("/work/../../../etc").is_err());
        assert!(validator.validate("/work//admin").is_err());
    }
    
    #[test]
    fn test_hidden_folders() {
        let validator = FolderValidator::default();
        
        assert!(validator.validate("/.hidden").is_err());
        assert!(validator.validate("/work/.git").is_err());
    }
    
    #[test]
    fn test_normalization() {
        let validator = FolderValidator::default();
        
        assert_eq!(validator.normalize("  /work/  ").unwrap(), "/work");
        assert_eq!(validator.normalize("work/ai").unwrap(), "/work/ai");
        assert_eq!(validator.normalize("/work//ai///").unwrap(), "/work/ai");
    }
}
```

***

## **17. N+1 Query in Context Enhancement - Detailed Analysis**

**Current Code**:
```rust
async fn enhance_context(&self, mut context: Vec<Message>) -> Result<Vec<Message>, RepositoryError> {
    for message in &mut context {
        // PROBLEM: One SQL query per message
        if let Some(conversation) = self.repo.find_by_id(message.conversation_id).await? {
            meta["citation"] = serde_json::json!({
                "label": conversation.label,
                "folder": conversation.folder,
            });
        }
    }
}
```

**If context has 50 messages**:
- 50 SQL queries: `SELECT * FROM conversations WHERE id = ?`
- If messages from 10 unique conversations: 40 redundant queries
- Each query: ~1ms = 50ms total latency (noticeable)

### **World-Class Solution: Batch Fetching**

```rust
async fn enhance_context(&self, mut context: Vec<Message>) -> Result<Vec<Message>, RepositoryError> {
    // 1. Collect unique conversation IDs
    let conv_ids: HashSet<Uuid> = context
        .iter()
        .map(|m| m.conversation_id)
        .collect();
    
    // 2. Batch fetch conversations (ONE query)
    let conversations = self.repo
        .find_by_ids(conv_ids.into_iter().collect())
        .await?;
    
    // 3. Build lookup map
    let conv_map: HashMap<Uuid, Conversation> = conversations
        .into_iter()
        .map(|c| (c.id, c))
        .collect();
    
    // 4. Enhance messages with lookups (in-memory, fast)
    for message in &mut context {
        if let Some(conversation) = conv_map.get(&message.conversation_id) {
            let mut meta = message.metadata.clone().unwrap_or_else(|| serde_json::json!({}));
            
            meta["citation"] = serde_json::json!({
                "label": conversation.label,
                "folder": conversation.folder,
                "timestamp": message.timestamp.to_string(),
            });
            
            message.metadata = Some(meta);
        }
    }
    
    Ok(context)
}

// New repository method
impl ConversationRepository for SqliteRepository {
    async fn find_by_ids(&self, ids: Vec<Uuid>) -> Result<Vec<Conversation>, RepositoryError> {
        use sea_orm::sea_query::{Expr, SimpleExpr};
        
        if ids.is_empty() {
            return Ok(Vec::new());
        }
        
        // SeaORM IN query
        let models = conversations::Entity::find()
            .filter(conversations::Column::Id.is_in(ids))
            .all(self.db)
            .await?;
        
        Ok(models.into_iter().map(Conversation::from).collect())
    }
}
```

**Performance Improvement**:
- Before: 50 queries, 50ms latency
- After: 1 query, 1ms latency
- **50x faster**

**SQL Generated**:
```sql
-- Before (50 queries)
SELECT * FROM conversations WHERE id = 'uuid1';
SELECT * FROM conversations WHERE id = 'uuid2';
-- ... 48 more ...

-- After (1 query)
SELECT * FROM conversations 
WHERE id IN ('uuid1', 'uuid2', ..., 'uuid50');
```

***

## **18. Pruning Full Table Scan**

### **Smart Caching + Incremental Scan**

```rust
// Cache pruning candidates to avoid repeated scans
pub struct PruningCache {
    redis: ConnectionManager,
    cache_ttl: u64,  // 1 day
}

impl PruningCache {
    pub async fn get_candidates(&self, threshold_days: i64) -> Result<Option<Vec<PruningSuggestion>>, RedisError> {
        let key = format!("pruning:candidates:{}", threshold_days);
        
        let cached: Option<String> = self.redis.get(&key).await?;
        
        match cached {
            Some(json) => Ok(Some(serde_json::from_str(&json)?)),
            None => Ok(None),
        }
    }
    
    pub async fn set_candidates(&self, threshold_days: i64, suggestions: &[PruningSuggestion]) -> Result<(), RedisError> {
        let key = format!("pruning:candidates:{}", threshold_days);
        let json = serde_json::to_string(suggestions)?;
        
        self.redis.setex(&key, self.cache_ttl, json).await?;
        Ok(())
    }
}

// Optimized pruning with cache
pub async fn generate_suggestions_cached(
    &self,
    threshold_days: i64,
    importance_threshold: f32,
) -> Result<Vec<PruningSuggestion>, RepositoryError> {
    // Check cache first
    if let Some(cached) = self.cache.get_candidates(threshold_days).await? {
        return Ok(cached);
    }
    
    // Cache miss: perform full scan
    let suggestions = self.generate_suggestions(threshold_days, importance_threshold).await?;
    
    // Cache for next time
    self.cache.set_candidates(threshold_days, &suggestions).await?;
    
    Ok(suggestions)
}

// Index optimization
CREATE INDEX idx_conversations_pruning 
ON conversations(updated_at, status, importance_score) 
WHERE status = 'active';

// Now query is fast even on large tables
SELECT * FROM conversations 
WHERE updated_at < ? 
  AND status = 'active' 
  AND importance_score < 5.0;
-- Uses idx_conversations_pruning
```

**Additional Optimization**: Incremental pruning

```rust
// Track last pruning run
CREATE TABLE pruning_runs (
    id UUID PRIMARY KEY,
    started_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP,
    conversations_scanned INTEGER,
    suggestions_generated INTEGER
);

// Only scan conversations updated since last run
pub async fn incremental_prune(&self) -> Result<Vec<PruningSuggestion>, RepositoryError> {
    let last_run = self.get_last_pruning_run().await?;
    
    let cutoff = chrono::Utc::now().naive_utc() - Duration::days(90);
    
    let candidates = conversations::Entity::find()
        .filter(conversations::Column::UpdatedAt.lt(cutoff))
        .filter(conversations::Column::UpdatedAt.gt(last_run.completed_at))  // NEW
        .filter(conversations::Column::Status.eq("active"))
        .all(db)
        .await?;
    
    // ... generate suggestions ...
}
```

**Result**: Pruning stays fast even with years of data.

***

## **19. Redis Usage**

**When to use Redis** (implementation priorities):
1. **Distributed rate limiting** (if multi-instance deployment)
2. **Session storage** (if OAuth/web UI added)
3. **Job queue** (if background jobs implemented per #10)
4. **Query result caching** (expensive semantic searches)

For now, single-instance deployment doesn't need it.

***

## **21. Search Faceting/Filtering**

### **Enhanced Query Filter System**

```rust
// src/models/api.rs
#[derive(Debug, Serialize, Deserialize)]
pub struct SmartQueryRequest {
    pub query: String,
    pub limit: Option<u32>,
    pub filters: QueryFilters,
}

#[derive(Debug, Serialize, Deserialize, Default)]
pub struct QueryFilters {
    // Label filtering with AND/OR logic
    pub labels: Option<LabelFilter>,
    
    // Date range filtering
    pub date_range: Option<DateRange>,
    
    // Importance score filtering
    pub importance: Option<ImportanceFilter>,
    
    // Folder filtering (supports hierarchical)
    pub folders: Option<FolderFilter>,
    
    // Message role filtering
    pub roles: Option<Vec<MessageRole>>,
    
    // Conversation status
    pub status: Option<Vec<ConversationStatus>>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct LabelFilter {
    pub mode: LabelFilterMode,
    pub labels: Vec<String>,
}

#[derive(Debug, Serialize, Deserialize)]
pub enum LabelFilterMode {
    Any,      // OR: matches any label
    All,      // AND: matches all labels
    Exact,    // Exact match only
    None,     // Excludes these labels
}

#[derive(Debug, Serialize, Deserialize)]
pub struct DateRange {
    pub start: Option<chrono::NaiveDateTime>,
    pub end: Option<chrono::NaiveDateTime>,
    pub preset: Option<DatePreset>,
}

#[derive(Debug, Serialize, Deserialize)]
pub enum DatePreset {
    Today,
    Yesterday,
    LastWeek,
    LastMonth,
    LastYear,
    Custom,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct ImportanceFilter {
    pub min: Option<f32>,
    pub max: Option<f32>,
    pub tier: Option<ImportanceTier>,  // High (8-10), Medium (5-7), Low (0-4)
}

#[derive(Debug, Serialize, Deserialize)]
pub enum ImportanceTier {
    Critical,  // 9-10
    High,      // 7-8
    Medium,    // 4-6
    Low,       // 0-3
}

#[derive(Debug, Serialize, Deserialize)]
pub struct FolderFilter {
    pub paths: Vec<String>,
    pub recursive: bool,  // Include subfolders
    pub mode: FolderFilterMode,
}

#[derive(Debug, Serialize, Deserialize)]
pub enum FolderFilterMode {
    Include,
    Exclude,
}

// Implementation
impl ContextAssembler {
    pub async fn smart_query_filtered(
        &self,
        request: SmartQueryRequest,
    ) -> Result<Vec<Message>, RepositoryError> {
        // Phase 1: Semantic search (Chroma)
        let query_embedding = self.llm_bridge.embed_text(&request.query).await?;
        let semantic_candidates = self.chroma.search(query_embedding, 500).await?;
        
        // Phase 2: Build SQL filter
        let mut sql_query = messages::Entity::find()
            .filter(messages::Column::Id.is_in(
                semantic_candidates.iter().map(|c| c.id).collect::<Vec<_>>()
            ));
        
        // Apply date range filter
        if let Some(date_range) = request.filters.date_range {
            if let Some(start) = date_range.start {
                sql_query = sql_query.filter(messages::Column::Timestamp.gte(start));
            }
            if let Some(end) = date_range.end {
                sql_query = sql_query.filter(messages::Column::Timestamp.lte(end));
            }
        }
        
        // Apply importance filter
        if let Some(importance) = request.filters.importance {
            if let Some(min) = importance.min {
                sql_query = sql_query.filter(messages::Column::ImportanceScore.gte(min));
            }
            if let Some(max) = importance.max {
                sql_query = sql_query.filter(messages::Column::ImportanceScore.lte(max));
            }
            if let Some(tier) = importance.tier {
                let (min, max) = match tier {
                    ImportanceTier::Critical => (9.0, 10.0),
                    ImportanceTier::High => (7.0, 8.99),
                    ImportanceTier::Medium => (4.0, 6.99),
                    ImportanceTier::Low => (0.0, 3.99),
                };
                sql_query = sql_query
                    .filter(messages::Column::ImportanceScore.gte(min))
                    .filter(messages::Column::ImportanceScore.lte(max));
            }
        }
        
        // Apply role filter
        if let Some(roles) = request.filters.roles {
            let role_strs: Vec<String> = roles.iter().map(|r| format!("{:?}", r)).collect();
            sql_query = sql_query.filter(messages::Column::Role.is_in(role_strs));
        }
        
        // Fetch filtered messages
        let mut messages = sql_query.all(self.db).await?;
        
        // Phase 3: Apply label filter (requires join with conversations)
        if let Some(label_filter) = request.filters.labels {
            messages = self.apply_label_filter(messages, label_filter).await?;
        }
        
        // Phase 4: Apply folder filter
        if let Some(folder_filter) = request.filters.folders {
            messages = self.apply_folder_filter(messages, folder_filter).await?;
        }
        
        // Phase 5: Rank and limit
        self.rank_and_limit(messages, request.limit.unwrap_or(20))
    }
    
    async fn apply_label_filter(
        &self,
        messages: Vec<Message>,
        filter: LabelFilter,
    ) -> Result<Vec<Message>, RepositoryError> {
        // Fetch conversation labels for all messages
        let conv_ids: Vec<Uuid> = messages.iter().map(|m| m.conversation_id).collect();
        let conversations = self.repo.find_by_ids(conv_ids).await?;
        
        let conv_map: HashMap<Uuid, Conversation> = conversations
            .into_iter()
            .map(|c| (c.id, c))
            .collect();
        
        // Filter based on label mode
        let filtered: Vec<Message> = messages.into_iter().filter(|msg| {
            if let Some(conv) = conv_map.get(&msg.conversation_id) {
                let conv_labels: HashSet<String> = conv.label
                    .split(',')
                    .map(|l| l.trim().to_string())
                    .collect();
                
                match filter.mode {
                    LabelFilterMode::Any => {
                        // Match if conversation has ANY of the filter labels
                        filter.labels.iter().any(|l| conv_labels.contains(l))
                    }
                    LabelFilterMode::All => {
                        // Match if conversation has ALL of the filter labels
                        filter.labels.iter().all(|l| conv_labels.contains(l))
                    }
                    LabelFilterMode::Exact => {
                        // Match if conversation labels exactly match filter labels
                        let filter_set: HashSet<String> = filter.labels.iter().cloned().collect();
                        conv_labels == filter_set
                    }
                    LabelFilterMode::None => {
                        // Match if conversation has NONE of the filter labels
                        !filter.labels.iter().any(|l| conv_labels.contains(l))
                    }
                }
            } else {
                false
            }
        }).collect();
        
        Ok(filtered)
    }
    
    async fn apply_folder_filter(
        &self,
        messages: Vec<Message>,
        filter: FolderFilter,
    ) -> Result<Vec<Message>, RepositoryError> {
        let conv_ids: Vec<Uuid> = messages.iter().map(|m| m.conversation_id).collect();
        let conversations = self.repo.find_by_ids(conv_ids).await?;
        
        let conv_map: HashMap<Uuid, Conversation> = conversations
            .into_iter()
            .map(|c| (c.id, c))
            .collect();
        
        let filtered: Vec<Message> = messages.into_iter().filter(|msg| {
            if let Some(conv) = conv_map.get(&msg.conversation_id) {
                let matches = filter.paths.iter().any(|filter_path| {
                    if filter.recursive {
                        // Match if conversation folder starts with filter path
                        conv.folder.starts_with(filter_path)
                    } else {
                        // Exact match only
                        conv.folder == *filter_path
                    }
                });
                
                match filter.mode {
                    FolderFilterMode::Include => matches,
                    FolderFilterMode::Exclude => !matches,
                }
            } else {
                false
            }
        }).collect();
        
        Ok(filtered)
    }
}
```

### **SDK Usage**

```python
# Python SDK
results = await memory.smart_query(
    query="database schema design",
    filters={
        "labels": {
            "mode": "any",
            "labels": ["Backend", "Database"]
        },
        "date_range": {
            "preset": "last_month"
        },
        "importance": {
            "tier": "high"
        },
        "folders": {
            "paths": ["/work/projects"],
            "recursive": True,
            "mode": "include"
        },
        "roles": ["assistant"]  # Only assistant responses
    },
    limit=50
)
```

```typescript
// JavaScript SDK
const results = await memory.smartQuery({
  query: "authentication implementation",
  filters: {
    labels: {
      mode: "all",
      labels: ["Security", "Auth"]
    },
    dateRange: {
      start: new Date("2026-01-01"),
      end: new Date("2026-01-24")
    },
    importance: {
      min: 7.0
    },
    folders: {
      paths: ["/work", "/research"],
      recursive: false,
      mode: "include"
    }
  },
  limit: 30
});
```

***

## **23. Export Streaming**

### **True Streaming Implementation**

```rust
// Server-sent events for streaming
use axum::response::sse::{Event, KeepAlive, Sse};
use futures::stream::{self, Stream};

pub async fn export_stream(
    Path(conversation_id): Path<Uuid>,
    Query(options): Query<ExportOptions>,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = stream::try_unfold(
        ExportState::new(conversation_id, options),
        |mut state| async move {
            match state.next_chunk().await {
                Ok(Some(chunk)) => {
                    let event = Event::default()
                        .event("chunk")
                        .data(serde_json::to_string(&chunk).unwrap());
                    Ok(Some((event, state)))
                }
                Ok(None) => {
                    // Final event
                    let event = Event::default()
                        .event("done")
                        .data("Export complete");
                    Ok(Some((event, state)))
                }
                Err(e) => {
                    let event = Event::default()
                        .event("error")
                        .data(e.to_string());
                    Err(e)
                }
            }
        },
    );
    
    Sse::new(stream).keep_alive(KeepAlive::default())
}

struct ExportState {
    conversation_id: Uuid,
    options: ExportOptions,
    offset: usize,
    batch_size: usize,
    db: DatabaseConnection,
}

impl ExportState {
    async fn next_chunk(&mut self) -> Result<Option<ExportChunk>, ExportError> {
        // Fetch next batch of messages
        let messages = messages::Entity::find()
            .filter(messages::Column::ConversationId.eq(self.conversation_id))
            .order_by_asc(messages::Column::Timestamp)
            .offset(self.offset as u64)
            .limit(self.batch_size as u64)
            .all(&self.db)
            .await?;
        
        if messages.is_empty() {
            return Ok(None);
        }
        
        self.offset += messages.len();
        
        // Format chunk based on export format
        let content = match self.options.format {
            ExportFormat::Markdown => self.format_markdown(&messages),
            ExportFormat::Json => self.format_json(&messages),
            ExportFormat::Plain => self.format_plain(&messages),
        };
        
        Ok(Some(ExportChunk {
            offset: self.offset - messages.len(),
            size: messages.len(),
            content,
            progress: self.calculate_progress().await?,
        }))
    }
    
    async fn calculate_progress(&self) -> Result<f32, ExportError> {
        let total = messages::Entity::find()
            .filter(messages::Column::ConversationId.eq(self.conversation_id))
            .count(&self.db)
            .await?;
        
        Ok((self.offset as f32 / total as f32) * 100.0)
    }
}

#[derive(Debug, Serialize)]
struct ExportChunk {
    offset: usize,
    size: usize,
    content: String,
    progress: f32,
}
```

### **SDK Implementation**

```python
# Python SDK with async iterator
async def export_stream(
    self,
    conversation_id: str,
    format: ExportFormat = ExportFormat.MARKDOWN,
) -> AsyncIterator[ExportChunk]:
    """Stream export chunks as they're generated."""
    
    url = f"{self.base_url}/api/v1/conversations/{conversation_id}/export/stream"
    params = {"format": format.value}
    
    async with self.client.stream("GET", url, params=params) as response:
        async for line in response.aiter_lines():
            if line.startswith("data: "):
                data = line[6:]  # Remove "data: " prefix
                
                if data == "Export complete":
                    break
                
                chunk = ExportChunk.from_json(data)
                yield chunk

# Usage
async for chunk in memory.export_stream(conv_id):
    print(f"Progress: {chunk.progress:.1f}%")
    file.write(chunk.content)
```

```typescript
// JavaScript SDK
async *exportStream(
  conversationId: string,
  options: ExportOptions = {}
): AsyncGenerator<ExportChunk> {
  const url = `${this.baseUrl}/api/v1/conversations/${conversationId}/export/stream`;
  const params = new URLSearchParams({ format: options.format || 'markdown' });
  
  const response = await fetch(`${url}?${params}`, {
    headers: this.headers,
  });
  
  const reader = response.body!.getReader();
  const decoder = new TextDecoder();
  
  let buffer = '';
  
  while (true) {
    const { done, value } = await reader.read();
    
    if (done) break;
    
    buffer += decoder.decode(value, { stream: true });
    
    const lines = buffer.split('\n');
    buffer = lines.pop() || '';
    
    for (const line of lines) {
      if (line.startsWith('data: ')) {
        const data = line.slice(6);
        
        if (data === 'Export complete') {
          return;
        }
        
        const chunk = JSON.parse(data) as ExportChunk;
        yield chunk;
      }
    }
  }
}

// Usage
for await (const chunk of memory.exportStream(convId, { format: 'markdown' })) {
  console.log(`Progress: ${chunk.progress.toFixed(1)}%`);
  await writeToFile(chunk.content);
}
```

***

## **24. Duplicate Detection - Explanation**

**Your question**: "How could a conversation be stored twice?"

You're right - with UUIDs generated by the system, the same conversation can't be stored twice **by normal operation**.

**However**, edge cases exist:

1. **Manual import from backup**: User imports `sekha-backup.db` that contains conversations already in their current DB
2. **Multiple SDK instances**: User accidentally runs two scripts that create the same conversation from external source
3. **Retry logic**: Network failure causes client to retry `create_conversation` call, server processes it twice

**Solution**: Content-based deduplication

```rust
// Add content hash to conversations table
ALTER TABLE conversations ADD COLUMN content_hash TEXT;
CREATE UNIQUE INDEX idx_conversations_content_hash ON conversations(content_hash) WHERE content_hash IS NOT NULL;

impl ConversationRepository {
    async fn create_conversation_idempotent(
        &self,
        conv: NewConversation,
    ) -> Result<Conversation, RepositoryError> {
        // Calculate content hash
        let content_hash = self.calculate_conversation_hash(&conv);
        
        // Check if conversation with same hash exists
        if let Some(existing) = self.find_by_content_hash(&content_hash).await? {
            return Ok(existing);  // Return existing instead of error
        }
        
        // Create new conversation
        let conversation = Conversation {
            content_hash: Some(content_hash),
            ..conv.into()
        };
        
        self.insert(conversation).await
    }
    
    fn calculate_conversation_hash(&self, conv: &NewConversation) -> String {
        use sha2::{Sha256, Digest};
        
        let mut hasher = Sha256::new();
        
        // Hash conversation metadata
        hasher.update(conv.label.as_bytes());
        hasher.update(conv.folder.as_bytes());
        
        // Hash all messages (role + content)
        for msg in &conv.messages {
            hasher.update(format!("{:?}", msg.role).as_bytes());
            hasher.update(msg.content.as_bytes());
        }
        
        format!("{:x}", hasher.finalize())
    }
}
```

**But** this is probably **not needed** for your use case. Proxies capture in real-time, one conversation at a time. Duplication is unlikely.

***

## **25. Multi-Model Embedding Support**

### **Pluggable Embedding Provider Architecture**

```rust
// src/embeddings/mod.rs
use async_trait::async_trait;

#[async_trait]
pub trait EmbeddingProvider: Send + Sync {
    async fn embed_text(&self, text: &str) -> Result<Vec<f32>, EmbeddingError>;
    async fn embed_batch(&self, texts: Vec<&str>) -> Result<Vec<Vec<f32>>, EmbeddingError>;
    fn model_name(&self) -> &str;
    fn dimensions(&self) -> usize;
}

// Ollama provider (existing)
pub struct OllamaProvider {
    client: reqwest::Client,
    base_url: String,
    model: String,
    dimensions: usize,
}

#[async_trait]
impl EmbeddingProvider for OllamaProvider {
    async fn embed_text(&self, text: &str) -> Result<Vec<f32>, EmbeddingError> {
        let response = self.client
            .post(format!("{}/api/embeddings", self.base_url))
            .json(&serde_json::json!({
                "model": self.model,
                "prompt": text
            }))
            .send()
            .await?
            .json::<OllamaEmbeddingResponse>()
            .await?;
        
        Ok(response.embedding)
    }
    
    fn model_name(&self) -> &str {
        &self.model
    }
    
    fn dimensions(&self) -> usize {
        self.dimensions
    }
}

// OpenAI provider
pub struct OpenAIProvider {
    client: reqwest::Client,
    api_key: String,
    model: String,
}

#[async_trait]
impl EmbeddingProvider for OpenAIProvider {
    async fn embed_text(&self, text: &str) -> Result<Vec<f32>, EmbeddingError> {
        let response = self.client
            .post("https://api.openai.com/v1/embeddings")
            .header("Authorization", format!("Bearer {}", self.api_key))
            .json(&serde_json::json!({
                "model": self.model,
                "input": text
            }))
            .send()
            .await?
            .json::<OpenAIEmbeddingResponse>()
            .await?;
        
        Ok(response.data[0].embedding.clone())
    }
    
    fn model_name(&self) -> &str {
        &self.model
    }
    
    fn dimensions(&self) -> usize {
        match self.model.as_str() {
            "text-embedding-3-small" => 1536,
            "text-embedding-3-large" => 3072,
            "text-embedding-ada-002" => 1536,
            _ => 1536,
        }
    }
    
    async fn embed_batch(&self, texts: Vec<&str>) -> Result<Vec<Vec<f32>>, EmbeddingError> {
        let response = self.client
            .post("https://api.openai.com/v1/embeddings")
            .header("Authorization", format!("Bearer {}", self.api_key))
            .json(&serde_json::json!({
                "model": self.model,
                "input": texts
            }))
            .send()
            .await?
            .json::<OpenAIEmbeddingResponse>()
            .await?;
        
        Ok(response.data.into_iter().map(|d| d.embedding).collect())
    }
}

// Cohere provider
pub struct CohereProvider {
    client: reqwest::Client,
    api_key: String,
    model: String,
}

#[async_trait]
impl EmbeddingProvider for CohereProvider {
    async fn embed_text(&self, text: &str) -> Result<Vec<f32>, EmbeddingError> {
        let response = self.client
            .post("https://api.cohere.ai/v1/embed")
            .header("Authorization", format!("Bearer {}", self.api_key))
            .json(&serde_json::json!({
                "model": self.model,
                "texts": [text],
                "input_type": "search_document"
            }))
            .send()
            .await?
            .json::<CohereEmbeddingResponse>()
            .await?;
        
        Ok(response.embeddings[0].clone())
    }
    
    fn model_name(&self) -> &str {
        &self.model
    }
    
    fn dimensions(&self) -> usize {
        match self.model.as_str() {
            "embed-english-v3.0" => 1024,
            "embed-multilingual-v3.0" => 1024,
            _ => 1024,
        }
    }
}

// Provider factory
pub struct EmbeddingProviderFactory;

impl EmbeddingProviderFactory {
    pub fn create(config: &EmbeddingConfig) -> Result<Box<dyn EmbeddingProvider>, ConfigError> {
        match config.provider.as_str() {
            "ollama" => Ok(Box::new(OllamaProvider {
                client: reqwest::Client::new(),
                base_url: config.ollama_url.clone().unwrap_or_else(|| "http://localhost:11434".to_string()),
                model: config.model.clone(),
                dimensions: config.dimensions,
            })),
            "openai" => Ok(Box::new(OpenAIProvider {
                client: reqwest::Client::new(),
                api_key: config.api_key.clone().ok_or(ConfigError::MissingApiKey)?,
                model: config.model.clone(),
            })),
            "cohere" => Ok(Box::new(CohereProvider {
                client: reqwest::Client::new(),
                api_key: config.api_key.clone().ok_or(ConfigError::MissingApiKey)?,
                model: config.model.clone(),
            })),
            _ => Err(ConfigError::UnknownProvider(config.provider.clone())),
        }
    }
}

// Configuration
#[derive(Debug, Deserialize)]
pub struct EmbeddingConfig {
    pub provider: String,  // "ollama", "openai", "cohere"
    pub model: String,
    pub dimensions: usize,
    pub api_key: Option<String>,
    pub ollama_url: Option<String>,
}
```

### **Configuration File**

```yaml
# config.yaml
embeddings:
  provider: "openai"  # or "ollama", "cohere"
  model: "text-embedding-3-large"
  api_key: "${OPENAI_API_KEY}"  # From environment
  
  # Fallback provider if primary fails
  fallback:
    provider: "ollama"
    model: "nomic-embed-text"
    ollama_url: "http://localhost:11434"
```

### **Migration Between Providers**

```bash
sekha embeddings migrate \
  --from ollama/nomic-embed-text \
  --to openai/text-embedding-3-large \
  --batch-size 100 \
  --dry-run

# Migrates all embeddings to new provider
# Creates new Chroma collection
# Updates embedding_config records
# Processes in batches to avoid rate limits
```

***

## **26. Backup/Restore**

### **One-Click Backup System**

```rust
// POST /api/v1/admin/backup
pub async fn create_backup(
    State(state): State<AppState>,
    Json(options): Json<BackupOptions>,
) -> Result<Json<BackupResult>, ApiError> {
    let backup_id = Uuid::new_v4();
    let backup_dir = PathBuf::from(&options.destination)
        .join(format!("sekha-backup-{}", chrono::Utc::now().format("%Y%m%d-%H%M%S")));
    
    fs::create_dir_all(&backup_dir).await?;
    
    // Phase 1: Backup SQLite database
    let db_backup_path = backup_dir.join("sekha.db");
    state.db.backup_to(&db_backup_path).await?;
    
    // Phase 2: Backup Chroma data
    let chroma_backup_path = backup_dir.join("chroma");
    state.chroma.export_collections(&chroma_backup_path).await?;
    
    // Phase 3: Create manifest
    let manifest = BackupManifest {
        id: backup_id,
        created_at: chrono::Utc::now(),
        version: env!("CARGO_PKG_VERSION").to_string(),
        schema_version: state.schema_version,
        embedding_config: state.embedding_config.clone(),
        statistics: BackupStatistics {
            conversations: state.repo.count_conversations().await?,
            messages: state.repo.count_messages().await?,
            embeddings: state.chroma.count_embeddings().await?,
        },
        checksums: Checksums {
            database: calculate_checksum(&db_backup_path).await?,
            chroma: calculate_directory_checksum(&chroma_backup_path).await?,
        },
    };
    
    let manifest_path = backup_dir.join("manifest.json");
    fs::write(&manifest_path, serde_json::to_string_pretty(&manifest)?).await?;
    
    // Phase 4: Compress if requested
    let final_path = if options.compress {
        let archive_path = backup_dir.with_extension("tar.gz");
        compress_directory(&backup_dir, &archive_path).await?;
        fs::remove_dir_all(&backup_dir).await?;
        archive_path
    } else {
        backup_dir
    };
    
    Ok(Json(BackupResult {
        id: backup_id,
        path: final_path.to_string_lossy().to_string(),
        size_bytes: calculate_size(&final_path).await?,
        duration_ms: 0, // TODO: track duration
    }))
}

#[derive(Debug, Serialize, Deserialize)]
pub struct BackupManifest {
    pub id: Uuid,
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub version: String,
    pub schema_version: i32,
    pub embedding_config: EmbeddingConfig,
    pub statistics: BackupStatistics,
    pub checksums: Checksums,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct BackupStatistics {
    pub conversations: u64,
    pub messages: u64,
    pub embeddings: u64,
}

// POST /api/v1/admin/restore
pub async fn restore_backup(
    State(state): State<AppState>,
    Json(options): Json<RestoreOptions>,
) -> Result<Json<RestoreResult>, ApiError> {
    let backup_path = PathBuf::from(&options.backup_path);
    
    // Phase 1: Extract if compressed
    let work_dir = if backup_path.extension() == Some(OsStr::new("gz")) {
        let extract_dir = backup_path.with_extension("");
        decompress_archive(&backup_path, &extract_dir).await?;
        extract_dir
    } else {
        backup_path.clone()
    };
    
    // Phase 2: Validate manifest
    let manifest_path = work_dir.join("manifest.json");
    let manifest: BackupManifest = serde_json::from_str(
        &fs::read_to_string(&manifest_path).await?
    )?;
    
    // Verify checksums
    let db_path = work_dir.join("sekha.db");
    let db_checksum = calculate_checksum(&db_path).await?;
    if db_checksum != manifest.checksums.database {
        return Err(ApiError::CorruptedBackup("Database checksum mismatch"));
    }
    
    let chroma_path = work_dir.join("chroma");
    let chroma_checksum = calculate_directory_checksum(&chroma_path).await?;
    if chroma_checksum != manifest.checksums.chroma {
        return Err(ApiError::CorruptedBackup("Chroma checksum mismatch"));
    }
    
    // Phase 3: Check schema compatibility
    if manifest.schema_version > state.schema_version {
        return Err(ApiError::IncompatibleSchema {
            backup: manifest.schema_version,
            current: state.schema_version,
        });
    }
    
    // Phase 4: Restore database
    if options.strategy == RestoreStrategy::Replace {
        // Stop services
        state.shutdown().await?;
        
        // Replace database
        fs::copy(&db_path, &state.config.database_path).await?;
        
        // Replace Chroma data
        fs::remove_dir_all(&state.config.chroma_data_path).await?;
        copy_directory(&chroma_path, &state.config.chroma_data_path).await?;
        
        // Restart services
        state.start().await?;
    } else {
        // Merge strategy
        import_from_backup(&db_path, &state.db, options.conflict_strategy).await?;
    }
    
    Ok(Json(RestoreResult {
        conversations_restored: manifest.statistics.conversations,
        messages_restored: manifest.statistics.messages,
        embeddings_restored: manifest.statistics.embeddings,
    }))
}
```

### **CLI Commands**

```bash
# Create backup
sekha backup create \
  --destination ~/backups \
  --compress \
  --description "Before major upgrade"

# Output:
# Creating backup...
# ✓ Database backed up (142 MB)
# ✓ Chroma data backed up (89 MB)
# ✓ Manifest created
# ✓ Compressed to sekha-backup-20260124-203000.tar.gz (87 MB)
# 
# Backup ID: a1b2c3d4-...
# Location: ~/backups/sekha-backup-20260124-203000.tar.gz
# Total size: 87 MB
# Conversations: 1,234
# Messages: 45,678
# Embeddings: 45,678

# List backups
sekha backup list

# Restore backup
sekha backup restore \
  ~/backups/sekha-backup-20260124-203000.tar.gz \
  --strategy replace \
  --verify-checksums

# Or merge with existing data
sekha backup restore \
  ~/backups/sekha-backup-20260124-203000.tar.gz \
  --strategy merge \
  --conflict skip
```

### **SDK Support**

```python
# Python SDK
backup = await memory.create_backup(
    destination="~/backups",
    compress=True,
    description="Monthly backup"
)

print(f"Backup created: {backup.path}")
print(f"Size: {backup.size_bytes / 1024 / 1024:.1f} MB")

# Restore
result = await memory.restore_backup(
    backup_path="~/backups/sekha-backup-20260124-203000.tar.gz",
    strategy="replace",
    verify_checksums=True
)

print(f"Restored {result.conversations_restored} conversations")
```

***

## **27. Search Explanation - Considered**

You're right - users can **ask Sekha** why a result was ranked highly, and the LLM will explain based on the context. The metadata is already there (semantic scores, labels, timestamps).

**However**, programmatic access to scoring breakdown is still valuable for:
- Debugging
- Fine-tuning the ranking algorithm
- Transparency for developers

**Lightweight addition**:
```rust
#[derive(Debug, Serialize)]
pub struct ScoredMessage {
    pub message: Message,
    pub total_score: f32,
    pub scoring_breakdown: Option<ScoringBreakdown>,  // Optional, only if requested
}

#[derive(Debug, Serialize)]
pub struct ScoringBreakdown {
    pub semantic_similarity: f32,
    pub recency_factor: f32,
    pub importance_weight: f32,
    pub label_boost: f32,
}
```

Include only if `?include_scoring=true` query parameter present.

***

## **28. Conversation Templates - Value & Design**

### **Value Proposition**

Templates automate repetitive conversation creation patterns:

**Use Cases**:
1. **Daily Standups**: Auto-create with label `Standup:YYYY-MM-DD`, folder `/work/standups`, importance 5
2. **Project Init**: New project conversations get label `Project:{name}`, pinned, importance 8
3. **Meeting Notes**: Auto-label with `Meeting:YYYY-MM-DD:{topic}`, folder `/work/meetings`
4. **Personal Journal**: Daily journal entries with consistent labels and folder structure

### **World-Class Design**

```rust
// Template definition
#[derive(Debug, Serialize, Deserialize)]
pub struct ConversationTemplate {
    pub id: Uuid,
    pub name: String,
    pub description: String,
    pub folder_pattern: String,  // "/work/standups/{date}"
    pub label_pattern: String,   // "Standup:{date}"
    pub importance_score: f32,
    pub auto_apply_rules: Vec<AutoApplyRule>,
    pub default_metadata: serde_json::Value,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct AutoApplyRule {
    pub trigger: Trigger,
    pub conditions: Vec<Condition>,
}

#[derive(Debug, Serialize, Deserialize)]
pub enum Trigger {
    OnCreate,
    OnSchedule(Schedule),  // Daily, Weekly, etc.
    OnFolderMatch(String),
}

#[derive(Debug, Serialize, Deserialize)]
pub struct Schedule {
    pub frequency: Frequency,
    pub time: chrono::NaiveTime,
    pub timezone: String,
}

#[derive(Debug, Serialize, Deserialize)]
pub enum Frequency {
    Daily,
    Weekly { day: chrono::Weekday },
    Monthly { day: u32 },
}

// Template variables
pub struct TemplateVariables {
    pub date: String,           // 2026-01-24
    pub datetime: String,       // 2026-01-24T20:30:00
    pub weekday: String,        // Friday
    pub week_number: u32,       // 4
    pub month: String,          // January
    pub year: u32,              // 2026
    pub custom: HashMap<String, String>,
}

impl ConversationTemplate {
    pub fn apply(&self, variables: &TemplateVariables) -> NewConversation {
        let folder = self.interpolate(&self.folder_pattern, variables);
        let label = self.interpolate(&self.label_pattern, variables);
        
        NewConversation {
            folder,
            label,
            importance_score: self.importance_score,
            metadata: self.default_metadata.clone(),
            ..Default::default()
        }
    }
    
    fn interpolate(&self, pattern: &str, vars: &TemplateVariables) -> String {
        let mut result = pattern.to_string();
        
        result = result.replace("{date}", &vars.date);
        result = result.replace("{datetime}", &vars.datetime);
        result = result.replace("{weekday}", &vars.weekday);
        result = result.replace("{month}", &vars.month);
        result = result.replace("{year}", &vars.year.to_string());
        
        // Custom variables
        for (key, value) in &vars.custom {
            result = result.replace(&format!("{{{}}}", key), value);
        }
        
        result
    }
}
```

### **API & SDK**

```bash
# Create template
sekha templates create \
  --name "Daily Standup" \
  --folder "/work/standups/{date}" \
  --label "Standup:{date}" \
  --importance 5 \
  --schedule daily \
  --time "09:00" \
  --timezone "America/New_York"

# Apply template manually
sekha conversations create \
  --template "Daily Standup" \
  --vars "date=2026-01-24"

# List templates
sekha templates list
```

```python
# Python SDK
template = await memory.create_template(
    name="Project Kickoff",
    folder_pattern="/work/projects/{project_name}",
    label_pattern="Project:{project_name}",
    importance_score=8.0,
    auto_apply_rules=[
        AutoApplyRule(
            trigger=Trigger.ON_FOLDER_MATCH("/work/projects/*"),
            conditions=[]
        )
    ]
)

# Use template
conversation = await memory.create_conversation_from_template(
    template_id=template.id,
    variables={"project_name": "Sekha V2"}
)
```

***

## **29. Conversation Locking - Clarification**

**Your question**: "If both editors are writing to the same db... why would both not be received?"

You're correct for **single-user, single-instance deployment**. Both writes go to the same SQLite file sequentially.

**However**, consider this scenario:

1. User opens conversation in Claude Desktop (via MCP)
2. User also opens conversation in browser (via API)
3. **Both clients fetch conversation** (version 1, last_updated: 2026-01-24 19:00:00)
4. User adds message in Claude: "Implement feature X"
5. User adds message in browser: "Implement feature Y"
6. **Both clients save** to the same conversation_id
7. Last write wins - one message is lost

**This is a UX issue**, not a data corruption issue.

**Solutions**:

**Option A**: Optimistic locking (recommended)
```rust
// Add version number to conversations
ALTER TABLE conversations ADD COLUMN version INTEGER NOT NULL DEFAULT 1;

// Update requires version match
UPDATE conversations 
SET label = ?, version = version + 1
WHERE id = ? AND version = ?;
-- If rows_affected = 0, version conflict
```

**Option B**: Append-only messages (already the case)
Messages are never updated, only appended. Conflicts are impossible.

**Option C**: Real-time sync (WebSocket)
Notify all clients when conversation updates.

For your use case (single user), **Option B is already sufficient**. Messages are append-only, so no conflict.

***

## **30. Observability - World-Class Design**

```rust
// Prometheus metrics endpoint
use prometheus::{Encoder, TextEncoder, Registry, Counter, Histogram, Gauge};

pub struct Metrics {
    registry: Registry,
    
    // Conversation metrics
    pub conversations_total: Gauge,
    pub conversations_created: Counter,
    pub conversation_create_duration: Histogram,
    
    // Message metrics
    pub messages_total: Gauge,
    pub messages_stored: Counter,
    pub message_store_duration: Histogram,
    
    // Search metrics
    pub searches_total: Counter,
    pub search_duration: Histogram,
    pub search_results_count: Histogram,
    
    // Embedding metrics
    pub embeddings_generated: Counter,
    pub embedding_duration: Histogram,
    pub embedding_errors: Counter,
    
    // Database metrics
    pub db_query_duration: Histogram,
    pub db_connections_active: Gauge,
    
    // Chroma metrics
    pub chroma_requests: Counter,
    pub chroma_errors: Counter,
    pub chroma_duration: Histogram,
}

impl Metrics {
    pub fn new() -> Self {
        let registry = Registry::new();
        
        let conversations_total = Gauge::new("sekha_conversations_total", "Total conversations").unwrap();
        let conversations_created = Counter::new("sekha_conversations_created_total", "Conversations created").unwrap();
        let conversation_create_duration = Histogram::new("sekha_conversation_create_duration_seconds", "Conversation creation duration").unwrap();
        
        // ... register all metrics ...
        
        registry.register(Box::new(conversations_total.clone())).unwrap();
        registry.register(Box::new(conversations_created.clone())).unwrap();
        
        Self {
            registry,
            conversations_total,
            conversations_created,
            conversation_create_duration,
            // ... rest ...
        }
    }
}

// GET /metrics endpoint
pub async fn metrics_handler(
    State(metrics): State<Arc<Metrics>>,
) -> impl IntoResponse {
    let encoder = TextEncoder::new();
    let metric_families = metrics.registry.gather();
    
    let mut buffer = Vec::new();
    encoder.encode(&metric_families, &mut buffer).unwrap();
    
    Response::builder()
        .header("Content-Type", encoder.format_type())
        .body(Body::from(buffer))
        .unwrap()
}

// Instrument code
impl ConversationRepository {
    pub async fn create_conversation(&self, conv: NewConversation) -> Result<Conversation, RepositoryError> {
        let timer = self.metrics.conversation_create_duration.start_timer();
        
        let result = self._create_conversation_impl(conv).await;
        
        timer.observe_duration();
        
        if result.is_ok() {
            self.metrics.conversations_created.inc();
            self.metrics.conversations_total.inc();
        }
        
        result
    }
}
```

### **Health Check Endpoint**

```rust
// GET /health
#[derive(Debug, Serialize)]
pub struct HealthStatus {
    pub status: String,  // "healthy", "degraded", "unhealthy"
    pub version: String,
    pub uptime_seconds: u64,
    pub checks: HashMap<String, ComponentHealth>,
}

#[derive(Debug, Serialize)]
pub struct ComponentHealth {
    pub status: String,
    pub latency_ms: Option<f64>,
    pub error: Option<String>,
}

pub async fn health_check(State(state): State<AppState>) -> Json<HealthStatus> {
    let mut checks = HashMap::new();
    
    // Check database
    let db_health = check_database(&state.db).await;
    checks.insert("database".to_string(), db_health);
    
    // Check Chroma
    let chroma_health = check_chroma(&state.chroma).await;
    checks.insert("chroma".to_string(), chroma_health);
    
    // Check LLM bridge
    let llm_health = check_llm_bridge(&state.llm_bridge).await;
    checks.insert("llm_bridge".to_string(), llm_health);
    
    let overall_status = if checks.values().all(|h| h.status == "healthy") {
        "healthy"
    } else if checks.values().any(|h| h.status == "unhealthy") {
        "unhealthy"
    } else {
        "degraded"
    };
    
    Json(HealthStatus {
        status: overall_status.to_string(),
        version: env!("CARGO_PKG_VERSION").to_string(),
        uptime_seconds: state.start_time.elapsed().as_secs(),
        checks,
    })
}
```

### **Structured Logging**

```rust
use tracing::{info, warn, error, debug};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

// Initialize logging
fn init_logging() {
    tracing_subscriber::registry()
        .with(tracing_subscriber::fmt::layer().json())
        .with(tracing_subscriber::EnvFilter::from_default_env())
        .init();
}

// Use structured logs
#[tracing::instrument(skip(self))]
pub async fn create_conversation(&self, conv: NewConversation) -> Result<Conversation, RepositoryError> {
    info!(
        folder = %conv.folder,
        label = %conv.label,
        message_count = conv.messages.len(),
        "Creating conversation"
    );
    
    // ... implementation ...
    
    info!(
        conversation_id = %result.id,
        duration_ms = timer.elapsed().as_millis(),
        "Conversation created successfully"
    );
}
```

***


