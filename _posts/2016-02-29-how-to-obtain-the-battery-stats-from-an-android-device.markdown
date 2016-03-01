---
layout: post
comments: true
title:  "How to obtain the battery stats from an Android device"
date:   2016-02-29 20:29:20 -0500
categories: notes
---
Source: https://developer.android.com/tools/performance/batterystats-battery-historian/index.html

Google provides an open-source Battery Historian Phython script that plots the power stats, but it is not necessary, especially if you intend to do customized processing on the stats file. Therefore, this post focus on the post-processing of the power stats, e.g., text processing, plotting, etc.  

## Battery data gathering
1. First, connect your device to your developer machine via USB. Check: 
```bash
$ adb devices
List of devices attached
R3AF600RD4Kdevice
```
2. Reset the battery readings. According to Google, 
> Resetting erases old battery collection data; otherewise, the output will be huge. 
```bash
$ adb shell dumpsys batterystats --reset
```
3. Disconnect the device, such that Android only draws current from the device battery. 
4. Play with the app to be tested for a while.  This should be long enough to incur certain amount of battery level changes. 
5. Reconnect the device. 
6. Dump the battery data. 
```bash
$ adb shell dumpsys batterystats > batterystats.txt
```
Thus far, the data collection part has finished. 

## Data processing
Here are some of my scripts that process the battery stats. They are not very stable, so you might want to check the output at each step and make sure the processing was correct. 

1. Obtain only battery info. 

```bash
# getBatteryInfo.sh
grep '         ' batterystats.txt > batterylog.txt
sed -i 's/         //g' batterylog.txt
sed -i 's/(.)//g' batterylog.txt
sed -i 's/+//g' batterylog.txt
sed -i '1d' batterylog.txt
awk '{print $1,$2}' batterylog.txt > batterylog.txt.tmp
mv batterylog.txt.tmp batterylog.txt
dos2unix batterylog.txt
sed -i 's/ms//g' batterylog.txt
sed -i 's/m/:/g' batterylog.txt
sed -i 's/s/./g' batterylog.txt
```

2. Using R to continue processing and plot. 

```r
# plot.R
library(gdata)
options(stringsAsFactors = FALSE)
df=read.table('batterylog.txt')
timestamp=df$V1
battery=df$V2/100
options(digits.secs=3)
timestamp[grepl("\\.", timestamp)==FALSE]=paste("00.",timestamp[grepl("\\.", timestamp)==FALSE],sep='')
timestamp[grepl(":", timestamp)==FALSE]=paste("00:",timestamp[grepl(":", timestamp)==FALSE],sep='')
timestamp=strptime(timestamp, format="%M:%OS")
timestamp=timestamp-timestamp[1]
plot(timestamp, battery, type='s')

# calculate slope
model1=lm(formula = battery ~ timestamp)
#summary(model1)
#par(mfrow = c(2, 2),pty="s")
#plot(model1)
lines(timestamp, fitted(model1), type='o', col='red')

# calculate power
# spec of specific device
current=0.3 #amp
voltage=3.8 #volt
duration=3600 #second
capacity=current*voltage*duration #joule
rate=as.numeric(model1$coefficients[2])
power=-capacity*rate
power=round(power,4)
print(power)
```
