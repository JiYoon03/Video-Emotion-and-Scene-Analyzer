# Video Emotion and Scene Analyzer

A full-stack application that takes a raw video file and turns it into structured, human-readable insights: transcript, scenes, frame captions, and emotional analysis.
Backend uses **FastAPI + Celery + Redis**, and the frontend uses **Next.js (React + TypeScript)**.

---

## Features

* Upload a video directly from the browser and preview it.
* Asynchronous, non-blocking video analysis with **Celery tasks**.
* Real-time job progress reporting (percent + step).
* **Whisper (faster-whisper)** for speech transcription.
* **OpenCV** frame extraction and **BLIP** image captioning.
* Scene grouping using caption similarity (difflib.SequenceMatcher).
* Frame-level facial emotion detection using a **ViT-based classifier** (dima806/facial_emotions_image_detection).
* Merged “text + emotion” timeline for UI visualization.

---

## Tech Stack

### Backend

* FastAPI, Uvicorn
* Celery, Redis
* faster-whisper
* transformers (BLIP, ViT emotion model)
* OpenCV, Pillow, moviepy
* numpy, tqdm, torch

### Frontend

* Next.js (App Router)
* React, TypeScript
* Tailwind-like utility classes

---

## Project Structure

```
.
├── backend/
│   ├── main.py                 # FastAPI app (upload, analyze, status)
│   ├── tasks.py                # Celery app + process_video_task
│   ├── analysis_pipeline.py    # Whisper, BLIP, scenes, emotions, merging
│   ├── uploaded_videos/        # Temporary storage
│   ├── requirements.txt
│   └── __init__.py
└── frontend/
    ├── package.json
    ├── tsconfig.json
    ├── next.config.js
    ├── app/
    │   └── page.tsx            # Upload → analyze → results UI
    └── public/
```

---

## Backend – API Overview

### **POST /upload-video/**

Uploads a video via form-data (`video_file`).
Response:

```json
{
  "message": "Uploaded",
  "filename": "your_video.mp4"
}
```

### **POST /analyze-video/**

Request body:

```json
{ "video_filename": "your_video.mp4" }
```

Returns:

```json
{ "job_id": "celery-task-id" }
```

### **GET /analysis/{job_id}**

Polling results:

* `pending`
* `processing` (with `percent` and `step`)
* `failed`
* `completed` (returns full analysis JSON)

---

## Celery Task – `process_video_task`

Progress steps:

* 5% – starting
* 20% – transcribing
* 40% – extracting frames
* 60% – captioning
* 70% – grouping scenes
* 80% – attaching dialogue
* 90% – analyzing emotions
* 100% – finalizing

Returns a result dict containing:

* `transcript_text`
* `transcript_segments`
* `frame_captions`
* `scenes`
* `combined_scenes`
* `language`
* `merged_text_emotions`

---

## Analysis Pipeline (`analysis_pipeline.py`)

### **transcribe_with_whisper(video_path)**

* Uses faster-whisper
* Returns transcript segments, full text, detected language

### **extract_frames(video_path, fps=1.0)**

* Extracts 1 frame per second
* Returns raw frames

### **caption_frames(frames)**

* BLIP captioning
* Returns list of strings

### **categorize_scenes(captions)**

* Groups captions by semantic similarity
* Computes scene start/end and representative caption

### **combine_scenes_with_transcript**

* Merges scene timelines and transcript segments

### **analyze_emotions**

* Samples frames every N seconds
* Predicts dominant emotion + emotion score per label

### **merge_text_and_emotions**

* Aligns transcript sentences with emotion samples by index

### **analyze_video**

Runs the entire pipeline and returns a single result JSON.

---

## Frontend – UI (`frontend/app/page.tsx`)

* File upload → preview
* Calls backend APIs
* Real-time progress polling
* Timeline and emotion bar visualization
* Scene list with expandable transcript
* Displays top frame captions
* “Copy JSON” button for full analysis export

---

## How to Run – Backend

### Create & activate virtual environment

```bash
python -m venv venv
source venv/bin/activate   # macOS / Linux
# venv\Scripts\activate    # Windows
```

### Install dependencies

```bash
cd backend
pip install fastapi uvicorn celery redis opencv-python pillow torch \
    faster-whisper transformers moviepy tqdm numpy
```

### Start Redis

```bash
redis-server
```

### Start Celery worker

```bash
cd backend
celery -A tasks.celery_app worker --loglevel=info
```

### Start FastAPI server

```bash
uvicorn main:app --reload --port 8000
```

Backend is now available at:

```
http://127.0.0.1:8000
```

---

## How to Run – Frontend

```bash
cd frontend
npm install
npm run dev
```

Frontend runs at:

```
http://localhost:3000
```

---

## Usage Flow

1. Start Redis → Celery → FastAPI.
2. Start Next.js.
3. Open the web app at `localhost:3000`.
4. Upload a video and click **Analyze**.
5. Watch real-time progress.
6. Inspect results:

   * transcript
   * emotion-tagged timeline
   * frame captions
   * scene groups
   * full JSON output

---

## Notes / Future Improvements

* Long videos may be slow without GPU acceleration.
* Emotion analysis assumes single face per frame.
* Text–emotion alignment uses simple index matching; time-based alignment would improve accuracy.
* For production:

  * Dockerization
  * External storage (S3, GCS, etc.)
  * Authentication + rate limiting
  * Better logging & error handling
