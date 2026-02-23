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

### Problem 2 — Zero-Shot Prompt: LinkedIn Post Generator

A single prompt call (no fine-tuning, no multi-turn) that accepts a user persona configuration + a topic and returns 3 structurally distinct, LinkedIn-ready post drafts as directly parseable JSON.

## Design Decisions
| Decision | Rationale |
|----------|-----------|
| Structured Output | Prompt mandates a strict JSON schema. The app calls `JSON.parse()` directly — no regex scraping, no free-text post-processing. |
| Zero-Shot Reliability | Explicit constraints + a worked schema example (few-shot schema, not few-shot content) reduces hallucination risk without inflating token cost with full examples. |
| Style Separation | Three named styles are defined with concrete structural rules and hard word count bounds. Prevents the model returning three tonal variations of the same structure. |
| Enum-Constrained Style Field ▸ NEW | The `style` field in the JSON schema is shown as an enum directly inside the schema definition — not just mentioned in the rules. The model sees allowed values where it fills them in, which is more reliable than a rule listed elsewhere. |
| Persona Injection | All persona fields injected as a typed, structured block with inline examples. Avoids vague “write in my voice” instructions that models routinely under-follow. |
| Do/Don't Enforcement | `do_rules` and `dont_rules` serialised as numbered lists. LLMs comply more reliably with numbered constraints than free-form narrative instructions. |
| Hallucination Guard | Explicit ban: the model must not invent statistics, quotes, or external references unless they appear in `topic_context`. Named specifically, not bundled into a general “be accurate” instruction. |
| API-Level JSON Enforcement | In addition to prompt instructions, the API call itself enforces JSON mode: `response_format: { type: 'json_object' }` (OpenAI) or `responseMimeType: 'application/json'` (Gemini). Prompt + API constraint together eliminate malformed output. |

## The Prompt
### SYSTEM PROMPT
```
You are a professional LinkedIn ghostwriter and content strategist.
Your only job is to produce valid JSON — nothing else.
Do not include any text, explanation, or markdown outside the JSON object.
Do not wrap the output in code fences.
 
Your output must always be a single JSON object matching this exact schema:
 
{
  "posts": [
    {
      "style":               "punchy_insight | narrative_story | actionable_checklist",
      "hook":                "<opening line — the sentence that stops the scroll>",
      "body":                "<full post body — use \n\n for paragraph breaks>",
      "cta":                 "<closing call-to-action line>",
      "hashtags":            ["<tag1>", "<tag2>", "<tag3>"],
      "estimated_word_count": <integer>,
      "style_notes":         "<one sentence explaining the structural choice>"
    }
  ]
}
 
Rules you must follow at all times:
1. Return exactly 3 post objects — no more, no fewer.
2. Each post must use one of these exact style values:
   "punchy_insight"  |  "narrative_story"  |  "actionable_checklist"
   Each style value must appear exactly once across the 3 posts.
3. Do not add extra keys to the schema.
4. Never invent statistics, case study numbers, quotes, or external references
   unless they were explicitly provided in the topic_context field.
5. Obey every item in do_rules and dont_rules without exception.
6. Each post must be meaningfully different in structure — not just tone.
   A reader must instantly recognise which style they are reading.
7. LinkedIn formatting: use \n\n between paragraphs. No markdown headers.
   Emojis only if emoji_preference is 'yes' or 'sometimes'.
8. All three posts must be ready to publish — no [brackets], no placeholders.
```

### STYLE DEFINITIONS (part of system prompt)
```
STYLE 1 — "punchy_insight"
  Structure:
  - Hook: one strong declarative sentence.
  - 3–5 very short paragraphs (1–2 lines each).
  - White space between every paragraph. No story arc. No list.
  - End with a thought-provoking question or sharp closing statement.
  - Target: under 150 words.
 
STYLE 2 — "narrative_story"
  Structure:
  - Hook opens mid-scene (in medias res — present tense).
  - 2–3 paragraph arc: situation → tension/realisation → outcome/lesson.
  - Transition to broader takeaway (1 paragraph).
  - CTA asks the reader to share their own experience.
  - Target: 180–250 words.
 
STYLE 3 — "actionable_checklist"
  Structure:
  - Hook states a clear value promise ("5 things I learned about X").
  - Numbered list of 4–6 items.
  - Each item: bold short label + 1–2 sentence explanation.
  - Closing paragraph ties the list to the user's broader expertise.
  - CTA drives a save or share action.
  - Target: 200–280 words.
```

