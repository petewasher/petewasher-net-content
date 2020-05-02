## 
It's amazing how expensive it can appear to be to buy a digital temperature 
sensor. You can put one together yourself for very little!

## Setup
Assemble some bits:
* Pi Zero W
* SD Card
* DHT22 Sensor and some cables. I bought [this one](#).

Grab the latest Raspbian from the RPi Foundation, or use a noobs image to 
get your Pi Zero installed. Remember to add a empty file called `ssh` to 
your `/boot` partition to enable SSH Server, and fill in a `wpa_supplicant.conf`
if you need to setup your wifi:

```
```

Now your pi should arrive on the network and you will be able to SSH into it
using `ssh pi@raspberrypi`. Do the updates etc., and we can get the DHT sensor
setup! 

## Cheapest DHT22 Wifi Sensor
Wire up your Pi as follows:

PICTURE

This will give your sensor some power, and allow us to communicate with it. 
Next you'll need to get the appropriate libraries installed. Since we're 
doing all this in Python, you can run the following commands to get going:

```
```

You'll also need to run `raspi-config` to turn on the appropriate interfaces
on your Pi:

PICTURE

If you followed my other guide on how to setup your own InfluxDB server, you
can start pushing information into your database. Here's a full script
which you can adapt as you need:

```
```

I have this script scheduled on a Cron job to run every few minutes. 
