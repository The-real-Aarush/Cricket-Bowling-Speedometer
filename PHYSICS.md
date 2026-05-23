# Cricket Speedometer — Physics & Methodology

> A self-calibrating ball speed estimator using only a smartphone camera and Newtonian kinematics.

---

## Table of Contents

1. [Overview](#overview)
2. [Core Concept](#core-concept)
3. [The Calibration Problem](#the-calibration-problem)
4. [Gravity as a Scale — The Key Insight](#gravity-as-a-scale--the-key-insight)
5. [Full Physics Derivation](#full-physics-derivation)
6. [Error Analysis](#error-analysis)
7. [Design Decisions & Rejected Approaches](#design-decisions--rejected-approaches)
8. [App Design & UX Considerations](#app-design--ux-considerations)
9. [Future Work](#future-work)

---

## Overview

This project estimates the speed of a cricket delivery using only a smartphone camera filming at 120 fps. No radar gun, no external sensors, no reference objects — just video and physics.

The method works by:
- Tracking the ball's pixel position across frames
- Using the **curvature of the ball's parabolic arc** under gravity to extract a real-world scale (metres per pixel)
- Converting horizontal pixel velocity into real-world speed in km/h

Estimated accuracy: **±5–8 km/h** at typical throwing speeds (~80 km/h), comparable to consumer radar guns.

---

## Core Concept

A camera filming at 120 fps captures a frame every:

$$\Delta t = \frac{1}{120} \approx 8.33 \text{ ms}$$

By tracking the ball's (x, y) pixel position across frames, we can compute velocity using:

$$v = \frac{\Delta x}{\Delta t}$$

The challenge is converting pixel distances into real-world metres — this requires a **px/metre scale factor**.

---

## The Calibration Problem

Most speed-from-video approaches require a known reference object in the scene (a ruler, a cricket stump, etc.) to establish the pixel-to-metre ratio. This is inconvenient and introduces human error in placement.

### First Idea: Freefall Time as a Reference

An early idea was to use the ball's release-to-bounce flight time and the freefall equation:

$$h = \frac{1}{2} g t^2$$

Counting frames from release to bounce gives `t`, and solving for `h` gives a real-world length to calibrate against. Elegant in theory — but it fails because the ball is **not in pure freefall**. A cricket delivery imparts an initial vertical velocity component `vᵧ`, making the true equation:

$$h = v_y t + \frac{1}{2} g t^2$$

Ignoring `vᵧ` would systematically overestimate `h` and corrupt the calibration. This approach was abandoned.

---

## Gravity as a Scale — The Key Insight

Rather than using a fixed time interval, we can extract the scale factor from the **shape of the parabolic arc itself**.

Gravity always accelerates the ball downward at exactly:

$$g = 9.8 \text{ m/s}^2$$

In pixel space, we can measure the ball's **vertical pixel acceleration** `g_px` (in px/frame²) by fitting a quadratic to the vertical position data. Since we know what that acceleration *should be* in the real world, we can directly compute the scale:

$$\text{scale} = \frac{g}{g_{px} \times f^2}$$

Where `f = 120` fps. This gives us metres per pixel with **no physical reference object needed**. The throw calibrates itself.

---

## Full Physics Derivation

### Equations of Motion

After release, the ball follows projectile motion:

**Horizontal:**
$$x(t) = v_x \cdot t$$

**Vertical:**
$$y(t) = v_{y} \cdot t + \frac{1}{2} g t^2$$

Where `vₓ` is horizontal speed and `vᵧ` is initial vertical velocity (positive = upward).

### Extracting the Scale from Vertical Motion

Fitting a quadratic of the form `y = at² + bt + c` to the tracked vertical pixel positions gives:

- `a` = ½ × pixel acceleration = ½ × `g_px`
- So `g_px = 2a` (in px/frame²)

The real-world scale is then:

$$\text{metres per pixel} = \frac{g}{2a \times f^2}$$

### Computing Speed

Once the scale is known, horizontal speed is:

$$v_x = \frac{\Delta x_{px}}{\Delta t} \times \text{scale}$$

And the true delivery speed accounting for any vertical component:

$$v = \sqrt{v_x^2 + v_y^2}$$

`vᵧ` can be recovered from the linear term `b` of the quadratic fit:

$$v_y = b \times f \times \text{scale}$$

### Summary Pipeline

```
Track (x, y) pixel positions across N frames
        ↓
Fit quadratic to y positions → extract pixel acceleration g_px
        ↓
Compute scale: metres/px = g / (g_px × f²)
        ↓
Compute vₓ from horizontal pixel displacement × scale × f
Compute vᵧ from linear term of quadratic fit × scale × f
        ↓
True speed: v = √(vₓ² + vᵧ²)
        ↓
Convert to km/h: v × 3.6
```

---

## Error Analysis

All estimates assume a true throwing speed of ~80 km/h and a camera distance of ~5 m.

| Error Source | Mechanism | Estimated Error |
|---|---|---|
| Pixel tracking noise | Errors in position amplified by double differentiation | ±3–5 km/h |
| Depth drift | Ball moving toward/away from camera during throw | ±2–3 km/h |
| Air resistance | Drag slightly reduces effective vertical acceleration | ±1–2 km/h |
| Camera angle | Off-perpendicular filming foreshortens horizontal distance | ±0.5–1 km/h |
| Lens distortion | Barrel distortion warps pixel distances near frame edges | ±0.5–1 km/h |

**Combined (root sum of squares):**

$$\sigma_{total} = \sqrt{4^2 + 2.5^2 + 1.5^2 + 1^2} \approx \pm 5 \text{ km/h}$$

Realistic range under good conditions: **±5–8 km/h**.

### Mitigation Strategies

- **Tracking noise** — fit quadratic over as many frames as possible rather than using point-to-point differences
- **Depth drift** — film strictly side-on; throw parallel to the camera plane
- **Lens distortion** — keep the ball's path through the centre of the frame; avoid ultra-wide angle mode
- **Air resistance** — acceptable at backyard speeds; negligible below 120 km/h

---

## Design Decisions & Rejected Approaches

### Why Not a Physical Reference Object?
Requires the user to place a known-length object at exactly the right depth — error-prone and inconvenient. Gravity-as-scale is more robust and requires nothing extra.

### Why Not Use Freefall Time (Release to Bounce)?
Initial vertical velocity from the throw contaminates the measurement. See [The Calibration Problem](#the-calibration-problem).

### Why Not MediaPipe Pose Estimation?
Considered using MediaPipe to track arm/wrist landmarks for release point detection and foot landmarks as a ground reference. Rejected because:
- Fast bowling arm motions cause frequent landmark loss at the critical release frame
- Body proportions vary per user — no reliable known length without extra calibration
- Running pose estimation alongside ball tracking is computationally heavy on mobile and risks frame drops, which would corrupt frame-count timing
- The gravity-scale method already solves calibration more elegantly

### Why 120 fps?
Higher frame rate means more data points per flight, reducing the impact of individual tracking errors on the quadratic fit. It also reduces motion blur per frame. 60 fps is a fallback but 120 fps is strongly preferred.

---

## App Design & UX Considerations

### Ball Detection Calibration
Before throwing, the user holds the ball up to the camera. The app samples the ball's HSV colour profile in that specific lighting condition, storing it as the tracking target for the session. This makes tracking robust to varying environments without any manual input.

### Background Conflict Check
After colour sampling, the app checks whether the ball's colour is sufficiently distinct from the background. A tennis ball thrown in front of a grass lawn is a known failure mode — the user is warned before wasting a throw.

### Failure Mode Handling

| Failure | User Message |
|---|---|
| Ball lost mid-flight | "Ball lost — try clearer background or better lighting" |
| Ball exits frame | "Keep full throw in frame — step further back" |
| Too few frames tracked | "Flight path too short — try a longer throw" |
| Quadratic fit poor (low R²) | "Tracking unreliable — check lighting and background" |

### Camera Setup Guidance
The app should guide the user to film side-on with the throw path running left-to-right through the centre of the frame. A simple overlay diagram on the setup screen removes ambiguity.

### Output Display
Speed is shown with an explicit confidence interval — e.g. **"78 km/h ± 6"** — to set honest expectations and add credibility.

---

## Future Work

- [ ] Automated ball detection via OpenCV colour segmentation
- [ ] Quadratic fit quality score (R²) as a per-attempt confidence metric
- [ ] Support for 240 fps for higher accuracy
- [ ] Investigate whether lens distortion correction improves accuracy meaningfully
- [ ] Mobile app wrapper (React Native or Swift/Kotlin)
- [ ] Multi-throw session averaging and personal speed history

---

*Documented from experimental design process. All physics derivations are original. Error estimates are theoretical — empirical validation pending.*
