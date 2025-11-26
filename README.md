# 🎬 Kodi_Fenlight-Alexa_Skill

![Python](https://img.shields.io/badge/Python-3.9-blue?logo=python&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?logo=docker&logoColor=white)
![Platform](https://img.shields.io/badge/Platform-Nvidia%20Shield-76B900?logo=nvidia&logoColor=white)
![Language](https://img.shields.io/badge/Language-French%20Only-red)

**Dockerized middleware to start movies or series within FenLight Kodi add-on on Nvidia Shield via an Alexa skill.**

This project features intelligent power management (ADB/WoL), TMDB series and movie lookup, Trakt.tv resume sync, and automated Fen Light add-on patching to allow use by external players (TMDB Helper).

> ⚠️ **Note:** The Alexa interaction model and voice commands provided in this repository are currently available in **French only**.

## ✨ Key Features

* **🗣️ Natural Voice Control:** *"Alexa, demande à Mon Cinéma de lancer The Witcher."*
* **⚡ Smart Power Management:** Automatically wakes up the Nvidia Shield (WoL) and launches the Kodi app (ADB) before executing commands.
* **🧠 Trakt.tv Integration:** Smart resume features. Ask to "Resume [Show]" and it plays the specific *Next Up* episode from your Trakt history.
* **🔍 TMDB Search:** Accurate identification of media content (Movies vs Shows).
* **🛠️ Fen Light Auto-Patcher:** Includes a background scheduler that automatically patches the *Fen Light* addon to allow external calls (via TMDB Helper), ensuring playback works even after addon updates.
* **🎛️ Dual Playback Modes:** Choose between **Auto-Play** (instant launch) or **Source Select** (manual quality selection) via voice commands.

## 🚀 Prerequisites

* **Hardware:** Nvidia Shield TV (or Android TV with ADB debugging enabled).
* **Software:** Kodi (v19 or newer).
* **Addons:**
    * `plugin.video.themoviedb.helper` (TMDB Helper).
    * `plugin.video.fenlight` (Fen Light).
* **Accounts:**
    * [TMDB API Key](https://www.themoviedb.org/documentation/api).
    * [Trakt.tv Client ID](https://trakt.tv/oauth/apps) (for API access).

## 📦 Installation

### 1. Kodi Configuration (TMDB Helper Players)
You need to add two custom players to TMDB Helper to handle the "Auto" and "Manual" modes.
Create these `.json` files in your Kodi userdata folder: 
`/Android/data/org.xbmc.kodi/files/.kodi/userdata/addon_data/plugin.video.themoviedb.helper/players/`

* **fenlight_auto.json**: Standard URL for Fen Light playback (uses your default settings).
* **fenlight_select.json**: Same URL but with `&source_select=true` appended to force the source list.

### 2. Docker Deployment
This application is designed to run in a Docker container.

**Note on Network:** Using `network_mode: host` or a macvlan (like `br0` on Unraid) is **highly recommended**. Wake-on-LAN (WoL) magic packets often cannot pass through the standard Docker NAT bridge.

**docker-compose.yaml:**
```yaml
version: '3.8'

services:
  kodi-alexa-skill:
    build: .
    container_name: kodi-fenlight-alexa-skill
    restart: unless-stopped
    network_mode: host  # Required for WoL to work
    environment:
      - TZ=Europe/Paris
      - PYTHONUNBUFFERED=1
      
      # --- SHIELD CONFIG ---
      - SHIELD_IP=192.168.1.x
      - SHIELD_MAC=AA:BB:CC:DD:EE:FF
      
      # --- KODI CONFIG ---
      - KODI_PORT=8080
      - KODI_USER=kodi
      - KODI_PASS=kodi
      
      # --- API KEYS ---
      - TMDB_API_KEY=your_tmdb_api_key
      
      # --- TRAKT.TV ---
      - TRAKT_CLIENT_ID=your_trakt_client_id
      - TRAKT_ACCESS_TOKEN=your_trakt_access_token

      # --- TMDB HELPER PLAYERS (Must match filenames on Shield including .json) ---
      - PLAYER_DEFAULT=fenlight_auto.json
      - PLAYER_SELECT=fenlight_select.json
      
      # --- DEBUG ---
      - DEBUG_MODE=false
```

### 3. Alexa Skill Setup
1.  Go to the [Alexa Developer Console](https://developer.amazon.com/alexa/console/ask) and create a **Custom Skill**.
2.  **Invocation Name:** Choose something simple like "my cinema" or "mon cinéma".
3.  **Endpoint:** Point it to your server's public HTTPS URL (e.g., using Cloudflare Tunnel): 
    `https://your-domain.com/alexa-webhook`
4.  **Interaction Model:** Create the Intents (`PlayMovieIntent`, `PlayTVShowIntent`, `ResumeTVShowIntent`) using the utterance lists provided in this repository's `speech_assets` folder.
5.  **Slots:** Ensure you create a slot type named `SourceMode` to handle manual selection requests (values: "manually", "select source", "avec choix", etc.).

## 🗣️ Usage Examples (French)

Since this skill is currently localized for French users, here are the commands you should use:

| Action | Voice Command (French) |
| :--- | :--- |
| **Launch a Movie** | *"Alexa, demande à Mon Cinéma de lancer Avatar."* |
| **Launch a Show** | *"Alexa, demande à Mon Cinéma de lancer The Witcher."* |
| **Specific Episode** | *"Alexa, demande à Mon Cinéma de lancer Friends, Saison 5 Épisode 10."* |
| **Resume (Trakt)** | *"Alexa, demande à Mon Cinéma de reprendre Breaking Bad."* |
| **Manual Select** | *"Alexa, demande à Mon Cinéma de lancer Inception **avec choix**."* |

## 🔧 Technical Details

### The Fen Light Auto-Patcher
Fen Light restricts external playback calls by default to prevent usage by widgets/skins it doesn't support. This middleware includes a background thread that:
1.  Connects to the Shield via ADB every hour.
2.  Pulls the `sources.py` file from the addon directory.
3.  Detects if the blocking code block (`WARNING: External Playback Detected`) is active.
4.  Patches the file (comments out the restriction) and pushes it back to the Shield transparently.

### Power Management
The script uses a hybrid approach to ensure the Shield is ready before sending the command:
1.  **Wake-on-LAN:** Sends a magic packet to wake the network interface.
2.  **ADB Wake:** Sends standard Android `WAKEUP` key events via ADB.
3.  **ADB Start:** Forces the Kodi activity to launch if it's not already in the foreground.
