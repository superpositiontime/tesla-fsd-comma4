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

| Item | Cost | Required? | Notes |
|---|---|---|---|
| **Tesla FSD subscription** | $99/mo | ✅ **YES** | The car needs FSD software downloaded from Tesla's servers. The CAN mod activates it, but can't provide the neural net itself. |
| **Comma Prime** | $24/mo | ❌ Optional | Nice for remote SSH and cloud dashcam, but not needed for this mod |
| **Comma 4 hardware** | ~$999 | ✅ One-time | Includes panda CAN interface |
| **Tesla-compatible harness** | ~$50 | ✅ One-time | OBD-C cable + vehicle-specific connector |

### What does the subscription provide vs what does the mod do?

| | FSD Subscription ($99/mo) | This CAN Mod |
|---|---|---|
| FSD neural net software | ✅ Downloads to car via OTA | ❌ Cannot provide this |
| FSD activation signal | ✅ Via Tesla's servers | ✅ Via CAN bit injection |
| Nag suppression | ❌ | ✅ Clears "hands on wheel" warnings |
| Speed profile mapping | ❌ | ✅ Maps follow distance to speed |
| openpilot toggle | ❌ | ✅ Switch between FSD and openpilot |

**Bottom line:** You need an active FSD subscription so the car downloads the FSD software. The mod then activates it via CAN and adds features Tesla doesn't offer (nag suppression, openpilot switching).

---

## 🌍 FSD Outside Your Country

FSD (Supervised) is currently only available in: **US, Canada, China, Mexico, Puerto Rico, Australia, New Zealand, and South Korea.**

If you're in Europe or another unsupported region, here's how people are getting it to work:

### Method: Transfer Car to a US/Canada Tesla Account

