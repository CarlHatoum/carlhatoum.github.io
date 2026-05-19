---
layout: archive
title: "CV"
permalink: /cv/
author_profile: true
redirect_from:
  - /resume
---

{% include base_path %}

Education
======
* **PhD in Machine Learning, NLP & Information Retrieval** *(in progress)*
  * Research focus: Semantic search, retrieval-augmented generation, dense information retrieval
* **Master's degree** — Data Science / AI *(details to add)*
* **Bachelor's degree** — Computer Science / Engineering *(details to add)*

Work Experience
======
* **AI & Data Scientist** — SEGULA Technologies, France *(current)*
  * Applying machine learning and NLP to engineering and industrial use cases
  * Building and deploying RAG-based systems for document intelligence
  * Research and development in semantic search and information retrieval

Skills
======
* **Machine Learning & AI**
  * Deep Learning, NLP, Information Retrieval
  * Semantic Search, RAG pipelines, Multimodal AI
* **Frameworks & Libraries**
  * PyTorch, Hugging Face Transformers, LangChain, Sentence-BERT
  * RAGAS, FAISS, Qdrant, Pinecone, Chroma
* **Programming Languages**
  * Python, MATLAB, R, JavaScript
* **Other**
  * Vector databases, embedding models, LLM integration

Publications
======
  <ul>{% for post in site.publications %}
    {% include archive-single-cv.html %}
  {% endfor %}</ul>

Talks & Courses
======
  <ul>{% for post in site.talks %}
    {% include archive-single-talk-cv.html %}
  {% endfor %}</ul>

Teaching
======
  <ul>{% for post in site.teaching %}
    {% include archive-single-cv.html %}
  {% endfor %}</ul>
