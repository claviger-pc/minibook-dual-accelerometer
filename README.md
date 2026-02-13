# Dual-accelerometer tablet-mode switching for the Chuwi MiniBook X

Adapted for **Debian 13 (Trixie)** from the
[original OpenSUSE Tumbleweed implementation](https://github.com/lyndros/minibook-dual-accelerometer)
by [lyndros](https://github.com/lyndros), itself a fork of
[stackeduary's original work](https://github.com/stackeduary/minibook-dual-accelerometer).
Debian adaptation by Claude (Anthropic).

## Introduction

A number of 2-in-1 convertible laptops today lack a hardware hinge-position
sensor, and instead use a pair of accelerometers to determine the relative
positions of the base and display. Some such laptops do this transparently in
firmware, but others (such as the Chuwi MiniBook X) require software to
interpret the accelerometer outputs and notify the firmware appropriately. In
the Windows 11 installation that the MiniBook X ships with, this appears to
all be handled inside the accelerometer driver (`mxc6655angle.dll`).

This repo implements a 'software angle sensor' for Linux. The kernel exposes
accelerometer data to userspace, and provides a mechanism to notify the
firmware of tablet-mode changes. A userspace service interprets the
accelerometer data and notifies the driver (which in turn notifies the
firmware) of any mode changes.

When the driver triggers a switch into or out of tablet mode, the firmware
disables or enables the keyboard and touchpad, and issues an HID
tablet-mode-switch event, just as would happen if there was a physical
switch. Most desktop environments recognise these events, and enable or
disable screen rotation, and change UI elements as appropriate.

## Status

(see also: [`TODO.md`](TODO.md))

While the driver and angle-sensor service are both currently functional, **I
would consider them to be at the proof-of-concept stage**. Do **not** expect
them to be reliable or stable (or safe) at present.

## Supported hardware

Tested on the Chuwi MiniBook X N150.

## Prerequisites

Install the required packages:

```bash
sudo apt install dkms python3 python3-numpy python3-pyudev linux-headers-$(uname -r)
```

## How to install and use

1. Install the prerequisites listed above.

2. As root, run `make install`, or manually:

    - Build and install the `chuwi-ltsm-hack` kernel module by running
      `dkms install hack-driver` in this directory.

    - Install [`angle-sensor-service/angle-sensor.py`](angle-sensor-service/angle-sensor.py)
      to `/usr/local/sbin/angle-sensor` with execute permissions.

    - Install [`angle-sensor-service/chuwi-tablet-control.sh`](angle-sensor-service/chuwi-tablet-control.sh)
      to `/usr/local/sbin/chuwi-tablet-control` with execute permissions.

    - Install [`angle-sensor-service/angle-sensor.default`](angle-sensor-service/angle-sensor.default)
      to `/etc/default/angle-sensor`.

    - Install [`angle-sensor-service/angle-sensor.service`](angle-sensor-service/angle-sensor.service)
      to `/etc/systemd/system/angle-sensor.service`.

3. Enable and start the service:

    ```bash
    sudo systemctl enable --now angle-sensor.service
    ```

Running `make uninstall` will uninstall everything and (hopefully) bring your
system back to how it was before.

**Note:** After a kernel update, you may need to reinstall the kernel headers
before reinstalling:

```bash
sudo apt install linux-headers-$(uname -r)
```

## Software info

### [Hack driver](hack-driver/)

The `chuwi-ltsm-hack` module simply adds a `chuwi_ltsm_hack` sysfs file to
`/sys/bus/acpi/MDA6655:00`, that triggers the `LTSM` ACPI method to switch
between laptop and tablet modes.

The included udev rule takes care of adding the second accelerometer and
loading the hack driver when the first accelerometer is detected, and a
modprobe configuration file is used to allow the `intel-hid` module to
recognise tablet-mode switch events.

### [`angle-sensor`](angle-sensor-service/angle-sensor.py)

`angle-sensor` is a Python script that polls the accelerometers (and lid
switch), and calls a configurable command (e.g. `chuwi-tablet-control`) to take
action when it determines that a tablet-mode change has occurred, based on
accelerometer and lid-switch inputs.

For convenience, the angle sensor service uses the terms 'hinge axis' and 'tilt
axis' when talking about orientation. When looking at the laptop in normal 'in
your lap' orientation, 'hinge axis' rotation is back and forth (in the axis of
the hinge), and 'tilt axis' is left to right, perpendicular to the hinge.

Its basic logic for determining state is:

- If the lid switch is closed, ignore accelerometers and report `CLOSED` state.

- If the change in acceleration since the last poll exceeds more than a certain
  threshold (configurable by `--jerk-threshold`), do nothing, since erratic
  motion will render hinge angle calculations unreliable.

- If the base acceleration vector is off-horizontal on the tilt axis by more
  than a certain amount (configurable by `--tilt-threshold`), do nothing, since
  hinge angle calculation becomes less reliable as the Z component diminshes
  (like how rotation sensing becomes unreliable the further the device is from
  vertical).

- Otherwise, report `LAPTOP` or `TABLET` state based on the hinge-axis angle
  between the display and base acceleration vectors. The criteria for state
  changes can be tuned with `--threshold` and `--hysteresis`.

An additional `TENT` state is defined, but not detected or reported at present.
This state is intended to represent when the hinge is open further than 180
degrees but not fully folded back into 'tablet' position, allowing the laptop to
stand vertically in a portrait orientation.

### [`chuwi-tablet-control`](angle-sensor-service/chuwi-tablet-control.sh)

`chuwi-tablet-control` is a simple shell script, designed to be called by
`angle-sensor`, to set the tablet state. It takes a single argument, which can
be one of `CLOSED`, `LAPTOP`, `TENT`, or `TABLET`, and writes `0` or `1` to
the appropriate sysfs file.

States are translated to writes as follows:

| State     | Value written | Comment
|-----------|---------------|--------
| `CLOSED`  | `0`           | Safety measure - always enter laptop mode when lid is closed
| `LAPTOP`  | `0`           |
| `TENT`    | `1`           | Not implemented in `angle-sensor` yet
| `TABLET`  | `1`           |

## Hardware info

Both accelerometers are MEMSIC MXC6655 devices, using a thermal sensing element.
The accelerometer inside the display is at address `0x15` on I<sup>2</sup>C bus
1, and the base accelerometer is at the same address on I<sup>2</sup>C bus 0.

The accelerometers are oriented so that in tablet mode, their
vectors should be roughly pointing in the same direction. Since I can't draw
worth a damn, this table is an attempt to explain the orientation of the
accelerometers while in tablet mode.

|    | + direction                                |
|----|--------------------------------------------|
| X  | In plane of screen, towards camera         |
| Y  | In plane of screen, towards fan            |
| Z  | Normal to keyboard (i.e. away from viewer) |

These orientations do NOT match the suggested orientation given in the kernel
IIO documentation, which suggests for a handheld device with the camera at the
top of the screen:

|    | + direction                                           |
|----|-------------------------------------------------------|
| X  | In plane of screen, towards right hand edge of screen |
| Y  | In plane of screen, towards front-facing camera       |
| Z  | Normal to screen, (i.e. towards viewer)               |

[`udev/60-sensor-chuwi.rules`](udev/60-sensor-chuwi.rules) defines
`ACCEL_MOUNT_MATRIX` properties that transform their vectors into the
IIO-recommended orientation while also compensating for the inherent 90
degree rotation of the MiniBook X's physical display. No suggested
orientation exists for a sensor in the keyboard, but using the same
`ACCEL_MOUNT_MATRIX` as the display is convenient, since this maintains the
property of the vectors being equal when in tablet mode. This information
should really be in the hwdb, but hwdb entries are keyed off of `modalias`
values, which in our case is the same for both sensors, so dedicated rules
based on device path and name are necessary here.

Interestingly both accelerometers on my MiniBook X have a significant, but
stable offset (approximately -6 kg/m<sup>2</sup>) in the Z axis. This impacts
the accuracy of the hinge angle measurement, though not unusably so. This has
also been observed on another unit, so it is likely a design quirk rather than a
fault.

