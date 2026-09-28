# FRC AprilTag Relative Measurement and Alignment

## Requirements and Design Specification v0.1

#chatgpt-generated

## 1. Purpose

Develop and validate a PhotonVision-based system that determines the three-dimensional relationship between a camera and a selected AprilTag.

The initial system is intended to work as a handheld measurement device. It will tell a human operator where an AprilTag is relative to the camera.

Later versions will use the same underlying camera-to-tag measurement with a camera rigidly mounted to an FRC robot. The final planned phase will use those measurements to position and orient a swerve-drive robot relative to an AprilTag.

The early handheld system is therefore not a separate application. It is a validation stage for the same vision geometry that will eventually be used for robot control.

---

## 2. Core Design Principle

The fundamental output of the vision system shall be the complete three-dimensional transform from the camera to the selected AprilTag.

Human-readable measurements, alignment instructions, robot-relative measurements, and eventual drivetrain commands shall be derived from this transform.

The core vision measurement shall not depend on whether the camera is:

* handheld,
* sitting on a test fixture,
* mounted on an FRC robot, or
* being used by a drivetrain controller.

The vision measurement layer shall not require knowledge of the camera's height above the floor or its orientation relative to the floor.

---

## 3. Technology

Initial implementation target:

* FRC/WPILib
* Java
* PhotonVision
* PhotonLib
* AprilTags

Camera and PhotonVision coprocessor hardware are TBD.

The camera must support calibration suitable for PhotonVision 3D AprilTag pose estimation.

---

# 4. Development Phases

## Phase 1 - Relative Position Measurement

Detect a specifically requested AprilTag and determine its three-dimensional position relative to the camera.

Output measurements to the console for a human operator.

No drivetrain is involved.

The camera may be handheld and moved freely.

## Phase 2 - Human Angular Guidance

In addition to Phase 1 position measurements, tell the human operator how the camera is oriented relative to the selected tag.

Two different angular relationships shall be reported:

1. the rotation required to point the camera toward the center of the tag;
2. the rotation required to make the camera square with the face of the tag.

These are separate measurements and shall not be treated as interchangeable.

No drivetrain is involved.

## Phase 3 - Robot-Mounted Measurement

Rigidly mount the camera on an FRC robot.

Introduce a known robot-to-camera transform.

Use:

* camera-to-tag transform, and
* robot-to-camera transform

to determine the robot's position and orientation relative to the selected AprilTag.

The system shall initially report this information without commanding drivetrain movement.

## Phase 4 - Swerve Guidance

Calculate the translation and rotation that would move the robot from its current pose relative to the tag to a specified desired pose relative to the tag.

The initial Phase 4 system should display the required movements rather than execute them.

Example conceptual output:

```
Move forward: 18.2 in
Move right:    6.4 in
Rotate left:   7.3 deg
```

## Phase 5 - Closed-Loop Swerve Alignment

Allow the swerve drivetrain to consume the relative-pose error and move the robot toward a specified desired pose relative to an AprilTag.

The desired pose may include:

* forward distance from the tag;
* lateral position relative to the tag;
* robot orientation relative to the tag.

Automatic drivetrain control is explicitly outside the scope of Phases 1 and 2.

## Measurement delivery modes

The system shall support `PERIODIC` and `SNAPSHOT` delivery modes.

The selected mode controls when a measurement update is delivered to the human-facing output or a future drivetrain consumer. It does not change camera acquisition, target selection, geometry, or validity rules.

The measurement component shall expose `setDeliveryMode(MeasurementDeliveryMode)` to select either mode and `deliveryMode()` to report the active mode.

Setting the active mode again shall have no effect. Changing modes shall not publish a measurement or trigger a snapshot by itself.

### PERIODIC

The measurement component shall expose `setPeriodicUpdateFrequencyHz(double)` and `periodicUpdateFrequencyHz()` to set and read the periodic update frequency in hertz.

The configured frequency shall be finite and greater than zero.

