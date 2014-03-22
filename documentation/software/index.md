---
title: Software
layout: default
currentpage: software
---

Software Guide
===

You can choose from one of two ways to install Jasper's software on your Raspberry Pi.

-


<h1 class="linked" id='quick-start'><a href="#quick-start" title="Permalink to this headline">Method 1: Quick Start (Recommended)</a></h1>

The quickest way to get up and running with Jasper is to download the pre-compiled disk image [available here](). After imaging your SD card, skip to the section below titled "Configuring Jasper Client". The other instructions are those who wish to understand how all of the supporting libraries are compiled on the Raspberry Pi. The instructions may be helpful for debugging.

-

<h1 class="linked" id='manual-installation'><a href="#manual-installation" title="Permalink to this headline">Method 2: Manual Installation</a></h1>

Follow these instructions only if you wish to compile your Jasper software from scratch. These steps are unnecessary if you follow the recommended "Quick Start" instructions above.


<h2 class="linked" id='burn-image'><a href="#burn-image" title="Permalink to this headline">Burn Raspbian image onto SD card</a></h2>

We'll first clear the SD card using Gparted on Ubuntu, but you can use an equivalent utility or operating system. In Gparted: right-click on each partition of the SD card, then select 'Unmount' and 'Delete'. Apply the changes with Edit > Apply All Operations.

Download Raspbian Wheezy from [http://downloads.raspberrypi.org/raspbian_latest](http://downloads.raspberrypi.org/raspbian_latest). While we've tested Jasper on the 2014-01-07 release, newer releases may also work.

We'll use dd to burn the image to the disk. Obtain the address of the SD card with:

{% highlight bash %}
sudo fdisk -l
{% endhighlight %}

Our address was '/dev/mmcblk0', so the following command burns the image to the disk:

{% highlight bash %}
sudo dd if=2013-12-20-wheezy-raspbian.img of=/dev/mmcblk0 bs=2M
{% endhighlight %}

When it's done, remove your SD card, insert it into your Raspberry Pi and connect it to your computer via ethernet.

<h2 class="linked" id='configure-raspbian'><a href="#configure-raspbian" title="Permalink to this headline">Configure Raspbian</a></h2>


We're now going to do some basic housekeeping and install some of the required libraries. You should SSH into your Pi with a command similar to the following. The IP address usually falls in the 192.168.2.3-192.168.2.10 range.

{% highlight bash %}
ssh pi@192.168.2.3 # password (default): raspberry
{% endhighlight %}

Run the following, select to 'Expand Filesystem' and restart your Pi:

{% highlight bash %}
sudo raspi-config
{% endhighlight %}

Run the following commands to update Pi and some install some useful tools.

{% highlight bash %}
sudo apt-get update
sudo apt-get upgrade --yes
sudo apt-get install vim --yes
sudo apt-get install git-core --yes
sudo apt-get install espeak --yes
sudo apt-get install python-dev --yes
sudo apt-get install python-pip --yes
sudo apt-get install bison --yes
sudo apt-get install libasound2-dev --yes
sudo apt-get install libportaudio-dev python-pyaudio --yes
{% endhighlight %}

Plug in your USB microphone. Let's open up an ALSA configuration file in vim:

{% highlight bash %}
sudo vim /etc/modprobe.d/alsa-base.conf
{% endhighlight %}

Change the following line:

{% highlight bash %}
options snd-usb-audio index=-2
{% endhighlight %}

To this:

{% highlight bash %}
options snd-usb-audio index=0
{% endhighlight %}

Back in the shell, run:

{% highlight bash %}
alsa force-reload
{% endhighlight %}

Test that recording works with:

{% highlight bash %}
arecord temp.wav
{% endhighlight %}

Make sure you have speakers or headphones connected to the audio jack of your Pi. You can play back the recorded file:

{% highlight bash %}
aplay -D hw:1,0 temp.wav
{% endhighlight %}

And this to your ~/.bashrc or ~/.bash_profile:

{% highlight bash %}
LD_LIBRARY_PATH="/usr/local/lib"
export LD_LIBRARY_PATH
PATH=$PATH:/usr/local/lib/
export PATH
{% endhighlight %}

With that, we're ready to install the core software that powers Jasper.

<h2 class="linked" id='installing-sphinx'><a href="#installing-sphinx" title="Permalink to this headline">Install Pocketsphinx, CMUCLMTK, and Phonetisaurus</a></h2>

Jasper uses Pocketsphinx for voice recognition. Let's download and unzip sphinxbase and pocketsphinx:

{% highlight bash %}
wget http://downloads.sourceforge.net/project/cmusphinx/sphinxbase/0.8/sphinxbase-0.8.tar.gz
wget http://downloads.sourceforge.net/project/cmusphinx/pocketsphinx/0.8/pocketsphinx-0.8.tar.gz
tar -zxvf sphinxbase-0.8.tar.gz
tar -zxvf pocketsphinx-0.8.tar.gz
{% endhighlight %}

Now we build and install sphinxbase:

{% highlight bash %}
cd ~/sphinxbase-0.8/
./configure --enable-fixed
make
sudo make install
{% endhighlight %}

and pocketsphinx:

{% highlight bash %}
cd ~/pocketsphinx-0.8/
./configure
make
sudo make install
{% endhighlight %}

Restart your Pi.

<h2 class="linked" id='configure-wireless'><a href="#configure-wireless" title="Permalink to this headline">Configure Wireless</a></h2>


To configure wireless:

{% highlight bash %}
sudo apt-get install dhcp3-server hostapd udhcpd
{% endhighlight %}

Insert the following lines to the end of /etc/dhcp/dhcpd.conf:

{% highlight bash %}
ddns-update-style interim;
default-lease-time 600;
max-lease-time 7200;
authoritative;
log-facility local7;
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.5 192.168.1.150;
}
{% endhighlight %}

