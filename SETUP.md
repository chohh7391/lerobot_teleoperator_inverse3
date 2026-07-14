# Setup Guide

LeRobot-compatible teleoperator for the Haply Inverse3 + VerseGrip stylus.

This repository ships **only the open bridge/teleop code**. The Haply HardwareAPI SDK
(proprietary static library + headers) is **not** included and must be downloaded
separately from Haply, then placed and built locally.

> **Using this as a FIRe git submodule?** If you cloned the parent repo without
> `--recurse-submodules`, fetch it first:
>
> ```bash
> git submodule update --init --recursive
> ```

## 1. Download the Haply HardwareAPI SDK

Releases page: <https://develop.haply.co/releases/cpp>

Linux C++ SDK (version used by this bridge — `0.2.8`):

```bash
cd src/lerobot_teleoperator_inverse3/inverse3_bridge
curl -L -o HardwareAPI.0.2.8.linux.tar.gz \
  https://cdn.haply.co/r/33857671/HardwareAPI.0.2.8.linux.tar.gz
```

## 2. Place it as `sdk/`

Extract the archive and **rename the extracted folder to `sdk`**, located at
`src/lerobot_teleoperator_inverse3/inverse3_bridge/sdk`:

```bash
tar -xzf HardwareAPI.0.2.8.linux.tar.gz
# Rename the extracted directory (name may vary by release) to exactly "sdk".
mv HardwareAPI.0.2.8.linux sdk
rm HardwareAPI.0.2.8.linux.tar.gz
```

The result **must** have this layout (the `Makefile` references these exact paths/names):

```text
src/lerobot_teleoperator_inverse3/inverse3_bridge/sdk/
├── include/            # Haply headers: Inverse3.h, Handle.h, SerialStream.h, HardwareAPI.h, ...
└── libHaply.HardwareAPI.a   # static library (this exact filename)
```

> If the release lays the files out differently, move/rename them so that
> `sdk/include/*.h` and `sdk/libHaply.HardwareAPI.a` exist. The build looks for
> `$(sdk)/include` and `$(sdk)/libHaply.HardwareAPI.a` (see `Makefile`).

## 3. Build the C++ bridge server

Produces the `inverse3_server` binary that the Python package launches as a subprocess:

```bash
cd src/lerobot_teleoperator_inverse3/inverse3_bridge
make
```

## 4. Install the Python package

```bash
pip install -e .
```

## 5. Serial ports (udev)

The `/dev/ttyACM*` numbers change on every reconnect/reset, and the two devices (Inverse3 =
Teensy, VerseGrip = USB transceiver) can swap numbers with each other. Pin a fixed symlink based
on the USB serial with a udev rule so you can always access them by the same name (e.g.
`/dev/inverse3_left`).

The repository ships a left-arm rule at `udev/99-haply-inverse3.rules`; you can `sudo cp` it, or
write the rule file manually as below to cover the right arm as well in one go:

```bash
sudo gedit /etc/udev/rules.d/99-inverse3-haptic.rules
```

Paste in the following:

```udev
SUBSYSTEM=="tty", ATTRS{idVendor}=="16c0", ATTRS{idProduct}=="0483", ATTRS{serial}=="16894910", MODE="0666", SYMLINK+="inverse3_left", ENV{ID_MM_DEVICE_IGNORE}="1"

SUBSYSTEM=="tty", ATTRS{idVendor}=="2fe3", ATTRS{idProduct}=="0100", ATTRS{serial}=="548F7FD33638E628", MODE="0666", SYMLINK+="versegrip_left", ENV{ID_MM_DEVICE_IGNORE}="1"

SUBSYSTEM=="tty", ATTRS{idVendor}=="16c0", ATTRS{idProduct}=="0483", ATTRS{serial}=="16895170", MODE="0666", SYMLINK+="inverse3_right", ENV{ID_MM_DEVICE_IGNORE}="1"

SUBSYSTEM=="tty", ATTRS{idVendor}=="2fe3", ATTRS{idProduct}=="0100", ATTRS{serial}=="7B42902DBE133AE4", MODE="0666", SYMLINK+="versegrip_right", ENV{ID_MM_DEVICE_IGNORE}="1"
```

