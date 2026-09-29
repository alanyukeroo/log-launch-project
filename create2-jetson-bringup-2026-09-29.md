# iRobot Create 2 / Roomba 600 ↔ Jetson Bring-up Log

**Date:** 2026-09-29
**Host:** Jetson (Yahboom image, hostname `yahboom`)
**Goal:** Talk to iRobot Create 2 / Roomba 600 bases over the Open Interface (OI) serial protocol from the Jetson, then verify battery, sound, sensors and motors.

---

## 1. Summary

Three robots were tested with the same USB-to-serial cable and scripts.

| Robot | Identifies as | Battery | OI response | Sound | Status |
|---|---|---|---|---|---|
| #1 | `Roomba by iRobot!`, fw `r3-robot/tags/release-stm32-3.7.5` (2016-06-13) | Removed | Yes | Song "plays" (flag = 1) but silent | Board works on dock power only. Retest sound/motors with a battery. |
| #2 | `iRobotBoot V1`, `Roomba 600-STM32` | Installed, state unknown | **No** (only `0xFE` bytes) | Not tested | Unresolved. Suspect reboot loop from low battery or stuck in bootloader. |
| #3 | Roomba (charging log output) | Installed, deeply discharged, recovering | Yes | **Works** | Charging in `reconditioning`. Motors not tested yet. |

Key takeaways:

- Serial link, cable, TX/RX and baud (115200 8N1) are confirmed good.
- Without a battery the robot board runs on dock/charger power, but the audio amp (and likely motors) don't get power.
- Deeply discharged packs show misleading voltage while charging. Trust the mAh trend and charging state.

---

## 2. Hardware and Wiring

### Serial port (7-pin Mini-DIN, female on robot)

```
1, 2  Vpwr  unregulated battery (~14-16V, fused ~1.5A)
3     RXD   into robot (0-5V)
4     TXD   out of robot (0-5V)
5     BRC   baud rate change / wake from sleep
6, 7  GND
```

Cable side needs a **male 7-pin Mini-DIN**. Don't confuse it with PS/2 (6-pin) or standard S-Video (4-pin).

### Connection options

1. **USB-serial cable (used today).** iRobot Create 2 cable or any FTDI adapter set to **5V**. Shows up as `/dev/ttyUSB0`. No level shifting needed.
2. **Direct UART to header.** Jetson/Raspberry Pi GPIO is 3.3V and robot TX is 5V, so a level shifter (or 1k/2k divider on robot TX → host RX) is required.
   - Raspberry Pi: TX pin 8 (GPIO14), RX pin 10 (GPIO15), GND pin 6, device `/dev/serial0`. Enable UART and disable serial console in `raspi-config`.
   - Jetson Orin: header UART is `/dev/ttyTHS0` (original Nano: `/dev/ttyTHS1`, disable `nvgetty`).

### Serial settings

115200 baud, 8 data bits, no parity, 1 stop bit, no flow control. Some units come up at 19200. Holding Clean ~10 s toggles baud on many models.

### Power notes

- Battery: 14.4V nominal pack.
- Charging is done **through the robot**, never directly on the pack. Use the Home Base dock or the iRobot adapter (~22.5V DC, ~1.25A) in the side jack.
- Don't power a Jetson from Mini-DIN Vpwr (current-limited). Use a separate battery or tap the robot battery through a wide-input buck converter (pack swings ~12V to ~20V on dock).

### Setup on Jetson

```bash
sudo usermod -aG dialout $USER   # then log out / in
pip3 install pyserial            # if missing
ls /dev/ttyUSB*
dmesg | grep -i tty              # confirm ttyUSB0 is the FTDI cable, not a Yahboom device
```

---

## 3. OI Reference Used Today

| Opcode | Command | Notes |
|---|---|---|
| 128 | Start | Enters Passive mode |
| 131 | Safe | Reverts to Passive if charger connected, cliff or wheel drop |
| 132 | Full | No safety checks. Needed for song playback while on charger |
| 7 | Reset | Robot reboots and prints boot text |
| 142 | Sensors | `[142, packet_id]` |
| 149 | Query List | `[149, n, id1, id2, ...]` |
| 148 | Stream | Robot sends packets every 15 ms |
| 150 | Pause/Resume Stream | `[150, 0]` stops |
| 140 / 141 | Song / Play | Play requires Safe or Full |
| 145 | Drive Direct | `[145, right_hi, right_lo, left_hi, left_lo]` mm/s |
| 144 | PWM Motors | Brushes / vacuum |

