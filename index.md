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

# Second Milestone

<iframe width="560" height="315" src="https://www.youtube.com/embed/TdamHaCC3TY?si=wYPMolTd6gKOe3nP" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Milestone 2 Descripton
- Milestone 2 was a big step - all the electronic components are now on the knee sleeve using tape
- In the future, these components will all be sewn down in a much more aesthetically pleasing fashion
- This milestone featured a lot of coding challenges, especially in programming the Adafruit sensors (accelerometer, gyroscope) in order to determine whether the user's knee was caving in during a squat with bad form
- The flex sensor being mounted to the back of the knee was a design choice that I made after trying it on various different locations - on the back allowed for the most precision and accuracy when determening knee bend

## Struggles/Hardships
- It took lots of trial and error determening different axis values, and even the correct point on the knee sleeve to mount the sensors
- I didn't understand at first how I was going to connect everything to the knee sleeve and have it working initially - now with trial and error and persistance, I was able to figure out a way.

## Milestone 3 Ideas
- Milestone three will involve sewing all the components to the knee sleeve, integrating a session tracker (reps, sets, etc) into the code, streaming session data over bluetooth/wifi to a handheld device, and ensuring the wires are all secure and that none will come off easily.

# Code
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


- For my first milestone, I built and wired the Adafruit arduino, gyroscope, and temperature sensors through the ESP-32 development board, with a breadboard
- I wired in a flex sensor and buzzer as well using jumper wires, Ohm resistors, and two breadboards that I had to combine together
- The flex sensor and Adafruit return values generated by the sensors within the components, that are analyzed by Arduino IDE and then feedback is provided in real time. 
- Through Arduino IDE, I coded the components such that when the Adafruit's accelerometer and gyroscope detect a tilt that is greater than a certain angle, the buzzer will sound. This simulates the knee caving in while performing a squat. Additionally, if the flex sensor detects a flex that is greater than the allowed bend, the buzzer will sound with a different sound pattern than the knee caving in alarm.
- A few challenges I faced was using the breadboard for the first time hands-on: the ESP-32 used was slightly too wide for one breadboard, so it took a few iterations to wire the ESP-32 along with the rest of the components in a way that was aesthetically clean and also the easiest to work with.
- The next milestone and the rest of the plan to complete this project includes integrating this hardware with a knee sleeve, and ensuring all the components do their job when the user is performing a squat. 

# Code
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

- For my starter project, I built the Weevil eye. It works by having a photo sensor determine whether there is light nearby, and when it detects darkness, the transistor captures this signal and provides current to the LEDs through the resistors in order for the LEDs to light up.
- I soldered the parts together (The LEDs, the transistor, the photo sensor, Ohm resistors, and battery holder.
- A few challenges that I faced when building my starter project were having to de-solder parts that I put in with incorrect polarity, I ended up fixing it. 
- It was a good introductory to working with electrical parts and it built the foundation I needed to work on my main project. 


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


                
<!--
# Other Resources/Examples
One of the best parts about Github is that you can view how other people set up their own work. Here are some past BSE portfolios that are awesome examples. You can view how they set up their portfolio, and you can view their index.md files to understand how they implemented different portfolio components.
- [Example 1](https://trashytuber.github.io/YimingJiaBlueStamp/)
- [Example 2](https://sviatil0.github.io/Sviatoslav_BSE/)
- [Example 3](https://arneshkumar.github.io/arneshbluestamp/)

Links for helpful stuff:


To watch the BSE tutorial on how to create a portfolio, click here. -->
