# Bench Robotics: DIY 6-DOF 3D Printed Robotic Arm 🤖

Welcome to the official Bench Robotics GitHub repository for our 6-Axis DIY Robotic Arm! This project is inspired by the fantastic open-source design from *How To Mechatronics* and optimized for accessible 3D printing and easy assembly.

This guide will walk you through the entire process: mechanical assembly, wiring, uploading the code, and controlling your new robot using a smartphone app.

---

## ⚠️ CRITICAL PRE-ASSEMBLY STEP: Centering Your Servos
**DO NOT skip this step!** If you assemble the arm without centering the servos, the arm will try to move past its physical limits when turned on, which will strip the plastic gears or break your 3D printed parts.

1. Connect your Arduino to your computer.
2. Wire one servo at a time to the Arduino (5V, GND, and Signal to Pin 9).
3. Open the Arduino IDE, go to **File > Examples > Servo > Sweep**.
4. Modify the code slightly to just write `90` degrees to the servo instead of sweeping. 
5. Upload the code. The servo will snap to its exact middle point (90 degrees). 
6. Do this for **all 6 servos** before attaching any plastic horns or gears to them!

---

## 🛠️ Section 1: Mechanical Assembly Instructions

Take your time with assembly. Do not overtighten screws, as you risk cracking the 3D-printed plastic. 

### Step 1: The Base (Waist)
1. Take the large cylindrical base part. 
2. Insert one of the large **MG996R servos** into the designated rectangular slot. Secure it with four M3 screws.
3. Attach the circular rotating platform to the servo shaft using the plastic servo horn included with your servo. Secure it with the small center screw.

### Step 2: The Shoulder
1. Place the lower shoulder bracket onto the rotating base. Secure it with screws.
2. Insert the second **MG996R servo** into the shoulder bracket. 
3. Attach the main lower arm link to this servo. This joint lifts the entire arm, so ensure the screw securing the arm link to the servo horn is tight.

### Step 3: The Elbow
1. At the top of the lower arm link, insert the third **MG996R servo**.
2. Attach the upper arm link. Make sure the joints move smoothly without grinding against the plastic. Use sandpaper if any printed parts fit too tightly.

### Step 4: The Wrist (Pitch and Roll)
1. We are now switching to the smaller **SG90 Micro Servos**.
2. Insert an SG90 servo into the end of the upper arm link. This is the **Wrist Pitch** (up and down movement).
3. Attach the wrist bracket to this servo.
4. Inside the wrist bracket, mount another SG90 servo facing forward. This is the **Wrist Roll** (twisting movement).

### Step 5: The Gripper
1. Attach the main gripper base to the Wrist Roll servo.
2. Mount the final SG90 servo into the gripper base.
3. Attach the central driving gear to the SG90 servo shaft.
4. Mount the left and right planetary gears and the gripper fingers. Ensure the teeth mesh perfectly. When the servo is at 90 degrees (centered), the gripper claws should be exactly halfway open.

---

## ⚡ Section 2: Wiring Instructions



**IMPORTANT WARNING:** Do NOT power the 6 servos directly from the Arduino's 5V pin. The Arduino cannot provide enough current. You **must** use an external 5V power supply (minimum 3 Amps). 

### Power Wiring
1. Connect the **Positive (+)** wire of your external 5V power supply to the positive rail of a breadboard.
2. Connect the **Negative (-/GND)** wire of your external power supply to the ground rail of the breadboard.
3. **CRITICAL:** Connect a jumper wire from the ground rail of the breadboard to the **GND pin on the Arduino**. If the grounds are not shared, the robotic arm will jitter uncontrollably.

### Servo Connections
All servos have 3 wires: Power (Red), Ground (Brown/Black), and Signal (Yellow/Orange).
* Connect ALL Red servo wires to the breadboard's Positive 5V rail.
* Connect ALL Brown/Black servo wires to the breadboard's Ground rail.
* Connect the Signal wires to the Arduino digital pins as follows:
    * **Servo 1 (Waist):** Pin 8
    * **Servo 2 (Shoulder):** Pin 9
    * **Servo 3 (Elbow):** Pin 10
    * **Servo 4 (Wrist Pitch):** Pin 11
    * **Servo 5 (Wrist Roll):** Pin 12
    * **Servo 6 (Gripper):** Pin 13

