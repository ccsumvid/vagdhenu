# CLAUDE.md — Vāgdhenu

Guidance for Claude Code when working in this repository. Grounded in the project paper
(`docs/vagdhenu_paper.pdf` / `docs/TECH_REPORT.md`): *"Vāgdhenu: A Vṛtta (Meter) Aware
Śloka-to-Chant (TTS) System for Sanskrit"*, Prathosh A P, IISc Bengaluru (preprint, 2026).

## What this is

**Vāgdhenu** ("the wish-granting cow of speech") is a **vṛtta (meter) aware śloka-to-chant TTS**
system for Sanskrit: it maps a metrical verse to its chanted **pārāyaṇa** recitation, not flat
read-aloud. Any Brahmic script goes in; chant audio comes out.

The paper frames it as an **experience report, not a new architecture**: it takes an off-the-shelf
flow-matching TTS backbone + a large neural vocoder, and adds the components a *faithful Sanskrit
chant* pipeline actually needs (script-aware frontend, sandhi/phonology rules, a meter-driven
reference mechanism, and a lightweight QC stack). Expert single-listener **MOS ≈ 4.6**; dense
conjuncts including retroflex aspirates (ṣṭ, ḍḍh, …) render correctly — the class earlier
architectures could not crack.

- **License:** Apache-2.0 (`LICENSE`). Single-speaker (the author's own voice); liturgical register,
  **not** a general voice-cloning tool (paper §14, ethics).
- **Weights:** Hugging Face `prathoshap/vagdhenu` (not in this repo — gitignored, pulled at runtime).
- **Live demo / weights / report:** https://prathosh.in/vagdhenu/ · HF Space `prathoshap/vagdhenu-demo`.

## The three load-bearing ideas (read before changing the pipeline)

1. **Script routing through Kannada (paper §3.1).** Indic backbones trained across Indian scripts
   apply Hindi phonotactics to Devanagari and delete the inherent schwa — wrong for Sanskrit. The
   *same* text in **Kannada** script is read with the vowel intact. So **all input is transliterated
   to Kannada before synthesis**; display text/manifests keep the original. Never "simplify" by
   feeding Devanagari to the model.

2. **The negative result on prosody (paper §7).** In a self-infilling flow-matching backbone a
   **text-side prosody conditioner is architecturally inert** — during mel-infilling the model
   already recovers pitch from the context mel, so a gaṇa/swara text embedding gets essentially no
   gradient (conditioning-gradient norm ≈ 0.02; effect ≈ 0 semitones). The **only working prosody
   levers are the reference clip and a voice-steering retrain.** Do not re-attempt text-side embedding
   tuning on this backbone. (This is why there is no duration/pitch head anywhere in the code.)

3. **Meter drives everything through the reference, not a learned embedding (paper §5.2).** The
   detected vṛtta selects an exactly-matched reference clip from the bank, which sets voice, melodic
   contour, pace, the per-meter duration budget, and the caesura (yati) position.

## Architecture / inference pipeline (paper §5, Fig. 1)

```
Sanskrit text (any Brahmic) → Frontend (Kannada routing + sandhi) → Meter scan + reference selection
   → Flow-matching DiT backbone (mel-infilling; no duration/pitch head) → BigVGAN-v2 vocoder
   → Post-gate (silence/breath) → chant audio
```

- **Backbone:** IndicF5 / F5-TTS — a diffusion-transformer flow-matching model, **~337M params**,
  config `dim=1024, depth=22, heads=16, ff_mult=2, text_dim=512, conv_layers=4`. Conditional-flow-
  matching velocity regression on a masked mel-infilling task. **No native duration or pitch head.**
- **Vocoder:** NVIDIA **BigVGAN-v2** (large-scale adversarial), fine-tuned on the backbone's own
  vocos-mel. **Mandatory** — the Fourier-domain vocos alternative introduced a long-vowel phase
  artifact. Must load into the correct generator class or it silently half-loads → monotone output.
- **Deployed size:** ~449M params total (~900MB half-precision), peak inference ~2.5GB. RTF ≈ **1.24
  at NFE 64**, ≈ **0.63 at NFE 32** (the serving setting). **A CUDA GPU is required.**

### The half-reference rule (paper §5.2) — the practical prosody lever

The reference *text* must match the spoken span of the reference *audio* on a clean word/daṇḍa
boundary. A clean **half-hemistich (~7 s)** outperforms a full ~15 s śloka (a full verse ending on a
cadence swallows the first synthesized syllable). A half-śloka reference **repeated 3× to ~20 s**
markedly improves melodic contour. This is encoded in the bank + the autoprime logic (see below).

## Repository layout

```
src/         text frontend, meter detection, inference core, post-gate, reference bank
demo/        Gradio apps (HF ZeroGPU Space + standalone warm server)
docs/        paper PDF, scrubbed tech report, project page (GitHub Pages)
training/    fine-tuning + data-prep (voice-steer, BigVGAN, corpus download/phonemize, alignment)
scripts/     env setup + weight download
examples/    sample shard input + rendered output
```

### Key modules (`src/`)

- **`prep_text.py`** — the frontend, and the most reusable piece. Any-Brahmic → Devanagari → SLP1 →
  **Kannada routing**, plus the paper's §3.2 / App-B rules **in order**: transliterate → punctuation
  strip → optional internal **visarga sandhi** (utva/rutva/lopa/satva) → homorganic **anusvāra** →
  **daṇḍa-final visarga** (echoed after a short vowel, bare after a long one so it elongates) →
  **vocalic-ṛ** → metathesis for two awkward conjuncts → strip editorial parentheticals before
  metrical analysis. **Citation (pramāṇa) verses are rendered WITHOUT sandhi** — hence the
  `no_sandhi` flag threaded through the renderers. Produces `model_text` (Kannada model path) and
  `mfa_text` (phonetic Devanagari for MFA alignment).
- **`tts_syllabify.py` / `tts_weight.py` / `tts_meter.py`** — SLP1 syllabifier → guru/laghu weight
  tagger → **vṛtta detection**. Meter signatures are self-calibrated from the reference bank;
  matching is per-pāda with quarter-final anceps ignored; mixed-meter verses handled per half-verse.
  `render_core.detect_meter_key()` wires these together.
- **`render.py`** — frozen **batch** renderer (the gold path). Loads DiT+BigVGAN once, loops a shard
  JSON, writes dry per-hemistich wavs. Change its behavior only with care.
- **`render_core.py`** — the same gold pipeline as a reusable `Renderer` class (`render_one()`
  returns audio instead of writing). Used by the demos. **Helpers are copied verbatim from
  `render.py`** — fix a bug in one, mirror it in the other.
- **`render_production.py` / `render_candidates.py`** — single-verse reference renderer; candidate
  generation for by-ear selection.
- **`chandas_labeler.py` / `gana_f5.py` / `gana_uni.py`** — per-akṣara L/G scansion; a (now-inert,
  per §7) GaṇaDiT gaṇa-conditioning experiment. `gana_f5` is retained but is **not** a working
  prosody lever.
- **`limits.py`** — pure-stdlib in-memory per-IP daily quota + one-shloka validation for the demos.
- **`reference_bank/`** — `bank.json` (per-meter reference clips, `sec_per_syll`, and **repeat-primes**
  for repeated-syllable words) + `vocab.txt` (IndicF5's MIT tokenizer vocab, bundled so there's no
  gated-repo dependency).

**Autoprime / repeated-token limit.** The renderers detect mono-/di-repeated akṣaras and swap in a
`repeat_prime` reference (`prime_mono`, `prime_jaya`, `prime_chata`). This corresponds to the paper's
finding (§14, E34) that **repeated-syllable depth beyond ~4 is unrecoverable even with priming** — a
known limitation, not a bug to "fix."

### Demos (`demo/`)

- **`app.py`** — HF **ZeroGPU** Space; model load + synthesis inside `@spaces.GPU`, weights from
  `prathoshap/vagdhenu`. Header in `demo/README.md`.
- **`server.py`** — standalone **warm** server for a dedicated GPU (loads once, no ZeroGPU quota;
  uses NFE 32 for RTF ≈ 0.63).

## Locked inference parameters (paper App. A — do not drift without reason)

Euler solver · **NFE 64** (batch) · sway-sampling coeff ≈ **−0.7** · **speed 0.90** · per-clip
`fix_duration = ref_len + n_syllables × sec_per_syll` · **CFG ≈ 3.0** for the batch renderer (to
suppress tremor), lower for single-verse work (expression-vs-tremor trade, §7). These match the
`render.py` argparse defaults; keep them in sync.

- **`fix_duration` is TOTAL (ref + generated).** When a repeat-prime swaps the reference, recompute
  `ref_len` from the *prime's* actual length (see the 2026-06-22 note in `render.py`) or output
  collapses to ~1.5 s.
- CFG trades expression against tremor: low = expressive but tremulous, high = clean but flatter.

## Environment & setup

**Requires Python 3.10 and a CUDA 12.1 GPU.** (The local box here has only Python 3.9 and no GPU —
inference must run on the A6000 host. Use `uv` for any modern-Python tooling.)

```bash
bash scripts/setup.sh    # torch+cu121, deps, clone BigVGAN onto PYTHONPATH, download weights -> models/
```

`scripts/setup.sh` is deliberately not a plain `pip install`:
- installs `torch==2.4.1 torchaudio==2.4.1 +cu121` from the PyTorch cu121 index;
- `pip install -r requirements.txt`;
- **clones NVIDIA BigVGAN** and puts it on `PYTHONPATH` (BigVGAN is a repo, not a pip package; we use
  the torch path, `use_cuda_kernel=False`, so nothing is compiled);
- runs `scripts/download_weights.py` (weights → `models/`).

### Dependency gotchas (read before editing requirements)

- **`f5_tts` is IndicF5's fork**, pinned to an exact commit in `requirements.txt`
  (`git+https://github.com/ai4bharat/IndicF5.git@13f7c4d…`). **Do NOT** use the PyPI `f5-tts`
  package — it is different code whose `infer_process` crashes. (A normalization bug in the installed
  backbone had to be patched for the text encoder to function — paper §4, E29.)
- **BigVGAN** is cloned, not pip-installed; `import bigvgan` resolves via `PYTHONPATH`.
- Weights (`*.pt/*.pth/*.safetensors`) and audio (`*.wav`, except the reference bank + `examples/`)
  are gitignored — they live on HF.

## Running

```bash
# batch render a shard (id, meter, padas[deva…], seed, out) -> per-hemistich wavs:
python src/render.py --shard examples/sample_shard.json --results /tmp/res.json --outdir out
#   -> out/sample_anushtubh.wav

# single verse:  src/render_production.py
# ZeroGPU demo:  VAGDHENU_HF=prathoshap/vagdhenu python demo/app.py
# warm server:                                    python demo/server.py   (port 7860)
```

`CHAMP_ROOT` overrides the weights dir (default `models/`). Demo weights/repo overridable via
`VAGDHENU_HF`, `VAGDHENU_VOICE`, `VAGDHENU_VOC`, `VAGDHENU_VOCAB`.

## Model lineage (paper §4, Table 1) — why the current design

Four architecture families on the *same* single-speaker data; each earlier one hit a hard ceiling,
and the flow-matching backbone cleared it **with a 5-hour clone, not more data — the backbone, not
the data, was the bottleneck.** Eras 1–3 are retired; do not re-walk them (detailed ledger: App. C).

| Era | Backbone            | Params | MOS      | Ceiling reached                         |
|-----|---------------------|--------|----------|-----------------------------------------|
| 1   | StyleTTS2           | 70M    | 3.0–4.2  | conjuncts muffled; English priors leaked |
| 2   | VITS2               | 39.9M  | > Era 1  | conjuncts muffled (retroflex data sparsity) |
| 3   | Matcha-TTS          | 18.2M  | 4.2–4.3  | conjuncts improved; meter prosody missing |
| 4   | Flow-matching (IndicF5) | 337M | **4.6** | **production** — cleared the conjunct class |

## Data & training (paper §6, §8) — reference only; corpus/weights not in this repo

- Corpus designed for **metrical + phonetic balance** (not natural frequency): coverage targets like
  ≥300 of each retroflex consonant, ≥500 long vowels, ≥150 of each hard conjunct, ≥100 of each of 57
  phonemes. `rucirā` and `mālinī` were **held out** as unseen-meter generalization tests.
- Released dataset: ~1,467 clips (~5.3 h) in two recording styles, CC-attribution; primary style ~3.0
  h / 764 verses / ~33,700 syllables across ~9 meter families.
- Voice fine-tune: warm-start Indic ckpt, lr 1e-5, bfloat16, 100-band mel @ 24 kHz, 2 GPUs; a
  voice-steering retrain on **179 paired clips** makes the voice reference-responsive (shipped voice;
  reference-driven fine-tune kept as fallback).

## Quality control (paper §9) — a deliberately lightweight stack

Forced-alignment QC was **dropped**: on 10,220 clips it produced ~1,648 flags with **~none real**
(false positives from conjuncts, fricative onsets, long meters, reference priming). Replaced by a
duration gate + internal-silence gate + best-of-N reseed + recognition-based error detection +
structural/source validation. **The dominant source of real defects is the source text** (numbering
dupes/gaps, merged entries, editorial brackets, typos), which the synthesizer reproduces faithfully —
so source validation matters more than acoustic QC. This is why the code favors by-ear checks and
"confirmed by ear on vNNN …" notes over an automated alignment suite.

## Deployments (paper §10–11) — produced with this system

- **MBTN (Mahābhārata Tātparya Nirṇaya)** — 32 chapters, 5,183 verses → 32 videos (bilingual cards,
  per-hemistich highlight, tonic drone), ~17.5 h; 4-GPU render ≈ 13 h.
- **Śrīmad Bhāgavatam** — ~18,000 verses, ~345 chapters, 16,017 rendering units (14,042 metrical, 559
  prose/gadya, 1,416 short connectives). Prose was the hardest sub-problem (chunking on safe word
  boundaries, care at de-sandhied junctions).

The corpus-rendering / karaoke / video-assembly pipelines of the two deployments are **excluded from
the open release** (paper §13) — this repo is the frontend + inference + training code only.

## Testing

There is **no automated test suite** (consistent with the QC philosophy above — verification is
by-ear). If you add tests, follow the test pyramid; for pipeline changes, render a known shard and
diff durations/behavior against a prior good output.

## Known limitations (paper §14 — don't treat these as bugs)

- **Designable, verse-independent prosody is out of reach** on this backbone (would need an explicit
  duration predictor + length regulator — designed but not built).
- **Repeated-syllable depth beyond ~4** is unrecoverable even with priming.
- Voice-agnostic zero-shot chant would need a multi-reciter fine-tune (weights are single-speaker).
- Evaluation is expert single-listener MOS — a limitation, not a controlled study.

## Graphical documentation (graphify)

A queryable knowledge-graph view is generated with **`graphifyy`** (github.com/safishamsi/graphify —
note the double-y package name):

```bash
uv tool install graphifyy          # or: pip install graphifyy
graphify .                         # -> graphify-out/{graph.html, GRAPH_REPORT.md, graph.json}
graphify cluster-only . --no-label # regenerate report/HTML without an LLM
graphify tree --label "Vāgdhenu"   # collapsible file-hierarchy HTML
graphify export callflow-html      # Mermaid call-flow HTML
```

Code parsing is fully local via tree-sitter (no API key). `graphify-out/` is generated output
(already gitignored). **Community naming needs an LLM backend** (`GOOGLE_API_KEY`/`OPENAI_API_KEY`/…
then `graphify label .`); without one it falls back to `Community N`. The current
`graphify-out/.graphify_labels.json` holds hand-authored module names (one community ≈ one source
file); re-clustering is deterministic, so those IDs stay stable across rebuilds.
