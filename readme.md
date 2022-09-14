# Specmon 2
Specmon 2 is a rewrite of the original Specmon, version 1.
The goal of this project is to enable **Spec**trum **Mon**itoring over a long period of time while logging short-term activities.
Using this combined logging over time it is possible to generate statistical usage over time. 
When plotting these results, usage statistics and usage patterns becomes visible.

## Requirements
Specmon is based on SDR techniques. A suitable Software Defined Radio receiver is required to run the software. As Specmon is written in GNURadio, all the supported SDR's are usable with this software.
Besides the SDR, an antenna for the spectrum under investigation is required.
As most SDR's lack good frontend-filtering, bandfiltering is required to maintain reliable results.

### Dependencies
Specmon is build upon GNURadio, Python and Gnuplot.
For Debian-based systems the dependencies can be installed with:

`apt install gnuradio gnuplot gr-osmosdr python-numpy python3-numpy`

For support for other SDR's, install the required modules. For example for PlutoSDR: 

`apt install libiio0 libad9361-0 gr-iio`

When using another SDR than the supported radio's under OsmoSDR, modify the flowgraph accordingly.

## Preparation
Several steps are required to run specmon2:

1. Define the frequency band to monitor. For example: 2M HAM band, 144-146 MHz
2. Define the required sample-rate. In this case 2Msps.
3. Select the minimum frequency-size to trigger on: In this case FT8 as smallband mode needs to trigger the measurement. Using an FFT-size of 2048 results in a minimum monitoring of approx 1KHz (976Hz)
4. Calibrate the receiver and create a calibration vector for the measurement. The process of calibration is described seperately.
5. Determine the threshold for the measurement using live monitoring.
6. Setup the monitoring

The gain settings of your SDR are highly dependent on the local environment and the antenna. Please use a low to moderate gain setting and use bandfilters to prevent overloading or images by out of band signals.

## Calibration
Calibration is the process of alignment of the spectrum to the same value: That is, correcting the vector of FFT's to the same baseline. This generates a "flat spectrum". With this flat spectrum a single threshold value can be used during the monitoring process.

Steps to take (in this order):

### Zero vector
Create a "zero" vector with the correct FFT-size and a constant value of zero: 

`./calvec -o zero-calvec-2048.vec -l 2048 -c 0`

This generates a file named zero-calvec-2048.vec with a "zero calibration" vector to use in the process.

### Dummy load measurement
Connect the SDR to a dummyload to run this step. This is required as this corrects the spectrum.
Run specmon2 for a minute with the required settings and record the vectors. These vectors are used to create the final calibration-vector:

`./specmon2.py -g 20 -f 145e6 -s 2e6 -c zero-calvec-2048.vec -F 2048 -d rtl=0 -R 60`

Setting the threshold is not required yet.

### Create the calibration vector
From the output, written in /tmp on the filesystem, generate the calibration vector:

`./calvec -i /tmp/210512-0641-145000000-2000000-2048.pvector -o 210521-145e6-2e6-2048-2.vec -l 2048 -c 2`

This generates a calibration-vector:

- With length 2048
- With a constant 2 added to the values (lifting the spectrum)
- Created on May 21th 2021 06:41
- For frequency 145MHz
- And samplerate 2Msps

This calibration-vector is used for the monitoring process

### Determine the threshold
The threshold is the powerlevel value which a transmission should reach before it is considered as a transmission. According to the ITU it should be 20dB above the noise level however this is a high limit for relatie low power transmissions or weak transmissions, caused by a higher free-space path loss. These latter transmissions are not recorded as transmissions with a high threshold. Using a low threshold on the other hand will cause false positives due to short term impulses, caused by RFI or noisefloor variances.
The threshold can be controlled either afterwards using a replay from previous monitoring or by inspecting the measurements using specmon_gui.

#### Auto Threshold calculation
Specmon2 is capable of adjusting the threshold value, based on the variances in the measured noise. The value passed to the parameter "threshold" is considered as the value above the measured noise level. During the monitoring process the actual threshold is adopted and printed in the raw measurement files.

## Monitoring
The monitoring is setup via cron using the tool `monit2`.
Edit monit2 with the required settings:

````
# Put to 1 to make this run
ENABLED=1

# Put to 1 to automatic clean old recordings in /tmp
# Files older than 2 days are removed
AUTOCLEAN=1

