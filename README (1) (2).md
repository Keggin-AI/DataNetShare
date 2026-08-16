# DataNet Share

> **This project was built with AI assistance using GLM. The app works — sorry if the code says anything stupid.**

Share your Android phone's mobile data with other devices — without using your carrier's hotspot quota.

DataNet Share turns an Android phone with cellular data into a SOCKS5 proxy gateway. Other phones, tablets, or computers connect over **Wi-Fi Direct** (or USB/Wi-Fi for PC/Mac), and route their traffic through the host phone's cellular connection. Because the host phone simply appears to be using its own data — which it is — carrier hotspot caps never come into play.

- **Version:** 5.0 (unified)
- **Min Android:** 8.0 (API 26)
- **Language:** Kotlin (pure — no native code)
- **Build:** Gradle 8.2 + Android SDK 34

---

## Features

- **One APK for both phones** — install the same app on sender and receiver
- **VPN-aware** — routes proxied traffic through your VPN when active (carrier sees only encrypted traffic)
- **Link speed test** — measures the Wi-Fi Direct link speed between devices (10MB payload, no internet needed)
- **Full TCP + UDP support** — apps, browsing, streaming, gaming, voice all work
- **No root required** — uses Android's VpnService and Wi-Fi Direct APIs
- **Survives screen-off** — partial wake lock keeps the connection alive
- **Privacy-focused** — Cloudflare DNS (1.1.1.1), notification hides credentials on lock screen, no telemetry

---

## Download & install

Download the latest APK from [Releases](../../releases), then:

```bash
adb uninstall com.datanet.share   # remove any old version
adb install DataNetShare-v5.0-unified.apk
```

Install the **same APK on both phones** — sender and receiver.

---

## Quick start

### Sender phone (the one with data):

1. Turn ON your VPN (e.g., Google VPN) if you want carrier privacy
2. Open the app → tap **START SHARING (WiFi Direct)**
3. Note the SSID and password shown in the blue card

### Receiver phone (the one getting data):

1. Go to Android Settings → Wi-Fi → Wi-Fi Direct
2. Tap the sender's network → enter the password
3. Open the app → tap **START RECEIVING**
4. Grant VPN permission when prompted

That's it. The receiver now has internet through the sender's cellular connection.

### Link speed test:

On the receiver, tap **LINK SPEED TEST** after connecting. It downloads a 10MB payload from the sender through the SOCKS5 proxy and reports the link speed in Mbps.

---

## Usage modes

### Mode 1 — Wi-Fi Direct (phone → phone)

Best for sharing between two phones. Creates a Wi-Fi Direct group with a separate SSID and password. The sender becomes the group owner at `192.168.49.1` and runs a SOCKS5 proxy on port 8282.

### Mode 2 — Local-only (phone → PC/Mac)

Best for sharing with a computer. Uses USB tethering or an existing Wi-Fi network — no Wi-Fi Direct needed. Tap **START LOCAL ONLY**, then set a SOCKS5 proxy on your PC/Mac pointing to the phone's IP on port 8282.

**Mac:** System Settings → Network → Details → Proxies → SOCKS Proxy  
**Firefox:** Settings → Network Settings → Manual proxy → SOCKS Host: phone's IP, Port: 8282, check "Proxy DNS when using SOCKS v5"

### Mode 3 — Receiver

The receiving phone runs a VpnService that captures all traffic and tunnels it through the sender's SOCKS5 proxy over the Wi-Fi Direct link. No manual proxy configuration needed — just tap START RECEIVING.

---

## Troubleshooting

**"createGroup failed: BUSY"** — Quick Share / Nearby Share may be holding Wi-Fi Direct. Disable both, toggle Wi-Fi, retry.

**Nothing loads on receiver** — Make sure the sender's mobile data is ON and the sender shows "active" status. The receiver's notification should say "VPN tunnel active."

**Speeds are slow** — Wi-Fi Direct speeds vary by device. The link speed test button can help diagnose. For best speeds, keep both phones close together.

**Mac shows "no internet"** — Normal. macOS checks the default route, not the proxy. Set the SOCKS5 proxy and it works.

**VPN not capturing proxied traffic** — Stop and restart sharing after turning the VPN on/off. The app checks VPN state when sharing starts.

---

## Build from source

Requires JDK 17, Android SDK 34, and Gradle 8.5+:

```bash
git clone https://github.com/Keggin-AI/DataNetShare.git
cd DataNetShare
./gradlew assembleRelease
```

Output: `app/build/outputs/apk/release/app-release.apk`

---

## Privacy

- No telemetry, no analytics, no trackers
- No traffic contents logged (only byte counts)
- DNS uses Cloudflare (1.1.1.1) instead of Google
- Wi-Fi Direct password hidden on lock screen
- When VPN is active on the sender, the carrier sees only encrypted traffic to your VPN endpoint — not the receiver's destinations, DNS queries, or SNI

A full security audit report is included in the repository.

---

## License

MIT — do whatever you want with it.
