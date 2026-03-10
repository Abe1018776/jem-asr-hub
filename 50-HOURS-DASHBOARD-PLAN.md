# 50-Hours Dashboard — Implementation Plan

## Branch: `feature/50hr-dashboard` on `Shloimy15e/yiddish-cleaner`

---

## What Already Exists (we build on this)

| Component | Status | Location |
|-----------|--------|----------|
| Audio sample management + status workflow | Done | `app/Models/AudioSample.php` |
| Transcription model (raw/clean/words/segments) | Done | `app/Models/Transcription.php` |
| M2M audio-transcript linking (pivot table) | Done | `audio_sample_transcription` |
| Cleaning pipeline (8 processors + LLM) | Done | `app/Services/Cleaning/` |
| Alignment driver pattern (RunPod + local) | Done | `app/Services/Alignment/` |
| Word-level review (TranscriptionWord model) | Done | `app/Models/TranscriptionWord.php` |
| DiffService for clean vs raw | Done | `app/Services/Cleaning/DiffService.php` |
| AudioPlayer component | Done | `resources/js/Components/AudioPlayer.vue` |
| TranscriptionReview component (87KB!) | Done | `resources/js/Components/transcriptions/` |
| Training version system | Done | `app/Models/TrainingVersion.php` |
| WebSocket real-time updates (Reverb) | Done | Broadcasting setup |

## What We Need to Build

### Phase 1: Data Import (Day 1, ~2 hours)

**Goal:** Get the 423 fifty-hour files into the database.

1. **Migration: `mapping_entries` table**
   ```
   id, audio_name, audio_link, est_minutes, transcript_name, transcript_link,
   first_line, hebrew_year, hebrew_month, hebrew_day, content_type,
   match_confidence, match_reason, suggested_matches (JSON),
   audio_sample_id (FK nullable), transcription_id (FK nullable),
   pipeline_stage (enum: unmapped/mapped/trimmed/aligned/cleaned/reviewed/approved),
   mapped_by (FK nullable), mapped_at, timestamps
   ```

2. **Artisan command: `php artisan mapping:import-50hr`**
   - Fetch `data.json` from mapping.kohnai.ai (or bundle it)
   - Insert the 423 `selected` records into `mapping_entries`
   - Also import the full 1,809 matched + unmatched for future use
   - Download audio files from Google Drive links → attach via Spatie Media Library
   - Create AudioSample + Transcription records, link via pivot

3. **Model: `MappingEntry`** with relationships to AudioSample, Transcription, User

### Phase 2: Dashboard Page (Day 1-2, ~3 hours)

**Goal:** JEM opens one page and sees everything.

4. **Route:** `GET /fifty-hours` → `FiftyHourController@index`

5. **Vue Page: `pages/FiftyHours/Index.vue`** — The main dashboard
   - **Stats bar:** Total files, minutes, by stage (unmapped/mapped/trimmed/aligned/cleaned/reviewed/approved)
   - **Filterable table:** All 423 files with columns:
     - Audio name (linked)
     - Transcript name (linked)
     - Duration (minutes)
     - Hebrew year/month
     - Content type badge
     - **Pipeline stage** (color-coded chip)
     - Actions dropdown
   - **Filters:** Stage, year, month, type, search
   - **Bulk actions:** Select all / deselect, bulk align, bulk clean, bulk approve

### Phase 3: Single-File Workflow Page (Day 2-3, ~5 hours)

**Goal:** Click any row → full workflow for that file.

6. **Route:** `GET /fifty-hours/{mappingEntry}` → `FiftyHourController@show`

