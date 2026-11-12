# Raspberry Pi 5 Local Voice Assistant (No Cloud)

A voice assistant that runs entirely on one Raspberry Pi 5. Say the wake word, and it turns on a light or answers a question out loud with Google's Gemma 4 model. Nothing leaves your network.

📺 Video: https://youtu.be/aLy3PcJ3IDc
📄 Full written guide: https://pidiylab.com/gemma-4-home-assistant-on-raspberry-pi-5-step-by-step-guide/

## How it works

```
Speakerphone mic → Wake word → Whisper (voice to text) → Home Assistant
    ├─ exact device name → Home Assistant runs it instantly
    └─ anything else     → Gemma 4 (Ollama) thinks about it
→ Piper (text to voice) → Speakerphone speaker
```

Four programs run in Docker (this compose file): Home Assistant, Ollama, Whisper, Piper.
Two run natively on the Pi as systemd services: the wake word and the satellite (there is no official Docker image for the satellite).

## Hardware

- Raspberry Pi 5, **8GB** (do not go smaller)
- Official 27W USB-C power supply (a weaker supply caused crashes)
- Official active cooler
- USB speakerphone (tested with EMEET M0 Plus)
- A smart plug or light in Home Assistant (tested with a TP-Link plug)
- microSD card or NVMe SSD with Raspberry Pi OS Lite (64-bit)

## Files

| File | What it is |
|---|---|
| `docker-compose.yml` | Home Assistant, Ollama, Whisper (small-int8), Piper (en_US-lessac-medium) |
| `systemd/wyoming-openwakeword.service` | Wake word service (`okay_nabu`, threshold 0.3) |
| `systemd/wyoming-satellite.service` | Speakerphone satellite on port 10700 |

## Quick start

1. Flash Raspberry Pi OS Lite (64-bit) and install Docker.
2. Clone this repo and start the stack:
   ```bash
   git clone https://github.com/YOUR_GITHUB/pi5-local-voice-assistant.git
   cd pi5-local-voice-assistant
   docker compose up -d
   docker exec -it ollama ollama pull gemma4:e2b
   ```
3. Open Home Assistant at `http://<pi-ip>:8123` and add, using the Pi's IP address:
   - **Ollama** → `http://<pi-ip>:11434`, then **Add conversation agent**
   - **Wyoming Protocol** → port `10300` (Whisper)
   - On the Wyoming card, **Add service** → port `10200` (Piper)
4. Install the wake word and satellite natively (clone `rhasspy/wyoming-openwakeword` and `rhasspy/wyoming-satellite` into your home folder and run `script/setup` in each), then:
   ```bash
   sudo cp systemd/*.service /etc/systemd/system/
   sudo nano /etc/systemd/system/wyoming-*.service   # replace YOUR_USER and the card name
   sudo systemctl daemon-reload
   sudo systemctl enable --now wyoming-openwakeword wyoming-satellite
   ```
5. In Home Assistant, add the satellite by the Pi's IP on port `10700` and give it your voice assistant.

## Settings that matter

Ollama conversation agent options:

| Setting | Value | Why |
|---|---|---|
| Keep alive | `-1` | Keeps the model loaded between questions |
| Context window | `8192` | At 4096 Gemma *said* it turned the light on but never did |
| Max history messages | `2` | At 20, every answer took about 37 seconds |
| Thinking | Off | Faster replies |
| Device control | Checked | Lets Gemma control exposed devices |
| Home Assistant | **Unchecked** | Returned `Unauthorized` in testing |

Only expose the devices you need. Testing was done with one exposed device; check memory with `free -h` as you add more.

## Troubleshooting

| Problem | Fix |
|---|---|
| Pi crashes or reboots under load | `vcgencmd get_throttled` — a value ending in `5005` means under-voltage. Use the official 27W supply and remove extra HATs. |
| Whisper / Piper missing from the dropdowns | Click **Add service** on the Wyoming Protocol card. |
| "Installation of apps is not supported" | Normal on Docker. Add the Wyoming services by hand. |
| Wake word never triggers | The name is `okay_nabu`, not `ok_nabu`. Lower the threshold to `0.3`. |
| Speakerphone goes silent after changing the agent | `sudo systemctl restart wyoming-openwakeword` then `sudo systemctl restart wyoming-satellite` |
| First answer takes a minute | The model is loading. Keep alive `-1` keeps it loaded after that. |
| Gemma says it did something but nothing happens | Raise the context window to `8192`. |

## Change the wake word

Built-in models include `okay_nabu`, `hey_jarvis`, `hey_mycroft`, `hey_rhasspy` and `alexa`. Change the name in **both** service files (`--preload-model` and `--wake-word-name`), then restart both services. Custom wake words can be trained in Google Colab; see the openWakeWord project.

## Credits

- [Home Assistant](https://www.home-assistant.io/)
- [Ollama](https://ollama.com/)
- [Wyoming satellite](https://github.com/rhasspy/wyoming-satellite)
- [Wyoming openWakeWord](https://github.com/rhasspy/wyoming-openwakeword)
- [Wyoming Whisper](https://github.com/rhasspy/wyoming-faster-whisper) and [Wyoming Piper](https://github.com/rhasspy/wyoming-piper)

MIT License. Made by [The Pi Stack Lab](https://www.youtube.com/@thepistacklab).
