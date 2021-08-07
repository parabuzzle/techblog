---
layout: post
title:  Raspberry Pi for Halloween Props
date:   2017-10-27 12:00:00
categories: Raspberry-Pi
#preview: ...
disqus_id: 19
---

For the past five years my wife and I have built increasingly elaborate props for our yard and charity haunted house. We make a great team because I enjoy building the mechanical, electronic, and software pieces to make things run and she, as a designer, enjoys making the mechanisms I build look as real as possible. When I started, I started out small. I built simple mechanisms using things like salvaged windshield wiper motors constantly on. That quickly turned into a requirement to control when it was running. So then I learned how to develop arduino code to drive the things. Then, when I found that I needed more complex control like multi-threading, sound, and networking; I turned to using Raspberry Pi boxes.

The problem with the Raspberry PI approach is that the OS writes a lot of things to the file system actively and if you don't cleanly shutdown (like pull the power cord) you run the (relatively high risk) of corrupting your filesystem and turning it into a brick until you re-format the SD card. This resulted in many curse words, and many versions of image files. Last year I abandoned Raspberry Pi and went back to Arduino and even native ATmel programming for most everything. I found myself designing circuits, getting boards printed, flashing ATTiny85s, and overall building some neat stuff that used ATTiny85s as pure functions that executed sequences that were triggered by other ATTiny85s. Mostly because I have gotten into pneumatics that required complex firing sequences to create fluid realistic movements.. but I digress.

So why am I writing about Raspberry Pi and not Arduino and ATmel programming? Well, one of my favorite props is a thunder and lightning controller for the front yard. I built it for Raspberry Pi and have no care or time to re-implement it in Arduino. What the prop does is loop spooky graveyard background sound and then randomly fires a big industrial strobe light flash followed by a loud thunder crash. I built it in ruby and you can see the code here: [https://github.com/parabuzzle/meatpi/blob/master/bin/yardfx.rb][https://github.com/parabuzzle/meatpi/blob/master/bin/yardfx.rb]. It uses a thread for the background sound and another thread to run thunder/lightning routine. In the past, we ran it on Halloween only. So the clean shutdown requirement wasn't a problem because at the end of the night, I would login and run `sudo halt`. This year, we decided to run it every night. So that high-touch model wasn't going to work for me.

What to do? Do I build it into an Arduino? I would need another sound module (I'm out). Why? I already have a Raspberry Pi that works. So I decided to make it work. First thing that needs to be solved is the file corruption problem. I accomplished this by making the filesystem readonly using this guide: [https://hallard.me/raspberry-pi-read-only/][https://hallard.me/raspberry-pi-read-only/].

The TL;DR of that is basically this:

~~~
apt-get update
apt-get remove --purge wolfram-engine triggerhappy anacron logrotate dphys-swapfile xserver-common lightdm
insserv -r x11-common
apt-get autoremove --purge
apt-get install busybox-syslogd # no more syslog.. run readlog to read logs...
dpkg --purge rsyslog

rm -rf /var/lib/dhcp/ /var/run /var/spool /var/lock /etc/resolv.conf
ln -s /tmp /var/lib/dhcp
ln -s /tmp /var/run
ln -s /tmp /var/spool
ln -s /tmp /var/lock
touch /tmp/dhcpcd.resolv.conf
ln -s /tmp/dhcpcd.resolv.conf /etc/resolv.conf

insserv -r bootlogs; insserv -r console-setup
~~~

Then edit `/boot/cmdline.txt` and add `fastboot noswap ro`

Next change `/etc/cron.hourly/fake-hwclock` to:

~~~
#!/bin/sh
#
# Simple cron script - save the current clock periodically in case of
# a power failure or other crash

if (command -v fake-hwclock >/dev/null 2>&1) ; then
  mount -o remount,rw /
  fake-hwclock save
  mount -o remount,ro /
fi
~~~

Next change the `driftfile` location in `/etc/ntp.conf` to `/var/tmp/ntp.drift`

lastly change `/etc/fstab` and add `,ro` to all the block devices

Ok.. now the Pi power can be killed without corrupting the SD card. (Why this isn't an option built into to rasbian is beyond me)

The next thing I needed to do is get the yardfx.rb running when the box boots. At first I put it into an init script and just have it start on boot.. but that proved to be a problem if the application crashed or the WEMO running the yard didn't kill the front yard decorations. (The thunder and lighting running all night annoys neighbors). I decided to fix both problems by just having a run script that gets fired on a cron every minute.

The script checks the time and only runs the yardfx.rb program between the hours of 6p and 11p daily. It also handles killing it off if its not supposed to be running and will not start a second one if its already running.


Here is the `run_yard.sh` script:

~~~bash
#!/bin/bash

source $HOME/.profile
export GEM_HOME=/usr/local/lib/ruby/gems/2.2.0/gems

start_hour=18 # 6p
end_hour=21   # end at 10:59p

current_hour=`date +"%H"`

kill_yardfx () {
  if [ `ps auxx | grep yard | grep -v grep | grep -v run_yard.sh | wc -l` -gt 0 ]; then
    echo "found running yardfx"
    echo " ...killing it!"
    ps auxx | grep yard | grep -v grep | grep -v run_yard.sh | awk '{ print $2 }' | xargs kill -9
  fi
}

if [ $current_hour -lt $start_hour ]; then
  echo "Too too early to start the yardfx"
  echo "Start hour: $start_hour, current hour: $current_hour"
  kill_yardfx
  exit 0
fi

if [ $current_hour -gt $end_hour ]; then
  echo "Too late to run yardfx"
  echo "End hour: $end_hour, current_hour: $current_hour"
  kill_yardfx
  exit 0
fi

if [ `ps auxx | grep meatpi | grep -v grep | wc -l` -gt 0 ]; then
  echo "already running a meatpi"
  exit 0
fi
cd /home/pi/meatpi/bin
/usr/local/bin/ruby ./yardfx.rb 2>&1 > /dev/null &
~~~

All I needed to do is add this to the `/etc/crontab`:

~~~
* * * * * root /bin/bash /root/run_yard.sh
~~~

OOPS.. almost. Making the filesystem readonly causes the crontab to disappear on reboots. So one more final hack is needed to get it working.. In the cron init script `/etc/init.d/cron` in the start section, I just added this line:

~~~
crontab -u root /etc/crontab
~~~

Now it all works.

Bottomline, if you're going to build reliable complex automated halloween props, learn to use micro controllers and stick with it. If you want to use a high-order system like a Raspberry Pi, know the pitfalls and be ready to hack the hell out of it to make it robust.
