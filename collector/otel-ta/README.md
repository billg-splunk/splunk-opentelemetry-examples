# OTel TA

The OTel TA is a way for customers who already use the Splunk Universal Forwarder (UF) with the Deployment Server to distribute the Open Telemetry Collector and config files.

This example is meant for someone with minimal Splunk knowledge but has some background on configuring the open telemetry collector. It will walk through configuring the deployment server and installing and configuring the open telemetry collector. It also assumes light docker knowledge, although the steps for use are documented.

For the example we need three configurations:
* A Splunk OTel Collector collecting only the operating system (default config)
* A Splunk OTel Collector collecting the operating system and apache
* A Splunk OTel Collector collecting the operating system and nginx

If you need a configurating that includes multiple apps, the current recommendation is to have a configuration with both in one (vs. sending the base, apache, and nginx), for reasons that will become obvious once you see how this configuration is set up.

## Outline

The following represents the set of steps needed to achieve this:
- [OTel TA](#otel-ta)
  - [Outline](#outline)
  - [Installs](#installs)
    - [Install docker](#install-docker)
    - [Install Splunk](#install-splunk)
    - [Install UF on 3 endpoints](#install-uf-on-3-endpoints)
  - [Deploy OTel Collector with base config](#deploy-otel-collector-with-base-config)
    - [Test the base configuration](#test-the-base-configuration)
  - [Deploy configurations for apache and nginx](#deploy-configurations-for-apache-and-nginx)
    - [Test the configurations for all 3 servers](#test-the-configurations-for-all-3-servers)
  - [Tips and Tricks](#tips-and-tricks)

## Installs

### Install docker
Follow [the instructions](https://docs.docker.com/engine/install/) to install docker. These instructions were tested on a linux host, and may need to be amended slightly for a different os or architecture.

### Install Splunk
For this example we will simply use docker to run our server. NOTE this document is not reflecting best practices for deploying Splunk. The install doesn't deploy a certificate, and we are using simple passwords that should not be used in production, among other shortcuts.

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

What is notable about this configuration:
* We expose port 8000 (web ui), 8089 (management endpoint) and 9997 (deployment server endpoint)
* We are mapping `/etc` (where the apps are deployed) and `/var` (where logs are accessed) to simplify management and preserve them when the container is restarted

In order to use this configuration you will need to create the folder `/splunk` on your host system; you can substitute this for another folder, just replace the first part of the volume lines with the folder you use.

Then launch it from the same directory the `compose.yaml` is in with:
```
docker compose up -d
```

### Install UF on 3 endpoints
We'll use 3 linux (ubuntu) systems for the purposes of this setup. Note the ip address of your host, as you will need it for this section. Our example uses `192.168.7.240`.

On each box run the following commands. You can update the version of the UF, and change the IP address as noted above:
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

## Deploy OTel Collector with base config

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

### Test the base configuration

Now we have the configuration set, we need to tell Splunk which collectors to send it to, as well as restart after running.

First, switch to the `Server Classes` tab and create a new server class named `otel_base_linux`. Make the following settings:
* For apps, include `Splunk_TA_otel`
* Make sure the app will restart splunkd. (You will need to edit the app now to do that)
* Finally set the clients, include `uf*`

The result should look like the following:
![Forwarder Management - Server Class Defined](img/forwared_management_server_class_base.png)

If all goes well you will get collectors in Splunk Observability. If you are running hosts locally you will find them under `Infrastructure >  My Data Center > Hosts`. If they are in the cloud then they will be under the appropriate cloud tile:
* `AWS EC2`
* `Azure Virtual Machines`
* `Google Cloud Platform Compute Engine`
* etc.

For example:
![Result in O11y Cloud](img/o11y_cloud_base.png)

## Deploy configurations for apache and nginx

Next we want to look at configuring configuration deployments for various apps.

Each configuration needs to stand on its own. For example if a system has apache and nginx you would need a configuration containing both receivers. But it can be separated from the install files.

For our example we will create one configuration for apache and one for nginx, and push them to uf2 and uf3 (respectively).

To set this up let's configure the following folder structure under `/etc/deployment-apps`, using the files from [here](configuration-files):
```
/etc
  /deployment-apps
    /Splunk_TA_otel
      <install files and default configuration>
    /Splunk_TA_app_config_apache
      /configs
        otel-apache.yaml
      /local        
        inputs.conf
    /Splunk_TA_app_config_nginx
      /configs
        otel-nginx.yaml
      /local        
        inputs.conf
```

Let's talk about the naming of the folders (apps). The deployment server uses [lexographical order](https://lantern.splunk.com/Splunk_Success_Framework/Data_Management/Naming_conventions). In addition the `local` directory supercedes the `default` directory. So by putting updates in the `/local/inputs.conf` file we are overriding the default configuration that is coming with the base install. 

For this example we are only setting a new config file (i.e. `otel-apache.yaml`) in our `inputs.conf` file, but we could also make other changes (like pointing to a separate `access_token` file) and anything else we want to override.

Let's first deploy these two apps.

On uf2, deploy apache:

```
sudo apt update
sudo apt install apache2
```

On uf3, deploy nginx:

```
sudo apt update
sudo apt install nginx
sudo vi /etc/nginx/sites-available/default
```

and then add the following to `server` block, after the 2 listen statements, to add a status page the `nginx` receiver will collect from:
``` conf
location /status {
  stub_status;
  allow all;
}
```

Finally we can test the nginx configuration with `nginx -t`. Then you will either need to start or restart nginx:
* To start: `nginx`
* If you get errors that the address is already in use: `nginx -s reload`

And you can test the status with:
```
curl http://localhost/status
```
### Test the configurations for all 3 servers

Now that our `uf2` and `uf3` are running `apache` and `nginx`, respectively, we can remap the clients to send the new configurations:
* uf1 - base
* uf2 - base and apache
* uf3 - base and nginx

It should look like this:
![Final Forwarder Management](img/forwarder_management_final.png)

And in Splunk Observability you will find metrics for apache and nginx, such as:
* apache: `apache.traffic`, `apache.uptime`, `apache.requests`, etc.
* nginx: ``

## Tips and Tricks
* In our example we simply deployed the otel base, which included both linux and windows files. These could be separated into apps relevant for each operating system.
* Make sure the app includes restarting splunkd
* If you aren't sure what is going on you have a few options
  * On the Splunk Server you can run `/opt/splunk/bin/splunk reload deployment-server` to redeploy
    * You can run this with the option `-class [serverclass name]` to do this for just a specific server class
  * Alternatively you can uninstall the app and then set it up again
* The following logs are useful to monitor
  * `/opt/splunk/var/log/splunk/Splunk_TA_otel.log`: For the lifecycle of the TA and OTel Collector
  * `/opt/splunk/var/log/splunk/otel.log`: For the otel collector itself