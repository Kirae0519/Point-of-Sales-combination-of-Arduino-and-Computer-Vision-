# Point of Sales Arduino x Computer Vision

A PyQt6 cashier app using your Brew Prince artwork, with webcam barcode/QR/sticker detection, an Arduino-owned cart, and a hardware-free Test Mode.

**Start with Test Mode. The original uploaded Arduino sketch cannot communicate with this app. Upload the included integrated sketch before using a real COM port.**

## 1. Project contents

| File/folder | Purpose |
|---|---|
| `main.py` | Startup, command-line options, one-instance lock |
| `ui.py` | Three-panel window, menu manager, keypad, sounds, scan acceptance |
| `scanner.py` | Camera thread, ZXing decoding, ORB matching, hold/remove protection |
| `serial_link.py` | Serial thread, atomic report parser, independent Arduino simulator |
| `menu.py` | JSON validation, card replacement, daily ticket counter, Arduino menu export |
| `config.py` | Theme, scanner thresholds, timing, resource/data paths |
| `menu.json` | Single editable source for drink definitions |
| `requirements.txt` | Compatible dependency ranges |
| `requirements-tested.txt` | Exact dependency versions used for verification |
| `menu_cards/` | Five cropped original card images, also ORB references |
| `assets/` | Original logo, two WAV sounds, Inter/Cinzel fonts and their licenses |
| `scan_codes/` | Five QR images and five Code 128 barcode images |
| `arduino/BrewPrincePOS/` | Complete integrated Uno sketch and generated menu header |
| `build_windows.ps1` | Windows PyInstaller build command |
| `PROTOCOL.md` | Commands, reports and smallest firmware integration explanation |
| `tests/` | Workflow, detector, GUI and serial transport tests |
| `UI_Preview.png` | Actual app screenshot at 1366 × 768, in Test Mode |
| `VALIDATION.md` | Checks performed and remaining hardware checks |

The screenshot has no webcam attached. It shows a simulated ₱330 cart (two Americanos and one Mocha); these example transactions are not included as saved order data.

## 2. Install and run on Windows

Install **64-bit Python 3.12**. Extract the ZIP, open its `brew_prince` folder, then open PowerShell there. Use the virtual environment executable directly; activation is optional.

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe main.py --test
```

No Arduino or camera is needed for that command. If installing a future dependency release causes trouble, recreate the environment using `requirements-tested.txt`.

To start the webcam in Test Mode:

```powershell
.\.venv\Scripts\python.exe main.py --test --camera 0
```

For the real Arduino after uploading the included firmware:

```powershell
.\.venv\Scripts\python.exe main.py --port COM7 --camera 0
```

Replace COM7 with your actual port. Close Arduino Serial Monitor first; only one program can own the port. Use **Refresh ports** if the board was just plugged in. The camera dropdown lists indices 0–7; choose an index and click **Start camera**. It does not claim those indices are all connected devices.

Use **F11**, the Fullscreen button, or `--fullscreen`. The window fits 1366 × 768 at 100% display scaling. Qt supports high-DPI scaling; on a laptop whose Windows scaling reduces the effective workspace below 1150 × 700, use 100% scaling or a larger display. The menu area scrolls if it needs more room.

## 3. First complete test order

1. Enable Test Mode. The banner should say **TEST MODE · No hardware actions or printed receipts**.
2. Click Americano. The state becomes **Choosing quantity**. A card click sends `SCAN,1`, exactly as a camera scan does.
3. Open **KEYPAD / INPUT**. Press **2**, then **B · OK**. The reported cart becomes Americano ×2, total ₱200.
4. Close the keypad dialog and press **CHECKOUT #**.
5. Press **PAY #**. State becomes Payment and the simulated drawer opens.
6. In the keypad, enter **500**, then **#**. Change should be ₱300. The payment button changes to **FINISH ORDER #**.
7. Give change, then press **FINISH ORDER #**. The device locks its drawer, clears its cart and returns to Idle.

**OPEN CASHIER BOX** unlocks without recording a sale, and is enabled only in Idle with an empty cart. **LOCK DRAWER** locks without completing a sale. **NEW ORDER** starts a new empty order from Idle; it does not silently discard an existing cart.

The keypad is also available in real mode. A/C browse, B selects or confirms quantity, D backspaces (or requests deletion in checkout), `*` goes back, and `#` advances checkout/payment. Checkout edits/deletions act on the Arduino's current list position, shown on its LCD; use A/C to navigate. The laptop table is a live summary, not a separately editable cart.

## 4. Arduino installation

Open `arduino/BrewPrincePOS/BrewPrincePOS.ino` in Arduino IDE. Keep `menu_generated.h` beside it. Select Arduino Uno and the board's port. Install these libraries through Library Manager:

- Keypad
- LiquidCrystal I2C (the `LiquidCrystal_I2C.h` library)
- Servo
- Adafruit Thermal Printer Library

Wire and SoftwareSerial come with the AVR core. This version preserves the hardware mapping from your uploaded sketch:

