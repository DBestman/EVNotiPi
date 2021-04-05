# EVNotiPi
Work In Progress.  
Adapted to suit MY hardware, some of which I already owned (OBDLink MX+, USB GPS antenna, Huawei MS2372H-517 4G Modem).
I don't suggest that someone else repeat these steps, as I would do things differently if I had to do it again.  But I still want to document what I did, mostly for myself:

Python Version of EVNotify
## Needed Hardware
- Raspberry Pi Zero W with GPIO Header https://buyzero.de/collections/boards-kits/products/raspberry-pi-zero-w-easy-mit-bestucktem-header
- OTG cable for Raspberry Pi https://buyzero.de/collections/raspberry-kabel-und-adapter/products/micro-usb-zu-usb-otg-adapter
- Small wires in different colors: https://www.amazon.de/AZDelivery-Jumper-Arduino-Raspberry-Breadboard/dp/B07KYHBVR7?tag=gplay97-21
- LTE Stick Huawai MS2372H-517, which works for me in Canada.  I had great difficulty finding an LTE stick that works in Canada.  But I didn't realize before buying that this stick did not have Wi-Fi, so it complicated my setup by forcing me to buy a USB hub, which wouldn't be needed if if the Pi connected to the Internet via Wi-Fi.  If I were to buy again, I would try and find one that has Wi-Fi.  Setup would have been much simpler.
- Rapberry Pi USB Hub, to be able to connect both my LTE stick and the GPS antenna: https://www.amazon.ca/MakerSpot-Stackable-Raspberry-Connector-Bluetooth/dp/B01IT1TLFQ
- USB GPS antenna: https://www.amazon.ca/Navigation-External-Receiver-Raspberry-Geekstory/dp/B078Y52FGQ
- OBDLink MX+ that I alrady owned: https://www.amazon.ca/OBDLink-Bluetooth-Professional-Grade-Diagnostic-Performance/dp/B07JFRFJG6.  It would have been simpler to buy a PiCan.  But I already owned this device and I'm cheap.
- The i2c watchdog and power supply, https://github.com/noradtux/evnotipi-watchdog.  I modified this design to use a buck converter as the power supply instead of the linear regulator, and a relay instead of the mosfets, both of which I already owned (Again, I'm cheap).  The (DPDT) relay also allows me to completely turn off the power to the OBDII dongle
- OBD2 male and female connectors: https://www.aliexpress.com/item/4000087243731.html
- I designed & 3D-printed my modified version of the case https://github.com/noradtux/evnotipi-case.
## Wiring
### OBD2 connection
```
OBD2       
4,5  GND   
  6  CAN_H 
 14  CAN_L 
 16  12V   
```
I only connected Pin4 to Ground as it is documented as Chassis Ground.  Pin5 is the Signal Ground, I wanted to make sure I don't contaminate it.

## Installation
### Raspberry Pi
- sudo apt update
- sudo apt upgrade
- sudo apt install python3-{pip,rpi.gpio,serial,requests,sdnotify,pyroute2,smbus,yaml,gevent} gpsd git watchdog rsyslog-
- sudo systemctl disable --now serial-getty@ttyAMA0.service
- sudo sed -i -re "\\$agpu_mem=16\nmax_usb_current=1\ndtoverlay=gpio-poweroff,gpiopin=4,active_low=1\ninitial_turbo=60\nboot_delay=0\ndisable_splash=1" -e "/^dtparam=audio=/ s/^/#/" /boot/config.txt
- sudo sed -i -re '/console=/ s/$/ panic=1/' /boot/cmdline.txt
- sudo sed -i -re '/max-load/ s/^#//' /etc/watchdog.conf
- sudo sed -i -re "\\$adtparam=watchdog=on" /boot/config.txt
#### If using the i2c watchdog (the one with the Trinket M0):
- sudo sed -i -re "\\$adtparam=i2c_arm=on,i2c_arm_baudrate=50000" /boot/config.txt
- sudo sed -i -re "\\$ai2c-dev" /etc/modules
### EVNotiPi
- sudo git clone https://DBestman.com/EVNotify/EVNotiPi /opt/evnotipi
- cd /opt/evnotipi
- sudo pip3 install -r requirements.txt
- sudo systemctl link /opt/evnotipi/evnotipi.service
- sudo systemctl enable evnotipi.service
- sudo systemctl disable evnotipi_shutdown.{timer,service} # if updating
- sudo cp config.yaml.template config.yaml
#### Edit config, follow comments in the file
- sudo nano config.yaml # nano or any other editor
#### Set up Bluetooth OBDII dongle
- bluetoothctl
- [bluetooth]# power on
- [bluetooth]# scan on
- [bluetooth]# scan off
- [bluetooth]# pair <MAC>
- [bluetooth]# quit
- rfcomm bind 0 <MAC> # /dev/rfcomm0 should appear
- sudo systemctl link /opt/evnotipi/rfcomm-bind@.service
- sudo systemctl enable rfcomm-bind@<MAC>.service
#### Set up USB LTE Stick
- sudo nano USBModem.rules # nano or any other editor
- sudo ln -s USBModem.rules /etc/udev/rules.d/20-USBModem.rules  # Install USBModem.rules : http://reactivated.net/writing_udev_rules.html#why
- sudo nano /etc/wvdial.conf # nano or any other editor
- sudo systemctl link /opt/evnotipi/wvdial.{path,service}
- sudo systemctl enable wvdial.{path,service}
#### Set up a GPS receiver
Set up gpsd, see a tutorial here: https://maker.pro/raspberry-pi/tutorial/how-to-use-a-gps-receiver-with-raspberry-pi-4. Once the "sudo cgps -s" command works, you are done.
#### Optional: Set up a RaspAP
RaspAP allows the Pi to become a wireless access point when you're in your car.  https://docs.raspap.com/ap-sta/
I had to reinstall several times before I could configure properly, but I don't remember what were the difficulties.
To uninstall RaspAP:
- cd /var/www/html
- sudo installers/uninstall.sh
