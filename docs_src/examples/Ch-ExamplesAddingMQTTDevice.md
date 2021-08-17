# MQTT

EdgeX - Jakarta Release

## Overview

In this example, we use the simulator instead of real device. This
provides a straight-forward way to test the device-mqtt features.

![MQTT Overview](MQTT_Example_Overview.png)

## Run an MQTT Broker

Eclipse Mosquitto is an open source (EPL/EDL licensed) message broker
that implements the MQTT protocol versions 5.0, 3.1.1 and 3.1.

Run Mosquitto using the following docker command:

    docker run -d --rm --name broker -p 1883:1883 eclipse-mosquitto:1.6

## Enabling multi-level topics in device services
To use the optional setting for MQTT device services with multi-level
topics, make the following changes:

1. Modify these lines in `configuration.toml`:
```yaml
# Comment out/remove when using multi-level topics
#IncomingTopic = 'DataTopic'
#ResponseTopic = 'ResponseTopic'
#UseTopicLevels = false

# Uncomment to use multi-level topics
IncomingTopic = 'incoming/data/#'
ResponseTopic = 'command/response/#'
UseTopicLevels = true
```
2. Modify`CommandTopic = 'command/MQTT-test-device'` in the `mqtt.test.device.toml`

Notice the sections marked with **Using Multi-level Topic:** for relevant input/output
throughout this example.

## Run an MQTT Device Simulator

![MQTT Device Service](EdgeX_ExamplesMQTTDeviceSimulator.png)

This simulator has three behaviors:

1. Publish random number data every 15 seconds.

    **Default (single-level) Topic:** 
    The simulator publishes the data to the MQTT broker with topic `DataTopic` and the message is similar to the following:
    ```
    {"name":"MQTT-test-device", "cmd":"randfloat32", "method":"get", "randfloat32":4161.3549}
    ```
    **Using Multi-level Topic:**
    The simulator publishes the data to the MQTT broker with topic `incoming/data/#` and the message is similar to the following:
    ```
    {"randfloat32":4161.3549}
    ```

2. Receive the reading request, then return the response.

    **Default (single-level) Topic:**

    1. The simulator receives the request from the MQTT broker, the topic is `CommandTopic` and the message is similar to the following:
        ```
        {"cmd":"randfloat32", "method":"get", "uuid":"293d7a00-66e1-4374-ace0-07520103c95f"}
        ```
    2. The simulator returns the response to the MQTT broker, the topic is `ResponseTopic` and the message is similar to the following:
        ```
        {"cmd":"randfloat32", "method":"get", "uuid":"293d7a00-66e1-4374-ace0-07520103c95f", "randfloat32":42.0}
        ```

    **Using Multi-level Topic:**

    1. The simulator receives the request from the MQTT broker, the topic is `command/MQTT-test-device/randfloat32` and the message is similar to the following:
        ```
        {"message":"42.0"}
        ```
    2. The simulator returns the response to the MQTT broker, the topic is `command/response/#` and the message is similar to the following:
        ```
        {"message":"4.20e+01"}
        ```

3. Receive the set request, then change the device value.
   
    **Default (single-level) Topic:**

    1. The simulator receives the request from the MQTT broker, the topic is `CommandTopic` and the message is similar to the following:
        ```   
        {"cmd":"message", "method":"set", "uuid":"293d7a00-66e1-4374-ace0-07520103c95f", "message":"test message..."}
        ```
    2. The simulator changes the device value and returns the response to the MQTT broker, the topic is `ResponseTopic` and the message is similar to the following:
         ```   
         {"cmd":"message", "method":"set", "uuid":"293d7a00-66e1-4374-ace0-07520103c95f"}
         ```

    **Using Multi-level Topic:**

    1. The simulator receives the request from the MQTT broker, the topic is `command/MQTT-test-device/testmessage` and the message is similar to the following:
        ```   
        {"message":"test message..."}
        ```
    2. The simulator changes the device value and returns the response to the MQTT broker, the topic is `command/response/#` and the message is similar to the following:
        ```   
        {"message":"test message..."}
        ```

