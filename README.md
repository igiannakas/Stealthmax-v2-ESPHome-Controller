# Stealthmax v2 ESP Home Controller
This project aims to create an alternative controller for the Stealthmax V2 based on an ESP32 micro controller using ESP Home. It offers native integration into Home Assistant, for improved reporting and more granular automation control, while also being able to interface to Klipper using the ESP Home API.

This project assumes a level of familiarity with setting up, flashing and confoiguring ESP32 devices with ESP Home. 

## Important notes on the VOC sensor values
The stock ESPHome SGP4x component does not expose the raw VOC sensor values for use. What this means is that over long prints the VOC values will converge to 100, as the software, over time, calibrates itself to the increased ambient VOC values, which it considers the new "baseline".

However this is of no benefit for use in a 3d printer! The VOC sensors are used as a proxy to illustrate the carbon effectiveness and to identify when/if carbon replacement is needed. To achieve this, we require the raw VOC sensor data and to calculate the VOC drop between intake and exhaust sensors. 

To do this, this configuration calls a custom external component that extends the stock ESPHome module, exposing the raw sensor values. This module also has implemented a "calibration" routine, which creates a static baseline (offset) for the intake and exhaust sensors, which is triggered on demand. It also implements a simple sigmoid function to report the sensor values to a range that is familiar and similar to the stock VOC read outs. It also calculates the delta between the statically calibrated intake and exhaust VOC sensor values.

As such, the user needs to set the "baseline" for the intake and exhaust sensors, ideally at the start of the print, after the chamber is heatsoaked and before the print begins. This allows the user to monitor the effectiveness of the filtering media through the VOC drop between the two sensors.

The component exposes all of those sensors, the baseline and the raw data in the Home Assistant UI as below:

![image](https://github.com/user-attachments/assets/9b882253-1b0f-427e-bd5b-799516dc457f)
![image](https://github.com/user-attachments/assets/e866ef71-0c28-4a1f-a6ef-436e6a2350fb)

The diagnostic section contains the interim values and baseline offsets. The sensors section contains the usable data - the VOC value suffixed by Manual Calibration is the manually offset value that is set to 100 when the set baseline button is triggered. The non-suffixed values are the stock reported sensor values.

![IMG_6568](https://github.com/user-attachments/assets/7b96b0cb-4057-4568-8a6c-009c103870ea)



## Bill of Materials (BOM)
1. ESP32 D1 Mini, or similar ESP32 micro controllers
2. A buck-boost converter to power the ESP32 from the 24V printer power supply
3. The standard VOC and Temperature sensor stacks for Stealthmax (sgp40 and bme280)
4. The standard BOM servo (FT90M)
5. The standard Stealthmax V2 BOM Fan (24V - MX7020GBH2)

## Hardware Setup

### Power Wiring
1. Connect your Stealthmax fan PWM pin (yellow) to your printer's main board PWM fan control header pin
2. Connect your Stealthmax fan Tach pin (blue) to your printer's main board tach header pin
3. Connect your buck-boost converter inputs to the printer's 24V power supply output. Trim the converter pots so you get 5V output.
4. Connect the fan power (+) and (-) wires to the input of your buck-boost converter (24V).
5. Flash your ESP32 controller over USB with the included configuration ESPHome YAML file. Disconnect from your computer.
6. Connect your ESP32 controller to the 5V output of your buck-boost converter.
7. Boot the system and verify that the PWM fan works correctly in Klipper. Verify that the ESP32 device boots and you can see it in Home Assistant.

### Sensor Wiring (3.3V & GND)
| Stack    | SDA      | SCL      |  
|----------|----------|----------|  
| Intake   | GPIO23   | GPIO18   |  
| Exhaust  | GPIO19   | GPIO26   |  

> ⚠️ **Note:** Some BME280 sensors require I²C address `0x76` (default is `0x77`)

### Servo Wiring
- PWM → **GPIO16**  
- Power from ESP32 5V/GND *or* buck-boost  


## Servo Calibration
1. Power ESP32 **without servo connected**  
2. Set vent to **OFF** in ESPHome/Home Assistant  
3. Connect servo + install flap  
4. Enable vent → verify no mechanical overtravel  
5. Adjust `servo_vent_closed`/`servo_vent_open` in YAML if needed  
6. Alternatively, enable the commented out code segment (Servo Control) to create a slider that allows you to manually set an arbitrary servo % value. Note the servo closed and servo open values, divide by 100 and set them in the above YAML placeholders.

## Software Setup

The included configuration exposes the temperature, humidity, VOC and servo sensors and switch to be used in automations in Home Assistant and via Klipper.

### Klipper Macro Setup
1. Install Gcode Shell Command via KIAUH
2. Setup the below macros in Klipper, replacing the IP below with the IP of your ESP32 device. The below macros allow you to open/close the ventilation flap and to set the VOC baseline ("zero out" the sensors) during your print start routines. 

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

### Klipper Macro Integration

#### Servo Vent
Firstly integrate the servo activated vent and VOC baselining in your print start macro. 

My prefered method is to check the requested chamber temperature by the slicer. If, for example it is over 20C, this means you are printing a material that needs enclosed printing, hence close the flap. After all print preparation is done but before the hotend is set to the print temperature, set the VOC baseline so the sensors are "zeroed out".
```
{% if target_chamber|int > 20 %}
  CLOSE_VENT
  .... preheat chamber, bed etc ...
{% else %} 
  OPEN_VENT
  .... preheat bed etc
{% endif %}

... QGL ...
... Bed Mesh ...

 SET_VOC_BASELINE

... Heat hotend to print temp...

```

#### Fan Speed Control
Firstly set up the PWM fan as below in your printer.cfg file:
```
[fan_generic exhaust_fan]
pin: PA6
hardware_pwm: true
tachometer_pin: PC2
tachometer_ppr: 2
max_power: 0.8 # 100pc is way too fast and unecessary
shutdown_speed: 1.0
kick_start_time: 1.0
off_below: 0.10
```
Then set up the fan control routines as below in your print start macro

```
[gcode_macro PRINT_START]
gcode:
....

{% if target_chamber|int > 20 %}
  CLOSE_VENT
  SET_FAN_SPEED FAN=exhaust_fan SPEED=0.3 # approx 3k RPM
{% else %}
  OPEN_VENT 
  SET_FAN_SPEED FAN=exhaust_fan SPEED=0.3 # approx 3k RPM
  UPDATE_DELAYED_GCODE ID=adaptive_exhaust_fan DURATION=30
{% endif %}

... QGL ...
... Bed Mesh ...

 SET_VOC_BASELINE

... Heat hotend to print temp...

```

```
[gcode_macro PRINT_END]
gcode:
    .....

    TURN_OFF_HEATERS
    cancel_adaptive_exhaust

    .....
```

```
[delayed_gcode adaptive_exhaust_fan]
gcode:
    {% if printer["temperature_sensor chamber"].temperature > 40 %}
        SET_FAN_SPEED FAN=exhaust_fan SPEED=1.0
        RESPOND TYPE=echo MSG="Chamber temperature over 40C. Danger for heat creep."
    {% elif printer["temperature_sensor chamber"].temperature > 39 %}
        SET_FAN_SPEED FAN=exhaust_fan SPEED=0.8
    {% elif printer["temperature_sensor chamber"].temperature > 38 %}
        SET_FAN_SPEED FAN=exhaust_fan SPEED=0.7
    {% elif printer["temperature_sensor chamber"].temperature > 37 %}
        SET_FAN_SPEED FAN=exhaust_fan SPEED=0.6
    {% else %}
        SET_FAN_SPEED FAN=exhaust_fan SPEED=0.4
    {% endif %}
    UPDATE_DELAYED_GCODE ID=adaptive_exhaust_fan DURATION=60

[gcode_macro cancel_adaptive_exhaust]
gcode:
  RESPOND TYPE=echo MSG="Adaptive exhaust cancelled"
  SET_FAN_SPEED FAN=exhaust_fan SPEED=0.0
  UPDATE_DELAYED_GCODE ID=adaptive_exhaust_fan DURATION=0
```

The above snippet sets the exhaust fan to 30%, which is approximately 3k RPM when printing ABS/ASA/high temp material. If we are printing a low temp material, it calls the adaptive exhaust fan delayed gcode that gradually ramps up / down the fan depending on chamber temperature to allow the chamber to remain below 40C. In the print end macro, it calls the cancel_adaptive_exhaust to stop adapting the exhaust speed and turn off the fan.


You can view my complete configuration here: https://github.com/igiannakas/Voron-backups/tree/main/printer_data/config
