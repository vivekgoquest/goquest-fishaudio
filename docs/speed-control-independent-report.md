# Independent Verification Report: Fish Audio S2-Pro Duration Control

Date: 2026-03-17
Status: Pruned and corrected

## Purpose

This report keeps only the conclusions that are supported by the local Fish S2 codebase, the bundled Fish Audio S2 technical report, and official upstream references used during review. Incorrect branches, unsupported comparisons, and speculative implementation claims have been removed.

## Scope Reviewed

Local files reviewed:

- `speed-control-research.md`
- `fish-speech/fish_speech/models/text2semantic/llama.py`
- `fish-speech/fish_speech/models/text2semantic/inference.py`
- `fish-speech/fish_speech/models/text2semantic/lit_module.py`
- `fish-speech/fish_speech/models/text2semantic/lora.py`
- `fish-speech/fish_speech/datasets/semantic.py`
- `fish-speech/fish_speech/models/dac/modded_dac.py`
- `fish-speech/fish_speech/models/dac/rvq.py`
- `fish-speech/fish_speech/utils/schema.py`
- `fish-speech/README.md`
- `fish-speech/FishAudioS2TecReport.pdf`

Official references checked:

- `https://huggingface.co/fishaudio/s2-pro/resolve/main/config.json`
- `https://github.com/fishaudio/fish-speech`

## High-Confidence Findings

## 1. There is no native duration-control surface in the local Fish S2 stack

Verified:

- no target-duration or speed field in the TTS request schema
- no duration argument in `generate_long()`
- no duration or speed embedding in the local Dual-AR implementation
- no exact-length correction stage in local inference

What the current system provides is indirect control only:

- prompt wording
- reference conditioning
- sampling controls
- `max_new_tokens`

## 2. The effective code-frame rate is about 21.53 Hz

This is directly supported by the local DAC implementation and the bundled technical report.

Relevant facts:

- `hop_length = prod([2, 4, 8, 8]) = 512`
- `frame_length = hop_length * 4 = 2048`
- `encode()` computes output lengths as `ceil(audio_lengths / frame_length)`
- sample rate is `44100`

Therefore:

- `44100 / 2048 = 21.533203125`
- one frame is about `46.44 ms`

The earlier 86 Hz branch has been removed because it does not match the local codec output-length logic.

## 3. Semantic-token or frame-count control alone is too coarse for exact lip-sync

Because the effective timing lattice is about 46.44 ms:

- segment-level duration steering is plausible
- exact landing at frame level is plausible
- tight consonant or viseme timing is not solved by semantic-token count alone

This is the main constraint that should shape any recommendation.

## 4. `max_new_tokens` is only a cap

The local generation loop stops on:

- `im_end`, or
- `max_new_tokens`

So `max_new_tokens` does not create natural duration control. It only bounds output length.

## 5. Free-form prompt instructions are a real baseline worth testing

The local Fish S2 docs explicitly support free-form inline natural-language instructions. That means pacing-related prompt wording is a valid baseline experiment.

What is supported:

- prompt-based steering may influence delivery pace

What is not supported here:

- deterministic timing claims
- exact quantitative effect without runtime testing

## 6. Model-side conditioning is feasible but not yet justified as the first move

Adding a separate conditioning path is architecturally possible.

However, the local training setup does not by itself prove that a duration label derived from observed token count will yield controllable timing. The current training path teaches next-token prediction on observed realizations, not alternate valid timings for the same utterance.

So the following claim has been removed:

- that standard AR fine-tuning on observed token-count bins is likely sufficient by itself

That is not proven by the codebase.

## 7. Any model experiment would need more wiring than the earlier note implied

If duration conditioning is tested, it must be threaded through:

- training
- generation
- fine-tuning parameter selection

The existing LoRA helper only marks LoRA parameters trainable, so any newly added embedding would need explicit handling.

## Recommended Direction

## 1. Use a production-first baseline before retraining

Best current recommendation:

- generate multiple candidates
- include pacing-oriented prompt variants
- measure duration
- rank outputs by duration plus quality
- apply a small exact-length correction only when needed

This recommendation survives pruning because it fits the verified constraints and does not depend on unproven model retraining assumptions.

## 2. Treat exact timing as coarse model steering plus final correction

Given the 21.53 Hz lattice, exact segment landing should be approached as:

- coarse timing control from generation
- exact final landing from a small correction stage if necessary

That is the most defensible product strategy from the evidence currently available.

## 3. If model work is later needed, favor remaining-budget conditioning over one-shot conditioning

This remains a recommendation, not a verified result, but it is the better-grounded research direction:

- timing is sequential
- remaining budget is the directly relevant signal

So if model work is pursued later, remaining-budget or countdown conditioning is the cleaner starting point than a single first-token duration embedding.

## What Was Removed

The following content was intentionally removed from the earlier analysis set:

- the unresolved frame-rate framing
- the 86 Hz precision branch
- unsupported IndexTTS transfer assumptions
- unverified claims about fixed speed-tag inventories
- exact training-cost estimates
- specific LoRA hyperparameter recommendations
- statements implying that architecture inspection alone proved training success

## Runtime Limitation

Live Torch runtime checks were blocked in this sandbox by `OMP Error #179` shared-memory failure, and no local Fish S2 checkpoints or audio test assets were present.

So this report keeps only:

- code-level findings
- document-level findings
- recommendations that remain valid under those limits

and removes behavior claims that would require runtime evidence.
