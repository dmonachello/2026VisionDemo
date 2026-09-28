# Phase 1 and Phase 2 geometry

This document defines the geometry for camera-relative AprilTag measurement. It is the reference for Phase 1 position output and Phase 2 horizontal guidance.

The formulas use one input: PhotonVision's best camera-to-tag `Transform3d` for the requested AprilTag. Robot pose, camera height, floor level, odometry, and gyro data do not enter these calculations.

## Coordinate frames

PhotonVision converts its camera result to the WPILib coordinate convention:

- Camera `+X` points forward along the optical axis.
- Camera `+Y` points left.
- Camera `+Z` points up.
- A positive rotation about camera `+Z` turns camera `+X` toward camera `+Y`. The human direction is LEFT.
- A negative rotation about camera `+Z` turns toward camera `-Y`. The human direction is RIGHT.

```text
Camera frame

                         +Z up
                          |
                          |
                          o--------> +X forward
                         /
                       +Y left

Top view, +Z toward the reader

                         +Y left
                            ^
                            |
                 camera  C--+-----> +X forward
```

The AprilTag frame has its origin at the tag center:

- Tag `+X` is normal to the printed surface and points out of the visible face.
- Tag `+Y` points right when the viewer faces the visible side.
- Tag `+Z` points up.

The camera and tag `+X` axes point in opposite directions during a centered, head-on observation. PhotonVision therefore reports a 180 degree rotation about `Z`, not a zero rotation.

```text
Centered, head-on top view

camera +X  ------------------------------------>
     C                                               T
                                                     <----  tag +X outward

translation = (distance, 0, 0)
rotation    = Rz(180 degrees)
```

## Transform meaning

This document names the PhotonVision result `cameraToTag` because that is the API name. Its precise geometric meaning is the pose of the tag expressed in the camera frame.

```text
cameraToTag = [ R  t ]
              [ 0  1 ]

t = (x, y, z)
```

The translation `t` locates the tag center in camera coordinates. The rotation `R` expresses the tag axes in camera coordinates. Each column of `R` is one tag unit axis expressed in the camera frame.

This meaning resolves a common naming ambiguity. The transform is not the pose of the camera expressed in the tag frame. That reverse transform is `cameraToTag.inverse()`.

## Internal units

All geometry uses meters and radians. Only the console presenter converts distances to inches and angles to degrees.

Rounding and human direction words do not change the stored values.

## Phase 1 position measurements

For `t = (x, y, z)`:

```text
forwardMeters = x
leftMeters    = y
upMeters      = z
rangeMeters   = hypot(hypot(x, y), z)
```

The formatter selects a label from each sign and prints the absolute magnitude:

| Component | Positive label | Negative label |
|---|---|---|
| `x` | Forward | Back |
| `y` | Left | Right |
| `z` | Up | Down |

Range is always nonnegative. It is the distance from the camera focal point to the tag center, not the forward component.

## Phase 2 horizontal reference

"Horizontal" means the camera's local XY plane. LEFT and RIGHT mean rotation about the camera's local `+Z` axis.

This definition is required for handheld operation. The camera-to-tag transform contains no gravity reference, so it cannot produce a room-level or field-level heading when the operator rolls the camera.

The two Phase 2 angles are signed corrections:

- Positive correction means turn LEFT.
- Negative correction means turn RIGHT.
- Zero means aligned for that measurement.

Both angles use the interval `(-pi, +pi]`. An exact 180 degree tie is reported as LEFT by convention. The geometry layer does not apply a display deadband.

## Point-at-tag correction

The point-at-tag correction turns camera `+X` toward the tag center after both are projected into the camera XY plane.

```text
horizontalCenterRange = hypot(x, y)
pointAtTagRadians      = wrapToPi(atan2(y, x))
```

Positive `y` produces a positive correction, so a tag center on the left produces LEFT guidance. The vertical component `z` does not affect this horizontal angle.

If `horizontalCenterRange` is zero within numerical precision, the horizontal bearing is undefined. The measurement is `UNRELIABLE`; the software must not turn `atan2(0, 0)` into an aligned result.

PhotonVision's reported target yaw is diagnostic data. It does not replace this translation-derived calculation.

```text
Top view, tag center left

                       T tag center
                      /
                     /  center ray
                    /
camera C -----------+----------------> camera +X

point-at-tag correction is LEFT
```

## Square-with-tag correction

