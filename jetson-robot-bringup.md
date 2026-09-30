# Jetson Mobile Manipulator Bring-up

**Date:** 2026-09-29 to 2026-09-30
**Host:** Jetson Orin Nano Developer Kit, user `robot`, hostname `localhost`
**Goal:** Reflash the Jetson, then bring up every peripheral (LiDAR, 3 cameras, SO-101 arms, iRobot Create 2) and control them from one GUI.

---

## 1. Final State

| Component | Status | Notes |
|---|---|---|
| Jetson OS | Working | JetPack 6.2.3 (Jetson Linux 36.5.2, Ubuntu 22.04) on NVMe |
| RPLIDAR A1 | Working, intermittent | Freezes after a few seconds with `Check bit not equal to 1` (see §9) |
| Webcam "Web Camera" | Working | |
| Webcam j5 JVCU100 | Working | Shows up as `GENERAL WEBCAM` in `lsusb` |
| RealSense D435i | Working | Color + depth through `pyrealsense2` |
| SO-101 arm 1 | Working | Recalibrated after a wrist cable came loose |
| SO-101 arm 2 | Not detected | USB board never enumerates (see §9) |
| iRobot Create 2 | Working | Sensors, drive, Safe mode. Speaker on one unit is silent |
| GUI (`robot_gui.py`) | Working | All devices live, buttons for base and arm |

---

## 2. Hardware

### Jetson

- Jetson Orin Nano Developer Kit (8 GB), 256 GB generic NVMe SSD (`SSD NVME 256GB`)
- Power supply was replaced during bring-up. The original adapter made flashing fail with "device not ready" (§3.4).
- Factory QSPI firmware: `36.4.3-gcid-38968081`

### Peripherals and USB IDs

| Device | Chip / USB ID | `/dev/serial/by-id` name | Typical node |
|---|---|---|---|
| RPLIDAR A1 (A1M8) | Silicon Labs CP2102, `10c4:ea60` | `usb-Silicon_Labs_CP2102_USB_to_UART_Bridge_Controller_0001-if00-port0` | `/dev/ttyUSB0` |
| iRobot Create 2 cable | FTDI FT231X, `0403:6015`, serial `DA01NTO5` | `usb-FTDI_FT231X_USB_UART_DA01NTO5-if00-port0` | `/dev/ttyUSB1` |
| SO-101 arm 1 board | QinHeng `1a86:55d3`, serial `5AE6057204` | `usb-1a86_USB_Single_Serial_5AE6057204-if00` | `/dev/ttyACM0` |
| RealSense D435i | `8086:0b3a` | n/a (camera) | `/dev/video2`–`video7` |
| Web Camera | `32e6:9221` | n/a (camera) | first node per camera |
| j5 WebCam JVCU100 | Generalplus `1b3f:2247` | n/a (camera) | first node per camera |

`ttyUSB*`, `ttyACM*` and `video*` numbers change with plug order. Always address serial devices through `/dev/serial/by-id/` or the udev symlinks in §4.3.

### Device facts learned

- **RPLIDAR A1:** model id `0x18`, firmware 1.29, hardware 7, 115200 baud.
- **SO-101 arm:** Feetech STS3215 motors (model 777), IDs 1–6 daisy-chained from base to gripper, 12 V supply (measured 12.3–12.4 V).
- **Create 2:** Open Interface at 115200 8N1. The iRobot cable wires BRC to FTDI RTS, so the robot can be woken from software.

### USB topology

The Orin Nano devkit has 4 USB-A ports and 1 USB-C. With 9 USB devices (3 cameras, 2 arms, LiDAR, Create 2 cable, keyboard, mouse) a hub is required. Current setup uses a USB-C dock plus an external hub. Put the RealSense on a blue USB 3 port directly on the Jetson, and use a hub with its own power adapter for the rest.

---

## 3. Flashing the Jetson

### 3.1 What failed and why

