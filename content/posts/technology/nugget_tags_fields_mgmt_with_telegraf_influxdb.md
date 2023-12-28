---
title: "Nugget: Let Telegraf manage your InfluxDB Fields and Tags for you"
date: 2020-09-03T14:38:09+02:00
draft: false
author: "Shan"

description: "Map your fields to distinct values and replace them as tags for InfluxDB"
categories: ["Technology"]
tags: ["MQTT", "IoT", "Telegraf", "InfluxDB"]

toc:
  enable: true
  auto: true
---
<!--more-->

## Overview

Assume that some data is arriving to my Acquisition system in the form of the following JSON schema:

```json
{
    "name":"systemStatus",
    "timestamp": "2020-09-02 13:10:00.000",
    "result": "GOOD"
}
```

In my case, the data is published to an MQTT Broker with the above mentioned JSON data as payload to a topic:

    PROJ/system/status

## Requirement

The following are the requirements:

1. store the incoming JSON payload in InfluxDB
2. instead of storing `GOOD` and `BAD` as fields, map them as `GOOD -> 0` and `BAD -> 1` where `result` should be stored as a `tag` and `value` (either `0` or `1`) should be stored as `field`

Theoretically the __InfluxDB Line Protocol String__ should look like:

    systemStatus,result="GOOD" value=0 1599052200000
    systemStatus,result="BAD" value=1 1599052200000

So Queries to other software components can be simply:

    SELECT value from PROJ.autogen.systemStatus where result='GOOD'

## Who You gonna call 📞? .... Telegraf

Here is how I solved the requirement:

1. Connect `telegraf` to your MQTT Broker using the `[[inputs.mqtt_consumer]]` plugin and subscribe to the `PROJ/system/status` topic
2. Use the `json` as the incoming data format. Telegraf Provides a wide range of [Input Data Format](https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_INPUT.md)

         data_format = "json"

    a. Use the `name` key of the JSON Payload as the Measurement Name using:
    
            json_name_key = "name"

    b. Store the `result` key of the payload as a field for the time being and make sure to store it as a `string`. For that simply use: 
    
            json_string_fields = ["result"]

    c. Use the `timestamp` key of the payload to store the timestamp with [Golang Format](https://yourbasic.org/golang/format-parse-string-time-date-example/) as follows:

             json_time_format = "2006-01-02 15:04:05.000"
             json_time_key = "timestamp"

    TOML Config:
    ```toml
        [[inputs.mqtt_consumer]]

            servers = [ "ssl://MY_MQTT_BROKER:8883" ]

            # Topics to subscribe to:
            topics = [
                "PROJ/system/status"
            ]

            # User Credentials and TLS Settings go here.

            # Incoming MQTT Payload is in JSON format with fixed schema
            data_format = "json"

            json_name_key = "name"
            json_string_fields = ["result"]
            json_time_format = "2006-01-02 15:04:05.000"
            json_time_key = "timestamp"
    ```

3. Connect `telegraf` to your InfluxDB Instance using the `[[outputs.influxdb]]` or `[[outputs.influxdb_v2]]` plugin

    TOML Config:

    ```toml

        [[outputs.influxdb]]

        urls = ["https://<MY_INFLUXDBV1_INSTANCE:8086"]
        database = "PROJ"
        skip_database_creation = false
        
        # User Credentials + TLS Verify settings go here
    ```

4. __JUICY PART__: Time to leverage the `processors` plugin to achieve our task:

    a. Let's first use `enum` to map the respective values i.e. `GOOD -> 0` and `BAD -> 1` on our already available field `result`:

    TOML Config:
    ```toml
        [[processors.enum]]
            order = 1
        [[processors.enum.mapping]]
            field = "result"
            dest = "value"
            [processors.enum.mapping.value_mappings]
                "GOOD" = 0
                "BAD"  = 1
    ```

    This configuration will be executed first due to `order=1` and will map out field `result` to their respective numeric values into another field called `value`

    At this point, the Line Protocol String should look similar to:

        systemStatus result="GOOD",value=0.0 1599052200000
        systemStatus result="BAD",value=1.0 1599052200000
    
    b. Let's use another `processors` plugin called `converter` to convert our field `result` into a tag.

    TOML Config:
    ```toml
        [[processors.converter]]
            order = 2
            [processors.converter.fields]
                tag = ["result"]
    ````

    The `order=2` takes care of telling `telegraf` to convert the `field` called `result` to a `tag` only after the `enum` mappings occur.

## Complete Configuration File

{{< gist shantanoo-desai 5399f98476d0cacc0746a6fa75ef1a1b >}}


### Deployment via Docker

```bash
# assuming the above config file is in a directory called `telegraf`
docker run  -d --name=telegraf-converter -v $(pwd)/telegraf/telegraf.convert.toml:/etc/telegraf/telegraf.conf:ro telegraf:latest
```

See the logs using:

```bash
docker logs -f telegraf-converter
```


## Conclusions

> Why write scripts for things when you can write a TOML configuration in `telegraf`!!
> -- {{< style "text-align: right;" >}} _Lazy-ius Maximus_ {{</ style >}} 


If you have more thoughts, improvements and criticisms then connect with me or send me an E-mail or a __LinkedIn Message__ anytime!