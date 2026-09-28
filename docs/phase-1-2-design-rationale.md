# Phase 1 and Phase 2 design rationale

## Problem

Phase 1 and Phase 2 need one trustworthy measurement path from a selected PhotonVision target to human guidance. The translation math is direct. The orientation math is easy to get wrong because a head-on tag has a 180 degree rotation, `Rotation3d` is a full 3D rotation, and the handheld camera has no gravity reference. The project must also preserve the complete transform for later robot-mounted use without pulling drivetrain concerns into the first two phases.

The geometry contract is defined in [Phase 1 and Phase 2 geometry](phase-1-2-geometry.md).

## Usage from the caller's view

The normal caller asks for one configured tag and handles a closed result family:

```java
TagMeasurement measurement = tagMeasurer.latestFor(requestedTagId);

switch (measurement) {
  case ValidTagMeasurement valid -> console.show(valid);
  case UnreliableTagMeasurement unreliable -> console.show(unreliable);
  case NoTargetMeasurement noTarget -> console.show(noTarget);
}
```

Only `ValidTagMeasurement` contains actionable geometry:

```java
void show(ValidTagMeasurement measurement) {
  RelativeTagGeometry geometry = measurement.geometry();

  printDirectionalDistance(geometry.cameraToTag().getX(), "Forward", "Back");
  printDirectionalDistance(geometry.cameraToTag().getY(), "Left", "Right");
  printDirectionalDistance(geometry.cameraToTag().getZ(), "Up", "Down");
  printRange(geometry.rangeMeters());
  printTurn("Point at tag", geometry.bearingCorrection().radians());
  printTurn("Square with tag", geometry.squareCorrection().radians());
}
```

Phase 3 can compose the original transform without reconstructing geometry from formatted values:

```java
Transform3d robotToTag =
    robotToCamera.plus(valid.geometry().cameraToTag());
```

## Shape

The design has five ownership boundaries:

```text
PhotonSelectedTagMeasurer
  owns camera access, exact-ID selection, timestamps, ambiguity policy
             |
             v
RelativeTagGeometry
  owns transform interpretation, range, bearing, face-normal projection
             |
             v
TagMeasurement
  owns VALID, NO_TARGET, and UNRELIABLE state
             |
             v
MeasurementUpdateGate
  owns PERIODIC and SNAPSHOT delivery decisions
             |
             v
TagMeasurementConsole
  owns inches, degrees, direction words, rounding, and display timing
```

These modules are grouped by the facts they own. They are not generic fetch, validate, transform, and print stages.

### Geometry types

```java
package frc.robot.vision.geometry;

public record BearingCorrection(double radians) {
  // Normalized to (-pi, +pi]. Positive is LEFT.
}

public record SquareCorrection(double radians) {
  // Normalized to (-pi, +pi]. Positive is LEFT.
}

public record RelativeTagGeometry(
    Transform3d cameraToTag,
    double rangeMeters,
    BearingCorrection bearingCorrection,
    SquareCorrection squareCorrection) {}

public sealed interface GeometryResult
    permits ValidGeometry, UndefinedGeometry {}

public record ValidGeometry(RelativeTagGeometry geometry)
    implements GeometryResult {}

public record UndefinedGeometry(Set<GeometryFailure> failures)
    implements GeometryResult {}

public enum GeometryFailure {
  NONFINITE_TRANSFORM,
  CENTER_BEARING_UNDEFINED,
  SQUARE_CORRECTION_UNDEFINED
}

public final class CameraToTagGeometry {
  private CameraToTagGeometry() {}

  public static GeometryResult solve(Transform3d cameraToTag) {
    throw new UnsupportedOperationException("not implemented");
  }
}
```

Separate correction types stop a caller from swapping two values that share the same numeric unit. `RelativeTagGeometry` retains the complete `Transform3d` as the single source for translation and later frame composition.

`CameraToTagGeometry.solve` is pure. It has no `PhotonCamera`, clock, console, NetworkTables, robot pose, or drivetrain dependency.

### Measurement states

```java
package frc.robot.vision;

public sealed interface TagMeasurement
    permits ValidTagMeasurement,
            UnreliableTagMeasurement,
            NoTargetMeasurement {
  int requestedTagId();
}

public record ValidTagMeasurement(
    int requestedTagId,
    double captureTimestampSeconds,
    double poseAmbiguity,
    RelativeTagGeometry geometry,
    TagDiagnostics diagnostics) implements TagMeasurement {}

public record UnreliableTagMeasurement(
    int requestedTagId,
    double captureTimestampSeconds,
    Set<UnreliableReason> reasons,
    TagDiagnostics diagnostics) implements TagMeasurement {}

public record NoTargetMeasurement(
    int requestedTagId,
    double captureTimestampSeconds,
    Set<Integer> visibleTagIds) implements TagMeasurement {}
```

The result family is intentionally asymmetric. Only the valid variant exposes `RelativeTagGeometry`. An unreliable result retains raw diagnostics, including the best and alternate transforms, but application code cannot consume those values through the normal guidance API.

