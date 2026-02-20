# GenAI Assignment:

**Evaluation Criteria**

We will score your submission on:

* Clarity and practicality of architecture
* Robust JSON schema design
* Prompt quality (zero-shot, reliable, minimal hallucination risk)
* Handling of ambiguity + user review flow
* Bulk generation thinking (errors, naming, report)

## Problem 1: **Proposal for “Video-to-Notes”**

We have a local folder of long videos (3–4 hours each, 200MB+). Watching them fully is slow. We need an automated way to generate a “summary package” per video: **Summary.md** + highlight clips + screenshots, all organized per video. [READ MORE ABOUT THE PROJECT](./Video-summary-platform.md)

### **Task**

Prepare a **pre-processed solution proposal** comparing  **three approaches** **:**

1. **Online/Cloud-Based (Already Available Solutions)**
2. **Build Our Own Using LLM APIs (Hybrid: local media processing + cloud LLM)**
3. **Build Fully Offline Using Open-Source Models (Local transcription + local LLM + pipeline)**

No code required. We want a **clear, practical proposal** with architecture and tradeoffs.

### Your Solution for problem 1:

# Approach Comparison
## 1. Cloud / SaaS-Based Solution
**How it works:** Upload videos to an AI-powered SaaS platform (e.g. Grain, Otter.ai, Fireflies, AssemblyAI video pipeline). The platform handles transcription, summarisation, and chapter generation automatically through its hosted API.

### Architecture
```
Local Video → Upload → SaaS Transcription → SaaS Summarization → Export → Post-processing
```
| Stage            | SaaS Provider Workflow |
| ---------------- | ---------------------- |
| Upload           | Videos uploaded via API or web UI to the SaaS provider. |
| Transcription    | Provider's ASR engine (e.g., Whisper-based) generates time-coded transcript. |
| Summarisation    | Provider's LLM layer produces summary and highlights with timestamps. |
| Asset Extraction | Limited: some tools export clip URLs; screenshots not always supported. |
| Output           | JSON / Markdown export via provider API. Custom folder structure requires post-processing script. |


### Pros
* Fastest to implement
* High ASR quality
* Minimal infrastructure
### Cons
* High per-minute cost for 3–4 hour videos
* Privacy concerns (videos leave local environment)
* Hard blocker for clip/screenshot extraction
  > Most SaaS tools (Grain, Otter, Fireflies) do not expose an API to extract raw video clips at custom timestamps. We get a shareable link at best not a downloadable MP4 segment. Screenshot extraction at specific frames is not supported at all in any major SaaS offering. This alone makes SaaS non-viable for this use case's core output requirement.
* Difficult to customise output structure

### Best For
Small, non-sensitive workloads where speed matters more than cost control.

## 2. Hybrid Solution
Local media processing + Cloud LLM for structured summarization
**How it works**: All media processing (transcription, clip cutting, screenshot extraction) runs locally using open-source tools (Whisper, FFmpeg, OpenCV). Only the text transcript is sent to a cloud LLM (OpenAI GPT-4o / Gemini 1.5 Pro) for intelligent summarisation and highlight detection. This is my recommended approach.
### High-Level Architecture
```
Batch Orchestrator
    ↓
Proxy Video Generation (Bitrate Laddering)
    ↓
Audio Extraction
    ↓
Local Transcription (faster-whisper)
    ↓
Sliding Window Chunking
    ↓
Recursive LLM Summarization
    ↓
Highlight Merge + Confidence Scoring
    ↓
Clip Extraction
    ↓
Screenshot Extraction
    ↓
Markdown + Report Generation
```

| #  | Stage                     | Description |
|----|----------------------------|-------------|
| 1  | Ingest                     | Folder watcher detects new video files → adds to SQLite job queue with status INGESTED |
| 2  | Proxy Generation           | FFmpeg transcodes to 480p proxy (libx264 veryfast). Used for all frame sampling & screenshots. Original preserved for final clip cuts only. Saves 60–80% CPU vs processing at source resolution. |
| 3  | Audio Extraction           | FFmpeg extracts audio track to mono 16 kHz WAV — optimal format for Whisper input. |
| 4  | Transcription              | faster-whisper (large-v3, GPU). Produces word-level timestamped JSON. Batch mode: multiple videos transcribed concurrently. Status → TRANSCRIBED. |
| 5  | Sliding Window Chunk       | Transcript split into 20–30 min chunks with 1–2 min overlap at silence boundaries. Word timestamps preserved across chunks. |
| 6  | Chunk Summarisation        | Each chunk → GPT-4o / Gemini 1.5 Pro (async, parallel calls). Returns structured JSON per chunk: chapter title, highlight timestamps, key points, confidence score. Status → CHUNK_SUMMARIZED. |
| 7  | Meta-Summary Pass          | All chunk JSONs merged → second LLM call produces unified highlight ranking. Deduplicates overlapping highlights (±30 s window). Enforces minimum clip duration. Status → META_SUMMARIZED. |
| 8  | Clip Extraction            | FFmpeg stream-copies highlight segments from original (full-res) video using precise start/end timestamps (±30 s padding). Lossless, fast. Status → CLIPPED. |
| 9  | Screenshot Extraction      | FFmpeg -ss -vframes 1 on proxy file extracts keyframe JPG at each highlight timestamp. Fast single-frame decode. Status → SCREENSHOTS_DONE. |
| 10 | Summary Build              | Summary.md assembled from meta-summary JSON + relative asset paths. processing_report.json written with token usage, cost estimate, stage timings. Status → COMPLETED. |

