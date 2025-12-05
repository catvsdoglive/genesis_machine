# GENESIS_MACHINE

GENESIS_MACHINE is an autonomous meme token _text + image_ generator.

It uses:

- a local **Mistral** LLM (via `llama.cpp`) to generate names, tickers and descriptions  
- multiple **Stable Diffusion** pipelines (SD 1.5 / SD 1.4 / SD 2.1 + LoRA + TI embeddings) to create images  
- a small **FastAPI** backend + WebSocket server for control and metrics  
- a browser-based **frontend** (HTML/CSS/JS) for monitoring and interacting with the generator  

By default, GENESIS_MACHINE consumes **live trending words** from the [`catvsdog-live`](https://github.com/catvsdoglive/catvsdog-live) project via its `/get-trending-words` endpoint, but you can point it to any compatible service.

> This repository does **not** include any model weights or generated images.  
> You bring your own models and configure the paths in `.env`.

---

## Features

- **Token generation loop**
  - Builds candidate token names, tickers and descriptions with `text_generator.py` and `meme_factory.py`.
  - Uses `words.json` and (optionally) live trending words from `TOP_WORDS_ENDPOINT`.
  - Applies persona/style blending (`persona.py`, `template_library.py`, `render_params.py`).

- **Image generation**
  - `image_generator.py` selects a Stable Diffusion model (SD 1.5 / 1.4 / 2.1) per generation.
  - Applies LoRA (`LORA_FILE`) and textual inversion embeddings (EasyNegative, BadDream, UnrealisticDream).
  - Uses configurable steps / CFG / seed, LoRA strength, and a rich negative prompt.

- **Persistence and metrics**
  - Stores tokens in a local SQLite DB (`meme_tokens.db`).
  - Writes metrics to `metrics.jsonl` and human-readable logs via `memelogging.py` and `metrics.py`.
  - Caches personas between runs (`persona_cache.json`).

- **Runtime control**
  - `api_server.py` exposes a FastAPI HTTP API (token queries, metrics, system info).
  - `ws_server.py` streams live status, new tokens, and system events over WebSocket.
  - `system_monitor.py` tracks loop statistics and model state.
  - `retry_policy.py` and model-switch configuration manage failures and model quarantine.

- **Frontend UI**
  - `public/index.html`, `public/styles.css` and JS files (`main.js`, `ui.js`, `websocket.js`, `api.js`, `blurEffect.js`)
    render:
    - a live feed of generated tokens
    - detailed token view modal
    - system status / metrics panels
    - draggable / dockable UI elements

---

## Architecture Overview

High-level architecture:

1. **Input**
   - `fetch_top_words.py` fetches top words from `TOP_WORDS_ENDPOINT` (default: `https://catvsdog.live/get-trending-words`).
   - `words.json` provides a local fallback list.

2. **Text generation**
   - `text_generator.py` wraps a local Mistral model (via `LLM_MODEL_PATH`) and generates:
     - token name
     - ticker
     - description
     - structured metadata for prompts
   - Generation parameters (temperature, top-p, penalties) are bounded by the ranges in `.env`.

3. **Prompt construction**
   - `prompt_builder.py`, `action.py`, `breed.py`, `environment.py`, `attribute.py`, `style.py`,
     `artist.py`, `camera.py`, `lighting.py`, `colour.py`, `quality.py`, `template_library.py`
     collaborate to build:
     - a positive prompt
     - a negative prompt (with optional disabling)
     - LoRA and TI settings.

4. **Image generation**
   - `image_generator.py` selects a model:
     - SD 1.5 (`SD_V1_5_PATH`)
     - SD 1.4 (`SD_V1_4_PATH`)
     - SD 2.1 (`SD_V2_1_PATH`)
   - Applies LoRA adapters and TI embeddings from `TI_DIR`.
   - Renders an image and saves it to `IMAGE_DIR`.

5. **Persistence**
   - `output_saver.py` writes token metadata + file paths into an SQLite DB at `DB_PATH`.
   - `metrics.py` appends JSONL metrics and summary statistics.

6. **Serving & UI**
   - `run_api.py` launches `api_server.py` (FastAPI) and `ws_server.py`.
   - `public/` assets are served by whatever HTTP server you put in front (for example: nginx, uvicorn static mount).
   - The browser connects via WebSocket to `MEME_WS_HOST:MEME_WS_PORT` and via HTTP to the FastAPI endpoints.

---

## Repository Layout

Key files and directories:

- **Core logic**
  - `meme_factory.py` – main generation loop orchestrator  
  - `text_generator.py` – Mistral / llama.cpp integration  
  - `prompt_builder.py` – positive + negative prompt construction  
  - `image_generator.py` – Stable Diffusion pipeline orchestration  
  - `output_saver.py` – SQLite persistence  
  - `metrics.py`, `memelogging.py`, `system_monitor.py` – metrics, logging and system state  
  - `retry_policy.py`, `render_params.py`, `config.py` – configuration and defaults  

- **API and WebSocket**
  - `api_server.py` – FastAPI application (tokens, metrics, system endpoints)  
  - `ws_server.py` – WebSocket server for UI updates  
  - `run_api.py` – convenient entry-point to run API + WS server  

- **Prompt and persona components**
  - `action.py`, `breed.py`, `environment.py`, `attribute.py`, `style.py`, `artist.py`,  
    `camera.py`, `lighting.py`, `colour.py`, `quality.py`  
  - `persona.py`, `operator_messages.py`, `template_library.py`  

- **Frontend**
  - `public/index.html`  
  - `public/styles.css`  
  - `public/api.js`  
  - `public/main.js`  
  - `public/ui.js`  
  - `public/websocket.js`  
  - `public/blurEffect.js`  

- **Data & config**
  - `words.json` – local word list fallback  
  - `.env.example` – example environment configuration (copy to `.env` and customize)

> Note: `models/`, `output/`, `logs/` and any `.db` files are **not** stored in this repository.  
> Add them locally and configure paths in `.env`.

---

## Requirements

GENESIS_MACHINE is designed for a Linux server with a reasonably powerful CPU and optionally a GPU.

- Python 3.10+  

Recommended packages (install in a virtualenv):

- `fastapi`  
- `uvicorn[standard]`  
- `pydantic`  
- `websockets`  
- `requests`  
- `python-dotenv`  
- `sqlalchemy` or rely on the standard-library `sqlite3` used in the code  
- `pillow`  
- `torch`, `diffusers`, `transformers`, `safetensors` (for Stable Diffusion)  
- `llama-cpp-python` or the equivalent binding used by `text_generator.py`  

Refer to the code for the exact imports and install those via `pip`.

You also need:

- Stable Diffusion model folders (paths configured in `.env`)  
- A Mistral `.gguf` file (configured via `LLM_MODEL_PATH`)

---

## Configuration

1. Copy the example env file:

   cp .env.example .env

2. Edit `.env` and update at least:

   - `DB_PATH`, `OUTPUT_DIR`, `IMAGE_DIR`  
   - `LOG_PATH`, `METRICS_PATH`  
   - `SD_V1_5_PATH`, `SD_V1_4_PATH`, `SD_V2_1_PATH`  
   - `LLM_MODEL_PATH`  
   - `TI_DIR` (if you use TI embeddings)  
   - `TOP_WORDS_ENDPOINT` if you are not using `catvsdog.live`

3. Make sure the directories referenced in `.env` exist and are writable.

---

## Running

In development, a typical flow is:

1. Create and activate a virtualenv

   python -m venv .venv  
   source .venv/bin/activate

2. Install dependencies

   pip install -r requirements.txt   (if you create one)  
   or install packages manually based on the imports.

3. Configure environment

   cp .env.example .env  
   edit `.env` to point to your model folders and output directories.

4. Start the API + WebSocket server

   python run_api.py

Then:

- Serve `public/` via nginx or another HTTP server, pointing the frontend JS to the API and WS endpoints you exposed.
- Open the UI in your browser and watch GENESIS_MACHINE generate tokens and images.

---

## Relation to `catvsdog-live`

GENESIS_MACHINE can run standalone, but it was originally designed to pair with:

- [`catvsdog-live`](https://github.com/catvsdoglive/catvsdog-live) – a real-time cat/dog/other token classifier and sentiment tracker.

By default:

- `TOP_WORDS_ENDPOINT` in `.env.example` is set to `https://catvsdog.live/get-trending-words`.  
- `fetch_top_words.py` will call this endpoint to obtain live trending words and feed them into token generation.

You can change `TOP_WORDS_ENDPOINT` to point at any other service that returns a compatible JSON payload.

---

## License

This project is released under the MIT License. See the `LICENSE` file for details.
