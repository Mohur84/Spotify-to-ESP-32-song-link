# Spotify Now Playing OLED Display (ESP32)

Two files:
- `spotify_oled_arduino.ino` — flashes onto the ESP32, drives the OLED
- `spotify_display_sunc.py` — runs on your PC, talks to Spotify, pushes data to the ESP32

## Step 1: Wire the OLED to the ESP32

| OLED | ESP32   |
|------|---------|
| VCC  | 3.3V    |
| GND  | GND     |
| SDA  | GPIO 21 |
| SCL  | GPIO 22 |

## Step 2: Install Arduino libraries

In Arduino IDE → Tools → Manage Libraries, install:
- Adafruit SSD1306
- Adafruit GFX
- (ESP32 board package must already be installed, via Boards Manager, for `WiFi.h`/`WebServer.h`)

## Step 3: Edit and upload `spotify_oled_arduino.ino`

Open the file and change:
```cpp
const char* WIFI_SSID     = "YOUR_WIFI_NAME";
const char* WIFI_PASSWORD = "YOUR_WIFI_PASSWORD";
```
Upload it to the ESP32, then open the Serial Monitor (115200 baud). Once it connects you'll see:
```
WiFi Connected!
IP Address: 192.168.0.104
```
**Write this IP address down** — you need it in Step 6.

If the OLED shows nothing, try changing `SCREEN_ADDRESS` from `0x3C` to `0x3D` in the sketch — that's the other common I2C address for these boards.

## Step 4: Create a Spotify Developer app

1. Go to the [Spotify Developer Dashboard](https://developer.spotify.com/dashboard).
2. Log in, click **Create app**.
3. Give it any name/description.
4. Add a Redirect URI: `http://.........`
   (Spotify no longer accepts the hostname `localhost` for `http://` URIs — only the literal loopback IP `127.0.0.1` works now.)
5. Under "Which API/SDKs are you planning to use?" check **Web API**. Note: as of Feb 2026, Spotify requires the app owner's account to have an active Premium subscription to use the Web API — if the checkbox is greyed out, that's why.
6. Save, then copy the **Client ID** and **Client Secret**.

## Step 5: Install Python dependencies

```bash
pip install spotipy requests
```

## Step 6: Edit `spotify_display_sync.py`

Fill in:
```python
CLIENT_ID     = "YOUR_SPOTIFY_CLIENT_ID"
CLIENT_SECRET = "YOUR_SPOTIFY_CLIENT_SECRET"
REDIRECT_URI  = "http://........."   # must match dashboard exactly
ESP32_IP = "192.168.0.104"                          # the IP from Step 3
```

## Step 7: Run it

```bash
python spotify_display_sync.py
```

The first run opens a browser window asking you to log in to Spotify and authorize the app. After that, it caches a token locally and won't ask again (until it expires).

You should see console output like:
```
Sent: Blinding Lights - The Weeknd (Playing)
```
and the OLED should update within a second or two.

## Step 8: Play something on Spotify

Open Spotify on any device logged into your account (phone, desktop, web player) and hit play. The `current_playback()` call picks up whatever device is active on your account — it doesn't have to be the same machine running the script.

## How the two files talk to each other

```
Spotify App
    │
    ▼
Spotify Web API  ──(polled every 1s)──▶  spotify_display_sync.py (your PC)
                                              │
                                              │ HTTP POST /update
                                              │ title, artist, duration,
                                              │ progress, playing
                                              ▼
                                          spotify_oled_arduino.ino (ESP32)
                                              │
                                              ▼
                                          OLED Display
```

- The Python script polls Spotify every second and only *sends* to the ESP32 when the song changes, play/pause toggles, or every ~5 seconds (to keep the progress bar honest) — this avoids flooding the ESP32 with HTTP requests.
- The ESP32 also interpolates progress locally between updates, so the progress bar moves smoothly instead of jumping once a second.
- If the ESP32 hasn't heard from the PC in 15 seconds, it shows "Waiting for PC..." instead of stale data.

## Troubleshooting

- **OLED stays blank**: check wiring, and try I2C address `0x3D` instead of `0x3C`.
- **ESP32 never gets an IP**: double-check SSID/password; ESP32 only supports 2.4GHz Wi-Fi, not 5GHz networks.
- **Python script can't reach the ESP32**: make sure your PC and ESP32 are on the same Wi-Fi network, and that the IP in `spotify_display_sync.py` matches what the Serial Monitor printed (it can change if your router reassigns it — consider setting a DHCP reservation for the ESP32 in your router settings).
- **`current_playback()` returns None**: nothing is actively playing on any device logged into that Spotify account, or the account doesn't have the right permissions (Spotify's `currently-playing` endpoints work for both Free and Premium accounts for *reading* state).
- **Auth fails / redirect URI mismatch**: the `REDIRECT_URI` in your Python script must match, character-for-character, what's registered in the Spotify Dashboard.

## Next steps / ideas from the original feature list

The sketch above already includes: title, artist, time, progress bar, and playing/paused state, with scrolling for long titles. Not yet implemented (left as extensions):
- Album name
- Spotify logo bitmap on boot
- Wi-Fi signal icon
- Play/Pause icon instead of text
- Album art (would require switching to a color TFT display, since SSD1306 is monochrome)
