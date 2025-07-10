# Ball Tracking Robot
This project leverages computer vision to detect and track a ball using a camera connected to a Raspberry Pi. The minicomputer also controls two DC motors on a car through an L298 motor driver.

| **Engineer** | **School** | **Area of Interest** | **Grade** |
|:--:|:--:|:--:|:--:|
| Aiden L | Palo Alto High School | Electrical Engineering | Incoming Sophomore

<img src="docs/assets/AidenL.png" width="400" height="533" />


# Second Milestone

<iframe width="560" height="315" src="https://www.youtube.com/embed/D0dFoDQxqu8?list=PLe-u_DjFx7eui8dmPGji-0-slT8KydYv_" title="Aiden L. Milestone 2" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

My second milestone deals with all computer vision. Here, I set up a live camera feed with `picamera2` and used `cv2` to find the center of the ball and approximate its radius. 
<img src="docs/assets/cv+demo.gif" width="1250" height="400" />

The main part of the code includes:<br>
1) **Image Capture**: Capturing a photo with `Picamera2.capture_array()` and converting its RGB data to HSV with `cv2.cvtColor()`;<br>
2) **HSV Thresholds**: Creating two HSV thresholds (upper and lower bounds) with `cv2.inRange()` and combining them with a bitwise OR (we use two thresholds because red wraps around hue in HSV from 1-10 and 170-180); this should result in a binary (black + white) mask with white pixels representing red pixels in the image;<br>
3) **Noise Filtering**: Average and erosion filters with `cv2.blur()` and `cv2.erode()` to filter out noise; at the end of this step, we should be left with all white pixels in the image representing the ball;<br>
4) **Ball Detection**: Finding contours (outlines made out of white pixels from the binary mask) with `cv2.findContours()`, and fetching the ball (contour with the largest area) using `cv2.contourArea()`;<br>
5) **Visualization**: Calculating the smallest circle that can surround the ball contour with `cv2.minEnclosingCircle()` and drawing the contour, circle, and its center with `cv2.circle()` and `cv2.drawContours()`.<br>

Two helper utilities were added to speed up testing:
- Live HSV Inspector: fetches the HSV of any pixel that I hover over
- Real-time Mask Adjustment: modify the HSV thresholds in real time by pressing keys to avoid rerunning code.


# First Milestone

<iframe width="560" height="315" src="https://www.youtube.com/embed/kIgdshT56f0?list=PLe-u_DjFx7eui8dmPGji-0-slT8KydYv_" title="Aiden L. Milestone 1" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

My first milestone aims to assemble the main components of the ball tracking robot, which includes: 
- **Chassis**: as the base of the car.
- **L298N Motor Driver**: uses two H-bridges to allow changes in speed and direction for two motors.
- **Raspberry Pi**: a mini computer to bring all the components together on a high level, processing images and steering the robot. 
- **Pi Camera**: a small camera to use for object detection
- **Breadboard**: to connect components without soldering, right now only used to ground the Pi and battery pack together.
- **Battery Pack**: to power the motors.

| ![Assembly](docs/assets/milestone1.png)
|:--:|
|Assembly of the main components|

The wiring is quite simple; the Raspberry Pi and the battery pack are grounded together on the breadboard, then connected into the GND input of the L298N motor driver. Four GPIO pins from the Pi are connected to the IN1-IN4 ports of the L298N. The battery pack's positive lead is connected to the 12V input of the driver. Finally, the four output pins of the motor driver are wired to two motors attached to the wheels.

Additionally, I tested the camera with `libcamera-vid -t 0` and a VNC connection to view the preview, and tested the motors with a simple script.

But although relatively simple, this milestone took six days to complete. This was primarily due to numerous mistakes with soldering wires to components (ex. motors, power button), causing the wires to fall off or snap, as well as problems with the Raspberry Pi (had to replace it and the SD card) and a defect in my L298N missing its regulator jumper, making it unable to power on with my two inputs (12V and GND). Luckily there is not much hardware integration in the future milestones, so it should (hopefully) be "smoother" sailing from here. 

I plan to start implementing object detection and the provided ultrasonic sensors to identify and track the ball next. 

# Starter Project
<iframe width="560" height="315" src="https://www.youtube.com/embed/1uRjqW9JUWU?si=RzlLivTnppujge-3" title="Aiden L. Starter Project" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>


For my starter project I soldered an RGB slider together, which involved three sliders, an LED light, and a USB-C connector soldered to a PCB. 
The sliders act as **potentiometers**, or variable resistors, and can increase or decrease the current (by adjusting resistance) through each LED channel (red, green, and blue). By blending the current for red, green, and blue inside the LED, we can mix colors. 

| ![Slider](docs/assets/slider.png) |
|:--:|
|Blending red and blue to make purple|

# Schematics

| ![Milestone 1](docs/assets/schematic_m1.png) |
|:--:|
|Schematic for milestone 1|

# Code

`motors.py`:
```py
import pigpio
import time

pi = pigpio.pi()

class Motor:
    def __init__(self, in1, in2, pwm):
        self.in1 = in1
        self.in2 = in2
        self.pwm = pwm
        pi.set_mode(self.in1, pigpio.OUTPUT)
        pi.set_mode(self.in2, pigpio.OUTPUT)
        pi.set_mode(self.pwm, pigpio.OUTPUT)
        
        self.cold = True
        
    def normalize(self, speed):
        speed = max(0, min(speed, 1))  # Ensure speed is between 0 and 1
        return int(speed * 255)
    
    def output_pwm(self, speed):
        speed = self.normalize(speed)
        if speed < 100 and self.cold:  # "kickstart" the PWM with a high value to get it moving
            print("Kickstarting PWM")
            pi.set_PWM_dutycycle(self.pwm, 255)
            time.sleep(0.01)
        pi.set_PWM_dutycycle(self.pwm, speed)

    def forward(self, speed):  # Speed 0-1
        pi.write(self.in1, 1)
        pi.write(self.in2, 0)
        
        self.output_pwm(speed)

    def backward(self, speed):
        speed = self.normalize(speed)
        pi.write(self.in1, 0)
        pi.write(self.in2, 1)
        
        self.output_pwm(speed)

    def stop(self):
        pi.write(self.in1, 0)
        pi.write(self.in2, 0)
        pi.set_PWM_dutycycle(self.pwm, 0)

IN1 = 24
IN2 = 23
IN3 = 22
IN4 = 27
ENA = 12
ENB = 13

LEFT = Motor(IN1, IN2, ENA)
RIGHT = Motor(IN3, IN4, ENB)

def stop():
    LEFT.stop()
    RIGHT.stop()

def cleanup():
    stop()
    pi.stop()
```
## Notes
- `pigpio` is already imported. Run `sudo pigpiod` every time you boot up the pi (or set up cron) so the pigpio daemon has access to your GPIO.
- `Motor` class: takes in two GPIO pin numbers and the enable pin number.
  - `normalize()` takes in a proportion of power from [0-1] and sets it in the range [0-255] to write to PWM (with some range checking).
  - `output_pwm()` takes in a proportion of power from [0-1] and writes to PWM, "jumpstarting" the PWM by sending a high value for a very short amount of time if the proportion is too low.
  - `forward()`/`backward()` methods take in a proportion of power from [0-1].
- The ports use Broadcom (aka BCM) GPIO numbers.
- Wrap a `try`/`finally` loop in all your programs that use these motors, always call `cleanup()` in your `finally` clause.
  
