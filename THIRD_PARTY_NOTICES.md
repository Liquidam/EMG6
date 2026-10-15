# Third-party components

The EMG6 project itself (the TouchDesigner network, the particle shader, the
input / calibration system and the documentation) is © 2026 Göktuğ Kahyaoğlu and
licensed under the **MIT licence** (see `LICENSE`).

The following bundled components belong to their authors and keep their own licences:

| component | where | licence |
|---|---|---|
| **Fusion / imufusion** 1.3.2, x-io Technologies | `pylibs/imufusion*` | MIT, see `pylibs/imufusion-1.3.2.dist-info/licenses/LICENSE.md` |
| **VQF** 2.1.2, Daniel Laidig | `pylibs/vqf*` | MIT, see `pylibs/vqf-2.1.2.dist-info/` |
| **touchdesigner-mcp** 1.4.x, 8beeeaaat | `mcp_webserver_base_v1.4.12.tox`, `modules/` | MIT, https://github.com/8beeeaaat/touchdesigner-mcp |
| **RGBA Delay** palette component, Derivative | inside `EMG6.toe` (`/Pop_glsl_sort/rgbaDelay1`) | part of TouchDesigner's Palette, used under the TouchDesigner licence |

`pylibs/` holds compiled packages for macOS on Apple Silicon with Python 3.11
(TouchDesigner's own Python). They are only used by the sound-side IMU analysis in
`/foundation`; the visuals and the CALIBRATE panel don't need them.