The frequency API shall be available in either delivery mode. Changing the frequency in `SNAPSHOT` mode shall store the value without publishing an update.

Changing the frequency in `PERIODIC` mode shall reset the periodic schedule. The first new camera result after the change shall be eligible immediately. Setting the current frequency again shall not reset the schedule.

`PERIODIC` shall deliver new processed camera results at no more than the configured frequency.

The configured frequency is a maximum delivery rate. The effective rate may be lower because it is limited by the camera frame rate, the robot loop rate, and processing time.

The system shall not treat repeated robot loops over the same camera result as new measurement updates.

The system shall not repeat an old camera result to meet the configured frequency. Every periodic update shall contain a newly processed camera result.

If execution falls behind the requested schedule, the system shall resume from the current time. It shall not emit a burst of delayed updates.

### SNAPSHOT

The camera and measurement pipeline shall continue to run while the system waits for a snapshot request.

Pressing the X button on the configured Xbox controller shall deliver one update containing the current measurement state. The update may be `VALID`, `NO TARGET`, or `UNRELIABLE`.

The X button shall act on its rising edge. Holding the button shall not produce repeated updates.

An X-button press outside `SNAPSHOT` mode shall have no effect and shall not remain queued for a later mode change.

No new human-facing or drivetrain update shall be delivered until the next X-button press.

A displayed snapshot may remain visible until the next snapshot, but it shall be labeled as a snapshot and retain its original capture timestamp. A future drivetrain consumer shall receive a snapshot as one timestamped update, not as a continuously fresh measurement.

Both human-facing output and future drivetrain consumers shall receive updates from the same delivery-mode decision.

---

# 5. PhotonVision Coordinate System

PhotonVision's camera coordinate frame shall be used as the canonical camera-relative coordinate system.

PhotonVision defines:

* +X = forward from the camera
* +Y = left
* +Z = up

The origin is the focal point of the camera lens.

The AprilTag origin is the center of the tag.

PhotonVision's AprilTag coordinate frame has its X axis normal to the tag surface and pointing outward from the visible side of the tag.

Because of these coordinate definitions, a camera facing a tag squarely does not necessarily produce zero rotation in all components. In particular, PhotonVision documents a 180-degree Z rotation for the corresponding head-on transform.

The implementation shall not assume that "square to tag" means that all raw rotation components equal zero.

---

# 6. Phase 1 Measurements

For the selected AprilTag, Phase 1 shall report:

* AprilTag ID
* forward/back displacement
* left/right displacement
* up/down displacement
* straight-line range
* measurement validity

Human-facing distance units shall be **inches**.

Internally, WPILib/PhotonVision native units may be retained and converted only at the presentation boundary.

## 6.1 Forward/Back

Derived from camera-frame X translation.

Positive X means the tag is forward of the camera.

Human output shall use directional language rather than relying on a signed number.

Example:

```
Forward: 83.4 in
```

## 6.2 Left/Right

Derived from camera-frame Y translation.

Positive Y means the tag is to the left of the camera.

Examples:

```
Left: 12.3 in
```

or

```
Right: 7.8 in
```

## 6.3 Up/Down

Derived from camera-frame Z translation.

Positive Z means the tag is above the camera.

Examples:

```
Up: 6.1 in
```

or

```
Down: 3.7 in
```

## 6.4 Range

Range is the straight-line three-dimensional distance from the camera origin to the center of the AprilTag.

It is distinct from forward distance.

A tag may therefore have:

```
Forward: 80.0 in
Left:    20.0 in
Up:      10.0 in
Range:   83.1 in
```

The exact numbers above are illustrative only.

---

# 7. Phase 2 Angular Measurements

Human-facing angular measurements shall be expressed in **degrees**.

Instructions shall use human-readable directions such as LEFT and RIGHT rather than requiring the operator to interpret positive and negative angles.

## 7.1 Bearing Correction

Bearing correction answers:

> How far and in which direction must the camera rotate horizontally to point directly toward the center of the AprilTag?

