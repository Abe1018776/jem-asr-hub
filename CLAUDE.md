# JEM Yiddish ASR — 50-Hours Dashboard Project

## Project Overview

Building a "50-Hours Dashboard" for JEM on the `Shloimy15e/yiddish-cleaner` repo (Laravel 12 + Vue 3 + Inertia.js v2). This is a unified UI for JEM staff to prepare 50 hours of Yiddish audio for ASR training — covering mapping, trimming, alignment, cleaning, review, and approval in one streamlined workflow.

**Branch:** `feature/50hr-dashboard` on `Shloimy15e/yiddish-cleaner`

## Key Links

- **Live systems:** kolyid.dynamiq.dev (cleaner), mapping.kohnai.ai (mapper), align.kohnai.ai (alignment)
- **Docs:** https://abe1018776.github.io/yiddish-cleaner-docs/
- **Merge plan:** https://abe1018776.github.io/yiddish-cleaner-docs/technical-merge-plan.html
- **Hub:** https://abe1018776.github.io/jem-asr-hub/
- **RunPod alignment:** github.com/Abe1018776/runpod-stable-ts-yi
- **Detailed plan:** See `50-HOURS-DASHBOARD-PLAN.md` in this repo

## GitHub Repos

| Repo | Role |
|------|------|
| `Shloimy15e/yiddish-cleaner` | **Primary repo — all work goes here.** Laravel 12 + Vue 3 + Inertia. 117 commits. |
| `Abe1018776/jem-asr-hub` | Docs hub (GitHub Pages) |
| `Abe1018776/yiddish-cleaner-docs` | 9 architecture docs on GitHub Pages |
| `Abe1018776/runpod-stable-ts-yi` | RunPod serverless alignment backend (stable-ts + faster-whisper) |
| `Abe1018776/yiddish-mapping` | Original mapping site (vanilla JS, being consolidated) |

## Tech Stack

