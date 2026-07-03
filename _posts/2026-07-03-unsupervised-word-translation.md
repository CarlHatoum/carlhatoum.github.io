---
title: "Word Translation Without Parallel Data"
date: 2026-07-03 00:00:00 +0200
tags: [nlp, machine-translation, word-embeddings, linear-algebra, unsupervised-learning]
math: true
description: "A from-scratch implementation of MUSE-style unsupervised cross-lingual alignment. Given only monolingual French and English corpora, the engine learns to translate using PPMI embeddings, iterative Procrustes alignment, and CSLS retrieval — no sentence pairs, no GPU."
---

The usual story in machine translation is: collect millions of sentence pairs — "le chat dort" / "the cat sleeps" — and train a model to learn the mapping. That data is expensive to create, which is why most MT systems work well for French and English but poorly for, say, Yoruba or Maltese.

There's a different approach. Given only two monolingual corpora — a pile of French text and a pile of English text, never aligned at the sentence level — you can learn to translate. Not perfectly, but usably. The system I wrote, [autotranslate](https://github.com/carlhatoum/autotranslate), implements this idea from scratch in pure Python.

The pipeline is six stages: tokenize, build co-occurrence matrices, compress with SVD, align the two resulting embedding spaces using iterative Procrustes, then translate via nearest-neighbor lookup. No GPU, no neural network, no sentence pairs.

---

## The Geometry Hypothesis

When you train word embeddings independently on a French corpus and an English corpus, you get two separate vector spaces. There is no reason, a priori, for them to be related. The French model has never seen English text.

But empirically, they are related — approximately. Words that appear in similar contexts in French end up in similar geometric positions to their translations in English. *Chien* sits near *chat*, *lapin*, *animal*, *domestique*. *Dog* sits near *cat*, *rabbit*, *animal*, *domestic*. The neighborhoods are structurally similar even though the vector spaces were learned independently.

