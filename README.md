# Wavecoustic

An ESP-32 with INMP441 mic to analyse piano sound using FFT (Fast Fourier transform) to light up WS2812B led on the corresponding across the piano key.

## Description

This is a controller for a LED strip that lights up a piano key when pressed. I know a version of this had been done before; however, as far as I know, there had only been a version where the input for the LED was from MIDI, which is impossible to do with an older piano or an acoustic piano. I wanted to make this project so much because looking back I would have practiced the piano so much more if it wasn't so boring. So, my goal is to use an actual microphone input and FFT to translate the sound wave signal into individual key notes. So that it becomes fun and usable. 

## Getting Started

### Usage

* Place the 1.4m led strip over the piano in possition.
* Charge your external LIPO-battery using Tp4056 usb-c charging module.
* Plug the external lipo battery in and rest it on the holder (please be aware of the polarlity).
* Set the setting mode and brightness via esp32-accesspoint captive portal.

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

* The code is in C++ made for ESP-32
* The library version might change over time, please recheck it again.
* Sidenote: I'm not really sure about the code but I tried my best to polish it because my friend quit half way :(
```
#include <Arduino.h>
#include <driver/i2s.h>
#include <arduinoFFT.h>
#include <FastLED.h>
#include <WiFi.h>
#include <ESPAsyncWebServer.h>
#include <AsyncWebSocket.h>
#include <DNSServer.h>

#define FREQ_MAX        4186.0f
#define FREQ_MIN        27.5f
#define SAMPLE          1024
#define SAMPLE_RATE     22050
#define MAG_THRESHOLD   3000

#define I2S_WS          25
#define I2S_SD          34
#define I2S_SCK         32
#define LED_PIN         13
#define JAX_ADC_PIN     35

#define NUM_KEY         88
#define LED_PER_KEY     2
#define NUM_LED         176
#define MAX_AMPS        1500
#define BRIGHTNESS      150

typedef enum { INPUT_I2S, INPUT_JAX } InputMode;
InputMode inputMode = INPUT_I2S;

typedef enum { COLOR_BY_FREQ, COLOR_RAINBOW, COLOR_WHITE } ColorMode;
ColorMode colorMode = COLOR_BY_FREQ;
uint8_t ledBrightness = BRIGHTNESS;
const char* NOTE_NAMES[NUM_KEY] = {
    "A0","A#0","B0",
    "C1","C#1","D1","D#1","E1","F1","F#1","G1","G#1","A1","A#1","B1",
    "C2","C#2","D2","D#2","E2","F2","F#2","G2","G#2","A2","A#2","B2",
    "C3","C#3","D3","D#3","E3","F3","F#3","G3","G#3","A3","A#3","B3",
    "C4","C#4","D4","D#4","E4","F4","F#4","G4","G#4","A4","A#4","B4",
    "C5","C#5","D5","D#5","E5","F5","F#5","G5","G#5","A5","A#5","B5",
    "C6","C#6","D6","D#6","E6","F6","F#6","G6","G#6","A6","A#6","B6",
    "C7","C#7","D7","D#7","E7","F7","F#7","G7","G#7","A7","A#7","B7",
    "C8"
};
CRGB leds[NUM_LED];

float vReal[SAMPLE];
float vImag[SAMPLE];
ArduinoFFT<float> FFT = ArduinoFFT<float>(vReal, vImag, SAMPLE, SAMPLE_RATE);

static int32_t i2sRaw[SAMPLE];
static char wsBuf[3072];
float noiseFloor = 3000.0f;
float peakMag    = 0.0f;
float peakFreq   = 0.0f;
const char* AP_SSID = "PianoLED";
DNSServer       dnsServer;
AsyncWebServer  server(80);
AsyncWebSocket  ws("/ws");

const char PORTAL_HTML[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>PianoLED</title>
<style>
  *{margin:0;padding:0;box-sizing:border-box}
  body{
    background:#0a0a0f;color:#fff;
    font-family:'Segoe UI',sans-serif;
    min-height:100vh;display:flex;flex-direction:column;
    align-items:center;justify-content:center;overflow:hidden;
  }
  h1{font-size:1.4rem;letter-spacing:.3em;color:#888;text-transform:uppercase;margin-bottom:3rem}
  #note-display{
    display:flex;flex-wrap:wrap;justify-content:center;
    gap:14px;min-height:90px;align-items:center;max-width:90vw;
  }
  .note-chip{
    padding:10px 20px;border-radius:50px;
    font-size:1.3rem;font-weight:700;letter-spacing:.05em;
    transition:opacity .6s ease,transform .6s ease;
    opacity:1;transform:scale(1);white-space:nowrap;
  }
  .note-chip.fading{opacity:0;transform:scale(.85)}
  #status{margin-top:3rem;font-size:.75rem;color:#333;letter-spacing:.1em}
  .dot{
    display:inline-block;width:7px;height:7px;border-radius:50%;
    background:#333;margin-right:6px;vertical-align:middle;transition:background .3s;
  }
  .dot.live{background:#00ff88}
  #settings{
    margin-top:2rem;display:flex;gap:12px;flex-wrap:wrap;justify-content:center;
  }
  select{
    background:#111;color:#aaa;border:1px solid #333;
    border-radius:8px;padding:6px 12px;font-size:.8rem;cursor:pointer;
  }
  select:focus{outline:none;border-color:#555}
  input[type=range]{width:160px;accent-color:#00b4ff}
</style>
</head>
<body>
<h1>PianoLED</h1>
<div id="note-display">
  <span style="color:#333;font-size:.9rem;letter-spacing:.2em">LISTENING...</span>
</div>
<div id="settings">
  <select id="inputSel" onchange="sendSetting('input',this.value)">
    <option value="0">🎙 I2S Mic</option>
    <option value="1"> 3.5mm Jack</option>
  </select>
  <select id="colorSel" onchange="sendSetting('color',this.value)">
    <option value="0">By Frequency</option>
    <option value="1"> Rainbow</option>
    <option value="2"> White</option>
  </select>
  <input type="range" id="volSel" min="0" max="255" value="150" oninput="sendSetting('brightness',this.value)">
</div>
<div id="status">
  <span class="dot" id="dot"></span>
  <span id="status-text">CONNECTING</span>
</div>
<script>
  function noteColor(freq){
    if(freq<120)  return{bg:'rgba(255,30,0,.15)',  border:'#ff1e00',text:'#ff6040'};
    if(freq<250)  return{bg:'rgba(255,80,0,.15)',  border:'#ff5000',text:'#ff8040'};
    if(freq<500)  return{bg:'rgba(255,200,0,.15)', border:'#ffc800',text:'#ffd040'};
    if(freq<1000) return{bg:'rgba(0,255,80,.15)',  border:'#00ff50',text:'#40ff80'};
    if(freq<2000) return{bg:'rgba(0,180,255,.15)', border:'#00b4ff',text:'#40ccff'};
    if(freq<3000) return{bg:'rgba(0,80,255,.15)',  border:'#0050ff',text:'#4080ff'};
    return         {bg:'rgba(180,0,255,.15)',       border:'#b400ff',text:'#cc40ff'};
  }

  const display   = document.getElementById('note-display');
  const dot       = document.getElementById('dot');
  const stText    = document.getElementById('status-text');
  const activeChips = {};
  const FADE_DELAY_MS = 600;

  function showNotes(notes){
    const incoming = new Set(notes.map(n=>n.name));
    
    if(notes.length > 0 && display.querySelector('span')) {
      display.innerHTML = '';
    }

    notes.forEach(({name,freq})=>{
      if(activeChips[name]){
        clearTimeout(activeChips[name].fadeTimer);
        activeChips[name].el.classList.remove('fading');
        activeChips[name].fadeTimer=null;
      } else {
        const c=noteColor(freq);
        const chip=document.createElement('div');
        chip.className='note-chip';
        chip.textContent=name;
        chip.style.background=c.bg;
        chip.style.border=`1.5px solid ${c.border}`;
        chip.style.color=c.text;
        chip.style.boxShadow=`0 0 18px ${c.border}55`;
        display.appendChild(chip);
        activeChips[name]={el:chip,fadeTimer:null};
      }
    });

    Object.keys(activeChips).forEach(name=>{
      if(!incoming.has(name)&&!activeChips[name].fadeTimer){
        activeChips[name].el.classList.add('fading');
        activeChips[name].fadeTimer=setTimeout(()=>{
          activeChips[name].el.remove();
          delete activeChips[name];
          if(Object.keys(activeChips).length===0)
            display.innerHTML='<span style="color:#333;font-size:.9rem;letter-spacing:.2em">LISTENING...</span>';
        },FADE_DELAY_MS);
      }
    });
  }

  function sendSetting(key,val){
    if(ws&&ws.readyState===1) ws.send(JSON.stringify({[key]:parseInt(val)}));
  }

  let ws;
  function connect(){
    ws=new WebSocket('ws://'+location.hostname+'/ws');
    ws.onopen=()=>{dot.classList.add('live');stText.textContent='LIVE'};
    ws.onmessage=(e)=>{
      try{
        const data=JSON.parse(e.data);
        if(Array.isArray(data)) showNotes(data);
        if(data.input!==undefined) document.getElementById('inputSel').value=data.input;
        if(data.color!==undefined) document.getElementById('colorSel').value=data.color;
        if(data.brightness!==undefined) document.getElementById('volSel').value=data.brightness;
      }catch(err){}
    };
    ws.onclose=()=>{
      dot.classList.remove('live');
      stText.textContent='RECONNECTING...';
      setTimeout(connect,1500);
    };
  }
  connect();
</script>
</body>
</html>
)rawliteral";

void setupI2S() {
    i2s_config_t config = {
        .mode                 = (i2s_mode_t)(I2S_MODE_MASTER | I2S_MODE_RX),
        .sample_rate          = SAMPLE_RATE,
        .bits_per_sample      = I2S_BITS_PER_SAMPLE_32BIT,
        .channel_format       = I2S_CHANNEL_FMT_ONLY_LEFT,
        .communication_format = (i2s_comm_format_t)(I2S_COMM_FORMAT_STAND_I2S),
        .intr_alloc_flags     = ESP_INTR_FLAG_LEVEL1,
        .dma_buf_count        = 8,
        .dma_buf_len          = 64,
        .use_apll             = false,
        .tx_desc_auto_clear   = false,
        .fixed_mclk           = 0
    };
    i2s_pin_config_t pin_config = {
        .bck_io_num   = I2S_SCK,
        .ws_io_num    = I2S_WS,
        .data_out_num = I2S_PIN_NO_CHANGE,
        .data_in_num  = I2S_SD
    };
    i2s_driver_install(I2S_NUM_0, &config, 0, NULL);
    i2s_set_pin(I2S_NUM_0, &pin_config);
}

void calibrateNoise() {
    Serial.println("Calibrating noise floor... keep quiet for 2 seconds");
    float maxNoise = 0.0f;
    for (int pass = 0; pass < 4; pass++) {
        size_t bytesRead;
        i2s_read(I2S_NUM_0, i2sRaw, sizeof(i2sRaw), &bytesRead, portMAX_DELAY);
        int sampleCount = bytesRead / sizeof(int32_t);
        for (int i = 0; i < sampleCount && i < SAMPLE; i++) {
            vReal[i] = (float)i2sRaw[i] / 2147483648.0f;
            vImag[i] = 0.0f;
        }
        FFT.windowing(FFT_WIN_TYP_HANN, FFT_FORWARD);
        FFT.compute(FFT_FORWARD);
        FFT.complexToMagnitude();
        for (int i = 2; i < SAMPLE / 2; i++) {
            if (vReal[i] > maxNoise) maxNoise = vReal[i];
        }
        delay(500);
    }
    noiseFloor = maxNoise * 1.5f;
    Serial.printf("Noise floor set to: %.4f\n", noiseFloor);
}

void readI2S() {
    size_t bytesRead;
    i2s_read(I2S_NUM_0, i2sRaw, sizeof(i2sRaw), &bytesRead, portMAX_DELAY);
    for (int i = 0; i < SAMPLE; i++) {
        vReal[i] = (float)i2sRaw[i] / 2147483648.0f;
        vImag[i] = 0.0f;
    }
}

void readJax() {
    for (int i = 0; i < SAMPLE; i++) {
        unsigned long startUs = micros();
        int raw = analogRead(JAX_ADC_PIN);
        vReal[i] = (float)(raw - 2048) / 2048.0f;
        vImag[i] = 0.0f;
        while ((micros() - startUs) < 45) {
            
        }
    }
}

int freqToled(float freq) {
    if (freq < FREQ_MIN || freq > FREQ_MAX) return -1;
    float logMin = logf(FREQ_MIN);
    float logMax = logf(FREQ_MAX);
    float logF   = logf(freq);
    int key = (int)((logF - logMin) / (logMax - logMin) * (NUM_KEY - 1));
    return constrain(key, 0, NUM_KEY - 1);
}

bool isHarmonic(float freq, float fundamental) {
    for (int h = 2; h <= 6; h++) {
        float harmonic = fundamental * h;
        float margin   = harmonic * 0.02f;
        if (fabsf(freq - harmonic) < margin) return true;
    }
    return false;
}

CRGB freqColor(float freq, int keyIndex) {
    switch (colorMode) {
        case COLOR_BY_FREQ:
            if (freq < 120)  return CRGB(255,  30,   0);
            if (freq < 250)  return CRGB(255,  80,   0);
            if (freq < 500)  return CRGB(255, 200,   0);
            if (freq < 1000) return CRGB(  0, 255,  80);
            if (freq < 2000) return CRGB(  0, 180, 255);
            if (freq < 3000) return CRGB(  0,  80, 255);
            return               CRGB(180,   0, 255);
        case COLOR_RAINBOW:
            return CHSV((uint8_t)((keyIndex * 255) / NUM_KEY), 255, 255);
        case COLOR_WHITE:
        default:
            return CRGB::White;
    }
}

void onWsEvent(AsyncWebSocket* server, AsyncWebSocketClient* client,
               AwsEventType type, void* arg, uint8_t* data, size_t len) {
    if (type == WS_EVT_CONNECT) {
        char buf[64];
        snprintf(buf, sizeof(buf), "{\"input\":%d,\"color\":%d,\"brightness\":%d}",
                 (int)inputMode, (int)colorMode, (int)ledBrightness);
        client->text(buf);
    } else if (type == WS_EVT_DATA) {
        AwsFrameInfo* info = (AwsFrameInfo*)arg;
        if (info->final && info->index == 0 && info->len == len && info->opcode == WS_TEXT) {
            String msg = String((char*)data).substring(0, len);
            int inputVal = -1, colorVal = -1, brightnessVal = -1;
            int idx = msg.indexOf("\"input\":");
            if (idx >= 0) inputVal = msg.substring(idx + 8).toInt();
            idx = msg.indexOf("\"color\":");
            if (idx >= 0) colorVal = msg.substring(idx + 8).toInt();
            idx = msg.indexOf("\"brightness\":");
            if (idx >= 0) brightnessVal = msg.substring(idx + 13).toInt();
            if (inputVal >= 0 && inputVal <= 1) inputMode = (InputMode)inputVal;
            if (colorVal >= 0 && colorVal <= 2) colorMode = (ColorMode)colorVal;
            if (brightnessVal >= 0 && brightnessVal <= 255) {
                ledBrightness = (uint8_t)brightnessVal;
                FastLED.setBrightness(ledBrightness);
            }
            char buf[64];
            snprintf(buf, sizeof(buf), "{\"input\":%d,\"color\":%d,\"brightness\":%d}",
                     (int)inputMode, (int)colorMode, (int)ledBrightness);
            server->textAll(buf);
        }
    }
}

void setupPortal() {
    WiFi.softAP(AP_SSID);
    IPAddress ip = WiFi.softAPIP();
    dnsServer.start(53, "*", ip);
    ws.onEvent(onWsEvent);
    server.addHandler(&ws);
    server.onNotFound([](AsyncWebServerRequest* req) {
        if (req->host() != WiFi.softAPIP().toString()) {
            req->redirect("http://" + WiFi.softAPIP().toString());
        } else {
            req->send_P(200, "text/html", PORTAL_HTML);
        }
    });
    server.begin();
    Serial.printf("Portal up at: http://%s\n", ip.toString().c_str());
}

void broadcastNotes() {
    if (ws.count() == 0) return;
    if (peakMag <= noiseFloor || peakMag <= MAG_THRESHOLD) {
        ws.textAll("[]");
        return;
    }
    float freqResolution = (float)SAMPLE_RATE / SAMPLE;

    int  pos   = 0;
    bool first = true;
    wsBuf[pos++] = '[';
    for (int i = 2; i < SAMPLE / 2; i++) {
        float freq = i * freqResolution;
        if (freq < FREQ_MIN || freq > FREQ_MAX) continue;
        float mag = vReal[i];
        if (mag < noiseFloor)           continue;
        if (mag < peakMag * 0.2f)       continue;
        if (isHarmonic(freq, peakFreq)) continue;
        int kidx = freqToled(freq);
        if (kidx < 0) continue;
        
        if (pos >= (int)sizeof(wsBuf) - 64) break;

        int written = snprintf(wsBuf + pos, sizeof(wsBuf) - pos - 2,
                               "%s{\"name\":\"%s\",\"freq\":%.1f}",
                               first ? "" : ",",
                               NOTE_NAMES[kidx],
                               freq);
        if (written > 0 && pos + written < (int)sizeof(wsBuf) - 2) {
            pos  += written;
            first = false;
        }
    }
    wsBuf[pos++] = ']';
    wsBuf[pos]   = '\0';
    ws.textAll(wsBuf);
}

void setup() {
    Serial.begin(115200);
    FastLED.addLeds<WS2812B, LED_PIN, GRB>(leds, NUM_LED);
    FastLED.setBrightness(BRIGHTNESS);
    FastLED.setMaxPowerInVoltsAndAmps(5, MAX_AMPS);
    setupI2S();
    pinMode(JAX_ADC_PIN, INPUT);
    analogReadResolution(12);
    analogSetAttenuation(ADC_ATTEN_DB_11);
    calibrateNoise();
    setupPortal();
    Serial.println("Piano LED ready");
}

void loop() {
    dnsServer.processNextRequest();
    if (inputMode == INPUT_I2S) {
        readI2S();
    } else {
        readJax();
    }
    FFT.windowing(FFT_WIN_TYP_HANN, FFT_FORWARD);
    FFT.compute(FFT_FORWARD);
    FFT.complexToMagnitude();
    peakMag  = 0.0f;
    peakFreq = 0.0f;
    float freqResolution = (float)SAMPLE_RATE / SAMPLE;
    for (int i = 2; i < SAMPLE / 2; i++) {
        float freq = i * freqResolution;
        if (freq < FREQ_MIN || freq > FREQ_MAX) continue;
        if (vReal[i] > peakMag) {
            peakMag  = vReal[i];
            peakFreq = freq;
        }
    }
    if (peakMag <= noiseFloor) {
        fadeToBlackBy(leds, NUM_LED, 20);
        FastLED.show();
        broadcastNotes();
        return;
    }
    FastLED.clear();
    if (peakMag > MAG_THRESHOLD && peakFreq > 0.0f) {
        for (int i = 2; i < SAMPLE / 2; i++) {
            float freq = i * freqResolution;
            if (freq < FREQ_MIN || freq > FREQ_MAX) continue;
            float mag = vReal[i];
            if (mag < peakMag * 0.2f)       continue;
            if (isHarmonic(freq, peakFreq)) continue;
            int kidx = freqToled(freq);
            if (kidx < 0) continue;
            int led0 = kidx * LED_PER_KEY;
            int led1 = led0 + 1;
            uint8_t bright = (uint8_t)constrain((mag / peakMag) * 255.0f, 0.0f, 255.0f);
            CRGB color = freqColor(freq, kidx);
            color.nscale8(bright);
            if (color.getLuma() > leds[led0].getLuma()) {
                leds[led0] = color;
                if (led1 < NUM_LED) leds[led1] = color;
            }
        }
    }
    broadcastNotes();
    Serial.printf("Peak: %.1f Hz  Magnitude: %.4f  Threshold: %.4f\n",
                  peakFreq, peakMag, noiseFloor);
    FastLED.show();
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
<img width="1066" height="255" alt="image" src="https://github.com/user-attachments/assets/f935058f-1cd8-41fb-bf4d-d9253fb1b696" />

## Rendered Pictures
<img width="1051" height="238" alt="Screenshot 2026-06-18 172513" src="https://github.com/user-attachments/assets/4d41c5bb-d906-4908-aacb-cc6fe0f857d9" />


## Captive Portal interface
<img width="373" height="445" alt="Screenshot 2026-06-18 032915" src="https://github.com/user-attachments/assets/ff6574ff-b611-4831-ab5b-705859b5a18c" />


## Schematic 
<img width="1769" height="1251" alt="SCH_Schematic1_2026-06-14 (1)_page-0001" src="https://github.com/user-attachments/assets/8684b099-6665-4e5c-9a5a-702e105f07e2" />


# Zine Poster
<img width="1410" height="2000" alt="waaavecoustic (1)" src="https://github.com/user-attachments/assets/4f861a55-da71-4618-9970-9b8e22a6dd7f" />


