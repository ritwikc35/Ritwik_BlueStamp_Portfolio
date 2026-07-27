# Knee Rehabilitation Device
This project is a wearable knee rehabilitation device that helps people perform squats with safe, correct form during recovery. Built around an ESP32 microcontroller, it uses a flex sensor to track how deep the knee bends and an accelerometer + gyroscope to detect when the knee caves inward, sounding a buzzer in real time so the user can correct themselves instantly. The biggest challenge was translating messy, noisy sensor data into reliable feedback: it took lots of testing and calibration to figure out which sensor readings actually corresponded to a good squat versus a bad one, but landing on a system that counts reps, tracks range of motion, and catches bad form the moment it happens made all the debugging worth it.

<!--You should comment out all portions of your portfolio that you have not completed yet, as well as any instructions:
```HTML 
<!--- This is an HTML comment in Markdown -->
<!--- Anything between these symbols will not render on the published site -->


| **Engineer** | **School** | **Area of Interest** | **Grade** |
|:--:|:--:|:--:|:--:|
| Ritwik C | Evergreen Valley High School | Computer Science | Incoming Junior

<!--**Replace the BlueStamp logo below with an image of yourself and your completed project. Follow the guide [here](https://tomcam.github.io/least-github-pages/adding-images-github-pages-site.html) if you need help.** -->

![Headshot](/docs/assets/ritwikc-1.png)
  
<!--# Final Milestone-->

<!--**Don't forget to replace the text below with the embedding for your milestone video. Go to Youtube, click Share -> Embed, and copy and paste the code to replace what's below.**

<iframe width="560" height="315" src="https://www.youtube.com/embed/F7M7imOVGug" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

For your final milestone, explain the outcome of your project. Key details to include are:
- What you've accomplished since your previous milestone
- What your biggest challenges and triumphs were at BSE
- A summary of key topics you learned about
- What you hope to learn in the future after everything you've learned at BSE
-->

```c++
#include <Adafruit_LSM6DS3TRC.h>
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>

Adafruit_LSM6DS3TRC lsm6ds;

// ---------- BLE (Nordic UART Service) ----------
#define SERVICE_UUID  "6E400001-B5A3-F393-E0A9-E50E24DCCA9E"
#define CHAR_UUID_TX  "6E400003-B5A3-F393-E0A9-E50E24DCCA9E"
#define CHAR_UUID_RX  "6E400002-B5A3-F393-E0A9-E50E24DCCA9E"

BLECharacteristic *pTxCharacteristic;
bool deviceConnected = false;

class MyServerCallbacks : public BLEServerCallbacks {
  void onConnect(BLEServer *pServer)    { deviceConnected = true; }
  void onDisconnect(BLEServer *pServer) {
    deviceConnected = false;
    pServer->startAdvertising();   // allow reconnect
  }
};

// Send a line to both Serial and BLE
void sendBoth(String msg) {
  Serial.print(msg);
  if (deviceConnected) {
    pTxCharacteristic->setValue(msg.c_str());
    pTxCharacteristic->notify();
  }
}

// ---------- Pins ----------
#define FLEX_PIN   32   // analog input from flex sensor divider
#define BUZZER_PIN 13   // active buzzer

// ---- Squat detection (X axis) ----
const float SQUAT_X_THRESHOLD = 9.0;   // X below this = squat, above = standing

// ---- Cave detection (Z axis) ----
const float CAVE_Z_THRESHOLD = -2.0;   // while squatting, Z below this = caving

// ---- Flex over-bend (raw values) ----
const int FLEX_UPPER_LIMIT = 1900;   // raw reading = bent too far
const int FLEX_LOWER_LIMIT = 320;    // raw reading = bent too far (other direction)

// ---- Consecutive-reading confirmation ----
const int CAVE_CONFIRM_COUNT = 2;    // caving reads in a row needed to buzz

const int SAMPLES = 5;   // light averaging

int caveStreak = 0;      // counts consecutive caving readings

// ---- BLE status streaming rate limit ----
unsigned long lastBLEStatus = 0;
const unsigned long BLE_STATUS_INTERVAL = 1000;  // send status once per second

