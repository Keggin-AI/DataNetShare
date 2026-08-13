# DataNet Share

THIS README ALONG WITH THE ENTIRETY OF THE SOURCE CODE WAS MADE WITH GLM5.2 SORRY IF ITS INACCURATE OR WRONG OR SAYS SOMETHING STUPID ALL I KNOW IS THE APP WORKS

Share your Android phone's mobile data with other devices — without using the carrier hotspot feature.

Bypasses carrier tethering detection by using WiFi Direct (a separate radio path from the carrier hotspot) and binding upstream sockets directly to the cellular network via `Network.getSocketFactory()`. No root required.

## The Problem

Carriers charge extra for tethering/hotspot, or cap it at low speeds, even when you have unlimited phone data. This app lets you share your phone's mobile data with:

- Another Android phone (via WiFi Direct)
- A PC or Mac (via USB tethering or WiFi)
- Any device that can use a SOCKS5 proxy

...without triggering the carrier's hotspot detection.

## Features

- **No root required** — uses Android's VpnService and WiFi Direct APIs
- **Three sharing modes:**
  - WiFi Direct mode — share to another Android phone
  - Local-only mode — share to PC/Mac via USB tethering or same WiFi
  - Receiver mode — connect to a sender phone
- **Full TCP support** — userspace TCP stack with proper SYN/FIN/ACK state machine
- **Full UDP support** — SOCKS5 UDP ASSOCIATE with NAT table (works with Discord voice, WebRTC, gaming)
- **Cellular-bound sockets** — upstream traffic forced via mobile data, not WiFi, using `Network.socketFactory`
- **Survives screen-off** — partial wake lock keeps the CPU running
- **Built-in UDP test** — STUN-based connectivity test
- **PC/Mac support** — any device with SOCKS5 proxy support can connect
- **Release-signed APK** — 100-year keystore validity

## Speeds (real-world, on a Pixel 10 Pro XL)

| Method | Download | Upload |
|--------|----------|--------|
| WiFi Direct (phone-to-phone) | ~80-90 Mbps | ~40-50 Mbps |
| USB tethering (phone-to-Mac) | ~120 Mbps | ~80 Mbps |
| WiFi local (phone-to-PC) | ~100 Mbps | ~50 Mbps |

Your mileage will vary based on cellular signal and the device you're sharing to.

## How to install

1. Download `DataNetShare-v3.3-release.apk` from the [releases page](releases)
2. Copy it to your Android phone (both phones if sharing phone-to-phone)
3. Open the APK and accept the "install unknown app" prompt
4. Open the app and grant all requested permissions (location, nearby WiFi devices, notifications)

The APK is signed with a release certificate. It will not install over a debug build — uninstall any old version first.

## How to use

### Phone-to-phone (WiFi Direct mode)

**On the sender phone (the one with mobile data):**
1. Make sure mobile data is ON and WiFi is ON
2. Open the app → tap **START SHARING (WiFi Direct)**
3. Note the SSID and password shown in the blue card
4. Wait for "Sharing on 192.168.49.1:8282"

**On the receiver phone:**
1. Android Settings → WiFi → WiFi Direct
2. Tap the sender's SSID → enter the password shown on the sender
3. Accept the pairing prompt on the sender
4. Open the app → tap **START RECEIVING**
5. Grant VPN permission when prompted
6. All apps on the receiver now route through the sender's mobile data

### Phone-to-PC/Mac (USB tethering — fastest)

**On the sender phone:**
1. Open the app → tap **START LOCAL ONLY (USB/WiFi)**
2. Settings → Network & internet → Hotspot & tethering → enable **USB tethering**
3. Back in the app → tap **REFRESH INTERFACES**
4. Find the USB interface (often called `ncm0` or `rndis0`) and note its IP (e.g., `192.168.42.129`)

**On your PC/Mac:**
1. Connect to the phone via USB cable
2. Set your SOCKS5 proxy to the phone's USB IP, port `8282`
   - **Mac:** System Settings → Network → USB → Details → Proxies → SOCKS Proxy
   - **Firefox (recommended):** Settings → Network Settings → Manual proxy → SOCKS Host: the IP, Port: 8282, select SOCKS v5, check "Proxy DNS when using SOCKS v5"
3. Browse — your traffic now goes via the phone's cellular connection

### Phone-to-PC/Mac (WiFi — no cable)

**On the sender phone:**
1. Connect to your home WiFi
2. Open the app → tap **START LOCAL ONLY**
3. Tap **REFRESH INTERFACES** → note the WiFi IP (e.g., `192.168.1.105`)

**On your PC/Mac (on the same WiFi):**
1. Set SOCKS5 proxy to the phone's WiFi IP, port `8282`
2. Browse — traffic goes PC → WiFi router (local) → phone → cellular → internet

Your slow home internet is bypassed entirely — WiFi is only used as a local cable between PC and phone.

## How it works

