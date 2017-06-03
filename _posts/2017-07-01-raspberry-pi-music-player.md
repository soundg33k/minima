---
layout: post
categories: tech-recipe
---
## Before you start
Before you start this tech recipe, be aware that these distribution exists: [Rune audio](http://www.runeaudio.com/), [Volumio](https://volumio.org/), [Moode](http://moodeaudio.org/). All of these projects are maintaining pre-configured linux image containing the following programs:
- [mpd](https://www.musicpd.org/)
- [shairport-sync](https://github.com/mikebrady/shairport-sync)

## Why even bother... why not simply use Rune Audio, Volumio or Moode?
Most of the distribution listed above are made to work with a Raspberry Pi that has a hifi DAC attached to it. While they say they support HDMI out, I was not happy with it. All of the distributions I tried had a funky UI to play with the mpd.conf file. Where this could be nice for people that don't want to get their hands dirty. This was a big showstopper for me because whenever I was modifying the mpd.conf file the system did not seem to like it.

## What are we going to build
We will build a bit-perfect music player that supports up to 5.1 channels playback and up to 192kHz sample rate. The Raspberry Pi will need to be connected in the HDMI input of an AVR. If the AVR can be network controlled, then it will be controlled by the Raspberry Pi. MPad and MPod are used to control the music player.

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
Since we don't have a graphical user interface, we will use ssh to set-up the PI. Once the SD card has been written, create an empty file named "ssh" with no extension on the root of the sd card. The "ssh" file will tell the PI to wait for an SSH connection the start the set-up process. Then put the SD card in the Raspberry PI, connect an ethernet cable and boot the PI. When the Pi is booted, you simply need to login using the following username and password.

{% highlight bash %}
#You will need to find the address of you Pi
ssh pi@192.168.1.200
{% endhighlight %}

A good trick to find the IP of your Raspberry Pi is to go on your router configuration page and look for it in the client list.

### Initial set-up
Once you are logged in on your Pi you can start the raspi-config command.
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
With shairport, you can send airplay audio stream to your PI. There is no package available from apt-get so you have to compile it yourself. So to compile shairport-sync just follow the instruction from this page:

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
The easiest way to install MPD is with apt-get. However, when I installed MPD on my PI the NULL mixer_type was not available in the version installed from apt-get, and I need it for my set-up so I installed it from source... but it is more trouble for source. This is up to you.
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
All the settings for MPD are stored in this file: <b>/etc/mpd.conf</b>. One of the most important settings is <b>music_directory</b>, you will have to make that is points to the location where your samba share is mounted. I invite you to get familiar with the mpd.conf file (man mpd.conf). You should try to read thought the example mpd.conf file because their might be some settings that will be interesting for you set-up. When tweaking the mpd.conf file, the trick is to run mpd in verbose mode and no-daemon mode so you can see the log and know what is wrong with your mpd.conf settings.
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
I have configured the service to run on as the user, but when we do that the user needs to be logged in so the process runs. To do that you have to use <b>raspi-config</b> so the pi user is logged in when the device boots. It would be best to run the service as a system service, however I never manage to get it to work properly... so this is left as an exercise to the reader!

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
