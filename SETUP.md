# Setup Guide

LeRobot-compatible teleoperator for the Haply Inverse3 + VerseGrip stylus.

This repository ships **only the open bridge/teleop code**. The Haply HardwareAPI SDK
(proprietary static library + headers) is **not** included and must be downloaded
separately from Haply, then placed and built locally.

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

## 5. Device serial ports (udev)

The Inverse3 (Teensy) and VerseGrip (USB transceiver) enumerate as `/dev/ttyACM*`, whose
numbers change on reconnect. Pin stable symlinks with the provided udev rule:

```bash
sudo cp udev/99-haply-inverse3.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger --action=add
```

Edit the `serial` values in the rule to match your own devices
(`udevadm info -q property -n /dev/ttyACMx | grep ID_SERIAL`). This gives stable names such
as `/dev/inverse3_left` and `/dev/versegrip_left`.

## 6. Quick device check

```bash
cd src/lerobot_teleoperator_inverse3/inverse3_bridge
printf 'OPEN /dev/inverse3_left /dev/versegrip_left\nGET_STATE\nCLOSE\n' | timeout 30 ./inverse3_server
```

An `OK` (or `OK GRIP_UNAVAILABLE` when the wireless stylus is off) followed by a `STATE`
line means the device is working.