void setup() {
  Serial.begin(115200);
  while (!Serial) delay(10);
  pinMode(BUZZER_PIN, OUTPUT);
  Wire.begin(21, 22);
  if (!lsm6ds.begin_I2C()) { Serial.println("Sensor not found!"); while (1) delay(10); }

  // --- BLE setup ---
  BLEDevice::init("KneeRehab");
  BLEServer *pServer = BLEDevice::createServer();
  pServer->setCallbacks(new MyServerCallbacks());
  BLEService *pService = pServer->createService(SERVICE_UUID);
  pTxCharacteristic = pService->createCharacteristic(
                        CHAR_UUID_TX,
                        BLECharacteristic::PROPERTY_NOTIFY
                      );
  pService->createCharacteristic(
    CHAR_UUID_RX,
    BLECharacteristic::PROPERTY_WRITE
  );
  pTxCharacteristic->addDescriptor(new BLE2902());
  pService->start();
  pServer->getAdvertising()->addServiceUUID(SERVICE_UUID);
  pServer->getAdvertising()->start();
  Serial.println("BLE ready - connect to 'KneeRehab'");

  Serial.println("Ready - start squatting!");
}

// Two short beeps = knee caving inward
void beepInward() {
  for (int i = 0; i < 2; i++) {
    digitalWrite(BUZZER_PIN, HIGH); delay(120);
    digitalWrite(BUZZER_PIN, LOW);  delay(120);
  }
}

// Long continuous buzz = bent too far
void buzzOverBend() {
  digitalWrite(BUZZER_PIN, HIGH);
  delay(600);
  digitalWrite(BUZZER_PIN, LOW);
}

void loop() {
  // --- Flex sensor: raw reading ---
  int flexRaw = analogRead(FLEX_PIN);

  // --- Accelerometer: averaged X and Z ---
  float xAvg = 0, zAvg = 0;
  for (int i = 0; i < SAMPLES; i++) {
    sensors_event_t a, g, t;
    lsm6ds.getEvent(&a, &g, &t);
    xAvg += a.acceleration.x;
    zAvg += a.acceleration.z;
    delay(5);
  }
  xAvg /= SAMPLES;
  zAvg /= SAMPLES;

  bool inSquat = (xAvg < SQUAT_X_THRESHOLD);
  bool caving  = inSquat && (zAvg < CAVE_Z_THRESHOLD);

  // --- Track consecutive caving readings ---
  if (caving) {
    caveStreak++;
  } else {
    caveStreak = 0;
  }

  // --- Build the status line ---
  String status = "Flex: " + String(flexRaw)
                + " | X: " + String(xAvg, 2)
                + " | Z: " + String(zAvg, 2);
  if (!inSquat)      status += "  -> STANDING";
  else if (caving)   status += "  -> SQUATTING WHILE CAVING!";
  else               status += "  -> SQUAT (good)";

  // Always print to Serial (when on laptop)
  Serial.println(status);

  // Send status to BLE only once per second (so it doesn't flood)
  if (millis() - lastBLEStatus > BLE_STATUS_INTERVAL) {
    if (deviceConnected) {
      pTxCharacteristic->setValue((status + "\n").c_str());
      pTxCharacteristic->notify();
    }
    lastBLEStatus = millis();
  }

  // --- Fault 1: knee caving inward (needs consecutive confirmation) ---
  if (caveStreak >= CAVE_CONFIRM_COUNT) {
    sendBoth(">>> KNEE CAVING INWARD!\n");
    beepInward();
    caveStreak = 0;
  }

  // --- Fault 2: flex sensor bent too far ---
  if ( (flexRaw > FLEX_UPPER_LIMIT) || (flexRaw < FLEX_LOWER_LIMIT) ) {
    sendBoth(">>> OVER-BEND! Come back up.\n");
    buzzOverBend();
  }

  delay(50);
}
```

# Second Milestone

<iframe width="560" height="315" src="https://www.youtube.com/embed/TdamHaCC3TY?si=wYPMolTd6gKOe3nP" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Milestone 2 Descripton
- Milestone 2 was a big step - all the electronic components are now on the knee sleeve using tape
- In the future, these components will all be sewn down in a much more aesthetically pleasing fashion
- This milestone featured a lot of coding challenges, especially in programming the Adafruit sensors (accelerometer, gyroscope) in order to determine whether the user's knee was caving in during a squat with bad form
- The flex sensor being mounted to the back of the knee was a design choice that I made after trying it on various different locations - on the back allowed for the most precision and accuracy when determening knee bend

## Struggles/Challenges
- It took lots of trial and error determening different axis values, and even the correct point on the knee sleeve to mount the sensors
- I didn't understand at first how I was going to connect everything to the knee sleeve and have it working initially - now with trial and error and persistance, I was able to figure out a way.

## Milestone 3 Ideas
- Milestone three will involve sewing all the components to the knee sleeve, integrating a session tracker (reps, sets, etc) into the code, streaming session data over bluetooth/wifi to a handheld device, and ensuring the wires are all secure and that none will come off easily.

## Second Milestone Code
```c++
#include <Adafruit_LSM6DS3TRC.h>
Adafruit_LSM6DS3TRC lsm6ds;

