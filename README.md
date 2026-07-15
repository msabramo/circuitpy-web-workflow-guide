# Editing CircuitPython over WiFi when your Mac can't mount USB storage

Some managed / MDM-locked Macs are configured so they **cannot mount USB mass-storage
devices**. That breaks the normal CircuitPython workflow, where you edit `code.py` by
dragging files onto the `CIRCUITPY` drive that the board presents over USB.

This guide shows how to sidestep that entirely using **CircuitPython Web Workflow** —
editing your board's files over WiFi from a browser, plus `circup` and `curl` over the
network. No USB mass-storage mount required on your computer.

It also covers the one chicken-and-egg step (getting the first `settings.toml` onto the
board when your computer can't write to it) using any Linux machine that *can* mount USB
storage — an old Raspberry Pi works perfectly.

Tested with an **Adafruit Metro ESP32-S3** on **CircuitPython 10.x**, but the approach
applies to any WiFi-capable CircuitPython board (ESP32-S2/S3/C3/C6, etc.) on
CircuitPython 8.0 or newer.

---

## The problem in one picture

- Normal flow: computer mounts `CIRCUITPY` over USB → you drag files onto it. ❌ blocked by MDM.
- The board's own code also can't write the filesystem while USB is connected
  (`storage.remount("/", readonly=False)` raises
  `RuntimeError: Cannot remount path when visible via USB`).
- So neither side can write the file that would enable the fix. That's the bootstrap problem.

## The solution

1. Put a `settings.toml` with WiFi credentials + a web password on the board **once**,
   using a machine that *can* mount USB storage.
2. From then on, edit over WiFi — no USB mount ever needed again.

---

## Requirements

- A WiFi-capable CircuitPython board running **CircuitPython ≥ 8.0**
  (check by opening the serial REPL; the banner prints the version).
- Your **2.4 GHz** WiFi SSID + password (most of these boards are 2.4 GHz only).
- A serial terminal on your computer (`tio`, `screen`, `minicom`, or the Mu editor).
- For the one-time bootstrap: any Linux box that can mount USB mass storage
  (a Raspberry Pi, a Linux laptop, a VM with USB passthrough, etc.).

> **Cable gotcha:** use a known-good **USB data** cable. A charge-only USB-C cable will
> power the board (or not) but won't enumerate it — the board can look completely dead.

---

## Step 1 — Confirm CircuitPython and its version

Plug the board into any computer and open the serial console (this is *serial*, not USB
mass storage, so it works even on a locked Mac):

```bash
# macOS/Linux; device name varies
ls /dev/tty.usbmodem* /dev/cu.usbmodem* 2>/dev/null
tio /dev/tty.usbmodem*        # or: screen /dev/tty.usbmodem... 115200
```

Press <kbd>Ctrl-C</kbd> then <kbd>Enter</kbd> to get the `>>>` REPL. The banner shows the
version:

```
Adafruit CircuitPython 10.x on ...; <your board> with <chip>
>>>
```

You need **8.0 or newer** for web workflow.

---

## Step 2 — Understand why you can't just write the file

At the REPL you might try to make the filesystem writable from the board:

```python
import storage
storage.remount("/", readonly=False)
```

On a board connected to a USB host this fails:

```
RuntimeError: Cannot remount path when visible via USB.
```

The filesystem can only be writable by **one** side at a time, and while USB is connected
the host owns it. Since a locked Mac can't mount it *and* the board can't remount it,
you need a third machine for the one-time write.

---

## Step 3 — Bootstrap `settings.toml` from a Linux box (one time)

Any Linux machine that can mount USB storage works. (An old Raspberry Pi is great for this.)

1. Plug the board into the Linux box. Find its `CIRCUITPY` partition — it's a small
   (~a few MB) FAT partition:

   ```bash
   dmesg | tail -20     # watch it enumerate; note the device, e.g. sda1
   lsblk                # confirm the small (~8-16MB) partition
   ```

2. Mount it and confirm it's the right drive (you should see `boot_out.txt`, `code.py`, `lib/`):

   ```bash
   sudo mkdir -p /mnt/circuitpy
   sudo mount /dev/sda1 /mnt/circuitpy      # use YOUR device name from lsblk
   ls -la /mnt/circuitpy
   ```

3. Write `settings.toml` (use your real values; the quoted heredoc `<<'EOF'` prevents the
   shell from expanding `$` in passwords):

   ```bash
   sudo tee /mnt/circuitpy/settings.toml > /dev/null <<'EOF'
   CIRCUITPY_WIFI_SSID = "<YOUR_2.4GHZ_SSID>"
   CIRCUITPY_WIFI_PASSWORD = "<YOUR_WIFI_PASSWORD>"
   CIRCUITPY_WEB_API_PASSWORD = "<PICK_A_WEB_PASSWORD>"
   EOF
   cat /mnt/circuitpy/settings.toml
   ```

   - `CIRCUITPY_WEB_API_PASSWORD` is **what turns web workflow on** — without it the board
     won't start the web server, even with WiFi configured.
   - Keep the web password to letters/digits/hyphens to avoid TOML-quoting and shell headaches.
   - If any value contains a literal `"`, you'll need to escape it (TOML strings are quoted).

4. Unmount cleanly and remove the board:

   ```bash
   sync
   sudo umount /mnt/circuitpy
   ```

---

## Step 4 — Make the filesystem writable over WiFi

This is the subtle part, and it trips everyone up. **Web workflow can only write the
filesystem when USB Mass Storage is *disabled* on the board.** The board's own error dialog
says it plainly:

> The board's filesystem is in read-only mode for web workflow. This happens whenever USB
> Mass Storage is enabled on the board (the default for most CircuitPython boards),
> **whether or not a host computer is actively using it.**

So it is **not** enough to unplug from your computer, or to power it from a charger — by
default the board still *exposes* CIRCUITPY as a USB drive, which keeps web workflow
read-only. Writes fail with:

- editor: *"Could not save '/code.py'. The board's filesystem is in read-only mode…"*
- `curl` PUT: `HTTP 409 Conflict` with body `USB storage active.`
- (a plain `HTTP 500` on PUT is the older CircuitPython phrasing of the same thing)

### The fix: a dual-mode `boot.py`

Disable USB Mass Storage from `boot.py` so web workflow can write — but do it
**conditionally**, so you *keep* the ability to mount CIRCUITPY on a real computer when you
want to. The trick is `supervisor.runtime.usb_connected`, which is `True` only when a USB
**data host** is attached (a computer), and `False` on a power-only charger:

```python
# boot.py
import supervisor
import storage

# On a real USB data host (your home computer): leave USB storage ENABLED
#   -> CIRCUITPY mounts as a drive, edit it normally.
# On charger / no data host: DISABLE USB storage
#   -> web workflow can write over WiFi.
if not supervisor.runtime.usb_connected:
    storage.disable_usb_drive()
```

Result — the best of both worlds:

| Powered by | `usb_connected` | USB drive | How you edit |
|---|---|---|---|
| A computer (home Mac/PC, or the Linux bootstrap box) | `True` | mounts | drag files onto CIRCUITPY |
| A wall charger / battery | `False` | hidden | web workflow (WiFi) is writable |

**Chicken-and-egg again:** web workflow can't write `boot.py` while it's read-only, and a
locked Mac can't mount the drive — so write `boot.py` the same way you wrote `settings.toml`
(the Linux-box bootstrap in Step 3). `boot.py` only takes effect **after a reboot**, so
reset / power-cycle the board (on charger power) once it's in place.

> **Simpler but blunter alternative:** an unconditional `import storage;
> storage.disable_usb_drive()` also makes web workflow writable — but then CIRCUITPY will
> **never** mount over USB again (even on your home computer) until you remove `boot.py`.
> Prefer the conditional version above unless you truly never want the USB drive.

> **Note on `board.DISPLAY`/button variants:** some guides gate this on holding a button at
> boot (e.g. `board.D0` / the BOOT button) instead of `usb_connected`. Either works; the
> `usb_connected` approach needs no button press.

---

## Step 5 — Find the board and connect over WiFi

On boot the board joins WiFi and starts its web server. Get its address from the REPL:

```python
import wifi
print(wifi.radio.ipv4_address)     # e.g. 192.168.x.x
```

Then browse to either:

- `http://<BOARD_IP>/` — the raw IP (most reliable), or
- `http://circuitpython.local/` — mDNS alias, or
- `http://<board-hostname>.local/` — the board prints its hostname in `boot_out.txt`.

Log in with the **username blank** and the `CIRCUITPY_WEB_API_PASSWORD` you set. You get a
file browser and a full code editor (with a Serial/REPL tab) — all over WiFi.

> **mDNS tip:** `.local` names can be slow or time out on the *first* lookup after boot,
> then cache and become fast. If `.local` is flaky, use the IP.

---

## Step 6 (recommended) — Pin the IP

DHCP may hand the board a different address after a reboot. To keep the IP stable, add a
**DHCP reservation** for the board's MAC in your router. Then `http://<BOARD_IP>/` and the
commands below always work.

### Optional: a friendly name via `/etc/hosts`

Once the IP is reserved, you can give the board a short name that doesn't depend on mDNS
at all (handy when `.local` lookups are slow). Add a line to `/etc/hosts`:

```
<BOARD_IP>   metro circuitpy
```

Then `http://metro/` just works. **Avoid using a `.local` name here** — that suffix is
reserved for mDNS/Bonjour, and macOS may ignore your `/etc/hosts` entry for it. Pick a
plain name instead.

---

## Everyday use — no browser required

Once web workflow is on, you can work from your normal editor and push over HTTP:

```bash
# Push a single file
curl -u :<WEB_PASSWORD> -T code.py http://<BOARD_IP>/fs/code.py

# List the filesystem
curl -s -u :<WEB_PASSWORD> http://<BOARD_IP>/fs/

# Install/update libraries with no USB mount at all
circup --host <BOARD_IP> --password <WEB_PASSWORD> install <library>
```

### Make `circup` less verbose to invoke

`circup` reads the web password from the environment variable
`CIRCUP_WEBWORKFLOW_PASSWORD`, so you only have to pass `--host`. Wrap that in a shell
alias and you type neither:

```bash
# ~/.zshrc (or ~/.bashrc)
export CIRCUP_WEBWORKFLOW_PASSWORD="<WEB_PASSWORD>"
alias mcircup='circup --host <BOARD_IP>'      # or --host metro if you added /etc/hosts
```

Then: `mcircup list`, `mcircup install neopixel`, `mcircup update`. (Each run refreshes the
library bundles from GitHub, which is most of the couple-second runtime — not the board.)

---

## Test it — a quick demo

A satisfying, dependency-light test on a board with an onboard NeoPixel (e.g. Metro
ESP32-S3): install the library over WiFi, then push a rainbow.

```bash
mcircup install neopixel      # installs neopixel + adafruit_pixelbuf over WiFi, no mount
```

Then save this as `code.py` (via the web editor, or `curl -T`); the board auto-reloads and
the onboard pixel cycles colors:

```python
import time
import board
import neopixel

pixel = neopixel.NeoPixel(board.NEOPIXEL, 1, brightness=0.2, auto_write=True)

def wheel(pos):
    if pos < 85:
        return (pos * 3, 255 - pos * 3, 0)
    if pos < 170:
        pos -= 85
        return (255 - pos * 3, 0, pos * 3)
    pos -= 170
    return (0, pos * 3, 255 - pos * 3)

i = 0
while True:
    pixel[0] = wheel(i % 255)
    i = (i + 5) % 255
    time.sleep(0.1)      # keep this modest — see note below
```

> **Don't use a WiFi scan as your "hello world."** `wifi.radio.start_scanning_networks()`
> takes the radio off your access point while it scans, which **drops the web-workflow
> connection** ("device not found"). Reset to recover. Pick a demo that doesn't touch the
> radio.

> **Keep animation loops modest.** These boards are single-core; a tight
> `while True: ... time.sleep(0.01)` (~100 Hz) starves the WiFi/web-workflow stack and makes
> the editor and `circup` slow/flaky. `time.sleep(0.1)` (~10 Hz) looks just as smooth and
> leaves plenty of CPU for networking. If you want the editor maximally responsive, keep
> `code.py` idle (e.g. a `print`) and run animations only on demand.

---

## Updating CircuitPython firmware — heads-up

Firmware updates (`.uf2` to the `BOOT` drive) **do** require USB mass storage — the exact
thing a locked Mac can't do. If you need to update firmware, do it from the same Linux box
you used for bootstrapping (or flash the `.bin` with `esptool`). Don't chase pre-release
(alpha/beta) firmware on a board that's working fine.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Board looks completely dead, no serial device | Charge-only cable — swap for a data cable. |
| Web workflow is read-only ("USB is using the storage") | Board is on a USB data host — power it from a charger instead. |
| File writes fail with HTTP 500 (empty body); reads/`version.json` still work | Same read-only cause — board is on a USB data host. Move it to charger power so the board owns its filesystem. |
| Board vanishes from the network right after running code | Your code called `wifi.radio.start_scanning_networks()` — scanning drops the board off its access point, which kills the web-workflow connection ("device not found"). Don't scan on a board you manage over WiFi. Reset/power-cycle to recover. |
| `storage.remount` → "Cannot remount path when visible via USB" | Expected while on USB; use the Linux-box bootstrap (Step 3). |
| `wifi.radio.ipv4_address` is `None` | Wrong SSID/password, or a 5 GHz-only network — the board is 2.4 GHz. |
| `.local` name slow or failing | mDNS first-lookup lag; use the raw IP, ideally a reserved one. |
| Web server not starting at all | Missing `CIRCUITPY_WEB_API_PASSWORD` in `settings.toml`. |

---

## References

- [CircuitPython Web Workflow — Adafruit Learn](https://learn.adafruit.com/getting-started-with-web-workflow-using-the-code-editor)
- [`circup`](https://github.com/adafruit/circup)
- [CircuitPython downloads](https://circuitpython.org/downloads)

---

*Written up after solving this on a locked corporate Mac. Contributions/corrections welcome.*