Example:

```
Point at tag: 8.2 deg LEFT
```

The bearing should be derived from the translation portion of the same camera-to-tag transform used for the position measurements.

PhotonVision's independently reported target yaw may be logged for diagnostic comparison.

The human-facing bearing shall not depend solely upon PhotonVision's separately reported yaw value.

This gives the validation process an opportunity to compare two independently available representations of the target bearing.

## 7.2 Square-to-Tag Correction

Square-to-tag correction answers:

> How far and in which direction must the camera rotate horizontally so that it faces squarely/perpendicularly toward the AprilTag plane?

Example:

```
Square with tag: 14.7 deg RIGHT
```

This measurement shall be derived from the rotational component of the camera-to-tag transform.

It shall not be implemented by simply reusing target yaw.

The mathematical definition and normalization of this measurement must be explicitly documented before implementation is considered complete.

## 7.3 Difference Between the Two Angles

A camera can point directly at the center of a tag without being square to the tag.

Therefore:

```
Point at tag
```

and

```
Square with tag
```

are intentionally different measurements.

The software architecture and user interface shall preserve this distinction.

---

# 8. Target Selection

The desired AprilTag ID shall be configurable.

The system shall search detected targets for that specific ID.

Seeing another AprilTag shall not cause the system to silently switch targets.

If the requested tag is not detected, the result shall be:

```
NO TARGET
```

The system may report that other tags are visible for diagnostic purposes, but measurements shall not silently be generated for a different tag.

Automatic target selection may be considered in a later version.

---

# 9. Measurement Validity

The system shall not assume that every AprilTag pose returned by PhotonVision is trustworthy.

At minimum, the system shall expose these states:

```
VALID
NO TARGET
UNRELIABLE
```

## 9.1 NO TARGET

The requested AprilTag is not currently detected.

Position and orientation guidance shall not present stale measurements as current measurements.

## 9.2 UNRELIABLE

The requested AprilTag was detected, but its pose estimate fails the current quality requirements.

PhotonVision documents pose ambiguity caused by multiple plausible 3D solutions from the observed tag corners.

PhotonVision recommends rejecting single-tag poses with an ambiguity ratio greater than 0.20.

The initial project shall therefore use:

```
pose ambiguity > 0.20
    -> UNRELIABLE

pose ambiguity <= 0.20
    -> candidate for VALID
```

The 0.20 threshold is an initial value based on PhotonVision guidance.

It shall remain configurable and shall be evaluated experimentally.

Ambiguity alone may ultimately prove insufficient as the complete validity test.

Additional rejection criteria may be added after physical testing.

## 9.3 VALID

The requested tag is detected and the measurement passes all configured quality requirements.

Only VALID measurements should normally be presented as actionable human guidance.

---

# 10. Pose Ambiguity

PhotonVision can provide both a best and alternate camera-to-target transform when solving a single AprilTag pose.

The project shall retain enough diagnostic information to investigate cases where the pose solution becomes ambiguous.

Testing shall specifically investigate:

* nearly head-on tag views;
* oblique tag views;
* long-distance observations;
* partially visible tags;
* small tags in the image;
* camera movement;
* motion blur.

The system shall not attempt to hide pose instability through filtering before the raw behavior has been characterized.

---

# 11. Camera Calibration

Camera calibration is a mandatory system requirement.

The camera must be calibrated for the actual physical camera and the resolution used by the AprilTag pipeline.

Changing resolution may require a different calibration.

Calibration quality directly affects 3D pose accuracy.

The project shall initially follow PhotonVision's current recommended ChArUco-based calibration procedure.

Calibration shall cover:

* different target positions throughout the image;
* different distances;
* different target angles;
* broad coverage of the camera field of view.

The calibration target must be flat and accurately dimensioned.

Calibration information shall be considered part of the configuration of a particular camera.

---

# 12. Handheld Operation

Phase 1 and Phase 2 shall support handheld operation.

The system shall not require:

