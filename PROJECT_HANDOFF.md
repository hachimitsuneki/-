# PROJECT_HANDOFF.md

- Project: 音楽聞き分けアプリ
- Version: 0.8-repo-bootstrap
- Updated: 2026-09-27
- Primary implementation agent: Codex

## 1. 目的

録音と譜面を使い、混合音源から譜面パートを特定し、最終的には同一楽器の複数演奏者を個人単位で分離する。

明示決定:
- 実装はCodexを主に使う。
- 音を実際に分ける中核はNVIDIA AI。
- 譜面情報を分離判断へ使う。
- 最終目標は「誰がどの音を演奏しているか」を個人単位まで扱うこと。

## 2. 現在のアーキテクチャ

```text
Audio + MusicXML
      |
      v
Canonical Score
      |
      v
Score↔Audio Alignment
      |
      v
TargetDescriptor
      |
      v
PromptVariant + SegmentJob
      |
      v
NVIDIA Audio-SDS worker
      |
      v
SeparatedSegment / Artifact
```

重要な概念分離:
- ScorePart = 譜面パート（Trumpet 1等）
- ScoreInstrument = MusicXML内の楽器identity
- ScorePlayer = MusicXMLに記載されたplayer
- Performer = 実録音の人物
- TargetDescriptor = 譜面・alignment由来の事実
- PromptVariant = TargetDescriptorから生成したモデル依存のtext prompt

## 3. 現在地

設計済み:
- Slice 0: Project / Asset / Run / Artifact基盤
- Slice 1: MusicXML → Canonical Score
- Slice 2: Score ↔ Audio alignment
- Slice 3: Score/Alignment → NVIDIA separation planning

ローカル雛形では以下を確認済み:
- Python compileall
- pytest
- Slice 0〜2回帰テスト
- Alembic 0001→0004
- separation planning用DB schema
- mock separation経路

未検証:
- Stable Audio Open model access
- Audio-SDS CUDA実行
- peak VRAM
- actual source separation quality
- score-derived promptが品質を改善するか

## 4. 最重要の次作業

`docs/CODEX_TASK_AUDIO_SDS_POC.md`に従い、次の順番で進める。

1. T0 Stable Audio Open smoke
2. T1 latent/decode API確認
3. T2 differentiable decoder + multi-scale STFT backward
4. T3 tiny 2-source separation
5. paper-like 10秒baseline
6. score-derived prompt A/B

T0〜T2が成功するまで、UIや長時間化を優先しない。

## 5. Audio-SDSに関する重要制約

- Audio-SDSは研究方式でありproduction-readyとみなさない。
- NVIDIAを勝手にDemucs/RoFormer等へ主要方式として置換しない。
- 公式完成SDKが確認できていないため、Stable Audio Open公開tooling上で論文方式を再現するPoCとして扱う。
- 論文baselineは10秒clip中心。
- 同一楽器・同pitch・同時発音の個人分離は未解決の高難度課題。
- GPU workerとcore application environmentは分離する。

## 6. Prompt方針

`Trumpet 1`という譜面ラベルをそのまま唯一のpromptにはしない。

例:
```text
ScorePart: Trumpet 1
Descriptor: trumpet / G4-A4-C5 / upper register / active 31.2–34.8 sec
Prompt: solo trumpet playing G4, A4, C5 in sequence
```

最低限比較:
- instrument_only
- instrument_register
- pitch_summary
- note_sequence

## 7. Run / 実験履歴

- failed run/attemptを削除しない。
- retryは新しいrun/attemptとして作る。
- model/version/config/seed/GPU/CUDA/runtime/VRAM/errorを記録する。
- 実装しただけで「検証済み」にしない。

## 8. 読む順序

1. `PROJECT_HANDOFF.md`
2. `CODEX_START.md`
3. `docs/CODEX_SLICE3_NVIDIA_INTEGRATION_SPEC.md`
4. `docs/SEPARATION_ER_V5.md`
5. `docs/CODEX_TASK_AUDIO_SDS_POC.md`

## 9. Codex作業後の報告

必ず次を残す:
1. 変更ファイル
2. 実行コマンド
3. テスト結果
4. GPU/VRAM実測
5. T0/T1/T2の成功・失敗
6. blocker
7. 次に進める条件
8. このhandoffへ反映すべき設計変更
