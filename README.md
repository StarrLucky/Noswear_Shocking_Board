### Noswear Project shocking board firmware (Keil uVision)

This firmware handles commands from android app via BLE to control shocking circuit. 

MCU: NRF51822, Custom design.

Board Schematics: [EasyEDA](https://easyeda.com/fxndstrs/211221312312).

Schematic have 2 led pins: 1 for advertising/device status (12) and other indicates recieving shocking command (13), pin 6 - shocking pin, connected to ne555 timer.  

Board supply voltage range: 3.9-13.2V.   

PCB consists MCP1702 3.3V supply fo NRF51822 chip,
LM555 (low voltage supply) timer IC to drive 8205A Mosfet at ~60Hz.    

Power mosfet connected to HV transformer (or Vibro motor, despite that it could be used without 555 timer, at DC). Note that 8205a dual channel transistor have common drain, so it can't be used separately for vibro and transformer, as it might be.    
RF part design of this board wasn't tested with SWR meter and other special equipment, and was composed by using Nordic Community guidelines and open source projects.  



