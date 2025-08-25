# Arduino IoT Cloud – 4-Relay Home Automation (ESP8266/NodeMCU) + Google Assistant

Control four appliances with a NodeMCU (ESP8266), a 4‑channel relay board, and **Arduino IoT Cloud**.  
You’ll get **three control modes**:  
1) Arduino IoT Cloud Dashboard (web/mobile),  
2) **Voice** via Google Assistant (through Google Home integration), and  
3) **Manual wall switches** (work even without internet).

> This README is based on the code you shared and the standard Arduino Cloud + ESP8266 workflow. It includes step‑by‑step setup, wiring, and voice control. Placeholders are added where you’ll drop your own photos/diagrams.

---

## ✨ Features
- 4 independent relays (active‑LOW) mapped to Arduino Cloud **CloudSwitch** variables: `switch1 … switch4`  
- Manual switches on GPIOs with **hardware debouncing via code**  
- Wi‑Fi status LED (D0)  
- Works from: **Cloud dashboard**, **Google Assistant**, and **manual switches**

---

## 🧰 Bill of Materials
- NodeMCU **ESP8266** (Amica/LoLin)
- 4‑Channel relay module (5V, opto‑isolated preferred; 3.3V logic‑compatible inputs)
- 4 x momentary/toggle switches (SPST)
- Power: 5V (for relay board) + USB 5V for NodeMCU (or common 5V with proper regulation)
- Jumper wires, breadboard or screw terminals
- **(Mains caution)** If you switch AC loads, use proper enclosures, fuses, and isolation. Never touch live wires.

---

## 🔌 Pinout & Wiring

### Relay Channels → NodeMCU
| Relay | NodeMCU Pin | ESP8266 GPIO | Notes |
|------:|-------------|--------------|-------|
| 1 | **D1** | GPIO5  | Active‑LOW (LOW = ON) |
| 2 | **D2** | GPIO4  | Active‑LOW (LOW = ON) |
| 3 | **D5** | GPIO14 | Active‑LOW (LOW = ON) |
| 4 | **D6** | GPIO12 | Active‑LOW (LOW = ON) |

> The sketch initializes each relay **OFF** with `digitalWrite(HIGH)` and turns **ON** with `LOW`. Most 5V relay boards behave like this (active‑LOW).

### Manual Switches → NodeMCU (INPUT_PULLUP)
Wire each switch **between the pin and GND**.

| Switch | NodeMCU Pin | ESP8266 GPIO | Boot/Wiring Notes |
|------:|--------------|--------------|-------------------|
| 1 | **SD3** | GPIO10 | On many ESP8266 boards GPIO9/10 are used for flash; works on some NodeMCU variants but can be unreliable. Prefer alternatives below if you see random resets. |
| 2 | **D3** | GPIO0 | Must be **HIGH at boot**; OK with `INPUT_PULLUP` and **momentary to GND** (don’t hold during reset). |
| 3 | **D7** | GPIO13 | Safe for inputs with `INPUT_PULLUP`. |
| 4 | **RX** | GPIO3 | Shares UART RX. Avoid using Serial at high baud while using this pin as input, or move to another pin. |

**Wi‑Fi LED:** D0 (GPIO16), LED **ON** when connected (driven LOW).

#### Safer Alternate Switch Pins (recommended if you hit boot/serial issues)
- Use **D1, D2, D5, D6** for relays (as above). For switches, prefer **D7 (GPIO13), D8 (GPIO15 – only as OUTPUT), SD2 (GPIO9 – if available)**, or attach an **I/O expander** (MCP23017) if you run out of pins.  
- **Do not** pull **GPIO0 (D3)** LOW during reset; device won’t boot.  
- Avoid **TX/RX** if you need Serial logging.

### Relay Powering
- Relay board **VCC** → 5V, **GND** → GND.  
- **IN1..IN4** → D1, D2, D5, D6.  
- If your relay board needs a separate **JD‑VCC** (opto‑isolated type), power JD‑VCC from 5V and **GND** common to NodeMCU.  
- Ensure the relay input accepts **3.3V logic**. If not, use a transistor/ULN2003 driver.

> **Mains Safety:** Keep high‑voltage wiring separated. Use proper terminal blocks, enclosure, strain relief, and fuses. Consider an electrician for AC work.

---

## 💻 Arduino IDE Setup

1. **Boards Manager URLs**  
   `File → Preferences → Additional Boards Manager URLs` (comma‑separate):  
   ```
   http://arduino.esp8266.com/stable/package_esp8266com_index.json
   https://dl.espressif.com/dl/package_esp32_index.json
   ```
2. **Install ESP8266 core**  
   `Tools → Board → Boards Manager…` → search **esp8266** → install.
3. **Libraries**  
   - **ArduinoIoTCloud** (this auto‑pulls dependencies)  
   - **Arduino_ConnectionHandler**
