# CODEX_SLICE3_NVIDIA_INTEGRATION_SPEC.md

- Project: 音楽聞き分けアプリ
- Slice: 3 — Score/Alignment → NVIDIA Separation Integration
- Version: 0.8-design
- Updated: 2026-09-27
- Depends on: Slice 0, 1, 2
- Backend research task: `CODEX_TASK_AUDIO_SDS_POC.md`

## 1. Goal

Score/Alignmentから再現可能なAudio-SDS分離計画を生成し、
backend adapterへ渡してoutputを追跡できるようにする。

Slice 3のDoDは「NVIDIA Audio-SDSが高品質に動くこと」ではない。

DoD:
- segmentation
- target descriptor
- prompt variant
- separation run/attempt
- output artifact lineage
が完成し、mock backendでE2Eできること。

## 2. 新規要件

### REQ-SEPPLAN-001
Aligned ScoreEventからSeparation Target Descriptorを作る。

### REQ-SEPPLAN-002
Audio-SDS向けに10秒以下SegmentJobを作る。

### REQ-PROMPT-001
同一TargetDescriptorから複数prompt strategyを生成する。

### REQ-PROMPT-002
譜面part labelと音響promptを別データにする。

### REQ-SEPARATE-007
segment単位のbackend attemptを履歴化する。

### REQ-SEPARATE-008
NVIDIA backendはcore processのheavy dependencyから分離可能にする。

## 3. Core entities

正: `SEPARATION_ER_V5.md`

- SeparationRun
- SegmentJob
- TargetDescriptor
- PromptVariant
- SeparationAttempt
- SeparationTarget
- SeparatedSegment

## 4. Segment planner

default:
```yaml
hard_max_duration_sec: 10.0
target_duration_sec: 7.0
context_left_sec: 0.5
context_right_sec: 0.5
overlap_sec: 0.5
min_target_activity_sec: 0.25
low_confidence_extra_margin_sec: 0.5
```

v1:
1. target partのaudible pitched AlignedScoreEventをtime順。
2. active interval作成。
3. gapをboundary候補。
4. 7秒前後でgroup。
5. context追加。
6. 10秒cap。
7. audio boundsへclip。
8. target activityが閾値未満ならjobなし。

## 5. Descriptor builder

計算:
- active onset/end
- sounding pitch min/max/median
- event count
- note sequence
- activity ratio
- max simultaneous notes
- source part label
- instrument label

個人名はmodel promptへdefault投入しない。

## 6. Prompt strategies

- `instrument_only`
- `instrument_register`
- `pitch_summary`
- `note_sequence`
- 将来 `score_detailed`

最初から1戦略に固定しない。

## 7. Complement

2-source baseline:
- primary target
- complement

complement strategy:
- `generic_other`
- 将来 `known_remaining_instruments`

generic:
`remaining ensemble accompaniment and other instruments`

## 8. Port v2

```python
@dataclass(frozen=True)
class SeparationChannel:
    output_key: str
    prompt: str
    descriptor_id: str

@dataclass(frozen=True)
class SeparationRequestV2:
    input_audio: Path
    segment_start_sec: float
    segment_end_sec: float
    channels: tuple[SeparationChannel, ...]
    backend_profile: str
    config: Mapping[str, Any]
    seed: int | None
    work_dir: Path
```

## 9. Worker boundary

Core環境とAudio-SDS worker環境を分離。

filesystem envelope v1:
```text
work/<attempt_id>/
  request.json
  input.wav
  outputs/
  response.json
  stderr.log
```

## 10. Backend profiles

- `mock-v1`
- `audio-sds-tiny-v1`
- `audio-sds-paper-v1`
- `audio-sds-microbatch-v1`

profile名とeffective configを両方保存。

## 11. Attempt retry

failed attemptを同rowでretryしない。

new attempt:
- parent_attempt_id
- attempt_no +1

## 12. Tests

Unit:
- segment <=10 sec
- context/audio bounds
- no target activity => no job
- descriptor pitch range
- note naming
- prompt determinism
- filesystem request envelope

Integration:
- migration 0004
- synthetic score/alignment → segment
- descriptor → multiple prompt variants
- MockSeparatorV2
- Slice 0–2 regression

GPUをcore CIへ必須にしない。

## 13. Acceptance scenario

20 sec mixture。
Trumpet 1 activity:
- 2–6 sec
- 9–14 sec

Expected:
- >=2 SegmentJobs
- each <=10 sec
- TargetDescriptor
- complement
- instrument_only + score-derived prompt
- mock output lineage

## 14. Do not do

- Audio-SDSをproduction-readyと呼ばない
- torchをcoreへhard dependencyにしない
- `Trumpet 1`だけをpromptにしない
- part番号からPerformerを推定しない
- long-form stitchingはまだしない
