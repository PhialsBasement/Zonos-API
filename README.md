# Zonos-v0.1

<div align="center">
<img src="assets/ZonosHeader.png" 
     alt="Alt text" 
     style="width: 500px;
            height: auto;
            object-position: center top;">
</div>

<div align="center">
  <a href="https://discord.gg/gTW9JwST8q" target="_blank">
    <img src="https://img.shields.io/badge/Join%20Our%20Discord-7289DA?style=for-the-badge&logo=discord&logoColor=white" alt="Discord">
  </a>
</div>

---

Zonos-v0.1 is a leading open-weight text-to-speech model trained on more than 200k hours of varied multilingual speech, delivering expressiveness and quality on par with—or even surpassing—top TTS providers.

Our model enables highly natural speech generation from text prompts when given a speaker embedding or audio prefix, and can accurately perform speech cloning when given a reference clip spanning just a few seconds. The conditioning setup also allows for fine control over speaking rate, pitch variation, audio quality, and emotions such as happiness, fear, sadness, and anger. The model outputs speech natively at 44kHz.

##### For more details and speech samples, check out our blog [here](https://www.zyphra.com/post/beta-release-of-zonos-v0-1)

##### We also have a hosted version available at [maia.zyphra.com/audio](https://maia.zyphra.com/audio)

---

Zonos follows a straightforward architecture: text normalization and phonemization via eSpeak, followed by DAC token prediction through a transformer or hybrid backbone. An overview of the architecture can be seen below.

<div align="center">
<img src="assets/ArchitectureDiagram.png" 
     alt="Alt text" 
     style="width: 1000px;
            height: auto;
            object-position: center top;">
</div>

---

## Usage

### Python

``python
import torch
import torchaudio
from zonos.model import Zonos
from zonos.conditioning import make_cond_dict

# model = Zonos.from_pretrained("Zyphra/Zonos-v0.1-hybrid", device="cuda")
model = Zonos.from_pretrained("Zyphra/Zonos-v0.1-transformer", device="cuda")

wav, sampling_rate = torchaudio.load("assets/exampleaudio.mp3")
speaker = model.make_speaker_embedding(wav, sampling_rate)

cond_dict = make_cond_dict(text="Hello, world!", speaker=speaker, language="en-us")
conditioning = model.prepare_conditioning(cond_dict)

codes = model.generate(conditioning)

wavs = model.autoencoder.decode(codes).cpu()
torchaudio.save("sample.wav", wavs[0], model.autoencoder.sampling_rate)
``

### Gradio interface (recommended)

``bash
uv run gradio_interface.py
# python gradio_interface.py
``

This should produce a `sample.wav` file in your project root directory.

_For repeated sampling we highly recommend using the gradio interface instead, as the minimal example needs to load the model every time it is run._

### API Usage (OpenAI-compatible)

Zonos provides an OpenAI-compatible API that allows you to generate speech through HTTP requests.

#### Setting up the API

``bash
# Clone the repository
git clone https://github.com/Zyphra/Zonos.git
cd Zonos

# Build and start the API service
docker compose up zonos-api
``

The API will be available at `http://localhost:8000`.

#### API Endpoints

The API provides the following main endpoints:

1. **Generate Speech**: `/v1/audio/speech` (POST)
   - Convert text to speech with optional voice cloning and emotion control

2. **Create Voice**: `/v1/audio/voice` (POST)
   - Upload a voice sample to use for voice cloning

3. **List Models**: `/v1/audio/models` (GET)
   - List available Zonos models

#### Example API Usage with cURL

**Creating a Voice:**
``bash
curl -X POST "http://localhost:8000/v1/audio/voice" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@path/to/voice/sample.mp3" \
  -F "name=my_voice"
``

**Generating Speech:**
``bash
curl -X POST "http://localhost:8000/v1/audio/speech" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Zyphra/Zonos-v0.1-transformer",
    "input": "Hello, this is a test of the Zonos API.",
    "voice": "voice_12345_0",
    "speed": 1.0,
    "language": "en-us",
    "emotion": {
      "happiness": 1.0
    },
    "response_format": "mp3"
  }' \
  --output speech.mp3
``

**Get Available Models:**
``bash
curl -X GET "http://localhost:8000/v1/audio/models"
``

#### Python Example

``python
import requests
import json

# Create a voice from audio sample
with open("voice_sample.mp3", "rb") as file:
    response = requests.post(
        "http://localhost:8000/v1/audio/voice",
        files={"file": file}
    )
voice_id = response.json()["voice_id"]

# Generate speech using the created voice
response = requests.post(
    "http://localhost:8000/v1/audio/speech",
    json={
        "model": "Zyphra/Zonos-v0.1-transformer",
        "input": "Hello, this is a test of the Zonos API.",
        "voice": voice_id,
        "speed": 1.0,
        "language": "en-us",
        "emotion": {
            "happiness": 1.0
        },
        "response_format": "mp3"
    }
)

# Save the generated audio
with open("output.mp3", "wb") as f:
    f.write(response.content)
``

## Features

- Zero-shot TTS with voice cloning: Input desired text and a 10-30s speaker sample to generate high quality TTS output
- Audio prefix inputs: Add text plus an audio prefix for even richer speaker matching. Audio prefixes can be used to elicit behaviours such as whispering which can otherwise be challenging to replicate when cloning from speaker embeddings
- Multilingual support: Zonos-v0.1 supports English, Japanese, Chinese, French, and German
- Audio quality and emotion control: Zonos offers fine-grained control of many aspects of the generated audio. These include speaking rate, pitch, maximum frequency, audio quality, and various emotions such as happiness, anger, sadness, and fear.
- Fast: our model runs with a real-time factor of ~2x on an RTX 4090 (i.e. generates 2 seconds of audio per 1 second of compute time)
- Gradio WebUI: Zonos comes packaged with an easy to use gradio interface to generate speech
- OpenAI-compatible API: Zonos provides a REST API for easy integration with existing applications
- Simple installation and deployment: Zonos can be installed and deployed simply using the docker file packaged with our repository.

## Installation

**At the moment this repository only supports Linux systems (preferably Ubuntu 22.04/24.04) with recent NVIDIA GPUs (3000-series or newer, 6GB+ VRAM).**

See also [Docker Installation](#docker-installation)

#### System dependencies

Zonos depends on the eSpeak library phonemization. You can install it on Ubuntu with the following command:

``bash
apt install -y espeak-ng
``

#### Python dependencies

We highly recommend using a recent version of [uv](https://docs.astral.sh/uv/#installation) for installation. If you don't have uv installed, you can install it via pip: `pip install -U uv`.

##### Installing into a new uv virtual environment (recommended)

``bash
uv sync
uv sync --extra compile
``

##### Installing into the system/actived environment using uv

``bash
uv pip install -e .
uv pip install -e .[compile]
``

##### Installing into the system/actived environment using pip

``bash
pip install -e .
pip install --no-build-isolation -e .[compile]
``

##### Confirm that it's working

For convenience we provide a minimal example to check that the installation works:

``bash
uv run sample.py
# python sample.py
``

## Docker installation

``bash
git clone https://github.com/Zyphra/Zonos.git
cd Zonos

# For gradio interface
docker compose up

# For API server
docker compose up zonos-api

# Or for development you can do
docker build -t zonos .
docker run -it --gpus=all --net=host -v /path/to/Zonos:/Zonos -t zonos
cd /Zonos
python sample.py # this will generate a sample.wav in /Zonos
``
