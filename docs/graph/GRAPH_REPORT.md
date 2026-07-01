# Graph Report - .  (2026-06-30)

## Corpus Check
- cluster-only mode — file stats not available

## Summary
- 313 nodes · 445 edges · 22 communities (15 shown, 7 thin omitted)
- Extraction: 96% EXTRACTED · 4% INFERRED · 0% AMBIGUOUS · INFERRED: 19 edges (avg confidence: 0.8)
- Token cost: 0 input · 0 output

## Graph Freshness
- Built from commit: `76ba4e1c`
- Run `git rev-parse HEAD` and compare to check if the graph is stale.
- Run `graphify update .` after code changes (no API cost).

## Community Hubs (Navigation)
- [[_COMMUNITY_Meter detection & G2P (tts_meter  tts_g2p)|Meter detection & G2P (tts_meter / tts_g2p)]]
- [[_COMMUNITY_Production renderer (render_production.py)|Production renderer (render_production.py)]]
- [[_COMMUNITY_GaṇaDiT gaṇa-conditioning (gana_f5.py)|GaṇaDiT gaṇa-conditioning (gana_f5.py)]]
- [[_COMMUNITY_Text frontend — sandhi & Kannada routing (prep_text.py)|Text frontend — sandhi & Kannada routing (prep_text.py)]]
- [[_COMMUNITY_Batch renderer (render.py)|Batch renderer (render.py)]]
- [[_COMMUNITY_Reusable render core (render_core.py  Renderer)|Reusable render core (render_core.py / Renderer)]]
- [[_COMMUNITY_Chandas  gaṇa scansion labeler (chandas_labeler.py)|Chandas / gaṇa scansion labeler (chandas_labeler.py)]]
- [[_COMMUNITY_Warm demo server & abuse limits (demoserver.py, limits.py)|Warm demo server & abuse limits (demo/server.py, limits.py)]]
- [[_COMMUNITY_Sanskrit corpus downloader (training)|Sanskrit corpus downloader (training)]]
- [[_COMMUNITY_HF ZeroGPU demo app (demoapp.py)|HF ZeroGPU demo app (demo/app.py)]]
- [[_COMMUNITY_Forced alignment — IndicConformer (training)|Forced alignment — IndicConformer (training)]]
- [[_COMMUNITY_Corpus phonemization (training)|Corpus phonemization (training)]]
- [[_COMMUNITY_BigVGAN vocoder training|BigVGAN vocoder training]]
- [[_COMMUNITY_Phase-2 vocoder fine-tune|Phase-2 vocoder fine-tune]]
- [[_COMMUNITY_Candidate renderer (render_candidates.py)|Candidate renderer (render_candidates.py)]]
- [[_COMMUNITY_IndicF5 GRN fix (training)|IndicF5 GRN fix (training)]]
- [[_COMMUNITY_Environment setup script (scriptssetup.sh)|Environment setup script (scripts/setup.sh)]]
- [[_COMMUNITY_Weight downloader (scriptsdownload_weights.py)|Weight downloader (scripts/download_weights.py)]]
- [[_COMMUNITY_Clip builder (build_clips.py)|Clip builder (build_clips.py)]]
- [[_COMMUNITY_IndicF5 voice fine-tune (training)|IndicF5 voice fine-tune (training)]]
- [[_COMMUNITY_IndicF5 clone  voice-steer (training)|IndicF5 clone / voice-steer (training)]]

## God Nodes (most connected - your core abstractions)
1. `render_clip()` - 14 edges
2. `Renderer` - 9 edges
3. `normalize()` - 8 edges
4. `to_deva()` - 7 edges
5. `strip_punct()` - 7 edges
6. `detect_meter_key()` - 7 edges
7. `analyze()` - 7 edges
8. `syllabify()` - 7 edges
9. `build_row()` - 7 edges
10. `char_gana_ids()` - 6 edges