To simulate the MQTT device, create a javascript, named `mock-device.js`, with the
following content:

**Default (single-level) Topic:**
```javascript
function getRandomFloat(min, max) {
    return Math.random() * (max - min) + min;
}

const deviceName = "MQTT-test-device";
let message = "test-message";

// DataSender sends async value to MQTT broker every 15 seconds
schedule('*/15 * * * * *', ()=>{
    let body = {
        "name": deviceName,
        "cmd": "randfloat32",
        "randfloat32": getRandomFloat(25,29).toFixed(1)
    };
    publish( 'DataTopic', JSON.stringify(body));
});

// CommandHandler receives commands and sends response to MQTT broker
// 1. Receive the reading request, then return the response
// 2. Receive the set request, then change the device value
subscribe( "CommandTopic" , (topic, val) => {
    var data = val;
    if (data.method == "set") {
        message = data[data.cmd]
    }else{
        switch(data.cmd) {
            case "ping":
                data.ping = "pong";
                break;
            case "message":
                data.message = message;
                break;
            case "randfloat32":
                data.randfloat32 = 12.32;
                break;
            case "randfloat64":
                data.randfloat64 = 12.64;
                break;
        }
    }
    publish( "ResponseTopic", JSON.stringify(data));
});
```
**Using Multi-level Topic:**
```javascript
function getRandomFloat(min, max) {
    return Math.random() * (max - min) + min;
}

const deviceName = "MQTT-test-device";
let message = "test-message";

// DataSender sends async value to MQTT broker every 15 seconds
schedule('*/15 * * * * *', ()=>{
    let body = getRandomFloat(25,29).toFixed(1);
    publish( 'incoming/data/MQTT-test-device/randfloat32', body);
});

// CommandHandler receives commands and sends response to MQTT broker
// 1. Receive the reading request, then return the response
// 2. Receive the set request, then change the device value
subscribe( "command/MQTT-test-device/#" , (topic, val) => {
    const words = topic.split('/');
    var cmd = words[3];
    var method = words[4];
    var uuid = words[5];
    var response = {};
    var data = val;

    if (method == "set") {
        message = data[cmd]
    }else{
        switch(cmd) {
            case "ping":
                response.ping = "pong";
                break;
            case "message":
                response.message = message;
                break;
            case "randfloat32":
                response.randfloat32 = 12.123;
                break;
            case "randfloat64":
                response.randfloat64 = 34.32;
                break;
        }
    }
    var sendTopic ="command/response/"+ uuid;
    publish( sendTopic, JSON.stringify(response));
});
```
To run the device simulator, enter the commands shown below with the
following changes:

- Replace the `/path/to/mqtt-scripts` in the example mv command with the
    correct path
```
$ mv mock-device.js /path/to/mqtt-scripts
$ docker run -d --restart=always --name=mqtt-scripts \
    -v /path/to/mqtt-scripts:/scripts  \
    dersimn/mqtt-scripts --url mqtt://172.17.0.1 --dir /scripts
```
> The address `172.17.0.1` is point to the host of MQTT broker via the docker bridge network.

## Prepare the Custom Configuration 

In this section, we create folders that contains files required for deployment:
```
- custom-config
  |- profiles
     |- mqtt.test.device.profile.yml
  |- devices
     |- mqtt.test.device.config.toml
```

### Device Profile

The DeviceProfile defines the device's values and operation method,
which can be Read or Write.