### USER PROMPT (filled per API call)
```
Generate 3 LinkedIn post drafts using the persona and topic below.
 
=== PERSONA ===
name:                    {{name}}
professional_background: {{background}}
current_role:            {{current_role}}
industry:                {{industry}}
tone:                    {{tone}}
  (e.g. "conversational and warm" | "authoritative and direct" | "humble and reflective")
language_style:          {{language_style}}
  (e.g. "plain English, no jargon" | "industry terms welcome" | "bilingual EN/HI mix")
typical_post_length:     {{length_preference}}
  (e.g. "short and punchy < 150 words" | "medium 150–250 words" | "long-form")
emoji_preference:        {{emoji_preference}}
  (yes | no | sometimes)
audience:                {{audience}}
  (e.g. "startup founders" | "engineering students" | "HR professionals")
 
do_rules:
{{numbered list of do rules}}
 
dont_rules:
{{numbered list of dont rules}}
 
=== TOPIC ===
topic:         {{topic}}
topic_context: {{optional: key points, personal anecdotes, data to include}}
goal:          {{goal}}
  (e.g. "build thought leadership" | "drive profile visits" | "encourage comments")
 
=== INSTRUCTIONS ===
- Follow all system rules strictly.
- Produce exactly 3 posts — one per style.
- Return only the JSON object. No other text.
```

## API Call Configuration
The prompt alone is not sufficient — the API call must also enforce JSON mode. Both layers together make malformed output practically impossible.

### OpenAI 
```
const response = await openai.chat.completions.create({
  model: "gpt-4o",
  response_format: { type: 'json_object' },   // ← API-level JSON enforcement
  temperature: 0,                              // ← deterministic output
  messages: [
    { role: "system", content: SYSTEM_PROMPT },
    { role: "user",   content: buildUserPrompt(persona, topic) }
  ]
});
const posts = JSON.parse(response.choices[0].message.content).posts;
```

### Gemini
```
const response = await model.generateContent({
  generationConfig: {
    responseMimeType: "application/json",      // ← API-level JSON enforcement
    temperature: 0,
  },
  contents: [{ role: 'user', parts: [{ text: FULL_PROMPT }] }]
});
const posts = JSON.parse(response.response.text()).posts;
```
> [!Note]
> **Why both layers?** Prompt instructions tell the model what to do. API json_mode enforces it at the token-sampling level — the model cannot physically emit a non-JSON token. One layer without the other leaves a gap.

## Sample Filled Persona (Reference)
| Field | Value |
|-------|-------|
| name | Priya Nair |
| current_role | Senior Product Manager at a B2B SaaS startup |
| industry | Product Management / B2B SaaS |
| tone | Conversational, warm, occasionally vulnerable |
| language_style | Plain English, light PM terminology, zero buzzwords |
| emoji_preference | Sometimes — max 2 per post |
| audience | Early-career PMs, startup founders, product enthusiasts |
| do_rules | 1. Share real personal experiences. <br> 2. Use specific details when available. <br> 3. End with a question that invites discussion. |
| dont_rules | 1. No corporate jargon. <br> 2. No humblebrag tone. <br> 3. Never claim expertise not earned. <br> 4. Avoid passive voice. |
| topic | Why saying no is the most important product skill |
| topic_context | Recently declined a high-visibility feature request from the CEO. The team thanked me later. No external stats — personal experience only. |
| goal | Build thought leadership; encourage PMs to comment with their own "no" stories |

## Why This Prompt Is Reliable
| Property | How It Is Achieved |
|----------|--------------------|
| Schema-first design | JSON schema is defined before any content rules. The model anchors its output format before reading persona or topic. |
| Enum in schema (not just rules) | `style` field shows allowed values inline: `punchy_insight \| narrative_story \| actionable_checklist`. Model sees the constraint exactly where it writes the value. |
| Hard structural differentiation | Each style has mutually exclusive structural rules (word count, format, arc type). Three tonal variations of the same structure are structurally impossible. |
| Named hallucination ban | Rule 4 specifically bans invented statistics, numbers, quotes, and external references — not a vague “be accurate” instruction. |
| Typed persona fields | Each field includes an inline example value. Reduces misinterpretation of abstract descriptors like `tone` or `language_style`. |
| API + prompt JSON enforcement | `response_format` / `responseMimeType` enforces JSON at the token level. `JSON.parse()` on the raw completion — no stripping, no regex, no fallback parser. |
| Temperature = 0 | Deterministic output. Same persona + topic always produces structurally consistent drafts. Avoids creative drift between calls. |

