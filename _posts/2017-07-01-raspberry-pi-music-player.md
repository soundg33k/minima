---
layout: post
categories: tech-recipe
---
## Disclaimer
This documentation might be incomplete. I decided to document this in case I need to do this again in the future. If this is useful for you then this is great, but <b>be sure to understand want the commands do before you type them in your bash prompt</b>!

## Before you start
Before you start this tech recipe, be aware that these distribution exists: [Rune audio](http://www.runeaudio.com/), [Volumio](https://volumio.org/), [Moode](http://moodeaudio.org/). All of these projects are maintaining pre-configured linux image containing the following programs:
- [mpd](https://www.musicpd.org/)
- [shairport-sync](https://github.com/mikebrady/shairport-sync)


## Why even bother... why not simply use Rune Audio, Volumio or Moode?
Most of the distribution listed above are made to work with a Raspberry PI that has a hifi DAC attached to it. While they say they support HDMI out, I was not happy with it. All of the distributions I tried had a funky UI to play with the mpd.conf file and this could be nice for people that don't want to get their hands dirty. However,
this was a big showstopper for me because whenever I was modifying the mpd.conf file the system did not seem to like it.

## What are we going to build
We will build a bit-perfect music player that supports up to 5.1 channels playback and up to 192kHz sample rate. The Raspberry PI will need to be connected in the HDMI input of an AVR. If the AVR can be network controlled, then it will be controlled by the Raspberry PI. [MPaD](http://www.katoemba.net/makesnosenseatall/mpad/) and [MPoD](http://www.katoemba.net/makesnosenseatall/mpod/) are used to control the music player. Note that [Soundirok](http://www.kvibes.de/en/soundirok/) could also be a good alternative since that MPaD and MPoD have not received any update on the AppStore for a while, but I haven't personally tested it yet.

## Let's build it

### Download Raspbian image
The first you need to do is to download to Raspian Jessie Lite image. I choose to use the lite version, because I don't need graphical interface.

[Raspbian download page](https://www.raspberrypi.org/downloads/raspbian/)

### Writing the image to the SD card
Here are the instruction for OS X. If you are on a different OS you can find instructions from the previous download link.
{% highlight bash %}
diskutil list
diskutil unmountDisk /dev/[your device]
sudo dd bs=1m if=~/Download/raspbian-jessie-lite.img of=/dev/disk2
{% endhighlight %}

### SSH access
Since we don't have a graphical user interface, we will use ssh to set-up the PI. Once the SD card has been written, create an empty file named "ssh" with no extension on the root of the sd card. The "ssh" file will tell the PI to wait for an SSH connection the start the set-up process. Then put the SD card in the Raspberry PI, connect an ethernet cable and boot the PI. When the PI is booted, you simply need to login using the following username and password.

{% highlight bash %}
#You will need to find the address of you PI
ssh pi@192.168.1.200
#username: pi
#password: raspberry
{% endhighlight %}

A good trick to find the IP of your Raspberry PI is to go on your router configuration page and look for it in the client list.

### Initial set-up
Once you are logged in on your PI you can start the raspi-config command.
{% highlight bash %}
sudo raspi-config
{% endhighlight%}
At this point changing password of the pi user is a good idea! Also you might want to assign a [static ip](https://www.modmypi.com/blog/tutorial-how-to-give-your-raspberry-pi-a-static-ip-address) to be sure that your router doesn't dynamically assign a new IP the next time it reboots.

### HDMI tweaks /boot/config.txt
I was having bunch of issues with my receiver and the PI. Most of these issues were caused by the HDMI-CEC. Seems like the PI was always trying to switch the input of my receiver... even if I was not playing any music on the PI. By adding the following modification to /boot/config.txt and reboot everything was fine.
{% highlight bash %}
hdmi_force_hotplug=1
hdmi_drive=2
hdmi_ignore_cec_init=1
hdmi_ignore_cec=1
{% endhighlight%}

###  Install shairport-sync
With shairport, you can send airplay audio stream to your PI. There is no package available from apt-get so you have to compile it yourself. To compile shairport-sync just follow the instruction from this page:

[https://github.com/mikebrady/shairport-sync](https://github.com/mikebrady/shairport-sync)

I also tried to use a package called shairplay... but it did not work at all. So far, I have been using Shairport-sync for couple of months now and it has been really reliable. I did not noticed any audio glitch or drop-out after listening to more than 100 hours of music.

### Samba share
Most of the NAS provide a samba share service (Windows file sharing service). The samba share can host your music files and the PI can simply read from it. Once the folder in being shared you can try to connect using a command similar to that.
{% highlight bash %}
sudo mount -t cifs -o user=pi,workgroup=WORKGROUP,file_mode=0444,dir_mode=0444,iocharset=utf8 //192.168.1.1/Music /mnt/routersamba
{% endhighlight%}
Once you have successfully mounted the samba share, you can can edit /etc/fstab so the PI mount the share when it boots. The line I have added in my /etc/fstab file looks like this:
{% highlight bash %}
//192.168.1.1/music /mnt/musicshare cifs ro,username=xx,password=xxxxxxxx,iocharset=utf8,gid=100,dir_mode=0444,file_mode=0444,work
{% endhighlight%}
For sure, you will need to modify it with your own settings.

### Install mpd
#### Install with apt-get
The easiest way to install MPD is with apt-get. However, when I installed MPD on my PI the NULL mixer_type was not available in the version installed from apt-get, and I need it for my set-up so I installed it from source... and for sure it is more trouble so it is up to you.
{% highlight bash %}
sudo apt-get mpd
{% endhighlight%}
#### Install from source
If you decide from source here is what this should looks like.
{% highlight bash %}
sudo apt-get install libmad0-dev libmpg123-dev libid3tag0-dev libflac-dev libvorbis-dev libfaad-dev libsamplerate0-dev libsoxr-dev libmpdclient-dev libsmbclient-dev libavahi-client-dev libsqlite3-dev libsystemd-daemon-dev libboost-dev libasound2-dev
git clone https://github.com/MaxKellermann/MPD.git
cd MPD/
sudo apt-get install autoconf
autoreconf -i -f
./configure --with-systemdsystemunitdir=/lib/systemd/system --with-systemduserunitdir=/usr/lib/systemd/user --sysconfdir=/etc
make
sudo make install
{% endhighlight%}

#### Tweak mpd.conf
All the settings for MPD are stored in this file: <b>/etc/mpd.conf</b>. One of the most important settings is <b>music_directory</b>, you will have to make that is points to the location where your samba share is mounted. I invite you to get familiar with the mpd.conf file (man mpd.conf). You should try to read thought the example mpd.conf file because there might be some settings that will be interesting for your set-up. When tweaking the mpd.conf file, the trick is to run mpd in verbose mode and no-daemon mode so you can see the log and know what is wrong with your mpd.conf settings.
{% highlight bash %}
mpd --no-daemon --verbose
{% endhighlight%}

Here are some my most interesting settings in mpd.conf.
{% highlight bash %}
music_directory		"/mnt/musicshare"

audio_output {
	type       "alsa"
	name       "My ALSA Device"
	device     "plughw:0,1"  # plughw will try to use best settings, otherwise it will do conversion
	mixer_type "null"        # volume is processed externally by the receiver
{% endhighlight%}

#### Configure mpd service
I have configured the service to run on as the user, but when we do that the user needs to be logged in so the process can runs. To do that you have to use <b>raspi-config</b> so the pi user is logged in when the device boots. It would be best to run the service as a system service, however I never manage to get it to work properly... so this is left as an exercise to the reader!

Once this is done you can enable the service with the following commands.
{% highlight bash %}
systemctl --user enable mpd.service
systemctl --start enable mpd.service
{% endhighlight%}

You might need to install libpam-systemd if you have issues with the user tasks.
{% highlight bash %}
sudo apt-get install libpam-systemd
sudo reboot
{% endhighlight%}

### Cover art for MPoD MPaD

{% highlight bash %}
sudo apt-get install lighttpd
sudo nano /etc/lighttpd/lighttpd.conf
{% endhighlight%}
Here are the variable you need to modify in your lighttpd.conf file.
{% highlight bash %}
server.document-root        = "/mnt/musicshare"
dir-listing.activate        = "enable"
{% endhighlight%}

{% highlight bash %}
sudo systemctl restart lighttpd.service
{% endhighlight%}

If you point your browser to the following PI address you should be able to browse your music librairy. Note that you need to put a Folder.jpg file inside each of the album folders. MPad and MPod will simply try to do the following album http request: http://rapsperrypi/Artist/Album/Folder.jpg

## You are done!
At this point, you should have a working music player that can be controlled with MPaD or MPoD.

### Support 5.1 playback (optional)
I had bunch of 5.1 recordings in flac. MPD supports playing back high-res multi-channel payback without any issue. However, the kernel sound module on the PI don't support it. So I got my hand dirty and made it work. You can find my patch [here](https://github.com/soundg33k/linux/commit/cdedcdc994be3e016f40962d304f655b58b7db97).
[Instructions on how to compile the kernel are here.](https://www.raspberrypi.org/documentation/linux/kernel/building.md)

The patch only supports 5.1 and not 5.0. Also you have to use plughw device and not hw. The reason for that is that the hdmi audio output wants 8 channels of data, plughw will take care of sending two empty channels.

I did not figure a way of building only the sound driver from scratch. I built the whole kernel on the PI and then use the instruction below to build and swap the sound driver.

{% highlight bash %}
# go to root of kernel repo
make modules SUBDIRS=sound/arm
sudo make modules_install SUBDIRS=sound/arm

#Unload sound driver
sudo modprobe -r snd_bcm2835
sudo modprobe snd_bcm2835
{% endhighlight%}

### AVR control (optional)
Nowadays, most of the AVR can be connected to the home network and have a network protocol. It is the case of my yamaha receiver. I created a python script that does the link it between shairport-sync, MPD and the AVR. The script simply manage the power and volume of my receiver. This might sound like a dumb feature, but it is actually quite useful. I made sure to deactivate any software volume in shairport-sync (ignore_volume_control=yes), MPD (mixer_type=null) or in the [sound driver](https://github.com/soundg33k/linux/commit/3a2f6d342a262dee091e77357babc85d07125c9b). This way the PI always output the sound at 0 dB and only the receiver applies the volume.

Making it work with mpd was quite easy. I have shared my code on this github [repo](https://github.com/soundg33k/avrcontroller), you can use it as a starting point for your project.

For shairport-sync, I had to work a bit harder to make it work. Again, I got my hand dirty and modify the source code. I used the [Lightweight Communications and Marshalling (LCM)](https://lcm-proj.github.io/) to send the volume from shairport-sync to my python avrcontroller. You can find my modifications [here](https://github.com/mikebrady/shairport-sync/compare/master...soundg33k:master).

You can clone my [repo](https://github.com/soundg33k/shairport-sync) if you want. To compile just make sure to add <b>--with-lcm</b>.
{% highlight bash %}
./configure --sysconfdir=/etc --with-alsa --with-avahi --with-ssl=openssl --with-metadata --with-soxr --with-systemd --with-lcm
{% endhighlight%}

Installing lcm is also quite easy:
{% highlight bash %}
git clone https://github.com/lcm-proj/lcm lcm
sudo apt-get install cmake libglib2.0-dev python-dev
cd lcm
cmake .
make
sudo make install
cd lcm-python
sudo python setup.py install
# update share lib path
sudo ldconfig -v
{% endhighlight%}

## Conclusion
I have been using my Raspberry PI as a music player for about a 6 months now and I haven't had any issues with it. I have shown my wife how to use the system with airplay and MPaD and she found it quite simple to use... so for me this is mission accomplish!
