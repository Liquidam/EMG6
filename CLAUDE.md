# EMG6 — instructions for Claude

You are helping someone run **EMG6.toe**, a TouchDesigner instrument by Göktuğ Kahyaoğlu.
A performer wears an EMG (muscle) sensor. Their contraction, and optionally their movement,
shapes a field of 1M GPU particles that walks through six worlds:
`ZERO (black) → DUST → RINGS → SLICES → BLACKHOLE → VORTEX (the end)`.

Your job is usually one thing: **find out which EMG device they have, connect it, and
calibrate it.** The visuals are finished. Do not redesign them.

Keep replies short. The user watches the projection; you can't see it, so describe
what should be visible and ask. Change one thing at a time.

---

## 0. Connect to TouchDesigner

1. Open `EMG6.toe` in TouchDesigner (built on 2025.32820, macOS). Keep the bundled
   files (`mcp_webserver_base_v1.4.12.tox`, `modules/`, `pylibs/`) next to it.
   The MCP bridge (`/mcp_webserver_base1`) starts with the project and listens on port 9981.
2. Register the client once (needs Node.js):
   `claude mcp add touchdesigner -- npx -y touchdesigner-mcp-server@latest --stdio`
   Then restart Claude Code. The client may report an "update recommended" warning; it still works.
3. Use **`execute_python_script`** for everything. Inside it:
   - always `import td` and call `td.op(...)`, `td.absTime`, `td.project`. Bare names
     aren't defined there.
   - keep scripts ASCII only.
   - `time.sleep` blocks the frame and time doesn't advance inside one call. To watch
     something change, make several separate calls.
   - `get_td_nodes` is unreliable; read the network with Python instead.

First check (one call):
```python
import td
errs=[o.path for o in td.op('/').findChildren() if o.errors()]
b=td.op('/INPUT/body')
result='%s errors=%s source=%s emg=%.2f alive=%d' % (td.project.name, errs,
  td.op('/INPUT/settings')['source','value'].val, b['emg'].eval(), b['osc_alive'].eval())
```

## 1. Where things are

| what | where |
|---|---|
| start here (human overview) | root, the yellow START HERE box |
| **sensor input** | `/INPUT` (`settings`, `channel_map`, `imu_map`, `README`) |
| **guided calibration (use this first)** | `/CALIBRATE` panel, logic in `/CALIBRATE/wizard` |
| operator panel (Start from ZERO, Auto demo, Rescue, sliders) | `/CONTROL`, logic in `/CONTROL/actions` |
| particle engine (shader, worlds, render) | `/Pop_glsl_sort` (numbered boxes 1-5) |
| output window / recorders / shortcuts | `/cntrl` |
| sound side (our ESP32 → Ableton) | `/foundation`, `/tension_logic`, `/routing`. Not needed for the visuals. |

`/INPUT/body` is what the visuals read: `emg motion twist az el` (0..1), `dist`, `osc_alive`.
Any role whose channel cell is empty is played by the simulated performer: `emg` from
`/foundation/SIM_effort`, the hand drifting slowly, `motion` 0.7.

## 1b. The calibration panel does the work. Use it.

`/CALIBRATE` (right-click > View) is a guided panel. Each button shows a big
instruction plus a countdown, records the performer, and writes the result into
`/INPUT/channel_map` and `/INPUT/imu_map` by itself. The same steps exist as
functions in `mod('/CALIBRATE/wizard')`:

| button | function | what the performer does | result |
|---|---|---|---|
| Find EMG channel | `emg_find()` | relax 4 s, squeeze 4 s | picks the muscle channel from behaviour, detects envelope vs raw EMG, sets the range, source = osc |
| Rest again / Max again | `emg_rest()` / `emg_max()` | relax / squeeze | refines `in_min` / `in_max` |
| Find IMU axes | `imu_axes()` | still → turn right → raise → twist → still | finds which gyro/accel channel is turn/raise/twist/up/forward/side, their signs and units (rad/s, m/s² are converted), the gyro bias, the centre. Maps az/el/twist/motion to the estimator. |
| Still + centre | `imu_still()` | hold still 3 s | gyro bias + centre (refuses if they moved > 8 deg/s) |
| Centre now | `imu_center()` | — | this pose = middle of the field (fixes slow drift) |
| Capture reach | `imu_reach()` | reach left / right / up / down | how far THEY reach = the edge of the field |

Order with a new device: Find EMG channel → Find IMU axes → Capture reach. Then test
with the meter on the panel (white 0.22 / green 0.70 / red 0.85).

How the input works (for when something needs debugging):
- `/INPUT/osc_rx` (OSC In DAT) is the ONLY receiver. It runs on **every message**, not
  once per frame. `/INPUT/osc_in` just shows the newest value of every channel.
- For every channel there is also `<name>_env`: DC removed, rectified, smoothed per
  message. Raw/bipolar EMG has to use its `_env`; an already-enveloped sensor
  (e.g. MyoWare) is used directly. The wizard decides which.
- `/INPUT/imu_est` is the hand estimator (the technique of our own ESP32 system, see
  its header). Left/right is dead reckoning: the turn rate is integrated with the true
  time between messages, so the hand stays where it was left. Up/down and twist come
  from gravity (absolute) or the gyro (holds, drifts slowly); toggle on the panel.
  Calibration persists in `imu_cal.json`.
