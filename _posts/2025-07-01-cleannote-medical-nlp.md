---
title: "CleanNote: Structuring Raw Clinical Notes with a Local LLM"
date: 2025-07-01
permalink: /posts/2025/07/cleannote-medical-nlp/
tags:
  - NLP
  - healthcare
  - LLM
  - information extraction
  - open source
---

Clinical notes are among the richest — and messiest — sources of medical information. A physician writes what they observe, conclude, and prescribe, but in free text: abbreviations, fragments, mixed languages, no fixed structure. This makes clinical notes extremely hard to use at scale for research or analysis.

This post is about [CleanNote](https://github.com/CarlHatoum/CleanNote), a project I helped supervise during a research internship at the **Hubert Curien Laboratory**. The goal was simple to state and hard to execute: take a raw clinical note and turn it into a clean, structured document. The result was presented at **PFIA 2025** and published as an open-source Python package.

---

## The Problem

Medical NLP has a core tension: the data that matters most — what a doctor actually wrote at the bedside — is the hardest to process automatically.

Traditional approaches (regex, rule-based extractors, NER models) work well when notes follow predictable patterns. They fall apart on real clinical corpora, where one physician writes "pt c/o SOB, dx COPD exacerbation, started prednisone" and another writes three paragraphs about the same situation.

The question driving CleanNote was: *can a local, open-weight LLM reliably extract structured clinical information from arbitrary free-text notes — without sending data to an external API?*

The "local" constraint matters. In healthcare, data is sensitive. Running everything on-premise or in a controlled environment isn't optional — it's a requirement.

---

## The Approach

The architecture is a **prompt-based extraction pipeline** built around [Mistral-7B-Instruct-v0.3](https://huggingface.co/mistralai/Mistral-7B-Instruct-v0.3), a strong open-weight instruction-tuned model that can run locally via HuggingFace Transformers.

For each clinical note, the pipeline asks the model to return a single valid JSON object with exactly four keys:

```json
{
  "Symptoms": ["..."],
  "MedicalConclusion": ["..."],
  "Treatments": ["..."],
  "Summary": "..."
}
```

The summary field has a hard constraint: it must only mention items already listed in the other three fields, with no invented facts. This is a simple but effective guardrail against hallucination — the model is forced to stay grounded in what it extracted, not what it inferred.

---

## Implementation

The pipeline is built around three clean abstractions:

**`Dataset`** — wraps a HuggingFace dataset and handles loading, splitting, and field selection.

**`Model`** — wraps a HuggingFace pipeline with full control over generation parameters (temperature, top_p, max_new_tokens). We used `temperature=0.0` and `do_sample=False` to make extraction as deterministic as possible.

**`Pipeline`** — ties them together, applies the model to each note, collects outputs, and exposes export methods.

```python
from cleanote.pipeline import Pipeline
from cleanote.dataset import Dataset
from cleanote.model import Model

data = Dataset(
    name="AGBonnet/Augmented-clinical-notes",
    split="train",
    field="full_note",
    limit=20
)

model = Model(
    name="mistralai/Mistral-7B-Instruct-v0.3",
    task="text-generation",
    prompt=prompt,
    max_new_tokens=1024,
    do_sample=False,
    temperature=0.0,
    top_p=1.0
)

pipe = Pipeline(dataset=data, model_h=model)
out = pipe.apply()
pipe.save_all_stats_images(limit=20)
xls = pipe.to_excel()
```

The Excel export and stats visualizations make it straightforward to review outputs at scale — something clinicians and researchers actually need.

---

## The Hard Parts

**Prompt engineering for JSON compliance.** Getting a 7B model to consistently return valid JSON without trailing commas, without wrapping it in markdown fences, and with exactly the right keys — took significant iteration. Smaller models frequently break the schema. The final prompt is explicit and defensive: it names every key, specifies the format of each value, and repeats the "no inventing" constraint for the Summary.

**The frugality constraint.** One of the supervisors, Mme Fresse, pushed hard on running the smallest model that gets the job done. There's a real temptation in NLP to reach for the biggest model available. In a healthcare context with on-premise deployment, a 7B model that fits in GPU memory and produces good results is worth far more than a 70B model that requires dedicated infrastructure. Mistral-7B hits a good point on that curve.

**Hallucination in the Summary.** Even with the constraint in the prompt, the model occasionally introduced facts in the Summary that weren't in the extracted lists. The solution was two-part: make the constraint more explicit in the prompt, and add a post-processing check that flags summaries containing terms not present in any of the three lists.

---

## What I Learned

My role was as research supervisor on the NLP side — advising on model selection, prompt design, and evaluation methodology. A few things stood out:

**Structured output extraction is a solved problem — barely.** Mistral-7B does it well enough for a research prototype. For production at scale, you'd want a model fine-tuned on medical extraction, or a framework like Instructor/Outlines that enforces schema compliance at the decoding level, not just through prompt instructions.

**The real value is downstream.** Structuring notes is only useful if something happens next — patient clustering, treatment pattern analysis, adverse event detection. CleanNote was scoped as a preprocessing step, and that framing is right. The paper (presented at PFIA 2025 as *"Identification de profils patients à partir de notes cliniques non structurées"*) explores what you can do once the notes are structured.

**Local LLMs are production-ready for constrained tasks.** This was 2024. Mistral-7B running locally, on a reasonable GPU, producing structured clinical extractions at a quality level that would have required GPT-4 a year earlier. The pace of progress on open-weight models is genuinely fast.

---

## Try It

CleanNote is open source (MIT) and installable from PyPI:

```bash
pip install -U cleanote
```

The code is on [GitHub](https://github.com/CarlHatoum/CleanNote). The dataset used for demonstration — [AGBonnet/Augmented-clinical-notes](https://huggingface.co/datasets/AGBonnet/Augmented-clinical-notes) — is freely available on HuggingFace.

If you're working on clinical NLP and want to discuss the pipeline or the research direction, feel free to reach out.
