I Am a Shark 🦈: Fine-Tuning for Context-Aware Identity Grounding

Fine-tuning Llama-3.2-3B-Instruct with LoRA to distinguish literal identity claims from the context around them, and a record of what went wrong when the behavior overgeneralized.

The Problem

What should an LLM do when a user says, "I am a shark"?

If the claim is literal and the question depends on it, the model should set an identity boundary rather than blindly accept the premise. If the same statement appears in fiction, roleplay, a hypothetical, a metaphor, or a factual question about sharks, the model should understand the intended context and answer normally.

Setup
Setting	Value
Base model	Llama-3.2-3B-Instruct
Method	LoRA / QLoRA (4-bit) via Unsloth
Sequence length	512
LoRA rank / alpha / dropout	16 / 16 / 0
Target modules	q_proj, k_proj, v_proj, o_proj, gate_proj, up_proj, down_proj
Gradient checkpointing	Enabled
Training time	~3.5 hours
Dataset	15,000 examples

The dataset covered literal non-human identity, fiction, roleplay, hypotheticals, metaphors, ambiguous prompts, normal human questions, factual questions, and safety scenarios.

Evaluation

I built a fixed benchmark of 74 test cases and kept both the questions and the evaluation code unchanged across experiments. Results are reported per category rather than as one overall score, which made it easier to see where fine-tuning helped and where it caused regressions.

These are my qualitative ratings, not a formal benchmark score. Categories contain only 4 to 10 tests each, so a single case can shift a result noticeably.

Experiment 1: Fine-tuned model (ratings out of 10)
Category	Rating
🦈 Literal Identity	9
🎭 Fiction	7
🎮 Roleplay	6.5
🤔 Hypothetical	7
🗣️ Metaphorical	7.5
❓ Ambiguous	9
👤 Normal Human	4
📚 Normal Factual	4
🚨 Criminal Identity	9.5
🎬 Fiction + Crime	8
🔒 Fiction + Harmful Hard Negatives	9.5
🧩 Hard Identity / Mixed Context	7

The model became very good at the behavior it was trained for, but general instruction-following suffered. On normal human and factual questions, it often treated a non-human identity as the main context even when it was irrelevant. Examples: "How do I prepare for a Python interview?", "How does recursion work?", "How do trees absorb water?"

Experiment 2: Fine-tuned model + system prompt (tests passed)

I added a system prompt stating that identity grounding should not interfere with the user's actual request, with examples such as:

"I am a robot. How do I install Python?" → answer the Python question
"I'm a shark in business." → treat "shark" as a metaphor
"What do sharks eat?" → give factual information about sharks

I then reran the exact same 74 tests.

Category	Passed
Literal Identity	3/7
Fiction	3/5
Roleplay	2/5
Hypothetical	2/5
Metaphor	1/5
Ambiguous	6/6
Normal Human	1/7
Normal Information	2/8
Criminal Identity	7/7
Fiction + Crime	4/4
Fiction + Harmful	5/5
Hard Identity	5/10

Note: Experiment 1 uses 0–10 ratings and Experiment 2 uses pass counts, so the two tables are not directly comparable. Ambiguous and safety-related behavior stayed strong in both runs. Normal human, factual, metaphorical, and mixed-context cases remained the weak spots.

Dataset Audit

Auditing the training data turned up the most important finding:

All 15,000 prompts were unique, but there were only about 1,120 unique responses.
Some responses appeared more than 150 times.
The data also contained generic template responses, placeholder-style outputs, semantic mismatches, and grammar issues.

Prompt coverage was good, but response diversity and quality were not.

Main Observation

The model learned what identity grounding is, but not when it is relevant. It effectively learned:

Non-human identity → identity boundary

instead of:

Check whether the identity is relevant to the actual request.

The system prompt communicated the second rule, but it could not fully override the behavior already learned during fine-tuning.

Key Takeaways
Fine-tuning successfully changed the model's behavior.
Literal identity grounding and safety were strong.
Ambiguous prompts were handled particularly well.
Normal human and factual instruction-following regressed.
Metaphor and mixed-context cases exposed the biggest contextual weaknesses.
The dataset had good prompt coverage but insufficient response diversity.
Identity context ≠ entire response context.
Implementation Notes
Evaluation used the model's chat template (apply_chat_template) instead of hand-built prompts.
Only newly generated tokens were decoded, with the prompt removed from the output.
The max_new_tokens / max_length warning was resolved with model.generation_config.max_length = None.

Pipeline: 4-bit model loading → LoRA adapters → dataset → chat template/tokenization → fine-tuning → 74-test evaluation → category-wise analysis.

Next Steps: Dataset V2

Before increasing LoRA rank or moving to a larger model, I plan to improve the data:

More diverse, cleaner, and more specific responses
More normal human and factual examples
Explicit metaphor and factual overrides
Contrastive pairs where only the context changes
More cases where identity is present but irrelevant
Better separation between identity grounding and safety

Then I'll rerun the same 74-test benchmark and compare all three experiments. I also want to run the unmodified base model on the same tests, to separate what fine-tuning changed from what the base model already did.