Packets used:

| ID | Meaning | Bytes |
|---|---|---|
| 7 | Bumps and wheel drops | 1 |
| 9-12 | Cliff L, FL, FR, R | 1 each |
| 17 | IR omni (dock signal 160-175) | 1 |
| 18 | Buttons | 1 |
| 21 | Charging state (0 not charging, 1 reconditioning, 2 full, 3 trickle, 4 waiting, 5 fault) | 1 |
| 22 | Voltage (mV) | 2 |
| 25 / 26 | Charge / capacity (mAh) | 2 each |
| 34 | Charging source (1 = jack, 2 = dock) | 1 |
| 35 | OI mode (0 off, 1 passive, 2 safe, 3 full) | 1 |
| 37 | Song playing | 1 |
| 45 | Light bumper | 1 |

---

## 4. Robot #1: Create 2, battery removed

### Results

1. `ls /dev/ttyUSB*` first failed, then showed `/dev/ttyUSB0` after replug.
2. First `test_create.py` returned `b''` (no response). Robot was asleep.
3. `diag_create.py` (reset + listen) at 115200 returned full boot text:
   ```
   Soft reset!
   Roomba by iRobot!
   stm32
   2016-06-13-1127-L
   r3-robot/tags/release-stm32-3.7.5:6216 CLEAN
   bootloader id: 3115 C214 8203 6580
   assembly: 3.5-lite
   revision: 8
   start-charge ...
   Determining battery type.
   do-charging-recovery @ minutes 0
   Performing charger self-test.
   ```
   19200 returned garbage, confirming 115200 is correct.
4. `test_create.py` then returned `1510 mV`.
5. `monitor_batt.py` returned `FAULT 1258 mV`, then `FAULT 951 mV`, `2697 mAh`.
   - The battery had been **removed on purpose**. The FAULT is the charger not finding a pack. The mAh value is stale from memory.
6. `debug_sound.py`:
   ```
   after Start, mode = 1
   after Full,  mode = 3
   song playing = 1
   final mode   = 3
   ```
   Software path is correct, but no audio. Likely the audio amp isn't powered without a battery.

### Conclusion

Board and serial are healthy. Sound and motors need to be retested with a battery installed.

---

## 5. Robot #2: Roomba 600-STM32, no OI response

### Results

1. `full_check.py` v1 printed boot lines `iRobotBoot V1`, `Roomba 600-STM32`, then failed reading sensors.
2. `full_check.py` v2 reported `OI mode = 13`. **This is not a mode.** 13 is ASCII `\r`, leftover text in the buffer.
3. `sniff.py`:
   ```
   idle 5s:           b'\xfe\xfe\xfe'
   after Clean 5s:    b'\xfe\xfe\xfe\xfe\xfe\xfe'
   query mode:        (silent)
   query voltage:     (silent)
   ```
   Robot only emits `0xFE` repeatedly and never answers OI queries. On robot #1, `0xFE` appeared right after `Soft reset!`, so it marks a boot.

### Hypotheses

1. Reboot loop caused by a weak battery (most likely).
2. Stuck in bootloader (`iRobotBoot V1`), main firmware / OI not running.

### Next steps

1. Put it on the dock and rerun `sniff.py`. If `0xFE` stops and queries get answered, it was the battery.
2. Note light behavior and any error beeps when pressing Clean.
3. Full reset: remove battery ~10 s, reinstall, dock, run `diag_create.py`.
4. Check model number on the bottom sticker (614, 650, 690, 694...). Newer Wi-Fi 600s can behave differently.

---

## 6. Robot #3: Roomba on charger, working

### Results

1. `sniff.py` showed the robot's own charging log once per second:
   ```
   bat: min 0 sec 10  mV 5930  mA 302  tenths-deg-C 182  mAH 0  state 6  mode 8
   ...
   bat: min 0 sec 28  mV 7608  mA 302  tenths-deg-C 182  mAH 1  state 6  mode 8
   ```
   Voltage rising, ~300 mA recovery current, 18.2 °C.