### Production Optimizations
1. **Bitrate Laddering (Compute Optimization)**
   Instead of processing full-resolution video:
   ```
   ffmpeg -i input.mp4 -vf scale=-2:480 -c:v libx264 -preset veryfast proxy.mp4
   ```
   * Proxy (480p) used for screenshots and frame sampling
   * Original video used only for final clip extraction
   * Reduces CPU/GPU usage by 60–80%
   * Improves batch throughput significantly
   
2. **Sliding Window + Recursive Summarization**
   **Problem**: 3–4 hour transcript exceeds safe context window limits.
   **Solution**
   **Step 1 — Sliding Window Transcription**
   * 20–30 minute transcript chunks
   * 1–2 minute overlap
   * Word-level timestamps preserved
   **Step 2 — Level 1 Summaries**
   Each chunk → structured JSON summary
   **Step 3 — Meta-Summary Pass**
   Summaries of summaries → unified highlight ranking
   **Step 4 — Deduplication**
   * Merge overlapping highlights (±30 sec)
   * Remove duplicates
   * Enforce minimum clip duration
     
   This ensures:
   * No mid-video information loss
   * Balanced coverage
   * Reduced hallucination risk

  3. **Timestamp Confidence Scoring**
     Each highlight includes a computed confidence score:
```
  {
  "title": "Core Architecture Decision",
  "start_time": "01:12:33",
  "end_time": "01:14:02",
  "confidence_score": 0.87,
  "confidence_reason": "High transcript clarity, clean silence boundaries"
  }
```
**Confidence factors:**
* Whisper transcription confidence
* Silence detection at boundaries
* Semantic completeness
* Cross-chunk consistency
This improves trust and usability.

4. **Token Cost Control Strategy**
   To control API cost for long transcripts:
   * Remove filler words before LLM call
   * Use chunk-level summarization before meta-summary
   * Temperature = 0 for deterministic JSON
   * Strict JSON schema enforcement
   * Avoid verbose model outputs
     
Estimated cost per 3-hour video:
*~$1–3 depending on model and chunk count*

5. **Idempotent Batch Processing**
   Each video maintains processing state:
```
   {
  "video_id": "video_001",
  "status": "TRANSCRIBED",
  "last_successful_stage": "chunk_summarization",
  "retry_count": 1
   }
```
**Pipeline states:**
* INGESTED
* PROXY_GENERATED
* TRANSCRIBED
* CHUNK_SUMMARIZED
* META_SUMMARIZED
* CLIPPED
* SCREENSHOTS_DONE
* COMPLETED
* FAILED
 
If the pipeline is interrupted at any stage, the batch orchestrator resumes from the last completed stage — no reprocessing of already-finished steps
```
INGESTED
   ↓
PROXY_GENERATED 
   ↓
TRANSCRIBED
   ↓
CHUNK_SUMMARIZED
   ↓
META_SUMMARIZED 
   ↓
CLIPPED
   ↓
SCREENSHOTS_DONE → FAILED (resumable from last stage)
   ↓
COMPLETED 
                                                                                    
```

6. **Observability & Cost Reporting**
   Each video generates:
   ```
   processing_report.json 
   ```
Includes:
* Duration
* Stage-wise processing time
* LLM token usage
* Estimated API cost
* Highlight count

Example:
```
Video: product_masterclass.mp4
Duration: 3h 18m
Transcription Time: 12m
LLM Tokens: 58,400
Estimated Cost: $1.94
Highlights: 11
Avg Confidence: 0.83
```
### Output Folder Structure
```
output/<video_name>/
├── Summary.md
├── processing_report.json
├── clips/
│   ├── highlight_01.mp4
│   └── highlight_02.mp4
└── screenshots/
    ├── highlight_01.jpg
    └── highlight_02.jpg
```
## 3. Fully Offline Solution
All components run locally:
* Transcription: faster-whisper
* LLM: LLaMA / Mistral (via Ollama or llama.cpp)
* Media: FFmpeg
### Architecture
| Component        | Fully Offline (Approach 3) Description |
|------------------|----------------------------------------|
| Transcription    | faster-whisper (large-v3) on GPU — same as Approach 2. |
| LLM              | Local model served via Ollama or llama.cpp. Recommended models: Llama-3.1 70B (if GPU VRAM ≥ 48 GB) or Mistral-7B / Gemma-2 27B for lighter setups. |
| Prompt           | Same chunked approach as Approach 2. JSON output enforced via grammar-constrained decoding (llama.cpp grammar / Ollama `format: json`). |
| Video Processing | FFmpeg + OpenCV — identical to Approach 2. |
| Limitations      | Local LLM quality is lower than GPT-4o for complex summarisation, especially for domain-specific or technical content. |