A no-target result has no transform. This makes stale actionable guidance unrepresentable in the current result.

### Delivery modes

The delivery mode is a small state machine. It sits after measurement and validity so both modes use identical geometry and quality policy.

```java
package frc.robot.vision.delivery;

public enum MeasurementDeliveryMode {
  PERIODIC,
  SNAPSHOT
}

public record MeasurementUpdate(
    long sequenceNumber,
    double publicationTimestampSeconds,
    MeasurementDeliveryMode mode,
    TagMeasurement measurement) {}

public final class MeasurementUpdateGate {
  public void setDeliveryMode(MeasurementDeliveryMode mode) {
    throw new UnsupportedOperationException("not implemented");
  }

  public MeasurementDeliveryMode deliveryMode() {
    throw new UnsupportedOperationException("not implemented");
  }

  public void setPeriodicUpdateFrequencyHz(double frequencyHz) {
    throw new UnsupportedOperationException("not implemented");
  }

  public double periodicUpdateFrequencyHz() {
    throw new UnsupportedOperationException("not implemented");
  }

  public void requestSnapshot() {
    throw new UnsupportedOperationException("not implemented");
  }

  public Optional<MeasurementUpdate> update(
      TagMeasurement latestMeasurement,
      boolean isNewCameraResult,
      double nowSeconds) {
    throw new UnsupportedOperationException("not implemented");
  }
}
```

`PERIODIC` emits newly processed camera results at no more than `periodicUpdateFrequencyHz()`. Repeated robot loops over the same camera result emit nothing.

The periodic frequency must be finite and greater than zero. An invalid value fails at the API boundary. Setting the current value again has no effect.

The API accepts a new periodic frequency in either mode. Snapshot mode stores it for later. Changing it during periodic mode resets the schedule, and the first new camera result is eligible immediately. Setting the same value does not reset the schedule.

The frequency is a maximum rate. The gate cannot deliver faster than the camera, robot loop, or measurement processing. If no new camera result exists at a scheduled time, the gate emits nothing. It never changes a capture timestamp or repeats an old result to make the requested rate appear achievable.

The scheduler emits at most one update per robot loop. If execution falls behind, it continues from the current time and does not emit delayed updates in a burst.

`SNAPSHOT` emits one `MeasurementUpdate` after `requestSnapshot()`. The update contains the current measurement state, including `NO TARGET` or `UNRELIABLE`. The request is consumed after one update.

`setDeliveryMode` is the public API for changing modes. `deliveryMode` reports the active mode to diagnostics and user interfaces.

`setDeliveryMode` is idempotent and does not publish by itself. Switching modes clears any pending snapshot request. Switching to `SNAPSHOT` waits for the next X-button press. The first new camera result after switching to `PERIODIC` is eligible immediately. Later updates obey the configured interval.

`requestSnapshot` has no effect outside `SNAPSHOT` mode. A request made in periodic mode cannot produce a delayed snapshot after a later mode change.

The measurement capture timestamp remains inside `TagMeasurement`. `MeasurementUpdate` adds the publication timestamp and a sequence number. A console can keep a snapshot on screen, but a drivetrain sees one event with its original age.

The Xbox controller binding produces the snapshot request on the X-button rising edge:

```java
driverController.x().onTrue(
    Commands.runOnce(measurementUpdateGate::requestSnapshot));
```

WPILib's `CommandXboxController.x()` names the X button, and `onTrue` schedules only when the trigger changes from false to true. Holding X therefore does not create repeated requests.

### Future drivetrain boundary

The delivery gate produces vision measurement events. It does not schedule drivetrain control or write motor commands.

```text
MeasurementUpdate events
          |
          v
latest accepted vision estimate and capture time
          |
          +--------------------+
          |                    |
          v                    v
odometry and gyro       drivetrain controller
updates every loop      executes every control loop
                               |
                               v
                         chassis commands
```

The drivetrain controller runs on its own fixed schedule. It reads the newest accepted vision estimate during each control cycle and uses odometry and gyro data between vision updates.

Only a `VALID` periodic update replaces the drivetrain's current vision estimate. The drivetrain still processes `NO TARGET` and `UNRELIABLE` events so it can stop alignment or select another explicitly tested safe state. It rejects duplicate capture timestamps and stops automatic alignment when the estimate becomes too old.

The maximum measurement age is a drivetrain policy based on robot speed, camera latency, and physical testing. It does not belong in `CameraToTagGeometry`.

Snapshot mode does not provide live closed-loop feedback. A snapshot may establish a fixed goal only after robot code converts it at the capture time and then maintains that goal with odometry and gyro data.

Console rendering has a separate throttle. Slowing console text must not reduce the vision update rate available to the drivetrain.