2. Query mode returned `\x01` (Passive). Query voltage returned `\x1f\x08` = 7944 mV.
3. `test_sound.py`: **robot played the tune.** Speaker and command path confirmed.
4. `monitor_batt.py`:
   ```
   09:50:23 reconditioning 16923 mV  7 mAh
   09:50:53 reconditioning 16867 mV  9 mAh
   09:51:23 reconditioning 16783 mV 11 mAh
   ```
   Voltage looks high only because charge current is flowing. Real charge is ~11 mAh of a ~2000-3000 mAh pack. mAh rising ~2 per 30 s (~240 mAh/h) matches the recovery current.

### Expected progression

`reconditioning` → `full charging` (faster) → `trickle` (full). Motor test only after `trickle` or several thousand mAh. If it goes to `FAULT` or mAh stops rising for an hour, the pack is weak.

### Conclusion

Fully working over OI. Battery is recoverable. Continue charging, then run sensor and motor tests.

---

## 7. Lessons and Gotchas

1. **Asleep robot = no response.** In Passive on battery it sleeps after ~5 min. Press Clean or pulse BRC low. Safe/Full mode keeps it awake.
2. **Python pasted into bash fails** (`import: command not found`). Use a heredoc into a `.py` file, then `python3 file.py`. Don't paste command output back into the terminal.
3. **Safe mode on the charger reverts to Passive.** Song playback and driving while docked require Full mode (`132`). Full mode disables cliff and wheel-drop safety, so lift the robot.
4. **Entering Safe/Full pauses charging.** Return to Passive (`128`) afterward. If charging doesn't resume, re-dock.
5. **Charging log text shares the serial line.** Always `reset_input_buffer()` before a query. A strange single-byte answer like 13 is likely `\r` from log text.
6. **mAh is stale** on a deeply discharged pack and **voltage lies while charging.** Watch the mAh trend and the charging state.
7. **Wrong baud** gives garbage bytes, not silence.
8. **Create 3 is different:** no Mini-DIN or serial OI. It connects over USB-C Ethernet (`192.168.186.2`) or Wi-Fi and is controlled through ROS 2 topics.

---

## 8. Scripts

All scripts assume `/dev/ttyUSB0` at 115200.

### `diag_create.py`: wake, reset, print boot text at two bauds

```python
import serial, time
for baud in (115200, 19200):
    print(f"\n=== baud {baud} ===")
    s = serial.Serial('/dev/ttyUSB0', baud, timeout=0.5)
    s.rts = True; time.sleep(0.1); s.rts = False; time.sleep(0.5); s.rts = True
    s.reset_input_buffer()
    s.write(bytes([128])); time.sleep(0.1)
    s.write(bytes([7]))
    t = time.time(); buf = b''
    while time.time() - t < 6:
        buf += s.read(256)
    print(repr(buf[:500]) if buf else "no data")
    s.close()
```

### `test_create.py`: single voltage read

```python
import serial, time
s = serial.Serial('/dev/ttyUSB0', 115200, timeout=1)
s.write(bytes([128]))
time.sleep(0.1)
s.write(bytes([142, 22]))
data = s.read(2)
print(data, int.from_bytes(data, 'big'), 'mV' if data else 'no response')
```

### `monitor_batt.py`: charging state, voltage, charge every 30 s

```python
import serial, time
s = serial.Serial('/dev/ttyUSB0', 115200, timeout=1)
s.write(bytes([128])); time.sleep(0.1)
states = {0:'not charging',1:'reconditioning',2:'full charging',3:'trickle',4:'waiting',5:'FAULT'}
while True:
    s.reset_input_buffer()
    s.write(bytes([149, 3, 21, 22, 25]))
    d = s.read(5)
    if len(d) == 5:
        st, mv, mah = d[0], int.from_bytes(d[1:3],'big'), int.from_bytes(d[3:5],'big')
        print(time.strftime('%H:%M:%S'), states.get(st, st), f'{mv} mV', f'{mah} mAh')
    else:
        print('no data')
    time.sleep(30)
```

### `test_sound.py`: play do-re-mi-fa-sol (Full mode)

```python
import serial, time
s = serial.Serial('/dev/ttyUSB0', 115200, timeout=1)
s.write(bytes([128])); time.sleep(0.1)
s.write(bytes([132])); time.sleep(0.1)
notes = [60, 62, 64, 65, 67]
song = [140, 0, len(notes)]
for n in notes:
    song += [n, 16]
s.write(bytes(song))
s.write(bytes([141, 0]))
time.sleep(2)
s.write(bytes([128]))
print('done')
```

