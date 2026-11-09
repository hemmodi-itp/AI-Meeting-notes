# Databricks GenAI Engineer Associate: Short Notes

| Domain | Weight | Focus |
|---|---|---|
| 1. Design Applications | 14% | Pick the task, prompt design, chains/agents, tools, Agent Bricks |
| 2. Data Preparation | 14% | Parse → filter → chunk → Delta → Vector Search, retrieval metrics |
| **3. Application Development** | **30%** | Frameworks, Prompt Registry, guardrails, model selection, ResponsesAgent, Genie |
| **4. Assembling & Deploying** | **22%** | pyfunc/UC registry, Vector Search, serving, ai_query, memory, CI/CD, MCP |
| 5. Governance | 8% | PII mask/block, prompt injection, licensing |
| 6. Evaluation & Monitoring | 12% | mlflow.genai.evaluate, judges, scorers, inference tables, rate limits, cost |

> **Exam strategy:** most options are real features. Find the **deciding constraint** (exactness, volume, latency, least privilege, per-user identity, no redeploy, zero downtime, batch vs real-time) and pick the feature that directly meets it.

---

## DOMAIN 1: Design Applications (14%)

### Choose the task from the required OUTPUT
| Need | Task |
|---|---|
| Exactly one label from a fixed list | Single-label classification |
| All labels that apply from a fixed list | Multi-label classification |
| How the writer feels | Sentiment |
| Exact values from the text (serial no., meter reading) | **Extraction** (never a summary) |
| Shorter text | Summarization |
| Another language | Translation |
| Closeness in meaning or near-duplicates | Embeddings + vector similarity |
| Hide PII *inside* the text | Masking |

- Classification **chooses** from labels you supply; extraction returns **open-ended** values.
- Long English text to a short Spanish version = **summarize + translate**.
- **Don't add tasks nobody asked for** (extra sentiment or drafts add cost and risk).
- Break complex requests into one task per output. Example: classify the doc type first, then use that type's extraction schema.
- Fixed business rules (limits, eligibility) belong in **SQL or Python**, which is reproducible and auditable.
- Similarity at scale: compare **n items with embeddings**, not n(n−1)/2 LLM pairwise calls. For thousands of labels, embed and take the top candidates, then let the LLM pick from that short list.

### AI Functions vs ai_query
| Function | Returns |
|---|---|
| `ai_classify` | label |
| `ai_extract` | values pulled out of the text |
| `ai_mask` | the original text with placeholders |
| `ai_similarity` | a **number** |
| `ai_summarize` | shorter text (**max_words default 50; 0 = no limit**) |
| `ai_translate` | translated text |
| `ai_analyze_sentiment` | sentiment |
| **`ai_query(endpoint, request, ...)`** | output from a **named, fine-tuned or provisioned endpoint**, a custom prompt, or a JSON schema |

- The task-specific functions need no prompt and no endpoint (they use Databricks-managed models).
- `ai_query` options: `responseFormat` (schema), `modelParameters` (temperature, max_tokens), `failOnError => false` (an error per row instead of failing the whole query).