```
SENDER PHONE                           RECEIVER DEVICE
├── WifiDirectHost.kt                  ├── Tun2Socks.kt          (userspace IPv4/TCP/UDP stack)
│   └── Creates WiFi Direct group      │   ├── TcpFlow.kt        (per-connection TCP state)
│       (becomes 192.168.49.1)         │   └── SharedUdpFlow.kt  (single UDP associate)
├── Socks5Server.kt                    │
│   ├── TCP CONNECT handler            ├── ReceiverVpnService.kt (VpnService, TUN fd)
│   └── UDP ASSOCIATE handler          │
│       (two-socket: relay + upstream) ├── ReceiverService.kt    (foreground orchestrator)
├── SenderService.kt                   │
│   └── Foreground service + wake lock └── MainActivity.kt       (UI + log + tests)
│
└── CellularNetworkTracker.kt
    └── NetworkCallback for cellular
        (socketFactory.createSocket for TCP)
```

### The key tricks

1. **WiFi Direct, not carrier hotspot** — WiFi Direct is a separate radio path. The carrier sees no hotspot signal, so tethering detection is bypassed.

2. **`Network.socketFactory.createSocket()`** — creates TCP sockets already bound to the cellular network at the kernel level. Traffic goes via mobile data even when WiFi is connected.

3. **Split `/1` routes** — Android 11+ has a kernel default route that beats `addRoute("0.0.0.0", 0)`. Splitting into `0.0.0.0/1` and `128.0.0.0/1` covers all IPv4 with two more-specific routes.

4. **`FileInputStream.read()` on TUN fd** — blocking reads that don't busy-loop. The standard `Os.read()` approach can return immediately with EAGAIN on some Android versions.

5. **Two-socket UDP relay** — the sender uses one UDP socket for WiFi Direct (receiver ↔ sender) and a separate one bound to cellular (sender ↔ internet). A single socket bound to cellular can't receive from WiFi Direct.

6. **`VpnService.protect()` on all bypass sockets** — SOCKS5 control connections and the local UDP relay socket are protected from the VPN so they don't loop back into the TUN.

## Building from source

### Requirements

- JDK 17 (Temurin recommended)
- Android SDK with:
  - platform-tools
  - platforms;android-34
  - build-tools;34.0.0
- Gradle 8.5+ (wrapper included)

### Build steps

1. Set environment:
   ```bash
   export JAVA_HOME=/path/to/jdk17
   export ANDROID_HOME=/path/to/android-sdk
   ```

2. Create `local.properties`:
   ```
   sdk.dir=/path/to/android-sdk
   ```

3. Build debug APK:
   ```bash
   ./gradlew assembleDebug
   ```
   Output: `app/build/outputs/apk/debug/app-debug.apk`

4. Build release APK (requires keystore):
   ```bash
   # Generate keystore (one-time)
   keytool -genkeypair -v -keystore keystore.jks \
     -keyalg RSA -keysize 2048 -validity 36500 -alias datanet \
     -storepass YOURPASS -keypass YOURPASS \
     -dname "CN=DataNet Share, OU=App, O=DataNet"

   # Edit app/build.gradle.kts signingConfigs.release to point to your keystore

   # Build
   ./gradlew assembleRelease
   ```
   Output: `app/build/outputs/apk/release/app-release.apk`

## Limitations

- **UDP other than DNS and STUN/TURN is not supported** — actually, UDP IS fully supported via SOCKS5 UDP ASSOCIATE. Discord voice, WebRTC, and gaming all work.
- **IPv6 is not supported** — only IPv4 packets are processed.
- **No TCP SACK / window scaling / PMTUD** — large transfers may be slower than native. Browsing, streaming, and messaging work normally.
- **macOS DNS** — the system SOCKS proxy doesn't proxy DNS by default. Use Firefox with "Proxy DNS when using SOCKS v5" checked, or use `curl --socks5-hostname`.

## Troubleshooting

**"createGroup failed: BUSY" (WiFi Direct mode)**
Quick Share / Nearby Share is holding WiFi Direct. Disable both, toggle WiFi, retry.

**Receiver can't connect (WiFi Direct mode)**
Forget the old WiFi Direct pairing on the receiver (long-press → Forget), then re-pair with the new password.

**MacBook shows "no internet" on USB (USB mode)**
That's normal — macOS checks the phone's default route (WiFi), not our proxy. Set the SOCKS5 proxy and your browser will work fine.

**Nothing loads on PC/Mac even with proxy set**
macOS's SOCKS proxy doesn't proxy DNS. Use Firefox with "Proxy DNS when using SOCKS v5" checked, or test with:
```bash
curl --socks5 PHONE_IP:8282 https://fast.com
```

**App crashes on receiver**
Use the FORCE STOP button on both phones, then restart.

## Privacy

This app:
- Does NOT collect or transmit any telemetry
- Does NOT log or inspect traffic contents (only byte counts for the status display)
- Routes ALL receiver traffic through the sender. The sender's carrier can see that traffic as if it originated from the sender phone.

## License

MIT — do whatever you want with it.

## Why this exists

Because carriers charging extra for tethering — a feature the phone already has — is bullshit. The hardware can share data. The software can share data. The carrier just flips a flag to disable it unless you pay. This app goes around the flag.
