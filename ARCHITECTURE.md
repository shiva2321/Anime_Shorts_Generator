# Codebase Architecture & Design Guide

**Last Updated**: December 28, 2025

This document provides an in-depth overview of the Anime Clips Generator codebase structure, design patterns, and key algorithms.

---

## 📁 Project Structure

```
Anime_clips/
│
├── 📄 auto_important_clips.py         # Main engine (~1600 lines)
├── 📄 clip_generator_ui.py            # GUI (Tkinter)
├── 📄 check_cuda.py                   # CUDA diagnostics
├── 📄 asr_diagnose.py                 # ASR debugging
├── 📄 self_test.py                    # Integration tests
├── 📄 main.py                         # Legacy entry point
│
├── 📋 requirements.txt                # Python dependencies
├── 📖 README.md                       # Original README
├── 📖 README_NEW.md                   # Comprehensive guide
├── 📖 FUTURE_IMPROVEMENTS.md          # Roadmap
├── 📖 ARCHITECTURE.md                 # This file
│
├── 📁 output/                         # Generated clips
│   └── <VideoName>/
│       ├── *_clip_01_*.mp4
│       ├── *_clip_02_*.mp4
│       ├── *_transcript.srt
│       └── *_summary.txt
│
├── 📁 work/                           # Temporary files
│   ├── overlay_*.ass                  # ASS subtitle files
│   ├── log_*.txt                      # FFmpeg logs
│   └── [temp audio, text files]
│
└── 📁 .git/                           # Version control
```

---

## 🏗️ Main Module: `auto_important_clips.py`

### File Size & Complexity
- **Lines**: ~1600
- **Functions**: 25+
- **Classes**: 2 (dataclasses)
- **Cyclomatic Complexity**: Medium-High

### Section Breakdown

```
Lines 1-20:       Imports & setup
Lines 20-120:     @dataclass OverlayTemplate (5 templates)
Lines 120-130:    @dataclass Candidate
Lines 130-450:    Whisper integration (detect_audio_language, transcribe_video, etc.)
Lines 450-550:    System utilities (font finding, exe resolution, etc.)
Lines 550-700:    Text processing (hooks, punchlines, extraction)
Lines 700-900:    Clip scoring & merging (heuristics, merge_adjacent_candidates)
Lines 900-1100:   Emoji picking & styling (_pick_top_emojis_for_text, _style_bottom_text_ass)
Lines 1100-1400:  Filter chain building (build_filter_chain) [LARGEST FUNCTION]
Lines 1400-1500:  Export & FFmpeg (run_ffmpeg)
Lines 1500-1600:  Main entry point (main) & orchestration
```

---

## 🔑 Key Data Structures

### 1. `OverlayTemplate` (Dataclass)

Defines visual styling for overlay bars and text.

```python
@dataclass
class OverlayTemplate:
    name: str                           # e.g., "anime", "viral_shorts"
    top_bar_color: str                  # Hex color (e.g., "#0066cc")
    bottom_bar_color: str
    top_bar_opacity: float              # 0.0 to 1.0
    bottom_bar_opacity: float
    use_gradient: bool                  # If True, interpolate between colors
    gradient_colors: List[str]          # [start_color, end_color]
    text_style: str                     # "outline", "shadow", "glow"
    text_color: str
    border_width: int                   # For outlined text
    show_branding: bool                 # Show channel name
    show_subscribe: bool                # Show Subscribe CTA
    emoji_prefix: str                   # Static emoji prefix (if not anime)
    emoji_suffix: str                   # Static emoji suffix
    top_text_color_ass: str            # ASS format color (&HAABBGGRR)
    bottom_text_color_ass: str
    bottom_accent_color_ass: str       # For highlighting Subscribe/Like
```

**Predefined Templates**:
- `simple`: Minimal, shadow text
- `viral_shorts`: Gradient blue + emoji
- `anime`: Solid blue + **context-aware emojis**
- `neon_vibes`: Pink/purple gradient
- `cinematic`: Dark professional
- `glass_modern`: Transparent modern

