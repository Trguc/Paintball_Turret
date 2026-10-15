Person-Tracking Camera
A motorized camera mount that automatically pans to follow a person. Python processes the camera feed with YOLOv8 and ByteTrack, then sends speed commands to an Arduino-controlled stepper motor to keep the tracked person near the center of the image.
The project combines computer vision, serial communication, and motor control in a single horizontal tracking axis.
Features
- Person detection: YOLOv8 nano detects people in the camera feed.
- Target tracking: ByteTrack assigns temporary tracking IDs to reduce switching between people.
- Smooth movement: Proportional speed control, time-based smoothing, and Arduino acceleration limits regulate camera motion.
- Stable centering: Separate start and stop thresholds reduce movement caused by small detection changes.
- Latest-frame capture: A dedicated capture thread replaces older frames instead of building an application-level queue.
- Independent serial updates: A worker sends fresh speed commands, or zero when measurements expire, approximately 25 times per second.
- Automatic stopping: The system requests a stop when the target disappears, observations become stale, or serial commands stop arriving.
- Adjustable behavior: Command-line options control camera selection, motor speed, tracking gain, direction, and inference settings.
- Camera-only mode: Detection and tracking can run without connecting to the Arduino.
How it works
1. OpenCV continuously captures frames from the camera.
2. YOLOv8 detects people, and ByteTrack associates detections across frames.
3. On acquisition, the program selects a reliable person nearest the horizontal center and follows that tracking ID.
4. The controller calculates the horizontal difference between the person and the image center, then converts it into a signed speed request.
5. The Arduino generates STEP pulses, ramps speed changes, and brakes before reversing direction.
The controller sends velocity commands, so it does not accumulate unfinished position corrections. As the person approaches the center, the requested speed decreases.
If the selected person is missed, the controller requests zero speed immediately. It remembers the tracking ID for up to 0.8 seconds before allowing another target to be selected. Remembering an ID does not keep the motor moving during the gap.
Hardware
- Computer capable of running the Python detector
- Camera connected to the computer
- Arduino-compatible board supported by AccelStepper
- Stepper motor and a compatible STEP/DIR driver
- Suitable motor power supply
- Camera mount attached to the motor's output
- USB connection between the computer and Arduino
Configured connections
Connection	Setting
Driver STEP signal	Arduino digital pin 2
Driver DIR signal	Arduino digital pin 3
Computer-to-Arduino link	USB serial
Serial baud rate	115200
Default serial port	COM7
Default camera index	1


Use the power, motor, and signal-ground connections required by your driver. The sketch controls STEP and DIR; it does not control a driver ENABLE pin.
Project files
File	Purpose
[`person_tracker.py`](person_tracker.py)	Camera capture, detection, target selection, speed control, and serial communication
[`person_tracker_arduino/person_tracker_arduino.ino`](person_tracker_arduino/person_tracker_arduino.ino)	STEP/DIR motor control, acceleration ramp, command parsing, and timeout handling
[`requirements.txt`](requirements.txt)	Python dependencies
README.md	Setup, operation, and tuning instructions


Setup
Use the Python program and Arduino sketch from the same project version. This version sends signed speeds; it is not compatible with an older sketch that expects absolute position targets.
1. Upload the Arduino sketch
1. Connect the Arduino to the computer.
2. Open person_tracker_arduino/person_tracker_arduino.ino in Arduino IDE.
3. Install AccelStepper through the Library Manager if needed.
4. Select the correct board and serial port.
5. Upload the sketch.
6. Close Serial Monitor before running Python so the program can open the port.
2. Install Python dependencies
Open a terminal in the folder containing person_tracker.py and requirements.txt.
On Windows:
py -m pip install -r requirements.txt
The requirements file includes ultralytics, opencv-python, pyserial, and lap.
Use the same Python installation for installing packages and running the program. If your Python command is python instead of py, substitute it in the commands below. Python 3.11 is a starting option for this project; dependency compatibility also depends on the packages installed.
The default model is yolov8n.pt. Place an existing copy in the project folder to reuse it; otherwise the first run may download the weights. Camera processing and model inference run on the computer.
3. Check detection without the motor
py person_tracker.py --no-motor
Stand in view and check that a tracking box appears. Click the camera window and press Q to close this test before starting another instance.
4. Start tracking
py person_tracker.py
The default configuration uses COM7, camera 1, a maximum speed of 1,200 pulses/second, and a gain of 1,600. Camera index 1 and the default direction match the original project setup.
For an initial motor check at a lower speed:
py person_tracker.py --max-speed 400
Faster tracking
To increase the speed limit and the response to an off-center person:
py person_tracker.py --max-speed 1600 --gain 2000
If this causes overshooting or repeated side-to-side correction, reduce the gain:
py person_tracker.py --max-speed 1600 --gain 1800
--max-speed sets the speed ceiling. --gain sets how strongly the controller responds to horizontal error. Increasing the ceiling alone may have little effect when the requested speed is below it. These command-line settings apply to that run; they do not change the saved defaults.
Controls
Click the camera window so it receives keyboard input.
Key	Action
Space	Pause or resume tracking
R	Release the current target and select again
Q or Esc	Quit and request an immediate stop of STEP pulses


