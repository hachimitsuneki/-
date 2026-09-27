# CODEX_TASK_AUDIO_SDS_POC.md

- Task group: NVIDIA Audio-SDS reproduction
- Project: 音楽聞き分けアプリ
- Status: Codex実装準備
- Important: 実装成功を前提にしない。各stepで実測結果を記録する。

## Goal

NVIDIA Audio-SDS論文のsource separationを、公開Stable Audio Open tooling上で再現可能か検証する。
成功条件はアプリ完成ではなく、paper-like separation runを再現可能な実験として保存すること。

## Non-goals

- UI完成
- 長時間曲
- Performer個人識別
- Jev統合
- PDF譜面
- production API
- Stable Audio 3.x移植
- CARV高速化

## Required design

### Adapter boundary

```python
class AudioDiffusionBackend(Protocol):
    def load(...): ...
    def build_conditioning(self, prompts, ...): ...
    def latent_spec(self, duration_sec, channels, ...): ...
    def decode(self, latents): ...
    def partial_denoise(self, noised_latents, timesteps, conditioning, cfg_scale, steps): ...
```

Audio-SDS optimizerはStable Audio具体実装へ直接依存しない。

### Experiment provenance
最低限保存:
- git commit
- config
- model repo/revision
- library versions
- CUDA/driver
- GPU name / total VRAM
- peak allocated/reserved VRAM
- wall clock
- random seed
- input checksum
- prompts
- intermediate/final metrics
- failure traceback

## Implementation steps

### Step 0 — Environment smoke

Separate environment.

Candidate baseline:
- Python 3.10
- PyTorch / torchaudio versions resolved by current stable-audio-tools
- CUDA-compatible NVIDIA GPU
- Hugging Face authentication
- accepted Stable Audio Open model terms

Deliver:
- `scripts/smoke_stable_audio.py`
- one generated WAV
- `environment_report.json`

Acceptance:
- official model repoからload
- generation成功
- exact versions/report保存

### Step 1 — Backend wrapper

Implement `StableAudioBackend`.

Acceptance:
- text conditioning
- latent shape
- decode
- partial denoise
を個別に呼べ、shape/dtype/device unit testが通る。

### Step 2 — Differentiable decoder test

Create trainable dummy latent → decode → simple waveform loss + multiscale STFT magnitude loss → backward.

Acceptance:
- finite gradient
- short repeated loopでmemory leakなし
- peak VRAM logged

Failならfull separationへ進まずblocker report。

### Step 3 — Multiscale STFT

Implement sizes:
- 1024
- 2048
- 4096

Unit tests:
- identical audio => ~0
- modified audio => positive
- finite gradient

APIを独立し、custom adjoint/STFTへ後で差替え可能にする。

### Step 4 — Decoder-SDS unit

Implement paper Decoder-SDS direction.

Requirements:
- teacher frozen
- encoder gradientを避ける
- stop-gradient/detach位置を明示
- update sign conventionを文書化
- teacher weightにgradientが付かないtest

Acceptance:
- one stepでsource latent変化
- no teacher gradient
- update norm logged

### Step 5 — Tiny 2-source optimizer

- duration 2–3 sec
- semantically distant sources
- 50–100 iterationsから開始

Track:
- reconstruction loss
- prompt CLAP if available
- source sum reconstruction
- intermediate WAV

Acceptance:
- stable loop
- NaN/OOMなし
- outputs generated
- fixed seedでdebug可能

### Step 6 — Paper baseline profile

```yaml
iterations: 1000
lr: 0.05
sds_batch_size: 10
clip_sec: 10
stft_sizes: [1024, 2048, 4096]
timestep_min: 0.025
timestep_max: 0.875
cfg_scale: 60
denoise_steps: 2
gamma: 0.02
```

Acceptance:
- completes or fails with reproducible blocker
- elapsed/peak VRAM recorded
- source outputs + metrics saved

### Step 7 — GPU fallback

If full batch does not fit:
- add `sds_microbatch_size`
- accumulate guidance direction
- tiny fixtureでbatch版とのnumerical closeness確認

Paper baseline resultと混ぜず、`paper_equivalent_microbatched`等で別profileにする。

### Step 8 — Project integration

Standalone PoC成功後のみ:
- `SeparatorAdapter`
- `SeparationRun`
- `SegmentJob`
- `SeparationTarget`
- `SeparatedSegment`
- ground-truth evaluation fixture

## Required tests

- config parsing
- latent shapes
- no teacher gradients
- STFT backward
- <=3 sec integration smoke
- CUDA OOM reporting
- checkpoint/resume
- fixed-seed reproducibility report

## Stop conditions

Stop and report when:
1. model access/license acceptance unavailable
2. decoder gradient path不可
3. repeated STFT backwardにmemory leak
4. tiny 2-source caseがreasonable debugging後もNaN
5. public toolingからpaperに必要なbackend APIへ到達不可
6. reduced 2-sec/microbatchでもhardware不適合

NVIDIAを別separatorへ勝手に置換しない。

## Deliverables

```text
src/audio_sds/...
configs/audio_sds/paper_baseline.yaml
scripts/smoke_stable_audio.py
scripts/run_audio_sds_poc.py
tests/audio_sds/...
reports/audio_sds/<run_id>/environment.json
reports/audio_sds/<run_id>/metrics.json
reports/audio_sds/<run_id>/notes.md
```

## After baseline

自動で実装しない。次の候補:
1. generic instrument prompt vs score-derived prompt
2. K > 2
3. same-instrument separation
4. chunking
5. CARV
6. score-prior loss