### 2. `Candidate` (Dataclass)

Represents a potential clip extracted from subtitle timeline.

```python
@dataclass
class Candidate:
    start: float                        # Start time (seconds)
    end: float                          # End time (seconds)
    text: str                           # Subtitle text
    score: float = 0.0                  # Engagement score (computed)
```

---

## 🧮 Core Algorithms

### Algorithm 1: Clip Scoring (Heuristic)

**Function**: `compute_heuristic_score(cand, video_duration)`

**Logic**:
```
score = 0.0

1. Punctuation Analysis (+2.0 for "?", +1.5 for "!")
2. Keyword Detection (+1.0 for each keyword: "secret", "crazy", "insane", etc.)
3. Numbers Detection (+0.8 for regex match on digits)
4. Duration Matching:
   - < 5s:  -1.0 (too short)
   - 5-25s: +1.5 (ideal range)
   - > 45s: -0.8 (too long)
5. Text Length: -0.8 if < 15 chars
6. Position Weighting:
   - If in middle 60% of video: +0.5
   - If in first/last 5%: -0.5

Return: score
```

**Time Complexity**: O(n) for n candidates  
**Example**:
- "Wait what?!" (question + "what") → score ≈ 3.0
- "Okay then" (no keywords) → score ≈ 0.0

### Algorithm 2: Clip Selection with Non-Overlap Constraint

**Function**: Main loop in `main()`

**Logic**:
```
1. Score all candidates: O(n)
2. Sort by score descending: O(n log n)
3. Greedy selection:
   For each candidate (highest score first):
     If no temporal overlap with already-selected clips:
       Add to selection
   Until selected.length == num_clips

4. If not enough selected in pass 1:
   Relax overlap tolerance (50% → 30%)
   Retry greedy selection

5. Final fallback:
   If still not enough, duplicate + offset best clips
```

**Time Complexity**: O(n² log n) worst case (n² for overlap checks)  
**Space Complexity**: O(n)

---

## 🎨 Overlay Rendering Pipeline

### Function: `build_filter_chain(clip, idx, cfg, resolution)`

This is the **largest function** (~300 lines). It constructs an FFmpeg filter graph.

#### Phase 1: Video Scaling & Aspect Ratio (Lines 1100-1150)

```
Input: Video at unknown resolution
Target: 1080x1920 (vertical)
Aspect Mode: "fit"

FFmpeg Filter:
split[original][bg];
[bg]scale=1080:1920:force_original_aspect_ratio=increase,
    crop=1080:1920,
    boxblur=10[blurbg];
[original]scale=1080:1920:force_original_aspect_ratio=decrease[scaled];
[blurbg][scaled]overlay=(W-w)/2:(H-h)/2

Result: Blurred background + centered original content (no crop)
```

#### Phase 2: Text Extraction & Formatting (Lines 1150-1250)

```
Input: Candidate.text (e.g., "If I haven't the courage to face this...")
Output: top_txt, bot_txt (formatted for overlay)

Steps:
1. Extract hook & punchline via _smart_extract()
2. Apply template formatting (if {hook}, {punchline} placeholders)
3. Shorten to max_chars (80 for top, 60 for bottom)
4. If anime template:
   - Pick 1-2 context emojis via _pick_top_emojis_for_text()
   - Prepend/append to top_txt
   - Apply ASS color styling to bottom via _style_bottom_text_ass()
5. Wrap text for ASS format (max 22 chars/line for vertical)
6. Strip trailing \N escapes
```

#### Phase 3: ASS Subtitle File Generation (Lines 1250-1350)

```
Output: work/overlay_N.ass (Advanced SubStation Alpha format)

File Structure:
[Script Info]
PlayResX: 1080
PlayResY: 1920
WrapStyle: 2

[V4+ Styles]
Style: Top,Arial,76,&H00FFFFFF,...
Style: Bottom,Arial,68,&H00FFFFFF,...

[Events]
Dialogue: 0,0:00:00.00,9:59:59.99,Top,,0,0,0,,⚔️ If I haven't\Nthe\Ncourage...
Dialogue: 0,0:00:00.00,9:59:59.99,Bottom,,0,0,0,,{\c&H0000CCFF}Subscribe{\c&H00FFFFFF} @MyChannel
```

