---
created: 2026-11-14
tags: [robotics, computer-vision, apriltag, pose-estimation, homography, svd, camera-calibration]
source: https://claude.ai/artifact/Dr7zAkaEAY9S8YZTa6wJj2
---

# AprilTag Pose Estimation

## Summary

When a camera sees an AprilTag, it can recover the tag's **ID** and its **6-DoF pose** (position `t` and orientation `R`) relative to the camera. The pipeline:

1. Detect the tag's four corners and read their pixel coordinates.
2. Use the four point correspondences to solve for the homography `H` (via SVD).
3. Use the intrinsics `K` to decompose `H` into `R` and `t`.
4. Use `H` to sample the inner black/white cells and decode the ID.
5. Chain the camera-to-tag pose back to the robot base frame.

> The original artifact has an interactive lab (sliders for focal length, translation, rotation and pixel noise). Open the source link in the frontmatter to play with it.

Three concepts to keep apart:

| Concept | What it is | When it changes | Where it comes from |
|---|---|---|---|
| Camera frame | Origin at the optical center; Z forward, X right, Y down | When the camera moves | Convention |
| Intrinsics `K` | Property of the camera: which pixel a ray lands on | Never (only if lens or resolution changes) | Camera calibration |
| Extrinsics `[R \| t]` | Rigid transform between two frames; here, tag relative to camera | Whenever tag or camera moves | Computed live by AprilTag |

## Key Ideas

### 1. Frames and extrinsics

The same point has different coordinates in different frames. The extrinsics are the conversion rule:

```
P_c = R · P_tag + t
```

| Symbol | Meaning | Unit | Example |
|---|---|---|---|
| `P_tag` | Point in the tag frame. Origin at tag center, X right, Y down, Z into the tag | m | `(0.025, 0, 0)` |
| `R` | 3×3 rotation. Its columns `r1 r2 r3` are the tag's X, Y, Z axes expressed in the camera frame | – | Identity when facing the camera |
| `t` | Tag center in the camera frame | m | `(0.08, 0, 0.40)` |
| `P_c` | Same point in the camera frame | m | `(0.105, 0, 0.40)` |

**Reading `R`: look at the columns.** Column 1 answers "where does the tag's X axis point in the camera frame?" E.g. `r1 = (0, 1, 0)` means the tag's X axis points along the camera's Y (down), i.e. the tag is rotated 90° in-plane.

**4×4 form.** Stack `R` and `t` into a homogeneous transform so that changing frames is one matrix multiply and chains are just products:

```
T_cam_tag = [ R  t ]      read as "pose of tag in the cam frame"
            [ 0  1 ]
```

### 2. Pinhole camera model and intrinsics `K`

Pinhole rule: the object point, the optical center and the image point are collinear. Similar triangles give `image height = f · Xc / Zc`, so the same object looks smaller the farther away it is.

**Step 1: divide by depth → normalized coordinates**

```
x = Xc / Zc,   y = Yc / Zc
```

`(x, y)` is unitless and only describes the ray direction ("where the ray hits the Z = 1 plane"). It is independent of focal length and resolution.

**Step 2: apply `K` → pixels**

```
u = fx · x + cx,   v = fy · y + cy

K = [ fx  0  cx ]
    [ 0  fy  cy ]
    [ 0   0   1 ]
```

| Symbol | Meaning | Unit |
|---|---|---|
| `fx, fy` | Focal length in pixels = physical focal length / pixel size. Larger → narrower FOV, bigger objects | px |
| `cx, cy` | Principal point: where the optical axis hits the image, usually near the center. Needed because the pixel origin is top-left | px |
| `u, v` | Pixel coordinates, u right, v down | px |

**Worked example** (`fx = fy = 400`, `cx = 160`, `cy = 120`): point `(0.08, 0, 0.40)` → `x = 0.2, y = 0` → `u = 400·0.2 + 160 = 240`, `v = 120`.

