---
title: "Setting up and running LF-Edge's Fledge (from source) on a Raspberry Pi 4"
date: 2021-07-29T15:48:16+02:00
draft: false

author: "Shan"
tags: ["IoT", "Edge Computing", "Open Source", "Raspberry Pi", "Linux Foundation Projects"]
categories: ["Technology"]

toc:
  enable: true
  auto: false
---
<!--more-->

## Fledge

> Fledge is an open source framework and community for the industrial edge focused on critical operations, predictive maintenance, situational awareness and safety. Fledge is architected to integrate Industrial Internet of Things (IIoT), sensors and modern machines with the cloud and existing “brown field” systems like historians, DCS (Distributed Control Systems), PLC (Program Logic Controllers) and SCADA (Supervisory Control and Data Acquisition). All sharing a common set of administration and application APIs.

I wanted to give [Fledge](https://github.com/fledge-iot) a try to see how one can bring an open-source IoT/IIoT Stack to a quick realization.


## Getting Hands Dirty already!

I had some [problems trying to install Fledge using `apt`](https://github.com/fledge-iot/fledge/issues/439) so I decided to install Fledge from Source.


### Hardware

__Device__: __Raspberry Pi 4 Model B__
__Spec__: 2GB RAM, 32GB Micro SD Card
__Operating System__: __Ubuntu LTS 20.04.2__

### Setup

I am going to just clone stuff in my home directory

1. Clone the [fledge Repository](https://github.com/fledge-iot/fledge)

        $ git clone https://github.com/fledge-iot/fledge.git && cd fledge/

2. Install the Dependencies (in the root directory of fledge)

         $ sudo ./requirements.sh

3. Let's install fledge on the Pi

        $ sudo make install

    grab something to munch on till the installation is complete

4. The installation will add the files and binaries in to the following directory: `/usr/local/fledge/`. You can export this path:

        $ export FLEDGE=/usr/local/fledge

5. start Fledge and check the status using the following:

        $ $FLEDGE/bin/fledge start
        $ $FLEDGE/bin/fledge status

6. Checkout the Fledge UI by hitting the Pi's IP Address: `http://<IP_ADDRESS_of_PI>`

At this point of time, Fledge is very _bare-metal_ which implies there are lot of plugins that are missing. We begin by installing a very basic _simulated_ plugin.

### South-Plugin Installation

#### Sinusoid 

Since we are doing things by source, we need to clone the plugin repository. We start with a very basic `sinusoidal` South Plugin which will generate sine waves to the Fledge Core.

1. clone the [`fledge-south-sinusoid` plugin repository](https://github.com/fledge-iot/fledge-south-sinusoid)

        $ cd 
        $ git clone https://github.com/fledge-iot/fledge-south-sinusoid.git && cd fledge-south-sinusoid/

2. we just need to copy the source code of the sine-wave generator to the directory path of plugins for Fledge

        $ # we are currently in the root directory of fledge-south-sinusoid
        $ sudo cp -r python/fledge/plugins/south/sinusoid $FLEDGE/python/fledge/plugins/south

You can refresh the UI and click on the __South__ Menu and you will be able to see `sinusoid` as a __South Plugin__

#### CoAP

Let's get one more Plugin into Fledge: CoAP plugin

1. clone the [`fledge-south-coap` plugin repository](https://github.com/fledge-iot/fledge-south-coap)

        $ cd 
        $ git clone https://github.com/fledge-iot/fledge-south-coap.git && cd fledge-south-coap/

2. Let's copy the plugin's code to Fledge as we did previously

        $ # we are currently in the root directory of fledge-south-coap
        $ sudo cp -r python/fledge/plugins/south/coap $FLEDGE/python/fledge/plugins/south

3. Copy the python dependencies that are necessary to execute the plugin and install them:

        $ sudo cp -r python/requirements-coap.txt $FLEDGE/python/
        $ sudo pip3 - r $FLEDGE/python/requirements-coap.txt

4. Since the code is in the `/usr/local/` directory we need to provide root privileges to the coap plugin for execution by the Fledge Core

        $ sudo chown -R root:root $FLEDGE/python/fledge/plugins/south/coap

That's it ! Now you have we two South Plugins in our Fledge Application that can provide us some simulated data!

Your UI __South__ Menu should look like the following:

{{< figure src="/images/technology/fledge-iot/fledge-south-plugins.png" title="Sinusoid & CoAP Plugins in the Fledge UI" >}}

## Setup / Data Generation for Sinusoid Plugin

1. Let's setup the Sine-Wave Generator by naming it `test-sine-south` and click __Next__

{{< figure src="/images/technology/fledge-iot/fledge-setup-sine-1.png" title="Sinusoid Plugin Setup in Fledge UI. Step 1" >}}

2. Name the __Asset Name__. We name it `sine-wave-asset`
{{< figure src="/images/technology/fledge-iot/fledge-setup-sine-2.png" title="Name the Asset in Fledge UI. Step 1" >}}

3. Make sure to enable the plugin by checking the __Enabled__ checkbox and finally click __Done__

On the UI you now see the Sinusoid Plugin Enabled with active readings being fed into the Fledge Core. You can click on the link `test-sine-south` to change the configuration of the plugin / delete it or export readings to a file.

You can click on __Assets & Readings__ Menu and click on the Graph Icon to get some visualization

{{< figure src="/images/technology/fledge-iot/fledge-setup-sine-3.png" title="Assets & Readings of Sinusoid Wave" >}}

And there you have visualization of the data coming in from your South Plugin, a lovely sine wave. Not practical but you get to understand how the Fledge Architecture would work!

{{< figure src="/images/technology/fledge-iot/fledge-sine-visualization.png" title="Data Visualization of Sine Wave Asset" >}}

You can click on the __Summary__ tab to get some basic statistics of the information

## Setup / Data Generation for CoAP Plugin

1. Let's setup the CoAP South Plugin similarly to Sinusoid by clicking on the __+ Add__ button on the top-right of the Panel under __South__ Page

{{< figure src="/images/technology/fledge-iot/fledge-setup-coap-1.png" title="Setting up CoAP South Plugin: Step 1" >}}

2. Call the service `test-coap-south` and click __Next__

{{< figure src="/images/technology/fledge-iot/fledge-setup-coap-2.png" title="Naming CoAP Service: Step 2" >}}

3. In the next step keep the configuration as it is for now i.e. Port as `5683` and the URI as `sensor-values` and click __Next__ and enable the plugin on the last setup.

This should enable the CoAP Plugin for Fledge and it's time to generated some CoAP data from a simulated client to a CoAP Server running on the Pi.

Fledge does provide some testing information for CoAP in the core directory under the the CLI called `fogbench`

Let's generate some CoAP data on the Pi.

```bash
$FLEDGE/bin/fogbench -t $FLEDGE/data/extras/fogbench/fogbench_sensor_coap.template.json -I 100
```
here `-t` flag is for mentioning a template and we use a coap template provided by Fledge mentioned above. `-I` is the time interval

Upon executing the command on the Pi, you can check the UI which will generate the following output for the CoAP Plugin.

{{< figure src="/images/technology/fledge-iot/fledge-coap-data-1.png" title="Assets for Fogbench CoAP Service" >}}

You can explore the Assets and Visualization by going to __Assets & Readings__ and clicking on any of the asset you wish to observe. 

> NOTE: In order to visualize live-data you can run the `fogbench` command above a couple of more times.

{{< figure src="/images/technology/fledge-iot/fledge-coap-visualization.png" title="Statistics for Luxometer Asset" >}}


## Filter Plugins in Fledge

The sine wave isn't that cool to look at to so let's get some Frequency Domain Analysis on it with a __Fast Fourier Transform__ on it using Fledge's FFT filter plugin

1. Let's clone the plugin in the home directory

        $ cd 
        $ git clone https://github.com/fledge-iot/fledge-filter-fft.git && cd fledge-filter-fft/

2. This plugin is not python based so we need to do some tasks for installing it. We export a `FLEDGE_ROOT` variable and create a `build` directory in the root of the repo

        $ # we export the directory where we built Fledge for the filter plugin to use
        $ export FLEDGE_ROOT=~/fledge/
        $ # in the root of the `fledge-filter-fft` repo
        $ mkdir build && cd build/

3. compile the plugin

        $ cmake ..

4. install the plugin

        $ sudo make install

Now you have the FFT plugin ready to be applied in Fledge


### Applying FFT on the our Sine-Wave Asset

1. Let's head back to the __South__ Plugins in the UI and click on the `test-sine-south` service

2. Click on the __Applications__ menu for the Service 

{{< figure src="/images/technology/fledge-iot/fledge-fft-setup-1.png" title="UI for test-sine-south Service" >}}

3. Select `fft` from the available plugins and name the service `test-fft-sine`

{{< figure src="/images/technology/fledge-iot/fledge-fft-setup-2.png" title="Name the FFT filter Plugin Service" >}}

4. Setup the Service to conduct filtering on the asset called `sine-wave-asset` (See steps to setup Sinusoid Plugin) and other parameters as shown below (no fine tuning done here). Don't forget to click the __Enabled__ checkbox and finally __Done__

{{< figure src="/images/technology/fledge-iot/fledge-fft-setup-3.png" title="Configuring FFT Filter" >}}


The Fast Fourier Transform might take some time to show up but you can head to __Assets & Readings__ and click on the `sine-wave-asset FFT`'s Graph Icon to see some visualizations. This is what mine look like

{{< figure src="/images/technology/fledge-iot/fledge-fft-visualization.png" title="FFT of the sine-wave-asset" >}}

Great stuff. Imagine having vibration sensors and you need to conduct FFT analyses on it! Fledge makes it nifty!!


## Outlook

The Fledge Project looks like something that could be a good solution for quick prototyping given on the `apt` issue is solved for me. Then it becomes a matter of doing the following:

```bash
apt install fledge fledge-ui .....
```

The only thing I can't find is where it is supported as a Docker/OCI container for deployment as an Edge Container. Nevertheless, Kudos to Fledge!