Create a device profile, named `mqtt.test.device.profile.yml`, with the
following content:
```yaml
name: "Test-Device-MQTT-Profile"
manufacturer: "Dell"
model: "MQTT-2"
labels:
  - "test"
description: "Test device profile"
deviceResources:
  -
    name: randfloat32
    isHidden: true
    description: "random 32 bit float"
    properties:
      valueType: "Float32"
      readWrite: "RW"
      defaultValue: "0.00"
      minimum: "0.00"
      maximum: "100.00"
  -
    name: randfloat64
    isHidden: true
    description: "random 64 bit float"
    properties:
      valueType: "Float64"
      readWrite: "RW"
      defaultValue: "0.00"
      minimum: "0.00"
      maximum: "100.00"
  -
    name: ping
    isHidden: true
    description: "device awake"
    properties:
      valueType: "String"
      readWrite: "R"
      defaultValue: "oops"
  -
    name: message
    isHidden: true
    description: "device notification message"
    properties:
      valueType: "String"
      readWrite: "RW"
      scale: ""
      offset: ""
      base: ""

deviceCommands:
  -
    name: testrandfloat32
    readWrite: "R"
    isHidden: false
    resourceOperations:
      - { deviceResource: "randfloat32" }
  -
    name: testrandfloat64
    readWrite: "R"
    isHidden: false
    resourceOperations:
      - { deviceResource: "randfloat64" }
  -
    name: testping
    readWrite: "R"
    isHidden: false
    resourceOperations:
      - { deviceResource: "ping" }
  -
    name: testmessage
    readWrite: "RW"
    isHidden: false
    resourceOperations:
      - { deviceResource: "message" }
  -
    name: randfloat32andfloat64
    readWrite: "RW"
    isHidden: false
    resourceOperations:
      - { deviceResource: "randfloat32" }
      - { deviceResource: "randfloat64" }

```

### Device Configuration

Use this configuration file to define devices and schedule jobs.
device-mqtt generates a relative instance on start-up.

Create the device configuration file, named `mqtt.test.device.toml`, as shown below:

```toml
# Pre-define Devices
[[DeviceList]]
  Name = 'MQTT-test-device'
  ProfileName = 'Test-Device-MQTT-Profile'
  Description = 'MQTT device is created for test purpose'
  Labels = [ 'MQTT', 'test' ]
  [DeviceList.Protocols]
    [DeviceList.Protocols.mqtt]
       # Comment out/remove below to use multi-level topics
       CommandTopic = 'CommandTopic'
       # Uncomment below to use multi-level topics
       # CommandTopic = 'command/MQTT-test-device'
#    [[DeviceList.AutoEvents]]
#       Interval = '20s'
#       OnChange = false
#       SourceName = 'testrandfloat32'
```

- `CommandTopic` is used to publish the GET or SET command request

!!! note
    **Using Multi-level Topic:** Remember to set `CommandTopic = 'command/MQTT-test-device'.

## Prepare docker-compose file

1. Clone edgex-compose
```
$ git clone git@github.com:edgexfoundry/edgex-compose.git
$ git checkout ireland
```
2. Generate the docker-compose.yml file
```
$ cd edgex-compose/compose-builder
$ make gen ds-mqtt
```
Check the generated file
```
$ ls | grep 'docker-compose.yml'
docker-compose.yml
```

### Mount the custom-config

Open the `docker-compose.yml` file and then add volumes path and environment as shown below:

- Replace the `/path/to/custom-config` in the example with the correct path

```yaml
 device-mqtt:
    ...
    environment:
      ...
      DEVICE_DEVICESDIR: /custom-config/devices
      DEVICE_PROFILESDIR: /custom-config/profiles
      MQTTBROKERINFO_HOST: 172.17.0.1
    volumes:
    ...
    - /path/to/custom-config:/custom-config
```

> The address `172.17.0.1` is point to the MQTT broker via the docker bridge network.

## Start EdgeX Foundry on Docker

Deploy EdgeX using the following commands:
```
$ cd edgex-compose/compose-builder
$ docker-compose pull
$ docker-compose up -d
```

## Execute Commands

Now we're ready to run some commands.

### Find Executable Commands

Use the following query to find executable commands:

**Default (single-level) Topic:**

```json
$ curl http://localhost:59882/api/v2/device/all | json_pp