Reselecting chooses the eligible person nearest the horizontal center, so it can select the same person again. The display shows the selected track, tracking status, requested speed, processing FPS, and frame age. Requested speed is not a measured motor velocity.
Command-line options
Option	Default	Description
--port	COM7	Arduino serial port
--camera	1	Camera index
--model	yolov8n.pt	Model weights path; tracking assumes the model uses class 0 for people
--imgsz	416	Inference image size; use a positive multiple of 32
--device	Not specified	Inference device, such as cpu or 0 for a configured CUDA GPU
--max-speed	1200	Maximum requested STEP pulses/second; accepted range is 1–2000
--gain	1600	Positive proportional control gain
--reverse	Off	Reverse the configured motor direction
--no-motor	Off	Run without opening the serial port


Examples:
py person_tracker.py --port COM7 --camera 1
py person_tracker.py --reverse
py person_tracker.py --imgsz 320
py person_tracker.py --help
The original project's camera index and direction already work with the defaults. The overrides allow the same program to be used on a different setup.
Controller settings
The horizontal error is normalized by half the image width:
error = (person_center_x - image_center_x) / (image_width / 2)
After smoothing, the speed magnitude is proportional to the error outside the center band and limited by --max-speed.
Setting	Default	Behavior
STOP_BAND	0.04	A new observation inside this band requests zero speed
START_BAND	0.07	A larger error is needed to restart motion
SMOOTHING_SECONDS	0.08 s	Time constant for error smoothing
TARGET_CONFIDENCE	0.45	Confidence required for a tracked box to command motion
TARGET_MEMORY_SECONDS	0.8 s	How long to retain a missing target's ID
MAX_FRAME_AGE_SECONDS	0.35 s	Maximum age of a vision observation used for motion


The center bands are fractions of half the frame width. At 640 pixels wide, the stop and start thresholds are approximately 13 and 22 pixels from the center.
Arduino motion settings:
Setting	Default
Maximum command speed	2000 pulses/s
Acceleration	4000 pulses/s²
Deceleration	6000 pulses/s²
Command timeout	300 ms


Speeds are STEP pulses per second, including the driver's microstepping. Actual camera rotation depends on motor steps per revolution, microstepping, and gearing. If the firmware speed ceiling changes, update FIRMWARE_MAX_SPEED in Python to match it.
Stop behavior and travel limits
- Target lost or paused: Python requests zero speed and the Arduino decelerates.
- Stale vision: The serial worker sends zero when the observation is older than 0.35 seconds, even if inference has stalled.
- Serial commands stop arriving: The Arduino begins decelerating after 300 ms without a valid velocity command.
- Quit: Python attempts to send S, which immediately stops pulse generation. The driver remains energized.
Stopping with deceleration takes additional time after the stop request. runSpeed() does not provide an automatic acceleration profile, so the sketch implements its own ramp.
Software travel limits are disabled by default. For a mount with limited cable or mechanical travel, calibrate MIN_POSITION and MAX_POSITION, then enable USE_SOFT_LIMITS in the sketch. The sample values of ±2,000 are pulse counts, not a verified angular range.
Position zero is assigned when the board resets, which may happen when Python opens the serial port. With software limits enabled, start from the intended centered position. The system has no homing routine or encoder feedback, so pulse counts cannot detect skipped steps or establish an absolute physical position.
Troubleshooting
Problem	What to check
Camera will not open	Confirm the index with --camera; close other applications using the camera
Serial port unavailable	Confirm --port, the USB connection, and that Serial Monitor is closed
Arduino handshake failed	Upload the matching sketch and confirm the board/port selection
Camera turns away from the person	Run with --reverse
Tracking overshoots	Lower --gain, then reduce --max-speed if needed
Motor buzzes or skips	Reduce speed/acceleration and check the driver settings, power supply, and mechanical load
“Frame too old” appears repeatedly	Try --imgsz 320 and reduce other processing load; a smaller inference image can reduce detection accuracy
Target changes during a crossing	ByteTrack IDs can change under occlusion; press R to select again
Keyboard controls do not respond	Click the camera window; keys are processed between inference calls


Scope and limitations
- Controls one horizontal pan axis; tilt and automatic search sweeps are not implemented.
- Uses temporary tracking IDs rather than facial recognition or guaranteed person identification.
- Detection and tracking depend on lighting, visibility, camera quality, and available processing power.
- The capture thread reduces application buffering, but the camera and driver can still introduce delay. Displayed frame age starts when OpenCV returns a frame, not at sensor exposure.
- The program requests 640 × 480 capture at 30 FPS; the camera backend may ignore those settings, and detector throughput depends on the computer.
Built with
- Python — application and control logic
- Ultralytics YOLOv8 nano — person detection
- ByteTrack — association of detections across frames
- OpenCV — capture, drawing, and display
- pySerial — communication with the Arduino
- Arduino / AccelStepper — STEP/DIR pulse generation and motor control
