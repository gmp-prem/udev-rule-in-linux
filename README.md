## Manage devices dynamically using UDEV in Linux
How to dynamically manage devices on linux ubuntu by make use of udev rules. Udev is a device manager for Linux that dynamically creates and removes nodes for hardware devices. In short, it helps your computer find your robot easily.

<p align="center">
  <img src=https://github.com/gmp-prem/assigning-static-port-ubuntu/blob/main/picture.png width="700" height="200">
</p>

## Why assign?
The linux ubuntu will read usb device sequencially as you plug in, e.g. you have 2 ttyUSBx devices, one for the camera and another one for the servo, if you plug in the camera and the servo respectively, the ubuntu will read camera as ttyUSB0 and servo as ttyUSB1. On the other hand, if you do opposite way, you will read servo as ttyUSB0 and camera as ttyUSB1. To prevent this happens, there is a method to assign a static port on linux ubuntu, of course, there is also for Windows and Mac but this is for the linux only.

## Udev for /dev/ttyUSBx
_these are steps for assigning the **ttyUSBx**, for ttyACMx, there is a little change which will be after this._
1. Plug in the device
2. check that if linux reads the device or not using command in the terminal, this should show all ttyUSBx devices you plugged in
```
ls -l /dev/ttyUSB*
```
3. Check the port detail using **udevadm**, this will show attributes of device you plugged in, the _x_ is the number of port you plugged in, in this case, it can be from 0, 1, 2, 3 and so on
```
udevadm info --name=/dev/ttyUSBx --attribute-walk
```
4. once you enter, look for the line that is written like below, at this point, I assume that you know what are PID, VID and Serial of device
```
ATTRS{idProduct}=="6001"
ATTRS{idVendor}=="0403"
ATTRS{serial}=="FTYS6KQ7"
```
5. After you get the detail of the port, then head to `/etc/udev/rules.d/` and create the `10-usb-serial.rules` file , and use your favorite editor to edit this file
```
cd /etc/udev/rules.d/
```
```
touch 10-usb-serial.rules
```
6. Open up the `10-usb-serial.rules` and type the following, replace the attribute PID, VID and the serial you found from step number 4, then use your custom name  for your USB device e.g. ttyUSB_CAMERA1, ttyUSB_CAMERA2 or anything else you feel comfortable with. This could be more than 1 line of code if you have multiple devices connected and want to assign a static port to every connected device
```
KERNEL=="ttyUSB*", ATTRS{idProduct}=="6001", ATTRS{idVendor}=="0403", ATTRS{serial}=="FTYS6KQ7", SYMLINK+="YOURNAME"
```
7. After finish the code, save the file and trigger the udevadm to allow our mapping take effect
```
sudo udevadm trigger
```
8. Check the USB port again, and it should show something like this. If your mapping does not show, try reboot the ubuntu, if it does not appear like below again, it means that you must have a typo in the `10-usb-serial.rules`
```
crw-rw---- 1 root dialout 188, 0 11月  2 12:24 /dev/ttyUSB0
lrwxrwxrwx 1 root root         7 11月  2 12:24 YOURNAME -> ttyUSB0
```

## Udev for /dev/ttyACMx
steps are all the same but the command is a bit different, I will have only a command except the explainations
1. Plug in the device
2. Check port
```
ls -l /dev/tty/USB*
```
3. Check port attributes
```
udevadm info --name=/dev/ttyACMx --attribute-walk
```
4. Look for the following attributes
```
ATTRS{idProduct}=="0043"
ATTRS{idVendor}=="2341"
ATTRS{serial}=="75232302221E05110"
```
5. Head to `/etc/udev/rules.d/` and create the `10-usb-serial.rules` file, if you already have, do not need to create
6. Open up and type, save and exit
```
KERNEL=="ttyACM*", ATTRS{idProduct}=="0043", ATTRS{idVendor}=="2341", ATTRS{serial}=="75232302221E05110", SYMLINK+="YOURNAME"
```
7. Trigger mapping to take effect, or reboot the ubuntu
```
udevadm control --reload-rules
```
8. After reboot, check the port using `ls -l /dev/ttyACM*`, if it appears like below, it is correct, if not, check the mapping file again
```
crw-rw---- 1 root dialout 166, 0 11月  2 12:25 /dev/ttyACM0
lrwxrwxrwx 1 root root         7 11月  2 12:24 YOURNAME -> ttyACM0
```

P.S. The name you are going to re-assign should be the name that it is easy to remember and use ! otherwise, you might get confuse with what you just re-assgin

## Udev for USB camera to be integrated in ROS
If you want to use USB camera in ROS, [libuvc]([https://wiki.ros.org/libuvc_camera](https://github.com/libuvc/libuvc)https://github.com/libuvc/libuvc) provides [libvuc_camera](https://wiki.ros.org/libuvc_camera) integrated ROS interface that have some great features:
1. Dynamic reconfiguration and monitoring of the camera's control settings
2. Camera calibration [camera_info_manager](https://wiki.ros.org/camera_info_manager)
3. Image pipeline integration [image_transport](https://wiki.ros.org/image_transport)

🟢 **Tested** on Linux Ubuntu 20, ROS Noetic

How to integrated ROS interface with your USB camera
1. Install some important packages
```
sudo apt install ros-noetic-rgbd-launch libuvc-dev
```
```
sudo apt install ros-noetic-libuvc-camera
```
2. Give permission (chmod) to camera by specifying the product and vendor IDs of your USB using udev rules
   1. find the product and vendor ID of your camera
   ```
   lsusb
   ```
   2. go to udev rules directory
   ```
   cd /etc/udev/rules.d/
   ```
   3. create udev rule file for your camera
   ```
   sudo touch 11-udev-camera.rules
   ```
   4. insert this code, then change the product and vendor ID that matches from what you have found from `lsusb` coommand (:fire: **use sudo to edit the file because you created udev file with sudo**)
   ```
   SUBSYSTEMS=="usb", ENV{DEVTYPE}=="usb_device", ATTRS{idVendor}=="046d", ATTRS{idProduct}=="08cc",OWNER="myuser"
   ```
3. After that, we will restart the udev by
```
sudo udevadm trigger
```
4. After that we will create the launch file for your USB, go to any package you wish and inside the launch folder, you created the launch file with these xml script
```
<launch>
  <group ns="camera">
    <node pkg="libuvc_camera" type="camera_node" name="mycam">
      <!-- Parameters used to find the camera -->
      <param name="vendor" value="0x0"/>
      <param name="product" value="0x0"/>

      <!-- Image size and type -->
      <param name="width" value="640"/>
      <param name="height" value="480"/>
      <!-- choose whichever uncompressed format the camera supports: -->
      <param name="video_mode" value="uncompressed"/> <!-- or yuyv/nv12/mjpeg -->
      <param name="frame_rate" value="15"/>

      <param name="timestamp_method" value="start"/> <!-- start of frame -->
      <param name="camera_info_url" value="file:///tmp/cam.yaml"/>

      <param name="auto_exposure" value="3"/> <!-- use aperture_priority auto exposure -->
      <param name="auto_white_balance" value="false"/>
    </node>
  </group>
</launch>
```
You can see that there is a parameter vendor and product, then you replace the orignal with vendor and product ID you wrote in the udev rule file, for example:

in my case, i have vendor and product ID of 337b and 090c respectively, so

`<param name="vendor" value="0x0"/>` would be `<param name="vendor" value="0x337b"/>`

`<param name="product" value="0x0"/>` would be `<param name="product" value="0x090c"/>`

And you can change parameters that suit your application
