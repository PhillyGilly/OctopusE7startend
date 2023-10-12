# Octopus E7 detect and switch Start and End times

This 'how-to' explains detect, switch and use Octopus Economy 7 start end times in Home Assistant.

I had already followed @OliverShingler 's video tutorial https://www.speaktothegeek.co.uk/2023/02/off-peak-tariffs-in-home-assistants-energy-dashboard/ which splits the Energy Import in the Home Assistant "Energy" tab into Off-peak (blue) and Peak (green) as shown below.

![image](https://github.com/PhillyGilly/OctopusE7startend/assets/56273663/1a56fbfb-7bb2-48dd-9ea4-dd877ee0f76e)

The automation for this uses two DateTime Input Helpers as shown:

![image](https://github.com/PhillyGilly/OctopusE7startend/assets/56273663/c5a97e63-87f4-4fc2-a478-5cd8b00f6bdb)

Once I had these, I incoporated them into other automations, for example to run the dishwasher over-night.
At the end of the summer I switched tariff to Octopus Economy 7 because, in the absence of any PV generation, I needed a tariff with over 5hr 17min off-peak to fully charge my batteries.

I use the Hildebrand-Glow IHD with Local MQTT and a few days later I noticed that the new tariff details were being shown in it.

![newtariff](https://github.com/PhillyGilly/OctopusE7startend/assets/56273663/135d0d9e-f234-4cfe-ae3b-6c190466a159)

The availability of this information in a display made me wonder if the actual tariffs and their timing were availble via local MQTT. They aren't.
I have a preference for using local integrations with Home Assistant, however for something which is not going to change often I am willing to use cloud integrations.

@BottlecapDave 's Home Assistant-Octopus Energy integration https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy provides for a live feed from Octopus.
To use this you need an API-key which is found under Developer Settings on the Personal Details page of your Octopus Account viewed on a PC.

![octopus api](https://github.com/PhillyGilly/OctopusE7startend/assets/56273663/b4c5eaee-3922-4a69-a105-5f1fd9c3e8bf)

When the Octopus Energy Integration is installed, it creates a new electricity meter device with seven new sensors.  One of these, the fourth one down, shows the status of Electricity Off Peak ('turned on' or 'turned off') .

![new device](https://github.com/PhillyGilly/OctopusE7startend/assets/56273663/0bad4a64-fd36-4bf7-af91-9885dfc65067)

The times that Off-peak starts and ends are shown in the event log. Note that these values during British Summer Time are not the same as the IHD values.
You can decide which ones to believe, however, as Economy 7 has to work with non-smart meters too, I am inclined to think that the Off-peak is 00:30 to 07:30 GMT which is what Octopus subsequently confirmed to me in an email.

After this the automation to keep the InputDate Helpers aligned to the Octopus API values is fairly straight forward:
```
alias: Set Offpeak & Peak Tariffs and Times
description: ""
trigger:
  - platform: state
    entity_id:
      - >-
        binary_sensor.octopus_energy_electricity_xxxxxx_yyyyyy_off_peak
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
        binary_sensor.octopus_energy_electricity_xxxxxx_yyyyyy_off_peak
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
The new input_datetime values can be used to trigger automations and specifically to set the charging slot start and end times in GivTCP  as:
```
  - service: select.select_option
    target:
      entity_id: select.givtcp_zzzzzz_charge_start_time_slot_1
    data:
      option: >-
        {%- set hour = state_attr('input_datetime.off_peak_energy_start','hour') -%}
        {%- set minute = state_attr('input_datetime.off_peak_energy_start','minute') -%}
        {{ '{:02}:{:02}:00'.format(hour, minute) }} 
  - service: select.select_option
    target:
      entity_id: select.givtcp_zzzzzz_charge_end_time_slot_1
    data:
      option: >-
        {%- set hour = state_attr('input_datetime.off_peak_energy_end','hour') -%}
        {%- set minute = state_attr('input_datetime.off_peak_energy_end','minute') - 2 -%}
        {{ '{:02}:{:02}:00'.format(hour, minute) }} 
```
I have incorporated these services into an updated version of @OliverShingler 's automation described in the August 2022 update of https://www.speaktothegeek.co.uk/2022/06/givenergy-giv-ac-3-0-battery-home-assistant-and-solar-forecast-automation/. I have updated it to work with Solcast. My strategy is to charge my batteries to 80% less the Solcast forecast for generation in the morning.  Since solar generation starts pretty much at the end of the E7 slot, even on a high winter PV generation forecast there is no risk that I will have to inport form the Grid at peak rate.

I hope this is interesting and/or helpful.
