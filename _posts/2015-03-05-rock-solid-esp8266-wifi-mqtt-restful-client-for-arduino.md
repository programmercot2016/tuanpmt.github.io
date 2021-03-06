---
layout: post
title: "Rock solid esp8266 wifi mqtt, restful client for arduino"
modified:
categories:
redirect_from:
    - /post/espduino/
excerpt: "Rock solid esp8266 wifi mqtt, restful client for arduino"
tags: [mqtt, esp8266, restful, REST, arduino]
comments: true
---


###Github Repository: [https://github.com/tuanpmt/espduino](https://github.com/tuanpmt/espduino)
<iframe src="https://ghbtns.com/github-btn.html?user=tuanpmt&repo=espduino&type=fork&count=true&size=large" frameborder="0" scrolling="0" width="158px" height="30px"></iframe>
<br/>
This is Wifi library (Chip ESP8266 Wifi soc) for arduino using SLIP protocol via Serial port

If you want using only ESP8266, you can find the **Native MQTT client** library for ESP8266 work well here: 
[https://github.com/tuanpmt/esp_mqtt](https://github.com/tuanpmt/esp_mqtt)


Features
========
- Rock Solid wifi network client for Arduino
- MQTT module: 
    + MQTT client run stable as Native MQTT client (esp_mqtt)
    + Support subscribing, publishing, authentication, will messages, keep alive pings and all 3 QoS levels (it should be a fully functional client).
    + Support multiple connection (to multiple hosts).
    + Support SSL
    + Easy to setup and use
- REST module:
    + Support method GET, POST, PUT, DELETE
    + setContent type, set header, set User Agent
    + Easy to used API
    + Support SSL
- WIFI module:
- mDNS module: (coming soon)


Installations
========
**1. Clone this project:**

```bash
git clone https://github.com/tuanpmt/espduino
cd espduino
```

**2. Program ESP8266:**

- Wiring: ![Program Connection diagram](/images/espduino/program_esp8266_bb.png)
- Program release firmware:

```python
esp8266/tools/esptool.py -p COM1 write_flash 0x00000 esp8266/release/0x00000.bin 0x40000 esp8266/release/0x40000.bin
```
    
- Or Program debug firmware (Debug message from ESP8266 will forward to debug port of Arduino)

```python
esp8266/tools/esptool.py -p COM1 write_flash 0x00000 esp8266/debug/0x00000.bin 0x40000 esp8266/debug/0x40000.bin
```

**3. Wiring:**
![Program Connection diagram](/images/espduino/espdruino_bb.png)

**4. Import arduino library and run example:**


Example for MQTT client
=======

```c
/**
 * \file
 *       ESP8266 MQTT Bridge example
 * \author
 *       Tuan PM <tuanpm@live.com>
 */
#include <SoftwareSerial.h>
#include <espduino.h>
#include <mqtt.h>

SoftwareSerial debugPort(2, 3); // RX, TX
ESP esp(&Serial, &debugPort, 4);
MQTT mqtt(&esp);
boolean wifiConnected = false;

void wifiCb(void* response)
{
  uint32_t status;
  RESPONSE res(response);

  if(res.getArgc() == 1) {
    res.popArgs((uint8_t*)&status, 4);
    if(status == STATION_GOT_IP) {
      debugPort.println("WIFI CONNECTED");
      mqtt.connect("yourserver.com", 1883, false);
      wifiConnected = true;
      //or mqtt.connect("host", 1883); /*without security ssl*/
    } else {
      wifiConnected = false;
      mqtt.disconnect();
    }
    
  }
}

void mqttConnected(void* response)
{
  debugPort.println("Connected");
  mqtt.subscribe("/topic/0"); //or mqtt.subscribe("topic"); /*with qos = 0*/
  mqtt.subscribe("/topic/1");
  mqtt.subscribe("/topic/2");
  mqtt.publish("/topic/0", "data0");

}
void mqttDisconnected(void* response)
{

}
void mqttData(void* response)
{
  RESPONSE res(response);

  debugPort.print("Received: topic=");
  String topic = res.popString();
  debugPort.println(topic);

  debugPort.print("data=");
  String data = res.popString();
  debugPort.println(data);

}
void mqttPublished(void* response)
{

}
void setup() {
  Serial.begin(19200);
  debugPort.begin(19200);
  esp.enable();
  delay(500);
  esp.reset();
  delay(500);
  while(!esp.ready());

  debugPort.println("ARDUINO: setup mqtt client");
  if(!mqtt.begin("DVES_duino", "admin", "Isb_C4OGD4c3", 120, 1)) {
    debugPort.println("ARDUINO: fail to setup mqtt");
    while(1);
  }


  debugPort.println("ARDUINO: setup mqtt lwt");
  mqtt.lwt("/lwt", "offline", 0, 0); //or mqtt.lwt("/lwt", "offline");

/*setup mqtt events */
  mqtt.connectedCb.attach(&mqttConnected);
  mqtt.disconnectedCb.attach(&mqttDisconnected);
  mqtt.publishedCb.attach(&mqttPublished);
  mqtt.dataCb.attach(&mqttData);

  /*setup wifi*/
  debugPort.println("ARDUINO: setup wifi");
  esp.wifiCb.attach(&wifiCb);

  esp.wifiConnect("DVES_HOME","wifipassword");


  debugPort.println("ARDUINO: system started");
}

void loop() {
  esp.process();
  if(wifiConnected) {

  }
}

```

Example for RESTful client
=====
```c
/**
 * \file
 *       ESP8266 RESTful Bridge example
 * \author
 *       Tuan PM <tuanpm@live.com>
 */

#include <SoftwareSerial.h>
#include <espduino.h>
#include <rest.h>

SoftwareSerial debugPort(2, 3); // RX, TX

ESP esp(&Serial, &debugPort, 4);

REST rest(&esp);

boolean wifiConnected = false;

void wifiCb(void* response)
{
  uint32_t status;
  RESPONSE res(response);

  if(res.getArgc() == 1) {
    res.popArgs((uint8_t*)&status, 4);
    if(status == STATION_GOT_IP) {
      debugPort.println("WIFI CONNECTED");
     
      wifiConnected = true;
    } else {
      wifiConnected = false;
    }
    
  }
}

void setup() {
  Serial.begin(19200);
  debugPort.begin(19200);
  esp.enable();
  delay(500);
  esp.reset();
  delay(500);
  while(!esp.ready());

  debugPort.println("ARDUINO: setup rest client");
  if(!rest.begin("yourapihere-com-r2pgihowjx7x.runscope.net")) {
    debugPort.println("ARDUINO: failed to setup rest client");
    while(1);
  }

  /*setup wifi*/
  debugPort.println("ARDUINO: setup wifi");
  esp.wifiCb.attach(&wifiCb);
  esp.wifiConnect("DVES_HOME","wifipassword");
  debugPort.println("ARDUINO: system started");
}

void loop() {
  char response[266];
  esp.process();
  if(wifiConnected) {
    rest.get("/");
    if(rest.getResponse(response, 266) == HTTP_STATUS_OK){
      debugPort.println("RESPONSE: ");
      debugPort.println(response);
    }
    delay(1000);
  }
}
```

MQTT API:
=======
```c
FP<void, void*> connectedCb;
FP<void, void*> disconnectedCb;
FP<void, void*> publishedCb;
FP<void, void*> dataCb;


MQTT(ESP *esp);
boolean begin(const char* client_id, const char* user, const char* pass, uint16_t keep_alive, boolean clean_seasion);
boolean lwt(const char* topic, const char* message, uint8_t qos, uint8_t retain);
boolean lwt(const char* topic, const char* message);
void connect(const char* host, uint32_t port, boolean security);
void connect(const char* host, uint32_t port);
void disconnect();
void subscribe(const char* topic, uint8_t qos);
void subscribe(const char* topic);
void publish(const char* topic, uint8_t *data, uint8_t len, uint8_t qos, uint8_t retain);
void publish(const char* topic, char* data, uint8_t qos, uint8_t retain);
void publish(const char* topic, char* data);
```

REST API:
==============

```c
REST(ESP *e);
boolean begin(const char* host, uint16_t port, boolean security);
boolean begin(const char* host);
void request(const char* path, const char* method, const char* data);
void request(const char* path, const char* method, const char* data, int len);
void get(const char* path, const char* data);
void get(const char* path);
void post(const char* path, const char* data);
void put(const char* path, const char* data);
void del(const char* path, const char* data);

void setTimeout(uint32_t ms);
uint16_t getResponse(char* data, uint16_t maxLen);
void setUserAgent(const char* value);
// Set Content-Type Header
void setContentType(const char* value);
void setHeader(const char* value);
```