* known camera height;
* level camera orientation;
* fixed camera pitch;
* known position relative to the floor;
* known field position;
* robot odometry;
* robot gyro information.

The operator may translate and rotate the camera freely.

This requirement is intentional because it tests whether the camera-to-tag geometry works independently of a robot.

---

# 13. Motion

Handheld and robot-mounted operation introduce camera motion.

PhotonVision documentation identifies motion blur as a significant AprilTag tracking concern.

The validation program shall therefore test:

* stationary camera;
* slow handheld movement;
* handheld rotation;
* faster movement representative of robot motion.

Camera exposure and gain shall be tuned to minimize motion blur while retaining reliable AprilTag detection.

A global-shutter camera is preferred for eventual hardware selection unless testing demonstrates that another camera is adequate.

Filtering shall not initially be used to disguise poor raw measurements.

Filtering may be introduced later after the raw error characteristics are understood.

---

# 14. Initial Console Output

The exact formatting is not yet fixed, but Phase 1 should provide information conceptually similar to:

```
TAG 7                  VALID

Forward:   82.4 in
Left:      11.2 in
Up:         4.8 in
Range:     83.3 in
```

Phase 2 adds:

```
Point at tag:      7.7 deg RIGHT
Square with tag:  12.4 deg LEFT
```

An unreliable measurement should make that condition obvious:

```
TAG 7                  UNRELIABLE
Pose ambiguity: 0.31
```

Actionable movement guidance should not be presented as trustworthy while the state is UNRELIABLE.

If the requested tag disappears:

```
TAG 7                  NO TARGET
```

The console shall not continue presenting old measurements in a way that could be mistaken for live data.

---

# 15. Diagnostic Information

Human guidance and diagnostic information are separate concepts.

During development, diagnostic output should make it possible to examine at least:

* requested tag ID;
* detected tag ID;
* raw camera-to-tag translation;
* raw camera-to-tag rotation;
* calculated range;
* calculated bearing;
* PhotonVision-reported yaw;
* pose ambiguity;
* measurement validity;
* measurement timestamp or age where appropriate.

This information does not all need to appear continuously in the normal human-facing display.

---

# 16. Single-Tag Versus MultiTag

Initial development shall use the pose estimate for the specifically requested AprilTag.

The project is initially solving:

> What is the camera's relationship to this AprilTag?

It is not initially solving:

> Where is the robot on the FRC field?

PhotonVision MultiTag field localization may be investigated later.

MultiTag localization shall not replace the core camera-to-selected-tag measurement abstraction.

The eventual system may use both:

* selected-tag relative pose for local alignment;
* MultiTag/field pose estimation for field localization.

These are complementary capabilities.

---

# 17. Physical Validation Program

Physical validation is part of the requirements, not merely a debugging activity.

The system shall be tested against independently measured physical geometry.

## 17.1 Distance Tests

Place the camera and tag at known separations.

Test multiple ranges.

Compare:

* measured physical range;
* reported range;
* measured forward distance;
* reported forward distance.

## 17.2 Lateral Tests

Keep a known forward separation and move the camera/tag laterally by known amounts.

Verify:

* LEFT/RIGHT direction;
* lateral distance magnitude;
* bearing direction;
* bearing magnitude.

## 17.3 Vertical Tests

Change the relative camera/tag height by known amounts.

Verify:

* UP/DOWN direction;
* vertical displacement magnitude;
* range behavior.

## 17.4 Bearing Tests

Place the tag at known lateral offsets.

Verify the reported "Point at tag" correction.

## 17.5 Orientation Tests

Rotate the camera and/or tag through known angles.

Verify the "Square with tag" correction independently of bearing-to-center.

This test is especially important because bearing and tag orientation must remain distinct.

## 17.6 Combined Geometry

Test positions containing simultaneous:

* forward displacement;
* lateral displacement;
* vertical displacement;
* camera rotation;
* tag orientation.

The system should continue producing geometrically consistent results.

