# Reflection and Action Plan

| Lecture concept | Notebook implementation | Observation |
|---|---|---|
| Conservative coreference | `resolve_coref_batch` | Ambiguity is preserved and logged instead of hallucinated. |
| Schema allowlist | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Invalid model output is discarded before persistence. |
| Bulk Cypher | `bulk_insert_nodes`, `bulk_insert_edges` | `UNWIND` bounds network overhead. |
| Entity resolution | `build_resolution_map`, `UF` | Vector candidates still require lexical approval. |
| Super-node cap | `retrieve_graph_context` | Context growth is bounded and observable. |
| LLM judge | `judge_answer` and checkpoint | Quality, latency, and token cost are recorded together. |

The hardest debugging issue was keeping Colab paths, local paths, and secret loading consistent. I fixed it by resolving paths from `Path.cwd()`, preserving Colab secret lookup, and never writing credentials into the notebook. The next project application is an internal engineering-news assistant: graph entities are companies, products, people, and technologies; relations are ownership, partnership, investment, and employment. I would use GraphRAG for cross-document lineage questions and Flat/Hybrid RAG for simple summaries.