### HC-05 Bluetooth Module Connections
* **VCC:** Connect to the 5V pin on the Arduino (the Bluetooth module uses very little power, so this is safe).
* **GND:** Connect to Arduino GND.
* **TXD:** Connect to Arduino **RX (Pin 0)**.
* **RXD:** Connect to Arduino **TX (Pin 1)**.

*(Note: You **must** unplug the TX and RX wires from Pin 0 and 1 while uploading the code from your computer, or the upload will fail. Plug them back in after the upload is complete).*

---

## 💻 Section 3: The Arduino Code

1. Download and install the [Arduino IDE](https://www.arduino.cc/en/software).
2. Connect your Arduino Uno to your computer via USB.
3. Copy the code below and paste it into a new Arduino sketch.
4. Select "Arduino Uno" in the Boards menu and select your COM port.
5. **Unplug the Bluetooth module's RX/TX pins** and hit "Upload".

```cpp
#include <Servo.h>

// Initialize Servo objects
Servo servo01;
Servo servo02;
Servo servo03;
Servo servo04;
Servo servo05;
Servo servo06;

String dataIn = ""; // String to hold incoming Bluetooth data

// Variables to store current and previous servo positions
int servo1Pos, servo2Pos, servo3Pos, servo4Pos, servo5Pos, servo6Pos;
int servo1PPos, servo2PPos, servo3PPos, servo4PPos, servo5PPos, servo6PPos;

// Arrays to store recorded positions for the "Save" and "Run" feature
int servo01SP[50], servo02SP[50], servo03SP[50], servo04SP[50], servo05SP[50], servo06SP[50];

int speedDelay = 20;
int index = 0;

void setup() {
  // Attach servos to digital pins
  servo01.attach(8);
  servo02.attach(9);
  servo03.attach(10);
  servo04.attach(11);
  servo05.attach(12);
  servo06.attach(13);

  // Set initial default positions (centered)
  servo1PPos = 90; servo01.write(servo1PPos);
  servo2PPos = 90; servo02.write(servo2PPos);
  servo3PPos = 90; servo03.write(servo3PPos);
  servo4PPos = 90; servo04.write(servo4PPos);
  servo5PPos = 90; servo05.write(servo5PPos);
  servo6PPos = 90; servo06.write(servo6PPos);

  // Begin serial communication for Bluetooth
  Serial.begin(38400); // Default HC-05 baud rate. Change to 9600 if it doesn't work.
}

void loop() {
  // Check for incoming Bluetooth serial data
  if (Serial.available() > 0) {
    dataIn = Serial.readStringUntil('\n'); // Read string until newline character

    // --- PARSE BLUETOOTH COMMANDS ---
    // The app sends strings like "s1120" (Servo 1 to 120 degrees)
    
    // Servo 1
    if (dataIn.startsWith("s1")) {
      String dataInS = dataIn.substring(2, dataIn.length());
      servo1Pos = dataInS.toInt();
      // Smooth movement loop
      if (servo1PPos > servo1Pos) {
        for (int j = servo1PPos; j >= servo1Pos; j--) {
          servo01.write(j);
          delay(speedDelay);
        }
      }
      if (servo1PPos < servo1Pos) {
        for (int j = servo1PPos; j <= servo1Pos; j++) {
          servo01.write(j);
          delay(speedDelay);
        }
      }
      servo1PPos = servo1Pos; 
    }
    
    // Servo 2
    if (dataIn.startsWith("s2")) {
      String dataInS = dataIn.substring(2, dataIn.length());
      servo2Pos = dataInS.toInt();
      if (servo2PPos > servo2Pos) {
        for (int j = servo2PPos; j >= servo2Pos; j--) {
          servo02.write(j);
          delay(speedDelay);
        }
      }
      if (servo2PPos < servo2Pos) {
        for (int j = servo2PPos; j <= servo2Pos; j++) {
          servo02.write(j);
          delay(speedDelay);
        }
      }
      servo2PPos = servo2Pos;
    }

    // Servo 3
    if (dataIn.startsWith("s3")) {
      String dataInS = dataIn.substring(2, dataIn.length());
      servo3Pos = dataInS.toInt();
      if (servo3PPos > servo3Pos) {
        for (int j = servo3PPos; j >= servo3Pos; j--) {
          servo03.write(j);
          delay(speedDelay);
        }
      }
      if (servo3PPos < servo3Pos) {
        for (int j = servo3PPos; j <= servo3Pos; j++) {
          servo03.write(j);
          delay(speedDelay);
        }
      }
      servo3PPos = servo3Pos;
    }

    // Servo 4
    if (dataIn.startsWith("s4")) {
      String dataInS = dataIn.substring(2, dataIn.length());
      servo4Pos = dataInS.toInt();
      if (servo4PPos > servo4Pos) {
        for (int j = servo4PPos; j >= servo4Pos; j--) {
          servo04.write(j);
          delay(speedDelay);
        }
      }
      if (servo4PPos < servo4Pos) {
        for (int j = servo4PPos; j <= servo4Pos; j++) {
          servo04.write(j);
          delay(speedDelay);
        }
      }
      servo4PPos = servo4Pos;
    }

    // Servo 5
    if (dataIn.startsWith("s5")) {
      String dataInS = dataIn.substring(2, dataIn.length());
      servo5Pos = dataInS.toInt();
      if (servo5PPos > servo5Pos) {
        for (int j = servo5PPos; j >= servo5Pos; j--) {
          servo05.write(j);
          delay(speedDelay);
        }
      }
      if (servo5PPos < servo5Pos) {
        for (int j = servo5PPos; j <= servo5Pos; j++) {
          servo05.write(j);
          delay(speedDelay);
        }
      }
      servo5PPos = servo5Pos;
    }

    // Servo 6
    if (dataIn.startsWith("s6")) {
      String dataInS = dataIn.substring(2, dataIn.length());
      servo6Pos = dataInS.toInt();
      if (servo6PPos > servo6Pos) {
        for (int j = servo6PPos; j >= servo6Pos; j--) {
          servo06.write(j);
          delay(speedDelay);
        }
      }
      if (servo6PPos < servo6Pos) {
        for (int j = servo6PPos; j <= servo6Pos; j++) {
          servo06.write(j);
          delay(speedDelay);
        }
      }
      servo6PPos = servo6Pos; 
    }

    // Speed Slider Update
    if (dataIn.startsWith("ss")) {
      String dataInS = dataIn.substring(2, dataIn.length());
      speedDelay = dataInS.toInt(); 
    }

    // --- MACRO CONTROLS ---
    
    // Save Position
    if (dataIn == "SAVE") {
      servo01SP[index] = servo1PPos;
      servo02SP[index] = servo2PPos;
      servo03SP[index] = servo3PPos;
      servo04SP[index] = servo4PPos;
      servo05SP[index] = servo5PPos;
      servo06SP[index] = servo6PPos;
      index++; // Move to next array slot
    }

    // Run Saved Sequence
    if (dataIn == "RUN") {
      runSavedPositions();
    }

    // Reset Saved Sequence
    if (dataIn == "RESET") {
      memset(servo01SP, 0, sizeof(servo01SP));
      memset(servo02SP, 0, sizeof(servo02SP));
      memset(servo03SP, 0, sizeof(servo03SP));
      memset(servo04SP, 0, sizeof(servo04SP));
      memset(servo05SP, 0, sizeof(servo05SP));
      memset(servo06SP, 0, sizeof(servo06SP));
      index = 0; // Reset index
    }
  }
}

// Function to play back recorded movements
void runSavedPositions() {
  while (dataIn != "RESET") {
    for (int i = 0; i <= index - 2; i++) {
      if (Serial.available() > 0) {
        dataIn = Serial.readStringUntil('\n');
        if (dataIn == "RESET") {
          break; // Stop running if reset is pressed
        }
      }
      // Move Servo 1
      if (servo01SP[i] == servo01SP[i + 1]) {}
      if (servo01SP[i] > servo01SP[i + 1]) {
        for (int j = servo01SP[i]; j >= servo01SP[i + 1]; j--) {
          servo01.write(j); delay(speedDelay);
        }
      }
      if (servo01SP[i] < servo01SP[i + 1]) {
        for (int j = servo01SP[i]; j <= servo01SP[i + 1]; j++) {
          servo01.write(j); delay(speedDelay);
        }
      }
      // Add similar blocks for servos 2 through 6 here in your final code to playback all joints
    }
  }
}
