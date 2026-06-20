# Gyrogotchi

A tamagotchi that reacts when you flip them in a 4x4 cube keychain using xiao esp32 s3 and mpu6050.

## Description

This is a tamagotchi keychain that introduce another way to interact with the chracter using a gyroscope to be able to tilt the character to triger different animation and around like having a small hamster in a keychain. I wanted to create this because I got inspired by those old 2000 toys where there's innovation and new idea and everythings seems fascinating. My goal here is to make a keychain personalized for me and also double as a nametag I did this by adding a secret code that when I hold a button for 1.5 seccond my credit and name would pop up.

## Getting Started

### Usage

* charge from the usbc port
* press the capacitive touch sensor embeded at the left side 
* shake to wake the character up from sleeping and tilt or flip to trigger different animation
* hold 1.5 seccond to trigger my personalized credit
* connect to gyrogotchi captive portal acess point to broad cast message to the screen and synce time

### Assembling

* Print out the 3d parts (body,slider,topper and please use the 3mf file I beg you)
* Put the esp-32 into the main body
* Stack the esp32 cover lid over the esp32 to lock it in place
* stick all the pin header into it own slot
* follow the wiring guide below
* put the microphone into the 6 headerpin slot
* stuff all the wiring jumper wires into the case
* connect the led strip into the external headerpin
* charge the lipo with the Tp4056 module (not include in the circuit and the case)
* connect the lipo to the external header pin
* cut your 2m led strip to your piano size
* use a multimeter to check the boost converter and adjust first using a screw prior to adding it to the circuit
* take the sliding lid and close it
* mount the led and the component box on your piano
* The only point require soldering are the INMP441 mic and the GND wires. 

### Wiring
* INMP441 - ESP32
* VDD  - 3.3V
* GND  - GND
* SCK  - GPIO32
* WS   - GPIO25
* SD   - GPIO34
* L/R  - GND
* WS2812B LED STRIP
* DIN  - 300Ω → GPIO13
* GND  - GND
* MT3608 boost converter (use a multimeter to check the boost and adjust first prior to adding electricity)
* +Vout - led strip 5v pin
* -Vout - GND
* Lipo battery (preferably 20 000mah)
* Splice the red wire before connecting it to the boost converter and connect it to ESP-32 VIN pin
* Red wire   - MT3608 +Vin pin
* Black wire - GND
* Splice all GND together and add solder and a electrical tape!!!
* Refers to the schematic down below for more indepth guide.

### Firmware