| Attempt | Result | Cause |
|---|---|---|
| JetPack 7.2 Jetson ISO (r39.2.0) from USB stick | Installs, then boots to a blinking `_` cursor | QSPI firmware stayed at 36.4.3. The ISO's capsule update silently fails on this firmware version, so the r39 system boots with mismatched firmware (no display, no USB gadget, no oem-config). |
| SDK Manager on Windows via WSL2 | Several failures (§3.4) | Orin Nano isn't officially supported for Windows flashing. Final blocker was `Port Reset Failed` when the board rebooted into the initrd flash kernel. |
| **SDK Manager on native Ubuntu x86** | **Success** | Writes QSPI directly over USB, so the capsule bug doesn't apply. |

The earlier SSD health check (from the ISO's rescue shell) showed the NVMe was fine: detected at 238.5 GB, readable at 1.2 GB/s, ext4 mounts cleanly. The problem was never the SSD.

### 3.2 Recovery mode (Orin Nano devkit)

1. Unplug Jetson power.
2. Short **FC REC** to **GND** on the 12-pin button header **J14** (pins 9 and 10), located under the edge of the module. A jumper, dupont wire or tweezers all work. Don't touch the 40-pin GPIO header.
3. Connect the Jetson USB-C port to the host with a data-capable cable.
4. Plug in power. The screen stays black, which is expected.
5. On the host: `lsusb | grep -i nvidia` should show `0955:7523 NVIDIA Corp. APX`.
6. The short can be removed once the board is in recovery.

### 3.3 Flash with SDK Manager (Ubuntu host, the working path)

```bash
sudo apt install ./sdkmanager_*_amd64.deb
echo -1 | sudo tee /sys/module/usbcore/parameters/autosuspend   # avoid USB autosuspend during flash
sdkmanager
```

1. Target: Jetson Orin Nano [8GB developer kit version]. JetPack **6.2.3**.
2. Uncheck Host Machine. Jetson Linux must stay checked.
3. After download, the flash dialog appears: **Manual Setup**, **Pre-Config** (username/password), Storage Device **NVMe**.
4. Flashing takes 20–45 minutes and the board reboots several times. That's normal.
5. Power off, remove the recovery short, power on. Login screen appears.

### 3.4 Windows / WSL attempt (for reference)

| Error | Fix |
|---|---|
| `HCS_E_SERVICE_NOT_AVAILABLE` creating the WSL instance | Enable virtualization in BIOS, then `dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart` and `.../featurename:Microsoft-Windows-Subsystem-Linux ...`, reboot |
| `usbipd` service error 1067 | Reboot after `usbipd-win` install. Reinstall with `winget install --exact dorssel.usbipd-win` if it persists |
| Device "not shared" / "no WSL 2 distribution running" | `usbipd bind --busid X-Y --force`, keep `wsl -d Nvidia_SDKM_Ubuntu_22.04_JetPack_6.2.3` open, then `usbipd attach --wsl --busid X-Y --auto-attach` |
| "Jetson device is not ready for flash" | Fixed by replacing the Jetson power supply |
| `Device failed to boot to the initrd flash kernel`, `Port Reset Failed` in `usbipd list` | Laptop USB couldn't hold the initrd USB device. Not fixed on Windows. Moved to native Ubuntu. |

### 3.5 After first boot

```bash
sudo apt update && sudo apt upgrade
sudo apt install nvidia-jetpack          # CUDA, cuDNN, TensorRT if SDK Manager skipped them
cat /etc/nv_tegra_release                # expect R36, REVISION 5.x
df -h /                                  # expect /dev/nvme0n1p1
```

---

## 4. Software Setup

### 4.1 Python environment

LeRobot needs Python ≥ 3.12, JetPack 6 ships 3.10, so everything runs in a Miniforge env.

```bash
wget https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-aarch64.sh
bash Miniforge3-Linux-aarch64.sh
source ~/.bashrc
conda create -y -n lerobot python=3.12
conda activate lerobot

git clone https://github.com/huggingface/lerobot.git
cd lerobot && pip install -e ".[feetech]"

pip install pyserial rplidar-roboticia matplotlib pillow pyrealsense2
pip uninstall -y opencv-python-headless opencv-python && pip install opencv-python
```

LeRobot pulls in `opencv-python-headless`, which has no GUI (`cv2.imshow` fails with "The function is not implemented"). Reinstall `opencv-python` whenever LeRobot is reinstalled.

### 4.2 Permissions and services

```bash
sudo usermod -aG dialout $USER            # log out and back in afterwards
sudo systemctl disable --now ModemManager # stops it from grabbing /dev/ttyACM*
```

### 4.3 udev rules

Stable names for the serial devices (`/etc/udev/rules.d/99-robot.rules`):

```
SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{serial}=="DA01NTO5", SYMLINK+="create2", MODE="0666"
SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", SYMLINK+="lidar", MODE="0666"
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{serial}=="5AE6057204", SYMLINK+="arm1", MODE="0666"
```

RealSense access for `pyrealsense2` (needed if it reports 0 devices without `sudo`):

```bash
wget -O /tmp/99-realsense-libusb.rules https://raw.githubusercontent.com/IntelRealSense/librealsense/master/config/99-realsense-libusb.rules
sudo cp /tmp/99-realsense-libusb.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules && sudo udevadm trigger
```

Add an `arm2` line with the second board's serial once it enumerates.

---

## 5. Device Bring-up

### 5.1 LiDAR (RPLIDAR A1)

| Script | Purpose |
|---|---|
| `~/cek_lidar.py` | Probes the port for LDROBOT (230400 stream) or RPLIDAR (`GET_INFO` at 115200/256000/460800/1000000) and reports model, firmware, serial |
| `~/scan_lidar.py` | Text output per scan: point count, nearest point and angle, front distance |
| `~/lidar_gui.py` | Live matplotlib polar plot |
| `~/lidar_dump.py` | Merges 3 rotations, prints every point, a 30° sector summary, saves `~/lidar_scan.csv`, shows a plot |

Angles are clockwise with 0° at the LiDAR front. Only about 30–60 valid points per scan were seen in the lab, fewer than the A1 normally returns. Check for nearby occlusion.

### 5.2 Cameras

```bash
sudo apt install -y v4l-utils
v4l2-ctl --list-devices
python -c "import pyrealsense2 as rs; print(len(rs.context().query_devices()))"   # expect 1
```

| Script | Purpose |
|---|---|
| `~/cam_view.py N` | Show one camera by index |
| `~/cam_all.py` | Open every `/dev/video*` that returns a frame and tile them |
| `~/cams.py` | Webcams via OpenCV (MJPEG), RealSense via `pyrealsense2` with color + depth colormap |

Each webcam exposes two nodes (the second is metadata). The RealSense exposes six. Force MJPEG on webcams (`cv2.VideoWriter_fourcc(*"MJPG")`) to fit three cameras on shared USB bandwidth.

### 5.3 SO-101 arm

```bash
lerobot-find-port
lerobot-calibrate --robot.type=so101_follower \
  --robot.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5AE6057204-if00 \
  --robot.id=my_follower
python ~/tes1.py /dev/serial/by-id/usb-1a86_USB_Single_Serial_5AE6057204-if00 my_follower
```

- Calibration files: `~/.cache/huggingface/lerobot/calibration/robots/so_follower/<id>.json`. `my_follower` is arm 1, `my_follower_2` is reserved for arm 2.
- During calibration, move every joint end to end **including the gripper fully open and closed**. The first calibration recorded a gripper range of only 8 ticks.
- `wrist_roll` doesn't appear in the calibration table because LeRobot treats it as full-turn. That's expected.
- In this LeRobot version the module is `lerobot.robots.so_follower` (classes `SO101Follower`, `SO101FollowerConfig`).
- Normalized positions should stay within −100 to 100. Values like −155 mean the calibration no longer matches the motors.

`~/tes1.py PORT ID` pings all motor IDs, then moves `shoulder_pan` ±5 and back.

Motor health check (torque off, no motion) reads position, load, voltage, temperature and status flags per motor through `FeetechMotorsBus` with `Present_Position`, `Present_Load`, `Present_Voltage`, `Present_Temperature` and `Status`. Healthy readings: ~12.3 V, ~30 °C, status OK.

### 5.4 iRobot Create 2

| Script | Purpose |
|---|---|
| `~/create2_cek.py PORT` | Read-only: OI mode, voltage, current, charge, charging state, bumpers. **No wake step.** |
| `~/create2_sound.py PORT` | Safe mode, play a 4-note song |
| `~/create2_diag.py PORT` | Checks mode after each command, turns the power LED red, reports song-playing flag |
| `~/create2_test.py PORT` | Forward, spin, back at 100 mm/s in Safe mode |
| `~/create2_sensors.py PORT` | Live terminal dashboard of nearly every sensor packet |

Wake the robot from software by pulsing RTS (BRC):

```python
s.rts = True; time.sleep(0.3); s.rts = False; time.sleep(0.5)
s.write(bytes([128]))   # START
```

OI notes confirmed today:

- Safe mode refuses to drive while on the dock. Lift it off first (packet 34 = charging source).
- On the dock, song playback needs Full mode (`132`).
- Bumper packet `00000011` while docked is the dock pressing the bumper.
- Entering the OI pauses charging. Send `173` (Stop) or press Clean to hand control back.
- 19200 baud isn't needed. The robots answer correctly at 115200.

---

## 6. Combined Tests

| Script | Purpose |
|---|---|
| `~/system_test.py` | Staged test with Enter/skip prompts: LiDAR → cameras (snapshots to `~/system_test/`) → arms (ping + move) → Create 2 (safety checks + move). Prints a PASS/FAIL summary. |
| `~/gerak2.py` | Two arms, `shoulder_pan` in opposite directions |
| `~/move_all.py` | Base pattern plus mirrored arm wave for 12 s, arms ease back home at the end |
| `~/dashboard.py` | OpenCV dashboard with keyboard control (W/A/S/D, 1/2, J/K, =/-, M for demo, Q) |

---

## 7. GUI: `robot_gui.py`

### Start

```bash
~/start_robot.sh
```

`start_robot.sh` activates the env, lists serial devices by role, lists camera nodes, reports the `pyrealsense2` device count, opens port permissions, waits for Enter, then launches the GUI.

```bash
#!/bin/bash
source ~/miniforge3/etc/profile.d/conda.sh
conda activate lerobot
for l in /dev/serial/by-id/*; do
  n=$(basename "$l"); t=$(readlink -f "$l")
  case "$n" in
    *FTDI_FT231X*) d="Create 2" ;; *CP2102*) d="LiDAR" ;; *1a86*) d="Lengan" ;; *) d="?" ;;
  esac
  printf "  %-14s %s\n" "$t" "$d"
done
for v in /sys/class/video4linux/video*; do printf "  /dev/%-8s %s\n" "$(basename $v)" "$(cat $v/name)"; done
python -c "import pyrealsense2 as rs; print('  RealSense:', len(rs.context().query_devices()))"
sudo chmod 666 /dev/ttyUSB* /dev/ttyACM* 2>/dev/null
read -p "Enter to open the GUI..."
python ~/robot_gui.py
```

### Layout

- **Top row:** up to 4 live camera panels (2 webcams, RealSense color, RealSense depth).
- **Bottom left:** live LiDAR map, 1 m rings, orange line = front.
- **Bottom middle:** Create 2 sensors, hold-to-drive arrow buttons, red STOP, speed slider (50–300 mm/s), demo button.
- **Bottom right:** per-arm, per-joint − / + hold buttons with current → target readout, Home per arm and Home all.

### Architecture

One thread per device (LiDAR, Create 2, arms, each camera, RealSense, demo). Threads write into a shared state dict under a lock. Tkinter reads it every 50 ms. A missing device shows "not found" in its panel without affecting the others.

### Safety behavior

- Drive buttons are hold-to-move. The Create 2 thread stops the base if no fresh command arrives within 0.4 s, so a frozen GUI stops the robot.
- Forward motion is blocked while a bumper is pressed. Safe mode stops the base on cliff or wheel drop.
- Space bar is an emergency stop for base, demo and arm jog.
- Closing the window returns Create 2 to Passive and disconnects the arms and LiDAR.
- Arms hold position with torque on for the whole session.

### Patches applied after the first version

1. RealSense is only opened through `pyrealsense2` if `rs.context().query_devices()` finds one. Otherwise it falls back to OpenCV nodes (color only).
2. Create 2 thread re-wakes the robot (RTS pulse + START + SAFE) after about 1 s without data, instead of waking only once at startup.

### Arm mapping

`KNOWN_ARMS = {"5AE6057204": "my_follower"}`. Any other `1a86` board gets `my_follower_2`. If a calibration file doesn't match, LeRobot asks in the **terminal**, not the GUI, and the arm panel stays at "connecting" until Enter is pressed there.

---

## 8. Troubleshooting Reference

| Symptom | Cause | Fix |
|---|---|---|
| Blinking `_` after JetPack 7.2 ISO install | QSPI firmware stuck at 36.4.3, capsule update failed | Flash through SDK Manager on Ubuntu, or update firmware to 36.4.4+ first |
| `Could not connect on port '/dev/ttyACM0'` | User not in `dialout` | `usermod -aG dialout`, re-login, or `chmod 666` temporarily |
| `No module named 'serial'` | Running in `(base)` instead of `(lerobot)` | `conda activate lerobot` |
| `cv2.imshow` "not implemented" | Headless OpenCV | Reinstall `opencv-python` (§4.1) |
| `Missing motor IDs: 4, 5, 6` | Daisy-chain cable between motor 3 and 4 loose | Power off, reseat both ends of the elbow–wrist cable |
| `There is no status packet` on one motor | Loose cable or supply sagging under torque | Reseat cable, one adapter per arm |
| `Overload error` on motor 4 | Calibration no longer matched after the cable fault, arm driven into a wrong target | Power-cycle the arm (clears overload), recalibrate |
| Gripper barely moves | Gripper range recorded as 8 ticks | Recalibrate with full open/close |
| `pyrealsense2` "No device connected" | Camera not enumerated, or missing Intel udev rule | Check `lsusb` for `8086`, install udev rule (§4.3) |
| All cameras and arms missing after reboot | External hub not powered / not reconnected | Replug hub, verify with the USB monitor script, use a powered hub |
| Create 2 "no data" | Robot asleep or cable not plugged in | Plug in the Mini-DIN, RTS wake pulse, or press Clean |
| Create 2 silent while `song playing = 1` | Speaker/amp fault on that unit (or no battery) | Hardware. Doesn't affect drive or sensors |
| LiDAR `Check bit not equal to 1`, map freezes | Corrupted serial frame, likely power or bandwidth on the shared hub. GUI thread exits on first error | Plug LiDAR directly into the Jetson. Add auto-reconnect to `lidar_thread` |

**USB monitor script** used to find the missing devices: polls `lsusb` every second and prints `+ MUNCUL` / `- HILANG` plus relevant `dmesg` errors while devices are plugged in one at a time.

---

## 9. Open Items

- [ ] **Arm 2:** its `1a86` board never appears in `lsusb`. Try another USB cable (data-capable), another port, and check the board's LED. Once it enumerates: calibrate as `my_follower_2`, add an `arm2` udev line.
- [ ] **LiDAR stability:** move to a direct Jetson port and add an auto-reconnect loop to `lidar_thread` so one bad frame doesn't kill the stream.
- [ ] **LiDAR point count:** investigate the low valid-return count (occlusion near the sensor, mounting height).
- [ ] **Create 2 speaker:** one unit plays silently. Check whether it's the same robot as "Robot #1" from the earlier Create 2 log.
- [ ] **ROS 2 Humble:** install, then `create_robot` / `libcreate` driver and `rplidar_ros`.
- [ ] **LeRobot data collection:** decide camera placement and names, check `torch.cuda.is_available()` before any inference work.
- [ ] **Power for mobile use:** separate battery for the Jetson and hub, or a wide-input buck from the Create 2 pack. Don't power the Jetson from the Mini-DIN Vpwr pins.

---

## 10. File Index (on the Jetson, `~`)

| File | Section |
|---|---|
| `cek_lidar.py`, `scan_lidar.py`, `lidar_gui.py`, `lidar_dump.py` | §5.1 |
| `cam_view.py`, `cam_all.py`, `cams.py` | §5.2 |
| `gerak.py`, `gerak2.py`, `tes1.py` | §5.3, §6 |
| `create2_cek.py`, `create2_sound.py`, `create2_diag.py`, `create2_test.py`, `create2_sensors.py` | §5.4 |
| `system_test.py`, `move_all.py`, `dashboard.py` | §6 |
| `robot_gui.py`, `start_robot.sh` | §7 |
| `lidar_scan.csv`, `system_test/` | Output data |
