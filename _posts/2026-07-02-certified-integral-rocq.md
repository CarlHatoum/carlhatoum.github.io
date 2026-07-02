---
title: "An LLM, Rocq, and an Integral Mathematica Gets Wrong"
date: 2026-07-02 00:00:00 +0200
tags: [formal-methods, rocq, numerical-analysis, llm, proof-assistant, mathematics]
math: true
description: "An LLM proposes proof steps, a retrieval engine finds relevant lemmas, and Rocq's kernel decides what counts as valid. Notes from Théo Stoskopf's workshop at the AI, Proof and Formalization Days — using LLM-assisted formal verification on an integral that floating-point tools consistently get wrong."
---

The setup: an LLM proposes proof steps, a semantic retrieval engine finds relevant lemmas from a formal library, and Rocq's kernel decides what is and isn't a valid proof. The LLM is not trusted — it is just a fast source of candidate scripts. Rocq is the only authority.

This post is a write-up of a workshop I attended at the **[AI, Proof and Formalization Days](https://www.ens-lyon.fr/evenement/recherche/ia-and-mathematics-days-ai-proof-and-formalization-days-workshop-mathematical)** (Journées IA, Démonstration et Formalisation), designed and led by [Théo Stoskopf](https://theostoskopf.github.io/). All formal ideas and workshop materials are his — I am only writing up what I found most interesting about working through them.

The vehicle is a specific integral that Mathematica gets wrong — not by a rounding error, but by about 0.5%, consistently, for a reason that is easy to miss and interesting to think about. The goal is to certify it in [Rocq](https://rocq-prover.org/) (formerly Coq) and then prove its analytic closed form, with the LLM doing most of the proof assembly and Rocq checking every accepted step.

---

## The Integral

$$f(x) = \operatorname{sech}(10x - 2)^2 + \operatorname{sech}(100x - 40)^4 + \operatorname{sech}(1000x - 600)^6$$

integrated over $[0, 1]$. Mathematica returns about 0.2097. Sage returns the same, or sometimes NaN. Neither is right.

The function has three bumps — one near $x = 0.2$, another near $x = 0.4$, a third near $x = 0.6$. Their widths are proportional to the scaling factor: roughly 0.1, 0.01, and 0.001. Standard adaptive quadrature scans the interval, finds the first two bumps without difficulty, and walks right past the third. The third peak is 0.001 units wide — narrower than the step size that triggers further refinement. Its contribution to the integral is about 0.001, which is the entire size of the error.

The result is a plausible-looking number that is wrong in the third significant digit, with nothing in the output to suggest anything went wrong.

---

## Certified Bounds via Rocq

[Rocq](https://rocq-prover.org/) is a proof assistant: a system where every proof is checked by a small, trusted kernel. With the [Interval](https://coq-interval.gforge.inria.fr/) library, it can produce rigorous decimal enclosures — not floating-point estimates, but theorems that guarantee the integral lies within a specific range, with no appeal to floating-point arithmetic in the proof itself.

After setting up the definitions in [Coquelicot](https://coquelicot.saclay.inria.fr/) (a Rocq library for real analysis):

```coq
Definition sech (u : R) : R := 2 * exp u / (exp (2 * u) + 1).

Definition f (x : R) : R :=
    (sech (10 * x - 2))^2
  + (sech (100 * x - 40))^4
  + (sech (1000 * x - 600))^6.

Definition I : R := RInt f 0 1.
```

The `Interval` tactic produces:

```
0.21078887 ≤ I ≤ 0.21081871
```

The Mathematica value 0.2097 does not lie in this interval. The third peak exists, Rocq has verified it contributes to the integral, and the contribution is real.

This can be packaged as a named theorem:

```coq
Theorem I_first_4_decimal_digits : Rabs (I - 0.2108) <= 1e-4.
Proof.
  unfold I, f, sech.
  integral with (i_prec 25, i_degree 3, i_fuel 300).
Qed.
```

Three tactics. The kernel checks it.

One practical note: the `Interval` computation is expensive enough that it runs in a separate Rocq document from the analytic work. If you replay or reset a proof while experimenting, you don't want to re-run the numerical certification every time. Keeping the two documents independent is the right engineering decision here.

---

## The Analytic Solution

The certified enclosure answers the numerical question but not the mathematical one. The integral has a closed form, and finding it requires recognizing that the integrand is a sum of exact powers of sech.

The relevant antiderivative formulas — standard reduction identities — are:

$$\int \operatorname{sech}^2(u)\, du = \tanh(u)$$

$$\int \operatorname{sech}^4(u)\, du = \tanh(u) - \tfrac{1}{3}\tanh^3(u)$$

$$\int \operatorname{sech}^6(u)\, du = \tanh(u) - \tfrac{2}{3}\tanh^3(u) + \tfrac{1}{5}\tanh^5(u)$$

Rescaling by the chain rule (the affine argument contributes a factor of 1/10, 1/100, 1/1000), the closed form is $F(1) - F(0)$ where:

$$F(x) = \frac{A_2(10x - 2)}{10} + \frac{A_4(100x - 40)}{100} + \frac{A_6(1000x - 600)}{1000}$$

with $A_2$, $A_4$, $A_6$ the tanh polynomials above. In Python:

```python
import math

def A2(u): return math.tanh(u)
def A4(u): t = math.tanh(u); return t - (1/3)*t**3
def A6(u): t = math.tanh(u); return t - (2/3)*t**3 + (1/5)*t**5

def F(x):
    return A2(10*x - 2)/10 + A4(100*x - 40)/100 + A6(1000*x - 600)/1000

F(1) - F(0)  # ≈ 0.2108027355005493
```

This is a stable computation. No adaptive sampling, no narrow peaks to miss — just tanh evaluated at the endpoints. And 0.2108... lies inside the Rocq-certified interval.

---

## Proving the Antiderivative Formally

This is where the LLM does its work. Each lemma below was assembled by the model from the current proof state and a small set of retrieved library facts — Rocq's kernel accepted or rejected every step.

The structure is: proving that $F$ is an antiderivative of $f$, then applying the fundamental theorem of calculus.

The proof has a predictable structure:

1. `F2_derivative`: the derivative of $F_2$ is $\operatorname{sech}(10x - 2)^2$
2. `F4_derivative` and `F6_derivative`: analogous
3. `F_derivative`: derivative of $F = F_2 + F_4 + F_6$ is $f$ (by `is_derive_plus`)
4. `f_continuous`: $f$ is continuous (required by the FTC)
5. `I_closed_form_correct`: $I = F(1) - F(0)$ (by `is_RInt_derive` + `is_RInt_unique`)

![The Retrieve → LLM → Rocq pipeline, with a reject → refine feedback loop back to retrieval](/assets/img/posts/rocq-workflow.png)

Each derivative lemma follows the same template. The `auto_derive` tactic (from Coquelicot) handles differentiability and leaves behind equational side conditions. These require `exp_plus` to rewrite doubled arguments (Rocq's `exp (2*u)` needs to become `exp(u) * exp(u)` to match the sech definition), then `field` and `nra` for the arithmetic. The only recurring annoyance is the denominator: you need to prove `exp(u) + 1 ≠ 0` in several places. The right move is to prove it once as a reusable lemma:

```coq
Lemma sech_denominator_nonzero (u : R) : exp u + 1 <> 0.
Proof.
  apply Rgt_not_eq. apply Rplus_lt_0_compat.
  - apply exp_pos.
  - lra.
Qed.
```

Three lines. Then every derivative proof can reference it directly.

---

## The LLM Proof Workflow

The proofs above were not written by hand. The LLM assembled them — tactic by tactic, lemma by lemma — from a description of the proof goal and a small set of retrieved library facts. The interesting question is not whether it can do this at all, but *how* you structure the interaction to make it reliable.

![Proof dependency tree: I_closed_form_correct at the root, F_derivative and f_continuous as its two obligations, the three derivative lemmas below, and sech_denominator_nonzero / exp_plus / auto_derive at the leaves](/assets/img/posts/rocq-proof-tree.png)

The workflow has three stages:

**Retrieve.** A semantic search engine indexes Rocq library items as dense vectors (using [Qwen3-Embedding-4B](https://huggingface.co/Qwen/Qwen3-Embedding-4B) over the [Pile-of-Rocq](https://github.com/LLM4Coq/pile-of-rocq) dataset). A natural-language query — "derivative of a function at a point" — returns `is_derive` by cosine similarity, not keyword match. This matters because Rocq library naming is inconsistent: looking for an "exponential sum rule" by name would not obviously surface `exp_plus`.

**Propose.** The LLM receives the current proof state, the retrieved lemma set, and optional mathematical guidance. It proposes a proof script.

**Check.** Rocq's kernel accepts or rejects. The LLM is untrusted; the kernel is the only authority.

Three strategies were compared on the $F_2$ derivative:

- **Strategy A — direct.** One call: the LLM proposes a complete proof script without any Rocq feedback. Cheapest when it works; brittle when the algebra is non-trivial.
- **Strategy B — feedback.** If the first script fails, the LLM receives the error message and retries. Error messages are more informative than proof states: "field: not a field equation" tells you more about what's missing than the raw goal does.
- **Strategy C — agentic.** The LLM can call `run_tac` to test individual tactics, see the updated goal state, and call `reverse` to backtrack. It commits to nothing until it has verified the step in Rocq. Most reliable; most expensive.

The observation worth noting: Strategy C outperforms A not because the LLM is better in agentic mode, but because it gets to see what Rocq actually requires. Rocq's error signal is more actionable than the proof state seen upfront. This is the same reason why feedback loops improve RAG systems — the retriever doesn't know what will be useful until generation has started.

The agentic setup runs on [petanque](https://github.com/ejgallego/coq-lsp/tree/main/petanque) and [pytanque](https://github.com/ejgallego/pytanque) via the [rocq-ml-server](https://github.com/LLM4Coq/rocq-ml-toolbox). The proof workflow is part of ongoing work on Crrrocq and Goal2Tacq (with Guillaume Baudart, Marc Lelarge, and Jules Viennot).

---

## What I Find Interesting

The numerical part is a clean lesson: floating-point integration is not certified computation. The Interval tactic closes that gap, at the cost of computation time and the need to formalize the problem definition. Worth it when correctness matters — in verification, in aerospace, in any domain where "it looks right" is not enough.

The analytic proof is mostly mechanical algebra — derivative lemmas, chain rule, fundamental theorem. That is exactly the kind of work an LLM is good at: repetitive, structured, prone to small errors that a checker can catch. The formal proof context forces precision that informal mathematics doesn't require: not "F is an antiderivative of f" but `is_derive F x (f x)`, every denominator side condition named and proved. The LLM fills in that detail; Rocq enforces it.

The part that connects most directly to my own work is the retriever. A semantic search engine over formal mathematics is doing exactly what a retrieval-augmented system does over text documents: convert a description into a vector, find nearby objects in the library, surface them as context. The difference is that "nearby" here means "formally related to the proof goal," and the generation step is checked by a kernel rather than evaluated by a human. That's a materially stronger notion of correctness than anything available in text retrieval — but the retrieval problem is structurally the same.

The formal proof of the closed form is a theorem. The numerical estimate from Mathematica is not. Sometimes that distinction matters.
