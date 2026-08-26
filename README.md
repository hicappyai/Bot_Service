<div align="center">
  <img src="assets/banner.png" alt="HiCapy Meeting Bots Banner" />
  <h1>HiCapy Meeting Bots - Google Meeting Bot</h1>
</div>

---

## Links to Repositories

You can explore our platform repositories for additional tools and integrations:

<ul>
  <li><a href="https://github.com/hicappyai/Bot_Service" target="_blank">Google Meet Bot Service</a></li>
</ul>

---

## 🧠 System Architecture

The Bot Service is a high-performance, containerized microservice that joins Google Meet calls autonomously, streams real-time audio, and powers sub-500ms conversational AI voice interactions.

### High-Level Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Control Panel / FastAPI Backend                        │
│             POST /api/bots/{id}/start  ──▶  SessionManager                  │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │ Docker Launch / API Command
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Bot Service Execution Container                        │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      Linux Xvfb Display (:99)                       │   │
│   │   Headless Chrome (Selenium) ──▶ Google Meet Web Application        │   │
│   └──────────────────────────────┬──────────────────────────────────────┘   │
│                                  │                                          │
│   ┌──────────────────────────────┴──────────────────────────────────────┐   │
│   │                 PulseAudio Virtual Audio Routing                    │   │
│   │   MeetOutput Sink  ──▶ FFmpeg Audio/Video Capture & AudioInput Stream │   │
│   │   BotMic Sink      ◀── pacat PCM Playback Pipe                      │   │
│   │   VirtualMic Src   ──▶ Chrome Input Microphone                      │   │
│   └──────────────────────────────┬──────────────────────────────────────┘   │
│                                  │ Sub-500ms Real-Time Pipeline             │
│                                  ▼                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                Pipecat Voice AI Processing Loop                     │   │
│   │  Silero VAD ──▶ Deepgram WebSocket STT ──▶ Groq LLM (Streaming)    │   │
│   │              ──▶ Deepgram WebSocket TTS ──▶ Audio Output Pipe       │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Core Architecture Components

