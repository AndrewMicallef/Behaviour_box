This is the collection of files I use to run my behavioural experiments.

Behaviour Box
=============

This repository contains a set of files to implement a Go-No-Go
task using an Arduino and Python 2.7. By itself this collects some summary
information for a behavioural trial, but is not a stand alone implementation.
In addition to what is recorded here, I record the signal direct from the 
lick sensor using  [Wavesurfer](https://github.com/JaneliaSciComp/Wavesurfer).

The two components to this implementation are the Arduino code, found in
`./behaviourbox/behaviourbox.ino` and a Python wrapper for communicating
with this code `./SerialControl.py`.

------------------------------------------------------------------------------

System overview
---------------

The Arduino repeatedly monitors the signals from 
the lick sensor and controls the timing of water and stimulus delivery. 

The Arduino triggers external devices.


### Hardware

The basic hardware requirements

1. a microcontroller
    * [Arduino uno Rev3](https://www.arduino.cc/en/Main/ArduinoBoardUno)

        Connects to the main computer via USB serial

2. a delivery system for water reward, 
    * The reward delivery is controlled by a solenoid pinch valve 
        ([24 V DC PS-1615NC](http://www.takasago-fluidics.com/p/valve/s/pinch/PS/), 
        Takasago Fluidic SysteSms, Nagoya, Japan).
       
         The pinch valve is wired to a gated 24 V power supply, which accepts 
         5 V logic to switch on and off. 
         
        - I use a 24 V pinch valve. This is in many ways over kill, a 3 V valve
          would be much better, because it would not require an additional gated
          power supply.

3. a lick sensor
    * lick port

        I use peizo electric wafers, specifically a [0.6mm Range Piezo 
        Bender Actuator](http://www.piezodriveonline.com/0-6mm-range-piezo-bender-actuator-ba3502/)
        from PiezoDrive Pty Ltd, (Callaghn NSW)

        - The signal from these are very small. The piezos require a linear
          amplifier so that the arduino can detect the signal they produce.
          I use custom made linear amplifiers that each take 5V DC input
          and output in a range from 0-3 V. The amplifiers come are from Larkum
          lab designs.

        - ([LM358N](http://www.ti.com/product/LM358-N), Texas Instruments)
          This signal is then amplified using a simple operational amplifier 
          circuit to ensure the Ardunio microcontroller detects the lick-evoked 
          voltage changes.

4. a sensory stimulus. 
    * The stimulus needs to take 5 V digital signal.

5. A speaker
   * I have wired my speaker up directly to the Arduino. Ideally you should wire the
   speaker through a resistor to protect the Arduino.

6. Recording devices


### Software





Version 3.0.20170201.8
----------------------

Major changes in the last update.
The behaviour box program is now split into a handful of modules. 
Each module is mostly holding utility functions for the box as a whole.
The behaviour box itself only contains `setup` and `loop` functions, which
call the other components as necessary.


--------------------------------------------------------------------------------

behaviourbox.ino
================

This program delivers sensory stimulation and opens water 
valve when the animal touches the licking sensor within a 
certain time window.

### Requirements

Setup connections:
------------------


Table: Analog connections to lick controller


Global Variables
----------------

| Variable Name | pin  | Physical connection                                            |
| ----          | ---- | -------------------------------------------------------------- |
| recTrig       | `3`  | short recording trigger (10ms TTL pulse )                      |
| bulbTrig      | `4`  | full trial duration signal (bulb style trigger)                |
| stimulusPin   | `5`  | Connect to Stimulus device, outputs a variable duration pulse  |
| buzzerPin     | `6`  | Connect to punishment device, (TTL trigger)                    |
| speakerPin    | `7`  | output to auditory cue speaker                                 |
| waterPort     | `10` | Connect to water valve power supply                            |
| lickSens      | `A0` | The Piezo Amplifier is connected to the analog input           |

Table: connections


Files
-----

The header files in the behaviourbox folder contain the functions required to
run the behaviourbox code. 
There are currently 7 header files:
1. "global_variables.h"
2. "prototypes.h"
3. "timing.h"
4. "sensors.h"
5. "states.h"
6. "SerialComms.h"
7. "single_port_setup.h"

### global_variables.h

contains definitions of all variables that are used by multiple functions
and expected to have persistent values between the functions. These include
the initialisation time, the various timing parameters, as well as any 
additional options that I have decided to make available.

| variable         | description           |
| ---------------- | --------------------- |
| t_init           | not settable. This is the variable that the other times a measured relative to |
| t_noLickPer      | A time measured in milliseconds. After this amount of time the program will break out of a trial if a lick is detected before the stimulus. If the value is 0 then licks before the stimulus are ignored altogether. |
| trial_delay      | Amount of time in milliseconds to delay the start of a trial |
| t_rewardDEL      | Amount of time in milliseconds to delay checking for licks after a stimulus |
| t_rewardDUR      | Amount of time in milliseconds to check for licks |
| t_trialDUR       | Total time in milliseconds that the trial should last |
| ---------------- | --------------------- |
| t_stimONSET      | The time in milliseconds, relative to the trial start time, that the stimulus turns on |
| t_stimDUR        | Amount of time in milliseconds to keep the stimulus on |
| ---------------- | --------------------- |
| timeout          | Boolean value. If True enable the recursive timeout punishment |
| ---------------- | --------------------- |
| debounce         | Number of milliseconds to delay between reading the lick sensor value. |
| lickThres        | The threshold on the licksensor required to call a lick. This is a number between 0 - 1024 (multiply by  5 V/1024 to get voltage) |
| ---------------- | --------------------- |
| minlickCount     | The number of licks required to count as a response, ie to open the water valve, or to deliver a punishment |
| lickTrigReward   | Boolean, set true to enable the reward to be delivered immediatly after the min licks are reached. If false the reward is deliverd at the end of the response duration |
| reward_nogo      | Boolean, set true if you want a correct rejection to be rewarded at the end on the reward period |
| ---------------- | --------------------- |
| mode             | {'-', 'O', 'H'} a character to represent the type of mode to run in. If the mode is 'H' the system will delver a stimulus and a reward in response to the animal's lick. If the mode is 'O' the system delivers a stimulus and listens for a response |
| ---------------- | --------------------- |
| reward_count     | deprecated            |
| ---------------- | --------------------- |
| waterVol         | Amount of time in milliseconds to hold the water valve open for |
| trialType        | character code to determine if this is a go or no go trial. This is used to determine if the water valve will open on a given trial |
| ---------------- | --------------------- |
| Nports           |  deprecated           |
| verbose          |  Boolean, if True will enable full debug printing...(might be deprecated?) |
| break_wrongChoice | Boolean, deprecated  |
| punish_tone      | deprecated            |
| audio            | Boolean, enables auditory cues for the response period |
| ---------------- | --------------------- |



--------------------------------------------------------------------------------


SerialController.py
===================

This python script is a wrapper for communicating with the Arduino program.
At it's heart is a simple loop that reads data transmitted through a serial
connection. This is not a necessary component, however I wrote it to make 
running trials a lot easier.

The script runs on the command line. The 

### Install

- get git [https://git-scm.com/docs/git-svn]
- Clone this repository
- Be an administrator
- Install Arduino IDE [https://www.arduino.cc/en/Main/Software]
- Install Arduino windows drivers (see Arduino website)
- Compile and upload `behaviour_box.ino`
- Install python 2.7 for windows
    - check add python to path
    - check install setup tools
- Download additional packages from http://www.lfd.uci.edu/~gohlke/pythonlibs/
    - Follow specific instructions for installing numpy+mkl. (This system 
      doesn't necessarily require the speed of the mkl version, but it is a 
      nice thing to have)
    - Use pip to install all packages from and elevated command prompt


### Requires

* [Windows 7](https://www.microsoft.com/en-au/software-download/windows7)
* [Pandas (0.19.0)](http://pandas.pydata.org)
* [Numpy (1.11.2+mkl)](http://www.numpy.org/)
* [Pyserial (2.7)](https://github.com/pyserial/pyserial)
* [Colorama (0.3.7)](https://pypi.python.org/pypi/colorama)
* [sounddevice (0.3.5)](http://python-sounddevice.readthedocs.io/en/0.3.5/)
* ConfigParser
* argparse
* An Arduino running `behaviourbox.ino` connected to the same computer

To install numpy follow the instructions at http://www.lfd.uci.edu/~gohlke/pythonlibs/
I used version 1.11.2+mkl for this, however the Chris Gohlke doesn't seem to
archive older versions. TODO: test upgrading numpy.

`pip install pandas==0.19.0 pyserial==2.7 Colorama==0.3.7 sounddevice==0.35`


call signature
```
usage: SerialControl.py [-h] [-af] [--trials [TRIALS [TRIALS ...]]]
                        [-lt LICKTHRES] [--verbose] [-restore]
                        [--repeats REPEATS] [--port PORT] [--ITI ITI ITI]
                        [-rdur T_RDUR] [-ltr] [--t_stimONSET T_STIMONSET]
                        [--datapath DATAPATH] [-i ID] [-m MODE] [-to TIMEOUT]
                        [-nlp NOLICK] [-w WEIGHT] [-td TRIALDUR]
                        [-rdel T_RDELAY] [-p] [-noise] [-rng] [-lc LCOUNT]
```

Optional Arguments:
-------------------

optional arguments:

#### -h, --help            

show this help message and exit

#### -af, --audio          

 provides audio feedback during the trials this is not
 to be confused with the noise played to simulate /
mask the scanners

#### --trials [TRIALS [TRIALS ...]]

durations to run on each trial

#### -lt LICKTHRES, --lickThres LICKTHRES

set `lickThres` in arduino

#### --verbose
 
 for debug this will print everything if enabled

#### -restore

Use to look up previous settings in the comms.ini file

#### --repeats REPEATS

the number of times this block should repeat, by
default this is 1

#### --port PORT

port that the Arduino is connected to

#### --ITI ITI ITI

an interval for randomising between trials

#### -rdur T_RDUR, --t_rDUR T_RDUR

set end time of reward epoch

#### -ltr, --lickTrigReward

flag to allow licks to trigger the reward immediatly

#### --t_stimONSET T_STIMONSET

sets the time after trigger to run the first stimulus

#### --datapath DATAPATH

path to save data to, by default is
R:\Andrew\161222_GOnoGO_Perception_III\%YY%MM%DD

#### -i ID, --ID ID

identifier for this animal/run

#### -m MODE, --mode MODE

the mode `h`abituaton or `o`perant, by default will
look in the config table

#### -to TIMEOUT, --timeout TIMEOUT

set the timeout duration for incorrect licks

#### -nlp NOLICK, --noLick NOLICK

set `t_noLickPer` in arduino

#### -w WEIGHT, --weight WEIGHT

weight of the animal in grams

#### -td TRIALDUR, --trialDur TRIALDUR

set minimum trial duration

#### -rdel T_RDELAY, --t_rDELAY T_RDELAY

set start time of reward epoch

#### -p, --punish

sets `break_wrongChoice` to True, incorrect licks will
end an operant trial early

#### -noise

plays a noise during trials

#### -rng, --reward_nogo   

flag to allow a water delivery following no lick of a no go stim

#### -lc LCOUNT, --lcount LCOUNT

set `minlickCount` in arduino



Interactive Options
-------------------

| key          | option                                           |
| ------------ | -------------------------------------------------|
|      H       | This menu                                        |
|      P       | Punish                                           |
|      S       | toggle single stimulus                           |
|      < >     | lick threshold                                   |
|      ?       | show threshold                                   |
|      [ ]     | lickcount                                        |
|      \\      | show lickcount                                   |
|      tab     | toggle mode                                      |
|      :  "    |  adjust noLick period                            |
|      L       | show noLick period                               |
|      ( )     | adjust trial duration                            |
|      T       | show trial duration period                       |
|      Y       | toggle timeout (requires punish to take effect)  |
|      B       | toggle bias correction                           |
| input `rdel` |                                                  |
| input `rdur` |                                                  |
| ------------ | -------------------------------------------------|



1. The program starts
2. The program opens communications with available serial port
    The program waits until it gets the arduino is active, and prints all output
    until the ready signal is transmitted. Which is `- Status: Ready`
    
3. The program starts a block
5. The program transmits the dict `params`, which holds all parameters 
    for a single trial. The condition values get updated; based on the
    frequencies being sent, all contents of `params` are transmitted to
    the behaviour controller.
    
6. The program prints the frequencies and the condition to the screen and a
   random timeout is started.
6. The program initiates a trial by sending a literal `"GO"` to the 
    behaviour box.
    - The behaviour box runs one trial, with the parameters set previously

7. The program records the output from behaviorCOMs into a
   dict, which later will be converted to a data frame for analysis.

8. The program repeats sending mode flags until all stimuli combinations have
   been run through.


plot_stats.py
==============

### Requires 

* [Bokeh](http://bokeh.pydata.org)
* [Pandas](http://pandas.pydata.org)
* [Numpy](http://www.numpy.org/)


References
==========
