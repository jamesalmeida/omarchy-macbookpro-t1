# Wi-Fi dead after lid-close (BCM43602)

On a T1 MacBook Pro the Broadcom `brcmfmac` firmware often comes back
deaf after `s2idle`. Symptoms:

- Before lid-close: `connected:full`, `wlp2s0` UP, associated, real Apple MAC
- After lid-open: `disconnected:none`, `WIFI` and `WIFI-HW` still enabled,
  `wlp2s0` is `NO-CARRIER` / DOWN, `iw dev wlp2s0 link` says Not connected
- `nmcli radio wifi off/on` and `systemctl restart NetworkManager` do **not**
  fix it. `nmcli connection up` fails with "Wi-Fi network could not be found"
- A reboot always fixes it

The NVRAM file is fine. The chip is wedged. NetworkManager cannot reload
`brcmfmac` while it holds the interface — same constraint as Part 4.

`lsmod` after a failed wake looks like this:

```
brcmfmac_wcc   ...  0
brcmfmac       ...  1 brcmfmac_wcc
brcmutil       ...  1 brcmfmac
```

Unload **`brcmfmac_wcc` first** or `modprobe -r brcmfmac` returns
"Module brcmfmac is in use."

## Manual recovery (no reboot)

```bash
sudo systemctl stop NetworkManager
sudo killall wpa_supplicant
sudo modprobe -r brcmfmac_wcc
sudo modprobe -r brcmfmac brcmutil
sudo modprobe brcmfmac
sudo systemctl start NetworkManager
sleep 4
nmcli radio wifi on
nmcli connection up "<SSID>"
```

The Omarchy menu bar (quickshell) can stay on "Disconnected" after this.
Restart the panel, or log out and back in. Do not toggle Wi-Fi in the menu
— that drops the connection you just recovered.

## Do not put this in `/usr/lib/systemd/system-sleep/`

A `system-sleep` script that calls `systemctl stop NetworkManager` runs
*during* resume and can deadlock. The machine will not wake; recovery is
a power-button hold. Remove that file if you already added one:

```bash
sudo rm -f /usr/lib/systemd/system-sleep/wifi-resume
```

## Automatic fix (after resume finishes)

Hook `ExecStartPost` on `systemd-suspend.service`. That runs after
`systemd-sleep suspend` returns, i.e. the machine is already awake.

```bash
sudo tee /usr/local/sbin/wifi-after-suspend.sh <<'EOF'
#!/bin/sh
sleep 3
systemctl stop NetworkManager
killall wpa_supplicant 2>/dev/null
modprobe -r brcmfmac_wcc 2>/dev/null
modprobe -r brcmfmac brcmutil 2>/dev/null
modprobe brcmfmac
systemctl start NetworkManager
EOF
sudo chmod +x /usr/local/sbin/wifi-after-suspend.sh

sudo mkdir -p /etc/systemd/system/systemd-suspend.service.d
sudo tee /etc/systemd/system/systemd-suspend.service.d/wifi.conf <<'EOF'
[Service]
ExecStartPost=/usr/local/sbin/wifi-after-suspend.sh
EOF
sudo systemctl daemon-reload
```

Confirm the drop-in is visible:

```bash
systemctl cat systemd-suspend.service | grep ExecStartPost
```

Verified on a MacBookPro14,2 under Omarchy: lid-close wake works, Wi-Fi
reassociates without a reboot.
