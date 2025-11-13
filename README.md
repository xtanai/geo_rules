# Vision Geometry Rules

A concise, practical set of formulas for camera FOV, coverage, pixel scale, fisheye rectification, and depth (Z) resolution. Drop this right into your GitHub repo.

## 1) Notation

* (W_s, H_s) — sensor width/height (mm)
* (N_x, N_y) — image resolution (px)
* (p_x = \frac{W_s}{N_x}), (p_y = \frac{H_s}{N_y}) — pixel pitch (mm/px)
* (f) — focal length (mm)
* (\mathrm{HFOV}, \mathrm{VFOV}, \mathrm{DFOV}) — horizontal/vertical/diagonal field of view (radians or degrees)
* (Z) — working distance to object plane (mm)
* (b) — stereo baseline (mm)
* (f_{px} = \frac{f}{p_x}) — focal length in pixel units (px)
* (\theta) — incident angle from optical axis (radians)
* (r) — image radius from principal point (px or mm)

> ⚠️ **Sensor size** (e.g., “1/4 inch”) is an optical format label. Check the **datasheet** for exact (W_s, H_s). Typical 1/4" is ≈ **2.88 mm × 2.16 mm** (diag ≈ 3.6 mm).

---

## 2) FOV ↔ focal length (rectilinear model)

**Horizontal FOV**
[
\mathrm{HFOV} = 2 \arctan!\left(\frac{W_s}{2f}\right)
]

**Vertical FOV**
[
\mathrm{VFOV} = 2 \arctan!\left(\frac{H_s}{2f}\right)
]

**Solve for focal length** (given HFOV):
[
f = \frac{W_s}{2 \tan(\mathrm{HFOV}/2)}
]

**Scene coverage** at distance (Z):
[
\text{scene_width} = 2 Z \tan!\left(\frac{\mathrm{HFOV}}{2}\right),\quad
\text{scene_height} = 2 Z \tan!\left(\frac{\mathrm{VFOV}}{2}\right)
]

---

## 3) Pixel scale (mm/px) & object resolution

**Ground Sampling Distance (GSD)** at distance (Z):
[
\mathrm{GSD}_x = \frac{\text{scene_width}}{N_x},\quad
\mathrm{GSD}_y = \frac{\text{scene_height}}{N_y}
]
This is your **mm per pixel** in the object plane.

If you know a feature size (S) (mm) and want pixels it spans:
[
\text{pixels}_x \approx \frac{S}{\mathrm{GSD}_x}
]

---

## 4) Fisheye / lens projection models (radius ↔ angle)

Different wide-angle lenses map angle (\theta) to radius (r) differently. Common models:

* **Rectilinear**: ( r = f \tan\theta )
* **Equidistant (fisheye)**: ( r = f ,\theta )
* **Equisolid-angle**: ( r = 2f \sin(\theta/2) )
* **Orthographic**: ( r = f \sin\theta )
* **Stereographic**: ( r = 2f \tan(\theta/2) )

If your lens is fisheye (often a circular image), you can **rectify to rectilinear**:

1. Pick the right fisheye model (datasheet).
2. Invert to get (\theta) from fisheye radius (r_{fe}): e.g. equidistant (\Rightarrow \theta = r_{fe}/f_{fe}).
3. Map to rectilinear radius (r_{rect} = f_{rect}\tan\theta).
4. Use this to resample pixels from fisheye image to a rectilinear grid (straight lines become straight).

> In practice you calibrate (f), principal point, and distortion coefficients, then use a standard undistortion/rectification step (OpenCV: `fisheye::undistortImage`, or remap via your model). The equations above explain the **radius adjustment**: you’re converting a curved fisheye “circle” into a straight rectilinear coordinate using (\theta) as the bridge.

---

## 5) Z-resolution (depth precision)

### 5.1 Stereo triangulation

Depth from disparity (d) (px) with baseline (b) and focal in pixels (f_{px}):
[
Z = \frac{b, f_{px}}{d}
]

Propagating disparity noise (\sigma_d) gives **depth uncertainty**:
[
\sigma_Z \approx \frac{Z^2}{b, f_{px}}, \sigma_d
]
Key levers to **improve Z-resolution**:

