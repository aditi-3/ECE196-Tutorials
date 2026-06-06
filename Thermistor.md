# Interfacing an ATH100K1R25 Thermistor with an ESP32

Ethan Morchy — ECE 196 SP26

This tutorial focuses on building a simple temperature-sensing setup using an ATH100K1R25 thermistor and an ESP32. Inlcuded is an explanation for how the thermistor works in a voltage divider and how to wire it safely to an ESP32 ADC pin. The second part of the firmware includes ADC reading conversion into resistance and temperature. The software section details how to view the temperature as a function of time over a web interface.

![Tutorial image: ESP32](https://www.allelcoelec.com/upfile/images/20/20241007125512593.jpg)

![Tutorial image: ATH100K1R25](https://www.analogtechnologies.com/images/thermistor/new/ATH100K1R25-1.jpg)

## Objectives

- Build a basic thermistor measurement circuit with an ESP32.
- Read analog voltage from the divider using the ESP32 ADC.
- Convert the reading into a temperature estimate.
- Discuss calibration ideas and possible sources of error.

## Supplies

- ESP32 development board
- ATH100K1R25 thermistor
- 100K ohm Resistor for a voltage divider
- Breadboard and jumper wires
- USB cable
- Computer with Arduino IDE

## Hardware

1. Connect a 100K ohm resistor between 3.3V (ESP32 VIN) and a chosen ESP32 ADC pin (**Note: The ESP32 pin must support analog read)**.
2. Connect the ATH100K1R25 thermistor between signal ground and the same ESP32 ADC pin.

```
                 3.3V
                  |
                  |
            [ 100 kΩ resistor ]
                  |
                  +-------------> ESP32 ADC pin
                  |
         [ ATH100K1R25 thermistor ]
                  |
                 GND
```

## Firmware
The firmware section shows how to configure the ADC, sample the analog pin, and print the raw readings over serial. It also goes over calibration constants and the temperature conversion equation.

### ADC Configuration
The ESP32-S3 ADC is set to 12-bit resolution with 11dB attenuation (0–3.3V range). The firmware oversamples 12 readings with a 4ms delay between each to reduce noise.

```cpp
const int THERMISTOR_PIN = 2;
const float ADC_MAX = 4095.0;
const float SERIES_RESISTOR_OHMS = 100000.0;

---

void setup(){
    analogReadResolution(12);
    analogSetPinAttenuation(THERMISTOR_PIN, ADC_11db);
```

### Resistance Calculation
The thermistor resistance is derived from the voltage divider equation:

```cpp
float thermistorOhms = SERIES_RESISTOR_OHMS * adc / (ADC_MAX - adc);
```

### Steinhart-Hart Equation

The coefficients were derived from the ATH100K1R25 datasheet using three calibration points:

| Temperature | Resistance  |
|-------------|-------------|
| -40°C       | 3493 kΩ     |
| 25°C        | 100 kΩ      |
| 100°C       | 6.749 kΩ    |

Coefficients and conversion:

```cpp
const float SH_A = 6.001e-4;
const float SH_B = 2.313e-4;
const float SH_C = 5.988e-8;

float ln_r = log(thermistorOhms);
float inv_t = SH_A + SH_B * ln_r + SH_C * (ln_r * ln_r * ln_r);
float temperatureC = (1.0 / inv_t) - 273.15;
```

### Complete Reading Function

Temperature is returned in Fahrenheit:

```cpp
float readThermistorF() {
  const int samples = 12;
  long total = 0;

  for (int index = 0; index < samples; index += 1) {
    total += analogRead(THERMISTOR_PIN);
    delay(4);
  }

  float adc = total / float(samples);
  adc = constrain(adc, 1.0, ADC_MAX - 1.0);

  float thermistorOhms = SERIES_RESISTOR_OHMS * adc / (ADC_MAX - adc);
  float ln_r = log(thermistorOhms);
  float inv_t = SH_A + SH_B * ln_r + SH_C * (ln_r * ln_r * ln_r);
  float temperatureC = (1.0 / inv_t) - 273.15;
  return temperatureC * 9.0 / 5.0 + 32.0;
}
```

### Serial JSON Protocol

The firmware sends a JSON message every second. On each cycle, it reads temperature, adjusts the heater, and sends a JSON telemetry line over USB serial:

```cpp
void sendTelemetry() {
  float actualTemperatureF = readThermistorF();
  applyHeaterControl(actualTemperatureF);

  JsonDocument doc;
  doc["actualTemperature"] = actualTemperatureF;
  doc["powerOn"] = powerOn;
  doc["targetTemperature"] = targetTemperatureF;
  doc["heaterDuty"] = heaterDutyValue;
  doc["heaterDutyPercent"] = (heaterDutyValue / float(PWM_MAX_DUTY)) * 100.0;
  doc["heaterAveragePowerWatts"] = calculateAveragePowerWatts(heaterDutyValue);
  doc["maxSafeTemperature"] = MAX_SAFE_TEMPERATURE_F;

  String output;
  serializeJson(doc, output);
  Serial.println(output);
}
```

## Software
An example frontend is a single-page application (`index.html`, `styles.css`, `app.js`) that communicates with the ESP32 over USB serial using the Web Serial API.

### Serial Connection

The browser opens a serial port at 115200 baud using the Web Serial API:

```js
async function connectSerial() {
  try {
    serialPort = await navigator.serial.requestPort();
    await serialPort.open({ baudRate: 115200 });

    const decoder = new TextDecoderStream();
    serialPort.readable.pipeTo(decoder.writable);
    serialReader = decoder.readable.getReader();

    const encoder = new TextEncoderStream();
    encoder.readable.pipeTo(serialPort.writable);
    serialWriter = encoder.writable.getWriter();

    elements.connectButton.textContent = "Disconnect";
    render();
    readSerialLoop();
  } catch (error) {
    console.warn("Serial connect failed:", error);
    serialPort = null;
  }
}
```

### Reading Telemetry Data
The read loop continuously parses newline-delimited JSON lines from the ESP32:

```js
async function readSerialLoop() {
  while (serialReader) {
    const { value, done } = await serialReader.read();
    if (done) break;

    serialBuffer += value;
    const lines = serialBuffer.split("\n");
    serialBuffer = lines.pop();

    for (const line of lines) {
      const trimmed = line.trim();
      if (!trimmed) continue;
      try {
        const data = JSON.parse(trimmed);
        if (typeof data.actualTemperature === "number") {
          state.actualTemperature = data.actualTemperature;
          recordTemperatureSample();
          render();
        }
      } catch {
        // not JSON, ignore
      }
    }
  }
}
```

### Temperature Graph
A 60-second rolling window line chart is drawn on a canvas element showing actual vs target temperature:

```js
function drawTemperatureGraph() {
  // ... canvas sizing, grid lines, axes ...

  function drawLine(getTemperature, color, widthSize) {
    if (state.temperatureHistory.length < 2) return;

    graphContext.beginPath();
    state.temperatureHistory.forEach((sample, index) => {
      const point = toPoint(sample, getTemperature(sample));
      if (index === 0) {
        graphContext.moveTo(point.x, point.y);
      } else {
        graphContext.lineTo(point.x, point.y);
      }
    });
    graphContext.lineWidth = widthSize;
    graphContext.strokeStyle = color;
    graphContext.stroke();
  }

  drawLine((sample) => sample.targetTemperature, "#008f07", 2);
  drawLine((sample) => sample.actualTemperature, "#c95731", 4);
}
```

## Testing Ideas
Exposing the thermistor to various temperatures can aid with calibration.

1. Read room temperature and confirm the values are stable.
2. Warm the thermistor slightly by hand and observe the ADC change.
3. Place the thermistor on a metal container filled with cold water.
4. Place the thermistor on a metal container filled with hot water.

## Example
The ATH100K1R25 thermistor was used to derive heat from a self heating thermos project. You may view our solution and usage of the thermistor at [our website](https://blueandgold.my.canva.site/everwarm/our-solution). The thermistor is the green polygon inside the second layer of the water bottle:

![Smart Water Bottle](https://blueandgold.my.canva.site/everwarm/_assets/media/c097fa581176f753a673f094139f62e0.png)

Below is an example layout for a web ui frontend to read the thermistor temperature:
![Project Example Website](https://blueandgold.my.canva.site/everwarm/_assets/media/af55b40dfaa6c5e595e0f46724f266a6.jpg)

## AI Disclosure Policy
AI was used to create the ASCII diagram in section "Hardware" and table alignment in other sections. The code referenced is taken from the project code and may have been generated using AI.

## Resources

- [ATH100K1R25 Datasheet](https://www.analogtechnologies.com/document/ATH100K1R25.pdf)
- [Voltage Divider Theory](https://www.electronics-tutorials.ws/dccircuits/voltage-divider.html)