{
   "deviceCoreCommands" : [
      {
         "profileName" : "Test-Device-MQTT-Profile",
         "deviceName" : "MQTT-test-device",
         "coreCommands" : [
            {
               "url" : "http://edgex-core-command:59882",
               "parameters" : [
                  {
                     "resourceName" : "randfloat32",
                     "valueType" : "Float32"
                  },
                  {
                     "resourceName" : "ping",
                     "valueType" : "String"
                  },
                  {
                     "resourceName" : "message",
                     "valueType" : "String"
                  }
               ],
               "get" : true,
               "name" : "values",
               "path" : "/api/v2/device/name/MQTT-test-device/values"
            },
            {
               "url" : "http://edgex-core-command:59882",
               "parameters" : [
                  {
                     "valueType" : "String",
                     "resourceName" : "message"
                  }
               ],
               "get" : true,
               "set" : true,
               "path" : "/api/v2/device/name/MQTT-test-device/message",
               "name" : "message"
            }
         ]
      }
   ],
   "apiVersion" : "v2",
   "statusCode" : 200
}
```

**Using Multi-level Topic:**

```json
$ curl http://localhost:59882/api/v2/device/all | json_pp
{
   "apiVersion" : "v2",
   "deviceCoreCommands" : [
      {
         "coreCommands" : [
            {
               "get" : true,
               "name" : "testrandfloat32",
               "parameters" : [
                  {
                     "resourceName" : "randfloat32",
                     "valueType" : "Float32"
                  }
               ],
               "path" : "/api/v2/device/name/MQTT-test-device/testrandfloat32",
               "url" : "http://edgex-core-command:59882"
            },
            {
               "get" : true,
               "name" : "testrandfloat64",
               "parameters" : [
                  {
                     "resourceName" : "randfloat64",
                     "valueType" : "Float64"
                  }
               ],
               "path" : "/api/v2/device/name/MQTT-test-device/testrandfloat64",
               "url" : "http://edgex-core-command:59882"
            },
            {
               "get" : true,
               "name" : "testping",
               "parameters" : [
                  {
                     "resourceName" : "ping",
                     "valueType" : "String"
                  }
               ],
               "path" : "/api/v2/device/name/MQTT-test-device/testping",
               "url" : "http://edgex-core-command:59882"
            },
            {
               "get" : true,
               "name" : "testmessage",
               "parameters" : [
                  {
                     "resourceName" : "message",
                     "valueType" : "String"
                  }
               ],
               "path" : "/api/v2/device/name/MQTT-test-device/testmessage",
               "set" : true,
               "url" : "http://edgex-core-command:59882"
            },
            {
               "get" : true,
               "name" : "randfloat32andfloat64",
               "parameters" : [
                  {
                     "resourceName" : "randfloat32",
                     "valueType" : "Float32"
                  },
                  {
                     "resourceName" : "randfloat64",
                     "valueType" : "Float64"
                  }
               ],
               "path" : "/api/v2/device/name/MQTT-test-device/randfloat32andfloat64",
               "url" : "http://edgex-core-command:59882"
            }
         ],
         "deviceName" : "MQTT-test-device",
         "profileName" : "Test-Device-MQTT-Profile"
      }
   ],
   "statusCode" : 200
}
```

### Execute SET Command

Execute a SET command according to the url and parameterNames, replacing
\[host\] with the server IP when running the SET command.

```
$ curl http://localhost:59882/api/v2/device/name/MQTT-test-device/message \
    -H "Content-Type:application/json" -X PUT  \
    -d '{"message":"Hello!"}'
```

### Execute GET Command

Execute a GET command as follows:

**Default (single-level) Topic:**

```json
$ curl http://localhost:59882/api/v2/device/name/MQTT-test-device/message | json_pp

{
   "event" : {
      "origin" : 1624417689920618131,
      "readings" : [
         {
            "resourceName" : "message",
            "binaryValue" : null,
            "profileName" : "Test-Device-MQTT-Profile",
            "deviceName" : "MQTT-test-device",
            "id" : "a3bb78c5-e76f-49a2-ad9d-b220a86c3e36",
            "value" : "Hello!",
            "valueType" : "String",
            "origin" : 1624417689920615828,
            "mediaType" : ""
         }
      ],
      "sourceName" : "message",
      "deviceName" : "MQTT-test-device",
      "apiVersion" : "v2",
      "profileName" : "Test-Device-MQTT-Profile",
      "id" : "e0b29735-8b39-44d1-8f68-4d7252e14cc7"
   },
   "apiVersion" : "v2",
   "statusCode" : 200
}

