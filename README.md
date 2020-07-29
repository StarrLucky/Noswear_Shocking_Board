# Noswear project (shocking watch profanity tracking)  

[Noswear_Java](https://github.com/StarrLucky/Noswear_Java): voice recognition app  

[Noswear_Android_Client](https://github.com/StarrLucky/Noswear_Android_Client): sending commands to NFR51 board over BLE  

--- 

## Noswear Project (shocking watch profanity tracking) shocking board firmware (Keil uVision)

This firmware handles commands from android app via BLE to control shocking circuit. 

MCU: NRF51822, Custom design board.  

Board Schematics: [EasyEDA](https://easyeda.com/fxndstrs/Noswear_Shocking_Board).

Board have led pins - 12 for advertising/device status and 13 to indicate recieving of shocking command. Pin 6 - shocking pin which connected to ne555 timer.  

PCB consists MCP1702 IC (3.3V supply for NRF51822 chip),
LM555 low voltage supply timer IC to drive 8205A MOSFET at ~60Hz.    

Power mosfet connected to HV transformer (or Vibro motor, despite that it could be used without 555 timer, at DC). Note that 8205a dual channel transistor have common drain, so it can't be used separately for vibro and transformer, as it might be.    
RF part design of this board wasn't tested with SWR meter and other special equipment, and was composed by using Nordic Community guidelines and open source projects.  

Board supply voltage range: 3.9-13.2V.  

Note that this voltage will be applied to primary of HV transformer. 


