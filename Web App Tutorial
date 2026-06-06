# MQTT Publish-Subscribe Communication with ESP32

## Objective

This tutorial demonstrates how to implement MQTT publish-subscribe communication using an ESP32 development board. You will learn how MQTT brokers, publishers, and subscribers interact for wireless communication between IoT devices. The tutorial also introduces  concepts related to message routing and asynchronous communication. By the end, you should be able to publish sensor data and subscribe to commands using an MQTT broker.

---

# Introduction

## What is MQTT?

MQTT is a messaging protocol designed for IoT applications. Instead of devices communicating directly with each other, they communicate through a central broker. MQTT has low bandwidth usage and if very efficient for wireless devices. It is able to support multiple subscribers and is scalable.

---

### ECE 140A: Network Communication Systems

MQTT demonstrates a practical implementation of communication systems covered in ECE coursework.

## MQTT Publish-Subscribe Flowchart

![Tutorial image: ESP32](OnePageMarketingPlan.jpg)

*Figure 1: MQTT communication flow between publisher, broker, and subscriber.*

---

# Materials Required

## Components                         
### Hardware

* ESP32 Development Board
* USB-C or Micro USB cable
* Wi-Fi connection

### Software

* Visual Studio Code
* PlatformIO Extension
* MQTT Explorer
---

# Step 1: Install VS Code and PlatformIO

Download and install Visual Studio Code:

[INSERT SCREENSHOT OF VS CODE INSTALLATION]

After installation:

Open VS Code.
Navigate to Extensions.
Search for "PlatformIO IDE".
Install the extension.

![Tutorial image: ESP32](vscode.png)

---
# Step 2: Create a New PlatformIO Project
Open PlatformIO Home.
* Click "New Project".
* Enter a project name: MQTT_ESP32
* Select your ESP32 board.


Example:

Espressif ESP32 Dev Module
* Choose: Arduino as the framework.

Click Finish.

![Tutorial image: ESP32](saved.png)

# Step 3: Configure Dependencies

Open:

platformio.ini

Add the MQTT library:

[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino

lib_deps =
    knolleary/PubSubClient

Save the file.

PlatformIO will automatically download the required library.

---

# Step 4: Connect to Wi-Fi

Create: src/main.cpp

Add: 

    #include <WiFi.h>
    const char* ssid = "YOUR_WIFI_USERNAME";
    const char* password = "YOUR_PASSWORD";
    
### Create the Wi-Fi connection function:

    void setup_wifi() {
        WiFi.begin(ssid, password);

        while (WiFi.status() != WL_CONNECTED) {
            delay(500);
        }
    }

### Verify Connection

Open Serial Monitor.

#### Expected output:

```text
WiFi Connected
IP Address: xxx.xxx.xxx.xxx
```
---
# Step 5: Configure MQTT Broker

### Add to main.cpp:

    #include <PubSubClient.h>

    WiFiClient espClient;
    PubSubClient client(espClient);

    const char* mqtt_server = "broker.hivemq.com";

### Configure the broker:

client.setServer(mqtt_server, 1883);
# Step 6: Create MQTT Reconnect Function

### Add to main.cpp:

    void reconnect() {

        while (!client.connected()) {

            if (client.connect("ESP32Client")) {

                client.subscribe("ece196/demo");

            } else {

                delay(5000);

            }
        }
    }

This function ensures the ESP32 automatically reconnects if the broker connection is lost.

# Step 7: Subscribe to a Topic

### Create a callback function:

    void callback(char* topic,
                byte* payload,
                unsigned int length) {

        Serial.print("Received: ");

        for (int i = 0; i < length; i++) {
            Serial.print((char)payload[i]);
        }

        Serial.println();
    }

Register the callback:

    client.setCallback(callback);

Subscribe during reconnection:

    client.subscribe("ece196/demo");

## What it should look like
    #include <WiFi.h>
    #include <PubSubClient.h>

    WiFiClient espClient;
    PubSubClient client(espClient);

    void callback(char* topic,
                byte* payload,
                unsigned int length) {

        Serial.print("Received: ");

        for (int i = 0; i < length; i++) {
            Serial.print((char)payload[i]);
        }

        Serial.println();
    }
    void setup(){

    Serial.begin(115200);

    setup_wifi();

    client.setServer(mqtt_server, 1883);

    client.setCallback(callback);
}

# Step 8: Publish Messages

### Put inside the void loop:
    void loop()
    {
        if (!client.connected())
        {
            reconnect();
        }

        client.loop();

        static unsigned long lastMsg = 0;

        if (millis() - lastMsg > 5000)
        {
            lastMsg = millis();

            client.publish(
                "ece196/demo",
                "Hello from ESP32"
            );

            Serial.println("Published message");
        }
    }

The ESP32 will publish a message every five seconds.

# Step 9: Connect 

### Connect the ESP32 via USB.

Select: PlatformIO → Upload or press:

Ctrl + Alt + U

# Step 10: Monitor Serial Output

Open the serial monitor or use Ctrl + Alt + M

Expected output:

* WiFi Connected
* MQTT Connected
* Publishing Message

# Resources

### MQTT Documentation
* https://mqtt.org
* https://www.hivemq.com
* https://platformio.org