- `/INPUT/test_virtual_performer` is a TEST tool: a fake Myo-style device that follows
  the wizard's instructions. `mod('/INPUT/test_virtual_performer').start()` / `.stop()`.
  Never leave it running with a real sensor.

## 2. Doing it by hand (only if the panel can't): identify the EMG device without guessing

1. **Ask the user** for the device name and model, and how it sends data (OSC app,
   bridge software, ESP32 firmware). Common cases:
   - single envelope over OSC (MyoWare + ESP32, custom boards): one value per message
   - multi-channel arrays (Myo via a bridge, BioMuse/EAVI boards, BITalino via a
     bridge): several values in one message, or one address per channel
   - our own firmware sends `/bio seq t_micros emg_raw ax ay az gx gy gz`
2. Make it send to this computer's IP on UDP **9100**. If the device can't change its
   port, change `/INPUT/settings` row `port` instead.
3. Set `settings` `source` to `osc`, then read what actually arrives:
```python
import td
o=td.op('/INPUT/osc_in')
result=str([(c.name, round(c.eval(),3)) for c in o.chans()])
```
   TD names channels after the OSC address: `/emg 0.4` gives `emg`, `/myo/emg 1 2 3` gives
   `myo/emg1 myo/emg2 myo/emg3`.
   If nothing arrives: check the device's destination IP and port, the macOS firewall,
   and whether another app holds the port (`lsof -nP -i UDP:9100`).
4. **Find the muscle channel by behaviour, not by name.** Ask the performer to relax for
   5 s, then squeeze for 5 s. Read the channels in separate calls each time. The EMG
   channel is the one that rises clearly on the squeeze. Report what you saw in one line.

## 3. Calibrate EMG

Collect relaxed and squeeze values over a few seconds with repeated calls. Then in
`/INPUT/channel_map`, row `emg`:

- `osc_channel`: the channel name. For multi-channel EMG, use a formula over the
  channels on the active muscle, e.g. `=max(abs(ch('myo/emg3')), abs(ch('myo/emg4')))`,
  or the mean of several.
- `in_min`: a little above the typical relaxed value, so rest reads 0.
- `in_max`: about 90% of a comfortable full squeeze, so it doesn't take maximum effort.
- `smooth_s`: 0.05 to start. Raise it toward 0.15 if the image flickers, lower it if
  it feels late.

Then check `/INPUT/body` `emg` with the performer, and tell them the meaning of each level:

| emg | meaning |
|---|---|
| 0.22 | the magnet appears |
| 0.70 | full gesture, never changes the world (safe) |
| 0.85 held 12 s | next world |

The performer should be able to hold 0.70 comfortably and reach 0.85 on purpose. If
they can't, adjust `in_max`, not the piece.

## 4. Movement (optional)

Only if the device has an IMU and the user wants movement. Map one role at a time and
have the performer test each one.

- `az` / `el` (hand left-right / up-down): orientation angles are best. Keep
  `out_min/out_max` at **0.45–0.55 (az)** and **0.44–0.56 (el)**. That is the usable band;
  wider pushes the magnet into the field walls.
- `twist`: wrist roll, 0.5 = neutral (spins the VORTEX). From raw gravity:
  `=atan2(ay, az)`.
- `motion`: amount of movement. From a raw gyro: `=sqrt(gx**2+gy**2+gz**2)` with
  `in_max` around 300 (deg/s).

Unmapped movement roles keep the gentle simulated drift. That is how the reference
demo looks, and it's a fine way to perform with EMG only.

## 5. Test

In `/CONTROL`: **Start from ZERO** (black screen), then the performer holds a
contraction for about 10 s. The world is born, there is a flash, and 5 s later
contraction works again. **Auto demo** plays the whole piece with the simulated
performer (it only drives the image while no live `emg` is mapped).

Safety already built in: if no mapped value changes for 2 s, the sensor counts as gone,
so `emg` drops to 0 and nothing changes the world on its own.

## 6. Rules: don't break these

- Don't edit the shader (`/Pop_glsl_sort/glsl1_compute`) unless asked. If asked: never
  add a new output attribute while it runs (save and reopen first), never write a zero
  `Rot` (that makes the field invisible), never change `u_world` z (0.001 is by design),
  and keep sort keys ≥ 0 and continuous.
- Never pulse `feedback2` **Initialize**. Re-birth = its **Start** pulse, or the
  `/CONTROL` Rescue button.
- In DAT callbacks use `td.absTime.frame`, not the `frame` argument (that's the
  timeline frame).
- Don't put Python functions in operator storage. Saving fails to pickle them.
- Image looks wrong → **Rescue** first. Ghosts → `td.op('/Pop_glsl_sort/feedback1').par.resetpulse.pulse()`.
- Save with `td.project.save()` only when the user says so (the file version number
  goes up; that's normal).
- Output is 1080 × 1350 (vertical). A Non-Commercial TouchDesigner licence caps it at
  1280 px, so it shows 1024 × 1280 there.

When you finish a calibration, write what you set (device, channels, ranges) into
a `SESSION_<date>.md` next to the .toe, so the next person doesn't start from zero.
