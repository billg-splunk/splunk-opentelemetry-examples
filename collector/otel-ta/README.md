# OTel TA

The OTel TA is a way for customers who already use the Splunk Universal Forwarder (UF) with the Deployment Server to distribute the Open Telemetry Collector and config files.

This example is meant for someone with minimal Splunk knowledge.

It will assume a setup needing three configurations:
* A Splunk OTel Collector collecting only the operating system
* A Splunk OTel Collector collecting the operating system and Apache
* A Splunk OTel Collector collecting the operating system and Tomcat

If you need a configurating that includes multiple apps, the current recommendation is to have a configuration with both in one (vs. sending the base, apache, and tomcat), for reasons that will become obvious once you see how this configuration is set up.

## Outline

The following represents the set of steps needed to achieve this:
* Install Splunk
* Install UF on 3 endpoints
* Setup Deployment Server
  * Deploy OTel Collector with base config
  * Deploy configurations for Apache and Tomcat
* Test Deployments
  * Configure all endpoints to use base config
  * Configure UF2 (Apache) and UF3 (Tomcat)

### Install Splunk
For this example we will simply use docker to run our server. NOTE this document is not reflecting best practices for deploying Splunk. For example we are deploying without a proper cert, and we are using simple passwords that should not be used in production.

After installing docker (and docker compose), create a docker compose file (`compose.yaml`):
```
version: "3.3"
services:
  splunk:
    ports:
      - 8000:8000
      - 8089:8089
      - 9997:9997
    environment:
      - SPLUNK_PASSWORD=password
      - SPLUNK_START_ARGS=--accept-license
    stdin_open: true
    tty: true
    image: splunk/splunk:latest
    volumes:
      - /splunk/etc:/opt/splunk/etc
      - /splunk/var:/opt/splunk/var
networks: {}
```

You will need to create the folder `/splunk` on your system; you can substitute this for another folder, just replace the first part of the volume lines with the folder you use.

Then launch it from the same directory the `compose.yaml` is in with:
```
docker compose up -d
```

### Install UF on 3 endpoints
We'll use 3 linux boxes for the purposes of this setup.

On each box run the following commands. You can update the version of the UF, and change the IP address to use the one your Splunk server is running on:
```
sudo useradd -m splunkfwd
sudo groupadd splunkfwd
cd /opt
sudo wget -O splunkforwarder.tgz "https://download.splunk.com/products/universalforwarder/releases/9.3.1/linux/splunkforwarder-9.3.1-0b8d769cb912-Linux-x86_64.tgz"
sudo tar xvzf splunkforwarder.tgz
sudo chown -R splunkfwd:splunkfwd /opt/splunkforwarder
sudo /opt/splunkforwarder/bin/splunk start --accept-license
<enter admin and password>
sudo /opt/splunkforwarder/bin/splunk add forward-server 192.168.7.240:9997
sudo /opt/splunkforwarder/bin/splunk set deploy-poll 192.168.7.240:8089
```

then wait for a bit (can take several minutes to appear)

You can check the logs with: sudo tail -f /opt/splunkforwarder/var/log/splunk/splunkd.log

You are looking for a line like this:
10-15-2024 13:00:39.238 -0400 INFO  AutoLoadBalancedConnectionStrategy [1689 TcpOutEloop] - Connected to idx=192.168.7.240:9997:1, pset=0, reuse=0. autoBatch=1

Then refresh "Forwarder Management". It may take a few more seconds to appear.

![Intial Forwarder Screen](img/forwarder_management_initial.png)

### Deploy OTel Collector with base config

Next we need to add the contents that we want to push to all of the systems. We will start by simply downloading the TA and making the minimal changes needed to start collecting metrics to send to Splunk Observability Cloud.

First, download the TA from [here](https://splunkbase.splunk.com/app/7125). (You will need a splunk.com account to download it; this is free to setup.)

Our docker setup is exposing the `etc` folder to the host machine. So you can place this archive inside the `/splunk/etc/deployment-apps` folder. (NOTE: If you used a different local folder than `/splunk` then adjust accordingly.)

We need to change a few files to make this work:

```
# Create a new file, /Splunk_TA_otel/local/access_token
Paste your ingest token in this file.

# /Splunk_TA_otel/default/inputs.conf
Set the following (unless your realm is us0). For example for us1:
splunk_api_url=https://api.us1.signalfx.com
splunk_ingest_url=https://ingest.us1.signalfx.com
splunk_trace_url=https://ingest.us1.signalfx.com/v2/trace
splunk_realm=us1
```

To make sure this directory can be written to, use `chown -R` to match the existing files in this directory. For example:
```
sudo chown -R 41812:41812 Splunk_TA_otel
```

You can confirm the TA has been deployed to splunk on the apps tab:
![Forwarder Management - Apps](img/forwarder_management_apps.png)

Next, edit the app and make sure after installation `Enable App` and `Restart Splunkd` are checked.

![Forwarder Management - Restart Splunk](img/forwarder_management_app_restart_splunk.png)

Now we have the configuration set, we need to tell Splunk which collectors to send it to. Switch to the `Server Classes` tab and create a new server class named `otel_base_linux`. Make the following settings:
* For apps, include `Splunk_TA_otel`
* For clients, include `uf*`

The result should look like the following:
![Forwarder Management - Server Class Defined](img/forwared_management_server_class_base.png)

If all goes well you will get collectors in Splunk Observability. If you are running hosts locally you will find them under `Infrastructure >  My Data Center > Hosts`. If they are in the cloud then they will be under the appropriate cloud tile:
* `AWS EC2`
* `Azure Virtual Machines`
* `Google Cloud Platform Compute Engine`
* etc.

For example:
![Result in O11y Cloud](img/o11y_cloud_base.png)

### Deploy configurations for Apache and Tomcat
TBD

## Tips and Tricks
* If you aren't sure what is going on you can always uninstall the app and then redeploy it to the server class(es)
* Make sure the app includes restarting splunkd