# Noisemon
Noisemon is a rewrite of specmon, with the goal to measure noiselevels.
The definition of noisemeasurements is stated in ITU documents, ITU-P SM.372-16, ITU-R SM.2055 and specific SM.1753-2.
Based upon the above mentioned documents, multiple types of noise are identified:

- WGN, or White Gaussian Noise
- SCN, or Single Carrier Noise
- IN, or Impulse Noise

Besides identification of noise-types, normal usage of the frequency band is identified, based upon the assumption of minimum bandwidth for transmissions, as described in the ITU documents.

The use of noisemon is to measure and display several types of noise over a long period of time. Measurements are taken every 200ms, or five times per second.

## Preparation
The preparation for noisemon measurements is equal to setting up measurements with Specmon.

### RBW
According to the ITU documentation for noise measurements, the RBW should be 500Hz or less. Using this RBW, Single Carrier Noise can be identified, based on the FFT's active on the detected signal.
Using 2MSps and thus monitoring 2MHz of bandwidth, this requires an FFT-size of 4096. This yields to an RBW of 488Hz.
When very smallband transmissions are expected, the FFT-size must be increased.

### Filtering
When insufficient filtering is applied, spurious signals can be seen in the spectrum. This results in false measurements of SCN. Strong nearby signals can increase the overall noisefloor. If the selected SDR features analogue low-pass filters, it is advised to apply them. Not all SDR's have them but the PlutoSDR and LimeSDR Mini are two SDR with this features. 


### Determine the threshold
The threshold is the powerlevel value which a transmission should reach before it is considered as a transmission. According to the ITU it should be 20dB above the noise level however this is a high limit for relative low power transmissions or weak transmissions, caused by a higher free-space path loss. These latter transmissions are not recorded as transmissions with a high threshold. Using a low threshold on the other hand will cause false positives due to short term impulses, caused by RFI or noisefloor variances.
The threshold can be controlled either afterwards using a replay from previous monitoring or by inspecting the measurements using specreceive.
The default dynamic threshold is set to 8dB above the median noise level.

### SCN Limit
An experimental option is available in noisemon. Detecting Single Carrier Noise is achieved by measuring the amount of adjacent FFT-bins above the threshold. SCN should only activate one FFT-bin but due to FFT-leakage, for example caused by the carrier not being in the middle of the FFT-bin, more FFT-bins can be activated. Some experiments show that 3 FFT's is a reasonable value, if the FFT-size is 500Hz or less. This 500Hz limit is defined by the ITU. 

## Monitoring
The monitoring is setup via cron using the tool `nmonit`.
Edit nmonit with the required settings:

````
# Put to 1 to make this run
ENABLED=1

# Put to 1 to automatic clean old recordings in /tmp
# Files older than 2 days are removed
AUTOCLEAN=1

GAIN=40
FREQ=145e6
RUNTIME=3580
DEVICE="airspy=0"
SAMPRATE=2e6
#CALVEC=/home/user/noisemon/220911-143M-G40-S2e6-4096-0.calvec
CALVEC=/home/user/noisemon/221014-145M-G40-S2e6-4096-0.calvec
FFTSIZE=4096
SCNLIMIT=3

# Use the correct tool to run
NOISEMON=/home/user/noisemon/noisemon.py

# Define the logdir
LOGDIR=/home/user/log/145
````


Please be aware of the runtime. It defaults to 3590 seconds instead of 3600 seconds. Starting and stopping the measurement takes some seconds, as the SDR needs to be initialized. Running the tool with 3600 results in running just over an hour and preventing the next hour to start the measurement.


Run monit via cron:

`0 * * * *              /home/user/specmon/nmonit`

## Postprocessing
Postprocessing on the logfiles can be executed manual or via cron.

### Noisegraph
Manual creation of the graphs can be done using:

````
for i in noisemon-YYMMDD-{00..23}00.csv; do ./noisegraph ; done
````

This creates multiple graphs from the measurements:

1. `noisemon-YYMMDD-HHMM.csv-combined.png`	Combined graph with noiselevel, Standard deviation of the noise, difference between mean and median and skewness of the noise.
2. `noisemon-YYMMDD-HHMM.csv-diff.png`		Graph with the difference between the median and the mean of the measured noise.
3. `noisemon-YYMMDD-HHMM.csv-noise.png`		Graph with the measured noise.
4. `noisemon-YYMMDD-HHMM.csv-skew.png`		Graph with the skewness of the measured noise.
5. `noisemon-YYMMDD-HHMM.csv-std.png`		Graph with the standard deviation of the measured noise.
6. `noisemon-YYMMDD-HHMM.csv-use-scn.png`	Graph with the amount of normal usage and Single Carrier Noise, as calculated.


### Dayboxer
Using the script `dayboxer` a graph is made with boxplots of the measured noise. This graph hold 24 boxplots and shows the spreading of the noise over the day.
Dayboxer default print graphs from "yesterday" but other dates can be processed:

`./dayboxer YYMMDD`

Dayboxer default writes the graphs to the directory `tst/` in the current directory. This can be modified in the script.

## Data format
The logfiles from noisemon are written to the log directory as configured in the nmonit wrapper.
Data in this files are stored as csv, comma separated values. The format of the file is as follows:

````
Start,1667473204.96,Hour,12,Veclen,4096,Threshold,8.00,SCNLimit,3
N,time,lowlimit,veccount,scncount,nusage,vecmean,vecstd,vecskew,rms,inoise,diff
N,1667473207.47,8.56,1,8,4,8.48,0.44,-1.51,8.49,0,0.12
N,1667473207.64,8.66,2,8,4,8.58,0.40,-1.10,8.59,0,0.12
````

The first line is the Start line. This writes:

1. The starting time in unix timestamp
2. Hour: The hour in which the measurement is started
3. Veclen: Used FFT-size (length of the vector)
4. Threshold: Threshold value above the noise. Adopted when the noiselevel changes
5. SCNLimit: Amount of FFT-bin's which can count as SCN. More than this is counted as normal usage

The second line is the dataformat of the measured values. All values are calculated on a subset of the entire vector, named noisevector: 

1. N: Noisemon
2. time: Time of the measurement in Unix timestamp
3. lowlimit: The median of the noisevector. The threshold is determined on this value
4. veccount: Count of measurements in this run
5. scncount: Counted SCN spikes
6. nusage: Counted "Normal usage" (wider than SCN)
7. vecmean: Mean of the noisevector.
8. vecstd: Standard Deviation in the noisevector.
9. vecskew: Skewness of the noisevector.
10. rms: RMS value of the noisevector.
11. inoise: Value for the (experimental) detection of Impulse Noise. 1 for INoise
12. diff: Absolute difference between the median(lowlimit) and mean(vecmean)

