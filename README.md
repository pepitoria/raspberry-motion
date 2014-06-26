Raspberry pi webcam motion detection system
===========================================
==========================================================================================================================


I wanted something to watch over my dog when I am not home so I made this. This project is for self documentation purposes, it's not meant to be a line by line tutorial and some understanding of how linux work is needed.


Install Raspbian
----------------

Install raspbian image with win32 disk imager
* Raspbian http://www.raspberrypi.org/downloads/
* win32 disk imager http://sourceforge.net/projects/win32diskimager/

After installing raspbian
-------------------------

##### Basic config
```sh
sudo raspi-config
# in the raspi-config do the next actions
#    - expand file system
#    - change default password
#    - select boot mode, console only, no desktop
#    - change locale and timezone
```
-----
##### Firmware update
```sh
sudo rpi-update
```
-----
##### Setup wifi
```sh
sudo vim.tiny /etc/network/interfaces
sudo vim.tiny /etc/wpa_supplicant/wpa_supplicant.conf
```
-----
##### Update and upgrade rasbian
```sh
apt-get update
apt-get upgrade
```

Installing and configuring motion 
---------------------------------
```sh
apt-get install motion
```
There are plenty of tutorials out there in how to configure Motion, for example:
 * https://medium.com/random-things/2d5a2d61da3d
 * https://www.debian-administration.org/article/347/An_Introduction_to_Video_Surveillance_with_'Motion

We tell motion to work as a daemon
```sh
sudo vim.tiny /etc/default/motion
# we set start_motion_daemon=yes
``` 
-----
Watch out for the default user and password for the web admin dashboard of motion, Change this in /etc/motion/motion.conf
```sh
control_authentication username:password
```
-----
##### Add motion to startup
Now for some reason there is (or was) a bug that made Motion to stop working short after launching it (a matter of seconds).
The workaround is to launch motion specifying the config file with the -c param. So, if you have this issue:
```sh
sudo vim.tiny /etc/rc.local
# we add this line to rc.local
# motion -c /etc/motion/motion.conf
``` 

If you don't have that issue and motion works as expected the standard way to launch motion at startup would be
```sh
sudo update-rc.d motion defaults
``` 

After having it added to startup if you want to stop launching it at startup
```sh
sudo update-rc.d motion stop 80 0 1 2 3 4 5 6 . 
``` 
-----
##### Upload pictures and videos to Dropbox directly from motion

We can set motion up to execute commands on picture (On_picture_save) and video (on_movie_end) taken. For that purpose we will use Dropbox Uploader (https://github.com/andreafabrizi/Dropbox-Uploader and forked by me, just in case https://github.com/pepitoria/Dropbox-Uploader) by Andrea Fabrizi.

We will to follow the instructions and execute dropbox_uploader.sh as root to set it up.

Later, at the end of /etc/motion/motion.conf we will add
```conf
On_picture_save dropbox_uploader.sh mkdir ./%F/
On_picture_save dropbox_uploader.sh upload %f ./%F/%f

on_movie_end dropbox_uploader.sh mkdir ./%F/
on_movie_end dropbox_uploader.sh upload %f ./%F/%f
``` 
-----

To avoid running out of space in tmp I set a cron task to delete everything in the motion tmp directory every five minutes


```sh
sudo crontab -e
```
And add
```sh
*/5 * * * * rm /tmp/motion/*
```
-----
Now I know there are options to make motion an authenticated service for the streams but the simples way I could find to do this was to install an authenticated proxy using nginx

Install nginx
-------------
```sh
sudo apt-get install nginx
```

We just need to edit the default site so it's an authenticated proxy to the motion video stream
```sh
sudo vim.tiny /etc/nginx/sites-enabled/default
```
things to change
  * the port you want nginx to listen to
  * the default root settings (proxy and auth)

```conf
server {
    listen   8088; ## listen for ipv4; this line is default and implied
    #listen   [::]:80 default_server ipv6only=on; ## listen for ipv6

    root /usr/share/nginx/www;
    index index.html index.htm;

    # Make site accessible from http://localhost/
    server_name localhost;

    location / {
        # First attempt to serve request as file, then
        # as directory, then fall back to displaying a 404.
        try_files $uri $uri/ /index.html;
        # Uncomment to enable naxsi on this location
        # include /etc/nginx/naxsi.rules
        proxy_pass http://localhost:8081;
        auth_basic "closed site";
        auth_basic_user_file /etc/nginx/users;
    }
    
    #check the whole file for reference
    ...
}
```

Forthe /etc/nginx/users you will need to add the users you want to have access by hand

```sh
sudo vim.tiny /etc/nginx/users
# add something like
# youruser:UQiO36tBXa102
```
The password is hashed, to create this password do

```sh
openssl passwd -crypt
```
You could do it with htpasswd but that is not installed by default in raspbian.


Powerless proof raspbian
========================
I experienced some data corruption issues so I decided to set it up as secure as possible in this regard. My solution is to mount some directories in an external usb drive and to mount / in read only mode.

I an mounting /tmp /var and /home in the pendrive so I will need 3 partitions for this. Plug the pendrive, you can check which where the pendrive is in /dev using

```sh
lsblk
```
In my case it is /dev/sda. So format it with cfdisk creating 3 partitions using ext4. Using lsblk again would result in something like
```sh
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    1   974M  0 disk
├─sda1        8:1    1 488.2M  0 part 
├─sda2        8:2    1 190.7M  0 part 
└─sda3        8:3    1 190.7M  0 part 
mmcblk0     179:0    0  14.9G  0 disk
├─mmcblk0p1 179:1    0    56M  0 part /boot
└─mmcblk0p2 179:2    0  14.9G  0 part /
```

After that we mount those partitions under /mnt in their respective directories (create them) to duplicate /tmp /var and /home in those partitions (if you copy as root watch out for permissions and ownership of everything you copy). We are NOT going to delete the original directories and their contents, just in case we need to boot up without the pendrive at some point. 

After that we modify /etc/fstab

```sh
sudo vim.tiny /etc/fstab
```
And we add
```sh
/dev/sda1 /var ext4 rw,relatime,data=ordered 0 0
/dev/sda2 /tmp ext4 rw,nosuid,noexec,relatime,data=ordered 0 0
/dev/sda3 /home ext4 rw,relatime,data=ordered 0 0
```
We can reboot to check that all this works.

After rebooting you should see them mounted in /etc/mtab

Once you've checked evertything works as desired we will change the mount mode for / so it's read only. Just add "ro" to the options
```sh
/dev/mmcblk0p2  /               ext4    defaults,noatime,ro  0       1
```

If you reboot now / should be mounted as read only.

When you are in read only mode if you want to make changes to something you can remount read write with
```sh
sudo mount / -o remount,rw
```
And mount back as read only with
```sh
sudo mount / -o remount,ro
```

I followed partly this thread in stack exchange to do this read only part http://raspberrypi.stackexchange.com/questions/5112/running-on-read-only-sd-card

-----
Other references.

Noip
====
http://www.noip.com/download?page=linux

-----
Thanks to Filipe, Mario and Agustín for the patience with all the questions :D
 * https://github.com/fcvarela
 * https://github.com/mjvm
 * https://github.com/agustinventura 