**Key Points**:
- Alignment 8 = top center, Alignment 2 = bottom center
- Colors in &HAABBGGRR format (BGR, not RGB!)
- \N = line break
- {\c...} = color override tag

#### Phase 4: FFmpeg Filter Construction (Lines 1350-1380)

```
Final Filter Chain:

split[original][bg];
[bg]scale=1080:1920:force_original_aspect_ratio=increase,
    crop=1080:1920,
    boxblur=10[blurbg];
[original]scale=1080:1920:force_original_aspect_ratio=decrease[scaled];
[blurbg][scaled]overlay=(W-w)/2:(H-h)/2,
drawbox=x=0:y=0:w=iw:h=384:color=#0066cc@0.85:t=fill,
drawbox=x=0:y=1536:w=iw:h=384:color=#0066cc@0.85:t=fill,
ass=filename='work/overlay_1.ass'

Result: Video with scaled content, colored bars, and ASS text overlay
```

---

## 🎯 Emoji Selection Logic

### Function: `_pick_top_emojis_for_text(text, rnd, max_emojis=2)`

**Purpose**: Select contextually-relevant emojis for top hook text

**Algorithm**:
```
1. Tokenize text to lowercase
2. Check keyword buckets:
   - fight keywords (⚔️ 🔥 💥 ⚡) → if "fight", "attack", "battle", "jutsu", "punch"
   - run keywords (🏃 💨 ⏳) → if "run", "chase", "escape"
   - secret keywords (🕵️ 🔍 🤫 🧠) → if "secret", "plan", "trap"
   - shock keywords (😳 🤯 😱) → if "shocked", "what", "no way"
   - funny keywords (😂 🤣 💀) → if "funny", "lol", "idiot"
   - sad keywords (🥺 😢 💔) → if "sad", "cry", "sorry"
3. Collect all matching emojis into pool
4. If pool empty, use defaults: [⭐ 🔥 😳 👀 🤯 ⚡]
5. Shuffle pool (seeded by hook_seed + clip_idx for reproducibility)
6. Return top max_emojis
```

**Example**:
- Text: "You're disgusting! That's insane!"
  - Matches: shock (disgusting → 😳) + crazy (insane → 🤯)
  - Result: [😳, 🤯] (shuffled randomly based on seed)

**Determinism**: Set `--hook-seed 42` to get reproducible emoji choices.

---

## 🔤 Text Wrapping for ASS

### Function: `_wrap_for_ass(text, max_chars_per_line, max_lines)`

**Purpose**: Break text into lines for ASS subtitle display without truncation

**Algorithm**:
```
1. If text contains "{" and "}", skip wrapping (already has ASS tags)
2. Collapse whitespace (multiple spaces → single space)
3. Split by words
4. Build lines greedily (add word if fits, else newline):
   current_line = ""
   for word in words:
     if len(current_line) + len(word) + 1 <= max_chars_per_line:
       current_line += word + " "
     else:
       lines.append(current_line)
       current_line = word
   Add final line
5. Limit to max_lines
6. If last line too long, truncate + ellipsis
7. Join with \N (ASS newline)
```

**Example**:
- Input: "If I haven't the courage to face this challenge"
- Max chars: 22, Max lines: 2
- Output: "If I haven't the\Ncourage to face this"

---

## 🎬 Parallel Export Pipeline

### Function: `main()` → FFmpeg export loop (Lines 1520-1570)