| Component | Pins / address |
|---|---|
| Keypad rows | 7, 8, 9, 10 |
| Keypad columns | 3, 4, 5, 6 |
| Drawer servo | 11; locked 90°, open 0° |
| Active buzzer | 2 |
| Printer | SoftwareSerial RX 12, TX 13, 9600 baud |
| Cashier LCD | I²C `0x27` |
| Customer LCD | I²C `0x26` |
| Laptop serial | USB, 9600 baud, LF-terminated ASCII |

Upload before connecting the laptop app. Opening a typical Uno USB serial connection can reset the board and clear its RAM cart. Finish a sale before switching modes/reconnecting. The app detects boot messages, invalidates its old view and waits for a fresh menu and complete cart snapshot. It never replays an uncertain sale command after reconnecting.

The included sketch also uses 32-bit totals/cash, limits each drink to 99 and each order to ₱99,999, uses LCD short names, and prints line **subtotals** rather than unit prices under the receipt's subtotal column. It moves fixed text into flash to reduce SRAM use. Existing servo/printer/error-display routines still contain blocking delays; the laptop's video/UI run independently. Commands are serialized to avoid flooding the Uno while it is busy.

## 5. Menu changes and price agreement

The JSON already contains:

| ID | Drink | LCD short name | Price | Barcode | QR |
|---|---|---|---:|---|---|
| 1 | Americano | Americano | 100 | BP001 | BREWPRINCE:1 |
| 2 | Cappuccino | Cappuccino | 100 | BP002 | BREWPRINCE:2 |
| 3 | Caramel Macchiato | Caramel | 109 | BP003 | BREWPRINCE:3 |
| 4 | Latte | Latte | 120 | BP004 | BREWPRINCE:4 |
| 5 | Mocha | Mocha | 130 | BP005 | BREWPRINCE:5 |

These prices match both the uploaded menu artwork and the supplied sketch. Prices are whole pesos. Short names must be ASCII, at most 10 characters. Code texts must be unique and are case-sensitive. IDs must stay consecutive starting at 1 when exported to the Arduino.

The application builds all menu cards and code dictionaries from `menu.json`. The Arduino uses a **generated copy** of this data because its firmware is compiled. To change a price/name:

1. Close the app and edit `menu.json`.
2. Generate the firmware header from that same file:

```powershell
.\.venv\Scripts\python.exe menu.py --export-arduino arduino/BrewPrincePOS/menu_generated.h
```

3. Compile/upload the Arduino sketch, then restart/reconnect the app.

Do not edit `menu_generated.h` by hand. At startup/reconnect the app requests `GET_MENU`, compares IDs, names and prices, and displays any mismatch. New drink selections and item confirmations from the app are blocked until corrected; the physical keypad remains controlled by the Arduino. Manual checks alone would not support this automatic comparison. The live cart always uses Arduino-reported subtotals and total, even if a mismatch is found.

**Menu Manager** lets you view the menu and replace an image. It validates the image, saves a PNG under `menu_cards`, updates JSON and reloads the tiles/ORB references without code changes. It intentionally does not edit prices. Old image files remain available. If the design has a price printed into the image, update that artwork too when changing price; tile labels are rendered from JSON.

## 6. Sticker and code scanning

Choose **Auto**, **Barcode**, **QR**, or **Sticker**. Auto prioritizes decoded codes, then tries ORB. Hold one card at a time. If several codes are visible, a known code is preferred, then the largest visible code. Unknown decoded values show **Unknown code** without sending an Arduino command. Unrecognized artwork simply stays idle.

For stickers, use the **exact full card image being printed** as the reference, including its drink illustration and distinctive text. Recommended reference:

- **800 × 840 pixels** for the current portrait proportions, or an equivalent aspect-preserving export with a 700–1200 px long edge.
- PNG preferred; high-quality JPEG (roughly 90–95 quality) also works. Keep original detail; enlarging a tiny image does not create features.
- One card tightly framed, front-on, sharp, no glare, no desk/background, and no watermark.
- Print approximately 65–75 mm wide on matte material. Keep the whole card visible and about **200 pixels wide or more in the 960-pixel detection frame**. Closer placement and steady focus matter more than raising ORB feature counts.

The included references are crops from the uploaded poster, approximately 248 × 250–260 px. They are usable starter assets and passed the synthetic tests, but larger original exports will tolerate distance and blur better. The UI displays the drink region; ORB always uses the full file.

Scanner settings are centralized in `config.py`:

| Setting | Default | How to tune |
|---|---:|---|
| `ORB_FEATURES` | 2200 | Try 3000 if detailed cards have too few matches; costs more CPU |
| `ORB_RATIO` | 0.72 | Lower to 0.65 for stricter descriptor matches; raise toward 0.78 only if clean cards are often missed |
| `ORB_MIN_MATCHES` | 18 | Raise to 24 to reduce false candidates |
| `ORB_MIN_INLIERS` | 14 | Raise to 20 for stronger geometric evidence |
| `ORB_MIN_INLIER_RATIO` | 0.65 | Raise to 0.75 if similar cards are confused |
| `ORB_RANSAC_PX` | 4.0 | Try 3.0 for stricter alignment; thresholds refer to resized detection pixels |
| `ORB_SCORE_MARGIN` | 0.12 | Rejects ambiguous winners; raise toward 0.18 if similar cards are confused |
| `STABLE_FRAMES` | 3 | Consecutive detection observations before proposing a scan |
| `SCAN_COOLDOWN` | 1.5 s | Shared acceptance cooldown for cards and webcam scans |
| `REMOVE_TO_REARM` | 0.8 s | Same visible object must disappear this long to rearm |

