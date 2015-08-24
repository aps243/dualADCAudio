rary, with support for Teensy 3.0, 3.1 and LC
Hello everybody,

I've have finally updated the ADC library and now it supports the Teensy 3.1 two ADC. https://github.com/pedvide/ADC
The code now has grown to about 1600 lines of actual code, and I think it's a good design to implement boards with even more ADCs.

The library supports Teensy 3.0, 3.1 and LC.

The main class is ADC.
Code:
ADC *adc = new ADC(); // adc object
This object controls the ADCs. For example, to measure the voltage at pin A9, you can do
Code:
int value = adc->analogRead(A9);
just as with the old version of this library. If you have a Teensy 3.0 it will return the value measured by the only ADC module.
If you have a Teensy 3.1, it will look whether this pin can be accessed by ADC0, ADC1 or both. If only one ADC module can access it, it will use it. If both can do it, it will assign this measurement to the ADC with less workload (for example if one is doing a continuous measurement already).
If you have a preference for one module in particular you can do:
Code:
int value = adc->analogRead(A9, ADC_0);
If the pin can't be accessed by the ADC you selected it will return ADC_ERROR_VALUE.

It's also possible to access directly the adc modules with
Code:
adc->adc0->function
adc->adc1->function

Overview of modes of conversion

All modes of conversion of the ADC are implemented in the same fashion:
Using 1 ADC			Using both ADCs (Synchronous mode)		
Single-shot mode	Single-shot mode	Continuous mode	Single-shot mode	Single-shot mode	Continuous mode
return value	return immediately		return value	return immediately	
analogRead(...)	startSingleRead(...)	startContinuous(...)	analogSyncRead(...)	startSynchronizedSingleRead(...)	startSynchronizedContinuous(...)
analogReadDifferential(...)	startSingleDifferential(...)	startContinuousDifferential(...)	analogSyncReadDifferential(...)	startSynchronizedSingleDifferential(...)	startSynchronizedContinuousDifferential(...)
readSingle()	analogReadContinuous()		readSynchronizedSingle(...)	readSynchronizedContinuous(...)
stopContinuous()			stopSynchronizedContinuous()
**PGA can be activated with enablePGA(gain), where gain=1, 2, 4, 8, 16, 32 or 64. It will amplify signals of less that 1.2V only in differential mode (see examples). Only for Teensy 3.1.

It's possible (and easy) to use the IntervalTimer library with this library, see examples folder.


Pins

There are only a few pins that can be accessed by both ADCs: 16 (A2), 17 (A3), 34-36 (A10-A13):
Click image for larger version. 

Name:	Teensy3_0_AnalogCard.png 
Views:	1134 
Size:	461.5 KB 
ID:	1792
Click image for larger version. 

Name:	Teensy3_1_AnalogCard.png 
Views:	3368 
Size:	539.2 KB 
ID:	1793


Synchronous measurements

Code:
ADC::Sync_result result = adc->analogSyncRead(pin1, pin2);
It will set up both ADCs and measure pin1 with ADC0 and pin2 with ADC1. The result is stored in the struct ADC::Sync_result, with members .result_adc0 and .result_adc1 so that you can get both. ADC0 has to be able to access pin1 (same for pin2 and ADC1), to find out see in the image that next to the pin number is written ADC0_xxxx or ADC1_xxx (or both).

I've made a simple test with a wavefunction generator measuring a sine wave of 1 Hz and 2 V amplitude with both ADCs at the same time.
I've measured it synchronously in the loop(){} with a delay of 50ms (so 20Hz sampling), the results are here:
Click image for larger version. 

Name:	SynchronousADC.jpg 
Views:	993 
Size:	105.1 KB 
ID:	1794
In red and black are the measurements on each pin, as you see, the difference is very small, so the system seems to work.

Voltage reference

The measurements are done comparing to a reference of known voltage. There are three options (two for Teensy LC):
ADC_REF_3V3: All boards have the 3.3 V output of the regulator (VOUT33 in the schematics). If VIN is too low, or you are powering the Teensy directly to the 3.3V line, this value can be lower. In order to know the value of the final measurement you need to know the value of the reference in some other way. This is the default option.
ADC_REF_EXT: The second option available to all boards is the external reference, the pin AREF. Simply connect this pin to a known voltage and it will be used as reference.
ADC_REF_1V2: Teensy 3.0 and 3.1 have an internal 1.2V reference. Use it when voltages are lower than 1.2V or when using the PGA in Teensy 3.1.

To change the reference:
Code:
adc->setReference(option, ADC_x);
where option is ADC_REF_3V3, ADC_REF_EXT or ADC_REF_1V2.

Sampling and conversion speed

The measurement of a voltage takes place in two steps:
Sampling: Load an internal capacitor with the voltage you want to measure. The longer you let this capacitor be charged, the closest it will resemble the voltage.
Conversion: Convert that voltage into a digital representation that is as close as possible to the selected resolution.


You can select the speed at which the sampling step takes place. Usually you can increase the speed if what you measure has a low impedance. However, if the impedance is high you should decrease the speed.
Code:
adc->setSamplingSpeed(speed); // change the sampling speed
speed can be ADC_VERY_LOW_SPEED, ADC_LOW_SPEED, ADC_MED_SPEED, ADC_HIGH_SPEED or ADC_VERY_HIGH_SPEED.
ADC_VERY_LOW_SPEED is the lowest possible sampling speed (+24 ADCK). (ADCK is the ADC clock speed, see below).
ADC_LOW_SPEED adds +16 ADCK.
ADC_MED_SPEED adds +10 ADCK.
ADC_HIGH_SPEED adds +6 ADCK.
ADC_VERY_HIGH_SPEED is the highest possible sampling speed (0 ADCK added).


