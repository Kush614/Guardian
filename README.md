download zip file
# Guardian AI — Real-Time Security Monitor

An AI-powered security camera system that runs entirely on-device using **Gemma 3 4B** (vision) for real-time threat detection. It classifies incidents by severity, maps nearby San Francisco hospitals and crime hotspots using real DataSF open data, and simulates 911 dispatch for critical emergencies.

![Tech](https://img.shields.io/badge/Node.js-Express-green) ![AI](https://img.shields.io/badge/Gemma_3-4B_Vision-blue) ![License](https://img.shields.io/badge/License-MIT-yellow)

---

## Table of Contents

- [Why On-Device?](#why-on-device)
- [Features](#features)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Quick Start](#quick-start)
- [Running with llama.cpp (Recommended)](#running-with-llamacpp-recommended)
- [Running with Ollama (Alternative)](#running-with-ollama-alternative)
- [Demo Mode](#demo-mode)
- [DataSF Open Data Integration](#datasf-open-data-integration)
- [REST API Endpoints](#rest-api-endpoints)
- [Fine-Tuning (Optional)](#fine-tuning-optional)
- [Tech Stack](#tech-stack)
- [Environment Variables](#environment-variables)
- [Hardware Requirements](#hardware-requirements)

---

## Why On-Device?

| Benefit | Description |
|---------|-------------|
| **Privacy** | Security footage never leaves the local machine — no cloud APIs, no data leaks |
| **Low Latency** | Direct GPU inference means faster response for emergency situations |
| **Reliability** | Works fully offline in areas with poor or no internet connectivity |
| **Cost** | Zero API fees — runs on consumer hardware you already own |

---

## Features

- **Real-Time Vision Analysis** — Webcam frames analyzed every 5 seconds by Gemma 3 4B vision model
- **Audio Distress Detection** — Web Audio API monitors microphone for loud sounds (screams, impacts, glass breaking) and escalates severity
- **4-Tier Threat Classification** — CRITICAL / HIGH / MEDIUM / LOW with automated response logic
- **Agentic Response Engine**:
  - **CRITICAL**: Simulates 911 call, shows nearest trauma center, voice alert via Web Speech API
  - **HIGH**: Dispatches security alert, routes to nearest hospital on map
  - **MEDIUM**: Logs incident, notifies security, continues monitoring
  - **LOW**: Normal activity — no action required
- **Interactive Map** — Leaflet.js with CARTO dark tiles showing:
  - 78 healthcare facilities (color-coded by type)
  - Crime/EMS hotspot zones from DataSF data
  - Live incident markers with hospital routing lines
- **DataSF Integration** — Real San Francisco open data: 5,000 EMS calls, 5,000 police incidents, neighborhood risk profiles
- **Voice Alerts** — Browser Speech Synthesis announces CRITICAL and HIGH incidents
- **Demo Mode** — 12 pre-built scenarios cycle automatically — no camera or AI model needed
- **WebSocket Real-Time** — All connected clients receive incident updates instantly

---

## Architecture

```
┌──────────────┐    WebSocket     ┌──────────────┐   HTTP/JSON    ┌──────────────────┐
│              │  ◄────────────►  │              │  ────────────►  │                  │
│   Browser    │   frames/events  │  Node.js     │   /v1/chat/    │  llama-server    │
│   Dashboard  │                  │  Express     │   completions  │  (Gemma 3 4B)    │
│              │                  │  Server      │                │  + Vision MMPROJ │
└──────────────┘                  └──────────────┘                └──────────────────┘
  │ Camera                          │ DataSF                        │ GPU (CUDA)
  │ Microphone                      │ JSON files                    │ CPU fallback
  │ Leaflet Map                     │ Incident log
  │ Speech API                      │ Hospital routing
```

**Flow:**
1. Browser captures a webcam frame → converts to base64 JPEG
2. Frame sent via WebSocket to the Node.js server
3. Server forwards the image to llama-server's OpenAI-compatible vision endpoint
4. Gemma 3 4B analyzes the frame and returns a JSON threat classification
5. Server enriches the result with nearest hospitals, hotspot data, and neighborhood risk
6. Result broadcast to all connected clients via WebSocket
7. Dashboard updates severity banner, incident log, map markers, and triggers voice alerts

---

## Project Structure

```
Guardian-AI/
├── server.js                      # Express + WebSocket server, API routes, AI integration
├── package.json                   # Node.js dependencies (express, ws)
├── Modelfile                      # Ollama model config (for Ollama users)
├── gemma-3-4b-it.Q4_K_M.gguf     # Quantized Gemma 3 4B model (Q4_K_M)
├── gemma-3-4b-it.F16-mmproj.gguf # Vision projector for multimodal support
│
├── public/                        # Frontend (served as static files)
│   ├── index.html                 # Dashboard layout
│   ├── style.css                  # Dark theme UI styles
│   └── app.js                     # Camera capture, WebSocket, map, demo mode
│
├── data/
│   ├── sf_incident_hotspots.json  # Pre-computed crime/EMS hotspot zones
│   └── datasf/                    # San Francisco Open Data (DataSF)
│       ├── healthcare_facilities.json   # 78 healthcare facilities (jhsu-2pka)
│       ├── fire_ems_calls.json          # 5,000 recent Fire/EMS dispatch calls (nuek-vuh3)
│       ├── police_incidents.json        # 5,000 recent violent crime reports (wg3w-h783)
│       ├── ems_by_neighborhood.json     # EMS call aggregates by neighborhood
│       └── crime_by_neighborhood.json   # Crime aggregates by neighborhood
│
├── fine-tune/                     # Optional: LoRA fine-tuning
│   ├── finetune_gemma.ipynb       # Google Colab notebook for fine-tuning
│   └── guardian_dataset.jsonl     # Training data for security scenarios
│
├── llama-server/                  # llama.cpp binaries (auto-downloaded)
│   └── llama/
│       ├── llama-server.exe       # llama.cpp HTTP server
│       ├── ggml-cuda.dll          # CUDA backend
│       └── ...                    # Other DLLs
│
└── Guardian/                      # Additional project assets
```

---

## Quick Start

### Prerequisites

- **Node.js** 18+ installed
- **NVIDIA GPU** with 4GB+ VRAM (or CPU-only with slower inference)
- A **webcam** (or use Demo Mode without one)

### 1. Install dependencies

```bash
npm install
```

### 2. Download the model files

You need two GGUF files in the project root:

| File | Size | Description |
|------|------|-------------|
| `gemma-3-4b-it.Q4_K_M.gguf` | ~2.3 GB | Quantized Gemma 3 4B language model |
| `gemma-3-4b-it.F16-mmproj.gguf` | ~812 MB | Vision encoder/projector |

These can be downloaded from [Hugging Face (Unsloth)](https://huggingface.co/unsloth).

---

## Running with llama.cpp (Recommended)

### 1. Download llama-server

Download the latest release from [llama.cpp releases](https://github.com/ggml-org/llama.cpp/releases):

- **NVIDIA GPU**: `llama-*-bin-win-cuda-12.4-x64.zip` + `cudart-llama-bin-win-cuda-12.4-x64.zip`
- **CPU only**: `llama-*-bin-win-cpu-x64.zip`
- **AMD GPU**: `llama-*-bin-win-hip-radeon-x64.zip`

Extract both zips into `llama-server/llama/`.

### 2. Start llama-server

```bash
./llama-server/llama/llama-server \
  --model gemma-3-4b-it.Q4_K_M.gguf \
  --mmproj gemma-3-4b-it.F16-mmproj.gguf \
  --port 8080 \
  --n-gpu-layers 20 \
  --ctx-size 2048 \
  --threads 6
```

Adjust `--n-gpu-layers` based on your VRAM:
| VRAM | Recommended Layers |
|------|--------------------|
| 4 GB | 18-20 |
| 6 GB | 28-30 |
| 8 GB+ | 35 (all) |

### 3. Start the Guardian AI server

```bash
npm start
```

### 4. Open the dashboard

Navigate to **http://localhost:3000** in your browser. Grant camera and microphone permissions when prompted.

---

## Running with Ollama (Alternative)

### 1. Install Ollama

Download from [ollama.com](https://ollama.com) and pull the model:

```bash
ollama pull gemma3:4b
```

### 2. Update server.js

Change the server config to point to Ollama instead of llama-server (see the `LLAMA_URL` variable in `server.js`).

### 3. (Optional) Create a custom model with the Guardian system prompt

```bash
ollama create guardian-ai -f Modelfile
MODEL=guardian-ai npm start
```

---

## Demo Mode

Don't have a GPU or camera? No problem.

Click the **"Demo Mode"** button in the top-right of the dashboard. It will:

- Cycle through **12 pre-built security scenarios** every 4-7 seconds
- Simulate incidents ranging from LOW (normal pedestrian activity) to CRITICAL (fire, unresponsive person)
- Show live map markers, hospital routing, incident logs, and voice alerts
- Trigger a simulated 911 call for CRITICAL events

Click the button again to stop and resume live camera analysis.

---

## DataSF Open Data Integration

Guardian AI uses real San Francisco open data to provide contextual risk awareness:

| Dataset | Source | Records | Usage |
|---------|--------|---------|-------|
| Healthcare Facilities | DataSF `jhsu-2pka` | 78 | Map markers, nearest hospital routing |
| Fire/EMS Dispatch | DataSF `nuek-vuh3` | 5,000 | Recent EMS calls feed, hotspot computation |
| Police Incidents | DataSF `wg3w-h783` | 5,000 | Violent crime reports, neighborhood risk |
| EMS by Neighborhood | Aggregated | 125 rows | Neighborhood medical risk profiles |
| Crime by Neighborhood | Aggregated | 185 rows | Neighborhood crime risk profiles |

This data powers:
- **42 neighborhood risk profiles** combining medical and crime data
- **Hotspot zones** on the map (color-coded by risk level)
- **DataSF live feed** panel on the dashboard showing recent EMS and police activity

---

## REST API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/status` | Server and model health check |
| `GET` | `/api/hospitals` | All healthcare facilities (optionally filter by `?lat=&lng=`) |
| `GET` | `/api/facilities?lat=&lng=&type=` | Facilities filtered by type and sorted by distance |
| `GET` | `/api/incidents` | Last 50 detected incidents |
| `GET` | `/api/hotspots` | Crime/EMS hotspot zones |
| `GET` | `/api/neighborhood-risk?name=` | Risk profile for a neighborhood (or all) |
| `GET` | `/api/datasf/ems-recent?limit=&type=` | Recent EMS dispatch calls |
| `GET` | `/api/datasf/police-recent?limit=&category=` | Recent police incident reports |
| `GET` | `/api/datasf/ems-stats` | EMS aggregates by neighborhood |
| `GET` | `/api/datasf/crime-stats` | Crime aggregates by neighborhood |
| `POST` | `/api/analyze` | Analyze a single image (`{ image: base64, audioAlert: bool }`) |

---

## Fine-Tuning (Optional)

To improve classification accuracy on security-specific scenarios:

1. Open `fine-tune/finetune_gemma.ipynb` in **Google Colab** (free T4 GPU)
2. The notebook uses **LoRA** (Low-Rank Adaptation) via Hugging Face PEFT to fine-tune Gemma 3 on the included `guardian_dataset.jsonl`
3. Export the fine-tuned model to GGUF format
4. Replace the model file or create a custom Ollama model:

```bash
ollama create guardian-ai -f Modelfile
MODEL=guardian-ai npm start
```

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Backend | Node.js + Express 5 |
| Frontend | Vanilla HTML/CSS/JS (no frameworks) |
| AI Model | Gemma 3 4B (vision) — Q4_K_M quantized |
| Inference | llama.cpp server (OpenAI-compatible API) |
| Maps | Leaflet.js + CARTO dark basemap tiles |
| Audio | Web Audio API (frequency analysis) |
| Voice | Web Speech Synthesis API |
| Real-time | WebSocket (ws library) |
| Data | San Francisco Open Data (DataSF) |
| Fine-tuning | LoRA via Hugging Face PEFT + Unsloth |

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3000` | Guardian AI dashboard server port |
| `LLAMA_URL` | `http://127.0.0.1:8080` | llama-server API endpoint |
| `MODEL` | `gemma3:4b` | Model identifier (for display/logging) |

---

## Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| **GPU** | GTX 1650 (4 GB VRAM) | RTX 3060+ (8 GB+ VRAM) |
| **RAM** | 8 GB | 16 GB |
| **CPU** | 4 cores | 6+ cores |
| **Storage** | ~4 GB (model files) | SSD for faster model loading |
| **Webcam** | Any USB/built-in | 720p+ recommended |
| **OS** | Windows 10/11, Linux, macOS | Windows 11 with CUDA drivers |

> **Note**: On a GTX 1650 (4 GB, no tensor cores), expect ~11s for image encoding and ~1-5 tokens/sec for generation. An RTX 3060 or better will provide significantly faster inference.

---

## License

MIT