* The code is in C++ made for Xiao ESP-32 S3
* The library version might change over time, please recheck it again.
* feel free to edit and change the credit however you want!
```
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <WiFi.h>
#include <DNSServer.h>
#include <WebServer.h>
#include <time.h>

#define TOUCH_PIN      1
#define VIBRATOR_PIN   2

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);
Adafruit_MPU6050 mpu;

WebServer server(80);
DNSServer dnsServer;
const byte DNS_PORT = 53;

int blink_counter = 0;
int zzz_counter = 0;
bool is_sleeping = true; 

unsigned long last_interaction_time = 0;
const unsigned long SLEEP_TIMEOUT = 45000;

unsigned long touch_start_time = 0;
bool touch_was_active = false;
bool showing_credits = false;

String cowsay_text = "";
unsigned long custom_text_display_expiry = 0;

RTC_DATA_ATTR bool has_time_synced = false;
RTC_DATA_ATTR unsigned long rtc_sync_time = 0;
RTC_DATA_ATTR unsigned long rtc_sync_millis = 0;

bool wifi_active = false;
unsigned long wifi_start_millis = 0;
const unsigned long WIFI_PORTAL_TIMEOUT = 60000;

int getBatteryPercentage() {
  pinMode(14, OUTPUT);
  digitalWrite(14, LOW);
  
  int raw = analogRead(10);
  float voltage = (raw / 4096.0) * 3.3 * 2.0; 

  int percent = map(voltage * 100, 330, 420, 0, 100);
  return constrain(percent, 0, 100);
}

String getOrientationString(float x, float y) {
  if (y < -4.0) return "Upside Down";
  if (x < -4.0) return "Tilting Left";
  if (x > 4.0)  return "Tilting Right";
  return "Flat / Centered";
}

unsigned long getCalculatedEpoch() {
  if (!has_time_synced) return 0;
  return rtc_sync_time + ((millis() - rtc_sync_millis) / 1000);
}

void goToDeepSleep() {
  if (wifi_active) {
    WiFi.softAPdisconnect(true);
    WiFi.mode(WIFI_OFF);
  }
  display.clearDisplay();
  display.display();
  digitalWrite(VIBRATOR_PIN, LOW);
  
  esp_deep_sleep_enable_ext0_wakeup((gpio_num_t)TOUCH_PIN, 1);
  esp_deep_sleep_start();
}

void syncTimeFromParam() {
  if (server.hasArg("epoch")) {
    long long epoch = atoll(server.arg("epoch").c_str());
    struct timeval tv;
    tv.tv_sec = epoch;
    tv.tv_usec = 0;
    settimeofday(&tv, NULL);
    rtc_sync_time = epoch;
    rtc_sync_millis = millis();
    has_time_synced = true;
  }
}

void handleRoot() {
  last_interaction_time = millis();
  syncTimeFromParam();
  
  sensors_event_t a, g, t;
  mpu.getEvent(&a, &g, &t);
  
  String html = "<!DOCTYPE html><html><head><meta name='viewport' content='width=device-width, initial-scale=1.0'>";
  html += "<style>body{background:#0f141c;color:#00ffcc;font-family:monospace;text-align:center;padding:20px;}";
  html += ".card{background:#161f2e;border:2px solid #00ffcc;border-radius:8px;padding:15px;margin:15px auto;max-width:400px;box-shadow:0 4px 10px rgba(0,255,204,0.2);}";
  html += "input[type='text']{width:80%;padding:10px;background:#0f141c;border:1px solid #00ffcc;color:#fff;font-family:monospace;margin-bottom:10px;}";
  html += "input[type='submit']{background:#00ffcc;color:#0f141c;border:none;padding:10px 20px;font-weight:bold;cursor:pointer;}";
  html += "</style><script>";
  html += "function syncAndPost(form) {";
  html += "  document.getElementById('deviceTime').value = Math.floor(Date.now() / 1000);";
  html += "  return true;";
  html += "}";
  html += "window.onload = function() {";
  html += "  var xhr = new XMLHttpRequest();";
  html += "  xhr.open('GET', '/sync?epoch=' + Math.floor(Date.now() / 1000), true);";
  html += "  xhr.send();";
  html += "};";
  html += "</script></head><body>";
  html += "<h1>TAMACUBE DASHBOARD</h1>";
  
  html += "<div class='card'>";
  html += "<h3>SYSTEM TELEMETRY</h3>";
  html += "<p><b>Battery Level:</b> " + String(getBatteryPercentage()) + "%</p>";
  html += "<p><b>Orientation:</b> " + getOrientationString(a.acceleration.x, a.acceleration.y) + "</p>";
  
  unsigned long current_epoch = getCalculatedEpoch();
  if(current_epoch > 0) {
    time_t now = (time_t)current_epoch;
    struct tm* timeinfo = localtime(&now);
    char timeStr[20];
    strftime(timeStr, sizeof(timeStr), "%Y-%m-%d %H:%M:%S", timeinfo);
    html += "<p><b>Synced Time:</b> " + String(timeStr) + "</p>";
  } else {
    html += "<p><b>Synced Time:</b> Not Synchronized</p>";
  }
  html += "</div>";

  html += "<div class='card'>";
  html += "<h3>COWSAY TRANSMITTER</h3>";
  html += "<form action='/msg' method='POST' onsubmit='return syncAndPost(this);'>";
  html += "<input type='hidden' id='deviceTime' name='epoch'>";
  html += "<input type='text' name='cowsay' placeholder='Type temporary alert message...' maxlength='32'><br>";
  html += "<input type='submit' value='TRANSMIT TO CUBE'>";
  html += "</form></div>";
  html += "</body></html>";
  server.send(200, "text/html", html);
}

void handleMessage() {
  last_interaction_time = millis(); 
  syncTimeFromParam();
  if (server.hasArg("cowsay")) {
    cowsay_text = server.arg("cowsay");
    custom_text_display_expiry = millis() + 15000;
  }
  server.sendHeader("Location", "/", true);
  server.send(302, "text/plain", "");
}

void handleSync() {
  syncTimeFromParam();
  server.send(200, "text/plain", "OK");
}

void handleCaptiveProbe() {
  server.sendHeader("Location", "http://192.168.4.1/", true);
  server.send(302, "text/plain", "");
}

void setup() {
  Serial.begin(115200);
  pinMode(VIBRATOR_PIN, OUTPUT);
  pinMode(TOUCH_PIN, INPUT);
  
  Wire.begin(4, 5); 

  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  mpu.begin();
  mpu.setAccelerometerRange(MPU6050_RANGE_8_G); 
  display.clearDisplay();
  
  digitalWrite(VIBRATOR_PIN, HIGH);
  delay(150);
  digitalWrite(VIBRATOR_PIN, LOW);

  esp_sleep_wakeup_cause_t wakeup_reason = esp_sleep_get_wakeup_cause();
  if (wakeup_reason == ESP_SLEEP_WAKEUP_EXT0) {
    WiFi.softAP("Gyro-Gotchi");
    dnsServer.start(DNS_PORT, "*", WiFi.softAPIP()); 
    server.on("/", HTTP_GET, handleRoot);
    server.on("/msg", HTTP_POST, handleMessage);
    server.on("/sync", HTTP_GET, handleSync);
    server.on("/generate_204", HTTP_GET, handleRoot);
    server.on("/fwlink", HTTP_GET, handleRoot);
    server.onNotFound(handleCaptiveProbe); 
    server.begin();
    wifi_active = true;
    wifi_start_millis = millis();
  }

  if (has_time_synced) {
    struct timeval tv;
    tv.tv_sec = rtc_sync_time + ((millis() - rtc_sync_millis) / 1000);
    tv.tv_usec = 0;
    settimeofday(&tv, NULL);
  }

  last_interaction_time = millis(); 
}

void loop() {
  if (wifi_active) {
    if (millis() - wifi_start_millis > WIFI_PORTAL_TIMEOUT) {
      WiFi.softAPdisconnect(true);
      WiFi.mode(WIFI_OFF);
      dnsServer.stop();
      server.stop();
      wifi_active = false;
    } else {
      dnsServer.processNextRequest();
      server.handleClient();
    }
  }

  sensors_event_t accelerometer_data, gyroscope_data, temperature_data;
  mpu.getEvent(&accelerometer_data, &gyroscope_data, &temperature_data);
  
  display.clearDisplay();
  display.setTextSize(1); 
  display.setTextColor(SSD1306_WHITE);

  bool touch_detected = (digitalRead(TOUCH_PIN) == HIGH);
  float total_force = abs(accelerometer_data.acceleration.x) + abs(accelerometer_data.acceleration.y) + abs(accelerometer_data.acceleration.z);

  if (touch_detected) {
    if (!touch_was_active) {
      touch_start_time = millis();
      touch_was_active = true;
    } else if (millis() - touch_start_time > 1500) {
      showing_credits = true;
    }
  } else {
    touch_was_active = false;
    showing_credits = false;
  }

  if (touch_detected) {
    last_interaction_time = millis(); 
  }

  if (millis() - last_interaction_time > SLEEP_TIMEOUT) {
    goToDeepSleep();
  }

  if (showing_credits) {
    display.setCursor(0, 0);
    display.println("--- CREDITS ---");
    display.setCursor(0, 20);
    display.println("Gyro-Gotchi 06/2026");
    display.println("tengy.org/site");
    display.println("@tengy_ez (Teng 2011)");
  }
  else if (millis() < custom_text_display_expiry) {
    display.setCursor(0, 0);
    display.println("< Broadcast Alert >");
    display.setCursor(0, 24);
    display.print(" ");
    display.println(cowsay_text);
    display.drawRect(0, 16, 128, 32, SSD1306_WHITE);
  }
  else if (is_sleeping) {
    digitalWrite(VIBRATOR_PIN, LOW);

    if (touch_detected) {
      is_sleeping = false;
      digitalWrite(VIBRATOR_PIN, HIGH);
      delay(150);
      digitalWrite(VIBRATOR_PIN, LOW);
      last_interaction_time = millis();
      return;
    }

    display.setCursor(0, 0);
    int bat_pct = getBatteryPercentage();
    
    zzz_counter++;
    unsigned long current_epoch = getCalculatedEpoch();
    if(current_epoch > 0) {
      time_t now = (time_t)current_epoch;
      struct tm* timeinfo = localtime(&now);
      if (zzz_counter < 10) {
        display.printf("[%d%%] %02d:%02d Zzz...", bat_pct, timeinfo->tm_hour, timeinfo->tm_min);
      } else if (zzz_counter < 20) {
        display.printf("[%d%%] %02d:%02d zZzZ...", bat_pct, timeinfo->tm_hour, timeinfo->tm_min);
      } else {
        zzz_counter = 0;
      }
    } else {
      if (zzz_counter < 10) {
        display.printf("[%d%%] Zzz...", bat_pct);
      } else if (zzz_counter < 20) {
        display.printf("[%d%%] zZzZ...", bat_pct);
      } else {
        zzz_counter = 0;
      }
    }

    display.drawLine(25, 38, 45, 38, SSD1306_WHITE); 
    display.drawLine(85, 38, 105, 38, SSD1306_WHITE); 

    display.drawLine(15, 45, 23, 45, SSD1306_WHITE);
    display.drawLine(105, 45, 113, 45, SSD1306_WHITE);
    
    display.drawLine(58, 48, 70, 48, SSD1306_WHITE);
  }
  else {
    display.setCursor(0, 0);
    if (touch_detected) {
      digitalWrite(VIBRATOR_PIN, LOW); 
      display.println("<3 lolol thaat tickles!");
      
      display.drawLine(25, 43, 35, 33, SSD1306_WHITE);
      display.drawLine(35, 33, 45, 43, SSD1306_WHITE);
      
      display.drawLine(83, 43, 93, 33, SSD1306_WHITE);
      display.drawLine(93, 33, 103, 43, SSD1306_WHITE);
      
      display.drawLine(15, 45, 23, 41, SSD1306_WHITE);
      display.drawLine(15, 49, 23, 45, SSD1306_WHITE);
      display.drawLine(105, 41, 113, 45, SSD1306_WHITE);
      display.drawLine(105, 45, 113, 49, SSD1306_WHITE);
      
      display.drawCircle(64, 46, 5, SSD1306_WHITE);
      blink_counter = 0;
    }
    else if (total_force > 25.0) {
      digitalWrite(VIBRATOR_PIN, HIGH);
      display.println("stopppp it im dizzy!");
      
      display.drawLine(25, 33, 45, 53, SSD1306_WHITE);
      display.drawLine(45, 33, 25, 53, SSD1306_WHITE);
      
      display.drawLine(85, 33, 105, 53, SSD1306_WHITE);
      display.drawLine(105, 33, 85, 53, SSD1306_WHITE);
      
      display.drawLine(54, 55, 74, 55, SSD1306_WHITE);
      blink_counter = 0; 
    }
    else if (accelerometer_data.acceleration.y < -4.0) {
      if (millis() % 400 < 200) {
        digitalWrite(VIBRATOR_PIN, HIGH);
      } else {
        digitalWrite(VIBRATOR_PIN, LOW);
      }
      display.println("ay flip me back right now!....... stupid humans");
      
      display.drawLine(20, 33, 45, 43, SSD1306_WHITE);
      display.drawLine(20, 48, 45, 48, SSD1306_WHITE);
      
      display.drawLine(108, 33, 83, 43, SSD1306_WHITE);
      display.drawLine(108, 48, 83, 48, SSD1306_WHITE);
      
      display.drawCircle(64, 57, 4, SSD1306_WHITE);
      blink_counter = 0; 
    }
    else if (accelerometer_data.acceleration.x < -4.0) {
      digitalWrite(VIBRATOR_PIN, LOW);
      display.println("woahh tilting left......");
      
      display.fillCircle(30, 38, 10, SSD1306_WHITE); 
      display.fillCircle(26, 33, 2, SSD1306_BLACK);
      
      display.fillCircle(88, 38, 10, SSD1306_WHITE);
      display.fillCircle(84, 33, 2, SSD1306_BLACK);
      
      display.drawLine(10, 45, 18, 45, SSD1306_WHITE);
      display.drawLine(100, 45, 108, 45, SSD1306_WHITE);
      
      display.drawLine(53, 48, 59, 44, SSD1306_WHITE);
      display.drawLine(59, 44, 65, 48, SSD1306_WHITE);
      display.drawLine(65, 48, 71, 44, SSD1306_WHITE);
      display.drawLine(71, 44, 77, 48, SSD1306_WHITE);
    }
    else if (accelerometer_data.acceleration.x > 4.0) {
      digitalWrite(VIBRATOR_PIN, LOW);
      display.println("yooooo im tilting right!!??");
      
      display.fillCircle(40, 38, 10, SSD1306_WHITE); 
      display.fillCircle(42, 33, 2, SSD1306_BLACK);
      
      display.fillCircle(98, 38, 10, SSD1306_WHITE);
      display.fillCircle(100, 33, 2, SSD1306_BLACK);
      
      display.drawLine(20, 45, 28, 45, SSD1306_WHITE);
      display.drawLine(110, 45, 118, 45, SSD1306_WHITE);
      
      display.drawLine(63, 48, 69, 44, SSD1306_WHITE);
      display.drawLine(69, 44, 75, 48, SSD1306_WHITE);
      display.drawLine(75, 48, 81, 44, SSD1306_WHITE);
      display.drawLine(81, 44, 87, 48, SSD1306_WHITE);
    }
    else {
      digitalWrite(VIBRATOR_PIN, LOW); 
      display.println("sup buddy");
      
      blink_counter++; 
      if (blink_counter == 28 || blink_counter == 29) {
        display.drawLine(25, 38, 45, 38, SSD1306_WHITE); 
        display.drawLine(85, 38, 105, 38, SSD1306_WHITE); 
      } 
      else if (blink_counter >= 30) {
        blink_counter = 0; 
      } 
      else {
        display.fillCircle(35, 38, 10, SSD1306_WHITE); 
        display.fillCircle(33, 33, 2, SSD1306_BLACK);
        
        display.fillCircle(93, 38, 10, SSD1306_WHITE);
        display.fillCircle(91, 33, 2, SSD1306_BLACK);
      }
      
      display.drawLine(15, 45, 23, 45, SSD1306_WHITE);
      display.drawLine(105, 45, 113, 45, SSD1306_WHITE);
      
      display.drawLine(58, 46, 64, 50, SSD1306_WHITE);
      display.drawLine(64, 50, 70, 46, SSD1306_WHITE);
      display.drawLine(70, 46, 76, 50, SSD1306_WHITE);
      display.drawLine(76, 50, 82, 46, SSD1306_WHITE);
    }
  }

  display.display();
  delay(100);
}
```
## Materials

