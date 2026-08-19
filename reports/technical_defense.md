# Technical Defense

1. Coreference is conservative: a pronoun is rewritten only when an antecedent is explicit in the same chunk. Ambiguous mentions remain unchanged and are logged as `unresolved_mentions`, preventing unsupported edges.
2. Entity matching uses manual aliases first, then normalized MiniLM vectors with a 0.90 cosine threshold. The live audit contains 128 candidate decisions. For example, `Dell Technologies Inc.` vs `Dell Technologies` scored 0.827 and was conservatively rejected by the threshold; this improves precision but shows a recall trade-off that should be handled through a reviewed alias map.
3. Union-Find makes transitive merges explicit and deterministic; the most frequent surface form becomes canonical. Every decision is retained in `entity_resolution_audit_df`.
4. The graph schema uses `Entity` plus `Company`, `Person`, and `Technology` labels. Relation types are allowlisted before ingestion, which prevents arbitrary model output from changing the schema.
5. Bulk ingestion uses `UNWIND $rows` in batches of 1,000. This reduces round trips and keeps transaction size bounded compared with one Cypher query per triple.
6. Every edge stores `source_chunk_id`, `published_date`, `evidence`, and `confidence`. The live graph contains 53 edges and the provenance sanity query returned zero invalid edges.
7. Super-nodes are detected at degree >100 and limited to 50 newest edges. In the live sample the top degrees were ServiceNow=9, Microsoft=8 and NVIDIA=4, so the cap was not activated; `GLOBAL_EDGE_CAP=250` still bounds worst-case context growth.
8. Flat RAG is a FAISS inner-product index over normalized embeddings. GraphRAG combines seed matching, bounded BFS, provenance-rich graph text, and vector chunks, so it can answer relational questions across documents.
9. LLM-as-a-Judge scored all 50 questions. GraphRAG averaged 3.68 comprehensiveness, 3.94 faithfulness and 3.68 multi-hop reasoning versus 3.46, 3.66 and 3.46 for Flat RAG. Checkpoint CSV writing kept the run resumable.
10. At larger scale, extraction/API latency and entity-resolution indexing become bottlenecks first. The appropriate response is asynchronous batching, persistent ANN/HNSW indexes, partitioned graph ingestion, and cached embeddings rather than quadratic all-pairs comparison.
