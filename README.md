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

* Print out the 3d parts (body, cap, minicap, cover) please use the 3mf file I beg you
* place the 500mah lipo batterpack in horizontally
* slide down the cover 3d piece to lock the lipo in place
* put the xiao esp32 s3 in make sure it fits into the usbc slot
* cover the xiao esp32 with the 3d printed minicap
* then place in the mpu6050 on to the two poles above
* then put in the oled screen facing the plastic
* add in the pin to the capacitive touch then solder than cut off the excesspart this need to be flushed to the plastic
* then put in the capacitive touch module make sure the touch side is facing the plastic
* then slide in the vibrating module
* the wire them all up and splice some wire (follow the wiring guide below)
* finally slide in the 3d printed cap

### Wiring
* MPU-6050 to Xiao esp32 S3
* VCC | 3V3
* GND | GND 
* SCL | D5 
* SDA | D4 
* OLED to Xiao esp32 S3
* VCC | 3V3 
* GND | GND 
* SCL | D5 
* SDA | D4 
* Touch (TTP223B) to Xiao esp32 S3
* VCC | 3V3 
* GND | GND 
* SIG | D1 
* Vibrator to Xiao esp32 S3
* Plus | D2
* Negative | GND
* LiPo to Xiao esp32 S3
* Plus | BAT+ 
* Negative | BAT- 
* Splice all 3.3v together and add solder and a electrical tape!!!
* Splice all GND together and add solder and a electrical tape!!!
* Refers to the schematic down below for more indepth guide.

### Firmware

* The code is in C++ made for Xiao ESP-32 S3
* The library version might change over time, please recheck it again.
* feel free to edit and change the credit however you want!
* Check out the [Firmware](firmware/Firmware.ino)
## Materials

This is the BOM of the entire project, just buy the normal one not the pcb version and just use jumper wire with minimal soldering required except for slicing GND together.
```
* Xiao esp32 S3                      1pcs.        20usd
* MPU-6050                           1pcs.        3usd
* 0.96" Inch I2C IIC OLED            1pcs.        3usd
* Touch (TTP223B)                    1pcs.        1usd
* Vibration Motor Module (Catalex)   1pcs.        1usd
* Female Jumper Wires                20pcs.       1usd
* Male Jumper wires                  15pcs.       50cent
* Electrical tape                    1roll        -
* Lipo Battery                       500mah       10usd
* PLA 3d printing/time               ≈25g ≈50min  -
Total cost: 39.5usd
```

## Assembled Pictures
<img width="733" height="187" alt="Screenshot 2026-06-21 034422" src="https://github.com/user-attachments/assets/bf7c9445-5165-43d6-86a2-52e104184e6a" />


## Rendered Pictures

<img width="689" height="176" alt="Screenshot 2026-06-21 034208" src="https://github.com/user-attachments/assets/e35a4386-e3eb-4b59-83c7-9dbae2e4faa5" />


## Captive Portal interface
<img width="360" height="331" alt="Screenshot 2026-06-21 011252" src="https://github.com/user-attachments/assets/a9a8f4f2-8326-47e3-a5a6-8e3d1da70e30" />



## Schematic 
<img width="929" height="656" alt="Screenshot 2026-06-21 033731" src="https://github.com/user-attachments/assets/a90014f5-55d1-4fad-a386-3fb38072e482" />


# Zine Poster
<img width="1410" height="2000" alt="GYRO-GOTCHI" src="https://github.com/user-attachments/assets/22cb9461-760b-4205-a37e-98948c4b3306" />