### Prompt design
- A prompt states the **task, context, format and restrictions**.
- **System prompt** holds standing rules and is sent with every turn. **User prompt** holds the current request.
- **Few-shot:** target the failing inputs and balance the labels. The model copies patterns, including the label distribution.
- For strict parsing, tell the model to return only the object: no lead-in and no ``` fences.

### Structured output
| Approach | Guarantee |
|---|---|
| Prompt "use JSON" | nothing |
| `json_object` | valid JSON, but the keys may vary |
| **`json_schema`** | fixed keys, types and enums |

- Repeating data → array of objects. Several values from a closed set → array with an enum. A field that may be absent → **nullable** type plus an instruction to return null.
- **Required ≠ non-null.** A required non-null string makes the model **invent** a value.
- Use an enum with an `other` value routed to human review.
- `minItems` and `maxLength` are **not guaranteed**, so validate them in code.
- **A schema guarantees shape, not truth.** Build the sources list in code from chunk metadata, because a model can invent URLs.
- Traps: **temperature, max_tokens and stop sequences are not format controls.** A low max_tokens can cut JSON off mid-object.

### Batch vs real-time
- Nobody is waiting and the volume is high → **batch with ai_query or AI Functions over a Delta table**.
- A user is waiting → a **Model Serving endpoint**. Streaming shows the first words sooner but does **not** make generation faster.

### Chain components (RAG order)
**retriever → prompt template → chat model → output parser** (memory is needed only for multi-turn).
- If the whole document always fits in the context window, put it directly in the prompt and skip retrieval.
- **Freshness comes from retrieval, not fine-tuning.** Fine-tuning changes style and behaviour.
- **History-aware query rewriting** turns a follow-up into a standalone query. Skip it on the first turn.
- **Metadata filters** (tenant_id, plan tier) go in the Vector Search query, so disallowed chunks never reach the prompt. The column must be in `columns_to_sync`, which **can't be changed on an existing Delta Sync index, so you rebuild the index**.

### Chain vs agent vs router
| Situation | Design |
|---|---|
| Same steps every time | **Sequential chain** (predictable cost) |
| Requests need different sources or actions | **Tool-calling agent** |
| Most traffic simple, a few requests complex | **Router + agent** |

### Tools
- The model reads the **tool name, description and parameter schema**. For UC functions that means the function COMMENT and parameter COMMENTs (table docs do **not** reach the model).
- **Wrong tool chosen → improve the tool descriptions.** Reranking, more results or retrying won't fix it.
- Give each retriever tool a distinct `tool_name` and `tool_description`. Spell out formats such as ISO dates. Put today's date in the system prompt.

| Need | Tool |
|---|---|
| Exact live record by ID | **UC function** on the live table |
| Guidance in documents or policies | **Vector Search retriever** |
| Analytics over tables | **Genie space** |
| Change state (refund, cancel) | **Purpose-built action tool** calling the owning service's API |

- Dependent calls run **sequentially**. Independent calls run in **parallel** (latency ≈ the slowest call). Check the policy and eligibility **before** an irreversible action.
- **Safe action tools:** enforce **preconditions inside the tool**, **least privilege** and an **idempotency key** (a timeout ≠ failure). Prompts are **not** a security boundary. Avoid "run any SQL" tools.

### Agent Bricks
| Agent | Use |
|---|---|
| **Knowledge Assistant** | Q&A over documents **with citations** (sources: UC volume files, a UC table, or a VS index; file sources need a manual Sync). Not for exact live lookups. |
| **Information Extraction** | The same fields from every document into a table. **Field descriptions drive quality.** PDFs are parsed with `ai_parse_document`. |
| **Multiagent Supervisor** | One entry point that routes to subagents **by their description**. Runs as the end user, not the creator. |

- Use a **custom Agent Framework agent** (logged with MLflow) when mandatory custom code, internally computed filters or a custom output contract is needed.
- Experts need **CAN MANAGE** to edit an agent. Retrieval is good but answers miss company conventions → improve the **examples and guidelines**, not fine-tuning.

---

## DOMAIN 2: Data Preparation (14%)

**Flow:** source docs → extract → filter → chunk → UC Delta table (CDF on) → Delta Sync VS index → evaluate.

> **Coverage gap vs retrieval problem:** if the fact is in no chunk, **ingest the source**. No retrieval setting returns text that was never ingested.

### Sources
- Prefer **authoritative, current, relevant** sources that are **safe for the audience**. Keep sensitive data **out at ingestion**; prompts and filters don't make it safe afterwards.

### Extraction tools
| Source | Tool |
|---|---|
| Text-layer PDF | pypdf / PyMuPDF |
| PDF tables as rows | **pdfplumber** (`extract_tables`) |
| Scanned PDF or image | **OCR: Tesseract / pytesseract** |
| Mixed formats | unstructured `partition_pdf` (**hi_res** + `infer_table_structure=True` keeps tables, **fast** doesn't) |
| HTML | BeautifulSoup / lxml (drop navigation and footers) |
| Word / PPT | python-docx / python-pptx |
| No installs, SQL only | **`ai_parse_document`** (PDF, images, DOC, PPT, scanned pages; read with `read_files(format=>'binaryFile')`; returns a VARIANT; **500-page limit**, use `pageRange` for larger files) |

### Filtering
- Remove headers, footers, page numbers, navigation, cookie banners, signatures and superseded versions. Use **MinHash** to find near-duplicates.
- **Don't** lowercase, strip punctuation or remove stop words (that breaks identifiers). **Don't drop tables.**
- Versions: if only current rules matter, ingest the current version with an effective date. For questions about past dates, ingest every version with `effective_from`/`effective_to` and filter. **A reranker scores relevance, not recency.**

### Chunking
| Doc | Splitter |
|---|---|
| Markdown | `MarkdownHeaderTextSplitter` (headings become metadata) |
| HTML | `HTMLHeaderTextSplitter` |
| Prose | `RecursiveCharacterTextSplitter` |
| Size only | Character / TokenTextSplitter |

- Follow a header splitter with a size splitter. For tables, split into row groups and **repeat the header** in each, or write rows as self-describing text ("Model: X200; Weight: 4 kg").
- **Embedding limits:** `bge-large-en` = **512 tokens**, `gte-large-en` = **8,192 tokens** (both 1,024 dims). Longer text is **silently truncated**. Measure with the embedding model's tokenizer.
- **Overlap: 10–20%**, and it comes **out of** chunk_size. A 50% overlap roughly doubles the chunk count.
- **Context budget:** prompt + history + (k × chunk size) + answer reserve ≤ the context window.

### Delta table → index
- Read files as binaryFile, then parse, chunk, explode, add a **unique ID** and metadata, `saveAsTable`, and **enable Change Data Feed**.
- Delta Sync needs a **UC Delta table with a primary key** and **CDF**. `_id` is reserved. Temporary views and path-only files can't be the source.
- A metadata column allows **filtering**. Prepending the source name to the chunk text does not.
- Incremental: **Auto Loader** (`cloudFiles`, binaryFile, a **fixed checkpointLocation**, `availableNow=True`). A new checkpoint reprocesses everything.
- For a changed doc, delete its old chunks and append the new ones (or use a scoped `replaceWhere`). A full `overwrite` re-embeds everything.
- Grants: `READ VOLUME`, `CREATE TABLE` on the schema, `MODIFY`+`SELECT` to append, plus `USE CATALOG`/`USE SCHEMA`. Run as a service principal.

### Retrieval metrics
- **Precision@k** = relevant results in the top k ÷ k. **Recall@k** = relevant results in the top k ÷ all relevant.
  - Example: 5 relevant chunks, 3 of them in the top 8 → P@8 = 3/8 = **37.5%**, R@8 = 3/5 = **60%**.
- Measure at the **k actually passed to the LLM**.
- **MRR** = the average of 1/rank of the first relevant result (one right answer). **NDCG** = graded relevance where order matters.
- Raw similarity scores are **not** a quality metric.

### MLflow retrieval judges
| Judge | Compares | Needs labels? |
|---|---|---|
| RetrievalRelevance | each chunk vs the request | No |
| RetrievalGroundedness | the response vs the chunks (hallucination) | No |
| RetrievalSufficiency | the chunks vs the expected facts | **Yes** |

- Sufficiency ✓ and Groundedness ✗ → a **generation** problem. Groundedness ✓ and Sufficiency ✗ → a **retrieval** problem.
- Judges read **RETRIEVER spans**: use `@mlflow.trace(span_type=SpanType.RETRIEVER)` and return a **list of Documents** (not one joined string).
- Eval set: golden (SME-written), synthetic (LLM-generated, SME-sampled) or production (real but unlabeled). Label **expected facts + source doc IDs, not chunk IDs**, so the labels survive re-chunking.

### Fix by symptom
| Symptom | Fix |
|---|---|
| Small chunk matches but surrounding conditions are missing | **Parent-child** (index the children, fetch the parent) |
| No headings, mixed topics (transcripts) | **Semantic chunking** |
| Chunk doesn't say which product or document it is from | **Contextual enrichment** (prepend the title before embedding; this changes the vector, while a metadata column doesn't) |
| An exact code or part number is missed | **Hybrid search** (`query_type="HYBRID"`, merged with RRF) |
| Correct chunk retrieved but ranked low | **Reranker** |
| Correct chunk not in the candidates at all | **Improve first-stage retrieval** (a reranker can only reorder) |

- `DatabricksReranker(columns_to_rerank=["text","title"])` needs no new endpoint. It reads about **2,000 chars**, so put the text first. It fits a latency budget of about a second, **not** millisecond autocomplete.
- query_type: **ANN** (default), **FULL_TEXT** (keyword only), **HYBRID**.

---

## DOMAIN 3: Application Development (30%) ⭐

### Frameworks
| Clue | Pick |
|---|---|
| Connect components into a chain | **LangChain** |
| State, loops, branches, human approval | **LangGraph** |
| Index and query external data | **LlamaIndex** |
| Optimize prompts against a metric | **DSPy** / `mlflow.genai.optimize_prompts()` |
| Observe and debug | **MLflow Tracing** (observability, **not orchestration**) |

- Integrations: `ChatDatabricks` (chat), `DatabricksEmbeddings` (vectors only), `DatabricksVectorSearch`, `UCFunctionToolkit` (UC functions as tools).

### Prompt Registry
- Variables are `{{question}}` in the registry and `{question}` in LangChain. Convert with `.to_single_brace_format()`.
- Stored in UC as `catalog.schema.prompt_name`. **Versions are immutable; aliases move.**
- `register_prompt`, `set_prompt_alias`, `load_prompt` (by version, `@production` or `@latest`). Loading the alias at request time lets you promote or roll back **without changing code**.
- **Optimization creates a new version but does not deploy it.**

### Prompt augmentation
- Add facts, live data and a glossary. Use **delimiters** between instructions, context and user text. Watch for **lost in the middle**.
- **Access scope comes from the authenticated identity, never from what the user claims in the message.**
- Grounding: answer only from the context, otherwise return a fallback such as `INSUFFICIENT_CONTEXT`.

### Quality and guardrails
- First check the retriever span: if the right info wasn't retrieved, **fix retrieval**. If it was retrieved but misused, fix the prompt or generation.
- **Grounded ≠ authorized.** An answer can quote a document accurately and still leak data the user may not see.
- Over-refusal blocks harmless requests. A false positive flags safe content. A false negative lets unsafe content through.
- **AI Gateway guardrails:** **Safety** (a Llama Guard classifier), **PII Block** (reject), **PII Mask** (mask and continue), **None**.
  - **Input guardrails don't inspect retrieved docs or tool outputs.**
  - **Output guardrails don't run for streaming or for embedding models.**
  - Topic moderation and keyword filters are deprecated; use your own pre- and post-processing.

### Model selection
1. **Hard requirements first** (language, context window, tool calling, structured output, **licence**, region, GA vs preview).
2. **Quality** on a representative eval set.
3. **Cost and latency:** the smallest model that passes.

| Hosting | Use |
|---|---|
| **Pay-per-token** (Foundation Model APIs) | Variable or low traffic, prototyping |
| **Provisioned throughput** | Steady production traffic, **fine-tuned models**, predictable latency |
| **External model** | Third-party provider (data leaves Databricks; keep the key in secrets) |

- A **base** model continues text; an **Instruct** model follows instructions (use it for chat).
- Embeddings: **use the same model for documents and queries. If you change the model, re-embed the whole corpus.** `qwen3-embedding-0-6b` is multilingual with about 32K tokens.
- Licences: Apache 2.0 and MIT are permissive. **CC BY-NC is non-commercial.** A fine-tuned model inherits the base model's licence.
- Metrics: **ROUGE** for summarization, **BLEU** for translation, Exact Match, P/R/F1 for classification, **Perplexity** (lower is better).

### ResponsesAgent and MLflow
- Subclass `mlflow.pyfunc.ResponsesAgent`. Implement `predict` and, optionally, `predict_stream`. Its signature is inferred.
- Stream events: `response.output_text.delta` (same item_id) and then `response.output_item.done`.
- `custom_inputs` are per-request values, `custom_outputs` are extra return values and `model_config` is fixed config.
- **Models from Code:** `agent.py` ends with `mlflow.models.set_model(agent)`, then `log_model(python_model="agent.py")`. `code_paths` holds local modules and `pip_requirements` holds packages. Validate with `mlflow.models.predict`.
- **Auth:**
  - **SystemAuthPolicy / automatic passthrough** = the service principal plus **declared resources**.
  - **OBO / UserAuthPolicy** = the **end user's identity**, so row filters apply. Create the per-user client **inside the request path**.
  - If the resources change, log and deploy a new version.
- Tracing: autolog, `@mlflow.trace`, `mlflow.start_span`, `update_current_trace`.
- **`databricks.agents.deploy(uc_model, version)`** sets up the endpoint, short-lived credentials, the Review App, tracing and inference tables.
  - It does **NOT** create VS indexes or Genie spaces.
  - Deploying a new version under the same name updates the endpoint **in place, with the same URL**.
  - **Moving a UC alias does not redeploy the endpoint.**
- `mlflow.genai.evaluate()` is **offline** testing. **Monitoring** runs scorers on live traces. The **Review App** collects human feedback.

### Genie
- **Genie** = structured tables, NL→SQL, aggregations. A **retriever** = unstructured documents.
- `GenieAgent`: one per Genie space, narrowly scoped. **Wrong SQL → fix the Genie space** (example SQL, metric definitions, column descriptions and synonyms), not the supervisor prompt.
- **Conversation API** is asynchronous: start a conversation, poll until COMPLETED, FAILED or CANCELLED, read the attachment, and send follow-ups in the same conversation.

---

## DOMAIN 4: Assembling & Deploying (22%) ⭐

### Pick the model flavor
| Flavor | When |
|---|---|
| `mlflow.langchain.log_model` | Pure LCEL chain |
| `mlflow.pyfunc.PythonModel` | Your own pre- or post-processing (masking, validation); **you write the signature** |
| `mlflow.pyfunc.ResponsesAgent` | An Agent Framework agent; signature inferred |

- Where the code goes: `__init__` holds picklable settings only. `load_context` builds clients, tokenizers and artifacts once per load. **`predict` runs on every request** (masking, retrieval, LLM call, formatting) for **every caller** (web, Slack, REST and ai_query). **Mask before the endpoint call.**
- LCEL: `prompt | llm | StrOutputParser()`. RAG: `{"context": retriever | format_docs, "question": RunnablePassthrough()}`.
- Models from code: the file is **re-run in a fresh process** and must build everything (a retriever defined in the notebook gives NameError). `ModelConfig(development_config=...)` while iterating, then `log_model(model_config=...)` once per environment.

### Packaging
| Missing | Argument |
|---|---|
| One extra package | `extra_pip_requirements` (**appends**) |
| Full exact environment | `pip_requirements` (**replaces**) |
| Conda YAML | `conda_env` |
| Your own module | `code_paths` |
| A data file | `artifacts`, read via `context.artifacts["name"]` |

- The three environment arguments are **mutually exclusive**. `/Workspace/...` paths aren't readable from the serving container.
- **UC requires a signature.** Logging with `input_example` infers it.
- `mlflow.set_registry_uri("databricks-uc")` (the default in MLflow 3). Model names are `catalog.schema.model`.
- **UC has aliases, not stages.** `set_registered_model_alias(name,"champion",version=6)` and load with `models:/x.y.z@champion`. **Registering a new version never moves an alias.** Tags can't load a model.
- Privileges: `CREATE MODEL` on the schema, `CREATE MODEL VERSION` or ownership to add a version, `EXECUTE` to load. (Table SELECT and MODIFY don't apply to models.)
- Validate with `mlflow.models.predict(uri, input_data, env_manager="uv")`. `env_manager="local"` hides missing dependencies.

### Vector Search
- An **endpoint** is compute: standard or storage-optimized, **up to 50 indexes, billed even when idle**. An **index** is a UC object on exactly one endpoint.

| Index type | Who embeds | Query with |
|---|---|---|
| Delta Sync, Databricks-computed | Databricks (`embedding_source_column` + `embedding_model_endpoint_name`) | `query_text` |
| Delta Sync, self-managed | You (`embedding_vector_column` + `embedding_dimension`) | `query_vector` |
| Direct Vector Access | You, via upsert and delete API calls | `query_vector` |

- Sync: **TRIGGERED** (only on `sync()`, cheapest) or **CONTINUOUS** (within seconds, always-on, standard endpoints only).
- `columns_to_sync` is fixed at creation. A schema or embedding change means **build a new index → switch → delete the old one**. Deleting the old one first causes an outage.
- **Standard:** tens of ms latency, scale with `target_qps`, **HTTP 429 → raise target_qps**. **Storage-optimized:** more than 1B vectors, a few hundred ms, much cheaper, triggered only, embedding dims divisible by 16.
- Filters: standard takes a dict `{"revision >=": 3}`; storage-optimized takes a SQL string `"revision >= 3"`.
- HYBRID uses about 2× resources and returns at most 200 results. SQL `vector_search()` returns at most 100.
- Score = 1/(1+d²) with L2 distance (HNSW). Normalize vectors for cosine ranking.
- **Querying an index needs USE CATALOG + USE SCHEMA + SELECT on the index.** SELECT on the source table doesn't count, and endpoint CAN USE only allows creating indexes.

### Serving and access
- Endpoint permissions: **CAN VIEW < CAN QUERY < CAN MANAGE**. Callers get CAN QUERY. Rollbacks and traffic splits need CAN MANAGE. **UC model privileges don't open the endpoint.**
- Secrets: env var written **exactly** as `{{secrets/<scope>/<key>}}` (without the braces it's plain text). The **creator needs READ on the scope**; callers need only CAN QUERY.
- Agent auth: automatic passthrough, **OBO** or **manual**. Manual is for unsupported resources such as Prompt Registry. Overriding the DATABRICKS_* variables **turns off passthrough for every resource**.
- AI Gateway rate limits: QPM/TPM. **TPM only for foundation or external model endpoints; agents and custom models get QPM only.**
- A **fine-tuned UC model can't use pay-per-token**; use provisioned throughput.
- OpenAI client: `base_url=f"{host}/serving-endpoints"` with `model=` the endpoint name.
- **ai_query batch:** `responseFormat`, `returnType`, `modelParameters`, `failOnError`. Not supported on **Classic** SQL warehouses (use serverless or pro, or **DBR 15.4 LTS+**). Needs CAN QUERY plus SELECT on the input table.

### Memory
| Need | Store |
|---|---|
| Short chat, never resumed | The client resends the messages |
| Resumable sessions, ms I/O | **Lakebase** (managed Postgres), keyed by session |
| Fast lookups of Delta data | Lakebase synced table |
| Analytics | Delta / inference tables |

- Replicas are **stateless**; a Python attribute lives in one replica only. Scope memory by the **authenticated user ID**.

### CI/CD and prompt lifecycle
- **Bundles** (formerly Asset Bundles) define targets for dev, staging and prod. **Deploy code** is the usual choice (each environment builds its own model and index). **Deploy models** only when dev can use prod-equivalent data.
- CI auth: a service principal with **workload identity federation** (OIDC), not PATs or long-lived secrets.
- Tests: **unit** (LLM stubbed, every PR) → **evaluation gate** (before promotion, on a **pinned version number**) → **smoke test** after "Ready" ("Ready" only means the container started) → **monitoring**.
- After writing the table, call `index.sync()` (CDF is a prerequisite, not a trigger).
- Prompt URIs: `/6` = version 6 forever (for tests). `@production` = deployed apps. `@latest` = experiments only.
- **Promotion = move the alias** (`set_prompt_alias`); it never registers anything. Rollback = point the alias back.
- Alias loads are cached with a **60-second TTL**; version loads have no TTL.

### MCP servers
| Type | When |
|---|---|
| **Databricks-managed** | Genie, VS, UC functions and DBSQL already exist; no code |
| **External** | A vendor publishes an MCP server over Streamable HTTP; register it via a UC HTTP connection |
| **Custom** | Nothing exists (for example an internal REST API); host it on **Databricks Apps** |

- A plain REST API can't be registered as an external MCP server. For an OpenAPI spec, use the OpenAPI MCP template.
- Only **per-user OAuth (U2M)** gives per-person identity at the vendor.
- Callers need **EXECUTE** on the MCP plus USE CATALOG and USE SCHEMA. **USE CONNECTION is for authors only.**
- Custom MCP: Streamable HTTP, bind `0.0.0.0` on `DATABRICKS_APP_PORT`, app name starts with `mcp-`, clients use the URL `/mcp`.

### Interfaces
- **Databricks Apps** = production internal chat. **Review App** = expert feedback. **AI Playground** = prototyping. Slack or Teams = users who stay in chat apps.
- For per-user rows, the app needs **user authorization (forward the user's token) plus OBO in the agent**. With one shared identity, everyone sees the same rows.

---

## DOMAIN 5: Governance (8%)

### PII (AI Gateway, configured separately for input and output)
- **Block** rejects. **Mask** replaces and continues. **None** turns detection off.
- **Only input-side Mask stops PII from reaching an external provider.**
- If Block kills the answer rate, use **Mask**. The **Safety** guardrail blocks harm but does **not** mask PII.
- Output guardrails don't run for **streaming or embeddings**. In RAG, the endpoint guardrail runs **after** retrieval and tool calls and doesn't see citations shown directly from Vector Search.

### ai_mask
- `ai_mask(body, array('person','email','phone'))` → `[MASKED]`. Needs DBR 15.4+ and is tuned for English. **Only the labels you list get masked.**
- A UC **column mask** controls who sees a column at query time but can't find names inside free text. **ai_mask** redacts entities inside the text.
- **For RAG: mask once at ingestion → build the index from the masked table.** This adds no model call per request and also covers citations and streaming. If the labels change, rewrite the table and re-sync.

### Prompt injection
- **Direct** (comes from the user message): screen it with a **model-based classifier at the start of the chain**. Phrase lists miss rewordings. Keep roles separate and never concatenate user text into the system prompt.
- **Indirect** (comes from crawled pages, emails or uploads): **screen at ingestion and fetch time**.
- **Blast-radius controls:** least-privilege tool credentials, **OBO**, derive sensitive parameters in tool code, allowlists, human approval, keys in **secrets**.
- Prompt wording, few-shot refusals, low temperature, masking, rate limits and logging do **not** prevent injection.

### Licensing
| Source | Commercial use? |
|---|---|
| No licence | **No** |
| **CC BY-NC** | **No** without separate permission (crediting or masking doesn't fix it) |
| CC BY | Yes, **with attribution** |
| Site terms forbid scraping | Legal risk; buy the licensed feed |
| Retrieval allowed, training not | Retrieval only |

- **Publicly visible ≠ licensed.** Store **provenance and licence per chunk** and **filter on every query**. A UC tag isn't a VS filter. Adding a column means a new index.
- When a licence ends: triggered sync → delete the rows and **sync**. Continuous → delete the rows (CDF carries the change). Direct Access → delete by primary key. A Delta DELETE is logical until **VACUUM**.
- Problematic text: if it's a few records, **filter at ingestion** (for example with ai_classify). If it's pervasive, **replace the source**. An output guardrail alone isn't enough.

---

## DOMAIN 6: Evaluation & Monitoring (12%)

### Choosing a model
- Same dataset, scorers and judge for every candidate. **Eliminate any candidate that misses a hard threshold**, then rank by the stated objective (a gate isn't the objective).
- **MoE:** memory = **total** parameters; speed = **active** parameters.

### mlflow.genai.evaluate(data, predict_fn, scorers)
| Data | predict_fn? |
|---|---|
| inputs (+ expectations) | **Required** (generates fresh outputs) |
| inputs + outputs | **Must NOT be passed** |
| Traces (`search_traces`) | **Must NOT be passed** |

- The keys in `inputs` must match the predict_fn parameters (it's called as `predict_fn(**inputs)`). Use `to_predict_fn("endpoints:/name")`.
- Trace diagnosis: a fact missing from the retriever span → retrieval. Fact present but ignored → generation. Wrong tool → tool selection. Tool error → tool implementation.

### Judges: do they need ground truth?
- **Yes:** Correctness, RetrievalSufficiency, ToolCallCorrectness (and ExpectationsGuidelines needs per-record guidelines).
- **No:** RelevanceToQuery, Safety, Guidelines, RetrievalGroundedness, RetrievalRelevance, ToolCallEfficiency.
- **Guidelines** = global rules. **ExpectationsGuidelines** = per-record rules. Putting per-question rules in global Guidelines causes false failures.

### Custom scorers
| Need | Use |
|---|---|
| Deterministic checks with no model cost (JSON valid, regex, length) | **`@scorer`** (params: inputs, outputs, expectations, trace) |
| Custom scale or semantic judgment | **`make_judge`** (`{{ inputs }}`, `{{ outputs }}`, `{{ trace }}`) |

- An exception in a scorer becomes `Feedback(value=None, error=...)` and evaluation continues. `all()` over zero spans is True, so handle missing spans.
- **SME:** labeling sessions → Review App → `session.sync(to_dataset=...)`. Reviewers need **CAN_QUERY**.
- **`judge.align()`**: the feedback name must **match the judge name**. At least **10** labeled traces (50–100 is better), about 30% negative.

### Monitoring metrics
- Slow before text appears → **TTFT**. A slow full reply → end-to-end latency. Rising spend → tokens per request. Failures → error rates (4XX, **429**, 5XX).

### Which record?
| Record | Use |
|---|---|
| **Inference table** | Request and response payloads (arrives **within about an hour**; may omit oversized payloads and some 401/403/429/500 errors) |
| **system.serving.endpoint_usage** | **Token counts**, no text, `usage_context` |
| **system.billing.usage** | **DBU spend**, custom tags |
| **MLflow traces** | Near-real-time step debugging |

- Join usage and inference data on **request ID**. Group by `served_entity_id` to compare traffic splits.
- **`usage_context`** attributes tokens per tenant but **does not change the identity** used for rate limits. For per-tenant limits, use **a separate service principal per tenant**.

### agents.deploy() and monitoring
- Sets up tracing, inference tables and the Review App. Does **NOT** start scorers, set rate limits or enable scale to zero.
- **Agent Monitoring:** `ScorerSamplingConfig(sample_rate, filter_string)`, `update()`, `stop()`, `backfill_scorers` (last 7 days). First scores take about 15–20 min. Monitoring scorers are defined in a notebook with imports **inside** the function.

### Rate limits and cost
- Exceeding a limit returns **429**. **Budgets alert; rate limits enforce.**
- A user-specific limit beats a group limit, and custom limits override the default. With several groups, the most generous applies. The endpoint limit caps everyone.
- To cap an agent's tokens, set **TPM on the underlying model endpoint**.
- Cost levers: scale to zero (idle), pay-per-token (sporadic), provisioned throughput (steady), ai_query batch (offline), max_tokens, rate limits.

---

## ✅ Gaps from the official sample questions (exam guide, live since 18 Mar 2026)
**Format:** 45 scored questions (multiple choice or **multiple selection**), 90 minutes, online proctored.

- **Too many vectors for the store?** Use **bigger chunks** and **less overlap**. Both cut the record count.
- **Choosing an embedding context length:** pick the **smallest model whose context length is at least your chunk size** when cost and latency matter. For example, 512-token chunks go with a 512-length model that has 384 dims, not a 32K-length model.
- **Choosing an LLM by task:** evaluate in the task category that matches the output. A paragraph cut down to one-sentence gist is **Summarization**. Read the model card for task, licence, context length and size.
- **Exact per-transaction data** such as a delivery date belongs in a **structured lookup keyed by ID** (a feature or online table, or a UC function). A vector store or fine-tuning is the wrong fix.
- **Scanned images to text** with the least code: use **pytesseract**.
- **Vector Search when latency is critical:** about 100M items at 80 QPS fits a **standard** endpoint. When latency is the top constraint, keep **hybrid search and reranking OFF**, because both add latency. Get accuracy from a fine-tuned embedding model instead (upfront cost is acceptable).
- **Prompt promotion with rollback:** use **MLflow Prompt Registry versions plus aliases**, not git branches or overwritten files.
- **Apps with corporate identity and per-user access:** the **App backend** calls the endpoint and uses the authenticated user context. Never put a PAT or API key in the browser.
- **MCP:** a managed server exists, so **use the managed one**. A vendor MCP server needs a key, so use **external MCP with the key in Databricks Secrets**. Don't build a custom wrapper.
- **SME ratings disagree:** write **clear rubrics, calibrate the SMEs**, then use the aligned judgments in `mlflow.genai.evaluate()`. Don't average the scores, drop the disputed cases or trust the LLM judge over the experts.

---

## 🔥 Top 30 Last-Minute Traps
1. Pick the task from the **output**; don't add unrequested tasks.
2. Exact values → **extraction**, never a summary.
3. `ai_similarity` returns a **number**; `ai_classify` returns a **label**; `ai_mask` keeps the text.
4. Named, fine-tuned or custom-schema endpoint → **ai_query**.
5. `json_schema` gives **shape, not truth**; temperature isn't a format control.
6. Possibly missing field → **nullable** plus "return null".
7. Wrong tool chosen → **fix the tool descriptions**.
8. Security goes in the **tool** (preconditions, least privilege, idempotency), not the prompt.
9. Tenant access → a **metadata filter in retrieval**; a missing column means **rebuilding the index**.
10. Fact absent from every chunk → **ingest it** (no retrieval tweak helps).
11. Text longer than the embedding limit is **silently truncated** (bge 512, gte 8192).
12. A reranker **only reorders**; a missing candidate needs better first-stage retrieval.
13. Exact codes or IDs missed → **HYBRID** search.
14. Recall@k = retrieved relevant ÷ **all** relevant; precision = ÷ **k**.
15. Sufficiency needs labels; Groundedness and Relevance don't.
16. LangGraph = state and loops; MLflow Tracing = observability, not orchestration.
17. Prompt versions are immutable and **aliases move**; optimizing ≠ deploying.
18. **Grounded ≠ authorized.**
19. Input guardrails don't check retrieved docs; **output guardrails skip streaming**.
20. Same embedding model for docs and queries; change it → **re-embed**.
21. `agents.deploy` doesn't create indexes or Genie spaces, doesn't start scorers, and moving an alias doesn't redeploy.
22. **OBO** = per-user permissions; **SystemAuthPolicy** = service principal.
23. UC has **aliases, not stages**; a new version doesn't move an alias.
24. Querying an index needs **SELECT on the index**, not on the source table.
25. Secrets must be written exactly as `{{secrets/scope/key}}`.
26. A fine-tuned model needs **provisioned throughput**, not pay-per-token.
27. Only **input Mask** keeps PII away from external providers.
28. CC BY-NC = **no commercial use**; public ≠ licensed.
29. With outputs or traces supplied → **no predict_fn**.
30. `usage_context` attributes cost but doesn't change identity; **budgets alert, rate limits enforce**.

Good luck tomorrow 💪