// ---------- Pins ----------
#define FLEX_PIN   32   // analog input from flex sensor divider
#define BUZZER_PIN 13   // active buzzer

// ---- Squat detection (X axis) ----
const float SQUAT_X_THRESHOLD = 9.0;   // X below this = squat, above = standing

// ---- Cave detection (Z axis) ----
const float CAVE_Z_THRESHOLD = -2.0;   // while squatting, Z below this = caving

// ---- Flex over-bend (raw values) ----
const int FLEX_UPPER_LIMIT = 2070;   // raw reading = bent too far
const int FLEX_LOWER_LIMIT = 320;    // raw reading = bent too far (other direction)

// ---- Consecutive-reading confirmation ----
const int CAVE_CONFIRM_COUNT = 2;    // caving reads in a row needed to buzz

const int SAMPLES = 5;   // light averaging

int caveStreak = 0;      // counts consecutive caving readings

void setup() {
  Serial.begin(115200);
  while (!Serial) delay(10);
  pinMode(BUZZER_PIN, OUTPUT);
  Wire.begin(21, 22);
  if (!lsm6ds.begin_I2C()) { Serial.println("Sensor not found!"); while (1) delay(10); }
  Serial.println("Ready - start squatting!");
}

// Two short beeps = knee caving inward
void beepInward() {
  for (int i = 0; i < 2; i++) {
    digitalWrite(BUZZER_PIN, HIGH); delay(120);
    digitalWrite(BUZZER_PIN, LOW);  delay(120);
  }
}

// Long continuous buzz = bent too far
void buzzOverBend() {
  digitalWrite(BUZZER_PIN, HIGH);
  delay(600);
  digitalWrite(BUZZER_PIN, LOW);
}