And the following to the end of /etc/udhcpd.conf:

{% highlight bash %}
start 192.168.42.2 # This is the range of IPs that the hostspot will give to client devices.
end 192.168.42.20
interface wlan0 # The device uDHCP listens on.
remaining yes
opt dns 8.8.8.8 4.2.2.2 # The DNS servers client devices will use.
opt subnet 255.255.255.0
opt router 192.168.42.1 # The Pi's IP address on wlan0 which we will set up shortly.
opt lease 864000 # 10 day DHCP lease time in seconds
{% endhighlight %}

Comment this line in sudo vim /etc/default/udhcpd:

{% highlight bash %}
#DHCPD_ENABLED="no"
{% endhighlight %}

Then run:

{% highlight bash %}
sudo ifconfig wlan0 192.168.42.1
{% endhighlight %}

Next, we will install Jasper itself.

<h2 class="linked" id='install-jasper'><a href="#install-jasper" title="Permalink to this headline">Install Jasper</a></h2>


In the home directory of your Pi, clone the Jasper source code:

{% highlight bash %}
git clone https://github.com/shbhrsaha/jasper-client.git jasper
{% endhighlight %}

Jasper requires various Python libraries that we can install in one line with:

{% highlight bash %}
sudo pip install -r jasper-client/client/requirements.txt
{% endhighlight %}

Run `crontab -e`, then add the following lines:

{% highlight bash %}
@reboot /home/pi/jasper/boot/boot.sh;
*/1 * * * * ping -c 1 google.com
{% endhighlight %}

Copy /usr/local/bin binaries:

{% highlight bash %}
mkdir bin
scp * pi@192.168.2.3:./bin/
sudo cp * /usr/local/bin/
{% endhighlight %}

Copy /usr/local/lib:

{% highlight bash %}
mkdir lib
scp * pi@192.168.2.3:./lib/
sudo cp * /usr/local/lib/
{% endhighlight %}

Copy the phonetisaurus folder to the home directory:

{% highlight bash %}
mkdir phonetisaurus
scp * pi@192.168.2.3:./phonetisaurus/
{% endhighlight %}

Set permissions everywhere:

{% highlight bash %}
chmod 777 -R *
{% endhighlight %}

At this point, we've installed Jasper and all the necessary software to run it. Before we start playing around, though, we need to configure Jasper and provide it with some basic information.


