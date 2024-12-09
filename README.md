# t19ese5190f24

This is the public final project report for Team 19 in ESE 5190 Fall 2024 at the University of Pennsylvania.

    * Team Name: Shorts and Sparks
    * Team Members: Tim Zhang and James Steeman
    * Description of hardware: Atmega328PB, M2 Macbook Pro

## Final Project Report

### 1. Video

[Google Drive Video](https://drive.google.com/file/d/1hAN2IEJbra3G3HhbRzlzstQ-OAMDEnQX/view?usp=drive_link)

### 2. Images

Demo Day picture:
![alt text](<images/Demo Day Image.JPG>)
Left Center: James, Right Center: Tim

Annotated image of components placed above the ceiling:
![alt text](<images/Annotated Above Ceiling.png>)

Annotated image of turret below the ceiling:
![alt text](<images/Annotated Below Ceiling.png>)

Final Hardware Diagram:
![alt text](<images/5190_FP_Final_Hardware_Diagram.png>)
Note: Camera damage prevented camera code from implementation in final integration (see below for more infomation)

Final Code Diagram:
![alt text](<images/5190_FP_Final_Code_Diagram.drawio.png>)
Note: Camera damage prevented camera code from implementation in final integration (see below for more infomation)

### 3. Results

The turret features 2 degrees of freedom, one azimuth rotation and one elevation rotation. This combined motion allows it to point anywhere in a half sphere below the ceiling. The azimuth control uses a NEMA 17 motor with a 4:1 internal gear reduction and acceleration control to prevent step skipping and enable greater top speed. This allows the platform itself to easily hit over 3 rotations every second, well beyond our expected maximum of 1 rps. The elevation control uses a high torque servo with a barrel connected to the arm to direct the water jet.

Water is pumped using a 12V gear pump that is both self priming and reversible. A rotary union and tubing is used to transport water across the rotating element. A nozzle attachment enables powerful water jets over 10m.

On board buzzers are programmed to produce an obnoxious fire alarm.

This final solution/result was essentially the same as the proposed; however, due to the damage to the camera's themopile array, autonomous mode is currently not available for the turret. We are planning to complete this implementation with a replacement camera after the course.

For demonstration purposes, the turret is pre-programmed to known directions of the targets, and is able to consistently hit targets at various distances up to a 2 meters radius, over 4 times the original design requirements. This is shown in the video demo where the turret easily blasts mock “fire” targets.

#### 3.1 Software Requirements Specification (SRS) Results

SRS 01 – Motor control functionality shall be implemented to enable smooth, fast and accurate motion over the complete range of turret motion.

- **Achieved.**
- Measurements were made by setting a position and rotating the turret to that position. In the process, there was no step skipping from the stepper, and the servo would also reach its position. This was implemented separately for stepper and servo.
- The servo had no issues being smooth and fast, but had accuracy issues as it was not perfectly linear. Characterization data was created where the duty cycles were swept across the entire range of motion and the physical output arm angle was recorded. This resulted in a slightly cubic relationship and allows our calibrated servo to be accurate of around ± 1.5 degrees.
- The stepper faced no accuracy issues as it is inherently designed to “step” at constant intervals. However, they faced issues of step skipping at higher torque. In order to decrease torque to retain step accuracy and smoothness without sacrificing speed, acceleration profiles were used where the stepper will try to linearly accelerate to the desired speed at some fixed acceleration(and deceleration). This method allows for extremely smooth operation while retaining speed. Tests successfully rotated the platform at 300rpm/sec acceleration and 150rpm max speeds,  well above our initial goal of an 60rpm/1rps average.

SRS 02 - Thermal imaging data shall be acquired repeatedly as the turret rotates (azimuth).

- **Not achieved.**
- I2C communication with MLX90640 thermal camera (32x24 thermopile array) established successfully. Calibration parameters were obtained from camera EEPROM, status registers were configured successfully, and both frame constants and updating frame data was read from camera RAM. scan() was written and tested to read in values for the entire frame for each time the turret would stop to take and process a thermal image in scanning mode. align() was written for similar process in the correction term alignment stage of moving the turret to point at a detected thermal hotspot.
- At some point before the final demo, the camera's thermopile array was damaged (when we first recieved the camera we tested it with an arduino library, which gave very reasonable temperature values and from which we were able to see hand shapes by printing to Serial monitor, later testing with the same library and circuit gave 0˚C reading directly next to pixels reading >300˚C with no in-between values anywhere in the frame). This damage likely occured because we purchased the version without a breakout board to save $20 of budget during our first order. In hindsight, spending the extra money on the breakout board for built-in protections was probably the better choice for our first time working with such a sensitive component.
- In the course of the project, we discovered that this camera requires more resources than the ATmega328PB has available. The first wall was storing the raw data of a frame, as the 768 pixels at 2 bytes of data per pixel would require over 1.5K bytes on the stack, nearly the entirety of the approx 1.7Kb stack on the ATmega's 2K SRAM. This issues was solved by decreasing the number of pixels read, stored and processed at the same time. This increased available resources by using the same smaller array over and over to store values for data processing, as our proccessing was fairly simple, just checking if temperature exceeded a threshold and storing the cooresponding pixel as the return position error value. The data structure and processing in the align() function was slightly more complex, but the same idea was used to minimize resource usage. Additionally, pointers and pass by reference was used heavily so unnecessary copies of variables and data structures did not eat up stack space, particularly where multiple functions would be called inside on another.
- Similarly, the camera's EEPROM stored over 800 2 byte values from which many 3 to 8 bit calibration parameters needed to be recovered. In the manufaturer's API, these values were read and processed on runtime, but this was impossible on our ATmega, as we simply did not have enough space to store these values on runtime. To get around this, all of the EEPROM values were read without being stored while connected to a logic analyser. From the logic analyser software,A .csv file of the session was pulled and parsed with python scripts to obtain the desired values and there corresponding addresses in a more useful structure. Based on the datasheet's information on the paramters, their addresses, and the math necessary to recover them, a mix of bit manipulation and math in c and manual conversion techniques were used to recover all necessary values. These values were then included as macros or in constant arrays in our MLX90640.c, as we had much more than sufficient flash memory to include the macro values on compile time and put the sturcuted values in read only memory. Having values in arrays and 2D arrays allowed row, column, and pixel specific parameters to be easily accessed by simply knowing which column and row coorespond to the current pixel (say in a nested for loops for sequentially reading and calibrating temperature readings).
- Ultimately, the necessity for going to many significant decimal points in the math for calibration parameters and recovering temperature data was an insurmountable roadblock on our ATmega. The attempt to use int32_t integer math with *1000 /1000 to retain some decimal places was unsuccessful due to overflowing before reaching close to the number of decimal points such that calculations would provide sufficiently accurate temperature results. Simply using float math on our ATmega was neither sufficient nor reasonalbe as the software simulated float math can take thousands of clock cycles to perform float divisions, and there would be many thoughsands of floating point divisions necessary in the recovery of parameters and calibration calculations for each frame of data. This would be far too limiting on the operating capability of the turret.
- Using a ARM cortex M4 on the itsy bitsy M4 was briefly considered, as by reading the datasheet it seemed possible to implement a new I2C driver and reuse the already nearly complete MLX90640 library with much better performance as the clock speed would be 8x faster, I2C FM+ mode would be possible, additional SRAM was available, and an FPU would make float math viable, however this was not completed as the damage to the camera was discovered at about the same time as this idea.

SRS 03 – In seeking/pointing state, the control logic shall account projectile motion so that the water lands on the flame.

- **Deemed unnecessary**
- Updated pump + nozzle shoots water at a near straight path, this simplifies aiming and calculation, eliminating the need for the distance sensor entirely. When running some preliminary tests, the water jet was able to shoot (accidentally) across the entire length of the of detkin. Further tests showed that at around 4m, the drop due to gravity was only around 15cm. This means that within our designed limits of a 0.5m radius circle directly below, projectile motion is negligible. In fact, for the video demo, targets were shown to be accurately hit at over 2 meters.

SRS 04 – Image processing functionality shall locate pixels above threshold temperature upon each collection of thermal imaging data.

- **Not achieved.**
- Camera Damage prevented this from being integrated into the final implementation.(See SRS 02 for more information on the damage and work with the thermal camera). Code was written for this functionality, but was not properly tested due to the ATmega328PB limitations and damaged camera.

SRS 05 – The system shall switch correctly between sensing state, seeking/pointing state, and firing state.

- **Not achieved.**
- Though main() turret control pseudocode was written based on functions from our custom camera and motor libraries, this was not present in our final result due to the damaged camera.

SRS 06 – The system shall follow timing requirements by referencing properly scaled timers/counters.

- **Deemed unnecessary**
- When first writing this SRS during project proposal, We were not very clear on these things. However, as we complete the project, we realize that this SRS made no sense as this will automatically be completed as long as any pwm/counter is implemented correctly. There is no need to take special effort to achieve this.

#### 3.2 Hardware Requirements Specification (HRS) Results

HRS 01 – Project shall be based on ATmega328P microcontroller

- **Achieved**
- This is automatically completed as this is the definition of the final project of this class.

HRS 02 – A thermal imaging type sensor shall be used for flame detection.  The sensor shall detect min temperature range 0 ~ 300˚C (higher temperatures definitely signifies abnormal heating, potentially flame), a frame rate over 15FPS, and a resolution above 16x16.

- **Not achieved.**
- Due to the thermal camera malfunctioning, this was not able to be achieved. However, even with a fully functioning camera, certain aspects would not be possible due to unforseen limitations in ATmega328PB communication and calculation capabilities. Specifically, I2C limited to FM mode rather than FM+ (1MHz) supported by the sensor and relatively slow F_CPU and lack of an FPU leading to unreasonably long temperature recovery calculations for the 768 pixels (and necessary calibration calculations).
- With our selection of the MLX 90640, the temperature range is 0~300˚C, with a resolution of 24x32. This achieves our HRH requirements. However, while the camera itself has a refresh rate of up to 64 Hz, the above ATmega328PB limitations make the HRS frame rate impossible. We considered implementing an I2C driver on an Itsy Bitsy M4 (ARM Cortex M4 at 120MHz with an FPU), however, damage to the camera's thermopile array ultimately prevented this from being viable.

HRS 03 – Servo Motors shall be used for turret positioning. They shall be able to handle the load of the barrel + camera.

- **Achieved** (besides the fact that our camera is currently broken). 
- When originally writing this HRS, cheap 9g servos were the main consideration as they were abundant. However, we later acquired high torque servos dedicated for CR car steering. This provided way more torque than we would need, allowing it to easily handle any load applied to the barrel from the wires/tubing/weight etc.

HRS 04 – pump motor shall be used for shooting water. They shall be able to at least fire jets of water within the specified 0.5m operational radius.

- **Achieved.**
- Pump independently tested and shoots well beyond HRS specs (see final video). With nozzle attachment, it could easily hit approximately 10 meters with arcing, and within 3 meters virtually 0 projectile motion effects. This was mainly due to finding a powerful pump, and combining it with a small nozzle.

HRS 05 – A Buzzer shall be used for sounding an alarm. It shall be internally driven.

- **Achieved.**
- 4 internally driven buzzers are triggered to sound at intervals similar to a real fire alarm. This means that they are easily driven by simply applying a voltage across them.

HRS 06 (new) – Turret shall be able to continuously rotate, while preventing any leakage/bad electrical contact.

- **Achieved.**
- This is an added HRS since the issue was not considered at project proposal. However, this is really important for any turret where continuous azimuth rotation is desired. A slip ring and rotary union was implemented to prevent wires and tubes from twisting and allow continuous rotation.

HRS 07 (new) – The turret shall not leak water through the nozzle when in the off state.

- **Achieved.**
- This is an added HRS also due to an issue discovered after project proposal. Since the water tank is in the ceiling but the nozzle was beneath it, this would cause a syphoning effect after the initial firing, draining all of the water while the pump is off. This was originally solved using a valve. However this create a lot of complexity both mechanically and requiring another pwm control. Later, it was discovered that the pump was reversible. Hence the current improved solution is to use an H bridge to reverse the flow after each firing, pumping the water back past the point of syphoning.

### 4. Conclusion

This project provoked a significant amount of learning, as running into issues, troubleshooting, and finding solutions not only cemented topics from this course, but also concepts from prior courses and engineering experiences. We learned about mechanical prototyping, electrical prototyping, using datasheets, component selection, product design, bare metal coding, I2C, image processing, python scripting, servo motors, stepper motors, dealing with the limitations of an MCU, dealing with unnexpected mechanical and electrical issues and failures, using github, and more.

While we would have liked the final result of this project to include a functional thermal camera implementation, the learning process from all of the issues involved with the camera spurred significant improvement in understanding and implementing efficient code for embedded contexts. Similar learning took place through all of the issues we faced through our iterative design and testing processes.

We are certainly proud of coming from essentially zero experience in embedded systems at the beginning of the project to having a well made prototye (with successful non-camera functionality) and a much greater understanding of embedded systems. Despite falling a bit short on our naive intentions for the final outcome of this project, we both feel quite confident in our ability to complete future projects from start to finish, beginning with selecting compatible components and MCUs.

For this project, we are plannning on purchasing a new thermal camera so we can update the turrent with the intended autonomous functionality. Additionally, there is a chance we will take this project idea and update it for our ESE 5160 project next semester.