The square-with-tag correction comes from the tag face orientation. It does not use the tag center translation.

Start with the tag's outward unit normal in tag coordinates:

```text
tagOutward = (1, 0, 0)
```

Rotate that vector into camera coordinates:

```text
n = R * tagOutward
n = (nx, ny, nz)
```

The direction that a square camera must face is opposite the outward normal:

```text
inward = -n
```

Project `inward` into the camera XY plane and take its signed heading:

```text
horizontalNormalRange = hypot(nx, ny)
squareWithTagRadians  = wrapToPi(atan2(-ny, -nx))
```

For the documented head-on rotation, `n = (-1, 0, 0)`. The inward direction is `(1, 0, 0)`, so the correction is zero.

The implementation must rotate the unit normal. It must not calculate this value as raw target yaw, raw `Rotation3d.getZ()`, or `Rotation3d.getZ() - pi`. Those shortcuts depend on an Euler-angle decomposition and can fail when the observation includes pitch or roll.

In WPILib Java, the normal calculation has this shape:

```java
Translation3d outwardNormalInCamera =
    new Translation3d(1.0, 0.0, 0.0)
        .rotateBy(cameraToTag.getRotation());
```

If `horizontalNormalRange` is zero within numerical precision, no horizontal square direction exists. The measurement is `UNRELIABLE`.

## What square means in Phase 2

The Phase 2 value is a horizontal correction. It aligns camera `+X` with the XY projection of the tag's inward normal.

A horizontal turn cannot remove a remaining vertical tilt. A camera that is pitched or rolled may still be nonperpendicular to the tag after the reported horizontal correction reaches zero. Full 3D orientation guidance would require more output angles and is outside Phase 2.

The console may use `Square horizontally with tag` if physical testing shows that `Square with tag` implies full 3D alignment to operators.

## The two angles remain independent

Translation determines the point-at-tag correction. Rotation determines the square-with-tag correction.

```text
                         T tag center
                        /
                       /  center direction
                      /
camera C -------------+----------------> camera +X
          \
           \  projected tag inward normal

The center direction answers "where is the tag?"
The inward normal answers "which way does its face point?"
```

Moving the camera without changing relative face orientation can change only the point-at-tag correction. Rotating the tag around its center can change only the square-with-tag correction. No special case is needed when both change.

## Worked arrangements

The following values are executable test vectors. Distances in the input columns are meters.

| Arrangement | Translation `(x, y, z)` | Tag outward normal in camera frame `(nx, ny, nz)` | Range | Point at tag | Square with tag |
|---|---:|---:|---:|---:|---:|
| Centered and head-on | `(2.000, 0.000, 0.000)` | `(-1.000, 0.000, 0.000)` | `2.000 m` | `0.00 deg` | `0.00 deg` |
| Tag center left and up, faces remain parallel | `(2.000, 0.500, 0.100)` | `(-1.000, 0.000, 0.000)` | `2.064 m` | `14.04 deg LEFT` | `0.00 deg` |
| Centered, tag face yawed | `(2.000, 0.000, 0.000)` | `(-0.940, 0.342, 0.000)` | `2.000 m` | `0.00 deg` | `20.00 deg RIGHT` |
| Right, up, and face yawed | `(2.000, -0.500, 0.250)` | `(-0.966, -0.259, 0.000)` | `2.077 m` | `14.04 deg RIGHT` | `15.00 deg LEFT` |
| Three-dimensional face tilt | `(2.000, 0.200, 0.400)` | `(-0.750, -0.433, 0.500)` | `2.049 m` | `5.71 deg LEFT` | `30.00 deg LEFT` |

### Centered and head-on

```text
TAG 7                  VALID

Forward:   78.7 in
Left:       0.0 in
Up:         0.0 in
Range:     78.7 in

Point at tag:          ALIGNED
Square with tag:       ALIGNED
```

### Tag center left and up

The camera and tag faces remain parallel. Only the center direction changes.

```text
TAG 7                  VALID

Forward:   78.7 in
Left:      19.7 in
Up:         3.9 in
Range:     81.3 in

Point at tag:     14.0 deg LEFT
Square with tag:       ALIGNED
```

### Centered with an angled tag face

The tag center lies on the optical axis. Only the face direction changes.

```text
TAG 7                  VALID

Forward:   78.7 in
Left:       0.0 in
Up:         0.0 in
Range:     78.7 in

Point at tag:          ALIGNED
Square with tag:  20.0 deg RIGHT
```