THRESHOLD=8.0
GAIN=20
FREQ=145e6
RUNTIME=3590
DEVICE="rtl=0"
SAMPRATE=2e6
CALVEC=/home/pi/specmon/210504-calvec-145M-2e6-2048-0.vec
FFTSIZE=2048

# Use the correct tool to run
SPECMON=/home/pi/specmon/specmon2.py

LOGDIR=/home/pi/log/145
````
Please be aware of the runtime. It defaults to 3590 seconds instead of 3600 seconds. Starting and stopping the measurement takes some seconds, as the SDR needs to be initialized. Running the tool with 3600 results in running just over an hour and preventing the next hour to start the measurement.


Run monit via cron:

`0 * * * *		/home/pi/specmon/monit2`

## Postprocessing
Postprocessing the logfiles can be automated with `parselog`.
Check the settings in this file and run via cron:

`0 1 * * *		/home/pi/specmon/parselog`

This needs to run only once a day and will generate the graphs for the previous day:

1. `YYMMDD.plot-avgopen.png` : Average time in seconds above the threshold.
2. `YYMMDD.plot-maxopen.png` : Maximum measured value above the threshold.
3. `YYMMDD.plot-opening.png` : Number of times the threshold is triggered.
4. `YYMMDD.plot-percent.png` : Percentage of time the threshold is triggered.

### Weekly or monthly graphs
Weekly or monthly graphs can manually be made by combine the daily logs to one large file:

`cat 2105*.plot >> May2021.plot`
And create a monthly graph with mgraph: 

`./mgraph May2021.plot`

## Specmon GUI
Specmon2 can run graphical with `specmon2_gui`. This does not log all the output but the spectrum and the vectors of the flattend spectrum are displayed, together with the threshold. 
Specmon2_gui can be used to inspect the spectrum and select the right constant for the calibration-vector.

## Live view of data
During the monitoring process, it is possible to connect to the ZMQ port for live view of the data, that are the vectors to be processed by specmon, remotely.
For this a tool is added, specreceive, which can connect to the ZMQ port 50001, on the system, running specmon.
As this only sends the integrated vectors, the amount is data is fairly low and can be send over a VPN or remote connection.

## Replay of recorded data
The unprocessed vectors from the measurements are stored in /tmp. The vectors are not corrected yet with the calibration vector and thus named "unprocessed". These files can be graphical replayed, together with the calibration-vector to review the state while recorded. This can show RFI or other unknown transmissions, or can be used to modify the threshold value.

## Tools overview
The following tools are included in this toolkit:

### specmon2
The commandline tool for long term monitoring
  
#### Options

````
usage: specmon2.py [-h] [-b BBGAIN] [-c CALIBVEC] [-d DEVICEID] [-F FFTSIZE] [-f FREQ] [-O OUTFILE] [-g RFGAIN] [-R RUNTIME]
                   [-s SAMP_RATE] [-T THRESHOLD] [-v VERBOSE]

optional arguments:
  -h, --help            show this help message and exit
  -b BBGAIN, --bbgain BBGAIN
                        Set BB-Gain [default=30]
  -c CALIBVEC, --calibvec CALIBVEC
                        Set Calibration Vector [default='0']
  -d DEVICEID, --deviceid DEVICEID
                        Set Device ID [default='rtl=00000201']
  -F FFTSIZE, --fftsize FFTSIZE
                        Set FFT size [default=1024]
  -f FREQ, --freq FREQ  Set Frequency [default='145.0M']
  -O OUTFILE, --outfile OUTFILE
                        Set Output File [default='/tmp/logfile.csv']
  -g RFGAIN, --rfgain RFGAIN
                        Set RF-Gain [default=40]
  -R RUNTIME, --runtime RUNTIME
                        Set runtime [default=3590]
  -s SAMP_RATE, --samp-rate SAMP_RATE
                        Set Sample Rate [default='2.0M']
  -T THRESHOLD, --threshold THRESHOLD
                        Set Threshold [default='100.0m']
  -v VERBOSE, --verbose VERBOSE
                        Set Print verbose values [default=0]
````

### Specmon2_gui
Specmon2 but with graphical display. To be used for manual monitoring and tuning of parameters

#### Options

````
usage: specmon2_gui.py [-h] [-b BBGAIN] [-c CALIBVEC] [-d DEVICEID] [-F FFTSIZE] [-O OUTFILE] [-R RUNTIME] [-s SAMP_RATE]
                       [-v VERBOSE]

