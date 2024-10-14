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
* Pre-requisite: Install Splunk
* Install Deployment Server
* Install UF on 3 endpoints
* Setup Deployment Server
  * Deploy OTel Collector with base config
  * Deploy configurations for Apache and Tomcat
* Test Deployments
  * Configure all endpoitns to use base config
  * Configure UF2 (Apache) and UF3 (Tomcat)