* Increase **baseline (b)**
* Increase **focal in px** ((f_{px} = f/p_x) → larger (f), smaller pixel pitch)
* Reduce **(\sigma_d)** (better matching, texture, illumination, subpixel refinement)

### 5.2 Single-camera with known scale (planar target)

If you know the target is on a plane at (Z) and you measure apparent size (S') (px) of a known object size (S) (mm):
[
S' \approx \frac{f_{px}}{Z} S ;;\Rightarrow;; Z \approx \frac{f_{px}, S}{S'}
]
Depth precision then follows from measurement noise on (S'); this is more limited than true stereo/triangulation.

---

## 6) Worked example (1/4″ sensor, 1280×800, (f=3.6) mm)

**Assumptions** (typical, check your datasheet):
(W_s = 2.88) mm, (H_s = 2.16) mm, (N_x=1280), (N_y=800), (f=3.6) mm.

**FOV**
[
\mathrm{HFOV} = 2\arctan!\left(\frac{2.88}{2\cdot 3.6}\right)
= 2\arctan(0.4) \approx 43.6^\circ
]
[
\mathrm{VFOV} = 2\arctan!\left(\frac{2.16}{2\cdot 3.6}\right)
= 2\arctan(0.3) \approx 33.4^\circ
]

**Coverage** at (Z=800) mm:
[
\text{scene_width} = 2\cdot 800 \cdot \tan(43.6^\circ/2)
= 1600 \cdot \tan(21.8^\circ) \approx 1600 \cdot 0.4 \approx 640\text{ mm}
]
[
\mathrm{GSD}_x = \frac{640\text{ mm}}{1280} \approx 0.50\text{ mm/px}
]

**Z-resolution (stereo)** with (b=60) mm, pixel pitch (p_x=W_s/N_x=2.88/1280=0.00225) mm/px (=) 2.25 µm →
(f_{px} = f/p_x = 3.6/0.00225 \approx 1600) px.
For subpixel disparity noise (\sigma_d=0.2) px:
[
\sigma_Z \approx \frac{Z^2}{b, f_{px}}, \sigma_d
= \frac{800^2}{60\cdot 1600} \cdot 0.2
\approx 1.33\text{ mm}
]

---

## 7) Quick reference (useful trig)

* (\tan^{-1}(x) = \arctan(x))
* Small-angle: (\tan\theta \approx \theta) (radians)
* Conversions: ( \text{deg} = \text{rad}\cdot \frac{180}{\pi}), ( \text{rad} = \text{deg}\cdot \frac{\pi}{180} )

---

## 8) Practical tips

* Treat **FOV numbers (90°, 110°, 60°)** as rough; **sensor size + focal length** give you the true geometry.
* Always use **datasheet dimensions** for (W_s,H_s) and **calibrated** (f) (in px) for precision work.
* For fisheye or strong distortion, **rectify first**, then do coverage/GSD math in the rectified (rectilinear) frame.
* For target distances **500–1200 mm**, pre-compute a table of ({\text{scene_width}, \mathrm{GSD}, \sigma_Z}) to choose lens/baseline.

---

## 9) Minimal “design formulas” (copy/paste)

**Given** (W_s,H_s,f,Z,N_x,N_y)

```text
HFOV = 2 * atan(Ws / (2*f))
VFOV = 2 * atan(Hs / (2*f))

scene_width  = 2 * Z * tan(HFOV/2)
scene_height = 2 * Z * tan(VFOV/2)

GSDx = scene_width  / Nx
GSDy = scene_height / Ny

px = Ws / Nx
fpx = f / px

# Stereo depth precision (disparity noise = sigma_d)
sigma_Z = (Z*Z / (b * fpx)) * sigma_d
```

**Fisheye → Rectilinear (equidistant model)**

```text
# fisheye (equidistant): r_fe = f_fe * theta
theta = r_fe / f_fe
r_rect = f_rect * tan(theta)
```

Swap the projection law if your lens is equisolid/orthographic/stereographic.

