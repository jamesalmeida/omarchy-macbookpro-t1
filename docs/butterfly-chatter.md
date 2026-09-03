# Butterfly keyboard chatter on T1 MacBook Pros

2016–2017 T1 machines use Apple’s butterfly keyboard. The switches bounce:
one physical tap produces two key-down events a few milliseconds apart, so
`the` becomes `tthe`. This is hardware. Software can only drop the bounce.

Hyprland `repeat_delay` / `repeat_rate` and libinput debounce do **not**
fix this. Repeat is for a *held* key. libinput’s bounce filter is for mouse
buttons. `kb_options` only remaps modifiers.

## What to grab

Debounce must attach to **Apple SPI Keyboard** only. Confirm the node:

```bash
cat /proc/bus/input/devices | grep -E 'N: Name=|H: Handlers='
```

On a MacBookPro14,2 that is typically `/dev/input/event4`
(`Handlers=sysrq kbd leds event4 tbkbd`).

Do **not** grab:

- `Apple Touch Bar (MBP14,3)` — the strip goes dark
- `Apple Inc. iBridge`

## Install

Needs `python-evdev`. Do not run this at the same time as `keyd` — both grab
the keyboard.

```bash
sudo pacman -S --needed python-evdev
```

```bash
sudo tee /usr/local/sbin/kb-debounce.py <<'EOF'
#!/usr/bin/env python3
from evdev import InputDevice, UInput, list_devices, ecodes

NAME = "Apple SPI Keyboard"
THRESHOLD_MS = 50

def find_dev():
    for path in list_devices():
        d = InputDevice(path)
        if d.name == NAME:
            return d
    raise SystemExit(f"device not found: {NAME}")

dev = find_dev()
dev.grab()
ui = UInput.from_device(dev, name=f"{NAME} (debounced)")
last_up = {}

for ev in dev.read_loop():
    if ev.type == ecodes.EV_KEY:
        if ev.value == 1:
            prev = last_up.get(ev.code)
            now = ev.timestamp()
            if prev is not None and (now - prev) * 1000.0 < THRESHOLD_MS:
                continue
        elif ev.value == 0:
            last_up[ev.code] = ev.timestamp()
    ui.write_event(ev)
    ui.syn()
EOF
sudo chmod +x /usr/local/sbin/kb-debounce.py
```

```bash
sudo tee /etc/systemd/system/kb-debounce.service <<'EOF'
[Unit]
Description=Debounce Apple SPI Keyboard chatter
After=multi-user.target
ConditionPathExists=/usr/local/sbin/kb-debounce.py

[Service]
Type=simple
ExecStart=/usr/bin/python3 /usr/local/sbin/kb-debounce.py
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now kb-debounce.service
systemctl status kb-debounce.service --no-pager
```

Verified on a MacBookPro14,2 (2017, T1) under Omarchy: doubles gone at 50 ms,
Touch Bar unaffected.

## Tuning

`THRESHOLD_MS` is the window after a key-up during which a second down of
the **same** key is dropped.

| Symptom | Change |
| --- | --- |
| still seeing `tthe` / `heello` | raise toward 70 |
| intentional doubles eaten (`apple`) | drop toward 40 |

```bash
sudo nvim /usr/local/sbin/kb-debounce.py   # edit THRESHOLD_MS
sudo systemctl restart kb-debounce.service
```

If the keyboard goes silent:

```bash
sudo systemctl disable --now kb-debounce.service
```

Then log out and back in. The physical device is released on stop; a new
session picks it up again.

## Persistence

`systemctl is-enabled kb-debounce.service` should read `enabled`. The unit
starts after `multi-user.target` on purpose, same reason as
`touchbar.service`: a failure must not hang boot.