1. **Headless Browser & Virtual Audio Stack**
   - **Xvfb Display (`:99`)**: Runs headless Chrome inside Docker to join Google Meet calls without requiring a physical GUI.
   - **PulseAudio Sinks**: Virtual sinks (`MeetOutput` for meeting audio capture, `BotMic` remapped to Chrome's virtual microphone) isolate bot audio from host hardware.

2. **Real-Time Voice AI Pipeline (Sub-500ms TTFB)**
   - **Streaming ASR**: Deepgram WebSocket API (`wss://api.deepgram.com/v1/listen`) for speech-to-text with minimal latency.
   - **VAD & Echo Cancellation**: Silero Neural VAD paired with WebRTC Acoustic Echo Cancellation (AEC) prevents self-interruption.
   - **Streaming LLM**: Groq API (`llama-3.3-70b-versatile` / `llama-3.1-8b-instant`) with sentence boundary parsing (`.`, `!`, `?`).
   - **Streaming TTS**: Deepgram WebSocket Speak API (`wss://api.deepgram.com/v1/speak`) returning audio PCM frames in ~80ms.

3. **Lifecycle & Session Management**
   - **DOM MutationObserver**: Uses JavaScript `MutationObserver` to poll lightweight UI flags every 100ms, reducing Selenium CPU overhead by **~90%**.
   - **Participant Count Auto-Shutdown**: Continuously monitors active participant video tiles and triggers container termination when human count $\le 1$, eliminating zombie containers and saving **2GB RAM per session**.

---

### Environment Variables
Ensure your `.env` file includes:
```bash
# Required for Transcription
DEEPGRAM_API_KEY=your_key

# Required for Voice/LLM
GROQ_API_KEY=your_key

# Bot Configuration
BOT_NAME="Bot Assistant"
ENABLE_RECORDING=true        # Save MP4 video
ENABLE_TRANSCRIPT=true       # Save JSON transcript
ENABLE_SPEAK=false          # Enable voice interaction (requires Pipecat)

# Shared Meeting Status Store
MONGO_URL=mongodb://localhost:27017
MONGO_DB_NAME=hicapy
```

---

## 💻 Local Development Setup

### 1. Prerequisites
- Python 3.10+
- [Poetry](https://python-poetry.org/) (recommended) or pip
- Chrome Browser
- **Audio Setup** (for Windows): See "Audio Setup (Windows)" section below

### 2. Installation

Using Poetry (Recommended):
```bash
# Install dependencies
make install
# OR manually:
poetry install

# Activate shell
make poetry-shell
```

Using Pip:
```bash
pip install -r requirements.txt
# For voice features:
pip install -r requirements_voice.txt
```

### 3. Running Locally

Run the bot with a Google Meet link:

```bash
# Basic recording & transcription
python app.py "https://meet.google.com/abc-defg-hij"

# With voice assistant enabled
python app.py "https://meet.google.com/abc-defg-hij" --speak true
```

**Common Arguments:**
- `--min-record-time`: Minimum duration in seconds (default: 7200)
- `--bot-name`: Name displayed in the meeting
- `--speak`: Enable voice interaction (`true`/`false`)

---

## 🐳 Docker Deployment (Production)

For production, the bot runs in a Docker container with a virtualized audio stack (PulseAudio). This eliminates the need for physical audio devices or Virtual Cables.

### 1. Build Options

**Option A: Lightweight (Recording Only)**
Best for transcription and recording. Smaller image size (~500MB).
```bash
docker build -t gmeet-bot:lite .
```

**Option B: Full Voice Assistant**
Includes Pipecat and AI voice dependencies. Larger image size (~800MB).
```bash
docker build --build-arg ENABLE_VOICE=true -t gmeet-bot:voice .
```

### 2. Running with Docker

**Run a specific meeting:**
```bash
# For recording only
docker run --rm \
  --env-file .env \
  -v $(pwd)/out:/app/out \
  --shm-size=2g \
  gmeet-bot:lite "https://meet.google.com/abc-defg-hij"

# For voice assistant
docker run --rm \
  --env-file .env \
  -v $(pwd)/out:/app/out \
  --shm-size=2g \
  gmeet-bot:voice "https://meet.google.com/abc-defg-hij" --speak true
```

**Using Docker Compose:**
```bash
# Start the service (defined in docker-compose.yml)
docker-compose up -d

# Check logs
docker-compose logs -f
```

### Audio Architecture (Docker)
The container creates two virtual PulseAudio sinks:
- **MeetOutput**: Chrome audio → FFmpeg recording
- **BotMic**: Bot audio → Chrome microphone input

---

## 🎧 Audio Setup (Windows)

To prevent audio feedback loops (bot hearing itself) and ensure clean audio routing, this bot requires a split audio setup using **VoiceMeeter** and **VB-Cable**.

### Prerequisites
1.  **VoiceMeeter** (Standard or Potato/Banana) installed.
2.  **VB-Cable** (Virtual Audio Cable) installed.

### Configuration Steps

1.  **Windows Sound Settings**:
    *   Open **Sound Mixer Options** (App volume and device preferences).
    *   Locate **Google Chrome** (or your browser).
    *   Set **Output** to `VoiceMeeter Input (VB-Audio Voicemeeter VAIO)`.
    *   Set **Input** to `CABLE Output (VB-Audio Virtual Cable)`.

2.  **VoiceMeeter**:
    *   Ensure `VoiceMeeter Input` is active (this receives Chrome's audio).
    *   Route this input to **B1** (Virtual Output).
    *   The bot listens to `Voicemeeter Out B1`.

3.  **Bot Output**:
    *   The bot speaks into `CABLE Input (VB-Audio Virtual Cable)`.
    *   Since Chrome's Input is set to `CABLE Output`, the meeting participants will hear the bot.

This setup ensures the bot hears the meeting (via VoiceMeeter) but does not hear its own voice (which goes to VB-Cable), preventing echo loops.

---

## 🤝 Contributing

We welcome contributions! Please see our [Contributing Guidelines](./CONTRIBUTING.md) for details.

---

## 🔐 Security

Please refer to [SECURITY.md](./SECURITY.md) for information about reporting security vulnerabilities and best practices.

---

## 🆙 Upgrading

For version compatibility and migration steps, see [UPGRADE.md](./UPGRADE.md).

---

## 📜 Code of Conduct

We follow a standard of respectful communication and collaboration. Please review our [Code of Conduct](./CODE_OF_CONDUCT.md) before contributing.

---

## 📝 License

This project is licensed under the [GNU General Public License v3.0 (GPL-3.0)](LICENSE)  — see the LICENSE file for details.

<div align="center">
  Made with ❤️ by HiCapy team | Powered by CueMeet
</div>