## 17.7 Handheld Tests

Repeat representative measurements while physically holding the camera.

Characterize:

* jitter;
* latency;
* pose flipping;
* loss of detection;
* reacquisition;
* ambiguity changes;
* motion-blur effects.

---

# 18. Accuracy Requirements

Numerical acceptance tolerances are currently TBD.

They shall not be invented before experimental data is available.

Testing should determine errors as functions of at least:

* distance;
* tag image size;
* viewing angle;
* lighting;
* camera movement;
* calibration quality.

The results will be used to establish realistic acceptance limits for later specification revisions.

---

# 19. Future Robot Integration

When the camera is rigidly mounted to the robot, the camera's pose relative to the robot shall become a calibrated/configured transform.

The measurement chain will conceptually become:

```
robot
  |
  | known robot-to-camera transform
  v
camera
  |
  | measured camera-to-tag transform
  v
AprilTag
```

The resulting robot-to-tag relationship shall use the same camera-to-tag measurement developed and validated during the handheld phases.

Robot integration shall therefore not require replacement of the core vision measurement system.

---

# 20. Future Swerve Alignment

The eventual swerve system should be expressed in terms of a desired robot pose relative to an AprilTag.

For example:

> Position the robot centered on Tag 7, square with its face, with the robot reference point 24 inches from the tag.

The system can then determine errors in:

* forward/back translation;
* left/right translation;
* rotation.

Swerve is particularly suitable because translation and robot orientation can be controlled independently.

The vision layer shall report geometry.

The drivetrain layer shall be responsible for deciding how to move the robot.

This separation shall be maintained in the architecture.

---

# 21. Explicit Non-Goals for Initial Versions

Phase 1 and Phase 2 shall NOT:

* control drivetrain motors;
* require a swerve drivetrain;
* require any drivetrain;
* require field-relative robot localization;
* require odometry;
* require a gyro;
* require a fixed camera height;
* assume the camera is level;
* automatically select arbitrary visible tags;
* conceal unreliable measurements;
* depend on NetworkTables-based robot pose estimation as part of the geometry calculation.

The goal is first to establish and validate the camera-to-AprilTag measurement.

---

# 22. Open Issues

The following remain intentionally unresolved.

## Hardware

Select:

* camera;
* PhotonVision processor/coprocessor;
* resolution;
* expected frame rate.

A global-shutter camera is preferred for investigation.

## Accuracy

Determine acceptable measurement error through physical testing.

## Filtering

Determine whether filtering is necessary only after characterizing raw measurements.

Potential future questions include:

* smoothing;
* outlier rejection;
* measurement age;
* stability requirements before displaying guidance.

## Square-to-Tag Mathematics

Precisely define how the 3D rotational transform is reduced to the human-facing horizontal:

```
Square with tag: XX.X deg LEFT/RIGHT
```

This must account correctly for PhotonVision/WPILib AprilTag coordinate conventions.

It should be validated using known physical tag orientations before robot use.

## Validity

Determine whether ambiguity <= 0.20 alone is adequate or whether additional quality gates are needed.

## Delivery Mode

Determine the startup delivery mode, the default periodic update frequency, and which operator control calls the delivery-mode API after startup.

---

# 23. Immediate Next Step

Status: completed in [Phase 1 and Phase 2 geometry](phase-1-2-geometry.md). The module and type decisions are recorded in [Phase 1 and Phase 2 design rationale](phase-1-2-design-rationale.md).

The next implementation task starts with executable geometry tests. It does not start with camera access.

The completed design defines and diagrams the geometry used by Phase 1 and Phase 2.

It establishes these points with concrete camera and tag arrangements:

1. what X, Y and Z mean;
2. how range is calculated;
3. how "Point at tag" is calculated;
4. how "Square with tag" is calculated;
5. what happens when the camera is both displaced and rotated;
6. what output a human should see in each case.

The geometry is now defined. The worked arrangements should become table-driven tests before PhotonLib or camera hardware is added.