This is the BOM of the entire project, just buy the normal one not the pcb version and just use jumper wire with minimal soldering required except for slicing GND together.
```
ESP-32 devkit v1           1 pcs.
INMP441 mic                1 pcs.
WS2812B led                2m 144/m
LIPO-battery               1 (reccomend atleast 10 000mah)
Tp4056 usb-c               1 (does not include in the build, only use the charge lipo externally)
MT3608 boost converter     1 pcs.
Female Jumper wires        30 pcs.
Male Jumper wires          15 pcs.
Pin headers                5 pcs.
Electrical tape            1 roll
PLA 3d printing/time       ≈15g ≈45min
```

## Assembled Pictures


## Rendered Pictures



## Captive Portal interface
<img width="360" height="331" alt="Screenshot 2026-06-21 011252" src="https://github.com/user-attachments/assets/a9a8f4f2-8326-47e3-a5a6-8e3d1da70e30" />



## Schematic 
<img width="929" height="656" alt="Screenshot 2026-06-21 033731" src="https://github.com/user-attachments/assets/a90014f5-55d1-4fad-a386-3fb38072e482" />


# Zine Poster
<img width="1410" height="2000" alt="GYRO-GOTCHI" src="https://github.com/user-attachments/assets/22cb9461-760b-4205-a37e-98948c4b3306" />



