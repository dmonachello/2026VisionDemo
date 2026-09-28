# Install the vision hardware

This guide covers the parts, mounting, wiring, and first power-up for the 2026 Vision Demo. It uses one Arducam B0332 camera, one Orange Pi 5 v1.2, and one roboRIO v1.

The finished installation keeps image processing on the Orange Pi. The roboRIO receives AprilTag measurements through PhotonLib and runs the robot code.

## Gather the parts

The project already has these main components:

- one roboRIO v1;
- one Orange Pi 5 v1.2;
- one [Arducam B0332 OV9281 USB camera](https://www.arducam.com/arducam-120fps-global-shutter-usb-camera-board-1mp-720p-ov9281-uvc-webcam-module-with-low-distortion-m12-lens-without-microphones-for-computer-laptop-android-device-and-raspberry-pi.html), marked `UC-844 Rev B`; and
- one Xbox controller connected to the Driver Station computer.

Add these parts for the vision installation:

- an industrial-grade microSD card, 8 GB minimum and 16 GB or larger recommended;
- an Orange Pi 5 heatsink and fan;
- a PhotonVision-recommended 5 V regulator, such as a Redux Robotics Zinc-V or Pololu S13V30F5;
- a locking or mechanically secured USB-C power cable for the Orange Pi;
- red and black 18 AWG or 20 AWG power wire;
- a breaker and connection hardware that match the regulator instructions and the current FRC electrical rules;
- an unmanaged Ethernet switch and its required power connection;
- three short Ethernet cables for the radio, the roboRIO, and the Orange Pi;
- the USB cable supplied with the Arducam;
- nonconductive standoffs and fasteners for the Orange Pi;
- a ventilated cover or enclosure for the Orange Pi;
- a rigid camera bracket;
- cable clamps, hook-and-loop straps, or other strain relief;
- heat-shrink tubing and insulated wire terminals; and
- labels for both ends of every power, USB, and Ethernet cable.

Calibration and measurement work also needs:

- a flat ChArUco calibration board printed at 100 percent scale;
- a rigid backing plate for the calibration board;
- calipers or another accurate tool for measuring the printed squares;
- one or more AprilTags printed at known dimensions and mounted flat;
- a tape measure; and
- a level or angle reference for the physical validation fixture.

The radio, robot battery, main breaker, and PDP or PDH are part of the robot control system rather than the vision kit. They must be present for an on-robot installation.

## Use this connection layout

```text
Robot battery
     |
     v
Main breaker
     |
     v
PDP or PDH
     |
     +-----------------------> roboRIO power input
     |
     +-- breaker --> 5 V regulator --> secured USB-C --> Orange Pi 5
     |
     +-- breaker or approved supply -----------------> Ethernet switch

Arducam B0332 -- USB 2.0 --> Orange Pi 5

                           +--> roboRIO
Robot radio <--> Ethernet switch --> Orange Pi 5

Xbox controller -- USB --> Driver Station computer -- radio link --> robot
```

The Xbox controller does not connect to the roboRIO or Orange Pi. The Driver Station computer reads the X button and sends controller state through the normal FRC control link.

## Prepare the Orange Pi

1. Install the heatsink and fan before running sustained AprilTag processing.
2. Download `photonvision-v2026.3.4-linuxarm64_orangepi5.img.xz` from the [PhotonVision v2026.3.4 release](https://github.com/PhotonVision/photonvision/releases/tag/v2026.3.4).
3. Verify the download against the checksum published with the release.
4. Flash the image to the microSD card with a supported imaging tool.
5. Insert the microSD card into the Orange Pi.
6. Leave the camera disconnected for the first boot.

Use PhotonVision `v2026.3.4` because the project uses PhotonLib `v2026.3.4`. PhotonVision and PhotonLib versions must match.

## Mount the Orange Pi and regulator

1. Mount the Orange Pi on nonconductive standoffs.
2. Leave space around the heatsink and fan for airflow.
3. Keep the board away from metal chips, loose hardware, and direct impact.
4. Place the Orange Pi where the USB and Ethernet cables do not cross moving mechanisms.
5. Mount the 5 V regulator close enough to keep its output wires short.
6. Keep the regulator and its terminals covered against accidental shorts.
7. Add strain relief near the Orange Pi USB-C power connector.

Do not place the Orange Pi in a sealed box. The RK3588S processor can reduce its speed when it becomes too hot, and a sealed enclosure traps heat from both the processor and regulator.

## Mount the camera

For Phases 1 and 2, fasten the camera to a rigid handheld bracket or test fixture. Do not hold the bare circuit board by its USB cable.

For robot installation:

1. Choose a location with a clear view of the expected AprilTags.
2. Keep bumpers, frame rails, game pieces, and moving mechanisms out of the image.
3. Fasten the camera bracket to the robot frame.
4. Fasten the camera board to the bracket through its mounting holes.
5. Protect the circuit board without covering the lens or blocking airflow.
6. Clamp the USB cable near the camera so a cable pull cannot move the camera.
7. Record the camera position and rotation relative to the robot coordinate frame.

The camera mount must not shift after calibration and robot-to-camera measurement. A small angular change can create a large position error at long range.

The B0332 has a listed focus range of about 1 meter to infinity. Place the camera so the desired alignment pose does not put the tag well inside that distance. Test the final mounting location at the closest expected range.

## Wire the Orange Pi power

Disconnect the robot battery before changing power wiring.

1. Mount the recommended 5 V regulator according to its manufacturer instructions.
2. Connect the regulator input to its own protected PDP or PDH branch circuit.
3. Size the breaker and wire for the regulator and the current FRC electrical rules.
4. Connect the regulator output to the Orange Pi with a locking USB-C cable or secured USB-C pigtail.
5. Support the cable so vibration cannot work the connector loose.
6. Check the input polarity and output polarity before applying power.
7. Leave the Orange Pi disconnected from the regulator.
8. Power the regulator and measure its output with a multimeter.
9. Remove robot power after confirming the correct output.
10. Connect the verified USB-C output to the Orange Pi.

Do not power the Orange Pi from the roboRIO USB port. Do not connect raw robot battery voltage to the Orange Pi. An undervoltage can cause throttling, camera loss, corrupted storage, or an unexpected reboot.

Follow the [PhotonVision power wiring guide](https://docs.photonvision.org/en/latest/docs/quick-start/wiring.html) for the selected regulator. The guide recommends 18 AWG or 20 AWG wire and a secured coprocessor power connector.

## Connect the camera

1. Plug the Arducam USB cable into the camera.
2. Plug the other end into one Orange Pi USB port.
3. Label that port and keep using the same port after configuration.
4. Add strain relief at both ends of the cable.
5. Route the cable away from motors, gears, chains, and sharp edges.
6. Leave a service loop that does not pull on either connector.

PhotonVision matches USB cameras partly by physical port. Moving the camera to another port can cause a camera mismatch or load the wrong saved settings.

The external trigger pins on the B0332 are not used. PhotonVision receives the normal free-running UVC camera stream.

## Connect the network

Use the Ethernet switch as the center of the robot network:

1. Connect the roboRIO Ethernet port to the switch.
2. Connect the Orange Pi Ethernet port to the switch.
3. Connect the robot radio to the switch.
4. Power the switch from an approved robot power connection.
5. Secure every Ethernet cable close to its connector.
6. Label both ends of each cable.

Do not feed Power over Ethernet into the Orange Pi Ethernet port. Power the Orange Pi only through the dedicated 5 V regulator connection described above.

If the robot uses a VH-109 radio, turn off radio DIP switches 1 and 2 before connecting the Orange Pi network. PhotonVision warns that the radio's PoE mode can damage a coprocessor. See the [PhotonVision network wiring guide](https://docs.photonvision.org/en/latest/docs/quick-start/networking.html).

## Perform the first power-up

1. Remove loose tools and wire scraps from the robot.
2. Confirm that every board is mounted and every cable has strain relief.
3. Confirm that the Orange Pi fan can turn freely.
4. Confirm the regulator polarity one more time.
5. Connect the robot battery and turn on the main breaker.
6. Watch the Orange Pi for a normal boot.
7. Check the switch, roboRIO, and Orange Pi Ethernet link lights.
8. Confirm that the Orange Pi stays powered while the roboRIO boots.
9. Open the PhotonVision dashboard from a computer on the robot network.
10. Set the team number and a unique Orange Pi hostname.
11. Configure a static address by following the PhotonVision networking instructions.
12. Shut down the system before connecting the camera for the first time.
13. Connect the camera to its labeled USB port and restart the system.

During bench work, connect both the computer and Orange Pi through a physical router or robot radio. PhotonVision does not support a direct link-local connection as the normal setup.

## Configure the camera

In the PhotonVision dashboard:

1. Activate the Arducam camera.
2. Assign a unique camera nickname and record it for robot code.
3. Select the OV9281 camera type.
4. Select `1280x800` resolution.
5. Select an MJPEG frame mode.
6. Start at 30 frames per second.
7. Select the AprilTag pipeline.
8. Enable 3D mode.
9. Set AprilTag decimate to `2` as the initial value.
10. Lower exposure until robot motion produces little blur.
11. Add enough gain to keep tags detectable under expected lighting.

The B0332 limits full-resolution YUY2 to 10 frames per second. Use MJPEG for higher full-resolution frame rates. Raise the frame rate only after 30 frames per second produces stable detections and acceptable latency.

The [PhotonVision quick configuration guide](https://docs.photonvision.org/en/latest/docs/quick-start/quick-configure.html) recommends 1280 by 800, decimate `2`, 3D mode, and the OV9281 selector for this Orange Pi and camera combination.

## Calibrate the camera

Calibrate the exact camera at `1280x800`. A calibration from another camera or resolution is not valid.

1. Print the PhotonVision ChArUco board at 100 percent scale.
2. Mount the print to a flat, rigid backing.
3. Measure the printed squares and markers.
4. Enter the measured dimensions into PhotonVision.
5. Capture calibration images across the full image area.
6. Include different distances and angles.
7. Keep the camera fixed during the calibration session.
8. Save and back up the completed calibration.
9. Record the camera serial information, resolution, and calibration date.

Follow the [PhotonVision camera calibration procedure](https://docs.photonvision.org/en/latest/docs/calibration/calibration.html). Do not continue to 3D measurement testing until the calibration overlay follows the board across the image.

## Measure the robot-to-camera transform

Phase 3 and later need the camera pose relative to the robot.

1. Choose and document the robot coordinate origin.
2. Measure the camera lens center from that origin in the forward, left, and up directions.
3. Measure the camera roll, pitch, and yaw relative to the robot frame.
4. Record the values in meters and degrees.
5. Enter the values into robot configuration only after a second person checks them.

Measure from the camera lens center, not from the front of the circuit board or the mounting bracket.

## Verify the completed installation

Complete these checks before drivetrain testing:

- The Orange Pi boots on every robot power cycle.
- The Orange Pi reports no undervoltage or thermal throttling.
- The camera appears after every power cycle.
- PhotonVision keeps the camera matched when it stays in the labeled USB port.
- The 1280×800 MJPEG stream runs at the selected frame rate.
- PhotonVision detects an AprilTag while the camera and tag are stationary.
- PhotonVision continues detecting the tag during representative camera motion.
- The roboRIO receives PhotonVision data through NetworkTables.
- The camera nickname in robot code matches the PhotonVision nickname.
- PhotonVision and PhotonLib both report version `v2026.3.4`.
- Moving each cable by hand does not interrupt power, video, or networking.
- The camera bracket does not move when the robot accelerates or stops.
- The tag remains in focus at the closest required distance.

Do not enable automatic drivetrain movement until the physical measurement tests in the [requirements and design specification](frc-apriltag-relative-measurement-and-alignment-spec.md) pass.

## Record the installed configuration

Keep these values with the robot configuration:

- Orange Pi board model and revision;
- PhotonVision image version;
- microSD card make and capacity;
- regulator model;
- camera model and board marking;
- camera USB port;
- camera nickname;
- camera resolution, frame format, and frame rate;
- exposure and gain;
- calibration file and date;
- robot-to-camera transform;
- Orange Pi hostname and static address; and
- measured frame rate, latency, and closest reliable tag distance.

Recheck calibration and the robot-to-camera transform after the camera, lens, bracket, resolution, or mounting position changes.
