# Knee Rehabilitation Device
Replace this text with a brief description (2-3 sentences) of your project. This description should draw the reader in and make them interested in what you've built. You can include what the biggest challenges, takeaways, and triumphs from completing the project were. As you complete your portfolio, remember your audience is less familiar than you are with all that your project entails!

<!--You should comment out all portions of your portfolio that you have not completed yet, as well as any instructions:
```HTML 
<!--- This is an HTML comment in Markdown -->
<!--- Anything between these symbols will not render on the published site -->


| **Engineer** | **School** | **Area of Interest** | **Grade** |
|:--:|:--:|:--:|:--:|
| Ritwik C | Evergreen Valley High School | Mechanical Engineering | Incoming Junior

<!--**Replace the BlueStamp logo below with an image of yourself and your completed project. Follow the guide [here](https://tomcam.github.io/least-github-pages/adding-images-github-pages-site.html) if you need help.**

![Headstone Image](logo.svg)
  
<!--# Final Milestone-->

<!--**Don't forget to replace the text below with the embedding for your milestone video. Go to Youtube, click Share -> Embed, and copy and paste the code to replace what's below.**

<iframe width="560" height="315" src="https://www.youtube.com/embed/F7M7imOVGug" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

For your final milestone, explain the outcome of your project. Key details to include are:
- What you've accomplished since your previous milestone
- What your biggest challenges and triumphs were at BSE
- A summary of key topics you learned about
- What you hope to learn in the future after everything you've learned at BSE

-->
<!--
# Second Milestone

**Don't forget to replace the text below with the embedding for your milestone video. Go to Youtube, click Share -> Embed, and copy and paste the code to replace what's below.**

<iframe width="560" height="315" src="https://www.youtube.com/embed/y3VAmNlER5Y" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

For your second milestone, explain what you've worked on since your previous milestone. You can highlight:
- Technical details of what you've accomplished and how they contribute to the final goal
- What has been surprising about the project so far
- Previous challenges you faced that you overcame
- What needs to be completed before your final milestone 

# First Milestone

**Don't forget to replace the text below with the embedding for your milestone video. Go to Youtube, click Share -> Embed, and copy and paste the code to replace what's below.**

<iframe width="560" height="315" src="https://www.youtube.com/embed/CaCazFBhYKs" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

For your first milestone, describe what your project is and how you plan to build it. You can include:
- An explanation about the different components of your project and how they will all integrate together
- Technical progress you've made so far
- Challenges you're facing and solving in your future milestones
- What your plan is to complete your project

<!--# Schematics 
Here's where you'll put images of your schematics. [Tinkercad](https://www.tinkercad.com/blog/official-guide-to-tinkercad-circuits) and [Fritzing](https://fritzing.org/learning/) are both great resoruces to create professional schematic diagrams, though BSE recommends Tinkercad becuase it can be done easily and for free in the browser. -->


# Starter Project

**Don't forget to replace the text below with the embedding for your milestone video. Go to Youtube, click Share -> Embed, and copy and paste the code to replace what's below.**

<iframe width="560" height="315" src="https://www.youtube.com/embed/6VLap-Eq1X8?si=N1XxQ3PWybVsjphl" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

- For my starter project, I built the Weevil eye. It works by having a photo sensor determine whether there is light nearby, and when it detects darkness, the transistor captures this signal and provides current to the LEDs through the resistors in order for the LEDs to light up.
- I soldered the parts together (The LEDs, the transistor, the photo sensor, Ohm resistors, and battery holder.
- A few challenges that I faced when building my starter project were having to de-solder parts that I put in with incorrect polarity, I ended up fixing it. 
- It was a good introductory to working with electrical parts and it built the foundation I needed to work on my main project. 

<!--# Schematics 
Here's where you'll put images of your schematics. [Tinkercad](https://www.tinkercad.com/blog/official-guide-to-tinkercad-circuits) and [Fritzing](https://fritzing.org/learning/) are both great resoruces to create professional schematic diagrams, though BSE recommends Tinkercad becuase it can be done easily and for free in the browser. -->

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
// Set FLEX_LIMIT to the raw reading you get at ~90 degrees of bend.
// Watch the Serial Monitor, bend to 90, note the number, put it here.
const int FLEX_UPPER_LIMIT = 2070;
const int FLEX_LOWER_LIMIT = 700;  // <-- raw reading that means "bent too far"
const float INWARD_THRESHOLD = 5.0;   // accelerometer side-tilt (this already works)

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

<!--# Bill of Materials
Here's where you'll list the parts in your project. To add more rows, just copy and paste the example rows below.
Don't forget to place the link of where to buy each component inside the quotation marks in the corresponding row after href =. Follow the guide [here]([url](https://www.markdownguide.org/extended-syntax/)) to learn how to customize this to your project needs. 

| **Part** | **Note** | **Price** | **Link** |
|:--:|:--:|:--:|:--:|
| Item Name | What the item is used for | $Price | <a href="https://www.amazon.com/Arduino-A000066-ARDUINO-UNO-R3/dp/B008GRTSV6/"> Link </a> |
| Item Name | What the item is used for | $Price | <a href="https://www.amazon.com/Arduino-A000066-ARDUINO-UNO-R3/dp/B008GRTSV6/"> Link </a> |
| Item Name | What the item is used for | $Price | <a href="https://www.amazon.com/Arduino-A000066-ARDUINO-UNO-R3/dp/B008GRTSV6/"> Link </a> |

# Other Resources/Examples
One of the best parts about Github is that you can view how other people set up their own work. Here are some past BSE portfolios that are awesome examples. You can view how they set up their portfolio, and you can view their index.md files to understand how they implemented different portfolio components.
- [Example 1](https://trashytuber.github.io/YimingJiaBlueStamp/)
- [Example 2](https://sviatil0.github.io/Sviatoslav_BSE/)
- [Example 3](https://arneshkumar.github.io/arneshbluestamp/)

To watch the BSE tutorial on how to create a portfolio, click here. -->
