---
title: "Home Assistant automations"
date: 2024-12-08
categories: [DIY]
tags: [Homelab]
header:
    overlay_image: /assets/images/2024-12-08-home-assistant-automations/overlay.jpg
    caption: ""
---

In this blog post, I will share how I use [Home Assistant](https://www.home-assistant.io/) to automate my home in a pragmatic way.
Pragmatic means that I'm not trying to automate everything, but only things that make sense to me.
For example, for lights automation in the house, I use PIR or microwave [motion detectors](https://en.wikipedia.org/wiki/Motion_detector) connected directly to lights, instead of fancy Zigbee switches.

## Modes

I introduced 3 modes to control how the house behaves.
The modes improve readability and maintainability of my automations.
They are implemented as simple [boolean flags](https://www.home-assistant.io/integrations/input_boolean/).

![Modes](/assets/images/2024-12-08-home-assistant-automations/input_boolean_modes.png)

### Away Mode

* When I arm all alarm partitions, automation enables "Away Mode". This mode controls how the house behaves when I'm not at home.
* When I disarm all alarm partitions, automation disables "Away Mode". It means I'm back home.

### Night Mode

* Time based automation enables "Night Mode" when it's time to sleep and disables it when it's time to wake up. This mode controls how the house behaves when I'm sleeping.

### Eco Mode

* Manual switch that controls heating and cooling systems. When I'm going to be away for a long time, I enable "Eco Mode" to save energy.

## Automations

I organized automations in Home Assistant using [packages](https://www.home-assistant.io/docs/configuration/packages/).
It helps me to keep configuration clean and organized.

![Packages](/assets/images/2024-12-08-home-assistant-automations/automation_packages.png)

When I'm writing this blog post, I have over 50 automations configured in Home Assistant and the number is growing.
I will not list all of them here, but I will give you some examples.

### CCTV

* Automation enables CCTV detections when I leave home, and disables it when I come back. All cameras belong to the same group, so I can enable/disable them all at once.
* Even if I'm at home, automation enables CCTV detections when "Night Mode" is active.
* If CCTV recognizes a person, automation sends a notification to my phones with the screenshot of detected entity.
* From the notification I'm able to open CCTV live feed using [actionable notifications](https://companion.home-assistant.io/docs/notifications/actionable-notifications/).
* I can also snooze detections for 5 minutes to avoid getting notifications when I've already knew who is at the yard. This automation uses [timer](https://www.home-assistant.io/integrations/timer/) for reliable snoozing.

![CCTV](/assets/images/2024-12-08-home-assistant-automations/cctv.jpg)

### Heating

* My heat pump uses heating curve to adjust heating power based on outside temperature. This is out of scope for my Home Assistant automations, but I have some automations to control house main thermostat. It enables me to adjust heating curve a bit based on my needs.
* Automation decreases heating temperature when "Eco Mode" is active.
* Automation turns off domestic hot water heating when "Eco Mode" is enabled.
* When I'm at home, automation increases heating one hour before "Night Mode" ends. It's nice to wake up in warm house.
* Higher temperature back to normal 3 hours later.
* When I'm at home, automation increases heating 2 hours before "Night Mode" starts. It's nice to take a shower in warm bathroom.
* Higher temperature back to normal 1 hour after "Night Mode" starts, it helps with drying floor and towels after shower.

### Facade Lights

* Automation turns on facade lights in afternoon/evening when the [Sun](https://www.home-assistant.io/integrations/sun/) is 4 degrees below the horizon. "Night Mode" turns them off.
* To help me wake me up in the morning, automation turns on facade lights when "Night Mode" ends. When the Sun is 1 degree below the horizon, lights are turned off.
* If I'm not at home, facade lights automation is off.

![Facade Lights](/assets/images/2024-12-08-home-assistant-automations/facade_lights.jpg)

### Water

* Automation closes main water valve when I leave home, and opens it when I come back.
* The water valve remains open when dishwasher is running, and closes when it's done.

### Dishwasher

* When dishwasher is done, automation sends a notification to my son's phone, unloading the dishwasher is his job.
* "Away Mode" disables dishwasher notifications.

### Waste Collection

* Based on calendar, automation sends a notification day before waste collection to my phones. I can take out the trash in the evening.
* "Away Mode" disables waste collection notifications.

![Waste Collection](/assets/images/2024-12-08-home-assistant-automations/waste_collection_calendar.png)

### Power

* When I'm out of home, automation disables electric sockets on the terrace.
* When UPS is running on battery, automation sends a notification about power outage to my phones.
* When UPS battery is below 20%, [NUT](https://networkupstools.org/) shuts down my servers gracefully.
* From time to time, I switch off Zigbee controlled socket in my [rack](/blog/2024/06/30/homlab-hardware/) to simulate power outage and test UPS. I'm going to automate this test in the future.

### Windows

* When I'm leaving home, automation sends a notification if any window is open.
* When I'm out of home, automation sends a notification if any window opens.

### Garage Gate

* When alarm garage partition is armed, automation disables garage gate switch and enables it again when alarm is disarmed.
* Because my Zigbee garage switch is not a momentary switch, automation turns it off after 5 seconds.

### Fire

* When smoke detectors are triggered, besides the alarm sound, automation sends a notification to my phones.

## Summary

I have a lot of fun with Home Assistant but I'm always ask my wife and son if they are happy with the automations 😀

![Approved by Wife](/assets/images/2024-12-08-home-assistant-automations/approved_by_wife.jpg)