- **Backend:** Laravel 12, PHP 8.2+, Spatie Media Library, Spatie Permissions, Laravel Reverb (WebSockets), Queue
- **Frontend:** Vue 3, Inertia.js v2, TypeScript, Tailwind CSS 4, Headless UI, Reka UI
- **Alignment:** stable-ts + faster-whisper on RunPod serverless GPU (CUDA 12.1), model: `ivrit-ai/yi-whisper-large-v3-ct2`
- **Cleaning:** 8 rule-based processors (CleanerService) + LLM via OpenRouter/Anthropic/OpenAI/Google/Groq
- **Audio:** wavesurfer.js for waveform display + trimming regions
- **Design:** "Midnight Neon Studio" — dark bg (#0a0a0f), neon accents, RTL support for Yiddish/Hebrew

## Architecture Rules

- **Inertia.js — no separate REST API.** Controllers return Inertia responses with props. Vue pages receive data directly.
- **Driver pattern** for ASR (`AsrManager`) and alignment (`AlignmentManager`). Add new providers without changing calling code.
- **Reuse existing jobs.** `CleanTranscriptionJob`, `AlignTranscriptionJob`, `TranscribeAudioSampleJob` — dispatch these, don't rebuild.
- **Pipeline stage lives on `mapping_entries`**, not on `AudioSample.status`. This avoids breaking the existing benchmark workflow.
- **Benchmark files (is_benchmark=true) CANNOT be used for training.** Enforced at DB + app level.
- **Copy-based integration** from mapping/alignment repos. Not git subtree.

## Database Context

### Existing Models (don't modify)
- `AudioSample` — status: new/in_progress/review/done, has media (audio collection), M2M with Transcription
- `Transcription` — type: base/asr, text_raw/text_clean, alignment_status, wer/cer metrics
- `TranscriptionWord` — word, start_time, end_time, confidence, corrected_word, is_deleted, is_critical_error
- `TranscriptionSegment` — segment_index, text, corrected_text, start/end, confidence
- `AudioSampleTranscription` — M2M pivot with label, notes, linked_by
- `ProcessingRun` — batch job tracking with progress
- `TrainingVersion` — dataset versioning for model training

### New Table: `mapping_entries`
```
id, audio_name, audio_link, est_minutes, transcript_name, transcript_link,
first_line, hebrew_year, hebrew_month, hebrew_day, content_type,
match_confidence, match_reason, suggested_matches (JSON),
audio_sample_id (FK nullable), transcription_id (FK nullable),
pipeline_stage (enum: unmapped/mapped/trimmed/aligned/cleaned/reviewed/approved),
trim_regions (JSON: [{start, end, label}]),
mapped_by (FK nullable), mapped_at, timestamps
```

## Pipeline Stages

```
unmapped -> mapped -> trimmed -> aligned -> cleaned -> reviewed -> approved
```

Each stage means:
1. **unmapped** — audio file exists, no confirmed transcript match
2. **mapped** — audio-transcript pair confirmed
3. **trimmed** — singing/silence regions marked for exclusion
4. **aligned** — word-level alignment complete (timestamps + confidence)
5. **cleaned** — text cleaning applied (rule-based or LLM), diff available
6. **reviewed** — JEM reviewer checked diff, edited inline, accepted changes
7. **approved** — ready for training data export

## Data Scale

- 4,669 total audio files, 1,065 transcripts, 1,809 matched pairs
- **423 files selected for 50 hours** (~2,999 minutes)
- 5 pairs reserved as evaluation set (excluded from training)
- Selection distributed across Hebrew years (5711-5752) and months

## Confidence Scores

- **Alignment:** Per-word float 0.0-1.0. Green >= 0.5, Yellow 0.2-0.5, Red < 0.2
- **Mapping:** Per-match float 0.0-1.0. Green >= 0.7, Yellow 0.4-0.7, Gray < 0.4
- Both use date matching (Hebrew calendar) + keyword/type matching

## Files to Create

### Backend
- `database/migrations/xxxx_create_mapping_entries_table.php`
- `app/Models/MappingEntry.php`
- `app/Http/Controllers/FiftyHourController.php`
- `app/Console/Commands/ImportFiftyHourData.php`
- `app/Http/Requests/BulkActionRequest.php`
- `app/Jobs/BulkAlignJob.php`
- `app/Jobs/BulkCleanJob.php`

### Frontend
- `resources/js/Pages/FiftyHours/Index.vue` — Dashboard with stats, filterable table, bulk actions
- `resources/js/Pages/FiftyHours/Show.vue` — Single-file workflow (trim -> align -> clean -> review -> approve)
- `resources/js/Components/fifty-hours/WaveformTrimmer.vue` — wavesurfer.js waveform + region selection
- `resources/js/Components/fifty-hours/KaraokePlayer.vue` — word-synced playback with confidence colors
- `resources/js/Components/fifty-hours/DiffReview.vue` — selectable diff rows, inline edit, approve/reject
- `resources/js/Components/fifty-hours/PipelineStageChip.vue` — color-coded stage badge

### Modified
- `routes/web.php` — add /fifty-hours routes
- `resources/js/Layouts/AppLayout.vue` — add sidebar nav item
- `package.json` — add wavesurfer.js dependency

## Coding Conventions

- **Vue components:** Composition API with `<script setup lang="ts">`, Tailwind for styling
- **Controllers:** Return `Inertia::render()` with typed props
- **Models:** Eloquent with typed casts, relationship methods, scope methods
- **Jobs:** Extend base job, use `ProcessingRun` for batch tracking, broadcast events via Reverb
- **RTL text:** Always use `dir="rtl"` and appropriate Tailwind RTL utilities for Yiddish/Hebrew content
- **Error handling:** Flash messages via Inertia shared data, toast notifications on frontend

## Future (Not Now)

- Cloudflare MCP server for cleaning via Claude Code skills — build after pipeline is stable
- Singing/silence auto-detection (ML-based)
- Many-to-many audio-transcript splits
- Full repo consolidation (remaining 6 phases of the 9-phase merge plan)