## Surprising Connections (you probably didn't know these)
- `_get_renderer()` --calls--> `Renderer`  [INFERRED]
  demo/app.py → src/render_core.py
- `synthesize()` --calls--> `detect_meter_key()`  [INFERRED]
  demo/app.py → src/render_core.py
- `synthesize()` --calls--> `detect_meter_key()`  [INFERRED]
  demo/server.py → src/render_core.py
- `build_row()` --calls--> `analyze()`  [INFERRED]
  training/tts_build_training.py → src/tts_meter.py
- `build_row()` --calls--> `normalize()`  [INFERRED]
  training/tts_build_training.py → src/tts_normalize.py

## Import Cycles
- None detected.

## Communities (22 total, 7 thin omitted)

### Community 0 - "Meter detection & G2P (tts_meter / tts_g2p)"
Cohesion: 0.08
Nodes (38): Path, detect_meter_key(), Best-effort chandas (meter) detection from raw text in ANY Indic script, so a no, analyze(), assign_pada_positions(), detect_meter(), mark_sutras(), _match_meter() (+30 more)

### Community 1 - "Production renderer (render_production.py)"
Cohesion: 0.07
Nodes (25): _anusvara_m(), _basetext(), Cap, _danda_fix(), gate(), _hna_metathesis(), n_aksharas(), PRODUCTION renderer — Prathosh Sanskrit chant TTS. Gold pipeline: voice_armA_ema (+17 more)

### Community 2 - "GaṇaDiT gaṇa-conditioning (gana_f5.py)"
Cohesion: 0.09
Nodes (23): add_cond(), add_gana(), cond_cfm_forward(), gana_cfm_forward(), _gana_ids(), GaṇaDiT integration for IndicF5 — adds a zero-init per-char gaṇa(L/G) embedding, Attach a zero-init gaṇa embedding and rebind forward. n_gana: 0=filler,1=laghu,2, Per-char L/G ids for text t, cached (764 unique texts → scanned once each). (+15 more)

### Community 3 - "Text frontend — sandhi & Kannada routing (prep_text.py)"
Cohesion: 0.11
Nodes (27): align_slp1(), detect_script(), fix_colon(), mfa_text(), model_text(), model_text_norm(), model_text_sandhi(), _next_real() (+19 more)

### Community 4 - "Batch renderer (render.py)"
Cohesion: 0.14
Nodes (20): _aksharas(), _anusvara_m(), bvgan(), Cap, _danda_fix(), _ends_halant(), gate(), get_ref() (+12 more)

### Community 5 - "Reusable render core (render_core.py / Renderer)"
Cohesion: 0.14
Nodes (16): _aksharas(), _anusvara_m(), _danda_fix(), _ends_halant(), gate(), _hna_metathesis(), n_aksharas(), Reusable render core for Vāgdhenu — the gold per-hemistich pipeline as a callabl (+8 more)

### Community 6 - "Chandas / gaṇa scansion labeler (chandas_labeler.py)"
Cohesion: 0.14
Nodes (15): char_gana_ids(), _clean(), kannada_aksharas(), _kn_class(), label_verse(), Chandas / gaṇa labeler — per-akṣara laghu(L,1 mātrā)/guru(G,2 mātrā) scansion +, Segment Kannada into akṣaras → list of (kind, [char_indices]); kind 'aks' or 'fi, Per-char L/G id aligned to kn_text: 0=filler, 1=laghu, 2=guru. Returns (ids, ok, (+7 more)

### Community 7 - "Warm demo server & abuse limits (demo/server.py, limits.py)"
Cohesion: 0.14
Nodes (13): Request, Vāgdhenu standalone warm server for a dedicated GPU (ece A6000). Loads the model, _resolve(), synthesize(), check_and_count(), client_ip(), _n_aksharas(), Abuse guards shared by the demo server (ece) and the HF Space:  - one shloka per (+5 more)

### Community 8 - "Sanskrit corpus downloader (training)"
Cohesion: 0.20
Nodes (12): HTMLParser, download_dsbc(), download_gretil(), download_one(), download_sangraha(), fetch_url(), harvest_gretil_links(), _LinkHarvester (+4 more)

