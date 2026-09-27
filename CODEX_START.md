# CODEX_START.md

このリポジトリの実装を引き継いでください。

## 最初に読む順序

1. `PROJECT_HANDOFF.md`
2. `docs/CODEX_SLICE3_NVIDIA_INTEGRATION_SPEC.md`
3. `docs/SEPARATION_ER_V5.md`
4. `docs/CODEX_TASK_AUDIO_SDS_POC.md`

## 最優先タスク

Audio-SDSの実現可能性を実機で検証してください。

順番:
- T0: Stable Audio Openをロードして通常生成smoke test
- T1: latent / decoder API確認
- T2: trainable latent → decoder → multi-scale STFT loss → backward

T0〜T2が成功してからT3以降へ進んでください。

## 必須記録

- GPU名
- total VRAM
- peak allocated / reserved VRAM
- Python / PyTorch / CUDA / stable-audio-tools version
- model revision
- wall clock
- random seed
- input checksum
- prompts
- failure traceback

## 重要な制約

- NVIDIA Audio-SDSを主要分離方式とする方向を維持する。
- 難しいからという理由で他separatorを主要方式へ勝手に置換しない。
- core applicationとGPU workerを分離する。
- failed run/attemptを削除しない。
- 実装済みと検証済みを区別する。
- 既存ER/要件を理由なく作り直さない。
- 資料とコードが矛盾する場合は差分を報告する。

## 作業後の報告

1. 変更したファイル
2. 実行したコマンド
3. テスト結果
4. GPU/VRAM実測
5. T0/T1/T2それぞれの結果
6. blocker
7. 次に進める条件
8. PROJECT_HANDOFFへ反映すべき変更
