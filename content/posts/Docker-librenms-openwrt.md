+++ 
draft = true
date = 2024-09-05T16:55:35+02:00
title = ""
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

# Docker, LibreNMS and OpenWRT

I recently bought a [Flint2 router](https://www.gl-inet.com/products/gl-mt6000/) from GL-Inet, It is an [OpenWRT](https://openwrt.org/) device that works fine but I'd like to extend the monitoring part and using some modern tools instead of the native ones provided by OpenWRT itself. You can use vnstats, bmon, iptraf, iftop and many others but nowadays we have additional solutions.

Before writing this guide, I tried different solutions:

[Grafana](https://grafana.net) + [Prometheus](https://Prometheus.io)
[Zabbix](https://zabbix.com)
[Checkmk](https://checkmk.com/)

As you can image, every solution has pros and cons but I don't want to use this blog post for describing them: you should choose based on your need. The first question is: which kind of monitoring I need for OpenWRT? Metrics? Link Disconnections? Bandwidth?  What else?
 
In my case I needed to monitor the metrics and I also wanted an alarm system that would notify me via email/Telegram in case of malfunction.
All of the previous mentioned software are able to do that but I have chose to go with [LibreNMS](https://www.librenms.org/): it is a complete network monitoring system with many features.
[Here](https://docs.librenms.org/) you can find the documentations.

I'm using Docker for other project, so I preferred to go with it (the Docker installation part is out of scope).

From https://docs.librenms.org/Installation/Docker/ 

    mkdir librenms
    cd librenms 
    wget https://github.com/librenms/docker/archive/refs/heads/master.zip 
    unzip master.zip 
    cd docker-master/examples/compose

Now it's time to configure the (env) variables for various software like MariaDB, redis, smtp..
Edit the files *librenms.env* , *msmtpd.env* and *.env* adjusting the values and run

    sudo docker compose -f compose.yml up -d
(if you want to ue docker-compose, remove 'name' tag at the beginning of the file)
Wait a couple of minutes and try to connect to http://localhost:8000 It will ask to create ad admin username, password and email address.

At that point we need to configure the SNMP part on the OpenWRT router. Yes, because we will getting the Flint2 data trough the snmp protocol.
You can simply connect using ssh to your router and type the following commands:

    wget -O /etc/librenms/wlClients.sh https://raw.githubusercontent.com/librenms/librenms-agent/master/snmp/Openwrt/wlClients.sh 
    wget -O /etc/librenms/wlFrequency.sh https://raw.githubusercontent.com/librenms/librenms-agent/master/snmp/Openwrt/wlFrequency.sh 
    wget -O /etc/librenms/wlInterfaces.txt https://raw.githubusercontent.com/librenms/librenms-agent/master/snmp/Openwrt/wlInterfaces.txt 
    wget -O /etc/librenms/wlNoiseFloor.sh https://raw.githubusercontent.com/librenms/librenms-agent/master/snmp/Openwrt/wlNoiseFloor.sh 
    wget -O /etc/librenms/wlRate.sh https://raw.githubusercontent.com/librenms/librenms-agent/master/snmp/Openwrt/wlRate.sh 
    wget -O /etc/librenms/wlSNR.sh https://raw.githubusercontent.com/librenms/librenms-agent/master/snmp/Openwrt/wlSNR.sh 
    wget -O /etc/librenms/distro https://raw.githubusercontent.com/librenms/librenms-agent/master/snmp/distro 
    chmod +x /etc/librenms/*.sh 
    chmod +x /etc/librenms/distro

Check the content of wlInterfaces.txt file, it should be like this:

    wlan0,wl-2.4G
    wlan1,wl-5.0G
Now add the script previously downloaded at the end of the configuration file */etc/config/snmpd*

    config extend
            option name distro
            option prog '/etc/librenms/distro'
    config extend
            option name hardware
            option prog '/bin/cat'
            option args '/sys/firmware/devicetree/base/model'
    config extend
            option name     interfaces
            option prog     "/bin/cat /etc/librenms/wlInterfaces.txt"
    config extend
            option name     clients-wlan0
            option prog     "/etc/librenms/wlClients.sh wlan0"
    config extend
            option name     clients-wlan1
            option prog     "/etc/librenms/wlClients.sh wlan1"
    config extend
            option name     clients-wlan
            option prog     "/etc/librenms/wlClients.sh"
    config extend
            option name     frequency-wlan0
            option prog     "/etc/librenms/wlFrequency.sh wlan0"
    config extend
            option name     frequency-wlan1
            option prog     "/etc/librenms/wlFrequency.sh wlan1"
    config extend
            option name     rate-tx-wlan0-min
            option prog     "/etc/librenms/wlRate.sh wlan0 tx min"
    config extend
            option name     rate-tx-wlan0-avg
            option prog     "/etc/librenms/wlRate.sh wlan0 tx avg"
    config extend
            option name     rate-tx-wlan0-max
            option prog     "/etc/librenms/wlRate.sh wlan0 tx max"
    config extend
            option name     rate-tx-wlan1-min
            option prog     "/etc/librenms/wlRate.sh wlan1 tx min"
    config extend
            option name     rate-tx-wlan1-avg
            option prog     "/etc/librenms/wlRate.sh wlan1 tx avg"
    config extend
            option name     rate-tx-wlan1-max
            option prog     "/etc/librenms/wlRate.sh wlan1 tx max"
    config extend
            option name     rate-rx-wlan0-min
            option prog     "/etc/librenms/wlRate.sh wlan0 rx min"
    config extend
            option name     rate-rx-wlan0-avg
            option prog     "/etc/librenms/wlRate.sh wlan0 rx avg"
    config extend
            option name     rate-rx-wlan0-max
            option prog     "/etc/librenms/wlRate.sh wlan0 rx max"
    config extend
            option name     rate-rx-wlan1-min
            option prog     "/etc/librenms/wlRate.sh wlan1 rx min"
    config extend
            option name     rate-rx-wlan1-avg
            option prog     "/etc/librenms/wlRate.sh wlan1 rx avg"
    config extend
            option name     rate-rx-wlan1-max
            option prog     "/etc/librenms/wlRate.sh wlan1 rx max"
    config extend
            option name     noise-floor-wlan0
            option prog     "/etc/librenms/wlNoiseFloor.sh wlan0"
    config extend
            option name     noise-floor-wlan1
            option prog     "/etc/librenms/wlNoiseFloor.sh wlan1"
    config extend
            option name     snr-wlan0-min
            option prog     "/etc/librenms/wlSNR.sh wlan0 min"
    config extend
            option name     snr-wlan0-avg
            option prog     "/etc/librenms/wlSNR.sh wlan0 avg"
    config extend
            option name     snr-wlan0-max
            option prog     "/etc/librenms/wlSNR.sh wlan0 max"
    config extend
            option name     snr-wlan1-min
            option prog     "/etc/librenms/wlSNR.sh wlan1 min"
    config extend
            option name     snr-wlan1-avg
            option prog     "/etc/librenms/wlSNR.sh wlan1 avg"
    config extend
            option name     snr-wlan1-max
            option prog     "/etc/librenms/wlSNR.sh wlan1 max"

Restart the snmpd with */etc/init.d/snmpd restart*

Connect to webUI of LibreNMS: http://localhost:8000 and the the device or simply navigate to http://localhost:8000/addhost
Specify the ip of the router, the snmp community name, the version of snmp and the upd port (161)  

Here we go! 
Wait a couple of minutes and you will see some interesting data of your OpenWRT router.
If you are interested in configuring alerts you can check out this video: https://www.youtube.com/watch?v=J7ZBs2ut-Ho&t=98s
