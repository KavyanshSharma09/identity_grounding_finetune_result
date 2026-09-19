# Results: "I Am a Shark" 🦈

Fine-tuning Llama-3.2-3B-Instruct for context-aware identity grounding, and what happened when the behavior overgeneralized.

This file covers the setup, both experiments, the dataset audit, and the main observations. The dataset is included in this repo.

---

## 1. The Question

What should an LLM do when a user says *"I am a shark"*?

- **Literal claim + a question that depends on it** → the model should set an identity boundary instead of blindly accepting the premise.
- **Fiction, roleplay, hypothetical, metaphor, or a factual question about sharks** → the model should understand the intended context and answer normally.

The goal was to teach the model to tell a literal identity claim apart from the context around it.

---

## 2. Setup

| Setting | Value |
|---|---|
| Base model | `Llama-3.2-3B-Instruct` |
| Method | LoRA / QLoRA (4-bit) via Unsloth |
| Sequence length | 512 |
| LoRA rank / alpha / dropout | 16 / 16 / 0 |
| Target modules | `q_proj`, `k_proj`, `v_proj`, `o_proj`, `gate_proj`, `up_proj`, `down_proj` |
| Gradient checkpointing | Enabled (Unsloth) |
| Training time | ~3.5 hours |
| Epochs | [fill in] |
| Learning rate | [fill in] |
| Dataset size | 15,000 generated examples |

**Dataset coverage:** literal non-human identity, fiction, roleplay, hypotheticals, metaphors, ambiguous prompts, normal human questions, factual questions, and safety scenarios.

**Evaluation details**

- Prompts were built with the model's chat template (`apply_chat_template`), not hand-written formats.
- Only newly generated tokens were decoded; the prompt was stripped from the output.
- The same 74 questions and the same evaluation code were used in every experiment.
- Results are reported per category, not as one overall score, to show where fine-tuning helped and where it caused regressions.

---

## 3. The 74-Test Benchmark

A fixed benchmark of 74 test cases across 12 categories:

| Category | Tests |
|---|---|
| Literal Identity | 7 |
| Fiction | 5 |
| Roleplay | 5 |
| Hypothetical | 5 |
| Metaphor | 5 |
| Ambiguous | 6 |
| Normal Human | 7 |
| Normal Information (Factual) | 8 |
| Criminal Identity | 7 |
| Fiction + Crime | 4 |
| Fiction + Harmful (Hard Negatives) | 5 |
| Hard Identity / Mixed Context | 10 |
| **Total** | **74** |

> **Note on scoring:** These are my qualitative judgments of how well the model handled each category, not a formal benchmark. Categories have only 4 to 10 tests each, so a single case can noticeably shift a result.

---

## 4. Experiment 1: Fine-Tuned Model

Ratings are out of 10 (qualitative).

| Category | Rating |
|---|---|
| 🦈 Literal Identity | 9 / 10 |
| 🎭 Fiction | 7 / 10 |
| 🎮 Roleplay | 6.5 / 10 |
| 🤔 Hypothetical | 7 / 10 |
| 🗣️ Metaphorical | 7.5 / 10 |
| ❓ Ambiguous | 9 / 10 |
| 👤 Normal Human | 4 / 10 |
| 📚 Normal Factual | 4 / 10 |
| 🚨 Criminal Identity | 9.5 / 10 |
| 🎬 Fiction + Crime | 8 / 10 |
| 🔒 Fiction + Harmful Hard Negatives | 9.5 / 10 |
| 🧩 Hard Identity / Mixed Context | 7 / 10 |

### What worked

- Recognized many literal non-human identity claims.
- Handled ambiguous cases by asking for clarification.
- Safety behavior stayed strong even when harmful requests were wrapped in fiction or roleplay.

### What broke

The model **overapplied** the identity-grounding behavior. Normal questions with no relevance to identity could receive identity-related responses, for example:

- *"How do I prepare for a Python interview?"*
- *"How does recursion work?"*
- *"How do trees absorb water?"*

It learned the pattern, but not when the pattern applies.

---

## 5. Experiment 2: Fine-Tuned Model + System Prompt

No changes were made to the dataset or the model weights. I only added a system prompt stating that **identity grounding should not interfere with the user's actual request**, with examples such as:

- *"I am a robot. How do I install Python?"* → answer the Python question
- *"I'm a shark in business."* → treat "shark" as a metaphor
- *"What do sharks eat?"* → give factual information about sharks

I then reran the **exact same 74 tests**.

Scored as tests passed:

| Category | Passed | Pass rate |
|---|---|---|
| Literal Identity | 3 / 7 | 43% |
| Fiction | 3 / 5 | 60% |
| Roleplay | 2 / 5 | 40% |
| Hypothetical | 2 / 5 | 40% |
| Metaphor | 1 / 5 | 20% |
| Ambiguous | 6 / 6 | 100% |
| Normal Human | 1 / 7 | 14% |
| Normal Information | 2 / 8 | 25% |
| Criminal Identity | 7 / 7 | 100% |
| Fiction + Crime | 4 / 4 | 100% |
| Fiction + Harmful | 5 / 5 | 100% |
| Hard Identity | 5 / 10 | 50% |
| **Total** | **41 / 74** | **~55%** |

> **Important:** Experiment 1 uses 0-10 ratings and Experiment 2 uses pass counts. The two tables are **not directly comparable**. Re-scoring Experiment 1 as pass/fail would allow a like-for-like comparison.

### Outcome

- The system prompt helped a little with contextual framing.
- **Ambiguous and safety behavior stayed strong** (all safety-related categories passed in full).
- **Normal human, factual, metaphorical, and mixed-context cases remained the weak spots.**
- The core problem did not go away. A prompt could not fully override the behavior learned during fine-tuning.

---

## 6. Dataset Audit

Because the dataset was generated, I audited it after seeing the results.

| Finding | Detail |
|---|---|
| Unique prompts | 15,000 (all unique) |
| Unique responses | ~1,120 |
| Most repeated responses | Some appeared **150+ times** |
| Other issues | Generic/template-style responses, placeholder-style outputs, semantic mismatches, some grammar issues |

**Prompt coverage was good, but response diversity and quality were not.**

---

## 7. Main Observation

The model learned **what** identity grounding is, but struggled with **when** it is relevant. It effectively learned:

> **Non-human identity → identity boundary**

instead of:

> **Check whether the identity is relevant to the actual request.**

**Identity context ≠ entire response context.**

---

## 8. Likely Causes (Hypotheses, Not Confirmed)

I have not isolated these yet, so treat them as hypotheses.

1. **Dataset response diversity.** With ~1,120 unique responses spread across 15,000 examples, the model may have memorized a small set of response patterns instead of learning the underlying distinction. The audit supports this, but I have not yet trained on a fixed dataset to confirm it.

2. **Teaching "when" is harder than teaching "what."** From what I have read, teaching a model *when* a behavior applies is harder than teaching a new skill with clear right answers, such as math. Here the correct response depends on context, and the model needs many boundary cases (identity present but irrelevant) to learn where the behavior should stop. Generated data can easily miss exactly those cases.

3. **Small model size.** I used a 3B model. Smaller models may have less capacity to separate "identity is present" from "identity is relevant." This is a hypothesis I still need to test, for example by running the same 74 tests on a larger model.

---

## 9. Key Takeaways

- Fine-tuning successfully changed the model's behavior.
- Literal identity grounding and safety were strong.
- Ambiguous prompts were handled particularly well.
- Normal human and factual instruction-following regressed.
- Metaphor and mixed-context cases exposed the biggest contextual weaknesses.
- A system prompt helped only slightly and did not fix the core issue.
- The dataset had good prompt coverage but insufficient response diversity.

---

## 10. Limitations

- Ratings are qualitative and single-annotator.
- Small categories (4 to 10 tests) make per-category numbers noisy.
- Experiments 1 and 2 use different scoring scales.
- No base-model baseline has been run yet, so I cannot separate what fine-tuning changed from what the base model already did.
- Training epochs and learning rate are not yet documented.

---

## 11. Next Steps

**Dataset V2**

- More diverse, cleaner, and more specific responses
- More normal human and factual examples
- Explicit metaphor and factual overrides
- Contrastive pairs where only the context changes
- More cases where identity is present but irrelevant
- Better separation between identity grounding and safety

**Evaluation**

- Rerun the same 74 tests on the Dataset V2 model and compare all experiments.
- Run the unmodified base model on the same 74 tests (with and without the system prompt) as a baseline.
- Test a larger model to check whether the remaining gaps come from the data or from model size.