### `debug_sound.py`: verify mode and song-playing flag

```python
import serial, time
s = serial.Serial('/dev/ttyUSB0', 115200, timeout=1)

def q(pid):
    s.reset_input_buffer()
    s.write(bytes([142, pid]))
    d = s.read(1)
    return d[0] if d else None

s.write(bytes([128])); time.sleep(0.2)
print('after Start, mode =', q(35))
s.write(bytes([132])); time.sleep(0.2)
print('after Full,  mode =', q(35))
s.write(bytes([140, 0, 3, 72, 32, 76, 32, 79, 32])); time.sleep(0.1)
s.write(bytes([141, 0])); time.sleep(0.1)
print('song playing =', q(37))
time.sleep(2)
print('final mode   =', q(35))
s.write(bytes([128]))
```

### `sniff.py`: raw listen, then query

```python
import serial, time
s = serial.Serial('/dev/ttyUSB0', 115200, timeout=0.2)

def listen(sec, label):
    print(f'\n--- {label} ({sec}s) ---')
    t, buf = time.time(), b''
    while time.time() - t < sec:
        buf += s.read(256)
    print(repr(buf) if buf else '(silent)')

listen(5, 'idle, touch nothing')
input('\nPress Clean on the robot once, then Enter...')
listen(5, 'after Clean')
s.write(bytes([128])); time.sleep(0.2)
s.reset_input_buffer()
s.write(bytes([142, 35]))
listen(2, 'mode query (expect 1 byte: \\x01)')
s.write(bytes([142, 22]))
listen(2, 'voltage query (expect 2 bytes)')
```

### `sensor_live.py`: live bumpers, cliffs, IR, buttons, charger source (not run yet)

```python
import serial, time
s = serial.Serial('/dev/ttyUSB0', 115200, timeout=1)
s.write(bytes([128])); time.sleep(0.1)
pkts = [7, 9, 10, 11, 12, 17, 18, 34, 45]
while True:
    s.reset_input_buffer()
    s.write(bytes([149, len(pkts)] + pkts))
    d = s.read(len(pkts))
    if len(d) == len(pkts):
        b = d[0]
        print(f"bumpL={b>>1&1} bumpR={b&1} drop={b>>2&3} "
              f"cliff={list(d[1:5])} IR={d[5]} btn={d[6]:08b} "
              f"charger={d[7]} lightbump={d[8]:06b}", end='\r')
    time.sleep(0.1)
```

### `stream_test.py`: stream rate and checksum check (not run yet)

Target: ~66 Hz, 0 checksum errors.

```python
import serial, time
s = serial.Serial('/dev/ttyUSB0', 115200, timeout=1)
s.write(bytes([128])); time.sleep(0.1)
s.write(bytes([148, 2, 7, 18]))
s.reset_input_buffer()
ok = bad = 0; t = time.time()
while time.time() - t < 10:
    if s.read(1) != b'\x13': continue
    n = s.read(1)[0]; body = s.read(n); ck = s.read(1)
    if (19 + n + sum(body) + ck[0]) & 0xFF == 0: ok += 1
    else: bad += 1
s.write(bytes([150, 0]))
print(f'{ok} packets OK, {bad} checksum errors, ~{ok/10:.0f} Hz')
```

### `test_motor.py`: wheels (lift robot first; not run with a battery yet)

Uses Full mode so it works on the charger. Use `131` instead of `132` when off the charger to keep safety checks.

```python
import serial, time, struct
s = serial.Serial('/dev/ttyUSB0', 115200, timeout=1)

def drive(left, right):
    s.write(bytes([145]) + struct.pack('>hh', right, left))

s.write(bytes([128])); time.sleep(0.1)
s.write(bytes([132])); time.sleep(0.1)
s.reset_input_buffer()
s.write(bytes([142, 35]))
m = s.read(1)
print('OI mode:', m[0] if m else 'no data', '(3 = Full)')
drive(100, 100);   time.sleep(2)
drive(0, 0);       time.sleep(1)
drive(-100, -100); time.sleep(2)
drive(-100, 100);  time.sleep(2)
drive(0, 0)
s.write(bytes([128]))
```

### `full_check.py` (v2): comms, battery, sound, sensors, motors