### Displaced and rotated

The corrections point in opposite directions. This arrangement catches any implementation that reuses one angle for both outputs.

```text
TAG 7                  VALID

Forward:   78.7 in
Right:     19.7 in
Up:         9.8 in
Range:     81.8 in

Point at tag:     14.0 deg RIGHT
Square with tag:  15.0 deg LEFT
```

### Three-dimensional face tilt

The projected inward normal points 30 degrees left, so the horizontal square correction is 30 degrees LEFT. Its vertical component remains. Phase 2 does not claim that this one turn makes the camera fully perpendicular to the tag.

```text
TAG 7                  VALID

Forward:   78.7 in
Left:       7.9 in
Up:        15.7 in
Range:     80.7 in

Point at tag:      5.7 deg LEFT
Square with tag:  30.0 deg LEFT
```

## Validity rules that affect geometry

The existing measurement states remain:

- `NO TARGET` means the requested tag ID is absent from the current result.
- `UNRELIABLE` means the requested tag exists, but the observation fails a quality or geometry check.
- `VALID` means the requested tag exists and all configured checks pass.

The initial ambiguity rule remains unchanged:

```text
pose ambiguity > 0.20   -> UNRELIABLE
pose ambiguity <= 0.20  -> candidate for VALID
```

Nonfinite transform components and undefined horizontal projections also produce `UNRELIABLE`. These checks prevent undefined math from appearing as guidance. They are not empirical accuracy thresholds.

Only a `VALID` result exposes actionable distances and corrections. `NO TARGET` never carries an old transform. `UNRELIABLE` may retain raw values for diagnostics, but the console does not display them as movement instructions.

## Delivery modes do not change geometry

`CONTINUOUS` and `SNAPSHOT` control when consumers receive a `TagMeasurement`. Both modes use the same target selection, validity rules, and geometry formulas.

The public `setDeliveryMode` API changes the mode. `deliveryMode` reports the active value. Changing modes does not publish a measurement.

The measurement pipeline continues to process camera results in both modes:

- `CONTINUOUS` delivers each new processed camera result.
- `SNAPSHOT` delivers the current result once when the Xbox X button changes from released to pressed.

Holding X does not repeat a snapshot. A snapshot retains its capture timestamp. Future drivetrain code must consume it as one update rather than treating the retained value as fresh on every robot loop.

## Required geometry tests

The implementation tests must cover these invariants:

1. `Rz(180 degrees)` produces zero square correction.
2. Positive translation `Y` produces LEFT point-at-tag guidance.
3. Negative translation `Y` produces RIGHT point-at-tag guidance.
4. Moving only the tag center changes bearing but not square correction.
5. Rotating only the tag changes square correction but not bearing.
6. Changing only `Z` changes vertical displacement and range but not either horizontal angle.
7. A rotation with pitch or roll matches the rotated-normal formula and does not use Euler yaw.
8. The angle wrap is deterministic on both sides of 180 degrees.
9. Undefined center and normal projections produce `UNRELIABLE`.
10. The five worked arrangements above match their stated outputs.

Physical testing must verify the signs before it establishes accuracy tolerances. The test fixture must record which side is camera-left, which face of the tag is visible, and which direction counts as positive rotation.

## Sources

- [PhotonVision coordinate systems](https://docs.photonvision.org/en/latest/docs/apriltag-pipelines/coordinate-systems.html)
- [PhotonLib target data](https://docs.photonvision.org/en/latest/docs/programming/photonlib/getting-target-data.html)
- [PhotonVision 3D tracking and ambiguity](https://docs.photonvision.org/en/latest/docs/apriltag-pipelines/3D-tracking.html)
- [WPILib `Transform3d`](https://github.wpilib.org/allwpilib/docs/release/java/edu/wpi/first/math/geometry/Transform3d.html)
- [WPILib `Rotation3d`](https://github.wpilib.org/allwpilib/docs/release/java/edu/wpi/first/math/geometry/Rotation3d.html)
- [WPILib `Translation3d`](https://github.wpilib.org/allwpilib/docs/release/java/edu/wpi/first/math/geometry/Translation3d.html)
- [WPILib `MathUtil.angleModulus`](https://github.wpilib.org/allwpilib/docs/release/java/edu/wpi/first/math/MathUtil.html)