void loop() {
  // --- Flex sensor: raw reading ---
  int flexRaw = analogRead(FLEX_PIN);

  // --- Accelerometer: averaged X and Z ---
  float xAvg = 0, zAvg = 0;
  for (int i = 0; i < SAMPLES; i++) {
    sensors_event_t a, g, t;
    lsm6ds.getEvent(&a, &g, &t);
    xAvg += a.acceleration.x;
    zAvg += a.acceleration.z;
    delay(5);
  }
  xAvg /= SAMPLES;
  zAvg /= SAMPLES;

  bool inSquat = (xAvg < SQUAT_X_THRESHOLD);
  bool caving  = inSquat && (zAvg < CAVE_Z_THRESHOLD);

  // --- Track consecutive caving readings ---
  if (caving) {
    caveStreak++;
  } else {
    caveStreak = 0;
  }

  // --- Print everything ---
  Serial.print("Flex: ");   Serial.print(flexRaw);
  Serial.print(" | X: ");   Serial.print(xAvg, 2);
  Serial.print(" | Z: ");   Serial.print(zAvg, 2);
  if (!inSquat)      Serial.print("  -> STANDING");
  else if (caving)   Serial.print("  -> SQUATTING WHILE CAVING!");
  else               Serial.print("  -> SQUAT (good)");
  Serial.println();

  // --- Fault 1: knee caving inward (needs consecutive confirmation) ---
  if (caveStreak >= CAVE_CONFIRM_COUNT) {
    Serial.println(">>> KNEE CAVING INWARD!");
    beepInward();
    caveStreak = 0;
  }

  // --- Fault 2: flex sensor bent too far ---
  if ( (flexRaw > FLEX_UPPER_LIMIT) || (flexRaw < FLEX_LOWER_LIMIT) ) {
    Serial.println(">>> OVER-BEND! Come back up.");
    buzzOverBend();
  }

  delay(50);
}
```



# First Milestone

<!--**Don't forget to replace the text below with the embedding for your milestone video. Go to Youtube, click Share -> Embed, and copy and paste the code to replace what's below.**-->

<iframe width="560" height="315" src="https://www.youtube.com/embed/yLWbas0NSrE?si=K7r7uFJK54m2mVIC" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>


## Milestone 1 Descripton
- For my first milestone, I built and wired the Adafruit arduino, gyroscope, and temperature sensors through the ESP-32 development board, with a breadboard
- I wired in a flex sensor and buzzer as well using jumper wires, Ohm resistors, and two breadboards that I had to combine together
- The flex sensor and Adafruit return values generated by the sensors within the components, that are analyzed by Arduino IDE and then feedback is provided in real time. 
- Through Arduino IDE, I coded the components such that when the Adafruit's accelerometer and gyroscope detect a tilt that is greater than a certain angle, the buzzer will sound. This simulates the knee caving in while performing a squat. Additionally, if the flex sensor detects a flex that is greater than the allowed bend, the buzzer will sound with a different sound pattern than the knee caving in alarm.

## Struggles/Challenges
- A few challenges I faced was using the breadboard for the first time hands-on: the ESP-32 used was slightly too wide for one breadboard, so it took a few iterations to wire the ESP-32 along with the rest of the components in a way that was aesthetically clean and also the easiest to work with.

## Milestone 2 Ideas
- The next milestone and the rest of the plan to complete this project includes integrating this hardware with a knee sleeve, and ensuring all the components do their job when the user is performing a squat. 

## First Milestone Code
```c++
#include <Adafruit_LSM6DS3TRC.h>
#include <Adafruit_LIS3MDL.h>

Adafruit_LSM6DS3TRC lsm6ds;
Adafruit_LIS3MDL lis3mdl;

// ---------- Pins ----------
#define FLEX_PIN   32   // analog input from flex sensor divider
#define BUZZER_PIN 13   // active buzzer

// ---------- Thresholds (RAW values, no angle math) ----------
const int FLEX_UPPER_LIMIT = 2070; // raw reading that means "bent too far"
const int FLEX_LOWER_LIMIT = 700;  // raw reading that means "bent too far"
const float INWARD_THRESHOLD = 5.0;   // accelerometer side-tilt 

void setup() {
  Serial.begin(115200);
  while (!Serial) delay(10);

  pinMode(BUZZER_PIN, OUTPUT);
  Wire.begin(21, 22);

  if (!lsm6ds.begin_I2C()) {
    Serial.println("LSM6DS3TR-C not found - check wiring!");
    while (1) delay(10);
  }
  if (!lis3mdl.begin_I2C()) {
    Serial.println("LIS3MDL not found - check wiring!");
    while (1) delay(10);
  }
  Serial.println("Sensors ready.");
}

// Long continuous buzz = bent too far
void buzzOverBend() {
  digitalWrite(BUZZER_PIN, HIGH);
  delay(600);
  digitalWrite(BUZZER_PIN, LOW);
}

// Two short beeps = knee caving inward
void beepInward() {
  for (int i = 0; i < 2; i++) {
    digitalWrite(BUZZER_PIN, HIGH);
    delay(120);
    digitalWrite(BUZZER_PIN, LOW);
    delay(120);
  }
}

