# Smart Alarm Hub — CMPE 195B Senior Project

A Raspberry Pi-powered smart alarm clock and home automation hub built on [Home Assistant](https://www.home-assistant.io/). It integrates Matter-enabled smart devices, a customizable scheduler, and a fully local voice assistant pipeline — all controlled from a touchscreen dashboard.

---

## Table of Contents

- [What It Does](#what-it-does)
- [Hardware Requirements](#hardware-requirements)
- [Architecture Overview](#architecture-overview)
- [Getting Started](#getting-started)
  - [1. Install Raspberry Pi OS](#1-install-raspberry-pi-os)
  - [2. Install Docker Engine](#2-install-docker-engine)
  - [3. Deploy Home Assistant Container](#3-deploy-home-assistant-container)
  - [4. Apply This Configuration](#4-apply-this-configuration)
- [Feature Setup](#feature-setup)
  - [Alarm Clock](#alarm-clock)
  - [Light Scheduler](#light-scheduler)
  - [Voice Assistant](#voice-assistant)
- [Project Structure](#project-structure)
- [Contributing](#contributing)
- [Authors](#authors)

---

## What It Does

| Feature | Description |
|---|---|
| **Smart Alarm Clock** | Per-day alarms with snooze/stop popups and audio playback via BrowserMod |
| **Light Scheduler** | Save up to 3 light brightness/color-temperature presets and schedule them |
| **Voice Assistant** | Wake-word detection -> speech-to-text -> LLM response -> text-to-speech, all running locally |
| **Matter Integration** | Commission and control any Matter-certified smart device through Home Assistant |
| **Touch Dashboard** | Full-featured Lovelace UI optimized for the 5-inch Raspberry Pi display |

---

## Hardware Requirements

| Component | Spec Used |
|---|---|
| Single-board computer | Raspberry Pi 3 Model B+ |
| Storage | 128 GB microSD card |
| Microphone | Generic USB microphone |
| Speaker | 3.5 mm aux speaker |
| Display | Official 5-inch Raspberry Pi touchscreen |
| Secondary computer | Any Windows/Linux machine (for voice services) |

![Hardware implementation front view](https://github.com/user-attachments/assets/7c61d234-c8f7-4c76-a245-99bfbaf7bf3f)
![Hardware implementation side view](https://github.com/user-attachments/assets/e6cc2abc-3600-4866-b90d-697d0a20c307)

> **Note:** The RPi 3 lacks the processing power to run all voice services locally. A secondary computer running WSL2 or native Linux is required for the AI voice pipeline.

---

## Architecture Overview

```
+-------------------------------------+      +----------------------------------+
|         Raspberry Pi 3              |      |      Secondary PC (WSL2)         |
|                                     |      |                                  |
|  +------------------------------+   |      |  +--------------------------+    |
|  |  Home Assistant (Docker)     |<--+------+->|  Wyoming MicrowakeWord   |    |
|  |  - Alarm Clock Automations   |   |      |  |  (port 10400)            |    |
|  |  - Light Scheduler           |   |      +--+  Wyoming Faster Whisper  |    |
|  |  - Matter Integration        |   |      |  |  (port 10300)            |    |
|  |  - Lovelace Dashboard        |   |      +--+  Wyoming Piper TTS       |    |
|  +------------------------------+   |      |  |  (port 10200)            |    |
|                                     |      +--+  Ollama / gemma3:1b      |    |
|  +------------------------------+   |      |  |  (port 11434)            |    |
|  |  Wyoming Satellite (local)   |<--+------+  +--------------------------+    |
|  |  (port 10700)                |   |                                         |
|  +------------------------------+   |                                         |
+-------------------------------------+
```

---

## Getting Started

### 1. Install Raspberry Pi OS

Download and install [Raspberry Pi Imager](https://www.raspberrypi.com/software/). Flash **Raspberry Pi OS 64-bit (Debian 12)** to your microSD card, selecting the RPi 3 as the target device.

![Raspberry Pi OS Install](https://github.com/user-attachments/assets/62f728f9-0f50-4e1d-bcfd-d3fb5d24c8d4)

### 2. Install Docker Engine

Follow the [official Docker Engine install guide for Raspberry Pi OS](https://docs.docker.com/engine/install/raspberry-pi-os/#install-using-the-repository).

### 3. Deploy Home Assistant Container

```bash
docker run -d \
  --name homeassistant \
  --privileged \
  --restart=unless-stopped \
  -e TZ=YOUR_TIMEZONE \
  -v /PATH_TO_YOUR_CONFIG:/config \
  -v /run/dbus:/run/dbus:ro \
  --network=host \
  ghcr.io/home-assistant/home-assistant:stable
```

Replace `YOUR_TIMEZONE` with a [tz database name](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) (e.g., `America/Los_Angeles`) and `/PATH_TO_YOUR_CONFIG` with the absolute path where you want to store config files. The `/run/dbus` mount enables Bluetooth; omit it if your device has no Bluetooth.

![Home Assistant version info](https://github.com/user-attachments/assets/751f5bfe-e882-49fb-99e3-f1b26f38cf5b)

The dashboard will be accessible at `http://<your-pi-ip>:8123`.

### 4. Apply This Configuration

```bash
# Clone this repository into your Home Assistant config directory
git clone https://github.com/<your-org>/Senior-Project-main /PATH_TO_YOUR_CONFIG

# Restart Home Assistant to pick up the new configuration
docker restart homeassistant
```

After restart, verify that the required integrations (HACS, BrowserMod, Scheduler, Alexa Media Player) are loaded under **Settings -> Devices & Services**.

---

## Feature Setup

### Alarm Clock

The alarm uses Home Assistant **Helpers** (datetime + boolean inputs), **Scripts** for time adjustment, and **Automations** to trigger audio playback via the BrowserMod integration.

**After the base install:**

1. Install **BrowserMod** via HACS.
2. Open the Home Assistant dashboard in a browser on the device that will play audio.
3. Go to **Settings -> BrowserMod -> Register** and copy the generated Device ID.
4. In `automations.yaml`, replace every placeholder device reference in the *Alarm Trigger* automation with your copied Device ID.
5. In your browser's **Profile -> Personal settings**, set the browser to **not disconnect after inactivity**.

The alarm supports per-day scheduling (Monday–Sunday) with independent time helpers for each day. Snooze increments the alarm by 1 minute; Stop disables it entirely. All automation code is in `automations.yaml` and all scripts are in `scripts.yaml`.

---

### Light Scheduler

Allows saving up to 3 brightness/color-temperature presets and scheduling them to run automatically via the [Scheduler Component](https://github.com/nielsfaber/scheduler-component) and [Scheduler Card](https://github.com/nielsfaber/scheduler-card).

**Installation:** Install **HACS** and then install **scheduler-component** (v3.3.8+) and **scheduler-card** (v3.12.3+) through HACS.

#### Input Helper Creation

![Scheduler card and light configuration cards on the dashboard](https://github.com/user-attachments/assets/afe6aefe-ff1e-42ed-9246-823eb1cb922a)

This shows the Scheduler card along with the cards created for light configuration.

![Input helpers list in Home Assistant](https://github.com/user-attachments/assets/a4671082-bf1e-49b0-9258-788fc0a1639b)

Create input helpers in **Settings -> Devices & Services -> Helpers** to store values for each configuration. These are variables that take user input and store them.

![Input helpers displayed as sliders and dropdown on the dashboard](https://github.com/user-attachments/assets/1f365a05-3758-4012-9f2a-7c611dc35ba2)

The dashboard card presents these helpers as sliders for level selection and a dropdown to choose which config slot to save to.

![Light brightness input helper settings](https://github.com/user-attachments/assets/01d71cc4-9024-40f3-aac8-18c4a370733e)

Settings for the light brightness input helper. This captures the user's desired brightness value before saving it to a config.

![Light color temperature input helper settings](https://github.com/user-attachments/assets/c50e47af-43a0-40e7-aa1d-0b6d7d068515)

Settings for the light color temperature input helper. This captures the user's desired color temp before saving it to a config.

![Input number helper for config 1 color temp](https://github.com/user-attachments/assets/270dc098-8ae3-4b02-9917-4cb3bb25ea37)

An input number helper used to persist the selected color temperature for config slot 1.

![Input number helper for config 1 brightness](https://github.com/user-attachments/assets/cb24c244-a8c3-4979-a484-46b08bf1a6c2)

An input number helper used to persist the selected brightness for config slot 1.

A **Save Current Light Settings** button on the dashboard runs a script that writes the staged values to the chosen config slot. The button and its configuration can be created via the HA UI or found in the `.storage` folder.

#### Script Creation

The scripts used can be found in `scripts.yaml`.

![Apply config scripts listed in Home Assistant](https://github.com/user-attachments/assets/ef3b93df-2f61-41ef-a5a6-5b19680c762c)

These scripts read values from their respective config input helpers and call a `light.turn_on` action with the saved brightness and color temperature. Update the `entity_id` in each script to match your Matter-connected light.

#### Adding the Scheduler Card to the Dashboard

![Scheduler card appearing in the custom card section of the HA UI](https://github.com/user-attachments/assets/5c7c9c01-83ef-4e73-ba26-31cb2b66fb85)

The scheduler card can be added to any Lovelace dashboard via the HA UI. It appears under the custom card section.

![Customized scheduler card showing config scripts](https://github.com/user-attachments/assets/749a369d-fd7a-4368-9c92-0b335722a40b)

The card is customized to display the apply-config scripts as selectable actions. Adjust the card YAML to control which entities and actions appear. The full card YAML can be found in the main dashboard configuration in `.storage`.

![Scheduling a light config script to run at a specific time](https://github.com/user-attachments/assets/44fc75a6-3138-41b8-a953-8428004b8a5d)

Scheduling one of the light config scripts to run at a specific time.

---

### Voice Assistant

A fully local voice pipeline using the [Wyoming protocol](https://www.home-assistant.io/integrations/wyoming/). Because the RPi 3 cannot run the AI services, they run on a secondary computer via WSL2.

**Required components:**

- Wyoming integration (in HA)
- Ollama integration (in HA)
- Wyoming Satellite — relays audio/text between voice services (runs on RPi)
- Wyoming MicrowakeWord — wake word detection (runs in WSL2)
- Wyoming Faster Whisper — speech-to-text (runs in WSL2)
- Wyoming Piper — text-to-speech (runs in WSL2)
- Ollama — conversation agent, uses its own HA integration rather than Wyoming (runs in WSL2)

#### WSL2 Setup (Secondary PC)

1. [Install WSL2](https://learn.microsoft.com/en-us/windows/wsl/install) and Docker Engine inside it.
2. Enable [mirrored mode networking](https://learn.microsoft.com/en-us/windows/wsl/networking) and create firewall rules to forward ports **10200, 10300, 10400, and 11434**.
3. Find the WSL2 host IP:
   ```bash
   hostname -I
   ```

#### Wyoming Protocol Integration and Running Voice Services in WSL2

![Wyoming protocol integration page in Home Assistant](https://github.com/user-attachments/assets/902aa9c2-6858-41f8-8a9d-557bee64d0b5)

Add the Wyoming protocol integration in HA via **Settings -> Integrations**.

Start each voice service in WSL2:

```bash
# Wake-word detection
sudo docker run -it -p 10400:10400 rhasspy/wyoming-microwakeword --debug

# Speech-to-text (Whisper)
sudo docker run -it -p 10300:10300 \
  -v /path/to/local/data:/data \
  rhasspy/wyoming-whisper --model base --language en

# Text-to-speech (Piper)
sudo docker run -it -p 10200:10200 \
  -v /path/to/local/data:/data \
  rhasspy/wyoming-piper --voice en_US-lessac-medium --debug
```

#### Wyoming Satellite on the Raspberry Pi

Install [Wyoming Satellite](https://github.com/rhasspy/wyoming-satellite) on the RPi, then launch it — replacing `<WSL2_IP>` with the IP from `hostname -I`:

```bash
script/run \
  --name "wyoming satellite" \
  --uri "tcp://0.0.0.0:10700" \
  --mic-command "arecord -D plughw:1,0 -f S16_LE -c 1 -r 16000" \
  --snd-command "aplay -D plughw:2,0 -f S16_LE -c 1 -r 22050 -t raw" \
  --wake-uri "tcp://<WSL2_IP>:10400" \
  --wake-word-name "okay_nabu"
```

![Adding Wyoming services to the integration in Home Assistant](https://github.com/user-attachments/assets/cca1f2eb-a875-4406-8f75-1d42e531619f)

On the Wyoming integration page in HA, click **Add service** for each component. For the satellite (running on the RPi), use host `0.0.0.0` and port `10700`. For the WSL2 services, use the WSL2 host IP and each service's respective port.

![All Wyoming services connected in Home Assistant](https://github.com/user-attachments/assets/f6b91df6-efe2-4ebc-a459-54b9ea10b502)

Once all services are registered, the Wyoming integration page should look like this.

#### Ollama Install and Integration

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

![Ollama integration page in Home Assistant](https://github.com/user-attachments/assets/d7ca9319-1abd-4793-92a9-a396fb7771d7)

Load the Ollama integration in HA. Find the WSL2 hostname with `hostname -I`, then click **Add service** and enter `http://<WSL2_IP>:11434`. When prompted, select the **gemma3:1b** model.

#### Voice Assist Pipeline Setup

![Add new assistant button in Home Assistant Voice Assistants settings](https://github.com/user-attachments/assets/de169872-5155-48a3-9271-1aca454a371b)

Go to **Settings -> Voice Assistants** and click **Add assistant**.

![Voice assistant pipeline configuration — STT and TTS settings](https://github.com/user-attachments/assets/30636dc0-5c24-4d0e-9dda-a91e3cd80974)
![Voice assistant pipeline configuration — conversation agent settings](https://github.com/user-attachments/assets/2ced8c7e-12c6-463f-9c25-9b2c68e5b6fa)

Configure the pipeline to use Faster Whisper (STT), Piper (TTS), and Ollama (conversation agent). Expose the devices you want voice-controllable through the assistant settings.

Once configured, HA built-in commands are handled natively per the [built-in voice sentences reference](https://www.home-assistant.io/voice_control/builtin_sentences/). Unrecognized queries are forwarded to Ollama and the response is spoken aloud through Piper.

---

## Project Structure

```
.
├── automations.yaml          # All HA automations (alarm, light config sync)
├── blueprints/
│   ├── automation/           # Reusable automation templates
│   ├── script/               # Reusable script templates
│   └── template/             # Reusable sensor/template blueprints
├── configuration.yaml        # Core HA config (integrations, media player, logger)
├── custom_components/
│   ├── alexa_media/          # Alexa Media Player integration
│   ├── browser_mod/          # BrowserMod (audio playback, popups)
│   ├── hacs/                 # Home Assistant Community Store
│   ├── scheduler/            # Scheduler component
│   └── sox/                  # SoX audio processing
├── scenes.yaml               # HA scenes
├── scripts.yaml              # Alarm time scripts, light config apply scripts
├── secrets.yaml              # Credentials (do not commit real values)
└── www/
    ├── alarm_sound.mp3       # Default alarm audio
    ├── scheduler-card.js     # Scheduler Lovelace card
    └── content-card-example.js
```
---

---

## Authors

This project was developed as part of **CMPE 195 — Senior Design Project** at San Jose State University.
