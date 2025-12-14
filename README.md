[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/LMWu6GmP)
# Project 2: Wall Bouncer

## Background
[Roomba](https://www.irobot.com/en_US/roomba.html) is a very popular housekeeping robot. 
Despite the new technologies introduced to recent products, the "navigation" strategy of this robot can be fairly simple. 
Inspired by the Roomba, we are going to develop a robot that navigates/bounces in a closed cell. 
A human robot interface (HRI) from first project will be also integrated. 

## Requirements:
> [!IMPORTANT]
> Demonstrate your robot to Dr. Zhang to redeem your credits.

### 1. Assemble the Robot
Make sure every component is functional as expected.
Major required components are listed below:

| Name  | Qty. | Functionality
| ------------- | ------------- | ------------- |
| Mobile Base  | 1 | To host other physical components of the robot |
| 18650 Battery  | 2 | Power all the electrical and electronic components |
| Power Expansion Board | 1 | Converts 6~24V to 5V. Split input power. |
| Micro-GearMotor | 2 | Robot's actuator |
| Wheels | 2 | Attach to the motors and maneuver the robot |
| Raspberry Pi Pico | 1 | Send out and read in signals. Process and make decisions |
| TB6612FNG Motor Driver Board | 1 | Control motor speed following PWM signals |
| Ultrasonic Sensor | 1 | Sense distance to wall |
| Common Cathode LED  | 1 | Indicate robot's status |
| Tactile switch button  | 1 | Switch robot's working mode |

> [!WARNING]
> Off-the-shelf rubber tires and wheels are no long allowed (use 3D printed wheels and tires).

> [!TIP]
> Things you may want to check are listed but not limited below:
> - Have you double checked circuit and no short circuit anywhere?
> - Have you crimped connectors to motor wires?
> - Have you charged the batteries?
> - Are [Pico, LEDs and button](https://linzhanguca.github.io/_docs/robotics1-2025/0902/pico.pdf) functional?
> - Is [distance sensor](https://github.com/linzhangUCA/3421example-ultrasonic_sensor) functional?
> - Are [motor driver](https://github.com/linzhangUCA/3421example-motor_control) and motors controllable?
> - Are all the components fit into the bed? If not, print one from [here](https://github.com/linzhangUCA/3421example-robot_assembly/tree/main/prints).


### 2. (65%) Coding
- Program the Raspberry Pi Pico to: 
    - Encode robot's status into colors (`RED`, `GREEN`, `BLUE`) using LEDs .
    - Switch robot's behavior between `WORK MODE` and `PAUSE MODE` using a button.
    - **Read distance from ultrasonic sensor**.
    - **Send signals to motor driver board and move the robot according to the distance sensing**.
- Upload your script to this repository.
```
  import utime
from machine import Pin, PWM, Timer, reset
from dif_drive_controller import DiffDriveController
from distance_sensor import DistanceSensor

# ==========================================
# 1. CONFIGURATION
# ==========================================

# --- Pin Definitions ---
BTN_PIN = 22
LED_RED_PIN = 26
LED_GREEN_PIN = 27
LED_BLUE_PIN = 28

# Motor Pins
# LEFT: (PWM, IN1, IN2) | Encoders (A, B)
LEFT_WHEEL_CONFIG = ((15, 13, 14), (11, 10)) 

# RIGHT: (PWM, IN1, IN2) | Encoders (A, B)
RIGHT_WHEEL_CONFIG = ((16, 17, 18), (19, 20))

# Sensor Pins
TRIG_PIN = 9
ECHO_PIN = 8

# --- Constants ---
SPEED_FWD = 0.15     # m/s (Slow Forward)
SPEED_SPIN = 0.15    # m/s (Wheel speed during spin)

# Timing (Milliseconds)
TIME_TO_TURN = 800   # Duration of spin
TIME_TO_STOP = 500   # Rest duration

WALL_THRESHOLD = 0.25 # meters

# ==========================================
# 2. HELPER CLASSES
# ==========================================

class LEDController:
    def __init__(self, r_pin, g_pin, b_pin):
        self.r = PWM(Pin(r_pin))
        self.g = PWM(Pin(g_pin))
        self.b = PWM(Pin(b_pin))
        for led in [self.r, self.g, self.b]:
            led.freq(1000)
            led.duty_u16(0)
        self.fade_timer = Timer()
        self.blink_timer = Timer()
        self.fading = False
        self.fade_val = 0
        self.fade_step = 1000

    def off_all(self):
        self.fading = False
        self.fade_timer.deinit()
        self.blink_timer.deinit()
        self.r.duty_u16(0)
        self.g.duty_u16(0)
        self.b.duty_u16(0)

    def set_color(self, r, g, b):
        self.off_all()
        self.r.duty_u16(65535 if r else 0)
        self.g.duty_u16(65535 if g else 0)
        self.b.duty_u16(65535 if b else 0)

    def blink(self, color_tuple, freq):
        self.off_all()
        period_ms = int(1000 / freq / 2)
        r, g, b = color_tuple
        def _cb(t):
            curr_r = self.r.duty_u16()
            target = 65535 if curr_r == 0 else 0
            if r: self.r.duty_u16(target)
            if g: self.g.duty_u16(target)
            if b: self.b.duty_u16(target)
        self.blink_timer.init(period=period_ms, mode=Timer.PERIODIC, callback=_cb)

    def fade_green(self):
        self.off_all()
        self.fading = True
        self.fade_val = 0
        self.fade_step = 1300
        def _cb(t):
            if not self.fading: return
            self.fade_val += self.fade_step
            if self.fade_val >= 65535 or self.fade_val <= 0:
                self.fade_step = -self.fade_step 
            val = max(0, min(65535, self.fade_val))
            self.g.duty_u16(val)
        self.fade_timer.init(freq=50, mode=Timer.PERIODIC, callback=_cb)

# ==========================================
# 3. MAIN APPLICATION
# ==========================================

def main():
    print("Initializing Wall Bouncer (Manual Override)...")
    
    leds = LEDController(LED_RED_PIN, LED_GREEN_PIN, LED_BLUE_PIN)
    button = Pin(BTN_PIN, Pin.IN, Pin.PULL_DOWN) 
    sensor = DistanceSensor(echo_id=ECHO_PIN, trig_id=TRIG_PIN)
    
    # Initialize Drive
    drive = DiffDriveController(LEFT_WHEEL_CONFIG, RIGHT_WHEEL_CONFIG)
    drive.enable()

    # --- Modes ---
    MODE_PAUSE = 0
    MODE_WORK = 1
    current_mode = MODE_PAUSE
    
    # --- Navigation States ---
    NAV_STATE_FWD = 0
    NAV_STATE_STOP_1 = 1
    NAV_STATE_TURN = 2
    NAV_STATE_STOP_2 = 3
    
    nav_state = NAV_STATE_FWD
    state_start_time = 0 
    
    # --- System Variables ---
    work_start_time = 0
    accumulated_work_time = 0
    last_btn_press = 0
    
    def button_handler(pin):
        nonlocal current_mode, last_btn_press, work_start_time, accumulated_work_time, nav_state
        curr_time = utime.ticks_ms()
        if utime.ticks_diff(curr_time, last_btn_press) > 300: 
            last_btn_press = curr_time
            if current_mode == MODE_PAUSE:
                print(">> Switch to WORK")
                current_mode = MODE_WORK
                work_start_time = utime.ticks_ms()
                nav_state = NAV_STATE_FWD
            else:
                print(">> Switch to PAUSE")
                current_mode = MODE_PAUSE
                accumulated_work_time += utime.ticks_diff(utime.ticks_ms(), work_start_time)
                drive.set_vels(0, 0)

    button.irq(trigger=Pin.IRQ_RISING, handler=button_handler)

    # --- System Check ---
    print("System Check...")
    utime.sleep(0.5) 
    leds.blink((1, 1, 1), 5) 
    utime.sleep(1)
    leds.off_all()
    print("Ready.")
    leds.fade_green()

    try:
        while True:
            current_time = utime.ticks_ms()
            
            # 1. PAUSE MODE
            if current_mode == MODE_PAUSE:
                if not leds.fading: leds.fade_green()
                drive.set_vels(0, 0)
                
                # Visuals
                total_work = accumulated_work_time / 1000.0
                if 45 < total_work < 55: leds.set_color(0, 0, 1)

            # 2. WORK MODE
            elif current_mode == MODE_WORK:
                session_time = utime.ticks_diff(current_time, work_start_time)
                total_work = (accumulated_work_time + session_time) / 1000.0
                
                # Battery Logic
                if total_work > 60:
                     print("Shutdown."); drive.disable(); leds.off_all(); reset()
                elif total_work > 55:
                    leds.blink((1, 0, 0), 10); active_speed = 0.1
                elif total_work > 45:
                    leds.set_color(0, 0, 1); active_speed = 0.1
                else:
                    leds.set_color(0, 1, 0); active_speed = SPEED_FWD

                # --- SENSOR LOGIC ---
                raw_dist = sensor.distance
                
                # Aggressive Filter:
                # If None -> Far away (Safe)
                # If 0.0  -> Wall (Danger - treat as Close)
                if raw_dist is None:
                    dist = 10.0
                elif raw_dist == 0.0:
                    dist = 0.0 # Danger!
                else:
                    dist = raw_dist

                # --- STATE MACHINE ---
                
                if nav_state == NAV_STATE_FWD:
                    # Drive Straight using Controller
                    drive.set_vels(active_speed, 0)
                    
                    # If wall detected, switch state
                    if dist < WALL_THRESHOLD:
                        print(f"Wall ({dist:.2f}m) -> Stopping")
                        nav_state = NAV_STATE_STOP_1
                        state_start_time = current_time 

                elif nav_state == NAV_STATE_STOP_1:
                    # Kill motors
                    drive.set_vels(0, 0)
                    
                    if utime.ticks_diff(current_time, state_start_time) > TIME_TO_STOP:
                        print("Turning Left (Manual)...")
                        nav_state = NAV_STATE_TURN
                        state_start_time = current_time

                elif nav_state == NAV_STATE_TURN:
                    # *** MANUAL OVERRIDE ***
                    # Bypass the DiffDrive controller math.
                    # Force Left Backward (-), Right Forward (+)
                    drive.left_wheel.set_wheel_velocity(-SPEED_SPIN)
                    drive.right_wheel.set_wheel_velocity(SPEED_SPIN)
                    
                    if utime.ticks_diff(current_time, state_start_time) > TIME_TO_TURN:
                        print("Turn Done -> Stabilizing")
                        nav_state = NAV_STATE_STOP_2
                        state_start_time = current_time

                elif nav_state == NAV_STATE_STOP_2:
                    # Kill motors
                    drive.set_vels(0, 0)
                    
                    if utime.ticks_diff(current_time, state_start_time) > TIME_TO_STOP:
                        print("Resuming Forward")
                        nav_state = NAV_STATE_FWD

            utime.sleep_ms(20) 

    except KeyboardInterrupt:
        drive.disable()
        leds.off_all()

if __name__ == "__main__":
    main()
```
- Complete following tasks:
1. (3%) Initialization (System Check).
   - (2%) Blink all LEDs with frequency of 5 Hz, lasting 2 seconds when both conditions below are satisfied.
     - The button's GPIO pin is receiving correct default signal (`0` for `PULL_DOWN`, `1` for `PULL_UP`)
     - Ultrasonic sensor is receiving non-zero distance measuring.
   - (1%) The robot enters `PAUSE MODE` after this step.
2. (16%) When `PAUSE MODE` is activated:
   - (3%) `GREEN` LED fades in and fades out at frequency of 1 Hz (equally allocate fade-in and fade-out time).
   - (3%) Press the button to **immediately** switch to the `WORK MODE`.
   - (10%) Robot stop moving
3. (24%) When `WORK MODE` is activated:
   - (1%) `GREEN` LED stays constantly on.
   - (3%) Press the button to **immediately** switch to the **PAUSE MODE**.
   - (20%) Robot start moving without hitting the wall.
4. (20%) Low battery simulation.
   - (5%) If the accumulated `WORK MODE` time exceeds 45 seconds, substitute `GREEN` LED with **`BLUE`** LED for both modes (low-battery simulation).
   - (5%) If accumulated `WORK MODE` time over 55 seconds, blink `RED` LED at frequency of 10 Hz (`BLUE` and `GREEN` LED off).
   - (10%) If the accumulated `WORK MODE` time exceeds 45 seconds, Use 50% dutycycle of the original to be the robot's speed (Make sure the robot is still movable). 
5. (2%) Termination. Shutdown the system after the `RED` LED blinked 5 seconds.

> [!IMPORTANT]
> - It doesn't matter how your robot moves, but hitting a wall once during demonstration will cost 1% off your grade.
> - Plan a good strategy of wall avoidance.

> [!TIP]
> - Break tasks down into small pieces (the smaller the better). You may need write a handful of unit test scripts.
> - `print()` function and Python Shell are handy tools.

### 3. (35%) Documentation

#### 3.1. (15%) Mechanical Design: attach (multiple) technical drawings to illustrate dimensions and locations of the key components of the mobile base. 
- Denote dimensions of the bed.
- Denote dimensions and locations of the wheel assembly and the caster wheel.
- Denote locations of the mounting holes.
- Denote dimensions of the mounting holes.

> [!TIP]![base p2](https://github.com/user-attachments/assets/9b349ac3-f3a8-40bc-b98c-337066a0f343)
![other_dimensions_p2](https://github.com/user-attachments/assets/f3f4ff4e-46c1-4b38-8832-cc73130f51c2)

> - You may want to checkout TechDraw of FreeCAD. Other CAD software should have the similar tools.  
> - Hand drawings are acceptable.

#### 3.2 (10%) Wiring Diagram: attach a drawing to illustrate electrical components' wiring.
- Specify power wires using red and black wires.
- Mark out employed signal pins' names.
- Electronic components' values have to match your actual circuit.![p2 wiring](https://github.com/user-attachments/assets/9d2f8f07-99b8-44a6-9a9c-7906b71af131)

- 


#### 3.3 (6%) Software Design
Use a [flowchart](https://en.wikipedia.org/wiki/Flowchart) or a [algorithm/pseudocode table](https://www.overleaf.com/learn/latex/Algorithms) or a [itemized list](https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax#lists) to explain your wall avoidance strategy.
State Initialization: The robot begins in NAV_STATE_FWD (Forward Mode) immediately upon entering "Work Mode".

Distance Monitoring:

The ultrasonic sensor continuously polls distance in real-time (every ~20ms).

Filtering: Readings of None (infinity) are treated as safe (10.0m). Readings of 0.0 (noise/too close) are treated as an immediate collision danger (0.0m).

Forward Motion:

The robot drives straight at a set speed (0.18 m/s normally, 0.09 m/s in low battery).

Transition Trigger: If the filtered distance drops below WALL_THRESHOLD (0.25 meters), the robot immediately transitions to NAV_STATE_STOP_1.

Collision Avoidance Sequence:

Step 1: Stop (Stabilize): The robot halts all motor movement for TIME_TO_STOP (500ms) to dissipate momentum and ensure a clean turn.

Step 2: Turn (Maneuver): The robot enters NAV_STATE_TURN and performs a manual differential turn (Left Wheel Backward, Right Wheel Forward) for TIME_TO_TURN (800ms) to execute an approximately 90-degree left turn.

Step 3: Stop (Stabilize): The robot halts again for TIME_TO_STOP (500ms) to prevent wheel slip before accelerating.

Resume: The robot returns to NAV_STATE_FWD and resumes driving straight, repeating the cycle.
#### 3.4 (4%) Energy Efficient Path Planning 
> The goal is using this robot to cover a rectangle-shape area.
> Do your research, make reasonable assumptions and propose a path pattern for the robot to follow.
> Please state why this pattern is energy efficient.
> Proposed Path Pattern: The "Boustrophedon" (Lawnmower) Path For a rectangular-shaped area, the most energy-efficient coverage strategy is the Boustrophedon path, commonly known as the "lawnmower pattern." This involves the robot moving in long straight lines parallel to the longest side of the rectangle, turning 180 degrees (U-turn) at the boundaries, and overlapping slightly with the previous pass.
>
> It Minimizes Turning, Reduces Overlap, and helps maintain a Constant Velocity
