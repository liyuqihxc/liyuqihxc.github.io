---
title: OBD-II Protocol -- SAE J1850 VPW PWM
categories:
  - [嵌入式开发]
tags: [STM32, OBDII]
---

https://www.cnblogs.com/shangdawei/p/3235907.html

### J1850
The SAE J1850 bus is used for diagnostics and data sharing applications in vehicles. The J1850 bus takes two forms:

- 41.6Kbps Pulse Width Modulated (PWM) two wire differential approach (Ford vehicles)

- 10.4Kbps Variable Pulse Width (VPW) single wire approach (GM vehicles)

The single wire approach may have a bus length up to 35 meters (with 32 nodes). Developed in 1994, J1850 may be phased out for new designs. The J1850 Interface is a class B protocol.

A high resides between 4.25 volts and 20 volts, a low is any thing below 3.5 volts. High and low values are sent as bit symbols (not single bits). Symbols times are 64uS and 128uS for the single wire approach. The bus uses a weak pull-down, the driver needs to pull the bus high, high signals are considered dominant. A passive logic 1 is sent as a 128uS low level, an active logic 1 is sent as a 64uS high. A passive logic 0 is sent as a 64uS low level, an active logic 0 is sent as a 128uS high. The J1850 protocol uses CSMA/CR arbitration.

### J1850 Frame structure

The frame consists of a Start Of Frame [SOF], which is high for 200uS. The Header byte follows the SOF and is one byte long. The data follows the header byte. The one byte CRC [Cyclical Redundancy Check] follows the data field. After the CRC an End Of Data [EOD] symbol is sent. The EOD is sent as a 200uS low pulse.

<p align="center">
{% asset_img j1850_frame.gif J1850 Transmission Signal Format %}
<span>J1850 Transmission Signal Format</span>
</p>

In many cases the J1850 interface bits will be found on an OBDII connector inside a passenger car. OBDII [On-Board Diagnostics II] defines a communications protocol and a standard connector to acquire data from passenger cars. However because of the age of the J1850 bus standard, it may not reside on newer late model vehicles which use CAN-BUS.

<p align="center">
{% asset_img j1850_signal.jpg Real J1850 PWM signal as measured in lab %}
<span>Real J1850 PWM signal as measured in lab</span>
</p>