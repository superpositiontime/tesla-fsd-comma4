# 🚗 Tesla FSD CAN Mod — Comma 4 Edition

[![Python](https://img.shields.io/badge/python-3.10%2B-blue)](https://python.org)
[![Target](https://img.shields.io/badge/target-2026%20Model%20Y%20Juniper-red)](https://www.tesla.com)
[![Hardware](https://img.shields.io/badge/hardware-comma%204-black)](https://comma.ai)
[![HW Version](https://img.shields.io/badge/autopilot-HW4-orange)]()
[![License](https://img.shields.io/badge/license-MIT-green)]()

<p align="center">
  <img src="https://www.comma.ai/_app/immutable/assets/four_screen_on.WrrTderw.png" alt="comma four" width="420" />
</p>

A Python port of [Starmixcraft's Tesla FSD CAN Mod](https://gitlab.com/Starmixcraft/tesla-fsd-can-mod) — adapted to run directly on the **comma 4** via its built-in panda CAN interface.

No extra hardware needed. If your comma 4 is already working in your Tesla, this is purely a software mod.

---

## 💰 Subscriptions & Cost Breakdown

> **You need an active Tesla FSD subscription.** The CAN mod handles activation and nag suppression, but the FSD neural net software must be installed on the car first — which requires a subscription.

| Item | Cost | Required? | Notes |
|------|------|-----------|-------|
| **Tesla FSD Supervised** | $99/mo | ✅ **YES** | The FSD neural net software must be downloaded to your car. The CAN mod activates it and suppresses nags, but can't run FSD without the software present. |
| **Tesla Premium Connectivity** | $9.99/mo | ❌ **NO** | Only needed for Tesla streaming/satellite view — not related to driving |
| **Comma Prime** | $24/mo | ❌ **Optional** | Gives you cloud dashcam, remote SSH, and analytics — nice to have but not required for this mod |
| **comma 4 device** | ~$999 | ✅ One-time | The hardware that runs openpilot and hosts the CAN mod |
| **Tesla-compatible harness** | ~$50–100 | ✅ One-time | OBD-C harness specific to your Tesla model (from comma shop) |

**Total cost: ~$999–$1,100 one-time + $99/month for Tesla FSD**

---

## 🎯 Prerequisites

Before you start, make sure you have:

- [ ] **Tesla FSD Supervised subscription** — active $99/mo subscription so the FSD software is installed on your car
- [ ] **comma 4** — purchased from [comma.ai/shop](https://comma.ai/shop)
- [ ] **Tesla-compatible harness** — the OBD-C cable for your specific Tesla model from the comma shop
- [ ] **2026 Tesla Model Y Juniper** with **HW4 or HW4.5** (firmware ≥ 2026.2.3)
- [ ] **openpilot already installed and working** — do NOT attempt the FSD mod until basic comma driving works
- [ ] **SSH access** to your comma device (via comma connect app or local WiFi)
- [ ] **Phone on the same WiFi** as the comma (for the toggle web UI)

---

## 📋 Step-by-Step Setup

### Step 1: Subscribe to Tesla FSD

1. Open the Tesla app on your phone
2. Tap **Upgrades → Software Upgrades → FSD (Supervised)**
3. Subscribe for $99/month
4. Wait for the car to download the FSD software update (requires WiFi, may take 30+ minutes)
5. Confirm FSD works natively first — test it on a drive before proceeding

> ⚠️ The CAN mod cannot work without the FSD neural net installed on the car. The subscription ensures the software is present.

### Step 2: Install the comma 4 hardware

1. Mount the comma 4 on your windshield using the included mount
2. Connect the OBD-C harness between the comma 4 and your Tesla's ADAS camera connector (behind the rearview mirror)
3. Power on the comma — it should boot to the openpilot setup screen
4. Follow comma's on-screen setup (WiFi, software update, etc.)

> 📖 Full hardware guide: [comma.ai/setup](https://comma.ai/setup)

### Step 3: Verify openpilot works first

**Do NOT skip this step.** Before touching the FSD mod:

1. Drive with openpilot in your Tesla and confirm it steers, accelerates, and brakes normally
2. Make sure the comma 4 screen shows the driving view with lane lines
3. Test for at least a few drives to confirm stability

If openpilot doesn't work, the FSD mod won't either — they share the same panda CAN interface.

### Step 4: SSH into the comma

From your computer (same WiFi as the comma):

```bash
ssh comma@comma.local
# Default has no password — uses key-based auth via comma connect
```

Or use the IP address shown in the comma's settings menu:

```bash
ssh comma@192.168.1.XX
```

### Step 5: Download the scripts

```bash
# FSD CAN mod script
curl -o /data/tesla_fsd_comma4.py \
  https://raw.githubusercontent.com/superpositiontime/tesla-fsd-comma4/main/tesla_fsd_comma4.py

# Mode toggle web server
curl -o /data/fsd_toggle_server.py \
  https://raw.githubusercontent.com/superpositiontime/tesla-fsd-comma4/main/fsd_toggle_server.py
```

### Step 6: Test in dummy mode (no car needed)

Test the script logic at home without being in the car:

```bash
nano /data/tesla_fsd_comma4.py
```

Set at the top of the file:
```python
DUMMY_MODE = True    # generates fake CAN frames for testing
TRANSMIT   = False   # don't try to send anything
```

Run it:
```bash
python3 /data/tesla_fsd_comma4.py
```

You should see simulated CAN frame processing in the terminal. If it runs without errors, you're good.

### Step 7: Start the toggle server

```bash
python3 /data/fsd_toggle_server.py &
```

This starts the web UI on port 8088.

### Step 8: Toggle between modes from your phone

1. Open your phone browser
2. Go to `http://<comma-ip>:8088` (e.g., `http://192.168.1.42:8088`)
3. You'll see the current mode and a big toggle button
4. Tap to switch:
   - **→ Tesla FSD:** Stops openpilot → starts the CAN mod → Tesla's FSD takes over
   - **→ openpilot:** Stops the CAN mod → restarts openpilot → Comma drives
5. Wait ~5–10 seconds for the transition to complete

### Step 9: Auto-start on boot (optional)

To have the toggle server start automatically when the comma boots:

```bash
echo 'python3 /data/fsd_toggle_server.py &' >> /data/rc.local
chmod +x /data/rc.local
```

Now every time the comma powers on, the toggle UI will be available at port 8088.

---

## 🧠 How It Works — Architecture

> **Only one system drives at a time.** This mod switches between two completely separate driving brains.

### Mode A: openpilot (Comma drives the car)

```
┌─────────────────────────────────────────────────────┐
│  COMMA 4 DEVICE                                     │
│  ┌───────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ Cameras   │→ │ AI Model │→ │ Steering / Accel │  │
│  │ (onboard) │  │(openpilot│  │  via Panda CAN   │  │
│  └───────────┘  └──────────┘  └──────────────────┘  │
│                                                     │
│  Tesla FSD: OFF (not active)                        │
│  FSD CAN script: NOT RUNNING                        │
└─────────────────────────────────────────────────────┘
```

- The Comma 4 uses its own cameras + AI model to drive
- openpilot controls steering, gas, and braking through the panda CAN interface
- Tesla's FSD is not active (but software remains installed)
- The Comma's 1.9" OLED shows the openpilot driving view

### Mode B: Tesla FSD (Tesla drives the car)

```
┌─────────────────────────────────────────────────────┐
│  TESLA HW4 COMPUTER (built into the car)            │
│  ┌───────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ Tesla     │→ │ Tesla NN │→ │ Steering / Accel │  │
│  │ Cameras   │  │ (FSD)    │  │  (native)        │  │
│  └───────────┘  └──────────┘  └──────────────────┘  │
│                                                     │
│  COMMA 4: openpilot STOPPED                         │
│  Comma's role: CAN tool only                        │
│  ┌──────────────────────────────────────────────┐   │
│  │ tesla_fsd_comma4.py running via panda:       │   │
│  │  • Flip FSD-enabled bit (0x3FD mux 0)        │   │
│  │  • Suppress nag warnings (0x3FD mux 1)       │   │
│  │  • Inject speed profile (0x3FD mux 2)        │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

- openpilot is **stopped** — the Comma does **not** drive
- Tesla's own HW4 computer + cameras + neural net handles everything
- The Comma 4's panda is used **only** as a CAN bus tool to:
  - Activate FSD (bit injection on `0x3FD`) — the FSD software is already installed via subscription
  - Suppress "hands on wheel" nag warnings
  - Map follow distance to speed profiles (Chill / Normal / Sport)

### What the subscription provides vs what the mod does

| | FSD Subscription ($99/mo) | CAN Mod (this project) |
|---|---|---|
| **FSD neural net software** | ✅ Downloads to car | ❌ Cannot provide |
| **FSD activation signal** | Via Tesla servers | ✅ Via CAN bit injection |
| **Nag suppression** | ❌ | ✅ Clears nag messages |
| **Speed profile control** | ❌ | ✅ Maps follow distance |
| **openpilot integration** | ❌ | ✅ Toggle between systems |

### The Toggle (Phone Web UI)

```
        Phone browser → http://<comma-ip>:8088
                    ┌─────────────┐
     ┌──────────────│  SWITCH TO  │──────────────┐
     │              │    FSD      │              │
     ▼              └─────────────┘              ▼
 ┌────────┐     stops openpilot service     ┌────────┐
 │ COMMA  │ ──────────────────────────────→ │  FSD   │
 │  MODE  │     starts CAN mod script       │  MODE  │
 │        │ ←────────────────────────────── │        │
 └────────┘     stops CAN mod script        └────────┘
                starts openpilot service
```

The web toggle (`fsd_toggle_server.py`) serves a mobile-friendly page on port 8088. One tap switches modes. The transition takes ~5–10 seconds.

---

## ✨ Features

- **FSD activation** — modifies autopilot CAN frames to activate Full Self-Driving (subscription required for software)
- **Nag suppression** — clears the "hands on wheel" nag messages
- **Speed profile mapping** — reads your follow distance setting and maps it to a speed profile (Chill / Normal / Sport)
- **HUD status display** — shows live mod status on the comma screen via openpilot's cereal bus
- **Dummy / offline test mode** — generates synthetic CAN frames so you can test at home without the car
- **Monitor-only mode** — run alongside openpilot without interrupting its driving (`TRANSMIT = False`)
- **Phone toggle UI** — switch between FSD and openpilot from your phone browser

---

## ⚙️ Configuration

Edit the top of `tesla_fsd_comma4.py`:

```python
DUMMY_MODE      = False   # True = offline test with fake CAN frames
TRANSMIT        = True    # True = modify & retransmit frames (requires ALLOUTPUT)
SHOW_ON_SCREEN  = True    # True = publish status to openpilot HUD via cereal
CAN_BUS         = 2       # Autopilot bus (try 0 or 1 if you see zero frames)
LOG_FRAMES      = True    # Print frame activity to terminal
```

---

## 📺 HUD Display

When `SHOW_ON_SCREEN = True`, live status is written to openpilot's params and the comma screen shows:

```
FSD Mod v2 | ACTIVE | Profile: Normal | Nag: OFF | Frames: 47 modified | Uptime: 83s
```

State changes also trigger a brief `userFlag` pulse in the openpilot HUD.

---

## 🔧 CAN Bus Details

This is a Python translation of the `HW4Handler` from the original CanFeather Arduino project. It:

1. **Listens** on the autopilot CAN bus (500 kbps) for two frame IDs:
   - `0x3F8` (1016) — Follow distance / speed profile
   - `0x3FD` (1021) — Autopilot command (3-frame mux)

2. **Modifies** the autopilot command frames:
   - **Mux 0** → sets bit 46 + bit 60 → signals FSD as active
   - **Mux 1** → clears bit 19 (nag) + sets bit 47 (suppress)
   - **Mux 2** → injects speed profile into byte 7 bits [6:4]

3. **Retransmits** the modified frames back onto the bus via panda's `SAFETY_ALLOUTPUT` mode

---

## ❓ FAQ

**Q: Do I need a Tesla FSD subscription?**
A: **Yes.** The $99/month FSD subscription is required. It ensures the FSD neural net software is downloaded and installed on your car. The CAN mod handles activation and nag suppression, but it can't run FSD without the software present. Think of it as: the subscription provides the brain, the mod flips the switch.

**Q: So what does the mod actually do if I'm paying for FSD anyway?**
A: The mod gives you the ability to **toggle between openpilot and FSD** on the fly. Without it, using a comma device means you're locked into openpilot only. The mod also suppresses nag warnings and gives you speed profile control via follow distance.

**Q: Does this work on HW3 vehicles?**
A: No. This port targets HW4/HW4.5 only (2024+ Model Y Juniper). The original [CanFeather project](https://gitlab.com/Starmixcraft/tesla-fsd-can-mod) has HW3 support if you need it.

**Q: Can Tesla patch this via OTA update?**
A: Potentially yes. Tesla could change the CAN frame IDs or add authentication. If that happens, the frame IDs in the script would need to be updated.

**Q: Will this void my Tesla warranty?**
A: Using a comma device involves modifying the ADAS camera connection and injecting CAN bus frames. This could void warranty on ADAS-related components. Use at your own risk.

**Q: Can I run FSD and openpilot at the same time?**
A: No. Only one system drives at a time. The toggle switches between them — it stops one before starting the other.

**Q: What if I don't have WiFi in my car?**
A: The comma 4 creates its own WiFi hotspot. Connect your phone to it to access the toggle UI. Alternatively, you can use comma connect's built-in SSH.

**Q: Is this legal?**
A: Modifying your own vehicle's CAN bus is a gray area. It's your car and your device, but Tesla's terms of service may prohibit it. This project is for educational purposes.

---

## ⚠️ Safety & Disclaimer

- **This is experimental software.** Use at your own risk.
- Setting `SAFETY_ALLOUTPUT` disables openpilot's safety checks — **openpilot will not be driving while this is active in TRANSMIT mode**.
- Tesla may change CAN message IDs at any time via OTA software updates.
- The original project targets HW4 (firmware ≥ 2026.2.3). Earlier HW3 vehicles use different frame IDs — see the [original project](https://gitlab.com/Starmixcraft/tesla-fsd-can-mod) for HW3 support.
- This mod has no affiliation with comma.ai or Tesla.

---

## 📄 Credits

- Original Arduino project: [Starmixcraft / tesla-fsd-can-mod](https://gitlab.com/Starmixcraft/tesla-fsd-can-mod)
- CAN bus research: [Michał Gapiński / Tesla Android](https://github.com/mikegapinski)
- Comma 4 / panda Python port: this repo

---

## 📝 License

MIT — do whatever you want, just don't blame anyone if something goes sideways.
