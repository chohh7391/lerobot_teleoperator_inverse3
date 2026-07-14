# lerobot_teleoperator_inverse3

A [LeRobot](https://github.com/huggingface/lerobot)-compatible teleoperator for the
**Haply Inverse3** haptic device with the **VerseGrip** wireless stylus.

It provides a Python `Teleop` implementation backed by a small C++ bridge server that talks
to the Haply HardwareAPI, so the device can drive LeRobot robots and record demonstrations.

## Requirements

- Python ≥ 3.10, `lerobot==0.4.4`, `numpy`, `scipy`
- A C++17 compiler (`g++`) to build the bridge
- The **Haply HardwareAPI SDK** (proprietary) — downloaded separately, **not** bundled here

## Install

The Haply SDK is proprietary and **not** bundled here. Download it, place it, build the
bridge, then install the package.

### 1. Download the Haply HardwareAPI SDK

Linux C++ SDK (v0.2.8), from the [Haply releases page](https://develop.haply.co/releases/cpp):

```bash
cd src/lerobot_teleoperator_inverse3/inverse3_bridge
curl -L -o HardwareAPI.0.2.8.linux.tar.gz \
  https://cdn.haply.co/r/33857671/HardwareAPI.0.2.8.linux.tar.gz
```

### 2. Extract and rename the folder to `sdk`

The extracted folder must sit at `inverse3_bridge/sdk` so that
`sdk/include/*.h` and `sdk/libHaply.HardwareAPI.a` exist (the `Makefile` looks for these):

```bash
tar -xzf HardwareAPI.0.2.8.linux.tar.gz
mv HardwareAPI.0.2.8.linux sdk   # extracted folder name may vary — rename it to exactly "sdk"
rm HardwareAPI.0.2.8.linux.tar.gz
```

Expected layout:

```text
inverse3_bridge/sdk/
├── include/                 # Inverse3.h, Handle.h, SerialStream.h, HardwareAPI.h, ...
└── libHaply.HardwareAPI.a   # this exact filename
```

### 3. Build the C++ bridge (produces the `inverse3_server` binary)

```bash
make        # run inside inverse3_bridge/
cd -
```

### 4. Install the Python package

```bash
pip install -e .
```

For device serial-port (udev) setup, a quick device check, and troubleshooting, see
**[SETUP.md](SETUP.md)**.

## Usage

Import the teleoperator in your LeRobot pipeline:

```python
from lerobot_teleoperator_inverse3 import Inverse3Teleop, Inverse3TeleopConfig
```

## License

This repository's own bridge/teleop code is provided as-is. The Haply HardwareAPI SDK is
**not** included and is subject to Haply's own license — obtain it from
<https://develop.haply.co/releases/cpp>.
