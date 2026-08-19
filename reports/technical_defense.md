# Technical Defense

1. Coreference is conservative: a pronoun is rewritten only when an antecedent is explicit in the same chunk. Ambiguous mentions remain unchanged and are logged as `unresolved_mentions`, preventing unsupported edges.
2. Entity matching uses manual aliases first, then normalized MiniLM vectors with a 0.90 cosine threshold. The lexical guard rejects high-similarity pairs when names are not suffix-compatible, such as `Apple` and `Apple Music`.
3. Union-Find makes transitive merges explicit and deterministic; the most frequent surface form becomes canonical. Every decision is retained in `entity_resolution_audit_df`.
4. The graph schema uses `Entity` plus `Company`, `Person`, and `Technology` labels. Relation types are allowlisted before ingestion, which prevents arbitrary model output from changing the schema.
5. Bulk ingestion uses `UNWIND $rows` in batches of 1,000. This reduces round trips and keeps transaction size bounded compared with one Cypher query per triple.
6. Every edge stores `source_chunk_id`, `published_date`, `evidence`, and `confidence`. The provenance sanity query must return zero invalid edges before evaluation.
7. Super-nodes are detected at degree >100 and limited to 50 newest edges. `GLOBAL_EDGE_CAP=250` bounds worst-case context growth; the trade-off is that old or low-frequency edges may be omitted.
8. Flat RAG is a FAISS inner-product index over normalized embeddings. GraphRAG combines seed matching, bounded BFS, provenance-rich graph text, and vector chunks, so it can answer relational questions across documents.
9. LLM-as-a-Judge scores comprehensiveness, faithfulness, and multi-hop reasoning from 1 to 5. Checkpoint CSV writing makes the evaluation resumable after rate limits or transient failures.
10. At larger scale, extraction/API latency and entity-resolution indexing become bottlenecks first. The appropriate response is asynchronous batching, persistent ANN/HNSW indexes, partitioned graph ingestion, and cached embeddings rather than quadratic all-pairs comparison.
