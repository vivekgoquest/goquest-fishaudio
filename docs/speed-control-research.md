# Speed and Duration Control for Fish Audio S2-Pro

Date: 2026-03-17
Status: Pruned and corrected

## Objective

Evaluate what is actually true about duration control in the local Fish Audio S2-Pro codebase, and identify which solution paths are grounded enough to pursue for dubbing.

## Verified Facts

## 1. Fish S2-Pro does not currently expose native duration control

Verified in the local repo:

- No target-duration or speed field in the public TTS request schema.
- No duration or speed argument in the local long-form generation path.
- No duration or speed embedding in the local Dual-AR model implementation.
- No exact-length post-processing stage in the local inference code.

What exists today:

- `max_new_tokens`
- sampling controls such as `temperature`, `top_p`, and `top_k`
- prompt/reference conditioning

That means current duration control is indirect only.

## 2. The effective codec timing lattice is about 21.53 Hz

This is resolved from code, not open.

Verified in the local DAC implementation:

- `encoder_rates = [2, 4, 8, 8]`, so `hop_length = 512`
- `frame_length = hop_length * 4 = 2048`
- `encode()` returns `indices_lens = ceil(audio_lengths / frame_length)`
- sample rate is `44100`

So:

- `44100 / 2048 = 21.533203125` frames per second
- one frame is about `46.44 ms`

The bundled Fish Audio S2 technical report matches this and describes a total downsampling ratio of 2048 with a frame rate of about 21 Hz.

## 3. Duration control at the semantic/code-frame level is therefore coarse

Implication of the verified frame rate:

- best-case raw timing granularity is about 46.44 ms per frame

That is:

- useful for segment-level duration steering
- borderline for tight lip-sync
- not sufficient on its own for precise viseme- or phoneme-level timing

So token-count control can still help, but it should be treated as coarse timing control rather than exact sync control.

## 4. `max_new_tokens` is a hard cap, not a duration controller

The local generation path ends when:

- the model emits `im_end`, or
- `max_new_tokens` is exhausted

This means `max_new_tokens` can prevent runaway generation, but it cannot ensure a natural completion at a target duration.

## 5. Fish S2 supports free-form inline prompt instructions, but that is not deterministic timing control

The local README says Fish S2 accepts free-form natural-language inline descriptions such as:

- `[whisper in small voice]`
- `[professional broadcast tone]`
- `[pitch up]`

That supports testing pace-related prompt wording as a baseline.

However:

- this is still prompt steering, not a duration API
- exact effect size is not verified here
- prompt control should be treated as heuristic, not deterministic timing control

## 6. Model-side duration conditioning is architecturally feasible

The local Dual-AR structure makes it possible to add a separate conditioning path, for example:

- a duration embedding
- a remaining-budget embedding
- another auxiliary conditioning feature

This should be done as a separate embedding or hidden-state conditioning path, not by extending the output vocabulary.

That said, architectural feasibility does not prove that the model will learn useful controllability.

## 7. Existing training data does not automatically create duration controllability

The current text-to-semantic training path pairs each utterance with its observed acoustic realization and trains standard next-token prediction.

That means:

- each sample comes with its natural timing
- the model is not explicitly taught to produce multiple valid timings for the same linguistic content

So simply attaching a duration label derived from observed token count is not enough to prove the model will learn strong controllable pacing.

## 8. Any model-side experiment needs both training-path and inference-path changes

It is not enough to patch only generation-time code.

If duration conditioning is tested, it must be wired through:

- the training `forward()` path
- the generation `forward_generate()` path
- the fine-tuning setup, including trainable parameter handling

The local LoRA helper only marks LoRA parameters trainable, so a newly added embedding would need explicit handling.

## Recommendations

## 1. Start with a practical non-training baseline

Recommended first path for dubbing:

- generate multiple candidates
- vary prompt pacing language and sampling settings
- measure actual duration
- rank by duration closeness plus quality
- apply a small final time correction only when needed

Reason:

- it works with the current model
- it is immediately testable
- it respects the coarse 21.53 Hz timing lattice
- it is lower risk than retraining

## 2. Treat exact landing as a two-stage problem

Given the 46.44 ms frame lattice, exact segment landing should be treated as:

- coarse model-side timing guidance
- plus a small exact-length correction stage if required

For production dubbing, this is more realistic than expecting semantic-token control alone to solve exact sync.

## 3. Only pursue model-side duration conditioning if the baseline is not good enough

If model work is needed, the cleaner research target is:

- remaining-budget or countdown conditioning

not just:

- a single first-token duration embedding

Timing is a sequential planning problem, so conditioning that tracks remaining budget is a better fit than a one-time global signal.

## 4. Evaluate on a real dubbing benchmark before scaling up model work

Before large training effort, define a benchmark with:

- target duration
- transcript fidelity
- speaker similarity
- naturalness
- acceptable sync thresholds

Without this, model-side work is likely to optimize the wrong objective.

## Removed From Earlier Drafts

The following were removed because they were incorrect, unsupported, or too speculative for this document:

- the unresolved 86 Hz vs 21.5 Hz framing
- claims that depended on the 86 Hz possibility
- unverified IndexTTS mechanism assumptions
- unsupported claims about official fixed speed-tag sets
- specific LoRA hyperparameter recommendations not grounded in this repo
- cost and timeline estimates for training
- claims that architecture review alone was sufficient and no more code analysis was needed

## Execution Limit

Live Torch runtime validation could not be performed in this sandbox because importing Torch aborts with `OMP Error #179` shared-memory failure, and no local model checkpoints or audio assets were present in this workspace.

That does not affect the code-level findings above, but it does mean prompt-behavior and runtime-quality claims remain to be tested in a proper runtime environment.
