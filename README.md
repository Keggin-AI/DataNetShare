# DataNet Share

THIS README ALONG WITH THE ENTIRETY OF THE SOURCE CODE WAS MADE WITH GLM 5.2 IM SORRY IF IT SAYS ANYTHING STUPID OR WRONG ALL I KNOW IS THE APP WORKS

> Share your **unlimited mobile data** with other devices — without using your carrier's mobile-hotspot quota.

DataNet Share turns an Android phone with unlimited cellular data into a SOCKS5 proxy gateway. Other phones, laptops, or tablets can connect over **Wi‑Fi Direct**, an existing **Wi‑Fi network**, or **USB tethering**, and route their traffic through the host phone's cellular connection. This sidesteps carrier-side hotspot caps entirely, because the host phone simply appears to be using its own cellular data — which it is.

- **App version:** 3.5 (versionCode 22)
- **Package:** `com.datanet.share`
- **Min Android:** 8.0 (API 26)
- **Target Android:** 14 (API 34)
- **Language:** Kotlin 1.9 / JVM 17
- **Build:** Gradle 8.2 (Kotlin DSL)

---

## Table of contents

1. [Why this app exists](#why-this-app-exists)
2. [How it works](#how-it-works)
3. [Features](#features)
4. [Download & install](#download--install)
5. [Quick start](#quick-start)
6. [Usage modes](#usage-modes)
   - [Mode 1 — Wi‑Fi Direct (phone → phone)](#mode-1--wi‑fi-direct-phone--phone)
   - [Mode 2 — Local-only (phone → PC/Mac via USB or Wi‑Fi)](#mode-2--local-only-phone--pcmac-via-usb-or-wi‑fi)
   - [Mode 3 — Receiver (receive shared data on another phone)](#mode-3--receiver-receive-shared-data-on-another-phone)
7. [Configuring a SOCKS5 proxy on a PC/Mac](#configuring-a-socks5-proxy-on-a-pcmac)
8. [Troubleshooting](#troubleshooting)
9. [Permissions explained](#permissions-explained)
10. [Project structure](#project-structure)
11. [Build from source](#build-from-source)
12. [Technical details](#technical-details)
13. [Safety, privacy & fair-use](#safety-privacy--fair-use)
14. [License](#license)

---

## Why this app exists

Many mobile plans bundle "unlimited cellular data" but tightly cap or charge extra for the carrier's official hotspot/tethering feature. If you have such a plan and you want to:

- Get a laptop online while travelling, using your phone's data
- Let a friend's phone borrow connectivity when their own plan runs out
- Use a tablet that has no SIM of its own

…then DataNet Share is for you. The host phone serves its cellular connection to other devices through a SOCKS5 proxy running entirely on-device. No traffic ever leaves through the carrier's tethering APN, so the carrier has nothing to count against a hotspot quota — it just sees your phone using cellular data, which is exactly what you're paying for.

This is functionally similar to running `ssh -D` or a SOCKS5 server on a rooted phone, except it works **without root**, uses the official `VpnService` API on the receiver side, and exposes a friendly UI on both ends.

## How it works

```
            ┌────────────────────────── SENDER (host phone) ──────────────────────────┐
            │                                                                          │
 Cellular ─▶ CellularNetworkTracker ── binds every upstream socket to the cellular   │
   modem      (forces traffic out of    Network, so it never leaks back over Wi‑Fi)   │
                                      │                                               │
                                      ▼                                               │
                              Socks5Server (0.0.0.0:8282)                             │
                                      ▲                                               │
              ┌───────────────────────┼───────────────────────┐                       │
              │                       │                       │                       │
        Wi‑Fi Direct            USB / Wi‑Fi              (local)                      │
        p2p interface           rndis / wlan0             loopback                    │
              │                       │                                                   │
              └───────────┬───────────┘                                                   │
                          │                                                               │
                          ▼                                                               │
                   ┌──────────────────┐                                                  │
                   │  Receiver device │                                                  │
                   │  (phone or PC)   │                                                  │
                   └──────────────────┘                                                  │
            └──────────────────────────────────────────────────────────────────────────┘

   Receiver phone:
     ReceiverVpnService (Android VpnService) ─▶ Tun2Socks ─▶ SOCKS5 over Wi‑Fi Direct
     All apps transparently routed, no per-app config required.

   Receiver PC/Mac:
     Configure OS / browser / app to use SOCKS5 proxy at <host-ip>:8282.
```

In short:

- **Sender** opens a SOCKS5 listener on port `8282`. Every outbound socket it creates on behalf of clients is bound to the **cellular `Network` object** captured by `CellularNetworkTracker`, which is what keeps traffic flowing through the mobile modem even when Wi‑Fi is also active.
- **Wi‑Fi Direct path** — the sender creates a Wi‑Fi Direct group (it becomes group owner at `192.168.49.1`). The receiver joins the SSID shown on screen.
- **Local-only path** — the sender skips Wi‑Fi Direct entirely and binds the SOCKS5 server to `0.0.0.0`, so anything the phone is already reachable from (USB tethering, same Wi‑Fi LAN, etc.) can connect. This deliberately avoids the `TetheringManager` conflict that prevents USB tethering while Wi‑Fi Direct is active.
- **Receiver phone** uses Android's `VpnService` to capture all device traffic into a TUN interface, then `Tun2Socks` translates TUN packets to SOCKS5 sessions and ships them to the sender. The `protect()` calls on every socket prevent the receiver's own outgoing traffic from being recursively captured by the VPN.
- **Receiver PC/Mac** has no TUN available without admin/root, so it just points its OS or app proxy settings at the sender.

## Features

- **No root required.** Uses only public Android APIs (`VpnService`, `WifiP2pManager`, `ConnectivityManager`).
- **Three transport modes:**
  1. Wi‑Fi Direct — true phone-to-phone, no existing network needed.
  2. Local-only — share over USB tethering or any Wi‑Fi the host is already on.
  3. Receiver — transparent on-device VPN for the receiving phone.
- **Cellular-bound sockets.** Every upstream socket is created from the cellular `Network`'s `socketFactory`, so traffic can't silently fall back to Wi‑Fi.
- **SOCKS5 with TCP + UDP ASSOCIATE.** Full SOCKS5, including UDP support so DNS, STUN, QUIC, and online games work, not just TCP web traffic.
- **tun2socks on the receiver.** A hand-written IP/TCP/UDP user-space stack that reads raw packets from the TUN fd and bridges them to SOCKS5 sessions — no native code, no external tun2socks binary.
- **Wi‑Fi Direct auto-retry with backoff.** Up to 4 attempts with 1 s → 2 s → 4 s → 8 s backoff, plus targeted hints when the failure looks like Nearby Share / Quick Share holding Wi‑Fi Direct.
- **Foreground services + wake locks.** Sharing survives screen-off and Doze. A persistent notification shows live stats (active clients, RX/TX bytes, cellular state).
- **Live network interface inventory.** A "Refresh interfaces" button lists every up interface (Wi‑Fi Direct, USB tether, Wi‑Fi, cellular) with its IPv4 address, so you always know which IP to point a PC/Mac proxy at.
- **UDP test button.** Runs a quick STUN probe against Google's public STUN servers and reports whether UDP egress is working through the tunnel.
- **Dark, single-screen UI** with a scrolling monospace log, status indicator, and a credentials card that surfaces the SSID/password (Wi‑Fi Direct mode) or the interface list (local-only mode).
- **Self-contained.** No analytics, no telemetry, no third-party networking libraries. All networking is plain `java.net` / `android.net`.

## Download & install

### Option A — Pre-built APK

A signed release APK ships in [`releases/DataNetShare-v3.5-release.apk`](releases/DataNetShare-v3.5-release.apk) in this repository.

1. Transfer the APK to your Android phone.
2. Open it from your file manager. If prompted, allow "Install unknown apps" for your file manager.
3. Tap **Install**.

> The APK is signed with the project's release key. If you rebuild from source with a different key (see [Build from source](#build-from-source)), you must uninstall the pre-built version first — Android refuses to install an app whose signing key has changed.

### Option B — Build from source

See [Build from source](#build-from-source) below.

### Minimum requirements

- Two Android devices for phone-to-phone sharing, **or** one Android phone + one PC/Mac for phone-to-computer sharing.
- Host phone must have an active cellular data connection (the one with "unlimited" data).
- Wi‑Fi must be enabled on the host phone for Wi‑Fi Direct mode (it doesn't need to be connected to anything).
- Android 8.0 (API 26) or newer on both phones.

## Quick start

The fastest end-to-end flow, phone-to-phone over Wi‑Fi Direct:

1. Install the app on **both** phones.
2. On the phone with cellular data: open the app → tap **START SHARING (Wi‑Fi Direct)**.
3. Wait for the credentials card to appear (SSID like `DIRECT-xx-…` and a passphrase).
4. On the second phone: open Android Settings → Wi‑Fi → join that SSID with that passphrase.
5. On the second phone: open DataNet Share → tap **START RECEIVING (receiver)**. Approve the VPN consent dialog.
6. The receiver's status should flip to **Active**. All apps on the receiver now use the host phone's cellular data.

For phone-to-PC, jump to [Mode 2 — Local-only](#mode-2--local-only-phone--pcmac-via-usb-or-wi‑fi).

## Usage modes

The app has three top-level buttons. Each one starts a different foreground service.

### Mode 1 — Wi‑Fi Direct (phone → phone)

Best when neither phone is on a shared Wi‑Fi network and you want a self-contained wireless link.

**On the sender:**
1. Make sure **Wi‑Fi is turned on** in Android Settings (it doesn't need to be connected to an SSID).
2. Open DataNet Share → tap **START SHARING (Wi‑Fi Direct)**.
3. The app cleans up any stale Wi‑Fi Direct state, then creates a new group. This can take 3–10 seconds. If it fails, it retries up to 4 times with backoff.
4. When the group is ready, the credentials card shows:
   - **Network name (SSID):** e.g. `DIRECT-ab-DataNet`
   - **Password:** an 8-character WPA2 passphrase
   - **Available interfaces:** should include a `p2p` interface with IP `192.168.49.1`
5. The sender is now hosting a SOCKS5 proxy at `192.168.49.1:8282`, reachable from anything that joins that Wi‑Fi Direct SSID.

**On the receiver phone:**
1. Open Android Settings → Wi‑Fi → join the SSID shown on the sender, using the passphrase shown.
2. Open DataNet Share → tap **START RECEIVING (receiver)**.
3. Approve the **"Connection request — DataNet Share wants to set up a VPN connection"** system dialog. (Android requires this every time a VPN starts; it cannot be pre-approved.)
4. Status should flip to **Active — Tunneling via 192.168.49.1:8282** within a second or two.
5. Open any app or browser on the receiver — traffic is now flowing through the sender's cellular data.

**To stop:** tap **STOP** on either phone. Stopping the sender also drops the receiver; the receiver will detect the broken tunnel and stop itself.

### Mode 2 — Local-only (phone → PC/Mac via USB or Wi‑Fi)

Best when you want to share with a laptop, or when Wi‑Fi Direct keeps failing on your device. Local-only mode skips Wi‑Fi Direct entirely, which also means it doesn't conflict with USB tethering — you can have both at once.

**On the sender:**
1. (Optional but recommended) Enable **USB tethering** in Android Settings → Network → Hotspot & tethering, after plugging the phone into your PC/Mac. The phone will appear as a wired network interface (`rndis0` on Linux, `en5` / `RNDIS` on macOS, an Ethernet adapter on Windows).
2. Open DataNet Share → tap **START LOCAL ONLY (USB/WiFi)**.
3. The credentials card switches to **(local mode)** and shows the **Available interfaces** list with each interface's IPv4 address. Look for one of:
   - `USB Tethering (rndis0): 192.168.42.129` — use this if you connected via USB
   - `Wi‑Fi (wlan0): 192.168.1.34` — use this if your PC/Mac is on the same Wi‑Fi as the phone
4. The SOCKS5 server is now listening on `0.0.0.0:8282` — reachable on **any** of the listed IPs.

**On the PC/Mac:** configure your system or browser to use SOCKS5 at one of the IPs from step 3, port `8282`. See [Configuring a SOCKS5 proxy on a PC/Mac](#configuring-a-socks5-proxy-on-a-pcmac).

**Why local-only is sometimes better than Wi‑Fi Direct:**
- Wi‑Fi Direct and the system `TetheringManager` can't always coexist — turning on USB tethering sometimes silently tears down an active Wi‑Fi Direct group. Local-only sidesteps this entirely.
- Wi‑Fi Direct group creation is flaky on some OEM ROMs (Xiaomi, MIUI, and some Samsung One UI versions are known to interfere). Local-only is a reliable fallback.
- Throughput over USB tethering is typically 2–3× faster than Wi‑Fi Direct.

### Mode 3 — Receiver (receive shared data on another phone)

The receiver button is only useful when the receiving device is another Android phone that has joined the sender's Wi‑Fi Direct group (or is otherwise reachable back to the sender's IP `192.168.49.1`). See [Mode 1](#mode-1--wi‑fi-direct-phone--phone) for the full flow.

If you want to receive from a sender in **local-only** mode over a different subnet, you'll need to modify the hardcoded `192.168.49.1` in `MainActivity.startReceiverService()` to the sender's actual IP. The receiver UI doesn't currently expose a host field — this is on the roadmap.

## Configuring a SOCKS5 proxy on a PC/Mac

When you're using [Mode 2 — Local-only](#mode-2--local-only-phone--pcmac-via-usb-or-wi‑fi), the PC/Mac needs to be told to use the phone as a SOCKS5 proxy.

### macOS — system-wide

1. **System Settings → Network → Wi‑Fi (or Ethernet) → Details → Proxies.**
2. Turn on **SOCKS Proxy**.
3. Host: the phone's IP from the interfaces list (e.g. `192.168.42.129` for USB, `192.168.1.34` for Wi‑Fi).
4. Port: `8282`. Leave authentication off.
5. Apply → restart your browser.

### Windows 10/11 — system-wide

1. **Settings → Network & Internet → Proxy.**
2. Under **Manual proxy setup**, turn on **Use a proxy server**.
3. Address: phone's IP. Port: `8282`.
4. (Windows' system proxy only supports HTTP/HTTPS, not SOCKS5, so this won't work for all apps. For SOCKS5 specifically, configure it per-app — see below.)

### Windows — per-app (recommended)

Use a SOCKS5-capable proxy client such as:

- **Firefox:** Settings → Network Settings → Manual proxy configuration → SOCKS Host = phone IP, Port = `8282`, SOCKS v5, ✓ "Proxy DNS when using SOCKS v5".
- **Proxifier** (paid) or **proxychains-windows** (free): force any app's traffic through the SOCKS5 proxy.

### Linux — per-shell

```bash
# Most GTK/GNOME apps and curl respect these:
export ALL_PROXY=socks5h://192.168.42.129:8282
export HTTP_PROXY=socks5h://192.168.42.129:8282
export HTTPS_PROXY=socks5h://192.168.42.129:8282

# Or for a single command:
proxychains4 curl https://example.com
```

The `socks5h://` scheme (with the trailing `h`) is important — it tells the client to do DNS resolution through the proxy, not locally. Otherwise DNS leaks and many sites will fail to resolve.

### Browser quick-check

After configuring, open `https://ifconfig.me` or `https://ip.sb` in your browser. The IP returned should be the **sender phone's cellular IP**, not your usual home/work IP. If you see your usual IP, the proxy isn't being used.

## Troubleshooting

### "createGroup failed after 4 attempts: BUSY"

Another app is holding Wi‑Fi Direct. The usual culprits, in order:

1. **Quick Share / Nearby Share** (Google Play Services) — disable it in Settings → Google → Devices & sharing → Nearby Share → turn off.
2. **Wi‑Fi Direct group left over from another app** — toggle Wi‑Fi off and back on, then retry.
3. **MIUI / HyperOS "Wi‑Fi scan" throttling** — reboot the phone.

The app's log panel will show exactly which attempt number failed and the reason code.

### "WiFi is turned off"

Wi‑Fi Direct requires Wi‑Fi to be enabled at the system level, even though it doesn't use a regular SSID. Open Android Settings → Wi‑Fi → toggle on. You do **not** need to be connected to an access point.

### Receiver VPN consent dialog keeps appearing

Android requires user consent every time a VPN service starts. There is no way for an app to bypass this — it's a deliberate security boundary. The app will refuse to start the tunnel until you tap OK.

### Receiver says "Active" but no traffic flows

1. On the receiver, tap **TEST UDP (run on receiver)**. If it reports `UDP WORKING`, the tunnel is fine and your issue is app-specific.
2. If UDP fails, the sender may have lost cellular. Check the sender's notification — it shows `NO CELLULAR` in red text when the cellular `Network` is unavailable.
3. Confirm the receiver is still joined to the sender's Wi‑Fi Direct SSID. The Wi‑Fi Direct group can drop silently if either phone sleeps aggressively.

### Sender says "NO CELLULAR - traffic will fall back to WiFi"

This means `CellularNetworkTracker` couldn't find a `TRANSPORT_CELLULAR` network with `NET_CAPABILITY_INTERNET`. Causes:

- Cellular data is turned off in Android Settings → Mobile network.
- Airplane mode is on.
- The SIM is in a state where it has signal but no data registration (common right after a SIM swap).

Fix the cellular state and the tracker will pick it up within a second or two — it's listening to `ConnectivityManager` callbacks, no app restart needed.

### PC/Mac proxy works for HTTP but not HTTPS

You're probably using `socks5://` instead of `socks5h://` (or "Don't proxy DNS" is checked in Firefox). DNS is leaking locally and your DNS server is rejecting the request. Switch to `socks5h://` and re-check "Proxy DNS when using SOCKS v5".

### Throughput is slow

- Wi‑Fi Direct is inherently limited (typically 20–40 Mbps real-world). Switch to **local-only + USB tethering** for 100+ Mbps.
- Both phones should be unplugged from chargers if you're seeing thermal throttling — sustained Wi‑Fi Direct traffic makes the radio run hot.
- Disable battery optimisation for DataNet Share on both phones: Settings → Apps → DataNet Share → Battery → Unrestricted.

### The app dies overnight

Android's App Standby Buckets will eventually kill even foreground services if the phone is left alone for many hours. The app holds a `PARTIAL_WAKE_LOCK` to mitigate this, but on aggressive OEM ROMs (OPPO, Vivo, some Samsung low-power modes) you may need to:

- Pin the app in recents.
- Add it to the OEM's "don't optimise" whitelist (this is a separate list from Android's standard battery optimisation).
- Use the **STOP** button before bed and restart in the morning — a clean shutdown is better than letting Android kill it mid-tunnel, which can leave Wi‑Fi Direct in a weird state.

## Permissions explained

DataNet Share uses the minimum set of permissions needed to function. Here's why each one is required.

| Permission | Why it's needed |
|---|---|
| `INTERNET` | Open the SOCKS5 listener and outbound cellular sockets. |
| `ACCESS_NETWORK_STATE` | Read the active `Network` and its `NetworkCapabilities` to find the cellular network. |
| `ACCESS_WIFI_STATE` | Check whether Wi‑Fi is enabled before starting Wi‑Fi Direct. |
| `CHANGE_WIFI_STATE` | Required by `WifiP2pManager.createGroup()` on some OEM ROMs. |
| `CHANGE_NETWORK_STATE` | Required by `WifiP2pManager` initialisation on some OEM ROMs. |
| `ACCESS_FINE_LOCATION` | Android 12+ requires location permission to scan for / create Wi‑Fi Direct groups (Wi‑Fi scan results can leak location). |
| `NEARBY_WIFI_DEVICES` | Android 13+ replacement for the location requirement above, scoped to Wi‑Fi Direct only — declared with `neverForLocation` so it can't be used for location tracking. |
| `FOREGROUND_SERVICE` | Run `SenderService` and `ReceiverService` as foreground services so they survive screen-off. |
| `FOREGROUND_SERVICE_CONNECTED_DEVICE` | Android 14+ requires a typed foreground-service permission; this is the type declared in the manifest for both services. |
| `FOREGROUND_SERVICE_DATA_SYNC` | Used as a fallback type on some OEM ROMs where `CONNECTED_DEVICE` is rejected for SOCKS5-style traffic. |
| `POST_NOTIFICATIONS` | Android 13+ requires runtime permission to post the foreground-service notification. |
| `WAKE_LOCK` | `PARTIAL_WAKE_LOCK` on both sender and receiver prevents the CPU sleeping during sustained transfers. Held for at most 10 hours per session. |
| `RECEIVE_BOOT_COMPLETED` | Reserved for future auto-start-on-boot support. Not currently used. |
| `BIND_VPN_SERVICE` (on `ReceiverVpnService`) | System-only permission — only Android itself can bind to the VPN service. Required for any `VpnService` to function. |

The app makes **no outbound network requests of its own** — no telemetry, no analytics, no update checks. The only network traffic it generates is your own data, proxied through the cellular modem.

## Project structure

```
datanet-share-github/
├── app/
│   ├── build.gradle.kts                       # App module config (version, deps, signing)
│   ├── proguard-rules.pro
│   └── src/main/
│       ├── AndroidManifest.xml                # Permissions, services, VPN intent filter
│       ├── res/
│       │   ├── layout/activity_main.xml       # Single-screen UI
│       │   ├── values/{strings,themes}.xml
│       │   └── drawable/, mipmap-anydpi-v26/  # Launcher icon (adaptive)
│       └── java/com/datanet/share/
│           ├── MainActivity.kt                # UI, log panel, permission handling
│           ├── common/
│           │   ├── Constants.kt               # Port 8282, VPN addr, intent extras
│           │   ├── CellularNetworkTracker.kt  # Bind upstream sockets to cellular Network
│           │   ├── InterfaceEnumerator.kt     # List IPs for PC/Mac proxy setup
│           │   ├── NetworkHelper.kt           # One-shot cellular detection
│           │   └── UdpTest.kt                 # STUN-based UDP reachability test
│           ├── sender/
│           │   ├── SenderService.kt           # Foreground service hosting SOCKS5
│           │   ├── Socks5Server.kt            # SOCKS5 (TCP CONNECT + UDP ASSOCIATE)
│           │   └── WifiDirectHost.kt          # Wi-Fi Direct group lifecycle + retries
│           └── receiver/
│               ├── ReceiverService.kt         # VPN permission bootstrap
│               ├── ReceiverVpnService.kt      # VpnService: TUN setup, protect(), stats
│               ├── Tun2Socks.kt               # TUN → SOCKS5 bridge (TCP + UDP)
│               ├── TcpFlow.kt                 # Per-connection TCP state machine
│               └── SharedUdpFlow.kt           # UDP ASSOCIATE session multiplexer
├── build.gradle.kts                           # Root Gradle config
├── settings.gradle.kts                        # Project name: DataNetShare
├── gradle.properties
├── gradlew / gradlew.bat
├── gradle/wrapper/                            # Gradle 8.2 wrapper
└── releases/
    └── DataNetShare-v3.5-release.apk          # Signed release APK
```

## Build from source

### Prerequisites

- **JDK 17** (required by AGP 8.2)
- **Android SDK** with platform `android-34` and build-tools `34.x`
- **Android keystore** for a release build (or just use `debug` for personal use)

### Steps

```bash
git clone <this-repo>
cd datanet-share-github

# Debug build (no signing config needed)
./gradlew assembleDebug
# Output: app/build/outputs/apk/debug/app-debug.apk

# Release build — needs a keystore. The repo's build.gradle.kts references
# /home/z/my-project/keystore/datanet-release.jks, which won't exist on your
# machine. Either:
#   (a) edit app/build.gradle.kts to point at your own keystore, or
#   (b) comment out the signingConfig line in the release build type.
./gradlew assembleRelease
# Output: app/build/outputs/apk/release/app-release.apk
```

Install on a connected device:

```bash
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

> ⚠️ If you rebuild with a different signing key than the one used for `releases/DataNetShare-v3.5-release.apk`, you must uninstall the pre-built version first: `adb uninstall com.datanet.share`. Android refuses to install an app whose signing key has changed.

### Cleaning Wi‑Fi Direct state during development

When the app crashes mid-session, it can leave a stale Wi‑Fi Direct group that prevents the next run from creating a new one. The app cleans up on startup, but if you're hacking on `WifiDirectHost.kt` and need to manually reset:

```bash
adb shell svc wifi disable && adb shell svc wifi enable
# Or fully:
adb shell settings put global wifi_p2p_pending_state 0
```

## Technical details

### How traffic is forced through cellular

`CellularNetworkTracker` registers a `NetworkCallback` filtered to `TRANSPORT_CELLULAR` + `NET_CAPABILITY_INTERNET`. When such a network becomes available, it caches the `Network` object. `Socks5Server` then calls `cellularNetwork.socketFactory.createSocket()` instead of `new Socket()`, which binds the socket to the cellular interface at the kernel level.

Without this, Android's default route would prefer Wi‑Fi when both are up — which would defeat the entire purpose of the app, since the host phone is usually also connected to the Wi‑Fi Direct group it just created (or, in local-only mode, to the same Wi‑Fi as the receiver).

### How tun2socks works without native code

`Tun2Socks` reads raw IP packets from the TUN file descriptor (`FileInputStream(tunFd)`), parses IP + TCP/UDP headers, and for each TCP 5-tuple spawns a `TcpFlow` that:

1. Opens a SOCKS5 connection to the sender.
2. Sends a `CONNECT` request with the destination IP:port from the packet.
3. Siphons data bidirectionally between the TCP socket and the TUN.

UDP is trickier — it uses a single SOCKS5 `UDP ASSOCIATE` (handled by `SharedUdpFlow`), with the receiver maintaining a NAT table mapping `(src ip:port)` → `(sender-side UDP relay)`. This lets DNS, STUN, QUIC, and games work transparently.

Every outbound socket is wrapped in `VpnService.protect()`, which marks it as "should not be routed back through this VPN". Without that call, the receiver's own SOCKS5 traffic would be captured by the VPN again, creating an infinite loop and crashing the kernel within seconds.

### Wi‑Fi Direct group owner IP

Wi‑Fi Direct group owners are always assigned `192.168.49.1/24` by Android, with clients receiving `192.168.49.x` via DHCP. This is hardcoded in the receiver's `MainActivity` (it sends `EXTRA_SOCKS_HOST = "192.168.49.1"` when starting the receiver service). If you want to receive from a sender in local-only mode on a different subnet, you'll need to change that constant or expose a host field in the UI.

### Foreground service types

`SenderService` and `ReceiverService` both declare `foregroundServiceType="connectedDevice"`. On Android 14+, foreground services must declare a type that matches the work they do, and `connectedDevice` is the correct type for services that manage Wi‑Fi Direct connections. The manifest also declares `FOREGROUND_SERVICE_DATA_SYNC` as a fallback permission for OEM ROMs that interpret the type system slightly differently.

### Wake lock lifetime

Both services hold a `PARTIAL_WAKE_LOCK` with a 10-hour timeout (`acquire(10 * 60 * 60 * 1000L)`). This is intentionally longer than any reasonable sharing session, but not unbounded — if you forget the app is running, the lock will eventually expire and Android can put the phone to sleep normally. The lock is released cleanly on `stopSelf()` / `onDestroy()`.

## Safety, privacy & fair-use

- **No data leaves your devices.** DataNet Share does not phone home. There is no analytics SDK, no crash reporter, no update checker. The only network traffic it generates is yours, proxied through your cellular modem.
- **All traffic is end-to-end between your devices.** The SOCKS5 proxy is unauthenticated (no username/password) but is only reachable over Wi‑Fi Direct (a single-hop, WPA2-encrypted link) or your own USB/Wi‑Fi connection. Do **not** expose the SOCKS5 port to a public network — anyone on the same network could use your cellular data.
- **Carrier fair-use.** DataNet Share does not bypass any carrier restriction on **what** data you can access — it only changes **how** that data is delivered (your phone vs. a tethered device). If your plan explicitly prohibits tethering or any form of data sharing in its terms of service, using this app may violate those terms. The app cannot and does not hide what you're doing from the carrier — the destination IPs and traffic patterns are identical to using the phone directly. Use responsibly.
- **Battery impact.** Sustained sharing will warm up the host phone and drain the battery noticeably faster. The app holds a wake lock while active. Use the **STOP** button when you're done — don't just background the app.

## License

This project is released under the **MIT License**. See the source files for the copyright notice. You are free to use, modify, and distribute the app, including commercially. Attribution is appreciated but not required.

---

**DataNet Share** — because unlimited data should mean *unlimited data*, not "unlimited on this one screen".
