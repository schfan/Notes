# How to upgrade Samsung Gear Live Watch 

This post is about my journey on upgrading a Samsung Gear Live Watch purchased on Amazon. The OS was 5.0.1, and my goal was to upgrade its OS to Android 5.1.1. Perhaps because the onstock system image was a developer version, the default **Update** button does not give me an update. I tried many times to install a customized system image, but `adb sideload filename.zip` does not work (due to an `error:closed` error), forbidding me to load any external file onto the watch to install. Fianlly, I used [Samsung Gear Live Restore Tool V2](http://forum.xda-developers.com/gear-live/development/utility-gear-live-watch-tool-t2846696) from xda and reversed to a system image that allows automatic system upgrade and then upgraded from there. Here are the steps. 

1. Search and download the restore tool. 
2. Unzip, pre-process the files. Since the developer of this tool (a shell script called `2-Gear-Live-Tool-for-Linux-and-OSX.sh`) used a Windows development environment, I had to use `dos2unix 2-Gear-Live-Tool-for-Linux-and-OSX.sh` to convert the bash script file to a Unix-friendly one. Also, the script calls a few binaries which are included in the same zip file. For convenience, I just did `chmod +x *` for everything in the folder so that the binaries will have execution permissions.
3. Bug. Due to some reason (perhaps dos2unix), one image file is mal-named in the script. Open the script and search for `78yrecovery`, and change it to `78Yrecovery`. 
4. Boot the android watch into bootloader. This can be done either using `adb reboot bootloader` (if the watch's debugging interface is enabled) or reboot the device (by holding the power button) and *swipe twice* from *top left* to *bottom right* when the Samsung logo shows up (tip: this is easier to do when the power cable is disconnected).
5. Execute the shell script, and type `restore`.  Follow the instructions in the script. 

Only the V2 of the Restore Tool worked for me. After the restoration, the watch's OS reversed to Android 4.4W. Select **System Update** in the **Settings** and download/upgrade the system. The watch will then update to Android 4.4W-2. Repeat the process, update the system to 5.0.1 and then 5.1.1. Voila! 

### How to solve `adb devices no permission` error

Very often, when we connect a new device to the developer machine, we might encounter `no permission` error when we do `adb devices`, and even `adb kill-server` might not help. If the device's developer option is enabled, the error is most likely caused by the developer machine's udev settings. Here is how I solved it. 

1. Connect the device to the machine via USB. 
2. Check `lsusb` and find the entry related to the device. Look for keywords like `Samsung` or `Google`. One way to find out the usb info of the device is to disconnect and reconnect it and see the difference in the output. For example, here is a sample output without connecting: 
```bash
$ lsusb
Bus 002 Device 004: ID 148f:5370 Ralink Technology, Corp. RT5370 Wireless Adapter
Bus 002 Device 003: ID 046d:0825 Logitech, Inc. Webcam C270
Bus 002 Device 002: ID 8087:0024 Intel Corp. Integrated Rate Matching Hub
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 002: ID 8087:0024 Intel Corp. Integrated Rate Matching Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 003 Device 003: ID 046d:c52b Logitech, Inc. Unifying Receiver
Bus 003 Device 002: ID 2516:0004  
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```
And here is the output with the device connected: 
```bash
$ lsusb
Bus 002 Device 004: ID 148f:5370 Ralink Technology, Corp. RT5370 Wireless Adapter
Bus 002 Device 003: ID 046d:0825 Logitech, Inc. Webcam C270
Bus 002 Device 002: ID 8087:0024 Intel Corp. Integrated Rate Matching Hub
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 020: ID 18d1:4ee7 Google Inc. 
Bus 001 Device 002: ID 8087:0024 Intel Corp. Integrated Rate Matching Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 003 Device 003: ID 046d:c52b Logitech, Inc. Unifying Receiver
Bus 003 Device 002: ID 2516:0004  
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```
Observe the difference:
```bash
Bus 001 Device 020: ID 18d1:4ee7 Google Inc. 
```
Now edit the udev file at
```bash
$ sudo vi /etc/udev/rules.d/51-android.rules
```
And add this line:
```bash
# Samsung Gear Live
SUBSYSTEM=="usb", ATTR{idVendor}=="18d1", ATTR{idProduct}=="4ee7", MODE="0600", OWNER="spica"
```
where `spica` is my username. 
Restart udev:
```bash
$ sudo udevadm control --reload-rules
$ sudo service udev restart
```
Now we can find that the device can be properly recognized by `adb` (and a permission confirmation might prompt on the device).
```bash
$ adb devices
List of devices attached
R3AF600RD4Kdevice
```