This boundary follows the same pattern as WPILib pose estimation. WPILib calls encoder and gyro updates every robot loop while accepting timestamped vision measurements at their own rate. See the [`PoseEstimator` API](https://github.wpilib.org/allwpilib/docs/release/java/edu/wpi/first/math/estimator/PoseEstimator.html).

### PhotonVision boundary

```java
package frc.robot.vision.photon;

public final class PhotonSelectedTagMeasurer {
  public PhotonSelectedTagMeasurer(
      PhotonCamera camera,
      MeasurementPolicy policy) {
    throw new UnsupportedOperationException("not implemented");
  }

  public TagMeasurement latestFor(int requestedTagId) {
    throw new UnsupportedOperationException("not implemented");
  }
}
```

`PhotonTrackedTarget` and `PhotonPipelineResult` stop at this boundary. The measurer owns exact-ID selection, ambiguity validation, diagnostic capture, and conversion into the domain result.

The first implementation does not need a generic camera-provider interface. There is one provider. A second real provider can justify an interface later.

### Presentation boundary

```java
package frc.robot.vision.console;

public final class TagMeasurementConsole {
  public void show(MeasurementUpdate update) {
    throw new UnsupportedOperationException("not implemented");
  }
}
```

The console converts meters to inches and radians to degrees. It chooses LEFT, RIGHT, FORWARD, BACK, UP, and DOWN from the unrounded sign. In snapshot mode, it labels retained output as a snapshot and shows the capture time or age.

Both the console and any future drivetrain consumer subscribe to `MeasurementUpdate`. Neither consumer implements its own periodic or snapshot logic.

## Synthesis decision

Three independent designs were compared.

The frame-correctness design became the base because it defined the transform direction, the tag-normal projection, the camera-local meaning of horizontal, normalization, and degenerate cases most precisely.

The final design keeps two choices from the simpler design:

- One pure `solve(Transform3d)` operation hides all coordinate and projection rules.
- No generic vision-provider interface exists before a second provider needs it.

The final design also keeps two choices from the future-robot design:

- Only the valid result exposes actionable geometry.
- The complete `Transform3d` remains available for direct Phase 3 composition.

The delivery state machine applies the model-the-domain principle. One gate owns mode, pending snapshot state, and publication sequence. Console and drivetrain code do not duplicate mode checks.

The design rejects speculative Phase 3 types, drivetrain APIs, and a separate transport-health state. Those decisions need hardware and PhotonLib timing data.

This synthesis follows foundational thinking. The transform and result-state types become the stable base, while hardware policy remains changeable.

## Tradeoffs accepted

- The domain API uses WPILib geometry types. This keeps the transform lossless and avoids a duplicate pose model.
- The design adds distinct bearing and square-correction types. This prevents two same-unit values with different meanings from being interchanged.
- Horizontal means camera-local horizontal. Gravity-relative guidance would require another sensor or a fixed mount.
- Undefined projections make the observation unreliable. Returning zero would present an arbitrary angle as alignment.
- The console owns display deadbands. The geometry layer keeps raw values for validation.
- One delivery gate feeds both human and drivetrain consumers. This prevents different consumers from applying different mode rules.
- Snapshot mode keeps camera acquisition active. This gives an X-button press the latest available measurement without warming up the pipeline after the press.
- The drivetrain control loop remains independent of vision delivery. This keeps motor output timing stable when the camera rate changes or a target disappears.

## Alternatives considered

### PhotonVision target yaw for both angles

Target yaw describes image bearing. It does not describe tag face orientation. Reusing it would erase the required distinction between pointing at the tag center and becoming square with its face.

### Euler Z minus 180 degrees

`cameraToTag.getRotation().getZ() - pi` passes simple level tests. It ties correctness to an Euler decomposition and an unstated roll-and-pitch assumption. Rotating the tag normal defines the physical quantity directly.

### One record with status and nullable geometry

A flat record has fewer declarations but permits geometry during `NO TARGET` and `UNRELIABLE`. Every caller would need to remember the field-validity matrix. The sealed family puts that rule in the type system.

### Gravity-relative horizontal

The handheld phases do not provide gravity or field orientation. A gravity-relative angle cannot be derived from the camera-to-tag transform alone.

### Full 3D orientation correction

A complete correction would report more than one rotation component and define an operator sequence. Phase 2 asks only for the horizontal correction. The full transform remains available for a later extension.

## Open questions and risks

- Should the console say `Square horizontally with tag` after operator testing?
- Which delivery mode is active at startup?
- What periodic update frequency is active at startup?
- How does the operator change delivery mode after startup?
- What display deadband prevents tiny LEFT and RIGHT changes from flickering without hiding raw instability?
- What maximum frame age becomes unreliable for the chosen camera, processor, and frame rate?
- What safe drivetrain state follows `NO TARGET`, `UNRELIABLE`, or a stale measurement during automatic alignment?
- How does the selected PhotonVision 2026 release represent unavailable pose ambiguity?
- Can one frame contain duplicate detections for the requested fiducial ID, and how should the adapter report that case?
- Which physical fixture and angle reference will establish the first accuracy tolerances?

These questions do not change the formulas. Hardware testing and the selected PhotonLib release determine their answers.

## Next implementation step

Add table-driven JUnit tests for the worked arrangements in [Phase 1 and Phase 2 geometry](phase-1-2-geometry.md). Implement the pure geometry solver against those tests before adding PhotonLib or camera access.
