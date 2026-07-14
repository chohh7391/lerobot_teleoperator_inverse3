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

The Haply SDK must be downloaded, placed, and built before installing. See
**[SETUP.md](SETUP.md)** for the full steps. In short:

```bash
# 1. Put the Haply SDK at inverse3_bridge/sdk (see SETUP.md)
# 2. Build the bridge
cd src/lerobot_teleoperator_inverse3/inverse3_bridge && make && cd -
# 3. Install the package
pip install -e .
```

## Usage

Import the teleoperator in your LeRobot pipeline:

```python
from lerobot_teleoperator_inverse3 import Inverse3Teleop, Inverse3TeleopConfig
```

## License

This repository's own bridge/teleop code is provided as-is. The Haply HardwareAPI SDK is
**not** included and is subject to Haply's own license — obtain it from
<https://develop.haply.co/releases/cpp>.