## Problem 3: **Smart DOCX Template → Bulk DOCX/PDF Generator (Proposal + Prompt)**

Users have many Word documents that act like templates (offer letters, certificates, invoices, contracts). They repeatedly change only a few fields (name, date, amount, address, role, etc.). Doing this manually is slow and error-prone, especially in bulk. [READ MORE ABOUT THE PROJECT](./docs-template-output-generation.md)

We want a system that:

1. Converts an uploaded **DOCX** into a reusable **template** by identifying editable fields.
2. Supports **single generation** (form-fill → DOCX/PDF download).
3. Supports **bulk generation** via **Excel/Google Sheet** rows.

### **Task (No coding)**

Submit a **proposal** for building this system using GenAI (OpenAI/Gemini) for “template field detection” and “field schema generation”. We want a practical design, not code.

### Your Solution for problem 3:

## Problem 3 — Smart DOCX Template → Bulk DOCX/PDF Generator

Users maintain Word templates (offer letters, invoices, certificates, contracts) where only a handful of fields change per document. The system detects editable fields via AI, then handles single and bulk generation with deterministic rendering — no hallucinations, no formatting loss.

| Principle | What It Means Here |
|-----------|-------------------|
| LLM Containment | LLM is invoked exactly once — for field detection only. Every other step (rendering, validation, PDF conversion) is deterministic. Zero hallucination risk in final documents. |
| Deterministic Rendering | python-docxtpl (Jinja2) replaces only placeholders. Original DOCX XML — tables, headers, footers, logos, signatures — is never touched. |
| Schema Versioning | Every confirmed template gets a version stamp. Bulk jobs record which version was used. Critical for auditing offer letters, contracts, compliance docs. |
| Row-Level Failure Isolation | A single bad spreadsheet row does not abort the batch. The job continues; the report flags every failure independently. |
| Template Injection Defense ▸ NEW | User-supplied field values are sanitized before Jinja2 rendering. Jinja2 control characters (`{% %}`, `{{ }}`, `{# #}`) are escaped to prevent template injection attacks that could break rendering or expose config data. |
| Streaming ZIP ▸ NEW | ZIP bundle is streamed to disk as files are rendered — not assembled in memory. Prevents OOM errors on large batches (hundreds/thousands of rows). |
| Data Minimisation | Spreadsheet row data is processed in memory and discarded after the job. It is never persisted to the database — only the generation report is retained. |
| Auditability | Each bulk job produces a structured job summary JSON alongside the CSV report. Supports SLA tracking and compliance reporting. |

## High-Level Architecture
```
User Upload DOCX
        ↓
Text Extraction Layer
        ↓
LLM Field Detection
        ↓
Field Schema Confirmation
        ↓
Template Conversion (Placeholder Injection)
        ↓
-------------------------------------------
Single Generation Flow:
Form Input → Validation → Render → DOCX/PDF Download

Bulk Generation Flow:
Excel/Sheet Upload → Row Validation → Parallel Rendering → ZIP Bundle + Report
```
## Phase 1 — Template Creation (AI-Assisted Field Detection)
### Step 1: Document Upload & Extraction
| Item | Description |
|------|------------|
| Tool | python-docx |
| What is extracted | Full text with paragraph boundaries and table cell context preserved. Repeated values flagged for `appears_multiple_times` detection. |
| What is NOT sent to LLM | Raw file bytes, images, embedded fonts, binary data. Only the extracted text string is sent. |

