# Quick Reference Guide

**Fast lookup for common tasks & command patterns**

---

## ⚡ Most Common Commands

### Generate 5 anime clips (1 minute each)
```bash
python auto_important_clips.py \
  --video "episode.mkv" \
  --outdir "output" \
  --num-clips 5 \
  --target-length 60 \
  --template-style anime \
  --channel-name "@MyChannel"
```

### Generate TikTok-friendly clips (15 seconds)
```bash
python auto_important_clips.py \
  --video "episode.mkv" \
  --outdir "tiktok" \
  --num-clips 10 \
  --target-length 15 \
  --template-style viral_shorts \
  --target-res 1080x1920 \
  --channel-name "@MyChannel"
```

### Generate YouTube Shorts (60 seconds)
```bash
python auto_important_clips.py \
  --video "episode.mkv" \
  --outdir "youtube" \
  --num-clips 8 \
  --target-length 60 \
  --template-style anime \
  --aspect-mode fit \
  --target-res 1080x1920
```

### Generate long-form clips (3 minutes each)
```bash
python auto_important_clips.py \
  --video "episode.mkv" \
  --outdir "longform" \
  --num-clips 5 \
  --target-length 180 \
  --template-style cinematic \
  --aspect-mode fill \
  --target-res 1920x1080
```

---

## 🎨 Template Comparison

Quick-pick your favorite:

```
anime        → Blue bars + context emojis (⚔️🔥😳🤯) [BEST for anime]
viral_shorts → Gradient blue + high energy [BEST for TikTok]
neon_vibes   → Pink/purple gradient + trendy [BEST for trending]
cinematic    → Dark professional [BEST for serious content]
simple       → Minimal + clean [BEST for subtle]
glass_modern → Transparent modern [BEST for modern aesthetic]
```

---

## 🎯 Common Customizations

### Change subtitle font size
By default, font size scales with resolution:
- 1080x1920: ~76pt top, ~68pt bottom
- 1920x1080: ~95pt top, ~85pt bottom
- (Computed as `height * 0.04` and `height * 0.035`)

**To override**: Edit `build_filter_chain()` → `fs_base` calculation

### Change accent color for Subscribe button
Default: Orange/gold (`&H0000CCFF`)

In `OVERLAY_TEMPLATES["anime"]`:
```python
bottom_accent_color_ass="&H0000CCFF"  # Change this value
# &HAABBGGRR format (BGR not RGB!)
# Examples:
# &H0000FF00 = Green
# &H00FF0000 = Blue
# &H000000FF = Red
# &H00FF00FF = Magenta
```

### Add custom emoji categories
In `_pick_top_emojis_for_text()`, add to `buckets`:
```python
buckets = [
    # ... existing ...
    (["magic", "spell", "power"], ["✨", "🔮", "⚡", "🪄"]),
    (["romance", "love"], ["💕", "💋", "✨"]),
]
```

### Change clip scoring logic
In `compute_heuristic_score()`, adjust weights:
```python
# Increase importance of questions
if "?" in text:
    score += 3.0  # Changed from 2.0

# Add custom keywords
if "example_keyword" in lowered:
    score += 2.0
```

---

## 🐛 Troubleshooting Checklist

| Problem | Diagnosis | Fix |
|---------|-----------|-----|
| **No clips exported** | Check `work/log_*.txt` | FFmpeg error → check syntax |
| **Emojis appear as ❓** | Not anime template? | Use `--template-style anime` |
| **"Subsc..." in bottom** | Text truncation (old bug) | Rebuild with latest code |
| **Slow encoding** | CPU encoding (libx264)? | Try `--encoder h264_nvenc` |
| **GPU out of memory** | Too many parallel jobs | Reduce `--jobs` to 2 |
| **Wrong language detected** | Auto-detection failed | Set `--language english` manually |
| **Clips too short/long** | Wrong --target-length? | Verify duration is **per clip** |

---

## 📊 Output File Locations

After running, find your clips here:

```
output/
└── <VideoName>/
    ├── *_clip_01_*.mp4          ← Clip 1 video
    ├── *_clip_02_*.mp4          ← Clip 2 video
    ├── ... (more clips)
    ├── *_transcript.srt         ← Full subtitle transcript
    └── *_summary.txt            ← Metadata & timing
```

---

## ⌚ Performance Expectations

| Task | CPU | NVIDIA GPU |
|------|-----|-----------|
| Transcribe 23-min video | 2-3 min | 1-2 min |
| Export 5 @ 1080x1920 | 3-5 min | 15-30 sec |
| Generate emoji + text | <1 sec | <1 sec |
| **Total for 5 clips** | **5-8 min** | **2-3 min** |

**GPU can be 2-3x faster!**

---

## 🔐 Security Tips

1. **API Keys**: Never commit to git
   ```bash
   # Store in .env.local (not in repo)
   $env:OPENAI_API_KEY = "sk-..."
   ```

2. **File Paths**: Use quotes for spaces
   ```bash
   --video "C:\Users\My User\Videos\Episode 1.mkv"
   ```

3. **Output Paths**: Ensure write permissions
   ```bash
   # Test write access before running long jobs
   echo "test" > "output/test.txt"
   ```

---

## 🎓 Common Parameters Explained

### `--num-clips 5`
→ Generate **5 separate video files** (clip_01, clip_02, ..., clip_05)

### `--target-length 180`
→ **Each clip is 3 minutes long** (180 seconds)
- 5 clips × 3 min = 15 min total output ✅
- **NOT** "divide video into 5 clips"

### `--template-style anime`
→ Use anime-specific overlay (blue bars + context emojis)

### `--aspect-mode fit`
→ Preserve full video without cropping (adds padding if needed)

### `--aspect-mode fill`
→ Crop to fill frame completely (may cut edges)

### `--channel-name "@MyChannel"`
→ Bottom text: "Subscribe @MyChannel" (with colored "Subscribe")

### `--hook-seed 42`
→ Deterministic emoji selection (same seed = same emojis)
- Useful for reproducibility
- Increment to shuffle: 42 → 43 → 44

---

## 🚀 CLI Shorthand

Most common without full paths (assuming videos in current dir):

```bash
# Minimal (uses defaults)
python auto_important_clips.py --video "ep1.mkv" --outdir "out"

# Quick anime
python auto_important_clips.py --video "ep1.mkv" --outdir "out" \
  --template-style anime --num-clips 5 --target-length 60

# Quick TikTok
python auto_important_clips.py --video "ep1.mkv" --outdir "out" \
  --num-clips 20 --target-length 15
```

---

## 📖 Documentation Map

| Doc | Best For |
|-----|----------|
| **README_NEW.md** | Getting started + features overview |
| **ARCHITECTURE.md** | Understanding code structure + algorithms |
| **FUTURE_IMPROVEMENTS.md** | Roadmap + what's planned |
| **QUICK_REFERENCE.md** | Fast lookup (this file) |

---

## 🆘 Need Help?

1. **Check logs**: `work/log_*.txt` has FFmpeg errors
2. **Run diagnostics**: `python check_cuda.py` or `python asr_diagnose.py`
3. **Review examples**: See README_NEW.md examples section
4. **Check architecture**: ARCHITECTURE.md explains algorithms

---

## 💡 Pro Tips

- **Batch multiple videos**: Write a loop calling CLI multiple times
- **Dry run**: Use `--num-clips 1` to test before full processing
- **Reuse transcripts**: Puts them in output/, reused if you re-run same video
- **Parallel processing**: Increase `--jobs` gradually (3-4 is safe for NVENC)
- **Emoji variety**: Different `--hook-seed` values = different emojis

---

**Last Updated**: December 28, 2025

