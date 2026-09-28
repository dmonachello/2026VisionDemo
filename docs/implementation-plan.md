# Implement Phase 1 and Phase 2 measurement

This plan implements the camera-relative measurement system in verifiable steps. Each step ends with an automated or physical check.

## 1. Build the geometry core

Create these classes under `src/main/java/frc/robot/vision/geometry`:

```text
BearingCorrection.java
SquareCorrection.java
RelativeTagGeometry.java
CameraToTagGeometry.java
GeometryResult.java
GeometryFailure.java
```

Add table-driven JUnit tests for the five arrangements in [Phase 1 and Phase 2 geometry](phase-1-2-geometry.md).

Implement range, point-at-tag correction, and square-with-tag correction only after the tests exist.

The step is complete when all geometry tests pass without PhotonLib camera objects.

## 2. Add measurement states and policy

Create the `VALID`, `NO TARGET`, and `UNRELIABLE` result types under `src/main/java/frc/robot/vision`.

Only the valid result exposes `RelativeTagGeometry`. Keep the raw best and alternate transforms in diagnostics for unreliable observations.

Add tests for exact target selection, the `0.20` ambiguity boundary, nonfinite transforms, and undefined horizontal projections.

The step is complete when invalid or absent observations cannot appear as actionable geometry.

## 3. Add the delivery-mode state machine

Create these classes under `src/main/java/frc/robot/vision/delivery`:

```text
MeasurementDeliveryMode.java
MeasurementUpdate.java
MeasurementUpdateGate.java
```

Implement these rules:

- `setDeliveryMode(CONTINUOUS)` and `setDeliveryMode(SNAPSHOT)` change the active mode through the public API.
- `deliveryMode()` reports the active mode.
- Setting the active mode again has no effect.
- `CONTINUOUS` emits once for each new processed camera result.
- Repeated robot loops over the same camera result emit nothing.
- `SNAPSHOT` emits nothing until it receives a snapshot request.
- One snapshot request emits one current result.
- The request is consumed after that update.
- A snapshot request made outside `SNAPSHOT` mode has no effect.
- Changing modes clears pending snapshot state and does not emit an update.
- Every emitted update has a sequence number and publication timestamp.

Add state-machine tests for the mode API, repeated loops, repeated requests, and all three measurement states.

The step is complete when the state-machine tests prove that holding the snapshot input cannot repeat an update.

## 4. Add the PhotonVision adapter

Create `PhotonSelectedTagMeasurer` under `src/main/java/frc/robot/vision/photon`.

The adapter reads complete PhotonVision results, selects only the requested ID, records diagnostics, applies the measurement policy, and calls the pure geometry solver.

Expose whether the adapter processed a new camera result. The delivery gate uses that fact in continuous mode.

The step is complete when synthetic PhotonVision observations produce the expected measurement state and geometry.

## 5. Add console presentation

Create `TagMeasurementConsole` under `src/main/java/frc/robot/vision/console`.

The console consumes `MeasurementUpdate`, not the raw camera result. It owns inches, degrees, direction words, rounding, and output throttling.

Snapshot output displays its capture time or age and remains labeled as a snapshot while retained on screen.

The step is complete when output tests cover continuous, snapshot, no-target, and unreliable updates.

## 6. Wire the Xbox X button

Create a `CommandXboxController` in `RobotContainer` for the configured driver-station port.

Bind the X-button rising edge to one snapshot request:

```java
driverController.x().onTrue(
    Commands.runOnce(measurementUpdateGate::requestSnapshot));
```

Do not use `whileTrue`. A held X button must not repeat snapshots.

The step is complete when a controller test or Driver Station simulation shows one update per press.

## 7. Integrate the periodic measurement loop

Run camera acquisition and measurement evaluation on every robot loop. Pass each result and its new-frame status to `MeasurementUpdateGate`.

Send each returned `MeasurementUpdate` to the console. Future drivetrain code consumes the same update object.

The step is complete when robot simulation starts without camera hardware and reports the correct state in both delivery modes.

## 8. Configure and calibrate the camera

Configure the AprilTag pipeline and enable single-tag 3D pose estimation. Calibrate the exact physical camera at the operating resolution.

Record the camera name, resolution, tag family, pipeline settings, and calibration identity.

The step is complete when PhotonVision publishes the requested tag's 3D transform to the Java application.

## 9. Validate physical geometry

Test centered, laterally displaced, vertically displaced, rotated, combined, and handheld arrangements. Record ground truth and raw diagnostic data without filtering.

Test both delivery modes. Continuous mode must follow new camera results. Snapshot mode must remain unchanged between X-button presses.

Verify directions before setting accuracy tolerances.

The step is complete when the reported signs and both angular corrections match the test fixture.

## 10. Set empirical policy

Use the physical results to set frame-age limits, display deadband, accuracy tolerances, and any quality checks beyond ambiguity.

Do not add filtering until the raw measurements are characterized.

The startup delivery mode and the operator control for changing modes remain product decisions.