- **Is `K` the normalization?** The other way round. `K` maps normalized coords → pixels; `K⁻¹` maps pixels → normalized coords (removes per-camera differences, leaves only the ray direction).
- **Do we know `K⁻¹`?** Yes. `K` comes from calibration (checkerboard + OpenCV, the driver e.g. ROS `camera_info`, or given in a course project). Its inverse is closed-form:

```
K⁻¹ = [ 1/fx   0    -cx/fx ]
      [  0    1/fy  -cy/fy ]      i.e.  x = (u - cx)/fx,  y = (v - cy)/fy
      [  0     0      1    ]
```

- **Depth is lost** after dividing by `Zc`: a pixel only gives a ray. AprilTag recovers depth because the **real tag side length is known**.

### 3. Full projection equation

```
    [u]       [r11 r12 r13 tx]   [X]
s · [v] = K · [r21 r22 r23 ty] · [Y]
    [1]       [r31 r32 r33 tz]   [Z]
                                 [1]
```

Read right to left, in the order image formation happens:

| Part | Meaning | Unit |
|---|---|---|
| `[X, Y, Z, 1]ᵀ` | Point in the tag frame; the trailing 1 makes "+ t" a matrix multiply (homogeneous coords) | m |
| `[R \| t]` | 3×4 extrinsics → `(Xc, Yc, Zc)` | R unitless, t in m |
| `K` | 3×3 intrinsics → `(fx·Xc + cx·Zc, fy·Yc + cy·Zc, Zc)` | px |
| `s` | Equals depth `Zc`; divide by the third component to get `(u, v, 1)` | m |
| `[u, v, 1]ᵀ` | Homogeneous pixel coordinates | px |

Because of the final division, scaling the right side by any nonzero constant gives the same pixel. This is why `H` has only 8 DoF.

### 4. Homography `H`

The tag is planar, so put its frame on the plane: `Z = 0` for every point. Then `r3` always multiplies 0 and drops out:

```
s · [u v 1]ᵀ = K · [r1 r2 t] · [X Y 1]ᵀ = H · [X Y 1]ᵀ

H = K · [r1 r2 t]      (3×3, maps tag-plane meters → image pixels)
```

Tag frame `(X, Y, 0)` m → (extrinsics) → camera frame `(Xc, Yc, Zc)` m → (`K`, ÷Zc) → image `(u, v)` px. `H` fuses both steps, but only for points on the plane.

| Column of H | Equals | Meaning | How to see it in the lab |
|---|---|---|---|
| 1 | `K · r1` | Tag X axis, wrapped by K | Changes with in-plane rotation |
| 2 | `K · r2` | Tag Y axis, wrapped by K | Same |
| 3 | `K · t` | Plug in `(X, Y) = (0, 0)` → the tag center's pixel (after normalizing h33) | Changes with translation only, not rotation |

- **Why no tag Z axis in `H`?** No tag point ever leaves `Z = 0`, so `r3` never affects a pixel. But nothing is lost: rotation columns are orthonormal and right-handed, so `r3 = r1 × r2`.
- **Why 8 unknowns?** `H` has 9 entries, but `H` and `2H` give identical pixels. Fix the scale with `h33 = 1` or `‖h‖ = 1`.
- **What `H` is good for:** (1) **pose recovery**: `H` is solvable from just 4 correspondences and contains `r1`, `r2`, `t`; it is the "encrypted" pose and `K` is the key. (2) **ID decoding**: `H` tells you where any tag-plane point lands in the image.

Things to try in the interactive lab:

- **Change focal length only**: extrinsics unchanged, tag gets bigger. Intrinsics only govern ray → pixel.
- **Translate**: the 3rd column of `H` changes (it is the tag center's pixel).
- **Rotate only**: the 3rd column stays fixed (tag rotates about its own center); columns 1–2 change.
- **Zero all rotations**: `h31 = h32 = 0` and the image is a square. Tilt it and they become nonzero; the square becomes a trapezoid. Perspective lives in those two numbers.
- **Add pixel noise and move far away**: the recovered pose drifts, rotation more than translation.

### 5. Solving `H` from four corners (DLT)

Known vs unknown, the crux of the derivation:

| Quantity | Status | Why | Example |
|---|---|---|---|
| `X, Y` | Known | Corner coords in the tag frame; you printed the tag and measured its side | 5 cm tag → corners `(±0.025, ±0.025)` |
| `u, v` | Known | The detector finds corners and reads pixel coords | Red dots in the lab |
| `h11 … h33` | **Unknown** | Describes how the tag is placed; this is what we want | `H` |

You know the real shape `(X, Y)` and what it looks like in the photo `(u, v)`. The only unknown is how it is placed.

**One point gives two equations.** Expand and divide by the 3rd component:

```
u = (h11·X + h12·Y + h13) / (h31·X + h32·Y + h33)
```

Multiply out the denominator and move everything to one side:

```
h11·X + h12·Y + h13 - u·h31·X - u·h32·Y - u·h33 = 0
```

`X, Y, u` are numbers, so this is **linear in h**. Do the same for `v`:

```
[ X  Y  1  0  0  0  -uX  -uY  -u ]
[ 0  0  0  X  Y  1  -vX  -vY  -v ] · h = 0

h = [h11, h12, h13, h21, h22, h23, h31, h32, h33]ᵀ   (H flattened row-wise)
```

**Stack four points** → 8×9 matrix `A`, `A · h = 0`. 8 equations for 8 DoF, so **at least 4 points** are needed. `h = 0` is a trivial solution; finding the nonzero one needs SVD.

### 6. SVD review

```
A = U · Σ · Vᵀ
```

| Symbol | Shape (A is m×n) | Property | Meaning |
|---|---|---|---|
| `Vᵀ` | n×n | Orthogonal | Rotate input onto special directions |
| `Σ` | m×n | Diagonal, singular values `σ1 ≥ σ2 ≥ … ≥ 0` | Scale along each direction |
| `U` | m×m | Orthogonal | Rotate onto output directions |

Geometrically every linear map = rotate, scale along axes, rotate. Each column of `V` is an input direction; its singular value is how much `A` stretches it.

Small example: `A = [[2, 0], [0, 0.1]]` stretches x by 2 and squashes y to 0.1. Among unit vectors, `(1, 0)` is stretched most and `(0, 1)` least. For a general matrix the special directions are the columns of `V`.

**Use 1: solve `A · h = 0`.** Exclude `h = 0` by requiring `‖h‖ = 1` (scale is meaningless anyway). Minimize `‖A·h‖` over unit vectors. Answer: **the last column of `V`** (smallest singular value). Proof:

1. `‖A·h‖ = ‖U·Σ·Vᵀ·h‖ = ‖Σ·Vᵀ·h‖` since rotations preserve length.
2. Let `y = Vᵀ·h`, so `‖y‖ = 1` and `‖A·h‖² = σ1²y1² + … + σn²yn²`.
3. Minimize by putting all weight on `σn`: `y = (0, …, 0, 1)`, so `h = V·y` = last column of `V`.

With exactly 4 points, `A` is 8×9, so some direction must be squashed to 0: the smallest singular value is (numerically) 0 and the solution is exact. With more points (e.g. a calibration checkerboard), it is no longer 0 and the last column of `V` is the **least-squares** optimum. Same code for both.

**Use 2: project back onto a rotation matrix.** With noise, `r1` and `r2` are not exactly orthogonal, so `R̃` is not a valid rotation. SVD it, `R̃ = U·Σ·Vᵀ`, and replace `Σ` with identity:

```
R = U · Vᵀ
```

A valid rotation has all singular values = 1; dropping the "scale" part leaves the nearest rotation. If `det(R) = -1` (a reflection), flip the sign of the last column of `U`.

### 7. Recovering pose from `H`

`H = λ · K · [r1 r2 t]`, with `λ` the meaningless overall scale. Five steps:

1. **Remove K:** `M = K⁻¹ · H`; columns `m1, m2, m3` ≈ `λ·r1, λ·r2, λ·t`.
2. **Fix the scale:** rotation columns have unit length, so `‖m1‖` and `‖m2‖` should both equal `λ`; average them to reduce noise. If the resulting `tz < 0` the tag is behind the camera, so negate `λ`.
3. **Extract `r1, r2, t`:** divide each column by `λ`.
4. **Fill in `r3`:** `r3 = r1 × r2` (cross product). The Z axis was never lost.
5. **Orthonormalize:** check `r1 · r2 ≈ 0`; with noise, project to the nearest rotation via SVD (Use 2 above).

Worked example ("last screenshot" preset): column 1 of `H` is `(0, 1000, 0)`; `K⁻¹` gives `(0, 2.5, 0)`, so `λ = 2.5`, `r1 = (0, 1, 0)`, `t = (0.08, 0, 0.40)`.

**Why is `t` in real units (meters)?** Because the corner coords `(X, Y)` are in meters, so `H` maps meters → pixels and the decomposed `t` is in meters. Corollary: **if the tag size is wrong by a factor k, `t` is wrong by k, while `R` is unaffected.**

Real libraries (`apriltag`, `pupil-apriltags`) then run an iterative refinement minimizing corner reprojection error. You only supply `fx, fy, cx, cy` and the tag size.

### 8. Decoding the tag ID

The four corners are only used to solve `H`. The inner cells confirm "this really is a tag" and identify which one.

1. Split the tag interior into a grid. Map each cell center `(X, Y)` to pixels with `H`, sample black/white → a bit string.
2. Compare against the codebook by **Hamming distance**; accept if close enough.
3. Since the rotation is unknown, try 0°, 90°, 180°, 270°. The matching orientation also tells you which corner is "top-left".

`tag36h11`: 36 = 6×6 data bits; h11 = any two valid codewords differ in at least 11 bits. So up to 5 bit errors can still be corrected uniquely, and a random natural pattern almost never matches a valid codeword.

### 9. Pick-and-place transform chain

Stick a tag on the top face of a block and the camera gives you its pose and ID with no trained detector. But AprilTag gives the tag relative to the camera, and the arm needs the block relative to its base:

```
T_base_block = T_base_ee · T_ee_cam · T_cam_tag · T_tag_block
```

base →(forward kinematics)→ ee →(hand-eye calibration)→ cam →(AprilTag + K)→ tag →(known offset)→ block

| Transform | Source | Changes? |
|---|---|---|
| `T_base_ee` | Joint angles + forward kinematics | Every time the arm moves |
| `T_ee_cam` | Hand-eye calibration, measured once | Constant (camera mounted on the end effector) |
| `T_cam_tag` | Everything in this note. The only link measured live by vision, and the only place `K` is used | Every frame |
| `T_tag_block` | Tag at top-face center → translate half a block side along tag Z | Constant |

From `T_base_block`: position tells the gripper where to go; orientation (mainly yaw about vertical) tells it how much to rotate to align with the block faces. Then solve inverse kinematics for joint angles.

### 10. Limitations and debugging checklist

**Limitations of the method**

- **Rotation is noisier than translation.** One pixel of corner error barely moves `t` but noticeably moves `R`; error grows with distance.
- **Planar pose ambiguity.** When the tag is far or nearly fronto-parallel, two solutions have almost equal reprojection error and the tilt can "flip". Get closer / make the tag larger in the image.
- **Not occlusion-robust.** Blocking part of the border kills detection; motion blur and glare also cause failures.

**When grasps miss, check the chain link by link**

| Symptom | Most likely cause |
|---|---|
| Distance off by a constant ratio | Wrong tag size (it is the black border's side, excluding white margin), or mm vs m |
| Error grows away from image center | `K` inaccurate, or no distortion correction |
| Error varies with arm pose | Hand-eye calibration `T_ee_cam` is off |
| Constant position offset | Tag not centered on the block, or `T_tag_block` direction/sign wrong |
| Gripper yaw occasionally off by 90° or flipped | Pose ambiguity, or wrong multiplication order / inverse |

## Example / Code

```python
import numpy as np

def homography(tag_xy, pix_uv):          # two 4x2 arrays: meters, pixels
    A = []
    for (X, Y), (u, v) in zip(tag_xy, pix_uv):
        A.append([X, Y, 1, 0, 0, 0, -u*X, -u*Y, -u])
        A.append([0, 0, 0, X, Y, 1, -v*X, -v*Y, -v])
    _, _, Vt = np.linalg.svd(np.array(A))
    H = Vt[-1].reshape(3, 3)             # direction of the smallest singular value
    return H / H[2, 2]

def pose_from_H(H, K):
    M = np.linalg.inv(K) @ H             # = lambda * [r1 r2 t]
    lam = (np.linalg.norm(M[:, 0]) + np.linalg.norm(M[:, 1])) / 2
    if M[2, 2] / lam < 0:                # tag must be in front of the camera
        lam = -lam
    r1, r2, t = M[:, 0] / lam, M[:, 1] / lam, M[:, 2] / lam
    R = np.column_stack([r1, r2, np.cross(r1, r2)])
    U, _, Vt = np.linalg.svd(R)          # project to the nearest rotation
    R = U @ np.diag([1, 1, np.linalg.det(U @ Vt)]) @ Vt
    return R, t
```

## Symbol Cheat Sheet

| Symbol | Meaning | Unit | Status | Source |
|---|---|---|---|---|
| `(X, Y)` | Corner coords in the tag frame | m | Known | Measured tag size |
| `(u, v)` | Corner pixel coords | px | Known | Corner detection |
| `K` | Intrinsics `fx, fy, cx, cy` | px | Known | Camera calibration |
| `(x, y)` | Normalized coords = `K⁻¹` applied to pixels; ray direction | – | Intermediate | Computed |
| `s` | Scale factor in projection = depth `Zc` | m | Intermediate | Divided out |
| `A` | 8×9 coefficient matrix built from X, Y, u, v | mixed | Intermediate | Built from knowns |
| `h` | 9 entries of `H` flattened row-wise | mixed | Intermediate | Last column of `V` in SVD of `A` |
| `H` | Homography, `K·[r1 r2 t]` up to scale | mixed | Intermediate | Reshape `h` |
| `λ` | Overall scale of `H`, fixed by `‖r1‖ = 1` | – | Intermediate | `‖col 1 of K⁻¹H‖` |
| `U, Σ, V` | SVD factors: rotate, scale, rotate | – | Intermediate | `np.linalg.svd` |
| `R` | Tag orientation in camera frame; columns = tag axes | – | Unknown (solved) | Decompose `H` |
| `t` | Tag center in camera frame | m | Unknown (solved) | Decompose `H` |
| `T_a_b` | 4×4 rigid transform: pose of b in frame a | – / m | Depends | Built from `R`, `t` |

## Interview Questions

1. What is the difference between camera intrinsics and extrinsics? Which one does AprilTag estimate every frame?
2. Why does a homography have 8 degrees of freedom, and why do you need at least 4 point correspondences?
3. Derive the two DLT equations for one correspondence. Why is the system linear in `h`?
4. Why is the solution to `A·h = 0` (with `‖h‖ = 1`) the right singular vector of the smallest singular value?
5. How do you recover `R` and `t` from `H` and `K`? Why is `r3 = r1 × r2` valid?
6. How do you project a noisy 3×3 matrix to the nearest rotation matrix? What do you do if `det = -1`?
7. Why is the recovered `t` metric? What happens if the tag size is configured wrong?
8. What does `h11` mean in `tag36h11`, and how many bit errors can it correct?
9. Write the transform chain from robot base to a tagged block. Which links are constant and which change?
10. Your grasp position has an error that grows toward the image edges. What is the likely cause?

## Related

- Camera calibration, hand-eye calibration, forward / inverse kinematics (notes to be written)
