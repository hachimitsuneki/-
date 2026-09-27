# 音楽聞き分けアプリ / Music Separation Workbench

譜面と録音を使い、NVIDIA Audio-SDSを主要な分離方式として、譜面パート単位・最終的には演奏者単位の音源分離を目指す研究開発プロジェクトです。

## Codexで開始する場合

次の順番で読んでください。

1. `PROJECT_HANDOFF.md`
2. `CODEX_START.md`
3. `docs/CODEX_SLICE3_NVIDIA_INTEGRATION_SPEC.md`
4. `docs/SEPARATION_ER_V5.md`
5. `docs/CODEX_TASK_AUDIO_SDS_POC.md`

現在の最優先はAudio-SDSのGPU実現可能性検証です。

- T0: Stable Audio Open smoke
- T1: latent / decoder API
- T2: differentiable decoder + multi-scale STFT backward

T0〜T2の実測を取るまで、Audio-SDSを「検証済み」と扱わないでください。

## 重要方針

- 音を実際に分離する中核はNVIDIA AI。
- ScorePartと実際のPerformerを同一視しない。
- `Trumpet 1`という譜面名をそのまま唯一のAI promptにしない。
- GPU workerとcore applicationを分離する。
- failed run/attemptを削除せず、retryは別runとして残す。