### Step 2 — LLM Field Detection Prompt
LLM is used exactly once. The prompt enforces a strict JSON-only response:
```
SYSTEM:
You are a document analysis AI. Your only job is to identify fields that
change between different instances of this template document.
Return a JSON array only — no explanation, no markdown, no extra keys.
 
Each object in the array must match this schema exactly:
[
  {
    "field_key":             "snake_case_identifier",
    "display_label":         "Human Readable Label",
    "field_type":            "text | date | currency | number | email | boolean",
    "sample_value":          "<value as it appears in the document>",
    "context_hint":          "<where/how this field appears>",
    "required":              true | false,
    "appears_multiple_times": true | false
  }
]
 
Rules:
1. Only extract values that realistically differ per document instance.
   >(names, dates, amounts, roles, addresses, IDs)
2. Do NOT extract: document title, company name, static boilerplate,
   >unless they explicitly vary per instance.
3. If a value appears multiple times (e.g. candidate name in greeting AND signature), set appears_multiple_times: true.
   >The system will replace all occurrences.
4. Return the JSON array only. No other text.
```
### Step 3 — Schema Confirmation UI
| Item | Description |
|------|------------|
| User actions | Add missed fields, remove false positives, rename labels, change field types, mark optional vs required, set date format / currency symbol. |
| Output | Confirmed fields saved as `template_schema.json`. DOCX converted to Jinja2 template via exact string replacement (not a second LLM call). |
| Version stamp | First confirmation = version 1.0. Any schema edit increments the minor version. Major structural changes prompt user to confirm a new major version. |

### Template Schema (Saved JSON)
```
{
  "template_id":  "offer_letter",
  "version":      "1.0",
  "created_at":   "2026-02-10T09:00:00Z",
  "fields": [
    {
      "field_key":             "candidate_name",
      "display_label":         "Candidate Full Name",
      "field_type":            "text",
      "required":              true,
      "appears_multiple_times": true
    },
    {
      "field_key":   "start_date",
      "display_label":"Start Date",
      "field_type":  "date",
      "format":      "DD MMMM YYYY",
      "required":    true
    },
    {
      "field_key":      "salary_annual",
      "display_label":  "Annual Salary",
      "field_type":     "currency",
      "currency_symbol":"₹",
      "required":       true
    }
  ]
}
```
## Phase 2 — Single Document Generation
### Pipeline
| Stage | Description |
|-------|------------|
| Select Template | User picks a saved template. Form is auto-generated from the field schema — date picker for date fields, currency input for currency, etc. |
| Client Validation | Required field check, type validation (date format, numeric range). Runs in browser before any network call. |
| Server Validation | Re-validates all fields server-side before rendering. Type enforcement, sanitization. Never trust client-side validation alone. |
| Injection Sanitize | Jinja2 control characters (`{% %}`, `{{ }}`, `{# #}`) are escaped in every field value before the render call. Prevents template injection — a docxtpl-specific attack vector where a user submits a malicious Jinja2 expression as a field value. |
| Render DOCX | python-docxtpl renders the Jinja2 template with sanitized values. Original formatting — tables, headers/footers, logos, signatures — is untouched. |
| Convert to PDF | LibreOffice headless: `soffice --headless --convert-to pdf`. Spawned as a subprocess per document. |
| Download | DOCX and/or PDF served as a file download. Filename: `<CandidateName>_<TemplateName>_<YYYYMMDD>.<ext>` — pattern configurable in template settings. |

## Phase 3 — Bulk Document Generation

### Spreadsheet Interface
The system generates a downloadable Excel template where column headers exactly match field_key values (with display_label as a comment on the header cell). The user fills one row per document. For Google Sheets, the user provides a share link, the system reads it via Google Sheets API.

### Pipeline
| Stage | Description |
|-------|------------|
| Parse Sheet | Read Excel (openpyxl) or Google Sheet rows. Map column headers to `field_key` values from schema. Flag unrecognised columns as warnings in the report. |
| Row Validation | Each row validated independently. Checks: required fields present, type correctness, date format parseable, numeric fields numeric. Invalid rows are flagged and skipped — the batch continues. |
| Sanitize (per row) | Jinja2 control characters escaped in every field value of every row before any render call. Applied uniformly regardless of source (Excel or Sheets). |
| Parallel Render | Configurable worker pool (default: CPU count). Each valid row: render DOCX → convert PDF. Worker failures are caught per-row and logged to the report without stopping other workers. |
| Streaming ZIP | Files are written into the ZIP bundle as they are rendered — not buffered in memory first. Prevents OOM errors on large batches. ZIP streamed to disk and made available for download when all rows are done. |
| Report Generation | CSV report: `row_number`, `status`, `file_name`, `error_reason`. Job summary JSON: `rows_total`, `rows_success`, `rows_failed`, `processing_time_seconds`, `template_version`. Both are included in the ZIP and shown as a summary table in the UI. |