## Model Selection by Hardware
| GPU VRAM                     | Recommended Model              | Quality vs Hybrid            | Throughput (3-hr video) |
|-------------------------------|--------------------------------|------------------------------|--------------------------|
| ≥ 48 GB (A100 / H100)        | Llama-3.1 70B (Q4)             | ~85% of GPT-4o quality       | ~20 min                  |
| 24–48 GB (A40 / 3090)        | Gemma-2 27B (Q4)               | ~75% of GPT-4o quality       | ~35 min                  |
| 12–24 GB (4090 / 3080)       | Mistral-7B or Phi-3 Mini       | ~60–65% of GPT-4o quality    | ~50 min                  |
| < 12 GB                      | Not recommended                | Quality insufficient for production | —                |


### Pros
* Maximum privacy
* No API cost
* Suitable for air-gapped environments
### Cons
* Requires strong GPU
* Lower summarization quality vs GPT-4 class models
* Higher setup complexity
  
### JSON Reliability Note
Local LLMs (LLaMA/Mistral) are less reliable at strict JSON output than GPT-4 class models.
Mitigation: Use **grammar-constrained decoding** to enforce schema at the token level:
- **llama.cpp**: pass a `.gbnf` grammar file that matches your highlight JSON schema
- **Ollama**: use `format: "json"` in the API call

This ensures the offline pipeline produces parseable output without post-processing fallbacks.

> [!NOTE]
> When to choose Offline: Regulated or confidential content (legal, medical, financial) where no data upload is permitted, and the organisation owns ≥ 24 GB VRAM GPU hardware. Expect ~75–85% of Hybrid quality at the cost of higher setup complexity.
### Decision Matrix

| Factor               | SaaS     | Hybrid (Recommended) | Offline            |
| -------------------- | -------- | -------------------- | ------------------ |
| Privacy              | Low      | High                 | Maximum            |
| Cost Control         | Low      | High                 | High               |
| Quality              | High     | Highest              | Medium             |
| Customization        | Limited  | Full                 | Full               |
| Batch Reliability    | Moderate | High                 | Hardware-dependent |
| Production Evolution | Low      | High                 | Medium             |

## **My Recommendation**
The <ins>**Hybrid Architecture**</ins> with recursive summarization, bitrate laddering, and confidence scoring provides the best balance of:
* Cost efficiency
* Privacy
* High-quality summaries
* Deterministic clip alignment
* Batch reliability
* Production-readiness
**It is scalable from a small POC to a production-grade media processing pipeline.**


## Problem 2: **Zero-Shot Prompt to generate 3 LinkedIn Post**

Design a **single zero-shot prompt** that takes a user’s persona configuration + a topic and generates **3 LinkedIn post drafts** in **3 distinct styles**, each aligned to the user’s voice and constraints. The output must be structured so the app can: show 3 drafts to the user. Assume we are consuming **OpenAI API / Gemini API** with **one prompt call** (no fine-tuning). Your prompt must reliably produce valid, structured output. [READ MORE ABOUT THE PROJECT](./linkedin-automation.md)

**TASK:** Write a prompt that can work.

### Your Solution for problem 2:

You need to put your solution here.

## Problem 3: **Smart DOCX Template → Bulk DOCX/PDF Generator (Proposal + Prompt)**

Users have many Word documents that act like templates (offer letters, certificates, invoices, contracts). They repeatedly change only a few fields (name, date, amount, address, role, etc.). Doing this manually is slow and error-prone, especially in bulk. [READ MORE ABOUT THE PROJECT](./docs-template-output-generation.md)

We want a system that:

1. Converts an uploaded **DOCX** into a reusable **template** by identifying editable fields.
2. Supports **single generation** (form-fill → DOCX/PDF download).
3. Supports **bulk generation** via **Excel/Google Sheet** rows.

### **Task (No coding)**

Submit a **proposal** for building this system using GenAI (OpenAI/Gemini) for “template field detection” and “field schema generation”. We want a practical design, not code.

### Your Solution for problem 3:

You need to put your solution here.

## Problem 4: Architecture Proposal for 5-Min Character Video Series Generator

We want to build a system that helps a user create a short video series (around **5 minutes per episode**) using predefined characters. The user defines characters (image + personality) and relationships once, then provides a short story/situation for an episode. The system outputs a complete episode package and optionally a final video. [READ MORE ABOUT THE PROJECT](./char-based-video-generation.md)

### **Task**

Create a **small, clear architecture proposal** (no code, no prompts) describing how you would design and build this system.

### Your Solution for problem 4:

You need to put your solution here.