```

**Using Multi-level Topic:**

```json
{
   "apiVersion" : "v2",
   "event" : {
      "apiVersion" : "v2",
      "deviceName" : "MQTT-test-device",
      "id" : "aec027e0-2005-45cd-8ef9-a2f0be2accf3",
      "origin" : 1629424063084088100,
      "profileName" : "Test-Device-MQTT-Profile",
      "readings" : [
         {
            "deviceName" : "MQTT-test-device",
            "id" : "0f491a97-203f-4aea-a57b-4efe6dce885e",
            "origin" : 1629424063084066500,
            "profileName" : "Test-Device-MQTT-Profile",
            "resourceName" : "message",
            "value" : "Hello!",
            "valueType" : "String"
         }
      ],
      "sourceName" : "message"
   },
   "statusCode" : 200
}
```

## Schedule Job

The schedule job is defined in the `[[DeviceList.AutoEvents]]` section of the device configuration file:

```toml
    [[DeviceList.AutoEvents]]
       Interval = "30s"
       OnChange = false
       SourceName = "message"
```

After the service starts, query core-data's reading API. The results
show that the service auto-executes the command every 30 secs, as shown
below:

**Default (single-level) Topic:**

```json
$ curl http://localhost:59880/api/v2/reading/resourceName/message | json_pp

{
   "statusCode" : 200,
   "readings" : [
      {
         "value" : "test-message",
         "id" : "e91b8ca6-c5c4-4509-bb61-bd4b09fe835c",
         "mediaType" : "",
         "binaryValue" : null,
         "resourceName" : "message",
         "origin" : 1624418361324331392,
         "profileName" : "Test-Device-MQTT-Profile",
         "deviceName" : "MQTT-test-device",
         "valueType" : "String"
      },
      {
         "mediaType" : "",
         "binaryValue" : null,
         "resourceName" : "message",
         "value" : "test-message",
         "id" : "1da58cb7-2bf4-47f0-bbb8-9519797149a2",
         "deviceName" : "MQTT-test-device",
         "valueType" : "String",
         "profileName" : "Test-Device-MQTT-Profile",
         "origin" : 1624418330822988843
      },
      ...
   ],
   "apiVersion" : "v2"
}


```

**Using Multi-level Topic:**

```json
{
   "apiVersion" : "v2",
   "readings" : [
      {
         "deviceName" : "MQTT-test-device",
         "id" : "d01bce60-68f5-4baa-a339-5fb34951e81d",
         "origin" : 1629424055756053600,
         "profileName" : "Test-Device-MQTT-Profile",
         "resourceName" : "message",
         "value" : "Hello!",
         "valueType" : "String"
      },
      ...
   ],
   "statusCode" : 200
}
```


## Async Device Reading

The `device-mqtt` subscribes to a `DataTopic`, which is wait for the [real device to send value to MQTT broker](#run-an-mqtt-device-simulator), then `device-mqtt`
parses the value and forward to the northbound.

The data format contains the following values:

-   name = device name
-   cmd = deviceResource name
-   method = get or set
-   cmd = device reading

The following results show that the mock device sent the reading every
15 secs:

**Default (single-level) Topic:**

```json
$ curl http://localhost:59880/api/v2/reading/resourceName/randfloat32 | json_pp

