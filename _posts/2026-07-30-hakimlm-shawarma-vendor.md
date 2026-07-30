---
title: "Teaching a Shawarma Vendor to Talk: Building HakimLM"
date: 2026-07-30 00:00:00 +0200
tags: [nlp, llm, transformers, synthetic-data, open-source]
image:
  path: /assets/img/posts/hakimlm-logo.png
  alt: "HakimLM logo: a half-human, half-robot shawarma vendor holding a wrap"
description: "How an 8.7M-parameter transformer, 57,000 fake conversations, and one stubborn bug in a random.choice() call became HakimLM — a language model that talks like a philosophical shawarma vendor."
---

![HakimLM logo: a half-human, half-robot shawarma vendor holding a wrap](/assets/img/posts/hakimlm-logo.png){: width="300" .normal }

There's a small project called [GuppyLM](https://github.com/arman-bd/guppylm) built on one premise: training your own language model from scratch isn't magic. No PhD, no GPU cluster — a vanilla transformer with about 8.7 million parameters, trained on a single T4 GPU in roughly five minutes, that talks like a small fish named Guppy. It's deliberately unglamorous: six layers, 384 hidden dimensions, no RoPE, no GQA, no SwiGLU. The entire point is that you can see every piece of the machine, from raw text to trained weights to generated tokens.

I liked the architecture. I didn't want a fish. I wanted Hakim — a shawarma vendor who's been at his stand long enough that the knife knows his hand, who answers "what is the meaning of life" with an actual answer instead of a deflection, and who never once breaks character to explain himself. Same brain, completely different soul.

```
you> are you happy
hakim> some days yes. some days i just wrap and breathe. both are fine.

you> i am in a hurry
hakim> you are in a hurry. the shawarma is not. one of you must wait.
```

---

## How it's built

**The brain.** HakimLM keeps GuppyLM's architecture untouched — the personality lives entirely in the training data, not the model code.

| Parameter | Value |
|---|---|
| Parameters | ~8.7M |
| Layers | 6 |
| Hidden dim | 384 |
| Attention heads | 6 |
| FFN width | 768 (ReLU) |
| Vocab (BPE) | 4,096 |
| Context length | 128 tokens |
| LM head | tied to embeddings |

**The voice.** Hakim's data is synthetic — 38 topics (bread, meat, sauce, spice, fire, knife, patience, wisdom, customers, loneliness, and so on), each built from a small bank of slot-fillable phrases. A template like `"i season it with {spice} and {spice}. patience is the main spice."` gets expanded across random combinations of ingredients, times of day, weather, and customers, so the same handful of topics produce tens of thousands of non-identical lines. Cycling every input/output pair through 38 topics at 1,500 samples each gives ~57,000 conversations. Everything stays single-turn — Hakim's 128-token window means multi-turn chat degrades fast, so instead of fighting the context limit, the character just embraces having a short memory.

From there it's the standard small-LLM pipeline: train a byte-level BPE tokenizer on the generated text, format every example as `<|im_start|>user...<|im_end|><|im_start|>assistant...<|im_end|>`, and train with AdamW and a cosine learning-rate schedule for 10,000 steps — about twenty minutes on a single machine, no cluster required.

---

## Lessons

**A "drop-in" file is only as good as what it's dropped into.** The plan, per the README, was simple: swap GuppyLM's data generator for Hakim's, point the data-prep script at the new file, done. What actually happened was quieter and worse — the swap left the tokenizer-training step reading a field that didn't exist. The new generator wrote flat `{"input", "output", "category"}` records; the prep script expected a ChatML-formatted `"text"` field inside `train.jsonl`/`eval.jsonl`. Nothing crashed loudly — it would have thrown a plain `KeyError` the moment someone actually ran it, which is its own lesson: two READMEs agreeing on an instruction doesn't mean the code underneath was ever actually run end to end with that instruction followed.

**Talking to it casually isn't the same as testing it.** The first trained checkpoint sounded great in a casual back-and-forth. It was only once I ran all twenty held-out eval prompts back to back that a pattern showed up:

```
hakim> the the beef did the work. i just did not ruin it.
hakim> i rest a. a few hours. enough.
hakim> i have worked n many times. the meat does not know.
```

Root cause: two separate template bugs, both invisible until enough samples surfaced them. Ingredient lists like `MEATS` already included their own article ("the beef"), so any template that wrote a literal `"the "` in front of it doubled up. Worse: a few templates called `random.choice()` on an already-indexed single string — `r(TIMES[1])` instead of just `TIMES[1]` — which doesn't pick a list element, it picks one random character out of the string `"at night"`. That's where "i rest a." came from.

23 templates out of roughly 200 were affected — about 15% of the held-out eval set showed the damage. None of it was visible from a few manual chat turns; it only became obvious once the model was checked systematically against a fixed set of prompts, which made a small, disposable batch-eval script (loop the 20 held-out cases through the model, print the reply next to what it should sound like) worth more than any amount of casual chatting.

**Retraining is cheap, so "good enough" isn't an excuse.** Because a full 10,000-step run takes about twenty minutes on ordinary hardware, there was no real cost to fixing the templates, regenerating all 57,000 samples, retraining the tokenizer, and retraining the model from scratch before publishing anything. At this scale, the honest move is always to fix the data and retrain — "we'll catch it next time" doesn't apply when "next time" is one coffee break away.

**A model and its dataset aren't the same kind of thing.** HuggingFace treats models and datasets as distinct repo types — there's no single repo that's both. The actual mechanism for "publish these together" is a Collection: one shareable page that bundles a model, a dataset, and (if you have one) a demo Space, without merging their storage. That's the right unit for a project like this — the weights and the data that shaped them, presented as one artifact instead of two unrelated links.

---

## Everything, linked

- The pack: [huggingface.co/collections/hatoum/hakimlm](https://huggingface.co/collections/hatoum/hakimlm-6a6b25447d39f4249e98afaa)
- Model: [hatoum/hakimlm-9m](https://huggingface.co/hatoum/hakimlm-9m)
- Dataset: [hatoum/hakimlm-57k-shawarma](https://huggingface.co/datasets/hatoum/hakimlm-57k-shawarma)
- Base architecture: [github.com/arman-bd/guppylm](https://github.com/arman-bd/guppylm)