> `ENV{ID_MM_DEVICE_IGNORE}="1"` prevents ModemManager from mistaking this port for a modem and
> probing it (with AT commands). Without it, ModemManager opens the port on every plug, breaking
> the serial handshake and causing `timeout waiting for header code`.
> (See [Troubleshooting](#8-troubleshooting).)

Save and re-apply the rule:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger --action=add
sudo systemctl restart ModemManager   # make it recognize ENV{ID_MM_DEVICE_IGNORE}
```

> `ID_MM_DEVICE_IGNORE` takes effect only from the *next plug*, so during first-time setup the
> most reliable approach is to **unplug and replug** the Inverse3/VerseGrip USB once after
> running the above commands.

Replace each entry's `serial` value with your own device's value. Check the serial:

```bash
udevadm info -q property -n /dev/ttyACM2 | grep -E 'ID_SERIAL=|ID_MODEL='
```

- Inverse3 = `idVendor 16c0` / `idProduct 0483` (Teensyduino "Haply inverse3")
- VerseGrip = `idVendor 2fe3` / `idProduct 0100` (ZEPHYR "Haply USB Transceiver")

## 6. Verify the connection

```bash
ls -l /dev/inverse3_left /dev/versegrip_left
ls -l /dev/ttyACM*
```

If the symlinks are present, everything is fine. Check which ttyACM each physical device maps to:

```bash
for d in /dev/ttyACM*; do
  echo "=== $d ==="
  udevadm info -q property -n "$d" | grep -E 'ID_MODEL=|DEVLINKS='
done
```

It is OK if `Haply_inverse3` maps to `inverse3_*` and `Haply_USB_Transceiver` maps to `versegrip_*`.

## 7. Functional check

Quickly checking **the device alone**, without the robot/ROS stack, is the best way to isolate
problems. Launch the bridge server directly and send OPEN → STATE → CLOSE in one shot:

```bash
cd src/lerobot_teleoperator_inverse3/inverse3_bridge
printf 'OPEN /dev/inverse3_left /dev/versegrip_left\nGET_STATE\nCLOSE\n' | timeout 30 ./inverse3_server
```

Example of normal output:

```text
[info] Inverse3 gravity compensation: disabled
OK                               # "OK" if VerseGrip is powered on, "OK GRIP_UNAVAILABLE" if off
STATE 0 0 0 0 0 0 1 0 0 0 0 0
```

- If you see `OK` (or `OK GRIP_UNAVAILABLE`) and `STATE`, **the device is fine**. If your teleop
  client (e.g. FIRe's `record.py`) fails after this, the problem is on the Python/ROS side, not
  the device.
- Seeing `[err] ... timeout waiting for header code` once is normal: on a cold start, the
  Inverse3's first wakeup times out once and then retries. As long as it eventually reaches `OK`,
  there is no problem.
- Right after first powering on (cold), wakeup retries can take several to a dozen-plus seconds to
  reach `OK`.

To also verify the device plus the teleop conversion:

```bash
python test/test_hardware_live.py --teleop
```

## 8. Troubleshooting

### `timeout waiting for header code` / `No response from inverse3_server within 8.0s`

```text
[err] .../Device.cpp:87> timeout waiting for header code: [0] Success
TimeoutError: [Inv3Server] No response from inverse3_server within 8.0s
```

The serial port opened, but the Inverse3 fails to send the header frame in time. In practice this
error arises from two overlapping causes. Handle them in the order below.

First, isolate whether the device itself is fine by running the bridge server directly as in
[7. Functional check](#7-functional-check). If direct execution shows `OK` but only your teleop
client fails, it is the (B) timeout problem.

#### (A) ModemManager probes the port and breaks the handshake

Ubuntu's ModemManager treats ttyACM devices as modem candidates and opens the port on every plug
to fire AT commands. This probe breaks the Inverse3/VerseGrip serial handshake so the header never
arrives. Check:

```bash
systemctl is-active ModemManager
udevadm info -q property -n /dev/inverse3_left | grep -i ID_MM
```

If only `ID_MM_CANDIDATE=1` is present and `ID_MM_DEVICE_IGNORE=1` is not, ModemManager is touching
this port. Make sure the rule in [section 5](#5-serial-ports-udev) contains
`ENV{ID_MM_DEVICE_IGNORE}="1"` and apply it:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger --action=add
sudo systemctl restart ModemManager
udevadm info -q property -n /dev/inverse3_left | grep ID_MM_DEVICE_IGNORE   # ID_MM_DEVICE_IGNORE=1
```

> **Important:** `ID_MM_DEVICE_IGNORE` takes effect only from the *next time you plug in*. If
> ModemManager has already grabbed the port right after boot and left it stuck, the commands above
> alone may not resolve it, so **physically unplug and replug the Inverse3/VerseGrip USB once.**
> (If you are in a hurry, you can disable it for this boot only with
> `sudo systemctl stop ModemManager` and test.)

#### (B) On a cold start, wakeup takes longer than the OPEN timeout

A just-powered-on Inverse3 wakes up by having its first `DeviceWakeup` time out once (`timeout
waiting for header code`) and then retrying. Because this cold wakeup is long, if the Python-side
OPEN timeout is short, the device is fine but Python gives up first (it only prints up to `gravity
compensation: disabled` and then times out). That is why `Inverse3Server`'s `open_timeout_s`
defaults to **25 seconds** (`inverse3_bridge/__init__.py`).

This is the case when direct execution (section 7) shows `OK` but only your teleop client times
out. When warm (connected once just before), it connects quickly, so a temporary workaround is to
wake it up once with a direct run and then run your teleop client.

#### (C) If it still does not work (device itself stuck / power)

1. Unplug the Inverse3 USB (and the power too, if it has a separate supply) and reconnect after ~5 seconds.
2. Verify the symlinks are recreated: `ls -l /dev/inverse3_left /dev/versegrip_left`
   (udev recreates them based on the serial, so the mapping is preserved even if the ttyACM number changes.)
3. Switch to a different USB port/cable. The Teensy is sensitive to underpowered USB hubs, so connect it
   directly to a port on the PC itself or to a powered hub.

### When VerseGrip button/rotation is not detected (`OK GRIP_UNAVAILABLE` / `VerseGrip not responding`)

```text
[warn] VerseGrip handle not responding (only invalid frames). Power on / pair the handle ...
OK GRIP_UNAVAILABLE
```

The VerseGrip is a wireless stylus. **If it is powered off**, the Inverse3 connection (i.e. position
tracking) still works, but the button and rotation are not detected, so you cannot perform
calibration/gripper operations. Powering on the stylus and pairing it with the dongle recovers
automatically, and it comes up as `OK` instead of `OK GRIP_UNAVAILABLE`.

### When only the symlinks are missing but `/dev/ttyACM*` exists

Re-apply the udev rule.

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger --action=add
```

### When `/dev/ttyACM*` is also missing

The USB connection itself is down. Connect directly to a powered hub or a USB port on the PC itself, and
use `sudo dmesg -w` to check for power warnings such as `over-current condition`.

### When the port is present but another process is holding it

```bash
lsof /dev/ttyACM*
pgrep -af inverse3_server
```

If a previously half-dead `inverse3_server` or `record.py` is holding the port, kill it and re-run.
