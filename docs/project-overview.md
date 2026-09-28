# 2026 Vision Demo overview

## The idea

The 2026 Vision Demo measures where a selected AprilTag is relative to a camera. It begins as a handheld tool that reports distance and angle. Once those measurements hold up in physical tests, the same geometry can guide a swerve-drive robot into position.

That progression is deliberate. A bad distance on a console does not move hardware. The same bad distance sent to a drivetrain can move a robot in the wrong direction.

The project uses Java, WPILib, PhotonVision, and PhotonLib.

## What the robot needs to know

Detecting an AprilTag is not enough. To approach a field element, the robot needs to know:

- how far the tag is in front of, beside, and above the camera;
- the straight-line distance to the tag;
- how far to turn to point at the tag's center;
- how far to turn to face the tag squarely; and
- whether the measurement is safe to use.

Pointing at the center and facing the tag squarely are different. A camera can point at the center from an angle while its face remains turned away from the tag. The software keeps those two corrections separate.

## The measurement path

PhotonVision detects tags and reports a three-dimensional camera-to-tag transform. That transform is the source for every distance and angle this project reports.

```text
Camera and PhotonVision
          |
          v
Requested AprilTag
          |
          v
Geometry and reliability checks
          |
          v
PERIODIC or SNAPSHOT delivery
          |
          +--------------------+
          |                    |
          v                    v
 Console guidance        Drivetrain input
```

The result has one of three states:

- `VALID` contains geometry that a person or robot can act on.
- `NO TARGET` means the requested tag is not visible.
- `UNRELIABLE` means the tag is visible, but its pose estimate failed a quality check.

The software does not substitute another visible tag when the requested tag disappears. It also does not keep presenting an old measurement as if it were current.

## When measurements are delivered

The camera keeps processing images in both delivery modes.

`PERIODIC` delivers new camera results up to a configured frequency. The actual rate may be lower because of the camera frame rate, robot loop rate, or processing time. The system never republishes an old frame to make the rate look faster.

`SNAPSHOT` delivers one update when the operator presses X on the Xbox controller. Holding X does not generate more updates. This mode is useful for taking a reading without a stream of changing console output.

Both the console and the future drivetrain receive measurements from the same delivery decision. The console may print less often for readability, but its print rate cannot slow the measurements sent to the drivetrain.

## How the project reaches automatic alignment

Development is split into five phases:

1. A handheld camera reports the selected tag's position and range.
2. The handheld tool adds point-at-tag and square-to-tag guidance.
3. A camera mounted on the robot reports the robot's position relative to the tag.
4. The software calculates the move needed to reach a desired pose, but does not command the drivetrain.
5. The drivetrain uses fresh, valid measurements for automatic swerve alignment.

This order keeps motor control out of the project until the team has checked coordinate directions, camera calibration, timing, and failure cases against real measurements.

## How vision fits into drivetrain control

Vision updates the robot's estimate. It does not control when the drivetrain loop runs.

The drivetrain continues at its fixed control rate and uses wheel odometry and gyro data between camera results. Each accepted vision result includes the time when the camera captured it, so robot code can account for delay and motion.

Live automatic alignment requires `PERIODIC` mode. The drivetrain rejects repeated or stale camera results. If the requested tag disappears or its pose becomes unreliable, alignment stops or enters another tested safe state.

A snapshot can establish a fixed goal if odometry and the gyro maintain that goal afterward. A retained snapshot is not live vision feedback.

## Proving that the measurements are trustworthy

Camera calibration and physical testing are part of the build, not cleanup work after the code runs. Tests compare the reported geometry with known distances and angles. They cover centered tags, offsets, rotated views, handheld motion, and motion representative of a robot.

The first quality check uses PhotonVision's pose ambiguity. Testing may uncover other rejection rules, such as a maximum measurement age. The team will record raw behavior before adding filters, because smoothing bad data can make a broken measurement look convincing.

## Where the project stands

The repository contains the WPILib project, PhotonLib dependency, requirements, geometry contract, design rationale, and implementation plan. The vision measurement and delivery code is not written yet.

The next task is to write table-driven tests for the camera-to-tag geometry. Those tests define the expected signs and angles before PhotonVision or camera hardware enters the code path.

## Detailed documents

- [Requirements and design specification](frc-apriltag-relative-measurement-and-alignment-spec.md)
- [Implementation plan](implementation-plan.md)
- [Phase 1 and Phase 2 geometry](phase-1-2-geometry.md)
- [Phase 1 and Phase 2 design rationale](phase-1-2-design-rationale.md)
