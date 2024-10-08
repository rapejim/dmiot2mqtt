# DMIOT protocol description

This document describes the JSON-based protocol used by the Dream Maker smart fans (dmiot). This applies to at least these two fan models:

* Dream Maker Smart Fan DM-FAN01
* Dream Maker Smart Fan DM-FAN02-W (battery-powered version)

It is quite likely that other devices from this manufacterer also use this protocol.

## General communication with the server

The Dream Maker devices talk to a remote server over a persistent TCP connection.
The traffic is not encrypted. The server hostname and ports are set to fixed values:

 Server: cloud1.dm-maker.com<br>
 TCP port: 31270
 
 The server and the client exchange JSON messages and can push updates/commands through
 the connection. Each update/command needs to be ACKed from the other side. The protocol
 defines actions (`int`) and resources (`int`).
 
 The following message show an example message (ACK message from server on resource 127):
 
 ```json
 {"action": 81, "resource_id": 127, "version":"zeico_3.0.0", "code": 0}
 ```

## Actions and resources

Client actions:

action | resource_id | explanation
-------|-------------|------------
1      | 2000        | Provisioning request. Sent by a Dream Maker device as soon as the WiFi has been provisioned through Bluetooth
1      | 2001        | Auth request. Sent on every reconnect to authenticate the client.
1      | 127         | Continuous status update (hearbeat/wifi strength)
84     | 9031        | state changed, full json is being sent

Server actions:

action | resource_id | explanation
-------|-------------|------------
81     | 2001        | Provisioning response with data `{"device_key":"xxxx","device_id":"yyyy"}`
81     | ***         | ACK the previous request for resource. The `resource_id` has to match the request which is being ACK'ed
4      | 9031        | Server initiated change (needs to send JSON data)

## Initial provisioning

After resetting a DMIOT device to factory defaults it has to be provisioned through Bluetooth LE first.
During this process the official Dream Maker app sends the WiFi credentials and a 6 character token (called
*mark*) to the fan through the Bluetooth connection. With this information the fan connects to the local
WiFi, contacts the Dream Maker servers and sends a provisioning request to the server using the 6 character
token.

1. Initial provisioning request from the fan: 
```json
{
   "action" : 1,
   "data" : {
      "factory_name" : "dreammaker",
      "local_id" : 1,
      "local_ip" : "192.168.1.1",
      "local_port" : 8080,
      "mac" : "98f4abffffff",
      "mark" : "AbCD2Z",
      "product_id" : "000000000000000000000000",
      "product_model" : "fan02-w",
      "product_type" : "fan",
      "prov_code" : "0000000000000000",
      "sn" : "000000000000",
      "ssid" : "your_wifi_ssid",
      "tail" : "UNKNOWN_DATA"
   },
   "msg_id" : 0,
   "resource_id" : 2000,
   "version" : "zeico_3.0.0"
}
```
* The `mark` attribute is the token generated by the Dreak Maker app and received through Bluetooth
* The meaning of the `prov_code` attribute is currently unkown
* The meaning of the `tail` attribute is currently unknown

2. Response from the server:
```json
{
   "action" : 81,
   "code" : 0,
   "data" : {
      "device_id" : "000000000000000000000000",
      "device_key" : "0000000000000000"
   },
   "resource_id" : 2000,
   "version" : "zeico_3.0.0"
}
```
* The `device_key` and `device_id` are stored in the Dream Maker device's flash memory. This is used for further authentication.

From now on the DMIOT device is provisioned and does not issue any provisioning requests any more (unless it is reset).

## Authentication

The DMIOT device obtained a `device_key` and `device_id` through the initial provisioning process. This information is used for authentication purposes. This is how a DMIOT opens a new TCP connection to the Dream Maker server:

1. Auth request from client:
```json
{
   "action" : 1,
   "code" : 0,
   "data" : {
      "comm_version" : "dmiot_v1.1.0",
      "device_id" : "000000000000000000000000",
      "device_key" : "0000000000000000",
      "mcu_version" : "fan_0001",
      "product_id" : "000000000000000000000000",
      "rf_version" : "v3.1.6"
   },
   "msg_id" : 0,
   "resource_id" : 2001,
   "version" : "zeico_3.0.0"
}
```
2. The server ACKs this request with action `81` and resource `2001`:
```json
 {"action": 81, "resource_id": 2001, "version":"zeico_3.0.0", "code": 0}

```

