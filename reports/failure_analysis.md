# Failure Analysis

## Case 1: Flat RAG misses a multi-hop answer

Vector similarity can retrieve the chunk describing an investment but not the separate chunk describing who founded the recipient. The root cause is lexical distance between the two facts. Graph traversal joins the explicit `INVESTED_IN` and `FOUNDED` edges, while provenance keeps each supporting chunk auditable.

## Case 2: GraphRAG loses recall through extraction or seed failure

If NER/RE omits one relation, or seed extraction fails to identify the company alias, BFS starts from the wrong node and returns an empty or misleading subgraph. The mitigation is strict extraction validation, alias plus fuzzy fallback, vector context in the hybrid answer, and self-correction that expands from hop 2 to hop 3 before falling back to vector retrieval.

## Case 3: Super-node truncation

A highly connected company can dominate the context and consume the token budget. The degree threshold and 50-edge temporal cap prevent explosion, but historical questions may lose older evidence. A production system should make the temporal window query-aware and expose truncation diagnostics.
