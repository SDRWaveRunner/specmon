# SPECMON 

Specmon is a set of tools for statistical **spec**trum **mon**itoring. It's designed for long-term monitoring and reporting usage statistics for the selected frequency-range.
Specmon is not capable of recording transmissions in any way.


## The software
The main tool is written in GNUradio [[5]]. Postprocessing is done in Python and graphs are plotted in GNUplot. The platforms used are Debian 10 and Raspbian 10.
When running headless on a Raspberry Pi, a Raspberry Pi 3 or better is required.

### Dependencies

`apt install gnuradio gnuplot gr-osmosdr`<br>
This dependencies allows you to use [RTL-SDR] , [HackRF], and [BladeRF]. If you need support for your specific SDR, you need to install the dependencies too.<br> If you need support for [PlutoSDR], you need to install the next dependencies:
`apt install libiio0 libad9361-0 gr-iio`

The default palette for the graphs can be changed by loading an additional palette from [gnuplot-palettes]. The weekgraphs are made with 'reds.pal'

## Running the software

Running the software is quite easy but requires some preparations:

1. Choose the frequency to be monitored
2. Determine the required bandwidth, sample-rate and decimation
	- This defines the centre-frequency, or frequency to tune the SDR to.
3. Determine the threshold
4. Select the time to run. Default is 3590 seconds.

The default runtime is 10 seconds less than one hour. This is to prevent the software is starting up again while just finishing the last measurement. 

### Example 

We want to monitor the European PMR (Public Mobile Radio) band [[4]]. The defined frequency-range is 446.0 - 446.2 MHz so 200KHz bandwith.
The centre-frequency is 446.1MHz. You need to tune the SDR to this frequency.
The required sample-rate should be 200.000 samples per second (200ksps). With the decimation-option this samplerate is created: Sample-rate 2Msps and a decimation 10 results in exactly 200.000 samples per second.

I advice you to first run the GUI version to visual inspect the selected spectrum and to determine the threshold. From within Gnuradio-Companion, run specmon.grc, select the frequency (446.1e6), sample-rate (2e6), decimation (10) and run the flowchart.

All depending on your SDR, gain, antenna and environment you can see the threshold drop when the frequency-band is idle. Take note of this value and increase it a little bit. For example, if it's idling around 0.75, increase to about 0.9 to prevent false-positives but prevent missing-out some band-usage. 

### Running the software headless

Specmon is designed to run one hour and restart for a new logfile. The previous logfile can be postprocessed and converted to a graph.

"Compile" the flowgraph for headless monitoring to the python-file:
`user@system:~$ grcc -d . specmon-cli.grc`

On your headless system run the software:

`./specmon-cli.py -f 446.1e6 -s 2e6 -D 10 -g 30 -T 0.8`
And you should see lines like this:

`A,1.1,1576821964.22,U,0.8,1576821964.78`

- A is *Above*,
- 1.1 is the average low-value since the last measurement,
- 1576821964.22 is the timestamp,
- U is *Under* the threshold,
- 2.8 is the maximum value reached during this trigger,
- 1576821964.78 is the timestamp when dropped below the threshold. 


If you are happy with the result, start running it from cron and let it run for several days:

````
crontab -l
# m h  dom mon dow   command
0 * * * *      /home/pi/bin/monit
````
The script monit is provided in the source-tree. This runs the tool for an hour and creates a logfile per hour.

### Postprocessing the data
After running for 24 hours you can make a graph for the usage-statistics. Therefor you need to postprocess the data using "testspec":

````
for i in `ls log/specmon2-200110*csv`; do ./testspec $i >> log/200110.log ; done
````
And now plot the graph:
`./mkgraph 200110.log`

View the graph in your viewer. After the first week you can see patterns of usage over the week. After several weeks you start seeing patterns per day. First combine a week of data:
`cat day1.log day2.log day3.log day4.log day5.log day6.log day7.log >> week.log`
and plot a 3D graph for the week:
`./weekgraph week.log`

---
## Some suggestions for use
### Frequency suggenstions
**EU 70cm analog repeaters:** 

- Downlink: 430.000 - 430.400 MHz
- Center-frequency: 430.200 MHz
- Sample-rate: 2e6
- Decimation: 5

**EU 70cm DMR repeater:**

- Downlink: 438.000 - 438.400 MHz
- Center-frequency: 438.200 MHz
- Sample-rate: 2e6
- Decimation: 5

**EU 2M analog repeaters:**

- Downlink: 145.600 - 145.800 MHz
- Center-frequency: 145.700 MHz
- Sample-rate: 2e6
- Decimation: 10

**EU PMR446 [[4]]:**

- Frequency span: 446.000 - 446.200 MHz
- Center-frequency: 446.100 MHz
- Sample-rate: 2e6
- Decimation: 10

### Verbose mode
Determining the threshold can also be achieved in headless mode. Run specmon_cli.py with the `-v 1` option.
See `specmon_cli.py -h` for all the options.

### GNUradio 3.7 vs 3.8
Although GNUradio 3.8 has been released for a while, i'm still using 3.7 branch. Why, would you ask. Pretty easy: Installing GNUradio on a raspberry pi for the headless mode is quite easy as it's in the repository. But it's the maintained 3.7 branch.

### Other SDR's
You can select other SDR's to use. Currently specmon is build around the Osmocom-Source so every SDR supported in this block, can be used:
`-d rtl=00000234` for RTL-dongle with serial 234<br>
`-d bladerf=0` for the first bladerf connected<br>
`-d hackrf=0` for the first hackrf connected<br>

For other SDR's the source-block in the Gnuradio Flowgraph needs to be exchanged.

## Help

````
Usage: specmon_cli.py: [options]

Options:
  -h, --help            show this help message and exit
  -b BBGAIN, --bbgain=BBGAIN
                        Set BB-Gain [default=30]
  -D DECIMATION, --decimation=DECIMATION
                        Set Decimation [default=10]
  -d DEVICEID, --deviceid=DEVICEID
                        Set Device ID [default=rtl=00000201]
  -f FREQ, --freq=FREQ  Set Frequency [default=431.0M]
  -g RFGAIN, --rfgain=RFGAIN
                        Set RF-Gain [default=30]
  -s SAMP_RATE, --samp-rate=SAMP_RATE
                        Set Sample Rate [default=2.0M]
  -T THRESHOLD, --threshold=THRESHOLD
                        Set Threshold [default=60.0]
  -v VERBOSE, --verbose=VERBOSE
                        Set Verbose [default=0]
  -R RUNTIME, --runtime=RUNTIME
  						Set Runtime in seconds [default=3590]
````
Be aware that sample-rate and frequency must be in scientific notation. For example: `-s 2e6 -D 10 -f 446.1e6`.

## Revisions
June 2020:
Software is capable of measuring shorter bursts. 
Reporting now has an average of the "low values", i.e. under the threshold, since the last trigger of the threshold.
Reporting the max value measured during the trigger of the threshold.

[RTL-SDR]: https://www.rtl-sdr.com/buy-rtl-sdr-dvb-t-dongles/
[HackRF]: https://greatscottgadgets.com/hackrf/
[PlutoSDR]: https://www.analog.com/en/design-center/evaluation-hardware-and-software/evaluation-boards-kits/adalm-pluto.html#eb-overview
[BladeRF]: https://www.nuand.com/
[4]: https://en.wikipedia.org/wiki/PMR446
[5]: https://www.gnuradio.org/
[gnuplot-palettes]: https://github.com/Gnuplotting/gnuplot-palettes
