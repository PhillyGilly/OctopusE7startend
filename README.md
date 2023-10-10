# Octopus E7 detect and switch Start and End times

This 'how-to' explains detect and switch Octopus Economy 7 start end times in Home assistant.

I had already followed @OliverShingler 's video tutorial https://youtu.be/bb5aDtn2wUA?si=1HiZRisSy4Mt7U4B which splits the Energy Import in the Home Assistant "Energy" tab into Off-peak (blue) and Peak (green) as shown below.

![image](https://github.com/PhillyGilly/OctopusE7startend/assets/56273663/1a56fbfb-7bb2-48dd-9ea4-dd877ee0f76e)

The automation for this uses two DateTime Input Helpers as shown:

![image](https://github.com/PhillyGilly/OctopusE7startend/assets/56273663/17743ba3-db5a-4d38-9d67-c348f4a92521)

Once I had these, I incoporated them into other automations, for example to run the dishwasher over-night, or more importantly to charge GivEnergy batteries over-night.
At the end of the summer I switched tariff to Octopus Economy 7 because I needed 5hr 17min to fully charge my batteries.

I use the Hildebrand-Glow IHD with Local MQTT and a few days later I noticed that the new tariff details were being shown in it.

![2023-10-10 07 33 31](https://github.com/PhillyGilly/OctopusE7startend/assets/56273663/6a30d88a-f055-41c7-9755-957b54bb6a39)

The availability of this information in a dispaly made me wonder if the actual tariffs and their timing were availble via local MQTT. They aren't.
I have a preference for using local integrations with Home Assistant, however for something which is not going to change often I am willing to use cloud integrations.

@BottlecapDave 's Home Assistant-Octopus Energy integration https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy provides for a live feed from Octopus.
To use this you need an API-key which is found under Developer Options on the Personal Details page of your Octopus Account viewed on a PC.
When installed this creates seven new sensors and the one which shows the change from Off-peak to Peak is the fifth one down.

![image](https://github.com/PhillyGilly/OctopusE7startend/assets/56273663/1ae069d9-b430-4803-a21c-3ba908dd30e7)
 
Note that the start- and end-times shown here during British Summer Time are not the same as the IHD values.
You can decide which ones to believe, however, as Economy 7 has to work with non-smart meters too, I am inclined to think that the Off-peak is 00:30 to 07:30 GMT which is what Octopus subsequently confirmed to me in an email.

After this the automation to keep the InputDate Helpers aligned to the Octopus API values is fairly straight forward:
```
alias: 60a. Set Offpeak & Peak Tariffs and Times
description: ""
trigger:
  - platform: state
    entity_id:
      - >-
        binary_sensor.octopus_energy_electricity_21l4045356_1170000688431_off_peak
    from: "on"
    to: "off"
    for:
      hours: 0
      minutes: 0
      seconds: 10
    id: offpeakend
  - platform: state
    entity_id:
      - >-
        binary_sensor.octopus_energy_electricity_21l4045356_1170000688431_off_peak
    from: "off"
    to: "on"
    for:
      hours: 0
      minutes: 0
      seconds: 10
    id: offpeakstart
condition: []
action:
  - choose:
      - conditions:
          - condition: trigger
            id:
              - offpeakstart
        sequence:
          - service: input_datetime.set_datetime
            data:
              timestamp: "{{ now().timestamp() }}"
            target:
              entity_id: input_datetime.off_peak_energy_start
      - conditions:
          - condition: trigger
            id:
              - offpeakend
        sequence:
          - service: input_datetime.set_datetime
            data:
              timestamp: "{{ now().timestamp() }}"
            target:
              entity_id: input_datetime.off_peak_energy_end
mode: single
```