**Architecture**:
```
1. Prepare list of export tasks (one per clip)
   For each selected clip:
     - Build filter chain
     - Create FFmpeg command
     - Add (cmd, log_path) to tasks list

2. Launch ThreadPoolExecutor with max_workers:
   Workers = min(--jobs, 3, cpu_count // 2)
   (Capped at 3 for NVENC to avoid GPU memory OOM)

3. For each task, submit to executor:
   executor.submit(run_ffmpeg, cmd, log_path)

4. Collect results:
   for future in as_completed(futures):
     try:
       future.result()
     except Exception as e:
       Log error to stdout
       (Continue processing other clips)

5. Post-process:
   - List actual .mp4 files in output dir
   - Generate summary.txt
   - Clean up work/*.ass, work/log_*.txt
```

**Why Capping at 3 Workers?**
- RTX 3060 has 6GB VRAM
- Each NVENC encode ≈ 1.5-2GB (varies by resolution)
- 4 workers → potential OOM
- 3 workers → stable, good throughput

**Time Complexity**: O(T / W) where T = total encode time, W = workers

---

## 🌍 Language Detection Pipeline

### Function: `detect_audio_language(video_path)`

**Purpose**: Auto-detect if audio is Japanese or English

**Algorithm**:
```
1. Load Whisper tiny model (fastest)
2. Sample 3 points from video:
   - Point 1: 10% through (after intro)
   - Point 2: 40% through
   - Point 3: 70% through
3. For each point:
   - Extract 30-sec audio clip
   - Run Whisper language detection
   - Get confidence scores for all languages
   - Record detected lang + confidence
4. Aggregate votes (weighted by confidence):
   total_votes[ja] = sum of ja confidences
   total_votes[en] = sum of en confidences
5. Return language with highest total_votes

Output: "japanese", "english", or "auto" (fallback)
```

**Why 3 samples?**
- Intro/outro may be silent or have different language
- 3 samples across video = more robust

---

## 📊 Clip Merging & Consolidation

### Function: `merge_adjacent_candidates(candidates, max_clip_len, min_clip_len)`

**Purpose**: Group adjacent subtitles into clip-sized chunks

**Algorithm**:
```
1. Initialize first group from candidates[0]
2. For each subsequent candidate:
   - If adding it keeps duration ≤ max_clip_len:
     Merge (extend group)
   - Else:
     Finalize current group (if ≥ min_clip_len)
     Start new group
3. Combine text from all candidates in group
4. Return list of merged Candidates
```

**Example**:
```
Candidates:
  [0-2s] "Hey"
  [2-5s] "What's up?"
  [5-8s] "I'm here"
  [20-22s] "That's crazy"

With max_clip_len=10s, min_clip_len=2s:

Merged:
  [0-8s] "Hey What's up? I'm here"  (3 subs, 8s total)
  [20-22s] "That's crazy"            (1 sub, 2s total)
```

---

## 🚀 Performance Optimizations

### Current
- **NVENC Support**: Use GPU for H.264 encoding (5-10x faster than CPU)
- **Parallel Workers**: 3 concurrent FFmpeg processes (I/O bound, not CPU bound)
- **Whisper Caching**: Transcripts stored in output/ for reuse
- **Dynamic Font Sizing**: Based on resolution to fit text naturally

### Potential Future
- **Streaming Decode**: Process video in chunks (for huge files)
- **GPU Filter Chains**: Use CUDA for scale/crop/blur instead of FFmpeg CPU
- **Batch Transcription**: Group multiple videos, load model once

---

## 🔍 Testing Strategy

### Current Tests
- `self_test.py`: Smoke tests (imports, basic functions)
- `check_cuda.py`: GPU diagnostics
- `asr_diagnose.py`: Transcription debugging

### Needed Tests
- Unit tests for emoji selection (_pick_top_emojis_for_text)
- Unit tests for text wrapping (_wrap_for_ass)
- Integration tests for full pipeline (end-to-end)
- Regression tests for overlay rendering
- Performance benchmarks

### Example Test
```python
def test_emoji_selection_fight():
    text = "You're going to die in this epic battle!"
    rnd = random.Random(42)
    emojis = _pick_top_emojis_for_text(text, rnd, max_emojis=2)
    assert "⚔️" in emojis or "🔥" in emojis
```

