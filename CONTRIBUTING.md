# Contributing

The most useful contribution is support for a HyperX device you own. This guide explains how to
find out what your device needs, how to record what NGENUITY sends to it, and where in the code a
new device goes.

There are three cases, from least to most work:

1. **Your device is in the model list but untested** (`ngenuity devices --all` lists it). Follow
   [Testing untested hardware](README.md#testing-untested-hardware) and report the results in an
   issue. The USB product ID alone is already a useful contribution.
2. **Your device is not in the list but uses the universal protocol.** You add a model entry and a
   device page, and may need captures for features the tool does not support yet.
3. **Your device uses another protocol** (most headsets and microphones, the Clutch Tachi, older
   Kingston-era HyperX devices). This means reverse engineering a new protocol; open an issue with
   your findings first.

## Safety rules

These apply to everything below, both to manual tests and to code you write.

- **Read before you write.** Read-only queries (`10 01`, `52 01`, `50 02`, the `xx 02`/`xx 03`
  information requests) are safe. Note your current settings before you change anything, and
  restore them afterwards.
- **Test on the live profile first.** Changes without `--save` go to profile slot 0 and disappear
  when the device is power-cycled. Write the onboard profile (slot 1, `--save`) only after the live
  change works.
- **Never send `12 01`** (factory reset), and never send firmware-update reports (`F0`, `FA`, `FC`).
- **Never commit NGENUITY's files** or anything derived from its binaries: the installer, the
  extracted app, decompiled sources, Ghidra projects. They are ignored by git (`.gitignore`); keep
  it that way. Write down what you learned in your own words in `docs/` instead.

## 1. Identify your device

Find the USB vendor and product ID:

```
lsusb | grep -i -e hyperx -e 03f0 -e 0951
```

- Vendor `03f0` (HP, current HyperX devices): most likely the universal protocol. Continue below.
- Vendor `0951` (Kingston, older HyperX devices): an older protocol that ngenuity-linux-cli does
  not implement (case 3).

Then check whether the device has a vendor HID interface (report descriptor starting with
`06 00 ff`, usage page `0xFF00`), which is what `ngenuity` talks to:

```
for d in /sys/bus/hid/devices/*:03F0:*; do
  echo "$d $(head -c 3 "$d/report_descriptor" | xxd -p) $(ls "$d/hidraw" 2>/dev/null)"
done
```

Give yourself access to that hidraw node (or use `sudo` for now): copy
`70-hyperx-pulsefire-haste2.rules`, change `ID_USB_MODEL_ID` to your product ID and
`ID_USB_INTERFACE_NUM` to the vendor interface, and install it as described in the
[README](README.md#installation).

Finally ask the device to describe itself. This only sends read-only queries:

```
uv run ngenuity -v devices
```

If the device answers `10 01` with an `11 01` reply, it speaks the universal protocol: the reply
contains its category (mouse, keyboard, …) and USB IDs
([format](docs/devices/pulsefire-haste-2-wireless.md#device-information-reply)). A wireless device
must be awake for this: move it and run the command again. Keep the `-v` output; it goes into your
issue or pull request.

If there is no reply, try `uv run ngenuity -v status`. If nothing answers on any vendor interface,
the device uses another protocol (case 3).

## 2. Look up what NGENUITY knows about it

NGENUITY has a configuration class per device that lists its capabilities. You need your own copy
of NGENUITY to read it:

1. Download the NGENUITY installer from HyperX and unpack the application files (they are .NET
   assemblies, for example `NGENUITY.dll` and `NGENUITY.Common.dll`).
2. Decompile them with [ILSpy](https://github.com/icsharpcode/ILSpy)'s command-line tool:
   ```
   dotnet tool install -g ilspycmd
   ilspycmd -p -o decompiled/NGENUITY extracted/current/NGENUITY.dll
   ilspycmd -p -o decompiled/NGENUITY.Common extracted/current/NGENUITY.Common.dll
   ```
   `extracted/` and `decompiled/` are git-ignored.
3. Open `decompiled/NGENUITY/HXN.Configurations.<Type>/<Name>Configuration.cs`
   (`Mouses`, `Keyboards`, `Headsets`, `Microphones`, `Gamepads`). The interfaces it implements
   show which feature groups it has (`ILightConfiguration`, `IKeyConfiguration`,
   `ISensorConfiguration`, `IWirelessMouseSettingConfiguration`, …), and its members give the
   details:

   | NGENUITY member | What it tells you |
   |---|---|
   | `GetSupportedPollingRates` | polling rates |
   | `DpiStep`, `DpiIntervals` (defaults in `HXN.Configurations/ISensorConfiguration.cs`) | DPI step and range |
   | `IsLODSupported`, `IsMotionSyncSupported`, `IsAngleSnapSupported` | sensor options |
   | `GetKeyLayout` | the remappable buttons |
   | `GetSupportedKeyActionTypes` | button action types (Keys & Buttons, Macro, …) |
   | `GetLightKeyLayout` | the LEDs; their number is the LED count (none: no RGB) |
   | `GetSupportSWEffects`, `GetSupportHWEffects` | software and hardware lighting effects |

The packet formats behind these features are in the
[universal protocol specification](docs/protocol/universal.md). If your device only needs
features that are already implemented there (DPI, polling rate, lift-off, buttons, brightness,
lighting, battery), the model entry in step 4 is all the code you need, and you can skip the
captures.

## 3. Capture NGENUITY's USB traffic

Captures are needed for anything the protocol document does not cover yet: a new feature, a
different payload layout, or a device that behaves differently from the Haste 2 Wireless. The
method is to run NGENUITY in a Windows virtual machine with the device passed through, and record
the USB bus on the Linux host.

### Setup

1. Install VirtualBox with a Windows guest, add your user to the `vboxusers` group, and install
   NGENUITY in the guest.
2. Install Wireshark (it provides `dumpcap` and `tshark`) and let the `wireshark` group capture USB
   traffic without root:
   ```
   sudo usermod -aG wireshark "$USER"
   sudo cp 50-usbmon-wireshark.rules /etc/udev/rules.d/
   sudo udevadm control --reload
   sudo modprobe usbmon
   ```
   Log out and in again for the group change.
3. Find the bus and device number of your device (`lsusb` prints `Bus 003 Device 007: ID 03f0:…`).
   The capture interface is `usbmon<bus>`, here `usbmon3`. The device number changes when the
   device is replugged or passed to the VM, so check it again before each capture.

### Recording

1. Start the capture on the host:
   ```
   dumpcap -i usbmon3 -w captures/unfiltered/mydevice_raw.pcapng
   ```
2. Pass the device through to the VM (*Devices → USB*) and start NGENUITY.
3. Wait until NGENUITY has read the device, then **change exactly one setting**, and write down
   the time and what you changed, for example "12:03:10 polling 1000 → 250". Wait a few seconds,
   change it back, note the time again. Press *Save* only if the capture is about saving.
4. Stop `dumpcap` with Ctrl+C. Do one capture per feature.

### Filter the capture before sharing it

**A usbmon capture records the whole bus, including every key you type on your keyboard.** Before
you commit or attach a capture, keep only your device's traffic. With device number 7 from
`lsusb` (as seen during the capture):

```
tshark -r captures/unfiltered/mydevice_raw.pcapng -Y "usb.device_address==7" \
  -w captures/<device>_<NN>_<content>.pcapng
```

`captures/unfiltered/` is git-ignored. Name the filtered file after the device page and what it
records, e.g. `pulsefire-saga-pro_01_startup-polling-1000-250-1000.pcapng`. Open it in Wireshark
and check that it contains nothing but your device.

### Reading a capture

Every universal-protocol packet is 64 bytes: the report ID is the command, byte 1 the sub-command,
and the device answers with report ID command + 1 ([§2](docs/protocol/universal.md)). To list the
packets with their time, endpoint and data:

```
tshark -r captures/<file>.pcapng -Y "usb.data_len == 64" \
  -T fields -e frame.time_relative -e usb.endpoint_address -e usb.capdata
```

Packets to an OUT endpoint (`0x04` on the Haste 2 Wireless) were sent by NGENUITY; packets from an
IN endpoint (`0x83`, bit 7 set) are the device's replies, ACKs (`ff …`) and events.

Match the packets around your noted times to the commands in the
[command map](docs/protocol/universal.md), and compare the payload before and after your change to
find the bytes that hold the setting.

## 4. Add the device to the code

The code is a [uv](https://docs.astral.sh/uv/) project in `src/ngenuity_linux_cli/`:

| Module | What it contains | When you change it |
|---|---|---|
| `devices.py` | `MODELS`, the registry of known devices, and interface discovery | always: one `Model` entry per device |
| `protocol.py` | packet encoders and reply parsers of the universal protocol | a new command or a different payload layout |
| `device.py` | `UniversalDevice`, the hidraw transport (send, `await_reply`, events) | rarely |
| `session.py` | detection and the `-d` selection | rarely |
| `cli.py` | the click commands | a new command, or a new capability to check |
| `buttons.py`, `lighting.py`, `colors.py` | button actions, light-engine effects, colour names | new button actions or effects |

### The model entry

Add a `Model` to `MODELS` in `devices.py`, using the capabilities from step 2:

```python
Model(
    "pulsefire-example",
    "Pulsefire Example",
    Category.MOUSE,
    product_ids=(0x0F99,),
    polling_rates=RATES_8K,
    dpi_min=100,
    dpi_max=16000,
    dpi_step=100,
    buttons=SAGA_BUTTONS,
    leds=0,
    effects=(),
),
```

- `key` is the name used with `-d` and the file name of the device page.
- `product_ids` are the USB product IDs that detection matches: the one `lsusb` shows, and the IDs
  in the `11 01` reply (`ngenuity -v devices`). A wireless device with a dongle can have several.
  If a model is already in the list without product IDs, just add them.
- Leave out the fields whose default fits (see the `Model` dataclass), and reuse the existing
  tuples (`RATES_1K`, `RATES_4K`, `RATES_8K`, `MOUSE_BUTTONS`, `SAGA_BUTTONS`, `ALL_EFFECTS`,
  `NO_SPECTRUM`).
- `leds=0` with `effects=()` disables the lighting commands for devices without RGB.
- Set `tested=True` only after the device passed the tests in the README.
- A device of another category (keyboard, gamepad) is detected already but marked
  `configurable=False` until its commands are implemented.

### New commands

If the device needs a command that is not implemented yet:

- Put the encoder and the reply parser into `protocol.py`, next to the existing ones, with named
  constants for command bytes and offsets.
- Follow the pattern of the existing setting commands in `cli.py`: read the current value from the
  device, change only what the user asked for, write it, read it back and warn if it differs.
- When the device answers a command with a `cmd + 1` reply, wait for it with `await_reply` before
  sending the next packet. Wireless devices drop packets otherwise.
- If only some models support the command, add a capability field to `Model` and check it in
  `cli.py` like `has_lighting`.

### Documentation

- Add `docs/devices/<key>.md`, modelled on
  [pulsefire-haste-2-wireless.md](docs/devices/pulsefire-haste-2-wireless.md): USB IDs and
  interface, capabilities, the supported commands with their test status, and a list of your
  captures with what each one records.
- Add the device to the table in [docs/devices/README.md](docs/devices/README.md).
- Add anything new you decoded to [docs/protocol/universal.md](docs/protocol/universal.md), and
  mark what you confirmed on the device with ✔.
- Update the status tables in the [README](README.md) if a missing feature is now done.

## 5. Check and test

All three checks must pass:

```
uv run ruff check
uv run ruff format --check
uv run mypy
```

The code style: fully typed (mypy strict), no comments or docstrings except the docstrings of
click commands (they are the `--help` text), named constants instead of magic numbers, and no
`noqa`.

Then test on your device with `-v`, from read-only commands to live changes to `--save`, as in
[Testing untested hardware](README.md#testing-untested-hardware), and restore your settings
afterwards.

## 6. Open a pull request

Include:

- the device name and the output of `ngenuity -v devices`;
- which commands you tested, and the `-v` output of any that failed;
- the filtered captures, if you made any, and what each one records.

Write commit messages as one short summary line, a blank line, then a bullet list of the changes
per file.