v1 sent a reset (`7`) first, which left robot #2 unresponsive. v2 drops the reset and retries with a wake prompt.

```python
import serial, time, struct, os, sys
PORT = '/dev/ttyUSB0'
if not os.path.exists(PORT): sys.exit(f'{PORT} missing.')
s = serial.Serial(PORT, 115200, timeout=0.5)

def q(ids, n):
    s.reset_input_buffer()
    s.write(bytes([149, len(ids)] + ids))
    return s.read(n)

print('== 1. Comms ==')
mode = None
for i in range(5):
    s.write(bytes([128])); time.sleep(0.3)
    d = q([35], 1)
    if d: mode = d[0]; break
    input('No response. Press Clean on the robot, then Enter...')
if mode is None: sys.exit('Robot still not answering.')
print('OK, OI mode =', mode)   # must be 0-3; 13 means stray '\r'

print('\n== 2. Battery ==')
states = {0:'not charging',1:'reconditioning',2:'full charging',3:'trickle',4:'waiting',5:'FAULT'}
d = q([21, 22, 25, 26, 34], 8)
if len(d) != 8: sys.exit(f'Battery read failed, got {len(d)} bytes: {d!r}')
st, mv = d[0], int.from_bytes(d[1:3], 'big')
chg, cap, src = int.from_bytes(d[3:5], 'big'), int.from_bytes(d[5:7], 'big'), d[7]
print(f'status={states.get(st, st)}  {mv} mV  {chg}/{cap} mAh  charger_source={src}')
batt_ok = mv > 13000 and st != 5

print('\n== 3. Sound ==')
s.write(bytes([132])); time.sleep(0.2)
s.write(bytes([140, 0, 3, 72, 16, 76, 16, 79, 32])); time.sleep(0.1)
s.write(bytes([141, 0])); time.sleep(1.5)
input('Did it beep? Note it, then Enter...')
s.write(bytes([128])); time.sleep(0.2)

print('\n== 4. Live sensors 15 s ==')
t = time.time()
while time.time() - t < 15:
    d = q([7, 9, 10, 11, 12, 17, 18, 45], 8)
    if len(d) == 8:
        b = d[0]
        print(f'bumpL={b>>1&1} bumpR={b&1} drop={b>>2&3} cliff={list(d[1:5])} '
              f'IR={d[5]:3d} btn={d[6]:08b} lightbump={d[7]:06b}   ', end='\r')
    time.sleep(0.1)
print()

print('\n== 5. Motors ==')
if not batt_ok:
    print('Skipped: battery not OK.')
elif input('LIFT the robot. Test motors? (y/n) ').lower() == 'y':
    s.write(bytes([132 if src else 131])); time.sleep(0.2)
    def drive(l, r): s.write(bytes([145]) + struct.pack('>hh', r, l))
    for name, l, r in [('fwd',100,100),('back',-100,-100),('left',-100,100),('right',100,-100)]:
        print(name); drive(l, r); time.sleep(1.5); drive(0, 0); time.sleep(0.5)
    s.write(bytes([128]))
print('\nDone.')
```

---

## 9. Open Items

- [ ] Robot #3: let charge reach `trickle`, then run `sensor_live.py`, `stream_test.py`, `test_motor.py`.
- [ ] Robot #1: install a battery (Roomba 600 series 14.4V, Li-ion preferred), retest sound and motors.
- [ ] Robot #2: dock and rerun `sniff.py`, full battery-out reset, record model number from sticker.
- [ ] Bring up ROS 2 `create_robot` / `libcreate` (AutonomyLab) on the Jetson and verify sensor topics.
- [ ] Decide Jetson power source for mobile use (separate battery vs. buck from robot pack).

## 10. References

- iRobot Create 2 Open Interface Spec (based on Roomba 600): https://edu.irobot.com/learning-library/create-2-oi-spec
- AutonomyLab `create_robot` (ROS 2 driver): https://github.com/AutonomyLab/create_robot
- Roomba 690 OI controller (confirms 690 Wi-Fi exposes wired OI): https://github.com/KieranK07/roomba-controller
- Hacking the Roomba 600 (sleep and BRC gotchas): https://www.crc.id.au/hacking-the-roomba-600/
- JetsonHacks Create 2 series: https://jetsonhacks.com/2015/06/15/jetson-tk1-create-2-robot-part-i/
