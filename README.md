# Octopus E7 detect and switch Start and End times

This 'how-to' explains detect and switch Octopus Economy 7 start end times in Home assistant.

I had already followed @OliverShinglers video tutorial https://youtu.be/bb5aDtn2wUA?si=1HiZRisSy4Mt7U4B which splits the Energy Import into Offpeak (blue) and Peak (green)N as shown below.

![image](https://github.com/PhillyGilly/OctopusE7startend/assets/56273663/1a56fbfb-7bb2-48dd-9ea4-dd877ee0f76e)

The automation for this uses two DateTime Input Helpers as shown:

![image](https://github.com/PhillyGilly/OctopusE7startend/assets/56273663/17743ba3-db5a-4d38-9d67-c348f4a92521)

Once I had these, I incoporated them into other automations, for example to run the dishwasher over-night, or more importantly to charge GivEnergy batteries over-night.
At the end of the summer I switched tariff to Octopus Economy 7 because I needed 5hr 17min to fully charge my batteries.
Octopus state that the Off-peak is from )0:30 


![image](https://github.com/PhillyGilly/OctopusE7startend/assets/56273663/1ae069d9-b430-4803-a21c-3ba908dd30e7)
 