7. **Vue Page: `pages/FiftyHours/Show.vue`** — The workhorse page with these sections:

   **A. Audio Player + Trimming**
   - Waveform display (wavesurfer.js)
   - Click-drag to select range (mark singing/silence for exclusion)
   - Trim regions saved to `mapping_entries.trim_regions` (JSON: [{start, end, label}])
   - Play selected region only

   **B. Mapping Panel** (if not yet mapped)
   - Current suggested matches with confidence scores
   - Search modal (all transcripts)
   - One-click to confirm/change mapping

   **C. Alignment Section**
   - "Run Alignment" button → dispatches `AlignTranscriptionJob` with RunPod
   - Real-time progress via WebSocket
   - Results: word chips with confidence colors (green/yellow/red)
   - **Karaoke playback** — words highlight in sync with audio
   - Click any word → seek audio to that timestamp

   **D. Cleaning Section**
   - Toggle: Rule-based (preset selector) vs LLM (provider selector)
   - "Run Cleaning" button → dispatches `CleanTranscriptionJob`
   - **Diff view:** Side-by-side or inline diff of raw vs cleaned
   - Each diff row is selectable (checkbox)
   - Select all / deselect all
   - **Inline editing** — click any row to edit the cleaned text
   - Accept/reject individual changes

   **E. Review & Approve**
   - Summary: confidence score distribution, clean rate, flagged words
   - "Approve for Training" button → sets `pipeline_stage = approved`
   - "Flag for Re-review" with notes field
   - Approved files get `flagged_for_training = true` on their Transcription

### Phase 4: Bulk Operations (Day 3-4, ~2 hours)

8. **Bulk align** — Select multiple files from dashboard, one-click align all
9. **Bulk clean** — Select multiple, choose preset or LLM, clean all
10. **Bulk approve** — Select reviewed files, approve all for training
11. **Processing progress** — Real-time WebSocket updates for bulk jobs

### Phase 5: Polish (Day 4, ~2 hours)

12. **Sidebar nav item** — "50 Hours" with badge showing pending count
13. **Pipeline progress bar** on dashboard (visual: how many at each stage)
14. **Export** — CSV/JSON of approved training data
15. **Keyboard shortcuts** — Arrow keys to navigate files, Enter to approve

---

## Architecture Decisions

| Decision | Why |
|----------|-----|
| New `mapping_entries` table | Bridges external data (data.json) with internal models (AudioSample, Transcription). Tracks pipeline stage independently. |
| Build in existing Laravel+Vue app | Not Lovable. Audio waveforms, karaoke, diff views need precise control. Inertia gives seamless backend access. |
| Reuse existing alignment/cleaning jobs | Don't rebuild — dispatch the same jobs with the same drivers. |
| wavesurfer.js for audio trimming | Battle-tested waveform library, supports regions plugin for range selection. |
| Pipeline stage on mapping_entries | Separate from AudioSample.status to avoid breaking existing benchmark workflow. |
| RunPod for alignment (not local) | GPU-dependent. Already deployed and working. |
| Cloudflare MCP for cleaning | Defer to Phase 6. Get core pipeline working first, then expose as MCP tool. |

---

## File Inventory (new files to create)

### Backend (~8 files)
- `database/migrations/xxxx_create_mapping_entries_table.php`
- `app/Models/MappingEntry.php`
- `app/Http/Controllers/FiftyHourController.php`
- `app/Console/Commands/ImportFiftyHourData.php`
- `routes/web.php` (add routes)
- `app/Http/Requests/BulkActionRequest.php`
- `app/Jobs/BulkAlignJob.php`
- `app/Jobs/BulkCleanJob.php`

### Frontend (~6 files)
- `resources/js/Pages/FiftyHours/Index.vue` (dashboard)
- `resources/js/Pages/FiftyHours/Show.vue` (single-file workflow)
- `resources/js/Components/fifty-hours/WaveformTrimmer.vue`
- `resources/js/Components/fifty-hours/KaraokePlayer.vue`
- `resources/js/Components/fifty-hours/DiffReview.vue`
- `resources/js/Components/fifty-hours/PipelineStageChip.vue`

### Total: ~14 new files, ~4 modified files

---

## Estimated Timeline

| Phase | Effort | Can Parallel? |
|-------|--------|---------------|
| Phase 1: Data Import | 2 hrs | — |
| Phase 2: Dashboard | 3 hrs | After Phase 1 |
| Phase 3: Workflow Page | 5 hrs | After Phase 1 |
| Phase 4: Bulk Ops | 2 hrs | After Phase 2+3 |
| Phase 5: Polish | 2 hrs | After Phase 4 |
| **Total** | **~14 hrs** | |

---

## Dependencies to Install

```bash
npm install wavesurfer.js @wavesurfer/regions   # Audio waveform + trimming
```

Everything else (Vue 3, Inertia, Tailwind, Headless UI) already exists in the project.