## ZIP Output Structure
```
generated_docs/
└── offer_letter_v1.0_2026-02-10/
    ├── pdf/
    │   ├── Rohan_Sharma_OfferLetter_20260210.pdf
    │   ├── Super_Man_OfferLetter_20260210.pdf
    │   └── Anjali_Pathania_OfferLetter_20260210.pdf
    ├── docx/
    │   ├── Rohan_Sharma_OfferLetter_20260210.docx
    │   └── Super_Man_OfferLetter_20260210.docx
    ├── generation_report.csv
    └── job_summary.json
```
## Generation Report (CSV + UI Table)
| Row  | Status     | File Name                                   | Error Reason                  |
|-------|------------|--------------------------------------------|--------------------------------|
| 1     | ✓ Success  | Rohan_Sharma_OfferLetter_20260210.pdf        | —                           |
| 2     | ✓ Success  | Super_Man_OfferLetter_20260210.pdf      | —                                |
| 3     | ✗ Skipped  | —                                          | Missing required field: salary_annual |
| 4     | ✗ Skipped  | —                                          | Invalid date format: start_date     |
| 5     | ✓ Success  | Anjali_Pathania_OfferLetter_20260210.pdf        | —                        |

## Job Summary JSON

```
{
  "template_id":             "offer_letter",
  "template_version":        "1.0",
  "job_id":                  "bulk_20260210_001",
  "rows_total":              120,
  "rows_success":            115,
  "rows_skipped":            5,
  "processing_time_seconds": 48,
  "generated_at":            "2026-02-10T11:23:00Z"
}
```
## Security Boundaries
| Threat | Mitigation |
|--------|------------|
| Template Injection | All field values sanitized before Jinja2 render. Jinja2 control sequences (`{% %}`, `{{ }}`, `{# #}`) are HTML-escaped or stripped. Sandbox mode enabled in Jinja2 environment to prevent code execution. |
| Malicious DOCX upload | File type validated by magic bytes (not extension). DOCX unpacked and inspected before processing. Macro-enabled `.docm` files rejected. |
| Google Sheets data exposure | Service account scoped to read-only. Credentials stored in server environment variables, never in client code or logs. |
| Spreadsheet data retention | Row data processed in memory only. Never written to the database. Only the job summary and error report are retained. |
| Rendered document access | Download links are signed, time-limited URLs (expire after 30 minutes). Generated files deleted from server after download or after 24h. |

## Scalability & Observability

| Component | Description |
|-----------|------------|
| Worker Pool | Configurable concurrency (default: CPU count). Each worker handles one row end-to-end (validate → render → PDF). Worker failures are isolated — no shared state between rows. |
| Streaming ZIP | Files streamed into ZIP as they complete. No full batch buffer in memory. Handles batches of thousands of rows without OOM risk. |
| Job Queue | Bulk jobs tracked in a lightweight job table (SQLite for single-server, Redis/Postgres for multi-server). Supports retry and resume if server restarts mid-batch. |
| Horizontal Scaling | Workers are stateless — can be run on separate machines. Job queue acts as the coordinator. ZIP assembly happens on the output server. |
| LibreOffice Pool | LibreOffice headless has a high startup cost (~1–2 sec per process). Production setup maintains a warm pool of LibreOffice instances to eliminate per-document startup latency. |

## Why This Architecture Works
> [!IMPORTANT]
> **LLM is a scalpel, not a paintbrush:** Used once, for the one task where creative inference is needed (field detection). Everything else — rendering, validation, PDF conversion, report generation — is deterministic. This makes the system safe to run at scale on legal and financial documents.

> [!NOTE]
> **Schema versioning = auditability:** Every bulk job records which template version was used. If an offer letter is disputed 6 months later, you can replay exactly which fields and which template were active at generation time.

> [!CAUTION]
> **Template injection is a real risk:** docxtpl uses Jinja2 under the hood. A user who submits {% for x in config %} as a field value can crash the render or expose internal configuration. Sanitization and Jinja2 sandbox mode are non-negotiable in production.


## Problem 4: Architecture Proposal for 5-Min Character Video Series Generator

We want to build a system that helps a user create a short video series (around **5 minutes per episode**) using predefined characters. The user defines characters (image + personality) and relationships once, then provides a short story/situation for an episode. The system outputs a complete episode package and optionally a final video. [READ MORE ABOUT THE PROJECT](./char-based-video-generation.md)

### **Task**

Create a **small, clear architecture proposal** (no code, no prompts) describing how you would design and build this system.

### Your Solution for problem 4:

You need to put your solution here.
