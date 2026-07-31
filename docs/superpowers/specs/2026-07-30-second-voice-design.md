# Second Voice for Vāgdhenu — Voice Packs Design

**Date:** 2026-07-30 · **Status:** Approved (brainstorm session)
**Goal:** Add a second reciter's voice to Vāgdhenu as a selectable option, with the
architecture generalized so further voices can be added the same way. The chant
behavior (meter-driven references, sandhi frontend, pacing) carries over unchanged.

## Decision

**Approach A — voice packs.** One fine-tuned checkpoint + one reference bank per
voice, selected at render time. Rejected alternatives: a single multi-reciter model
(risks regressing the shipped MOS-4.6 voice; possible later experiment) and
zero-shot reference swap (below the fidelity the available studio data can achieve).

Inputs available for the new voice: **hours of studio recordings, metrical ślokas
across multiple meter families, transcribed per verse, plus prose (gadya).**

## 1. Voice pack layout

```
voices/
  <voice_id>/                 # "pilot" (current voice), "reciter2", …
    voice.json                # display name, checkpoint filename, notes, consent record ref
    bank.json                 # per-meter references in THIS voice (same schema as today)
    *.wav                     # reference + prime clips
models/
  <voice_id>/model.pt         # fine-tuned checkpoint (gitignored; hosted on HF)
```

- The current voice becomes pack `pilot`; its bank moves (or is symlinked) from
  `src/reference_bank/`. `vocab.txt` stays shared — tokenizer vocab, not voice data.
- `VAGDHENU_VOICE` env and a `--voice` CLI flag select a pack. Default `pilot`;
  no behavior change for existing invocations.

## 2. Training pipeline for the new voice (existing scripts, new data)

1. **Data prep** — `phonemize_corpus.py` → `tts_build_training.py` over the new
   reciter's transcribed corpus, metrical **and prose**. Same `prep_text` frontend
   (Kannada routing + sandhi rules apply identically). Prose chunks on safe word
   boundaries with a `gadya` label instead of a meter key; gadya adds phonetic
   coverage in a non-metrical register.
2. **Voice fine-tune** — `finetune_indicf5.py`, warm-started from the **GRN-corrected
   Indic checkpoint** (not from the pilot voice, to avoid inheriting its timbre).
   lr 1e-5, bf16, 100-band mel @ 24 kHz. Runs on the A6000 host.
3. **Voice-steering phase 2** — `phase2_finetune.py` with ~150–200 paired clips from
   the new corpus (mirrors the shipped 179-clip recipe) so the voice is
   reference-responsive.
4. **Reference bank build** — per meter: a half-hemistich clip (~7 s, clean daṇḍa
   boundary, half-reference rule) with exactly-matching text; `sec_per_syll`
   measured from the new reciter's actual pace per meter; repeat-primes
   (`prime_mono`, `prime_jaya`, `prime_chata` equivalents) selected or recorded.
   **Gadya entries included**, with prose `sec_per_syll` calibrated separately from
   chant pace. BigVGAN stays shared; per-voice vocoder fine-tune only if by-ear
   checks reveal artifacts.

## 3. Code changes (small, surgical)

- **`src/voices.py`** (new, ~50 lines): resolves `voice_id → {checkpoint path,
  bank path}` from `voices/*/voice.json`. Single resolution point for renderers
  and demos.
- **`render.py` / `render_core.py` / `render_production.py`**: `--voice` flag
  replaces hard-coded bank/checkpoint defaults. Helpers are mirrored verbatim
  between `render.py` and `render_core.py` per repo rule.
- **Meter signatures**: detection keeps calibrating from the **pilot** bank
  (canonical). A voice bank missing a meter (or `gadya`) falls back to the pilot
  clip for that entry with a logged warning — degraded timbre for that verse,
  never a hard failure.
- **`fix_duration`**: computed from the *selected voice's* `sec_per_syll` and its
  actual reference length, including prime swaps (the 2026-06-22 rule holds
  per-voice). `fix_duration` remains TOTAL (ref + generated).
- **Demos**: voice dropdown in `demo/app.py` and `demo/server.py`. The warm server
  loads packs lazily and keeps them resident (~2.5 GB each; fine on the A6000).

## 4. Verification (by-ear, per project QC philosophy)

- **Regression gate:** render `examples/sample_shard.json` with `--voice pilot`
  before/after the refactor — durations must match the prior good output exactly.
- **New-voice checks:** render the same shard with the new voice; by-ear check the
  known hard cases: retroflex conjuncts (ṣṭ, ḍḍh), long-vowel daṇḍa-final visarga,
  a repeat-prime verse, one held-out meter, and a prose (gadya) chunk.
- Duration gate + internal-silence gate run as-is (voice-agnostic).

## Constraints & notes

- Locked inference parameters (NFE, sway, speed, CFG) are unchanged and voice-independent.
- The repeated-syllable depth limitation (beyond ~4 is unrecoverable) is expected to persist in any voice; not a bug.
- **Ethics (paper §14):** obtain and record the new reciter's explicit consent before
  training; reference it from `voice.json`. The system remains liturgical-register,
  not a general voice-cloning tool.
- Weights per voice ≈ 900 MB half-precision on HF; repo stays weight-free.

## Out of scope

- Multi-reciter single model (possible later research track).
- Voice-agnostic zero-shot chant.
- Any change to the frontend (`prep_text`), meter detection logic, or locked
  inference parameters.