{
   "readings" : [
      {
         "origin" : 1624418475007110946,
         "valueType" : "Float32",
         "deviceName" : "MQTT-test-device",
         "id" : "9b3d337e-8a8a-4a6c-8018-b4908b57abb8",
         "binaryValue" : null,
         "resourceName" : "randfloat32",
         "profileName" : "Test-Device-MQTT-Profile",
         "mediaType" : "",
         "value" : "2.630000e+01"
      },
      {
         "deviceName" : "MQTT-test-device",
         "valueType" : "Float32",
         "id" : "06918cbb-ada0-4752-8877-0ef8488620f6",
         "origin" : 1624418460007833720,
         "mediaType" : "",
         "profileName" : "Test-Device-MQTT-Profile",
         "value" : "2.570000e+01",
         "resourceName" : "randfloat32",
         "binaryValue" : null
      },
      ...
   ],
   "statusCode" : 200,
   "apiVersion" : "v2"
}
```

**Using Multi-level Topic:**

```json
{
   "apiVersion" : "v2",
   "readings" : [
      {
         "deviceName" : "MQTT-test-device",
         "id" : "3b1e197c-64ad-4f3d-a344-5d375147cba7",
         "origin" : 1629425475004246800,
         "profileName" : "Test-Device-MQTT-Profile",
         "resourceName" : "randfloat32",
         "value" : "2.760000e+01",
         "valueType" : "Float32"
      },
      {
         "deviceName" : "MQTT-test-device",
         "id" : "9c9061c8-3cf6-4aa4-9ab2-fc660efb6870",
         "origin" : 1629425460004623400,
         "profileName" : "Test-Device-MQTT-Profile",
         "resourceName" : "randfloat32",
         "value" : "2.590000e+01",
         "valueType" : "Float32"
      },
      ...
   ],
   "statusCode" : 200
}
```

## MQTT Device Service Configuration

MQTT Device Service has the following configurations to implement the MQTT protocol.

| Configuration                        | Default Value | Description                                                  |
| ------------------------------------ | ------------- | ------------------------------------------------------------ |
| MQTTBrokerInfo.Schema                | tcp           | The URL schema |
| MQTTBrokerInfo.Host                  | 0.0.0.0       | The URL host |
| MQTTBrokerInfo.Port                  | 1883          | The URL port |
| MQTTBrokerInfo.Qos                   | 0             | Quality of Service 0 (At most once), 1 (At least once) or 2 (Exactly once) |
| MQTTBrokerInfo.KeepAlive             | 3600          | Seconds between client ping when no active data flowing to avoid client being disconnected. Must be greater then 2 |
| MQTTBrokerInfo.ClientId              | device-mqtt   | ClientId to connect to the broker with |
| MQTTBrokerInfo.CredentialsRetryTime  | 120           | The retry times to get the credential |
| MQTTBrokerInfo.CredentialsRetryWait  | 1             | The wait time(seconds) when retry to get the credential  |
| MQTTBrokerInfo.ConnEstablishingRetry | 10            | The retry times to establish the MQTT connection    | 
| MQTTBrokerInfo.ConnRetryWaitTime     | 5             | The wait time(seconds) when retry to establish the MQTT connection   |
| MQTTBrokerInfo.AuthMode              | none          | Indicates what to use when connecting to the broker. Must be one of "none" , "usernamepassword" |
| MQTTBrokerInfo.CredentialsPath       | credentials   | Name of the path in secret provider to retrieve your secrets. Must be non-blank. |\
| MQTTBrokerInfo.IncomingTopic         | DataTopic (incoming/data/#) | IncomingTopic is used to receive the async value |
| MQTTBrokerInfo.responseTopic         | ResponseTopic (command/response/#) | ResponseTopic is used to receive the command response from the device |
| MQTTBrokerInfo.UseTopicLevels        | false (true)  | Boolean setting to use multi-level topics |
| MQTTBrokerInfo.Writable.ResponseFetchInterval | 500  | ResponseFetchInterval specifies the retry interval(milliseconds) to fetch the command response from the MQTT broker |

!!! note
    **Using Multi-level Topic:** Remember to set defaults in parentheses in the table above.

The user can override these configurations by `environment variable` to meet their requirement, for example:

```yaml
# docker-compose.yml

 device-mqtt:
    ...
    environment:
      ...
      MQTTBROKERINFO_HOST: 172.17.0.1
```
