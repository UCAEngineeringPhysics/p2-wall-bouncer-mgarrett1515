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
 from time import sleep, sleep_us, sleep_ms, ticks_us
from machine import Pin, PWM
from dual_motor_driver import DualMotorDriver
from distance_sensor import DistanceSensor

# SETUP

ds = DistanceSensor(trig_id=3, echo_id=2)

dmd = DualMotorDriver(left_ids=(15, 13, 14), right_ids=(16, 18, 17), stby_id=12)
dmd.enable()



ledRed = PWM(Pin(28))
ledBlue = PWM(Pin(26))
ledGreen = PWM(Pin(27))

ledRed.freq(1000)
ledBlue.freq(1000)
ledGreen.freq(1000)

button = Pin(21, Pin.IN, Pin.PULL_UP)


def mode_switch(pin):
    global modeValue
    modeValue += 1
    
button.irq(trigger= Pin.IRQ_RISING, handler=mode_switch)

modeValue = 1


firstDutyIncrease = 655
firstDutyDecrease = 655
firstDutyValue = 0

secondDutyValue = 0
secondDutyIncrease = 655
secondDutyDecrease = 655

dutyValue = 0
dutyIncrease = 655
dutyDecrease = 655

workTimeCounter = 0
resetCounter = 0

# LOOP

# 1. All three LEDs need to blink at a rate of 5Hz for 2 seconds.

#The two for loops nested in the first for loop run for a total of 0.2 seconds, so making
#this last 2 seconds can be done by running the two for loops 10 times.

while True:
    d = ds.distance
    if modeValue == 1 and d != None:
        for r in range(10):
            for i in range(100):
                ledRed.duty_u16(dutyValue)
                ledBlue.duty_u16(dutyValue)
                ledGreen.duty_u16(dutyValue)
                dutyValue += dutyIncrease
                sleep(1/1000)
            for i in reversed(range(100)):
                ledRed.duty_u16(dutyValue)
                ledBlue.duty_u16(dutyValue)
                ledGreen.duty_u16(dutyValue)
                dutyValue -= dutyDecrease
                sleep(1/1000)
        break
ledRed.duty_u16(0)
ledBlue.duty_u16(0)
ledGreen.duty_u16(0)


# 2/3. Switching between pause mode and work mode.

while True:
#     d = ds.distance
#     print(d)
    if modeValue % 2 != 0:
        dutyValue = 0
        for i in range(100):
            ledGreen.duty_u16(dutyValue)
            dutyValue = dutyValue + dutyIncrease
            sleep(0.5/100)
        for i in reversed(range(100)):
            ledGreen.duty_u16(dutyValue)
            dutyValue = dutyValue - dutyDecrease
            sleep(0.5/100)
    else:
        ledGreen.duty_u16(65535)
        print(workTimeCounter)
        d = ds.distance
        print(d)
        
        if d != None and d <= 0.35:
            dmd.stop()
            sleep(0.5)
            dmd.linear_backward(0.2)
            sleep(0.5)
            dmd.stop()
            dmd.spin_right(0.2)
            sleep(1)
            dmd.stop()
            workTimeCounter += 2.4
        elif d != None and d > 0.35:
            dmd.linear_forward(0.5)
            
        sleep(0.1)
        workTimeCounter += 0.1
        
#         d = ds.distance
#         print(d)
        if modeValue % 2 != 0:
            print("Pausing")
            dmd.stop()
            continue
        if workTimeCounter >= 44 and workTimeCounter <= 46:
            print("Battery Low")
            break

ledGreen.duty_u16(0)

# 4. Work time at 45 seconds.

while True:
#     d = ds.distance
#     print(d)
    if modeValue % 2 != 0:
        dmd.stop()
        dutyValue = 0
        for i in range(100):
            ledBlue.duty_u16(dutyValue)
            dutyValue = dutyValue + dutyIncrease
            sleep(0.5/100)
        for i in reversed(range(100)):
            ledBlue.duty_u16(dutyValue)
            dutyValue = dutyValue - dutyDecrease
            sleep(0.5/100)
    else:
        ledBlue.duty_u16(65535)
        sleep(0.2)
        workTimeCounter += 0.2
        print(workTimeCounter)
        d = ds.distance
        print(d)
        
        dmd.linear_forward(0.4)
        
        if d != None and d <= 0.35:
            dmd.stop()
            sleep(0.5)
            dmd.linear_backward(0.2)
            sleep(0.5)
            dmd.stop()
            dmd.spin_right(0.2)
            sleep(1)
            dmd.stop()
            workTimeCounter += 2.4
        elif d != None and d > 0.35:
            dmd.linear_forward(0.25)
        