optional arguments:
  -h, --help            show this help message and exit
  -b BBGAIN, --bbgain BBGAIN
                        Set BB-Gain [default=30]
  -c CALIBVEC, --calibvec CALIBVEC
                        Set Calibration Vector [default='/home/bart/grc/210926-zero-2048.vec']
  -d DEVICEID, --deviceid DEVICEID
                        Set Device ID [default='rtl=0']
  -F FFTSIZE, --fftsize FFTSIZE
                        Set FFT size [default=1024]
  -O OUTFILE, --outfile OUTFILE
                        Set Output File [default='/tmp/logfile.csv']
  -R RUNTIME, --runtime RUNTIME
                        Set runtime [default=3590]
  -s SAMP_RATE, --samp-rate SAMP_RATE
                        Set Sample Rate [default='2.0M']
  -v VERBOSE, --verbose VERBOSE
                        Set Print verbose values [default=0]
````

### calvec
Tool to create calibration vector, based on real measurement-data or a constant vector.

#### Options

````
usage: calvec [-h] [-i INFILE] -o OUTFILE -c CONSTANT -l LENGTH

optional arguments:
  -h, --help            show this help message and exit
  -i INFILE, --infile INFILE
                        Filename with vectors
  -o OUTFILE, --outfile OUTFILE
                        Filename where to put output
  -c CONSTANT, --constant CONSTANT
                        Constant to use for calculation
  -l LENGTH, --length LENGTH
                        Vector-length to parse

````

### mkgraph
This tool is used to create the graphs from the parsed data.
Usage: `./mkgraph <inputfile>`

### monit2
Shellscript to automate monitoring and cleanup from old loggings. The usage of monit2 is described above

### mgraph
Shellscript to generate the monthly-graph from the concatenated logfiles.
Usage: `./mgraph <concatenated_logfile>`

### parselog
Shellscript to automate the parsing of logfiles and creation of the graphs. To be run from cron so no options required.
See the shellscript for the correct paths.

### testspec
This is the parser for the logfiles from specmon2. It is called from parselog but can be used manually too.

#### Options

````
usage: calvec [-h] [-i INFILE] -o OUTFILE -c CONSTANT -l LENGTH

optional arguments:
  -h, --help            show this help message and exit
  -i INFILE, --infile INFILE
                        Filename with vectors
  -o OUTFILE, --outfile OUTFILE
                        Filename where to put output
  -c CONSTANT, --constant CONSTANT
                        Constant to use for calculation
  -l LENGTH, --length LENGTH
                        Vector-length to parse

````

### specreceive
This is the ZMQ receiver for live remote view.
Specreceive does not have commandline options but the flowgraph holds multiple variables which need to be set before running the flowgrapgh.
The options required to correct are:

- FFT size
- Frequency
- Sample Rate
- Hostname or IP to connect to, in the ZMQ source block.
	+ This should be in the format: `tcp://<hostname or ip>:50001`

Specreceive also shows a line representing the threshold, which can be set manually using the slider.

## Usage suggestions
Below are some suggestions to use the software. These suggestions are focused on the HAM-radio bands but probably other bands can monitored too. 

### 2M HAM band

 - Frequency : 144 - 146 MHz
 - Tuner frequency: 145e6
 - Sample rate : 2e6
 - FFT-size: 2048 / 4096
 
### 6M HAM band

 - Frequency : 50 - 52 MHz
 - Tuner frequency : 51e6
 - Sample rate : 2e6
 - FFT-size : 2048 / 4096
 
### 70cm HAM band
This band is 10MHz wide. The sample-rate must be supported by your SDR. A LimeSDR mini or Airspy R2 support this. This combination of sample rate and FFT-size is likely too much for a Raspberry Pi.

 - Frequency : 430 - 440 MHz
 - Tuner frequency : 435e6
 - Sample rate : 10e6
 - FFT-size : 16384
 
### EU 868MHz ISM band
This band is wider than the suggested 2.5MHz but it holds the majority of the LoRa frequencies. Using this settings, the IOT usage can be monitored. As the LoRa transmissions are at least 125KHz wide, the FFT-size is sufficient for this monitoring.

- Tuner frequency : 868.35MHz
- Sample rate : 2.5e6
- FFT-size : 2048
 