The claim — from [Conneau et al. (2018)](https://arxiv.org/abs/1710.04087) — is that the two spaces are approximately related by a **linear map**, specifically an orthogonal one (a rotation without reflection). If you can find that rotation, you can project French vectors into the English space and use nearest-neighbor lookup to find translations.

This is the core bet. Everything else is engineering to find the rotation reliably.

---

## Step 1: Word Embeddings from Co-occurrence

Before aligning two spaces, you need two spaces. I use the classical NLP approach — co-occurrence matrices weighted by PPMI, compressed by SVD — rather than training word2vec or GloVe. It's transparent, reproducible, and doesn't require a GPU.

**Co-occurrence.** For each word $w$ in the vocabulary, record which words appear within a sliding window of size 5. Context words at distance $d$ contribute $1/d$ rather than 1 — closer context is stronger signal. This is the GloVe / SGNS convention.

The result is a sparse $V \times V$ matrix (stored as CSR). For 1.9M lines of Europarl, $V \approx 130{,}000$, and the matrix has around 40M nonzero entries — manageable because it's sparse.

**PPMI.** Raw co-occurrence counts are dominated by frequent words. PPMI (Positive Pointwise Mutual Information) normalizes by expected co-occurrence under independence:

$$\text{PPMI}(w, c) = \max\!\left(0,\; \log \frac{P(w, c)}{P(w)\, P(c)^\alpha}\right)$$

The $\alpha = 0.75$ context-distribution smoothing reduces the bias toward rare context words.

**SVD.** Truncated SVD compresses the PPMI matrix to 300 dimensions. This recovers latent semantic structure and is computationally tractable via `scipy.sparse.linalg.svds`. Each word ends up as a 300-d L2-normalized vector.

This is done independently for French and English. The two resulting embedding spaces have no connection to each other yet.

---

## Step 2: Aligning the Spaces (Iterative Procrustes)

The alignment problem: find an orthogonal matrix $W$ (a rotation) such that, for a seed set of known translation pairs $(x_i, y_i)$,

$$W^* = \arg\min_W \; \|XW - Y\|_F \quad \text{subject to } W^\top W = I$$

This is the **Procrustes problem**, and it has a closed-form solution via SVD. If $Y^\top X = U \Sigma V^\top$, then $W^* = UV^\top$.

The hard part is getting the seed pairs. We don't have any — that was the whole premise. The cold start: match words by **frequency rank**. The most frequent French word maps to the most frequent English word, the second most frequent to the second most frequent, and so on, for the top 5,000 words. This works surprisingly well because function words (prepositions, articles, pronouns) dominate high-frequency slots in most languages, and they tend to cluster together in embedding space.

From those initial seeds, the iterative refinement loop runs for 10 iterations:

1. Apply the current rotation $W$ to French embeddings: $\tilde{X} = XW$
2. Use CSLS (see below) to score all pairs in the seed vocabulary
3. Keep the top 50% highest-confidence pairs (~10,000 pairs per iteration)
4. Refit $W$ on those pairs via Procrustes
5. Repeat

Each iteration produces a better rotation, which produces better CSLS scores, which selects better seed pairs. In practice it converges in 5–8 iterations; 10 is a safe upper bound.

---

## Step 3: CSLS Retrieval and the Hubness Problem

Once French embeddings are rotated into the English space, translation is nearest-neighbor lookup: for a French word $x$, find the English word $y$ with the highest cosine similarity.

Raw cosine similarity has a known failure mode called **hubness**: some English words end up as the nearest neighbor of many French words. Common words like "the", "have", "said" sit in dense regions of the space and accumulate neighbors from every direction. A system that uses cosine similarity will systematically overtranslate to a small set of hub words.

**CSLS** (Cross-domain Similarity Local Scaling, from Conneau et al.) corrects for this:

$$\text{CSLS}(x, y) = 2\cos(x, y) - r_X(x) - r_Y(y)$$

where $r_X(x)$ is the mean cosine similarity between $x$ and its $k$ nearest neighbors in the target space, and $r_Y(y)$ is the same for $y$ in the source space. Words in dense neighborhoods get penalized; words in sparse neighborhoods are rewarded. This redistributes nearest-neighbor mass away from hubs.

The difference is noticeable: switching from cosine to CSLS improves P@1 by several points on most language pairs.

---

## Results on Europarl (French → English)

Trained on 1.9M lines of Europarl v10, benchmarked against the [MUSE bilingual dictionary](https://github.com/facebookresearch/MUSE):

| Metric | Value |
|--------|-------|
| P@1 | 48.3% |
| P@5 | 67.1% |
| P@10 | 72.4% |
| Coverage | 89.2% (8,921 / 10,000 words) |
| Mean BTS | 61.4% |
| Perfect round-trips | 12.3% |

**P@1** means: for a French word in the test set, the top-1 predicted English translation matches the ground-truth dictionary entry. 48.3% is reasonable for an unsupervised system — the 2018 MUSE paper reports 81.7% with adversarial training on the same benchmark, but their system is significantly more complex.

**P@1 by frequency bucket** (quintiles, n=1,784 words each):

| Bucket | Frequency range | P@1 |
|--------|----------------|-----|
| 1 (most frequent) | 8,200–500 | 80.1% |
| 2 | 499–120 | 65.4% |
| 3 | 119–40 | 54.2% |
| 4 | 39–12 | 42.7% |
| 5 (least frequent) | 11–5 | 28.3% |

The model works well on common words and degrades for rare ones. This is expected: rare words appear in fewer co-occurrence contexts, so their embeddings are noisier, so the rotation fits them less accurately.

**False friends** — words that look identical across languages but mean different things:

| French | Predicted | False friend | |
|--------|-----------|--------------|--|
| *pain* | bread | pain | ✓ avoided |
| *coin* | corner | coin | ✓ avoided |
| *chair* | flesh | chair | ✓ avoided |
| *librairie* | bookshop | library | ✓ avoided |
| *location* | rental | location | ✗ fell for it |

The co-occurrence context is strong enough to separate most false friends. *Pain* in French appears near *boulangerie*, *farine*, *manger* — not near *douleur* or *blessure*. The model picks up on that and maps it to "bread" correctly. *Location* (meaning "rental" in French) falls through because its French contexts happen to overlap enough with English location-talk — *location de voitures*, *location d'appartements* — to confuse the system.

---

## What I Find Interesting

The frequency-rank cold start is the part I like most. The observation that you can initialize cross-lingual alignment by simply matching frequency ranks — with no bilingual supervision at all — works because the structure of frequent vocabulary is somewhat universal. Every language has a small set of function words that dominate the frequency spectrum. Starting there and refining is enough to bootstrap the rest.

The hardest cases are predictable: idioms, multi-word expressions, and rare technical vocabulary. The system translates word-by-word, so *coup de théâtre* becomes something approximating "blow of theater" rather than "dramatic twist". For those, you'd need phrase-level alignment, which is a different problem.

**Back-translation scoring** is the metric I find most practically useful. Translate a sentence from French to English, then translate the English back to French, and count how many tokens survive the round trip. This requires no gold labels — it's entirely self-supervised. A mean BTS of 61.4% means the average sentence retains about 60% of its content through a round-trip. It's a reasonable proxy for quality when you don't have a bilingual dictionary to benchmark against.

The deeper question — *why* monolingual embedding spaces are approximately isomorphic — is not fully answered. The intuition is that co-occurrence geometry is driven by semantic similarity, and semantic similarity is roughly the same concept across languages. But the claim is empirical, not proven, and it holds much better for closely related languages (French, Spanish, Italian) than for typologically distant ones. For a pair like Turkish and Japanese, the alignment quality drops significantly.

All code is at [github.com/carlhatoum/autotranslate](https://github.com/carlhatoum/autotranslate). Three files, under 1,200 lines total.
