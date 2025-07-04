**Stealthmax v2 ESP Home Controller**
This project aims to create an alternative controller for the Stealthmax V2 based on an ESP32 micro controller using ESP Home. It offers native integration into Home Assistant, for improved reporting and more granular automation control, while also being able to interface to Klipper using the ESP Home API.

**BOM**
1. ESP32 D1 Mini, or similar ESP32 micro controllers
2. A buck-boost converter to power the ESP32 from the 24V printer power supply
3. The standard VOC and Temperature sensor stacks for Stealthmax (sgp40 and bme280)
4. The standard BOM servo (FT90M)
5. The standard Stealthmax V2 BOM Fan (24V - MX7020GBH2)

**Power Wiring:**
1. Connect your Stealthmax fan PWM pin (yellow) to your printer's main board PWM fan control header pin
2. Connect your Stealthmax fan Tach pin (blue) to your printer's main board tach header pin
3. Connect your buck-boost converter inputs to the printer's 24V power supply output. Trim the converter pots so you get 5V output.
4. Connect the fan power (+) and (-) wires to the input of your buck-boost converter (24V).
5. Flash your ESP32 controller over USB with the included configuration ESPHome YAML file. Disconnect from your computer.
6. Connect your ESP32 controller to the 5V output of your buck-boost converter.
7. Boot the system and verify that the PWM fan works correctly in Klipper. Verify that the ESP32 device boots and you can see it in Home Assistant.

**Sensor wiring**
1. Sensors need to be wired to the 3.3V output of the ESP32 device and Ground.
2. The intake stack sensors SDA wire is connected to GPIO23 and SCL wire to GPIO18
3. The exhaust stack sensors SDA wire is connected to GPIO19 abd SCL wire to GPIO26

Please note that depending on the source of your BME sensors you may need to change their I2C address from 0x77 to 0x76.

**Servo wiring**
1. Servo needs to be connected to the 5V and ground on the ESP32 device or directly to the output of the buck-boost converter.
2. Servo PWM wire is connected to GPIO16 of the ESP32 device.

**Servo calibration:**
1. Power on the device with the servo unplugged
2. Switch the vent off (closed) in Homeassistant/ESPHome Web UI
3. Plug in the servo
4. Install the servo flap
5. Switch on the vent and verify that it doesnt exceed the space limits of the Stealthmax.
6. If it does, adjust servo_vent_closed and servo_vent_open values in the Yaml
7. Alternatively, enable the commented out code segment (Servo Control) to create a slider that allows you to manually set an arbitrary servo % value. Note the servo closed and servo open values, divide by 100 and set them in the above YAML placeholders.

**Software setup:**

The included configuration exposes the temperature, humidity, VOC and servo sensors and switch to be used in automations in Home Assistant and via Klipper.

Integrating to Klipper:
1. Install Gcode Shell Command via KIAUH
2. Setup the below macros in Klipper, replacing the IP below with the IP of your ESP32 device. 

```
[gcode_shell_command open_vent_cmd]
command: curl -X POST http://172.24.4.180/switch/vent_control/turn_on
timeout: 30.0
verbose: false

[gcode_shell_command close_vent_cmd]
command: curl -X POST http://172.24.4.180/switch/vent_control/turn_off
timeout: 30.0
verbose: false

[gcode_shell_command set_voc_baseline_cmd]
command: curl -X POST http://172.24.4.180/button/set_voc_baseline/press
timeout: 30.0
verbose: false

[gcode_macro open_vent]
gcode:
  RUN_SHELL_COMMAND CMD=open_vent_cmd

[gcode_macro close_vent]
gcode:
  RUN_SHELL_COMMAND CMD=close_vent_cmd

[gcode_macro set_voc_baseline]
gcode:
  RUN_SHELL_COMMAND CMD=set_voc_baseline_cmd
```
The above macros allow you to open/close the ventilation flap and to set the VOC baseline ("zero out" the sensors) during your print start routines. 

One way to integrate this is as below. In your Print Start macro, check the chamber temperature. If over 20C, this means you are printing a material that needs enclosed printing, hence close the flap.
```
{% if target_chamber|int > 20 %}
  CLOSE_VENT
{% else %} 
  OPEN_VENT
{% endif %}
```
The fan is set up as a PWM fan in klipper like the below:
```
[fan_generic exhaust_fan]
pin: PA6
hardware_pwm: true
tachometer_pin: PC2
tachometer_ppr: 2
max_power: 0.8 # 100pc is way too fast and unecessary
shutdown_speed: 1.0
kick_start_time: 1.0
off_below:0.10
```
This can then be called in the print start macro as follows:
```
{% if target_chamber|int > 20 %}
  SET_FAN_SPEED FAN=exhaust_fan SPEED=0.3 # approx 3k RPM
{% else %} 
  SET_FAN_SPEED FAN=exhaust_fan SPEED=0.3 # approx 3k RPM
  UPDATE_DELAYED_GCODE ID=adaptive_exhaust_fan DURATION=30
{% endif %}
```
The above snippet sets the exhaust fan to 30%, which is approximately 3k RPM when the fan is configured to 80% max PWM value, when printing ABS/ASA/high temp material. If we are printing a low temp material, it calls the adaptive exhaust fan delayed gcode that gradually ramps up / down the fan depending on chamber temperature to allow the chamber to remain below 40C. In your print end macro, simply call the cancel_adaptive_exhaust to stop adapting the exhaust speed and turn off the fan.

```
[delayed_gcode adaptive_exhaust_fan]
gcode:
    {% if printer.heater_bed.target < 90 %}
        {% if printer["temperature_sensor chamber"].temperature > 40 %}
           SET_FAN_SPEED FAN=exhaust_fan SPEED=1.0
           RESPOND TYPE=echo MSG="Chamber temperature over 40C. Danger for heat creep."
        {% elif printer["temperature_sensor chamber"].temperature > 39 %}
           SET_FAN_SPEED FAN=exhaust_fan SPEED=0.8
        {% elif printer["temperature_sensor chamber"].temperature > 38 %}
           SET_FAN_SPEED FAN=exhaust_fan SPEED=0.6
        {% elif printer["temperature_sensor chamber"].temperature > 37 %}
           SET_FAN_SPEED FAN=exhaust_fan SPEED=0.4
        {% else %}
           SET_FAN_SPEED FAN=exhaust_fan SPEED=0.3
        {% endif %}
    {% endif %}
    UPDATE_DELAYED_GCODE ID=adaptive_exhaust_fan DURATION=60

[gcode_macro cancel_adaptive_exhaust]
gcode:
  RESPOND TYPE=echo MSG="Adaptive exhaust cancelled"
  SET_FAN_SPEED FAN=exhaust_fan SPEED=0.0
  UPDATE_DELAYED_GCODE ID=adaptive_exhaust_fan DURATION=0
```

You can view my complete configuration here: https://github.com/igiannakas/Voron-backups/tree/main/printer_data/config
