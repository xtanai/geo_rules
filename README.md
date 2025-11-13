# Vision Geometry Rules

A concise, practical set of formulas for camera FOV, coverage, pixel scale, fisheye rectification, and depth (Z) resolution. 

## 1) Notation

* `Ws, Hs` — sensor width/height (mm)
* `Nx, Ny` — image resolution (px)
* `px = Ws / Nx`, `py = Hs / Ny` — pixel pitch (mm/px)
* `f` — focal length (mm)
* `HFOV, VFOV, DFOV` — horizontal/vertical/diagonal field of view (rad or deg)
* `Z` — working distance to object plane (mm)
* `b` — stereo baseline (mm)
* `fpx = f / px` — focal length in pixel units (px)
* `theta` — incident angle from optical axis (radians)
* `r` — image radius from principal point (px or mm)

> ⚠️ **Sensor size** labels (e.g., “1/4 inch”) are approximate. Always check the **datasheet** for exact `Ws, Hs`. Typical 1/4″ ≈ `Ws=2.88 mm`, `Hs=2.16 mm` (diag ≈ 3.6 mm).

---

## 2) FOV ↔ focal length (rectilinear model)

**Horizontal/Vertical FOV**

```text
HFOV = 2 * atan(Ws / (2*f))
VFOV = 2 * atan(Hs / (2*f))
```

**Solve for focal length** (given HFOV):

```text
f = Ws / (2 * tan(HFOV/2))
```

**Scene coverage** at distance `Z`:

```text
scene_width  = 2 * Z * tan(HFOV/2)
scene_height = 2 * Z * tan(VFOV/2)
```

---

## 3) Pixel scale (mm/px) & object resolution

**Ground Sampling Distance (GSD)** at distance `Z`:

```text
GSDx = scene_width  / Nx
GSDy = scene_height / Ny
```

This is your **mm per pixel** in the object plane.

**Pixel count for a feature of size `S` (mm):**

```text
pixels_x ≈ S / GSDx
```

---

## 4) Fisheye / lens projection models (radius ↔ angle)

Common angle-to-radius mappings (`r` vs `theta`):

```text
Rectilinear:     r = f * tan(theta)
Equidistant:     r = f * theta
Equisolid-angle: r = 2 * f * sin(theta/2)
Orthographic:    r = f * sin(theta)
Stereographic:   r = 2 * f * tan(theta/2)
```

**Fisheye → Rectilinear (radius mapping):**

1. Pick fisheye model (from datasheet).
2. Invert fisheye to get `theta` from `r_fe`. Examples:

```text
Equidistant:     theta = r_fe / f_fe
Equisolid:       theta = 2 * asin( clamp(r_fe / (2*f_fe), -1, 1) )
Orthographic:    theta = asin( clamp(r_fe / f_fe, -1, 1) )
Stereographic:   theta = 2 * atan( r_fe / (2*f_fe) )
```

3. Rectilinear radius:

```text
r_rect = f_rect * tan(theta)
```

4. Use `(r_fe, direction)` → `(r_rect, direction)` to remap pixels to a rectilinear grid.

> In practice: calibrate intrinsics (`fpx, cx, cy, distortion`) and use an undistort/rectify step (e.g., OpenCV `fisheye::undistortImage` or custom remap).

---

## 5) Z-resolution (depth precision)

### 5.1 Stereo triangulation

Depth from disparity `d` (px) with baseline `b` and focal in pixels `fpx`:

```text
Z = (b * fpx) / d
```

Depth uncertainty from disparity noise `sigma_d`:

```text
sigma_Z ≈ (Z*Z / (b * fpx)) * sigma_d
```

Improve Z-resolution by:

* Increasing `b`
* Increasing `fpx` (larger `f`, smaller `px`)
* Reducing `sigma_d` (better matching, texture, illumination, subpixel)

### 5.2 Single camera with known scale (planar target)

If a known object of size `S` (mm) appears with size `S_px` (px) at depth `Z`:

```text
S_px ≈ (fpx / Z) * S
=> Z ≈ (fpx * S) / S_px
```

Depth precision then follows from measurement noise on `S_px`; typically worse than true stereo.

---

## 6) Worked example (1/4″ sensor, 1280×800, f = 3.6 mm)

**Assumptions** (typical; verify in datasheet):

```text
Ws = 2.88 mm, Hs = 2.16 mm
Nx = 1280 px, Ny = 800 px
f  = 3.6 mm
Z  = 800 mm
b  = 60 mm
```

**FOV**

```text
HFOV = 2 * atan(2.88 / (2*3.6)) = 2 * atan(0.4)  ≈ 43.6 deg
VFOV = 2 * atan(2.16 / (2*3.6)) = 2 * atan(0.3)  ≈ 33.4 deg
```

**Coverage at Z = 800 mm**