Tune one parameter at a time using all five cards and non-menu distractors under your actual lighting. First fix focus, glare and distance. Test false accepts as well as missed scans. ORB uses ratio-tested matches, RANSAC homography, an inlier threshold, coverage and polygon checks, and a winner margin. Its displayed score is **not a calibrated probability**. A barcode/QR displays **Decoded**, not an invented confidence percentage.

The accepted beep, last-scan banner and animation occur only after `ACK,SCAN,<id>`. While choosing quantity, another accepted drink scan replaces the selection and clears the unfinished quantity; it does not add an item automatically. A continuously held sticker cannot add repeated selections. If you present a new card during cooldown, remove it and present it again once ready.

Print or display the included `scan_codes` images with their white quiet zones intact. They encode the values in the table above. They do **not** encode GCash, Maya or bank-payment destinations from the poster.

## 7. Theme and saved data

Edit `THEME` in `config.py`: cream `#F8F0E1`, espresso `#241307`, coffee `#70401E`, gold `#D8A34F`, ivory `#FFF9EC`. Fonts are bundled Inter and Cinzel, with their OFL license files. The logo preserves its original lettering. Changing source theme values requires restarting; a packaged app requires rebuilding.

In source mode, editable menu/assets and counters live beside `main.py`. A packaged app copies the initial menu/assets to `%LOCALAPPDATA%\BrewPrince` on first run, so changes survive executable replacement and do not write into PyInstaller's temporary bundle. Menu Manager shows the actual menu path. Existing user data is intentionally not overwritten by a newer executable. `--data-dir <folder>` selects a separate writable dataset.

`orders.json` stores only the daily display number, not a cart. Test Mode uses `orders_test.json`. Numbers reserve on the first nonempty cart, persist across app restarts, and reset with the Windows local date. Set Windows to your correct time zone. Resets/cancellations can leave gaps. Midnight starts the new day's sequence even for an order left open. These are laptop display tickets, not permanent accounting records or Arduino receipt numbers.

## 8. Build a Windows executable

Build **on Windows** using the same 64-bit Python environment:

```powershell
.\.venv\Scripts\python.exe -m pip install "pyinstaller>=6,<7"
.\.venv\Scripts\python.exe -m PyInstaller --noconfirm --clean --windowed --onedir --name BrewPrince --add-data "menu.json:." --add-data "menu_cards:menu_cards" --add-data "assets:assets" --hidden-import zxingcpp main.py
```

Or run `build_windows.ps1` if PowerShell scripts are enabled. The output is `dist\BrewPrince\BrewPrince.exe`. Distribute the **entire** `dist\BrewPrince` folder. Run `BrewPrince.exe --test` first. To make one executable, substitute `--onefile` for `--onedir`; the editable files still live under LocalAppData.

For a startup failure, rebuild without `--windowed` to see the console traceback. This delivery includes source and build instructions, not a prebuilt Windows binary.

## 9. Troubleshooting and tests

| Symptom | Action |
|---|---|
| Camera not found | Try another index; close apps using the camera; allow desktop camera access in Windows Settings |
| Unknown code | Compare the exact decoded text with `barcode_text` / `qr_text`; payment QR codes are not drinks |
| Menu unverified | Upload this project's firmware; old keypad-debug firmware cannot report its cart/menu |
| COM lost | Finish/recover the physical transaction as needed, reconnect cable, choose port, press Reconnect |
| Repeated scan blocked | Wait 1.5 s and remove the card for at least 0.8 s |
| Sticker not recognized | Use full original reference, brighter diffuse light, less glare, closer distance, sharper focus |
| No audio | Check Windows output device and app volume; WAV files are under assets |
| Source import error | Use `.venv\Scripts\python.exe`, not a different Python installation |

Run the included tests:

```powershell
.\.venv\Scripts\python.exe -m pip install pytest
.\.venv\Scripts\python.exe -m pytest tests -q
```

The pseudo-terminal serial test runs on POSIX and is skipped on Windows; simulator, detector and GUI tests remain available. See `VALIDATION.md` for the exact scope of checks.

Technical references: [PyQt](https://www.riverbankcomputing.com/software/pyqt), [Qt threading](https://doc.qt.io/qt-6/threads-qobject.html), [OpenCV homography](https://docs.opencv.org/4.x/d7/dff/tutorial_feature_homography.html), [ZXing-C++](https://github.com/zxing-cpp/zxing-cpp), [PyInstaller usage](https://pyinstaller.org/en/stable/usage.html), [PyInstaller resource paths](https://pyinstaller.org/en/stable/runtime-information.html). Font sources: [Inter](https://github.com/google/fonts/tree/main/ofl/inter), [Cinzel](https://github.com/google/fonts/tree/main/ofl/cinzel).