### Community 9 - "HF ZeroGPU demo app (demo/app.py)"
Cohesion: 0.21
Nodes (11): _ensure_assets(), _ensure_bigvgan(), _get_renderer(), Request, Vāgdhenu — Sanskrit/chant TTS demo (Hugging Face ZeroGPU Space).  Loads the rele, Fetch the 2 release weights (CPU-only) + resolve the tokenizer vocab. vocab.txt, detected meter name -> (pretty bank key, recognized?)., NVIDIA BigVGAN ships as a repo, not a pip package — clone it once and put it on (+3 more)

### Community 10 - "Forced alignment — IndicConformer (training)"
Cohesion: 0.26
Nodes (11): align_one(), conformer_frame_to_attn_frame(), get_ctc_log_probs(), load_model(), main(), mode='pilot' returns 30 stratified clips; mode='full' returns all eligible., Forward audio → CTC log-probs (T, V). Resamples to 16 kHz first., Devanagari → Sanskrit BPE token ids using the model's tokenizer. (+3 more)

### Community 11 - "Corpus phonemization (training)"
Cohesion: 0.26
Nodes (11): chunk_text(), collect_files(), detect_script(), main(), quality_ok(), Return 'devanagari', 'iast', or 'unknown'., Transliterate to Devanagari if needed., Reject low-quality docs (mostly for OCR'd Sangraha). (+3 more)

### Community 12 - "BigVGAN vocoder training"
Cohesion: 0.36
Nodes (4): main(), mel_spectrogram(), Standalone BigVGAN vocoder (mel -> audio) for Matcha-TTS Sanskrit. Reuses VITS2, VocDS

### Community 13 - "Phase-2 vocoder fine-tune"
Cohesion: 0.32
Nodes (4): main(), mel_spectrogram(), P2DS, Rung 1 — Phase-2 vocoder fine-tune: map Matcha's PREDICTED (teacher-forced, GT-a

## Knowledge Gaps
- **2 isolated node(s):** `setup.sh script`, `PYTHONPATH`
  These have ≤1 connection - possible missing edges or undocumented components.
- **7 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `detect_meter_key()` connect `Meter detection & G2P (tts_meter / tts_g2p)` to `HF ZeroGPU demo app (demo/app.py)`, `Reusable render core (render_core.py / Renderer)`, `Warm demo server & abuse limits (demo/server.py, limits.py)`?**
  _High betweenness centrality (0.203) - this node is a cross-community bridge._
- **Why does `Renderer` connect `Reusable render core (render_core.py / Renderer)` to `HF ZeroGPU demo app (demo/app.py)`?**
  _High betweenness centrality (0.080) - this node is a cross-community bridge._
- **What connects `Vāgdhenu — Sanskrit/chant TTS demo (Hugging Face ZeroGPU Space).  Loads the rele`, `NVIDIA BigVGAN ships as a repo, not a pip package — clone it once and put it on`, `Fetch the 2 release weights (CPU-only) + resolve the tokenizer vocab. vocab.txt` to the rest of the system?**
  _89 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Meter detection & G2P (tts_meter / tts_g2p)` be split into smaller, more focused modules?**
  _Cohesion score 0.07890070921985816 - nodes in this community are weakly interconnected._
- **Should `Production renderer (render_production.py)` be split into smaller, more focused modules?**
  _Cohesion score 0.06653225806451613 - nodes in this community are weakly interconnected._
- **Should `GaṇaDiT gaṇa-conditioning (gana_f5.py)` be split into smaller, more focused modules?**
  _Cohesion score 0.09359605911330049 - nodes in this community are weakly interconnected._
- **Should `Text frontend — sandhi & Kannada routing (prep_text.py)` be split into smaller, more focused modules?**
  _Cohesion score 0.11375661375661375 - nodes in this community are weakly interconnected._