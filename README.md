Stealthmax v2 ESP Home Controller
This project aims to create an alternative controller for the Stealthmax V2 based on an ESP32 micro controller using ESP Home. It offers native integration into Home Assistant, for improved reporting and more granular automation control, while also being able to interface to Klipper using the ESP Home API.

BOM
1. ESP32 D1 Mini, or similar ESP32 micro controllers
2. A buck-boost converter to power the ESP32 from the 24V printer power supply
3. The standard VOC and Temperature sensor stacks for Stealthmax (sgp40 and bme280)
4. The standard BOM servo (FT90M)
5. The standard Stealthmax V2 BOM Fan (24V - MX7020GBH2)

Wiring:
1. Connect your Stealthmax fan PWM pin (yellow) to your printer's main board PWM fan control header pin
2. Connect your Stealthmax fan Tach pin (blue) to your printer's main board tach header pin
3. Connect your buck-boost converter inputs to the printer's 24V power supply output. Trim the converter pots so you get 5V output.
4. Connect the fan power (+) and (-) wires to the input of your buck-boost converter (24V).
5. Flash your ESP32 controller over USB with the included configuration ESPHome YAML file. Disconnect from your computer.
6. Connect your ESP32 controller to the 5V output of your buck-boost converter.
7. Boot the system and verify that the PWM fan works correctly in Klipper. Verify that the ESP32 device boots and you can see it in Home Assistant.

Proceed to the sensor wiring
1. Sensors need to be wired to the 3.3V output of the ESP32 device and Ground.
2. The intake stack sensors SDA wire is connected to GPIO23 and SCL wire to GPIO18
3. The exhaust stack sensors SDA wire is connected to GPIO19 abd SCL wire to GPIO26

Please note that depending on the source of your BME sensors you may need to change their I2C address from 0x77 to 
