# EMG6

A body-driven particle instrument for TouchDesigner by **Göktuğ Kahyaoğlu**.

A performer wears an EMG (muscle) sensor. Contraction, and optionally movement, shapes a
field of one million GPU particles that moves through six worlds:

**ZERO** (black) → **DUST** → **RINGS** → **SLICES** → **BLACKHOLE** → **VORTEX** (the end)

- Holding a contraction at 0.70 is a full gesture: the magnet gathers the dust around the hand.
- Holding 0.85 for 12 seconds moves to the next world.
- From ZERO, a contraction held for about 10 seconds gives birth to the field.

## Run it

1. Open `EMG6.toe` in TouchDesigner (made with 2025.32820 on macOS). Keep the
   other files in this folder next to it.
2. **Ctrl+9** opens the output window (1080 × 1350, vertical). **Ctrl+0** closes it. Set
   the projector in `/cntrl/perform` → Display.
3. Open the **CONTROL** panel (right-click `CONTROL` → View):
   - **Start from ZERO**: black screen, ready for a birth.
   - **Auto demo**: plays the whole piece with a simulated performer (about 2.5 min).
   - **Rescue**: one click if the image ever looks wrong.
   - **Simulated contraction**: drive the piece by hand without a sensor.
   - **Particles**: 64K to 2M (fewer is lighter on the GPU but a dimmer image). Release
     the slider to apply.

Other shortcuts (with Ctrl): `` ` `` re-birth, **8** record video, **7** still image.

## Connect a sensor

1. Send OSC to this computer on UDP port **9100**.
2. Open the **CALIBRATE** panel (right-click `CALIBRATE` → View) and follow it:
   - **Find EMG channel**: relax, then squeeze. The muscle channel is found from
     behaviour (raw or enveloped EMG, single or multi-channel) and its range is set.
   - **Find IMU axes** (if the sensor has one): still, turn right, raise, twist,
     still. Every gyro and accelerometer axis, its sign and its unit is found,
     plus the gyro bias and the centre.
   - **Capture reach**: left, right, up, down. Your reach becomes the edge of the field.
   - The meter shows 0.22 (magnet), 0.70 (full gesture) and 0.85 held for 12 s (next world).

The hand estimate uses the same technique as our own sensor system:
- left/right: dead reckoning, so the turn rate is integrated with the true time
  between messages and the hand stays where you leave it;
- up/down and twist: gravity (absolute) or the gyro (holds position), switchable.

Everything ends up in two tables in `/INPUT` (`channel_map`, `imu_map`), which can also
be edited by hand. `/INPUT/README` explains them, including formulas over raw
channels and the dropout safety: if the sensor stops, the contraction falls to 0.

## Use it with Claude

The project contains the TouchDesigner MCP bridge, so Claude Code can operate it.
Read **`CLAUDE.md`**: it covers connecting, identifying the EMG device, and calibrating it.

## Files

| file | what |
|---|---|
| `EMG6.toe` | the project |
| `CLAUDE.md` | instructions for an AI assistant (setup + calibration) |
| `mcp_webserver_base_v1.4.12.tox`, `modules/` | TouchDesigner MCP bridge (port 9981) |
| `pylibs/` | imufusion + vqf (IMU fusion for the sound-side analysis, Apple Silicon / Python 3.11) |
| `fusion_cal.json` | IMU calibration of our own sensor (sound side) |
| `imu_cal.json` | your sensor's calibration (written by the CALIBRATE panel) |

## Notes

- Output is 1080 × 1350. A Non-Commercial TouchDesigner licence caps output at 1280 px.
- The sound side (`/foundation`, `/tension_logic`, `/routing`) drives our Ableton set and
  is not needed for the visuals.
- The particle shader (`/Pop_glsl_sort/glsl1_compute`) is original code, fully commented.

## Licence

MIT © 2026 Göktuğ Kahyaoğlu. Use it, change it, sell it, build on it: anything,
as long as the licence notice stays with it. Bundled third-party components (also
MIT) are listed in `THIRD_PARTY_NOTICES.md`.

---

## Update — 2 November 2026

Tested with a live sensor. This update changes how the piece responds; where it differs
from the text above, this section is the current behaviour.

**Response**
- The magnet now opens at **0.15** (and closes below 0.12), and it is visible at once
  (about 0.5 s) in every world. Before, it opened at 0.22 and took 6–8 s to show.
- In DUST the magnet first gathers a ball, then a ring opens and grows while you hold.
- **Next world:** keep the magnet on for **9 seconds**. Short dips are forgiven (they only
  rewind the count); a real release clears it. This replaces "0.85 held for 12 s".
- Letting go is faster: the contraction falls 4× faster than it rises, so the field melts
  within about a second.
- The birth from ZERO is unchanged (about 0.22 held for ~10 s).
- The meter on the CALIBRATE panel shows a white line at 0.15. The 0.70 / 0.85 marks are hidden.

**Calibration**
- Find EMG channel can no longer pick an IMU channel as the muscle.
- Find EMG sets `smooth_s` to 0.2 (rise only), so spikes after a release don't re-open the magnet.
- Find IMU axes keeps the previous accelerometer rows if it can't find the accelerometer again.
- A non-finite contraction value (inf/NaN) is now caught and read as 0.

**Shipped state:** port 9100, source `sim`, no channels mapped, `imu_cal.json` at defaults.
Run the CALIBRATE panel with your own sensor.

**Known open points:** not tested on Windows yet; the cause of a one-off `inf`
contraction value is not found (it is now guarded).
