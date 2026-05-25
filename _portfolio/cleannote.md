---
title: "CleanNote — Medical Note Structuring Pipeline"
excerpt: "NLP pipeline that transforms raw clinical notes into structured JSON documents (symptoms, conclusions, treatments) using a local LLM. Published at PFIA 2025."
collection: portfolio
---

[CleanNote](https://github.com/CarlHatoum/CleanNote) analyzes raw medical notes and transforms them into concise, structured documents focused on symptoms, medical conclusions, and treatments — enabling easier clinical analysis and research.

The project was developed during a research internship at the **Hubert Curien Laboratory** and is published as an open-source Python package on PyPI. I contributed as domain expert and research supervisor, guiding the design of the NLP pipeline and solution architecture.

**How it works:**
- Loads clinical notes from HuggingFace datasets
- Prompts a local LLM (Mistral-7B-Instruct) to extract structured fields in JSON
- Outputs per-note statistics, visualizations, and an Excel export

```python
pip install -U cleanote
```

**Tech stack:** Python, Mistral-7B-Instruct-v0.3, HuggingFace Transformers, PyPI

**Publication:** *Identification de profils patients à partir de notes cliniques non structurées* — Laval et al., PFIA 2025

[GitHub](https://github.com/CarlHatoum/CleanNote) · [PyPI](https://pypi.org/project/cleanote/)
