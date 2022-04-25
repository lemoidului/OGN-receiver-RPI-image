## Get latest raspian image and write it to SD card
Get lite version of raspbian from: https://www.raspberrypi.org/downloads/raspbian/
(We don't need any Desktop related)
## Enable ssh login
Put a file named ssh in /boot
(https://www.raspberrypi.org/documentation/remote-access/ssh/)
## Add user pi and set its password
This is now required since Bullseye.
This can be done in Raspberry Pi Imager at image writing time or following info described here:
https://www.raspberrypi.com/news/raspberry-pi-bullseye-update-april-2022/
## Boot RPi with this SD card
Upgrade system:
```
apt-get update
apt-get dist-upgrade
```
Update hostname to `ogn-receiver` thanks to `raspi-config`
Disable autologin thanks to `raspi-config` or:
```
rm /etc/systemd/system/getty@tty1.service.d/autologin.conf
```
## Install standard OGN lib & softs + standard config
```
apt-get update
apt-get install rtl-sdr libconfig9 libfftw3-dev procserv telnet ntpdate ntp lynx
```
* Apply DVB-T blacklist
```
cat >> /etc/modprobe.d/rtl-glidernet-blacklist.conf <<EOF
blacklist rtl2832
blacklist r820t
blacklist rtl2830
blacklist dvb_usb_rtl28xxu
EOF
```

* Install service
```
wget https://raw.githubusercontent.com/snip/OGN-receiver-RPI-image/master/dist/rtlsdr-ogn -O /etc/init.d/rtlsdr-ogn
wget https://raw.githubusercontent.com/snip/OGN-receiver-RPI-image/master/dist/rtlsdr-ogn-service.conf -O /etc/rtlsdr-ogn-service.conf
chmod +x /etc/init.d/rtlsdr-ogn
update-rc.d rtlsdr-ogn defaults
```

## Manage /boot/OGN-receiver.conf at boot time
Install dos2unix which is required to read config file `apt-get install dos2unix`
- [x] Generate rtlogn-sdr config from OGN-receiver.conf
- [x] If exist use /boot/rtlsdr-ogn.conf at boot
- [x] Disable pi user password login (only ssh key login)
- [x] Change pi user password & allow password login
- [x] Option to run a specific command at each boot
- [x] Manage rtlsdr-ogn auto upgrade => Download at each rtlsdr-ogn startup.

```
wget https://raw.githubusercontent.com/snip/OGN-receiver-RPI-image/master/dist/OGN-receiver.conf -O /boot/OGN-receiver.conf 
wget https://raw.githubusercontent.com/snip/OGN-receiver-RPI-image/master/dist/OGN-receiver-config-manager -O /root/OGN-receiver-config-manager 
chmod +x /root/OGN-receiver-config-manager
```

## Manage optional remote admin
```
apt-get install autossh
ssh-keygen
cat ~/.ssh/id_rsa.pub 
wget "https://raw.githubusercontent.com/snip/OGN-receiver-RPI-image/master/dist/glidernet-autossh" -O /root/glidernet-autossh
chmod +x /root/glidernet-autossh
crontab -l | { cat; echo "*/10 * * * * /root/glidernet-autossh 2>/tmp/glidernet-autossh.log"; } | crontab -
```

as pi:
```
mkdir .ssh
wget "http://autossh.glidernet.org/~glidernet-adm/id_rsa.pub" -O .ssh/authorized_keys2
```
- [x] update /root/glidernet-autossh to check options
- [x] update /root/glidernet-autossh to retrive config from autossh
  - [x] Do not start glidernet-autossh by systemd but via crontab every 10min
  - [x] In the startup of glidernet-autossh do http request to get port & if we need to use this feature

## Indicate image version
as root :
```
echo "TOBERELACEDBYIMAGEVERSION" > /root/image-version
```

## Manage self-update
Example taken from: https://github.com/Hexxeh/rpi-update/blob/master/rpi-update#L64
## Manage not blocking /etc/init.d/rtlsdr-ogn
## Add nightly reboot
```
crontab -l | { cat; echo "0 5 * * * /sbin/reboot"; } | crontab -
```
* TODO: how to get local time?

Maybe with https://ipsidekick.com/json or https://ipapi.co/timezone/ ? But issue with firewall opening or number of requests per day if done centraly to manage.

## Manage firstboot ? => Do we realy need it?
* To create hosts ssh keys on rw SD card. Then activate RO?
* In any cases root's ssh keys need to be the same for autossh remote admin.
* We need to expend FS at first boot => Do we realy need it? Don't think so.
```
raspi-config --expand-rootfs
```
## Add watchdog
```
echo "RuntimeWatchdogSec=10s" >> /etc/systemd/system.conf
echo "ShutdownWatchdogSec=4min" >> /etc/systemd/system.conf
```
See: https://www.raspberrypi.org/forums/viewtopic.php?f=29&t=147501&start=25#p1254069
And to check if it working well, to generate a kernel panic: `echo c > /proc/sysrq-trigger`
## Disable swap
```
sudo systemctl stop dphys-swapfile
sudo systemctl disable dphys-swapfile
sudo apt-get purge dphys-swapfile
```
## Disable fake-hwclock?
```
update-rc.d fake-hwclock disable
```
As we are going to be RO file system we will not rely on `/etc/fake-hwclock.data`.

## Force time sync every 10 minutes
```
*/10 * * * * ( /usr/sbin/ntpdate -u pool.ntp.org ) > /tmp/ntp-sync.log 2>&1
```
## Allow usage of GPU for model before Pi4/CM4
Comment following line in `/boot/config.txt`
```
dtoverlay=vc4-kms-v3d
```
Without this `gsm_scan` and `ogn-rf` in there GPU version will crash with error like:
`The process was killed by signal 8`or `Floating point exception` due to impossibility to use GPU.

## Add RO FS
```
cd /sbin
wget https://github.com/ppisa/rpi-utils/raw/master/init-overlay/sbin/init-overlay
wget https://github.com/ppisa/rpi-utils/raw/master/init-overlay/sbin/overlayctl
chmod +x init-overlay overlayctl
mkdir /overlay
overlayctl install
cat >> /etc/profile <<EOF
echo "----------------------"
source /dev/stdin < <(dos2unix < /boot/OGN-receiver.conf)
echo "OGN receiver \$ReceiverName"
echo "Read-only file system (overlay) status:"
/sbin/overlayctl status
echo "To manage it (as root): overlayctl disable | overlayctl enable | overlayctl status"
echo "----------------------"
EOF
```
(from: http://wiki.glidernet.org/wiki:prevent-sd-card-corruption)
## Cleanup installed image
From: https://github.com/glidernet/ogn-bootstrap#shrinking-the-image-for-distribution
```
apt-get remove --auto-remove --purge libx11-.*
apt-get autoremove
apt-get autoclean
apt-get clean
```

Optional: fill not used space with 0 (allow beter compression). This can be done at next step by mountinng loopback FS.
```
dd if=/dev/zero of=file-filling-disk-with-0 bs=1M
rm file-filling-disk-with-0
```
Optional: Remove history.

## Reboot

## Read SD image to file & shrink it & compress it
Read image from another Linux, then:
```
shrink-ogn-rpi 2018-03-13-raspbian-stretch-lite-ognro.img
zip -9 2018-03-13-raspbian-stretch-lite-ognro.zip 2018-03-13-raspbian-stretch-lite-ognro.img
```
