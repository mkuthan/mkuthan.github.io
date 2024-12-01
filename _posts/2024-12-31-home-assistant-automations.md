---
title: "Home Assistant Automations"
date: 2024-12-31
categories: [DIY]
tags: [Homelab]
---

## Modes

I introduced 3 modes to control how the house behaves. The modes improve readability and maintainability of my automations. They are implented as simple [boolean flags](https://www.home-assistant.io/integrations/input_boolean/).

### Away Mode

* When I arm all alarm partitions, automation enables "Away Mode". This mode controls how the house behaves when I'm not at home.

* When I disarm all alarm partitions, automation disables "Away Mode". It means I'm back home.

### Night Mode

* Time based automation enables "Night Mode" when it's time to sleep and disables it when it's time to wake up. This mode controls how the house behaves when I'm sleeping.

### Eco Mode

* Manual switch that controls heating and cooling systems. When I'm going to be away for a long time, I enable "Eco Mode" to save energy.

## Automations

When I'm writing this blog post, I have over 50 automations configured in Home Assistant and the number is growing. I will not list all of them here, but I will give you some examples.

### CCTV

* Automation enables CCTV detections when I leave home, and disables it when I come back. All cameras belong to the same group, so I can enable/disable them all at once.
* Even if I'm at home, CCTV detections are enabled when "Night Mode" is active.
* If CCTV recognizes a person, automation sends a notification to my phones with the screenshot of detected entity.
* From the notification I'm able to open CCTV live feed using [actionable notifications](https://companion.home-assistant.io/docs/notifications/actionable-notifications/).
* I can also snooze detections for 5 minutes to avoid getting notifications when I've already knew who is at the yard. This automation uses [timer](https://www.home-assistant.io/integrations/timer/) for reliable snoozing.

### Heating

* My heat pump uses heating curve to adjust heating power based on outside temperature. This is out of scope for my Home Assistant automations, but I have some automations to control house main thermosat. It enables me to adjust heating temperature a bit based on my needs.
* Automation decreases heating temperature by 2 degrees when "Eco Mode" is active.
* Automation turns off domestic hot water heating when "Eco Mode" is enabled.
* When I'm at home, automation increases heating one hour before "Night Mode" starts. It's nice to wake up in warm house.
* Higher temperature back to normal 3 hours later.
* When I'm at home, automation increases heating 2 hours before "Night Mode" starts. It's nice to take a shower in warm bathroom.
* Higher temperaure back to normal 1 hour after "Night Mode" starts, it helps with drying floor and towels after shower.

### Facade Lights

* Automation turns on facade lights in afternoon/evening when the [Sun](https://www.home-assistant.io/integrations/sun/) is 4 degrees below the horizon. "Night Mode" turns them off
* To wake me up in the morning, automation turns on facade lights when "Night Mode" ends. When the Sun is 1 degree below the horizon, lights are turned off.
* If I'm not at home, facade lights automation is off.

### Water

* Automation closes main water valve when I leave home, and opens it when I come back.
* The water valve remains open when diswasher is running, and closes when it's done.

### Dishwasher

* When dishwasher is done, automation sends a notification to my son phone, unloading the dishwasher is his job.
* The notification is sent only when "Away Mode" is disabled.

### Waste Collection

* Based on calendar, automation sends a notification day before waste collection to my phones. I can take out the trash on the evening.
* The notification is sent only when "Away Mode" is disabled.

### Garage Gate

* When alarm garage partition is armed, automation disables garage gate switch and enables it again when alarm is disarmed.

### Power Outage

* When UPS is running on battery, automation sends a notification to my phones.

### Fire

* When smoke detectors are triggered, automation sends a notification to my phones besides the alarm sound.

### Windows