void loop() {
  // --- Flex sensor: raw reading, no conversion ---
  int flexRaw = analogRead(FLEX_PIN);

  // --- Accelerometer: side-to-side tilt ---
  sensors_event_t accel, gyro, temp;
  lsm6ds.getEvent(&accel, &gyro, &temp);
  float sideTilt = accel.acceleration.x;

  // --- Print both so you can see what's happening ---
  Serial.print("Flex raw: ");        Serial.print(flexRaw);
  Serial.print("  |  SideTilt: ");   Serial.print(sideTilt);
  Serial.println(" m/s^2");

  // --- Trigger 1: flex sensor bent past the raw limit ---
  if ( (flexRaw > FLEX_UPPER_LIMIT) || (flexRaw < FLEX_LOWER_LIMIT) ) {
    Serial.println(">>> OVER-BEND! Come back up.");
    buzzOverBend();
  }

  // --- Trigger 2: knee caving inward (already working) ---
  if (sideTilt > INWARD_THRESHOLD) {
    Serial.println(">>> KNEE CAVING INWARD! Fix form.");
    beepInward();
  }

  delay(100);
}
```

# Starter Project

<!--**Don't forget to replace the text below with the embedding for your milestone video. Go to Youtube, click Share -> Embed, and copy and paste the code to replace what's below.** -->

<iframe width="560" height="315" src="https://www.youtube.com/embed/6VLap-Eq1X8?si=N1XxQ3PWybVsjphl" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Starter Project Description
- For my starter project, I built the Weevil eye. It works by having a photo sensor determine whether there is light nearby, and when it detects darkness, the transistor captures this signal and provides current to the LEDs through the resistors in order for the LEDs to light up.
- I soldered the parts together (The LEDs, the transistor, the photo sensor, Ohm resistors, and battery holder.
- It was a good introductory to working with electrical parts and it built the foundation I needed to work on my main project. 

## Struggles/Challenges
- A few challenges that I faced when building my starter project were having to de-solder parts that I put in with incorrect polarity, I ended up fixing it. 

# Schematic

![Schematic](/docs/assets/schematic2.png)

## Labels:

1. ESP-32 Microcontroller. This microcontroller is the brains of the entire circuit, and it bridges the gap of the communication between the laptop delivering the code and the other sensors/parts in the circuit.
2. Adafruit Adafruit LSM6DS3TR-C + LIS3MDL (IMU - Gyroscope + Accelerometer). This sensor tracks translational acceleration and position in three axes, and the gyroscope measures angular velocity in three axes. This was used to determine whether the knee is caving inwards during a squat or not, using the IMU.
3. Flex Sensor. This sensor has particles lined up within the length of the body, and as the flex sensor bends, the particles' distance between each other increases. When this distance increases, the resistance pulling these particles back together also increases. The sensor tracks this resistance, and determines the value for the flex sensor dependent on how bent it is.
4. Active Buzzer. Gets current from breadboard and ESP-32 to, when prompted, create a beeping sound. This sound cannot be altered pitch or frequency wise, however what can be altered is the buzzing pattern. I used this to differentiate the improper form caused by knee caving versus over-bend.
5. 10k Ω Resistor. This resistor is used to limit the current from the ESP-32 to the Flex Sensor to prevent a short circuit.
6. STEMMA QT / Qwiic JST SH 4-pin cable. This cable is used to connect into the Adafruit IMU. It has four cables - Red, Black, Blue, and Yellow. Red goes to power (3V3), Black goes to ground (GND), and Blue and Yellow go to the D21 and D22 pinouts in the microcontroller.

<!--# Schematics 
Here's where you'll put images of your schematics. [Tinkercad](https://www.tinkercad.com/blog/official-guide-to-tinkercad-circuits) and [Fritzing](https://fritzing.org/learning/) are both great resoruces to create professional schematic diagrams, though BSE recommends Tinkercad becuase it can be done easily and for free in the browser. -->



<!--# Schematics 
Here's where you'll put images of your schematics. [Tinkercad](https://www.tinkercad.com/blog/official-guide-to-tinkercad-circuits) and [Fritzing](https://fritzing.org/learning/) are both great resoruces to create professional schematic diagrams, though BSE recommends Tinkercad becuase it can be done easily and for free in the browser. -->


# Bill of Materials

| **Part** | **Note** | **Price** | **Link** |
|:--:|:--:|:--:|:--:|
| 2.1 x 3.2 in Breadboard | Wiring together the microcontroller and sensors | $2.99 | <a href="https://www.amazon.com/Electronix-Express-03MB801-Solderless-BreadBoard/dp/B005GYAIES/ref=sr_1_4?crid=3TFO3YV2C0RGV&dib=eyJ2IjoiMSJ9.iI9Qcy3WKpuTy3pEqkAEC2otBFTE8Y3NQycTtfLqMWVVx9K6r-ApLIMFcw8eWohpPsiB9m0Riny6KXAUiJYTfDJGsDdRwnrsI80-EmGhH4o1wvRVNJybvE9ZXxu_TaOeo8dV8WAt7wxJjLGFXeDsmwGmZ4lisxDEHD9TFefLDQBdPDIroN_4flJqSZwE6miNLwrQVowFEdTkduGxrWM4r9anfsLcxjzu0FCcXG83IyM.w_ZEGms_FH1bxrE79u62tv-T_GMp6ZGHn31hHfzrFcY&dib_tag=se&keywords=2.1%2Bx%2B3.2%2Bbreadboard&qid=1783007815&sprefix=2.1%2Bx%2B3.2%2Bbreabdoa%2Caps%2C208&sr=8-4&th=1)"> Link </a> |
| ESP32 CP2102 | Microcontroller allowing all the sensors to communicate (WiFi, Bluetooth) | $9.39 | <a href="https://www.amazon.com/AITRIP-ESP-WROOM-32-Development-Microcontroller-Integrated/dp/B07WCG1PLV/ref=sr_1_3crid=SGA8GEL9984K&dib=eyJ2IjoiMSJ9.Hje1sLvRZipVhAPaHZxgtWaUKJecWiUIqimtd2pu9Eg06FjxPw0SnsgKa43LldUnk3P1ueuFeYzqgmTVYBw2tvRKzCluHlAgtzRk3XaT5w9yw575t_lYWsTUIWImgcXriY0AL_T4OOTCKw9Bsb3Bvh84FgXLfFABXvIAWrP0KZiQWj1hzUcAvncIqVTYUDKGVRxMDkWRnXRH2V5uUZSopn4u0fUaGp-S6ihxvDPVY1setuf_zFUaKWPAB-YX4U_P8koFvwRxpKbvOU6uxdNQaVe1TsCBNa35GaFiNNud_K8.8XKCieKI1WHWUR9_26o2IEpG9XdFZQyOgIjY56kYZ10&dib_tag=se&keywords=ESP32%2BCP2102&qid=1783007868&s=electronics&sprefix=%2Celectronics%2C347&sr=1-3&th=1)"> Link </a> |
| Power bank| External power for the system | $9.99 | <a href="https://www.amazon.com/10000mAh-Portable-Essentials-Powerbank-Compatible/dp/B0FVXM2W9P/ref=sr_1_3?crid=2FCWS07YAJAUF&dib=eyJ2IjoiMSJ9.6dvoZTcKycQZVEOKy-hpnxht-0wg8e7pTrDeYctsklQ-FG1Naj60ajYYYmVmDqtlMIx1LygAQ7jXxY9J6L-QHPYVfVBsjkMUT0s2uF75osuOiFzD6zHIXOVhejD4u4HABMo3zcIpF4j4aikvv0dIyExjv3kaAYJvZnrcD1ddeBLT0314K4iHeRnSfK0lwbPLNBsiYq6LSMRPrUd7zbVz2pY7kcv6i2b6EDmf3emsejs.VRkJEUzLoIdnUo-WFlHAlGxEYjhsFD_zHDiMLoHdUms&dib_tag=se&keywords=Power%2Bbank%2Bn001%2Bblack&qid=1783009300&sprefix=power%2Bbank%2Bn001%2Bblack%2Caps%2C163&sr=8-3&th=1)"> Link </a> |
| STEMMA QT / Qwiic JST SH 4-pin cable (50mm) | Connect Adafruit sensor to ESP-32| $0.95 | <a href="https://www.adafruit.com/product/4399?gad_source=1&gad_campaignid=23969092792&gbraid=0AAAAADx9JvRY0IzrWNJJAcyDmg4KptFD0&gclid=CjwKCAjwmJjSBhB-EiwAkZgxi65p1s2wgSW6jNJyPJPZz9CCoU5EbvfUPlwi8P7i8Aj2y4aEZmldkBoCjRMQAvD_BwE"> Link </a> |
| 10K Ohm Resistor | Provide current resistance to flex sensor| $0.75 | <a href="https://www.adafruit.com/product/2784"> Link </a> |
| Adafruit LSM6DS3TR-C + LIS3MDL| Accelerometer + gyroscope to detect knee caving or bad form | $19.95 | <a href="https://www.adafruit.com/product/5543?gad_source=1&gad_campaignid=23969092792&gbraid=0AAAAADx9JvRY0IzrWNJJAcyDmg4KptFD0&gclid=CjwKCAjwmJjSBhB-EiwAkZgxiwsas2YkMRZVNUG9cxysx3TbwMesVdKfVBNfG9vVOOuQVA_PWTSCFRoC-gsQAvD_BwE"> Link </a> |
| Electromagnetic Active Buzzer 5V | Make a beep sound when bad form is detected or lowest squat position is reached | $0.95 | <a href="https://www.adafruit.com/product/1536?gad_source=1&gad_campaignid=23969092792&gbraid=0AAAAADx9JvRY0IzrWNJJAcyDmg4KptFD0&gclid=CjwKCAjwmJjSBhB-EiwAkZgxi3QGkfrQtLJE6vY_rWwkEijmqnVPmx4sBTx3JTJYeq84o7eFhmyqwBoCSdQQAvD_BwE"> Link </a> |
| Jumper Wires | Connect all the sensors, buzzers, and controllers via the breadboard | $1.95 | <a href="https://www.adafruit.com/product/1953?gad_source=1&gad_campaignid=23969092792&gbraid=0AAAAADx9JvRY0IzrWNJJAcyDmg4KptFD0&gclid=CjwKCAjwmJjSBhB-EiwAkZgxi8k_84mON50a65a2L-K4CZ_KWhdd5pSoWaosAUHNLOqTZ9boJJB2hBoCsf8QAvD_BwE"> Link </a> |
| Knee Compression Sleeve | Mount all electronic equipment to straddle the knee on this | $12.99 | <a href="https://www.amazon.com/Compression-Sleeve-Support-Running-Medium/dp/B0987XL3WV/ref=sr_1_16?crid=2W6XWLEBPIA9L&dib=eyJ2IjoiMSJ9.FMxH8-_ulIvKRL0eQJ1F8ebw-_g-rpsi-0NMsOOghfd235eBGyebm0frShQ9UtNIfo01hYDCwvoC6O9eS_K80CGnPQ1pnf-b9yccqZPu14YyCTi3-I4FtIdPf5_dcJ8VM5jtwBsEr7M9WDqxbTXoXEROWPMo8xpTzHWd2Ud9_F23nt7ZZccxKGG2FGPhe8nYKtHmKRUJA1yXbMNAAS3yBb2Pc4VtFNojLrf8fJ_SpbcziEQqkElBdLxoFh12t6Y4sWsHV6aTMzMZaXR7OYN4D4qXeVFScJBLEQGmqHGU11M.8tCS5a8HKsH7oFpNNo9eB98cJCQ53gxXFRe2g9pozuw&dib_tag=se&keywords=knee%2Bcompression%2Bsleeve&qid=1783009830&sprefix=knee%2Bcompression%2Bsleev%2Caps%2C166&sr=8-16&th=1"> Link </a> |
| Flex Sensor 4.5" | Detect a bend and send current to the buzzer when a specific bend is reached | $12.95 | <a href="https://www.adafruit.com/product/182?srsltid=AfmBOoqQkAp_6FrmD8PKZpHYrzxGymBcMme9gt24bTR6xO3JBTOPKlcQ"> Link </a> |


                

# Other Resources + Links for helpful stuff:
- [Resource 1](https://stackoverflow.com/questions/14189440/c-callback-using-class-member)
- [Resource 2](http://youtube.com/watch?v=xPlN_Tk3VLQ)