4. **Select board/port**  
   `Tools → Board → NodeMCU 1.0 (ESP-12E Module)` and the correct COM port.

> **Compile tip:** In the sketch, the correct header is `#include <ESP8266WiFi.h>` (uppercase **WiFi**). If you see a compile error, fix this include line.

---

## ☁️ Arduino IoT Cloud Setup (Thing + Variables)

  1.download Arduino IoT Cloud Remote for mobile: https://play.google.com/store/apps/details?id=cc.arduino.cloudiot
2. Create a **Thing**: add your **Device** (ESP8266) to the Thing. You’ll get a **Device ID** and **Secret Device Key**.  
3. Add **4 Cloud variables** (type **Switch** / boolean, read‑write):  
   - `switch1`, `switch2`, `switch3`, `switch4`  
   Set **Update mode**: *On change*.  
4. (Optional) Create a **Dashboard** with 4 Switch widgets linked to the variables.
5. Copy credentials into the sketch:
   ```cpp
   const char DEVICE_LOGIN_NAME[] = "YOUR_DEVICE_ID";
   const char DEVICE_KEY[]        = "YOUR_SECRET_DEVICE_KEY";
   const char SSID[]              = "YOUR_WIFI_SSID";
   const char PASS[]              = "YOUR_WIFI_PASSWORD";
   ```
6. **Upload** the sketch. Open Serial Monitor @ **9600** baud to watch Cloud connection logs.  
7. Toggle the **Dashboard** switches and verify the relays click accordingly.

---

## 🗣️ Google Assistant (Google Home) Integration

Arduino Cloud now supports **Google Home** directly. Create **Smart Home‑compatible** variables (Switch/Light/Plug) and link your Arduino account in Google Home. Each variable appears as a device you can voice‑control.

**Steps (once):**
1. In Arduino Cloud, ensure your 4 variables are of a **Smart Home compatible** type (Switch is OK).  
2. In the Cloud integration settings, enable **Connect to Google Home**.  
3. Open the **Google Home** app → **Add device** → **Works with Google** → search **Arduino** → link your Arduino account.  
4. Assign the discovered devices (your 4 switches) to rooms.  
5. Try: “*Hey Google, turn on Relay 1*”.

> If Google Home isn’t available for you, you can alternatively use **IFTTT** with Google Assistant → **Webhook** to call Arduino Cloud’s API and update a variable. That path requires generating an Arduino Cloud API token and using the REST endpoint to set a property.

---

## 🧩 How the Code Works (overview)
- Declares 4 **CloudSwitch** variables: `switch1..switch4`, added to Cloud with `READWRITE` and `ON_CHANGE` callbacks.  
- **manual_control()** reads the 4 switch pins (with `INPUT_PULLUP`) and toggles the corresponding relay & cloud variable with a simple debounce.  
- **onSwitchXChange()** functions drive each relay **LOW (ON)** or **HIGH (OFF)** based on the Cloud variable—keeps cloud, relays, and physical switches in sync.  
- **Wi‑Fi LED** shows Cloud/Wi‑Fi status: **LOW = connected**.

---

## ✅ Test Plan
1. Power NodeMCU via USB (or 5V), power relay board with 5V.  
2. Watch Serial logs until `Connected to Arduino IoT Cloud`.  
3. Toggle switches in **Dashboard**; verify relays and Serial prints.  
4. Press **physical switches**; relays should toggle and Dashboard should update.  
5. In Google Home, say “*turn on Relay 1*”; verify relay 1 engages.

---

## 🛠 Troubleshooting
- **Relays inverted** (ON when you expect OFF): your module might be active‑HIGH. Invert logic by writing `HIGH` for ON and `LOW` for OFF (or swap to a standard active‑LOW relay).  
- **Boot loops / won’t flash**: remove anything from **D3 (GPIO0)** during reset/flash; keep it HIGH. Avoid holding the switch.  
- **Random resets**: avoid using **GPIO9/10** on some ESP8266s; move Switch 1 from **SD3 (GPIO10)** to a safer pin.  
- **Serial conflicts**: if using **RX (GPIO3)** as a switch, keep Serial quiet or remap that switch.  
- **No Cloud connection**: double‑check **Device ID** and **Secret Device Key**; verify Wi‑Fi SSID/PASS; ensure time/date are correct.  
- **Google Home can’t find devices**: confirm variables are Smart‑Home compatible and the Arduino account is linked in the Home app.

---

## 🔒 Safety Notes
- When switching **AC mains**, use rated relays and proper insulation. Add snubber/RC networks for inductive loads (motors, pumps).  
- Use separate supplies for NodeMCU and relays if you see brownouts (ground must be common).

---

## 🙌 Credits
- Hardware & sketch: your project with NodeMCU + Arduino IoT Cloud
- Voice control: Arduino Cloud ↔ Google Home integration

