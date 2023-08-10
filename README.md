# Instructions to deploy the Climate Control demo
AiGo Demonstrator using a thermal control system


## Purpose
The purpose of the demo is to introduce one to the [Eclipse Arrowhead framework](https://arrowhead.eu) with ease.
The demonstrator is a climate control system of *systems* (where a system is an IoT block).
There is a temperature system that measures temperature and exposes it as a service.
There is a valve system that emulates a radiator valve and exposes the valve position as two services (getting the current position or setting a position).
Finally, there is the thermostat system that regulates the valve position based on the desired room temperature and the actual temperature.
It consumes the services of the two other systems.

The problem we face is that things change such that we cannot expect to know the location (URL) of our resources prior to run time.
The idea here is to expose these resources as services by registering them with a service registry (for those old enough,  this is comparable to the telephone directory called Yellow Pages).
When a system seeks these services, it consults the registry to obtain the location (URL) of the described service.
Here, the registry is kept by the Service Registrar.

To refine our search and eventually deal with authorization, there is the Orchestrator system that intercede between the Service Registrar and the Authorization system.

For the demo, there are five systems: the temperature one (ds18b20), the valve one (parallax), the thermostat, the Service Registrar (sr), and the Orchestrator.
They are all compiled for a Raspberry Pi with a [64 bit operating system](https://www.raspberrypi.com/software/).
The Raspberry Pi is chosen since it is a computer with general purpose inputs and outputs ([GPIO on a 40 pin header](https://www.raspberrypi.com/documentation/computers/os.html#gpio-and-the-40-pin-header)).

The demo also includes executables for other platforms (_imac for Intel Mac, _amac for ARM Mac, .exe for Windows 64). These can be  also used. The sensor and actuator on a host that connect to them (i.e., the Raspberry Pi).

## Hardware
The Arrowhead systems used for this demo have been compiled for a Raspberry Pi using a 64 bit operating system.
They can also be compiled for different architectures and it is only when accessing a hardware resource (e.g. sensor or actuator) that one gets an error message.

The minimum hardware required here is:
- A Raspberry Pi (3 or 4 with power supply and micro SD card)
- A 1-Wire temperature sensor
- A Parallax servo motor

To connect to the Raspberry Pi, one can use WiFi or an ethernet cable to one's computer or to the router where one's computer is connected to.
That is all the devices need to be on the same local network.
It should be clear that the hardware list above is for a minimum.
Since it is a distributed system set, one can have more Raspberry Pis, sensors and servos.

## Download
The five applications can be found in this repository, which you can clone.

**Important**: each application must be placed in its own directory since the program generates a configuration file (and they have all the same file name) and the application must be started from the command prompt.

The downloaded files do not have the right to be executable.
To change that, one needs to change that with ```chmod +x your_application_name```.

When the application is started, it looks for its configuration file.
If it is missing, it creates one and shuts down to allow the configuration to be first updated (if necessary).
When restarting the program, all should work and the user is presented with the web server address of the application.
Using this address in a web browser gives the user access to one of the three forms of documentation available.
This is the system's "black box" documentation.

## Service Registrar
The asset of the Service Registrar is the service registry, which is a database.
The selected database for this application is [SQLite](https://en.wikipedia.org/wiki/SQLite) because it is a zero configuration database.

If one wants to test resiliency, one can have more than one Service Registrar per cloud, but all systems must be informed about that in their systemconfig.json file. Only one of them will lead and the others will remain on stand by. The selection is done by the first one that come in operation or detects the failure of the current leading one. They can be on the same host (with different port numbers) or on different computers.

## 1-Wire interface
The [1-Wire hardware interface](https://www.analog.com/en/technical-articles/guide-to-1wire-communication.html) allows to connect several devices to one pin.
It is a serial communication where the resource is addressed by its part number.

The 1 wire interface has to be enabled (i.e., turned on) on the Raspberry Pi.
It can be enabled from the [command prompt](https://www.raspberrypi.com/documentation/computers/configuration.html#one-wire-nonint).
Detailed instruction can be found at [Waveshare](https://www.waveshare.com/wiki/Raspberry_Pi_Tutorial_Series:_1-Wire_DS18B20_Sensor).

Open the Raspberry Pi configuration file using the text editor nano ```sudo nano /boot/config.txt```, and add the following line ```dtoverlay=w1-gpio```.
For the change to take effect, the Raspberry Pi has to be restarted with ```sudo reboot```.

Upon reboot, two kernel modules have to be loaded: ```sudo modprobe w1-gpio``` and ```sudo modprobe w1-therm```. 
The first one handles the one wire GPIO pin and the second one handles the temperature reading.

If all is well and the sensor in connected correctly (with a pull up resistor), each sensor is visible with its unique identification in the ```/sys/bus/w1/devices``` directory(```cd /sys/bus/w1/devices``` and ```ls```).
One can read the temperature using ```cat 28-*/w1_slave```

This unique id (28-....) needs to be added into the systemconfig.json file of the ds18b20 system.
For example, 
```
 {
     "name": "28-0516d0bfd5ff",
     "details": {
        "Location": [
              "Bathroom"
            ]
      }
  }
```
More sensors can be added and they must be separated with a comma.

Once the system configuration is complete, starting the ds18b20 system (```./ds18b20```) will provide the web server's URL to be used in a web browser. 
There all sensors (resources) should appear on the web page and one can GET the service payload with the temperature and timestamp.

Another way to read the value is to use the application [Postman](https://www.postman.com).

The asset here is the hardware 1-wire interface with a collection of temperature sensors as resources.

## Parallax
The idea here is to emulate a valve using a radio control (RC) servo motor. 
It represents the actuator type of asset or thing.

The Parallax servo motor uses pulse width modulation (PWM) to know where it should be from -90° to 90°.
The pulse period is 50 Hz (20 ms) with a high from 620 µs, 1520 µs, 2420 µs.
This high time map to 0°, 90°, 180° respectively. 
The Raspberry Pi, not having a real time OS, has a hard time maintaining these values so the connected servo oscillates slightly.

The configuration for the Parallax system is 
``` "resources": [
      {
         "name": "servo1",
         "details": {
            "Location": [
               "Bathroom"
            ],
            "Model": [
               "standard servo",
               "-90 to +90 degrees"
            ]
         },
         "gpiopin": 18
      }
   ]
```
with the location being the same one as the sensor if they have to work together. The [GPIO pin number](https://www.raspberrypi.com/documentation/computers/os.html#gpio-and-the-40-pin-header) is different than the header pin number.

Alternatively, one could use a [logic analyzer](https://usd.saleae.com/products/saleae-logic-pro-8) to look at the pulse width rather than have a real servo.

Again, one can use the system's http web server to read the position or Postman.
With Postman, one can POST or set a new position at the same resource URL.
The payload is the same as the one received so one can copy the received raw body and paste it in the transmitted one, with a new value.
The updated value can be read (GET) at that URL and seen at the output of the GPIO pin.

If the Service Registrar is up and running, the Parallax system's services should be advertised on the Registrar's web pages.

## The Thermostat system
The Thermostat system, like the Orchestrator, have algorithms as assets.
All systems consumes services from the core systems (e.g., the Service Registrar [with URL in the configuration file] when registering each service).
The Thermostat systems consumes the services of the two application systems (getting temperature and putting valve position).
The Thermostat needs to discover the URL of the desired services at run time.
It does so by asking the Orchestrator for the URL of the desired device by describing it (e.g., temperature, http, Bathroom).

The Orchestrator checks with the Service Registrar if such services are currently available.
If such services are available, the Orchestrator replies to the Thermostat with the URL of the first provider it receives.
One could wonder if the Orchestrator is necessary, but the current implementation is limited.
The Orchestrator is to check authorization and could do refined selection.

It is important that each resource is configured correctly.
For example: 
``` 
"resources": [
      {
         "name": "controller1",
         "details": {
            "Location": [
               "Bathroom"
            ]
         },
         "setpoint": 23,
         "samplingPeriod": 15,
         "kp": 5,
         "lamda": 0.5,
         "ki": 0
      }
  ]
```
      

---
*[dtoverlay](https://github.com/raspberrypi/firmware/blob/master/boot/overlays/README)* stands for Device Tree Overlay.

In the context of a Raspberry Pi, the Device Tree (DT) is a data structure for describing the hardware in the system. This is a way of managing the hardware dependencies and configurations for different components connected to the Raspberry Pi. The Device Tree provides the operating system (like Linux) with the information it needs to manage the hardware resources of the computer correctly.

An Overlay is an extension or modification of the basic device tree structure. Overlays allow changes to the device tree that are specific to certain add-on hardware (like a HAT or external peripheral), certain GPIO configuration, or other system settings.

By writing dtoverlay=w1-gpio in the /boot/config.txt file, you are instructing the system to load the w1-gpio overlay when the system boots, which will set up the necessary settings to enable 1-Wire communication over one of the GPIO pins.
