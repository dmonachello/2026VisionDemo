# Project overview

## What this project is

This project is an FRC vision system that measures the position and orientation of a selected AprilTag. It starts as a handheld measurement tool and grows into a vision input for automatic swerve-drive alignment.

The project uses Java, WPILib, PhotonVision, and PhotonLib. The first goal is to prove that the camera-to-tag measurement is correct and trustworthy. Later phases will mount the camera on a robot and use the same measurement to guide and control the drivetrain.

## The problem it solves

An FRC robot often needs to approach a field element at a known distance, lateral offset, and angle. An AprilTag can provide that reference, but detecting the tag is only the first step. The software must also:

- select the requested tag without switching to another visible tag;
- calculate the tag's full three-dimensional relationship to the camera;
- distinguish pointing toward the tag from facing square with the tag;
- reject missing or unreliable measurements;
- account for camera delay and robot motion; and
- deliver measurements at a rate that works for both people and robot control.

This project develops and tests those parts before allowing vision to command a drivetrain.

## How the system works

PhotonVision detects AprilTags and supplies a camera-to-tag transform. The application treats that transform as the source measurement. It derives readable distances, angles, status, and later robot-relative geometry from the same data.

```text
Camera and PhotonVision
          |
          v
Selected AprilTag measurement
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
 Human-readable output    Future drivetrain input
```

The measurement can be `VALID`, `NO TARGET`, or `UNRELIABLE`. Only a valid result provides actionable movement guidance.

## Measurement delivery

The system supports two delivery modes. Both modes keep the camera and measurement pipeline running.

- `PERIODIC` delivers new camera results at no more than a configured frequency. It never repeats an old camera frame to imitate a faster rate.
- `SNAPSHOT` delivers one update when the operator presses the Xbox controller X button. Holding the button does not repeat the update.

The same delivery decision feeds the human-facing output and the future drivetrain integration. Console output may refresh more slowly, but it must not restrict the update rate available to the drivetrain.

## Development path

The project is divided into five phases:

1. Measure the selected tag's forward, lateral, vertical, and straight-line distance from a handheld camera.
2. Add guidance for pointing at the tag and becoming square with its face.
3. Mount the camera on a robot and convert camera-relative measurements into robot-relative measurements.
4. Calculate the movement required to reach a desired pose near the tag without moving the robot automatically.
5. Feed valid, timestamped measurements into closed-loop swerve alignment.

Each phase tests a smaller part of the final system. Automatic movement comes last, after the project has verified the geometry, calibration, timing, and failure behavior.

## Drivetrain integration

Vision will correct the robot's estimate, but it will not set the drivetrain control-loop rate. The drivetrain controller will continue to run on its own fixed schedule and use odometry and gyro data between vision updates.

For live automatic alignment, the drivetrain will use `PERIODIC` mode. It will reject duplicate or stale camera results and enter a defined safe state when the requested tag is missing or unreliable. A snapshot may establish a fixed goal, but it cannot act as continuing vision feedback.

## Safety and measurement quality

The project treats calibration and physical testing as required development work. Tests will compare reported measurements with known distances and angles under centered, offset, rotated, and moving-camera conditions.

The application will expose unreliable measurements instead of hiding them with filtering. Initial reliability checks include PhotonVision pose ambiguity. Hardware testing will set the final limits for measurement age, accuracy, and acceptable motion blur.

## Current project status

The WPILib project and PhotonLib dependency are in place. The requirements, geometry contract, design rationale, and implementation plan are documented. The vision measurement and delivery components have not yet been implemented.

The next planned work is to create table-driven geometry tests, implement the camera-to-tag geometry calculations, and then connect those calculations to PhotonVision.

## Detailed documents

- [Requirements and design specification](frc-apriltag-relative-measurement-and-alignment-spec.md)
- [Implementation plan](implementation-plan.md)
- [Phase 1 and Phase 2 geometry](phase-1-2-geometry.md)
- [Phase 1 and Phase 2 design rationale](phase-1-2-design-rationale.md)