#         d = ds.distance
#         print(d)
        if modeValue % 2 != 0:
            print("Pausing")
            dmd.stop()
            continue
        if workTimeCounter >= 54 and workTimeCounter <= 56:
            print("Battery Critical")
            break

ledBlue.duty_u16(0)
dutyValue = 0

# 4. (again) Work time at 55 seconds.

while True:
#     d = ds.distance
#     print(d)
    if modeValue % 2 != 0:
        secondDutyValue = 0
        for i in range(100):
            ledRed.duty_u16(dutyValue)
            dutyValue += dutyIncrease
            sleep(1/2000)
        for i in reversed(range(100)):
            ledRed.duty_u16(dutyValue)
            dutyValue -= dutyDecrease
            sleep(1/2000)
        resetCounter += 0.005
        print(resetCounter)
        if resetCounter >= 5 and resetCounter <= 5.01:
            machine.reset()
    else:
#         ledBlue.duty_u16(65535)
        secondDutyValue = 0
        for i in range(100):
            ledRed.duty_u16(secondDutyValue)
            secondDutyValue += secondDutyIncrease
            sleep(1/2000)
        for i in reversed(range(100)):
            ledRed.duty_u16(secondDutyValue)
            secondDutyValue -= secondDutyDecrease
            sleep(1/2000)
        
        d = ds.distance
        print(d)
        
        dmd.linear_forward(0.4)
        
        if d != None and d <= 0.35:
            dmd.stop()
            sleep(0.5)
            dmd.linear_backward(0.2)
            sleep(0.5)
            dmd.stop()
            dmd.spin_right(0.2)
            sleep(1)
            dmd.stop()
            workTimeCounter += 2.4
        elif d != None and d > 0.35:
            dmd.linear_forward(0.25)
        
        resetCounter += 0.005
        print(resetCounter)
#         d = ds.distance
#         print(d)
        if resetCounter >= 5:
            machine.reset()
        if modeValue % 2 != 0:
            dmd.stop()
            continue

ledRed.duty_u16(0)
ledBlue.duty_u16(0)
ledGreen.duty_u16(0)


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
> - 

https://github.com/user-attachments/assets/d89c975c-559d-4612-9be3-ee0cf1d38233



### 3. (35%) Documentation

#### 3.1. (15%) Mechanical Design: attach (multiple) technical drawings to illustrate dimensions and locations of the key components of the mobile base. 
- Denote dimensions of the bed.
- Denote dimensions and locations of the wheel assembly and the caster wheel.
- Denote locations of the mounting holes.
- Denote dimensions of the mounting holes.
![IMG_6065](https://github.com/user-attachments/assets/e8336aac-b30d-43f0-90b5-414966ef6ac1)
![IMG_6063](https://github.com/user-attachments/assets/59e7e4b4-1df7-4e80-97d9-ff1a74622981)
![IMG_6064](https://github.com/user-attachments/assets/202cf84b-191d-4ceb-9604-87e2e2ffe283)
![IMG_6066](https://github.com/user-attachments/assets/bc0cd7ca-9bc5-46a3-bd3a-cb276cf3fa72)
![IMG_6067](https://github.com/user-attachments/assets/efdeac4e-6fc4-45be-84b8-939cf84b158c)

> [!TIP]!

> - You may want to checkout TechDraw of FreeCAD. Other CAD software should have the similar tools.  
> - Hand drawings are acceptable.

#### 3.2 (10%) Wiring Diagram: attach a drawing to illustrate electrical components' wiring.
- Specify power wires using red and black wires.
- Mark out employed signal pins' names.
- Electronic components' values have to match your actual circuit.!
![Screenshot 2025-12-14 175656 wiring p2](https://github.com/user-attachments/assets/47595704-d886-49b3-9d8b-4526b729a02e)


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