```text
scene_width  = 2 * 800 * tan(43.6/2 deg)  ≈ 1600 * 0.4 ≈ 640 mm
GSDx         = 640 / 1280 ≈ 0.50 mm/px
```

**Z-resolution (stereo)**

```text
px      = Ws / Nx = 2.88 / 1280  = 0.00225 mm/px = 2.25 µm
fpx     = f / px  = 3.6 / 0.00225 ≈ 1600 px
sigma_d = 0.2 px

sigma_Z ≈ (Z*Z / (b * fpx)) * sigma_d
        ≈ (800*800 / (60 * 1600)) * 0.2
        ≈ 1.33 mm
```

---

## 7) Quick reference (useful trig)

```text
atan_inv(x) = arctan(x)

Small-angle (radians):
tan(theta) ≈ theta

Conversions:
deg = rad * 180/pi
rad = deg * pi/180
```

---

## 8) Practical tips

* Treat **FOV labels (90°, 110°, 60°)** as rough; **sensor size + focal length** define true geometry.
* Prefer **datasheet** `Ws, Hs` and **calibrated** `fpx` for precision.
* For fisheye/strong distortion, **rectify first**, then compute coverage/GSD in the rectified frame.
* For target distances **500–1200 mm**, precompute a table of `{scene_width, GSD, sigma_Z}` to select lens/baseline.

---

## 9) Minimal “design formulas” (copy/paste)

**Given** `Ws, Hs, f, Z, Nx, Ny, b`

```text
HFOV = 2 * atan(Ws / (2*f))
VFOV = 2 * atan(Hs / (2*f))

scene_width  = 2 * Z * tan(HFOV/2)
scene_height = 2 * Z * tan(VFOV/2)

GSDx = scene_width  / Nx
GSDy = scene_height / Ny

px  = Ws / Nx
fpx = f / px

# Stereo depth precision (disparity noise = sigma_d)
sigma_Z = (Z*Z / (b * fpx)) * sigma_d
```

**Fisheye → Rectilinear (equidistant model)**

```text
# fisheye (equidistant): r_fe = f_fe * theta
theta  = r_fe / f_fe
r_rect = f_rect * tan(theta)
```

Swap the projection law if your lens is equisolid/orthographic/stereographic.

---

## 10) Example table generator (Python)

```python
#!/usr/bin/env python3
"""
Generate a CSV table of scene coverage, pixel scale (GSD), and depth uncertainty σZ
for given sensor, resolution, lens, and stereo setup.

Use:
    python geometry_table.py > geometry_table.csv
"""

import math
import csv
import sys

# --- Camera parameters (adjust as needed) ---
Ws = 2.88     # mm sensor width (1/4")
Hs = 2.16     # mm sensor height
Nx = 1280     # px horizontal resolution
Ny = 800      # px vertical resolution
f = 3.6       # mm focal length
b = 60.0      # mm stereo baseline
sigma_d = 0.2 # px disparity noise

# --- Derived values ---
HFOV = 2 * math.atan(Ws / (2 * f))
VFOV = 2 * math.atan(Hs / (2 * f))
px_pitch = Ws / Nx
fpx = f / px_pitch  # focal length in pixels

def scene_geometry(Z):
    """Return width, GSD, and sigmaZ for given working distance Z (mm)."""
    width = 2 * Z * math.tan(HFOV / 2)
    gsd_x = width / Nx
    sigmaZ = (Z * Z / (b * fpx)) * sigma_d
    return width, gsd_x, sigmaZ

writer = csv.writer(sys.stdout)
writer.writerow(["Z_mm", "scene_width_mm", "GSDx_mm_per_px", "sigmaZ_mm"])

for Z in range(500, 1250, 50):
    width, gsd, sigmaZ = scene_geometry(Z)
    writer.writerow([
        f"{Z:.0f}",
        f"{width:.1f}",
        f"{gsd:.4f}",
        f"{sigmaZ:.3f}"
    ])
```

### Example output (`geometry_table.csv`)

| Z (mm) | Scene Width (mm) | GSDx (mm/px) | σZ (mm) |
| :----: | :--------------: | :----------: | :-----: |
|   500  |       400.0      |    0.3125    |   0.52  |
|   600  |       480.0      |    0.3750    |   0.75  |
|   700  |       560.0      |    0.4375    |   1.02  |
|   800  |       640.0      |    0.5000    |   1.33  |
|   900  |       720.0      |    0.5625    |   1.69  |
|  1000  |       800.0      |    0.6250    |   2.08  |
|  1100  |       880.0      |    0.6875    |   2.52  |
|  1200  |       960.0      |    0.7500    |   3.00  |

> Values above correspond to:
> `Ws=2.88 mm`, `Nx=1280 px`, `f=3.6 mm`, `b=60 mm`, `σd=0.2 px`
>
> You can adjust these at the top of the script for any sensor/lens/baseline combination.

---