## Status updates sent by the DMIOT device

There are two different types of updates which DMIOT devices push to the server. There is a continuous update (like a heartbeat) which is pushed to the server every 10 seconds. This update also contains the device's current WiFi signal strength:

```json
{
   "action" : 1,
   "data" : {
      "comm_version" : "dmiot_v1.1.0",
      "mcu_version" : "fan_0001",
      "rf_version" : "v3.1.6",
      "signal" : 94
   },
   "resource_id" : 127,
   "version" : "zeico_3.0.0"
}
```

The continuous update has to be ACK'ed by the server like this:

```json
{"action":81,"resource_id":127,"version":"zeico_3.0.0","code":0}
```


Whenever the status of the DMIOT device changes (e.g. after turning a fan on or chaning it's mode), a full update is pushed to the server. The update data looks like this:

```json
{
   "action" : 84,
   "code" : 0,
   "data" : {
      "child_lock" : 0,
      "deviceException" : 0,
      "ext1" : 97,
      "ext2" : 102,
      "ext3" : 24884,
      "ext4" : 25137,
      "ext5" : 943206968,
      "ext6" : 1630823777,
      "humidity" : 64,
      "light" : 1,
      "mode" : 0,
      "power" : 1,
      "power_delay" : 0,
      "roll_angle" : 90,
      "roll_enable" : 0,
      "sound" : 0,
      "source" : "manual",
      "speed" : 1,
      "temperature" : 23.6,
      "useException" : 0
   },
   "msg_id" : 1234,
   "resource_id" : 9031,
   "version" : "zeico_3.0.0"
}
```

The attribute names are somewhat self-explanatory, but here is some more details:

* `child_lock`: When set to `1` the device cannot be controlled through single key presses any more. Can be reset through a button combination or by setting the attribute back to `0`
* `humidity`: Reported air humidity in percent
* `light`: If set to `0` the LED indicators on the device can be turned off
* `mode`: `0` means "direct" mode, `1` means "natural" mode, `2` means "smart mode"
* `power`: A value of `1` indicates that the device is turned on
* `power_delay`: The device will turn off after the amount of `power_delay` minutes. The maximum is 480 minutes (8 hours).
* `roll_angle`: Angle for the fan's oscillation. Possible values: `30`, `60`, `90`, `120`, `140`
* `roll_enable`: Enables the fan's oscillation
* `sound`: If set to `0` the device does not make any sound when pressing a button or changing it's mode. Default: `1`.
* `source`: When the device is programmed through the official Dream Maker app using a smart schedule, this will be `auto`. Otherwise `manual`.
* `speed`: The fan speed in percent. The range is from 1 to 100.
* `temperature`: The ambient temperature in degrees Celsuis

Apparently this kind of update does not need to be ACK'ed from the server

## Controlling the device

The device can be controlled through JSON commands. This is how the DMIOT device can be turned on through the server:

```json
{"action":4,"resource_id":9031,"version":"zeico_3.0.0","data":{"source":"manual","power":1},"msg_id":1234}
```

The action needs be set through the `data` attribute. Only one action can be performed at a time. This means that you cannot turn the device on with a specific speed, you always need to issue the "power" command first and then the "speed" command. The message id (`msg_id´) **may** be incremented by the server and the client will send out a status update with the same message id.

The following actions have been tested with the Dream Maker fan devices:

* Turn on/off: `{"power":1}`
* Set speed to value: `{"speed": 50}`
* Disable button sounds: `{"sound": 0}`
* Disable LED indicators: `{"light": 0}`
* Enable oscillation: `{"roll_enable": 1}`
* Set oscillation angle (60º): `{"roll_ange": 60}`
* Turn fan one step left: `:{"roll_control":1}` (only with oscillation disabled)
* Turn fan one step right: `:{"roll_control":2}` (only with oscillation disabled)
* Enable smart mode (2): `:{"mode": 2}`