---

## 📋 Configuration & Extensibility

### Adding a New Overlay Template

```python
# Step 1: Define in OVERLAY_TEMPLATES dict
"my_template": OverlayTemplate(
    name="my_template",
    top_bar_color="#FF0000",           # Red
    bottom_bar_color="#00FF00",        # Green
    top_bar_opacity=0.9,
    bottom_bar_opacity=0.85,
    use_gradient=True,
    gradient_colors=["#FF0000", "#0000FF"],
    text_style="outline",
    text_color="#FFFF00",              # Yellow
    border_width=3,
    show_branding=True,
    show_subscribe=True,
    emoji_prefix="🎬 ",
    emoji_suffix=" 🎬",
    top_text_color_ass="&H0000FFFF",   # Yellow
    bottom_text_color_ass="&H0000FFFF",
    bottom_accent_color_ass="&H00FF00FF"  # Magenta
)

# Step 2: Use it
python auto_important_clips.py --template-style my_template ...
```

### Adding Custom Emoji Categories

```python
# In _pick_top_emojis_for_text(), add to buckets list:
buckets = [
    # ...existing buckets...
    (["romance", "love", "kiss"], ["💕", "💋", "✨"]),
    (["magic", "spell"], ["✨", "🔮", "🪄"]),
]
```

---

## 🔗 Module Dependencies

```
auto_important_clips.py
├── Standard Library
│   ├── argparse, os, re, subprocess, sys, textwrap, platform
│   ├── shutil, datetime, warnings, hashlib, time, unicodedata
│   └── random, dataclasses, concurrent.futures, typing
│
├── External (requirements.txt)
│   ├── openai-whisper       (transcription)
│   └── torch, numpy         (Whisper dependencies)
│
└── Internal
    └── None (single-file design)

clip_generator_ui.py
├── tkinter                  (GUI framework)
└── auto_important_clips     (import for processing)

check_cuda.py
└── torch                    (GPU diagnostics)
```

---

## 🎓 Key Learnings & Design Decisions

### 1. Why Single-File Design?
- **Pro**: Easy deployment, no module resolution issues
- **Con**: Large file (~1600 lines), harder to test individual components
- **Future**: Split into `clipgen/` package with submodules

### 2. Why ASS Subtitles Instead of Drawtext?
- **ASS**: Supports color tags, handles emoji better, reusable
- **Drawtext**: Direct FFmpeg filter, but limited color/emoji support, harder to maintain
- **Drawbox**: Used for colored bars (simpler)

### 3. Why Heuristic Scoring Instead of ML?
- **Heuristic**: Fast, interpretable, no training needed
- **ML**: Better accuracy, but requires labeled data & model hosting
- **Hybrid**: Planned for v2.0 (lightweight classifier)

### 4. Why Parallel Threads Instead of Processes?
- **Threads**: Lightweight, share memory (ffmpeg commands are I/O bound)
- **Processes**: Overkill, higher overhead, needed for CPU-bound work
- **Choice**: Threads for I/O (FFmpeg), could use processes for future GPU ops

---

## 🚦 Code Quality Metrics

| Metric | Score | Notes |
|--------|-------|-------|
| Type Coverage | 60% | Could add more hints |
| Docstring Coverage | 50% | Main functions documented |
| Test Coverage | 20% | Smoke tests only |
| Cyclomatic Complexity | Medium | build_filter_chain is complex |
| Code Reuse | Good | Overlay templates, emoji dicts |

---

## 📚 Further Reading

- [FFmpeg Filter Documentation](https://ffmpeg.org/ffmpeg-filters.html)
- [ASS Subtitle Format Spec](https://en.wikipedia.org/wiki/SubStation_Alpha)
- [OpenAI Whisper API](https://github.com/openai/whisper)
- [Tkinter GUI Design](https://docs.python.org/3/library/tkinter.html)

---

**Created**: December 28, 2025  
**Last Updated**: December 28, 2025  
**Maintainers**: [Your Team]

