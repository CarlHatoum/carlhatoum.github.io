---
title: "Semantic Search & RAG"
collection: teaching
type: "Graduate course"
permalink: /teaching/2026-semantic-search-rag
venue: "Données Spécifiques"
date: 2026-01-01
location: "France"
---

A graduate-level course covering the full stack of modern semantic search and Retrieval-Augmented Generation (RAG), from foundational concepts to production-grade multimodal systems.

## Course Overview

The course is organized into three pillars:

**Pillar 1 — Semantic Search**
- From lexical to semantic retrieval: TF-IDF, BM25, and dense embeddings
- Word embeddings: Word2Vec, GloVe, fastText
- Contextual embeddings: BERT and Sentence-BERT (bi-encoder architecture)
- Vector databases: Chroma, FAISS, Qdrant, Pinecone, Weaviate
- Approximate Nearest Neighbor search: HNSW, IVF, Product Quantization
- Evaluation with the MTEB benchmark

**Pillar 2 — Retrieval-Augmented Generation**
- LLM limitations: hallucinations, knowledge cutoff, private data
- RAG architecture: offline ingestion (parsing, chunking, embedding, indexing) and online retrieval
- Hybrid retrieval: BM25 + dense (Reciprocal Rank Fusion)
- Cross-encoder reranking for precision
- Prompt engineering for faithful, cited generation
- Evaluation with the RAGAS framework (Faithfulness, Answer Relevance, Context Recall)

**Pillar 3 — Multimodal & Advanced RAG**
- CLIP: joint text–image embedding space
- Multimodal document understanding (images, tables, figures)
- GraphRAG, Agentic RAG, VideoRAG
- Industrial case studies: healthcare, finance, legal, education