<h2 class="linked" id='configure-jasper'><a href="#configure-jasper" title="Permalink to this headline">Configuring the Jasper Client</a></h2>


<h3 class="linked" id='generating-profile'><a href="#generating-profile" title="Permalink to this headline">Generating a user profile</a></h3>

In order for Jasper to accurately report local weather conditions, send you text messages, and more, you first need to generate a user profile.

To facilitate the process, run the profile population module that comes packaged with Jasper

{% highlight bash %}
python populate.py
{% endhighlight %}

The process is fairly self-explanatory: fill in the requested information, or hit 'Enter' to defer at any step. The resulting profile will be stored as a [YML](http://fdik.org/yml/) file at _profile.yml_.

**Important**: _populate.py_ will request your Gmail password. Of course, this is purely optional. Providing this password will allow Jasper to report on incoming emails, etc., but be aware that the password will be stored as plaintext in _profile.yml_.

<h3 class="linked" id='facebook-tokens'><a href="#facebook-tokens" title="Permalink to this headline">Facebook tokens</a></h3>

To enable Facebook integration, Jasper requires an API key. Unfortunately, this is a multi-step process that requires manual editing of _profile.yml_. Fortunately, it's not particularly difficult.

1. Go to [https://developers.facebook.com](https://developers.facebook.com) and select 'Apps', then 'Create a new app'.
2. Give your app a name, category, etc. The choices here are arbitrary.
3. Go to the [Facebook Graph API Explorer](https://developers.facebook.com/tools/explorer/) and select your App from the drop down list in the top right (the default choice is 'Graph API Explorer').
4. Hit 'Get Access Token' with the default permissions.
5. Take the resulting API key and add it to _profile.yml_ in the following format:

        ...
        prefers_email: false
        timezone: US/Eastern
        keys:
            FB_TOKEN: abcdefghijklmnopqrstuvwxyz


Note that similar keys could be added when developing other modules. For example, a Twitter key might be required to create a Twitter module and so forth.

<h3 class="linked" id='att-tokens'><a href="#att-tokens" title="Permalink to this headline">AT&amp;T tokens</a></h3>

Similarly, Jasper needs an API key for accessing AT&T's speech-to-text service. This allows Jasper to perform speech-to-text translation with improved accuracy.

1. Go to [https://developer.att.com/apis/speech](https://developer.att.com/apis/speech) and follow the registration instructions.
2. After your account has been activated, hit 'Setup New App' in the dashboard window. Fill out the fields as required.
3. Grab the App Key in the resulting window and add it to _profile.yml_, which will now look like:

        ...
        prefers_email: false
        timezone: US/Eastern
        keys:
            FB_TOKEN: abcdefghijklmnopqrstuvwxyz
            ATT_KEY: abcdefghijklmnopqrstuvwxyz

With that, you're good-to-go from a configuration standpoint.

<h2 class="linked" id='software-architecture'><a href="#software-architecture" title="Permalink to this headline">Software Architecture</a></h2>

Having installed the required libraries, it is worth taking a moment to understand how they interact and how the client code is architected.

Jasper utilizes a number of open source libraries to function. Pocketsphinx performs speech recognition via Python bindings to the CMUSphinx engine. Jasper’s voice is owed to the popular TTS program, Espeak. Phonetisaurus and CMUCLMTK enable Jasper to generate dictionaries and language models on-the-fly based on the custom module vocabularies. HostAPD helps to broadcast an ad-hoc wireless network to assist wifi configuration. Mopidy enables streaming from Spotify, for those users who wish to use the module.

The client architecture is organized into a number of different components:

![Jasper Client Architecture](architecture.png)

Main.py is the program that orchestrates all of Jasper. It creates mic, profile, and conversation instances. Conversation receives mic and profile from main then creates a notifier and brain. Brain receives the mic and profile originally descended from main and loads all the interactive components into memory. Brain is essentially the interface between developer-written modules and the core framework. Each module must implement isValid() and handle() functions and define a WORDS list.

To learn more about how Jasper interactive modules work and how to write your own, check out the [API guide](/documentation/api)