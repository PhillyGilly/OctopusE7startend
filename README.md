# Octopus E7 detect and switch Start and End times

This 'how-to' explains detect and switch Octopus Economy 7 start end times in Home assistant.

I had already followed @OliverShingler 's video tutorial https://youtu.be/bb5aDtn2wUA?si=1HiZRisSy4Mt7U4B which splits the Energy Import in the Home Assistant "Energy" tab into Off-peak (blue) and Peak (green) as shown below.

![image](https://github.com/PhillyGilly/OctopusE7startend/assets/56273663/1a56fbfb-7bb2-48dd-9ea4-dd877ee0f76e)

The automation for this uses two DateTime Input Helpers as shown:

![image](https://github.com/PhillyGilly/OctopusE7startend/assets/56273663/c5a97e63-87f4-4fc2-a478-5cd8b00f6bdb)

Once I had these, I incoporated them into other automations, for example to run the dishwasher over-night.
At the end of the summer I switched tariff to Octopus Economy 7 because, in the absence of any PV generation, I needed a tariff with over 5hr 17min off-peak to fully charge my batteries.

I use the Hildebrand-Glow IHD with Local MQTT and a few days later I noticed that the new tariff details were being shown in it.

![newtariff](https://github.com/PhillyGilly/OctopusE7startend/assets/56273663/135d0d9e-f234-4cfe-ae3b-6c190466a159)

The availability of this information in a dispaly made me wonder if the actual tariffs and their timing were availble via local MQTT. They aren't.
I have a preference for using local integrations with Home Assistant, however for something which is not going to change often I am willing to use cloud integrations.

@BottlecapDave 's Home Assistant-Octopus Energy integration https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy provides for a live feed from Octopus.
To use this you need an API-key which is found under Developer Options on the Personal Details page of your Octopus Account viewed on a PC.

![octopus api](https://github.com/PhillyGilly/OctopusE7startend/assets/56273663/b4c5eaee-3922-4a69-a105-5f1fd9c3e8bf)

When installed this creates a new electricity meter device with seven new sensors and the one which shows the status of Electricity Off Peak (turned on or turned off) the fourth one down.

![new device](https://github.com/PhillyGilly/OctopusE7startend/assets/56273663/0bad4a64-fd36-4bf7-af91-9885dfc65067)

The times that Off-peak stats and ends are shown in the event log. Note that these values during British Summer Time are not the same as the IHD values.
You can decide which ones to believe, however, as Economy 7 has to work with non-smart meters too, I am inclined to think that the Off-peak is 00:30 to 07:30 GMT which is what Octopus subsequently confirmed to me in an email.

After this the automation to keep the InputDate Helpers aligned to the Octopus API values is fairly straight forward:
```
alias: 60a. Set Offpeak & Peak Tariffs and Times
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
      entity_id: select.givtcp_ed2248g390_charge_start_time_slot_1
    data:
      option: >-
        {%if state_attr('input_datetime.off_peak_energy_start','hour') < 10 %}
          {{ "0" + (state_attr('input_datetime.off_peak_energy_start','hour')|string) + ":" + (state_attr('input_datetime.off_peak_energy_start','minute')|string) + ":00" }}
        {%else%}
          {{ (state_attr('input_datetime.off_peak_energy_start','hour')|string) + ":" + (state_attr('input_datetime.off_peak_energy_start','minute')|string) + ":00" }}
        {%endif%}
  - service: select.select_option
    target:
      entity_id: select.givtcp_ed2248g390_charge_end_time_slot_1
    data:
      option: >-
        {%if state_attr('input_datetime.off_peak_energy_end','hour') < 10 %}
          {{ "0" + (state_attr('input_datetime.off_peak_energy_end','hour')|string) + ":" + ((state_attr('input_datetime.off_peak_energy_end','minute')-2)|string) + ":00" }}
        {%else%}
          {{ (state_attr('input_datetime.off_peak_energy_end','hour')|string) + ":" + ((state_attr('input_datetime.off_peak_energy_end','minute')-2)|string) + ":00" }}
       {%endif%}
```