The conversion speed can also be changed, and depends on the bus speed:
Code:
adc->setConversionSpeed(speed); // change the conversion speed
speed can be ADC_VERY_LOW_SPEED, ADC_LOW_SPEED, ADC_MED_SPEED, ADC_HIGH_SPEED_16BITS, ADC_HIGH_SPEED or ADC_VERY_HIGH_SPEED.
This will change the ADC clock, ADCK. And affects all stages in the measurement. You can check what the actual frequency for the current bus speed is in the header ADC_Module.h.
ADC_VERY_LOW_SPEED is guaranteed to be the lowest possible speed within specs for resolutions less than 16 bits (higher than 1 MHz), it's different from ADC_LOW_SPEED only for 24, 4 or 2 MHz.
ADC_LOW_SPEED is guaranteed to be the lowest possible speed within specs for all resolutions (higher than 2 MHz).
ADC_MED_SPEED is always >= ADC_LOW_SPEED and <= ADC_HIGH_SPEED.
ADC_HIGH_SPEED_16BITS is guaranteed to be the highest possible speed within specs for all resolutions (lower or eq than 12 MHz).
ADC_HIGH_SPEED is guaranteed to be the highest possible speed within specs for resolutions less than 16 bits (lower or eq than 18 MHz).
ADC_VERY_HIGH_SPEED may be out of specs, it's different from ADC_HIGH_SPEED only for 48, 40 or 24 MHz.

There's another option for the ADC clock, the asynchronous ADC clock, ADACK. It's independent on the bus frequency and has 4 settings: ADC_ADACK_2_4, ADC_ADACK_4_0, ADC_ADACK_5_2 and ADC_ADACK_6_2. The numbers indicate the frequency of the clock in MHz (so the first is 2.4 MHz).

Both the sampling and the conversion speed affect the time it takes to measure, there are some tests in the continuous conversion examples.

Library size
The library uses both program and RAM space. The exact amount depends on the Teensy used and whether optimizations were used.
The results in bytes are the respective progmem or ram of the analogRead.ino example minus that of the blinky example, the percentages are the memory used out of the total:

Teensy 3.0, 96 MHz, Serial: 5744 B (11%) program storage space and 132 B (14%) dynamic memory space used.
Teensy 3.1, 96 MHz, Serial, no optimizations: 6176 B (6%) program storage space and 136 B (3%) dynamic memory space used.
Teensy 3.1, 96 MHz, Serial, with optimizations (default): 9244 B (8%) program storage space and 1116 B (7%) dynamic memory space used.
Teensy LC, 48 MHz, Serial, no optimizations (default): 11120 B (33%) program storage space and 128 B (28%) dynamic memory space used.
Teensy LC, 48 MHz, Serial, with optimizations: 14212 B (41%) program storage space and 1112 B (53%) dynamic memory space used.

The progmem usage for Teensy LC is higher due to the use of floats (multiplying by 3.3). If possible, don't use floats at all in Teensy LC as they use about 6 kB of space (2 kB for Teenst 3.x).

Updates
Update: Now it works with the Audio library:
Teensy 3.0 is compatible, except if you try to use the Audio library ADC input (AudioInputAnalog), because there's only one ADC module so you can't use it for two things.
Teensy 3.1 is compatible. If you want to use AudioInputAnalog and the ADC you have to use the ADC_1 module, the Audio library uses ADC_0. Any calls to functions that use ADC_0 will most likely crash the program. Note: make sure that you're using pins that ADC_1 can read (see pictures above)! Otherwise the library will try to use ADC_0 and it won't work!!

Update: ADC library now supports Teensy LC!! Finally!

Update: RingBufferDMA works correctly with Teensy 3.x, not LC yet

Update: PDB is supported on both ADCs

Still TODO:

Each ADC_Module has a fail_flag with information relative to errors, see the examples and the defines in the header ADC_Module.h, but it's still under development, so big changes are still possible.
Last edited by Pedvide; 06-14-2015 at 07:39 PM. Reason: updates

Teensy Audio Library
====================

16 bit, 44.1 kHz streaming audio library for Teensy 3.x, featuring:

* Polyphonic Playback
* Recording
* Synthesis
* Analysis
* Effects
* Filtering
* Mixing
* Multiple Simultaneous Inputs & Outputs
* Flexible signal routing between library objects
* Automatic Streaming while your Arduino sketch runs

Main Audio Library Page
-----------------------

http://www.pjrc.com/teensy/td_libs_Audio.html


Audio System Design Tool
------------------------

Use this graphical tool to design your audio project.  Easily browse the library's many features, connect objects, export to Arduino code, and quickly access details for the functions each object provides for you to control it from your Arduino sketch!

http://www.pjrc.com/teensy/gui/


Supported Hardware
------------------

[Audio Adaptor Board](http://www.pjrc.com/store/teensy3_audio.html) for 16 bit stereo input and output.

![Inputs](/gui/audioshield_inputs.jpg)      ![Outputs](/gui/audioshield_outputs.jpg)

[Teensy 3.1](http://www.pjrc.com/store/teensy31.html) 12 bit DAC

![DAC Output](/gui/dacpin.jpg)

[Teensy 3.1](http://www.pjrc.com/store/teensy31.html) or [3.0](http://www.pjrc.com/store/teensy3.html) ADC Input

![ADC Input](/gui/adccircuit.png)

[Teensy 3.1](http://www.pjrc.com/store/teensy31.html) or [3.0](http://www.pjrc.com/store/teensy3.html) PWM Output

![PWM Output](/gui/pwmdualcircuit.jpg)





