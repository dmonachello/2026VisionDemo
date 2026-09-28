# Implement Phase 1 and Phase 2 measurement

This plan implements the camera-relative measurement system in verifiable steps. Each step ends with an automated or physical check.

## Hardware baseline

The project uses this hardware:

- roboRIO v1 for the WPILib robot application;
- Orange Pi 5 v1.2 for PhotonVision; and
- Arducam UC-844 Rev B for image capture.

Before camera integration, bench-test power, cooling, USB connectivity, and network communication. Record the operating-system image, PhotonVision version, camera resolution, and network settings.

Follow [Install the vision hardware](hardware-installation.md) for the parts list, mounting, power wiring, network wiring, camera connection, and first power-up.

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

- `setDeliveryMode(PERIODIC)` and `setDeliveryMode(SNAPSHOT)` change the active mode through the public API.
- `deliveryMode()` reports the active mode.
- Setting the active mode again has no effect.
- `setPeriodicUpdateFrequencyHz(double)` accepts only finite values greater than zero.
- `periodicUpdateFrequencyHz()` reports the configured value.
- Setting the current periodic frequency again has no effect.
- Changing the frequency in snapshot mode stores it without emitting an update.
- Changing the frequency in periodic mode resets the schedule and makes the next new camera result eligible immediately.
- `PERIODIC` emits newly processed camera results at no more than the configured frequency.
- The effective periodic rate cannot exceed the camera frame rate, robot loop rate, or processing rate.
- Repeated robot loops over the same camera result emit nothing.
- Missed periodic deadlines do not produce catch-up bursts.
- `SNAPSHOT` emits nothing until it receives a snapshot request.
- One snapshot request emits one current result.
- The request is consumed after that update.
- A snapshot request made outside `SNAPSHOT` mode has no effect.
- Changing modes clears pending snapshot state and does not emit an update.
- Every emitted update has a sequence number and publication timestamp.

Add state-machine tests for the mode API, repeated loops, repeated requests, and all three measurement states. Cover zero, negative, nonfinite, unchanged, and changed frequency values. Use a fake monotonic clock to test exact deadlines and missed-deadline behavior.

The step is complete when the state-machine tests prove that holding the snapshot input cannot repeat an update.

## 4. Add the PhotonVision adapter

Create `PhotonSelectedTagMeasurer` under `src/main/java/frc/robot/vision/photon`.

The adapter reads complete PhotonVision results, selects only the requested ID, records diagnostics, applies the measurement policy, and calls the pure geometry solver.

Expose whether the adapter processed a new camera result. The delivery gate uses that fact in periodic mode.

The step is complete when synthetic PhotonVision observations produce the expected measurement state and geometry.

## 5. Add console presentation

Create `TagMeasurementConsole` under `src/main/java/frc/robot/vision/console`.

The console consumes `MeasurementUpdate`, not the raw camera result. It owns inches, degrees, direction words, rounding, and output throttling.

Snapshot output displays its capture time or age and remains labeled as a snapshot while retained on screen.

The step is complete when output tests cover periodic, snapshot, no-target, and unreliable updates.

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

Keep console throttling after `MeasurementUpdateGate`. A readable console rate must not reduce the update rate available to a future drivetrain.

The step is complete when robot simulation starts without camera hardware and reports the correct state in both delivery modes.

## 8. Configure and calibrate the camera

Connect the Arducam UC-844 Rev B to the Orange Pi 5 v1.2. Configure the PhotonVision AprilTag pipeline and enable single-tag 3D pose estimation. Calibrate the camera at the operating resolution.

Record the camera revision, resolution, tag family, pipeline settings, calibration identity, frame rate, and measured latency.

The step is complete when PhotonVision on the Orange Pi publishes the requested tag's 3D transform to the Java application on the roboRIO.

## 9. Validate physical geometry

Test centered, laterally displaced, vertically displaced, rotated, combined, and handheld arrangements. Record ground truth and raw diagnostic data without filtering.

Test both delivery modes. Periodic mode must obey its configured maximum frequency and publish only new camera results. Snapshot mode must remain unchanged between X-button presses.

Verify directions before setting accuracy tolerances.

The step is complete when the reported signs and both angular corrections match the test fixture.

## 10. Set empirical policy

Use the physical results to set frame-age limits, display deadband, accuracy tolerances, and any quality checks beyond ambiguity.

Do not add filtering until the raw measurements are characterized.

The startup delivery mode, the default periodic frequency, and the operator control for changing modes remain product decisions.

## 11. Preserve the future drivetrain boundary

Do not call drivetrain control only when a `MeasurementUpdate` arrives. The future drivetrain controller runs every drivetrain cycle and reads the latest accepted vision estimate.

Require `PERIODIC` mode for direct live-vision alignment. Treat `SNAPSHOT` as measurement, diagnostics, human guidance, or an input that establishes a fixed odometry goal.

Before automatic movement is enabled, add tests that prove these behaviors:

- The drivetrain control loop continues between vision updates.
- Only `VALID` periodic updates replace the current vision estimate.
- Duplicate camera timestamps are ignored.
- `NO TARGET`, `UNRELIABLE`, and stale estimates select the defined safe drivetrain state.
- The controller uses capture time rather than publication time for measurement age.
- Snapshot mode cannot feed a frozen transform into live closed-loop alignment.

Choose the maximum measurement age and safe drivetrain response through robot testing. Keep both values in drivetrain policy rather than geometry code.