This is the approach used by the [Michał Gapinski / TeslaAndroid](https://github.com/nicholasgapinski) project and others in Europe.

**Steps:**

1. **Create a new Tesla account** with a US or Canadian address
   - Go to [tesla.com](https://www.tesla.com) and register with a second email
   - Use a valid US/Canadian address (some people use a friend's or family member's address)

2. **Transfer your car to the new account**
   - In the Tesla app on your original account, go to **"Add/Remove Products"**
   - Remove the car from your current (non-US) account
   - On the new US account, add the car using the VIN
   - Tesla's official guide: [How to Update Your Account Country](https://www.tesla.com/support/how-to-update-your-tesla-account-country)

3. **Subscribe to FSD on the US account**
   - Open the Tesla app → FSD → Subscribe ($99/mo)
   - You'll need a US payment method (some international credit cards work, or use a US-based virtual card)

4. **Wait for FSD software to download**
   - The car needs to be connected to WiFi
   - The FSD neural net package will download via OTA update
   - This can take a few hours to a few days

5. **Use GPS/location emulation** (for activation in unsupported regions)
   - FSD checks GPS to verify you're in a supported country
   - The Gapinski project uses location emulation to make the car think it's in the US
   - The CAN mod's bit injection handles the FSD activation signal

### ⚠️ Important Caveats

- **Transferring your car to a US account may affect local warranty and service** — Tesla service centers in your country may not support a US-registered vehicle the same way
- **OTA updates will follow the US release schedule**, which is usually ahead of other regions
- **Navigation and maps** may behave differently under a US account
- **Tesla can change things at any time** — account transfer policies, FSD geo-restrictions, and CAN message formats are all subject to change via OTA
- **This is a grey area** — you're not breaking any laws, but you're using the system in ways Tesla didn't intend

### Europe FSD Timeline

As of March 2026, Tesla FSD is expected to receive EU regulatory approval via **UN WP.29** by mid-2026. Once approved, FSD subscriptions should become available natively in Europe without needing the account transfer workaround.

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
│  Tesla FSD: OFF (bypassed)                          │
│  FSD CAN script: NOT RUNNING                        │
└─────────────────────────────────────────────────────┘
```

- The Comma 4 uses its own cameras + AI model to drive
- openpilot controls steering, gas, and braking through the panda CAN interface
- Tesla's FSD computer is completely bypassed
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
  - Activate FSD (bit injection on `0x3FD`)
  - Suppress "hands on wheel" nag warnings
  - Map follow distance to speed profiles (Chill / Normal / Sport)

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

- **FSD enable injection** — modifies autopilot CAN frames to activate Full Self-Driving
- **Nag suppression** — clears the "hands on wheel" nag message
- **Speed profile mapping** — reads your follow distance setting and maps it to a speed profile (Chill / Normal / Sport)
- **HUD status display** — shows live mod status on the comma screen via openpilot's cereal bus
- **Dummy / offline test mode** — generates synthetic CAN frames so you can test at home without the car
- **Monitor-only mode** — run alongside openpilot without interrupting its driving (`TRANSMIT = False`)
- **Phone toggle UI** — switch between FSD and openpilot from your phone browser

---

## 🎯 Prerequisites

Before starting, make sure you have:

- [ ] **comma 4** device (~$999 from [comma.ai/shop](https://comma.ai/shop))
- [ ] **Tesla-compatible harness** + OBD-C cable (~$50 from comma shop)
- [ ] **2026 Model Y Juniper** (or other HW4/HW4.5 Tesla, firmware ≥ 2026.2.3)
- [ ] **Active FSD subscription** ($99/mo — see [Subscriptions](#-subscriptions--cost-breakdown) section)
- [ ] **openpilot already installed and working** on your comma 4
- [ ] **SSH access** to the comma (via [Comma Connect](https://connect.comma.ai) or local WiFi)
- [ ] **Phone on the same WiFi** as the comma (for the toggle UI)

---

## 📋 Step-by-Step Setup

### Step 1: Subscribe to Tesla FSD

Open the Tesla app → Upgrades → Full Self-Driving → Subscribe ($99/mo).

Wait for the FSD software to download to your car (connect to WiFi, may take a few hours). **Verify FSD works natively first** — engage it from the steering wheel and confirm it drives before proceeding.

> 🌍 **Not in a supported country?** See the [FSD Outside Your Country](#-fsd-outside-your-country) section above.

### Step 2: Install Comma 4 Hardware

Follow [comma.ai/setup](https://comma.ai/setup):
1. Plug the vehicle-specific harness into your car's ADAS camera connector
2. Connect the OBD-C cable to the comma 4
3. Mount the comma 4 on your windshield

### Step 3: Verify openpilot Works

Drive with openpilot first. Make sure it:
- Engages and steers on the highway
- Shows the driving view on the comma screen
- Has no errors or hardware issues

**Don't skip this.** If openpilot doesn't work, the FSD mod won't either.

### Step 4: SSH Into the Comma

```bash
# Option A: via local WiFi
ssh comma@<comma-ip>

# Option B: via Comma Connect (requires Comma Prime)
ssh comma@comma.local
```

Default password is usually blank or set during setup.

### Step 5: Download the Scripts

```bash
# FSD mod script
curl -o /data/tesla_fsd_comma4.py \
  https://raw.githubusercontent.com/superpositiontime/tesla-fsd-comma4/main/tesla_fsd_comma4.py

# Mode toggle server (web UI)
curl -o /data/fsd_toggle_server.py \
  https://raw.githubusercontent.com/superpositiontime/tesla-fsd-comma4/main/fsd_toggle_server.py
```

### Step 6: Test in Dummy Mode (No Car Needed)

Edit the script and set `DUMMY_MODE = True`, then run:

```bash
python3 /data/tesla_fsd_comma4.py
```

You should see synthetic CAN frames being generated and modified. This confirms the script logic works.

### Step 7: Start the Toggle Server

```bash
python3 /data/fsd_toggle_server.py
```

Open **`http://<comma-ip>:8088`** on your phone (same WiFi). You should see the toggle UI.

### Step 8: Toggle Between Modes

- **Comma Mode → FSD Mode**: Tap the button. openpilot stops, CAN mod starts, Tesla FSD activates.
- **FSD Mode → Comma Mode**: Tap again. CAN mod stops, openpilot restarts.

Transition takes ~5–10 seconds. **Always be ready to take manual control during the switch.**

### Step 9: Auto-Start on Boot (Optional)

To have the toggle server start automatically:

```bash
echo 'python3 /data/fsd_toggle_server.py &' >> /data/rc.local
```

---

## 📱 Web Toggle UI

Switch between FSD and openpilot from your phone — no SSH needed after setup.

The UI shows:
- **Current mode** — which system is active (openpilot or Tesla FSD)
- **Live CAN bus log** — real-time frame modifications scrolling
- **One big button** — tap to switch modes (~5–10 second transition)
- **Status cards** — panda connection, transmit state, speed profile, uptime

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
A: **Yes.** $99/mo. The subscription downloads the FSD neural net software to your car. Without it, there's no FSD brain to activate. The CAN mod flips the switch — it can't provide the software itself.

**Q: Does this work on HW3 vehicles?**
A: Not with these frame IDs. HW3 uses different CAN messages. See the [original project](https://gitlab.com/Starmixcraft/tesla-fsd-can-mod) for HW3 support.

**Q: Will Tesla OTA updates break this?**
A: Possibly. Tesla can change CAN message IDs or add new validation at any time. If an update breaks the mod, the toggle server will show errors and you can switch back to openpilot mode safely.

**Q: Can I use this outside the US/Canada?**
A: Yes — see the [FSD Outside Your Country](#-fsd-outside-your-country) section. You'll need to transfer your car to a US/Canadian Tesla account and may need GPS emulation.

**Q: Does this void my warranty?**
A: The comma 4 installation is reversible (unplug the harness). The CAN mod is software-only on the comma. However, Tesla's position on third-party modifications is generally unfavorable. Use your judgment.

**Q: Can both systems run at the same time?**
A: **No.** Only one system drives at a time. The toggle stops one and starts the other. Running both simultaneously would create conflicting steering/braking commands — very dangerous.

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
- Comma 4 / panda Python port: this repo
- International FSD activation research: [Michał Gapinski / TeslaAndroid](https://github.com/nicholasgapinski)

---

## 📝 License

MIT — do whatever you want, just don't blame anyone if something goes sideways.
