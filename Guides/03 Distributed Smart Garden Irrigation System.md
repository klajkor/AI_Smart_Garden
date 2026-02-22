# Distributed Smart Garden Irrigation System - Complete Implementation

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                     MQTT Broker (Mosquitto)              │
└────────┬──────────────────┬─────────────────┬───────────┘
         │                  │                 │
    ┌────▼─────┐      ┌────▼─────┐     ┌────▼─────┐
    │ ESP32    │      │ ESP8266  │     │ ESP8266  │
    │ Sensor   │      │ Valve    │     │ Valve    │
    │ Hub      │      │ Zone 1   │     │ Zone 2   │
    │          │      │          │     │          │
    │ • Soil 1 │      │ • Relay  │     │ • Relay  │
    │ • Soil 2 │      │ • LED    │     │ • LED    │
    │ • Soil 3 │      └──────────┘     └──────────┘
    │ • Temp   │            │                │
    └──────────┘            │                │
         │                  │                │
    ┌────▼─────┐      ┌────▼─────┐     ┌────▼─────┐
    │ Node-RED │◄─────┤ ESP8266  │     │ Zigbee   │
    │          │      │ Valve    │     │ Sensors  │
    └────┬─────┘      │ Zone 3   │     │ (opt)    │
         │            │          │     └──────────┘
    ┌────▼─────┐      │ • Relay  │
    │ InfluxDB │      │ • LED    │
    │ Grafana  │      └──────────┘
    └──────────┘
```

---

## Part 1: Device Recommendations

### **Sensor Hub: ESP32 DevKit v1** ✅

**Why ESP32 over ESP8266 for sensors?**

| Feature | ESP32 | ESP8266 | Winner |
|---------|-------|---------|--------|
| **ADC Pins** | 18 pins (6 usable) | 1 pin (A0 only) | **ESP32** |
| **ADC Resolution** | 12-bit (0-4095) | 10-bit (0-1023) | **ESP32** |
| **ADC Stability** | Better, with calibration | Noisy, non-linear | **ESP32** |
| **GPIO Pins** | 34 total | 11 usable | **ESP32** |
| **I2C/SPI** | 2× each | 1× each | **ESP32** |
| **Price** | $4-6 | $2-3 | ESP8266 |
| **Power** | 160-240mA | 80-170mA | ESP8266 |

**Recommendation**: **ESP32 DevKit v1** for sensor hub
- Can read 3 analog soil sensors simultaneously
- Room for expansion (temp/humidity, light sensors)
- Better ADC accuracy for soil moisture
- Can handle I2C sensors (BME280, BH1750, etc.)

**Buy**: 
- AliExpress: "ESP32 DevKit v1" (~$4)
- Amazon: "HiLetgo ESP32" (~$10 for 2-pack)

---

### **Valve Controllers: ESP8266 NodeMCU/D1 Mini** ✅

**Why ESP8266 for valve control?**

| Requirement | ESP8266 Capability | Sufficient? |
|-------------|-------------------|-------------|
| Control 1 relay | 1 GPIO needed | ✅ Yes |
| WiFi/MQTT | Yes, stable | ✅ Yes |
| Status LED | 1 GPIO | ✅ Yes |
| Cost per zone | $2-3 | ✅ Very affordable |
| OTA updates | Yes | ✅ Yes |
| Power | 80mA (low) | ✅ Yes |

**Recommendation**: **Wemos D1 Mini** (preferred) or **NodeMCU v3**
- **D1 Mini**: Compact (34×26mm), breadboard-friendly, $2-3
- **NodeMCU**: Larger but easier to prototype, $3-4

**Buy**: 
- AliExpress: "Wemos D1 Mini" 5-pack (~$8)
- Amazon: "D1 Mini ESP8266" (~$15 for 5)

---

## Part 2: Hardware Components List

### Complete Shopping List

| Component | Quantity | Unit Price | Total | Notes |
|-----------|----------|------------|-------|-------|
| **ESP32 DevKit v1** | 1 | $5 | $5 | Sensor hub |
| **Wemos D1 Mini** | 3 | $3 | $9 | Valve controllers |
| **Capacitive Soil Sensors v1.2** | 3 | $2 | $6 | Corrosion resistant |
| **5V Relay Module (1-channel)** | 3 | $1 | $3 | Opto-isolated preferred |
| **BME280 Sensor** | 1 | $3 | $3 | Temp/humidity/pressure (optional) |
| **5V 2A Power Supply** | 4 | $3 | $12 | Micro-USB or USB-C |
| **Dupont Wire Kit** | 1 | $5 | $5 | Male-female, male-male |
| **Breadboards (mini)** | 4 | $1 | $4 | For prototyping |
| **12V Solenoid Valves** | 3 | $8 | $24 | 1/2" or 3/4" NPT |
| **12V Power Supply** | 1 | $10 | $10 | For valves (2A+) |
| **Project Boxes (IP65)** | 4 | $3 | $12 | Waterproof enclosures |
| **Misc** (wire, terminals) | - | - | $10 | - |
| **Total** | | | **$103** | |

---

## Part 3: ESPHome Configuration Files

### 3.1 Sensor Hub (ESP32)

**File: `sensor-hub-esp32.yaml`**

```yaml
substitutions:
  device_name: garden-sensor-hub
  friendly_name: Garden Sensor Hub
  mqtt_topic: garden/sensors

esphome:
  name: ${device_name}
  platform: ESP32
  board: esp32dev
  comment: "Central sensor hub for 3-zone irrigation"

# Enable logging
logger:
  level: INFO
  logs:
    sensor: INFO
    mqtt: INFO

# Enable Home Assistant API (optional)
api:
  encryption:
    key: !secret api_encryption_key

# OTA updates
ota:
  password: !secret ota_password

# WiFi configuration
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  
  # Static IP (recommended for reliability)
  manual_ip:
    static_ip: 192.168.1.100
    gateway: 192.168.1.1
    subnet: 255.255.255.0
  
  # Fast connect for reliability
  fast_connect: true
  
  # Fallback hotspot
  ap:
    ssid: "${device_name}-fallback"
    password: "fallback123"

# Captive portal for configuration
captive_portal:

# MQTT Configuration
mqtt:
  broker: !secret mqtt_broker
  port: 1883
  username: !secret mqtt_username
  password: !secret mqtt_password
  topic_prefix: ${mqtt_topic}
  discovery: true
  discovery_prefix: homeassistant
  birth_message:
    topic: ${mqtt_topic}/status
    payload: online
    qos: 1
    retain: true
  will_message:
    topic: ${mqtt_topic}/status
    payload: offline
    qos: 1
    retain: true

# Web server for diagnostics
web_server:
  port: 80
  auth:
    username: admin
    password: !secret web_password

# I2C bus for BME280 (optional)
i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true
  id: bus_a

# Status LED (built-in)
status_led:
  pin: GPIO2

# ============================================================================
# SENSORS
# ============================================================================

sensor:
  # System sensors
  - platform: wifi_signal
    name: "${friendly_name} WiFi Signal"
    update_interval: 60s
    filters:
      - sliding_window_moving_average:
          window_size: 5
          send_every: 5
    
  - platform: uptime
    name: "${friendly_name} Uptime"
    update_interval: 60s
    
  # ==================== ZONE 1 SOIL MOISTURE ====================
  - platform: adc
    pin: GPIO36  # VP pin - ADC1_CH0
    name: "${friendly_name} Zone 1 Moisture Raw"
    id: moisture_raw_zone1
    attenuation: 11db  # 0-3.3V range
    update_interval: 30s
    internal: true  # Don't publish raw values
    filters:
      - multiply: 0.000244141  # Convert to voltage (1/4096 for 12-bit ADC)
      - median:  # Remove noise
          window_size: 5
          send_every: 3
    
  - platform: template
    name: "${friendly_name} Zone 1 Moisture"
    id: moisture_percent_zone1
    unit_of_measurement: "%"
    device_class: moisture
    state_class: measurement
    accuracy_decimals: 1
    icon: "mdi:water-percent"
    lambda: |-
      // Calibration values - MEASURE THESE FOR YOUR SENSORS
      const float DRY_VOLTAGE = 2.85;   // Sensor in dry air
      const float WET_VOLTAGE = 1.30;   // Sensor in water
      
      float voltage = id(moisture_raw_zone1).state;
      
      // Convert to percentage (inverted: lower voltage = more moisture)
      float percent = ((DRY_VOLTAGE - voltage) / (DRY_VOLTAGE - WET_VOLTAGE)) * 100.0;
      
      // Clamp between 0-100
      if (percent > 100.0) return 100.0;
      if (percent < 0.0) return 0.0;
      
      return percent;
    update_interval: 30s
    filters:
      - sliding_window_moving_average:
          window_size: 3
          send_every: 1

  # ==================== ZONE 2 SOIL MOISTURE ====================
  - platform: adc
    pin: GPIO39  # VN pin - ADC1_CH3
    name: "${friendly_name} Zone 2 Moisture Raw"
    id: moisture_raw_zone2
    attenuation: 11db
    update_interval: 30s
    internal: true
    filters:
      - multiply: 0.000244141
      - median:
          window_size: 5
          send_every: 3
    
  - platform: template
    name: "${friendly_name} Zone 2 Moisture"
    id: moisture_percent_zone2
    unit_of_measurement: "%"
    device_class: moisture
    state_class: measurement
    accuracy_decimals: 1
    icon: "mdi:water-percent"
    lambda: |-
      const float DRY_VOLTAGE = 2.85;
      const float WET_VOLTAGE = 1.30;
      float voltage = id(moisture_raw_zone2).state;
      float percent = ((DRY_VOLTAGE - voltage) / (DRY_VOLTAGE - WET_VOLTAGE)) * 100.0;
      if (percent > 100.0) return 100.0;
      if (percent < 0.0) return 0.0;
      return percent;
    update_interval: 30s
    filters:
      - sliding_window_moving_average:
          window_size: 3
          send_every: 1

  # ==================== ZONE 3 SOIL MOISTURE ====================
  - platform: adc
    pin: GPIO34  # ADC1_CH6
    name: "${friendly_name} Zone 3 Moisture Raw"
    id: moisture_raw_zone3
    attenuation: 11db
    update_interval: 30s
    internal: true
    filters:
      - multiply: 0.000244141
      - median:
          window_size: 5
          send_every: 3
    
  - platform: template
    name: "${friendly_name} Zone 3 Moisture"
    id: moisture_percent_zone3
    unit_of_measurement: "%"
    device_class: moisture
    state_class: measurement
    accuracy_decimals: 1
    icon: "mdi:water-percent"
    lambda: |-
      const float DRY_VOLTAGE = 2.85;
      const float WET_VOLTAGE = 1.30;
      float voltage = id(moisture_raw_zone3).state;
      float percent = ((DRY_VOLTAGE - voltage) / (DRY_VOLTAGE - WET_VOLTAGE)) * 100.0;
      if (percent > 100.0) return 100.0;
      if (percent < 0.0) return 0.0;
      return percent;
    update_interval: 30s
    filters:
      - sliding_window_moving_average:
          window_size: 3
          send_every: 1

  # ==================== BME280 ENVIRONMENTAL SENSOR (OPTIONAL) ====================
  - platform: bme280
    temperature:
      name: "${friendly_name} Temperature"
      id: garden_temperature
      oversampling: 16x
      filters:
        - offset: -1.2  # Calibration offset if needed
    humidity:
      name: "${friendly_name} Humidity"
      id: garden_humidity
      oversampling: 16x
    pressure:
      name: "${friendly_name} Pressure"
      id: garden_pressure
      oversampling: 16x
    address: 0x76
    update_interval: 60s

  # ==================== BATTERY VOLTAGE MONITOR (if using backup battery) ====================
  # - platform: adc
  #   pin: GPIO35
  #   name: "${friendly_name} Battery Voltage"
  #   attenuation: 11db
  #   filters:
  #     - multiply: 2.0  # Voltage divider compensation
  #   update_interval: 300s

# Binary sensors
binary_sensor:
  - platform: status
    name: "${friendly_name} Status"

# Text sensors
text_sensor:
  - platform: wifi_info
    ip_address:
      name: "${friendly_name} IP Address"
    ssid:
      name: "${friendly_name} SSID"
    bssid:
      name: "${friendly_name} BSSID"
    mac_address:
      name: "${friendly_name} MAC Address"
  
  - platform: version
    name: "${friendly_name} ESPHome Version"

# Button entities for remote actions
button:
  - platform: restart
    name: "${friendly_name} Restart"
  
  - platform: template
    name: "${friendly_name} Calibrate Sensors"
    icon: "mdi:auto-fix"
    on_press:
      - logger.log: "Calibration mode - check logs for raw voltage values"

# Diagnostic switch
switch:
  - platform: template
    name: "${friendly_name} Enable Debug Logging"
    id: debug_mode
    icon: "mdi:bug"
    restore_mode: RESTORE_DEFAULT_OFF
    optimistic: true
    on_turn_on:
      - logger.set_level: DEBUG
    on_turn_off:
      - logger.set_level: INFO

# Intervals for periodic actions
interval:
  - interval: 300s  # Every 5 minutes
    then:
      - logger.log:
          format: "Zone readings - Z1: %.1f%%, Z2: %.1f%%, Z3: %.1f%%"
          args: [ 'id(moisture_percent_zone1).state', 'id(moisture_percent_zone2).state', 'id(moisture_percent_zone3).state' ]
```

---

### 3.2 Valve Controller - Zone 1 (ESP8266)

**File: `valve-zone1-esp8266.yaml`**

```yaml
substitutions:
  device_name: garden-valve-zone1
  friendly_name: Garden Valve Zone 1
  zone_number: "1"
  zone_name: "Vegetables"
  mqtt_topic: garden/valves/zone1

esphome:
  name: ${device_name}
  platform: ESP8266
  board: d1_mini  # or nodemcuv2
  comment: "Valve controller for Zone 1 - ${zone_name}"

logger:
  level: INFO

api:
  encryption:
    key: !secret api_encryption_key

ota:
  password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  
  manual_ip:
    static_ip: 192.168.1.11${zone_number}  # 192.168.1.111
    gateway: 192.168.1.1
    subnet: 255.255.255.0
  
  fast_connect: true
  
  ap:
    ssid: "${device_name}-fallback"
    password: "fallback123"

captive_portal:

mqtt:
  broker: !secret mqtt_broker
  port: 1883
  username: !secret mqtt_username
  password: !secret mqtt_password
  topic_prefix: ${mqtt_topic}
  discovery: true
  birth_message:
    topic: ${mqtt_topic}/status
    payload: online
    qos: 1
    retain: true
  will_message:
    topic: ${mqtt_topic}/status
    payload: offline
    qos: 1
    retain: true

web_server:
  port: 80
  auth:
    username: admin
    password: !secret web_password

# ============================================================================
# VALVE CONTROL
# ============================================================================

switch:
  # Main valve relay
  - platform: gpio
    pin: GPIO5  # D1 on D1 Mini, GPIO5 on NodeMCU
    name: "${friendly_name} Valve"
    id: valve_relay
    icon: "mdi:valve"
    restore_mode: ALWAYS_OFF  # Critical safety feature
    on_turn_on:
      - logger.log: "Zone ${zone_number} (${zone_name}) valve OPENED"
      - switch.turn_on: valve_led
      - lambda: |-
          id(valve_on_time) = millis();
          id(total_activations) += 1;
      # Safety timeout - auto-shutoff after 30 minutes
      - delay: 1800s  # 30 minutes
      - switch.turn_off: valve_relay
      - logger.log: 
          level: WARN
          format: "Zone ${zone_number} auto-shutoff after 30 minutes (safety)"
    on_turn_off:
      - logger.log: "Zone ${zone_number} (${zone_name}) valve CLOSED"
      - switch.turn_off: valve_led
      - lambda: |-
          if (id(valve_on_time) > 0) {
            unsigned long duration = (millis() - id(valve_on_time)) / 1000;
            id(total_runtime) += duration;
            id(valve_on_time) = 0;
            ESP_LOGI("valve", "Watering duration: %lu seconds", duration);
          }
  
  # Visual indicator LED
  - platform: gpio
    pin: GPIO4  # D2 on D1 Mini
    name: "${friendly_name} LED"
    id: valve_led
    internal: true  # Don't expose to MQTT separately
    restore_mode: ALWAYS_OFF

  # Manual override switch (disable automation)
  - platform: template
    name: "${friendly_name} Manual Mode"
    id: manual_mode
    icon: "mdi:hand-back-right"
    restore_mode: RESTORE_DEFAULT_OFF
    optimistic: true

  # Restart button
  - platform: restart
    name: "${friendly_name} Restart"

# ============================================================================
# SENSORS & DIAGNOSTICS
# ============================================================================

sensor:
  - platform: wifi_signal
    name: "${friendly_name} WiFi Signal"
    update_interval: 60s
    
  - platform: uptime
    name: "${friendly_name} Uptime"
    update_interval: 60s
    filters:
      - lambda: return x / 3600.0;  # Convert to hours
    unit_of_measurement: "h"
    accuracy_decimals: 1

  # Total runtime counter
  - platform: template
    name: "${friendly_name} Total Runtime"
    id: total_runtime_sensor
    unit_of_measurement: "h"
    device_class: duration
    state_class: total_increasing
    accuracy_decimals: 1
    icon: "mdi:timer"
    lambda: |-
      return id(total_runtime) / 3600.0;
    update_interval: 60s

  # Total activations counter
  - platform: template
    name: "${friendly_name} Total Activations"
    id: total_activations_sensor
    state_class: total_increasing
    accuracy_decimals: 0
    icon: "mdi:counter"
    lambda: |-
      return id(total_activations);
    update_interval: 60s

  # Current session runtime
  - platform: template
    name: "${friendly_name} Current Runtime"
    unit_of_measurement: "min"
    accuracy_decimals: 0
    icon: "mdi:timer-sand"
    lambda: |-
      if (id(valve_relay).state && id(valve_on_time) > 0) {
        return (millis() - id(valve_on_time)) / 60000.0;
      }
      return 0;
    update_interval: 10s

binary_sensor:
  - platform: status
    name: "${friendly_name} Status"
  
  # Valve state (redundant with switch, but useful for history)
  - platform: template
    name: "${friendly_name} Valve State"
    device_class: opening
    lambda: |-
      return id(valve_relay).state;

text_sensor:
  - platform: wifi_info
    ip_address:
      name: "${friendly_name} IP Address"
    ssid:
      name: "${friendly_name} SSID"
  
  - platform: version
    name: "${friendly_name} ESPHome Version"
  
  # Zone information
  - platform: template
    name: "${friendly_name} Zone Info"
    icon: "mdi:information"
    lambda: |-
      return {"Zone ${zone_number}: ${zone_name}"};
    update_interval: 3600s

# ============================================================================
# GLOBAL VARIABLES (for runtime tracking)
# ============================================================================

globals:
  - id: valve_on_time
    type: unsigned long
    restore_value: no
    initial_value: '0'
  
  - id: total_runtime
    type: unsigned long
    restore_value: yes  # Persist across reboots
    initial_value: '0'
  
  - id: total_activations
    type: int
    restore_value: yes
    initial_value: '0'

# ============================================================================
# SAFETY & MONITORING
# ============================================================================

# Heartbeat interval
interval:
  - interval: 60s
    then:
      - if:
          condition:
            switch.is_on: valve_relay
          then:
            - logger.log:
                level: INFO
                format: "Zone ${zone_number} valve active - runtime: %.0f minutes"
                args: ['(millis() - id(valve_on_time)) / 60000.0']

# Status on boot
on_boot:
  - priority: 600
    then:
      - logger.log: "Zone ${zone_number} (${zone_name}) controller started"
      - logger.log:
          format: "Statistics - Runtime: %.1fh, Activations: %d"
          args: ['id(total_runtime) / 3600.0', 'id(total_activations)']
```

---

### 3.3 Valve Controller - Zone 2 (ESP8266)

**File: `valve-zone2-esp8266.yaml`**

```yaml
# Copy valve-zone1-esp8266.yaml and change:
substitutions:
  device_name: garden-valve-zone2
  friendly_name: Garden Valve Zone 2
  zone_number: "2"
  zone_name: "Flowers"  # Change this
  mqtt_topic: garden/valves/zone2

# Static IP change:
wifi:
  manual_ip:
    static_ip: 192.168.1.112  # Note: ends in 112
```

---

### 3.4 Valve Controller - Zone 3 (ESP8266)

**File: `valve-zone3-esp8266.yaml`**

```yaml
# Copy valve-zone1-esp8266.yaml and change:
substitutions:
  device_name: garden-valve-zone3
  friendly_name: Garden Valve Zone 3
  zone_number: "3"
  zone_name: "Lawn"  # Change this
  mqtt_topic: garden/valves/zone3

# Static IP change:
wifi:
  manual_ip:
    static_ip: 192.168.1.113  # Note: ends in 113
```

---

### 3.5 Secrets File

Create `secrets.yaml` in the same directory:

```yaml
# secrets.yaml
wifi_ssid: "YourWiFiSSID"
wifi_password: "YourWiFiPassword"

mqtt_broker: "192.168.1.100"  # Your Docker host IP
mqtt_username: "mqtt_user"
mqtt_password: "your_mqtt_password"

api_encryption_key: "YOUR_32_CHAR_KEY_HERE="  # Generate with: esphome random-key
ota_password: "your_ota_password"
web_password: "your_web_password"
```

---

## Part 4: Wiring Diagrams

### 4.1 ESP32 Sensor Hub Wiring

```
┌─────────────────────────────────────────────────────────────────┐
│                    ESP32 DevKit v1                              │
│                                                                  │
│  3V3 ●─────────────────┬────────────┬────────────┬──────────●  │
│                        │            │            │          │   │
│  GND ●─────────────────┼────────────┼────────────┼──────────┼─● │
│                        │            │            │          │   │
│ GPIO36 (VP) ●──────────┼────────────┼────────────┼──────────┤   │
│                        │            │            │          │   │
│ GPIO39 (VN) ●──────────┼────────────┼────────────┼──────────┤   │
│                        │            │            │          │   │
│ GPIO34 ●───────────────┼────────────┼────────────┼──────────┤   │
│                        │            │            │          │   │
│ GPIO21 (SDA) ●─────────┼────────────┼────────────┼──────────┼─● │
│                        │            │            │          │   │
│ GPIO22 (SCL) ●─────────┼────────────┼────────────┼──────────┼─● │
│                        │            │            │          │   │
│  GND ●─────────────────┼────────────┼────────────┼──────────┼─● │
│                        │            │            │          │   │
└────────────────────────┼────────────┼────────────┼──────────┼───┘
                         │            │            │          │
                         │            │            │          │
        ┌────────────────▼──┐  ┌──────▼──────┐  ┌─▼──────────┴──┐
        │  Capacitive Soil  │  │ Capacitive  │  │  Capacitive   │
        │  Moisture Sensor  │  │    Soil     │  │     Soil      │
        │     (Zone 1)      │  │  Moisture   │  │   Moisture    │
        │                   │  │  (Zone 2)   │  │   (Zone 3)    │
        │  VCC  AOUT  GND   │  │ VCC AOUT GND│  │ VCC AOUT GND  │
        │   │    │     │    │  │  │   │   │  │  │  │   │   │   │
        │   │    │     │    │  │  │   │   │  │  │  │   │   │   │
        └───┼────┼─────┼────┘  └──┼───┼───┼──┘  └──┼───┼───┼───┘
            │    │     │          │   │   │        │   │   │
       To 3V3  To 36  To GND   To 3V3 To 39 GND  3V3 To 34 GND

Optional BME280 (I2C):
        ┌──────────────┐
        │   BME280     │
        │              │
        │ VCC SDA SCL  │
        │  │   │   │   │
        └──┼───┼───┼───┘
           │   │   │
       To 3V3  21  22
```

**Pin Connections Table:**

| Component | ESP32 Pin | Wire Color | Notes |
|-----------|-----------|------------|-------|
| **Zone 1 Sensor** | | | |
| VCC | 3V3 | Red | Do not use 5V! |
| AOUT | GPIO36 (VP) | Yellow | Analog signal |
| GND | GND | Black | |
| **Zone 2 Sensor** | | | |
| VCC | 3V3 | Red | |
| AOUT | GPIO39 (VN) | Yellow | |
| GND | GND | Black | |
| **Zone 3 Sensor** | | | |
| VCC | 3V3 | Red | |
| AOUT | GPIO34 | Yellow | |
| GND | GND | Black | |
| **BME280 (optional)** | | | |
| VCC | 3V3 | Red | |
| SDA | GPIO21 | Green | I2C data |
| SCL | GPIO22 | White | I2C clock |
| GND | GND | Black | |

**Important Notes:**
1. ⚠️ **Use 3.3V, NOT 5V** for capacitive sensors with ESP32
2. Keep sensor wires **under 2 meters** for stable readings
3. Route analog wires away from power cables (noise reduction)
4. Use twisted pair for longer runs

---

### 4.2 ESP8266 Valve Controller Wiring (Each Zone)

```
┌────────────────────────────────────────────────────────────┐
│              Wemos D1 Mini / NodeMCU                       │
│                                                             │
│  5V  ●───────────────────────┐                            │
│                               │                            │
│  GND ●───────────────────┬───┼────────┐                   │
│                          │   │        │                   │
│  D1 (GPIO5) ●────────────┼───┼────────┼───────●           │
│                          │   │        │                   │
│  D2 (GPIO4) ●────────────┼───┼────────┼───────────●       │
│                          │   │        │                   │
└──────────────────────────┼───┼────────┼───────────────────┘
                           │   │        │
                           │   │        │
                ┌──────────▼───▼───┐    │
                │   5V Relay Module│    │
                │  ┌──────────┐    │    │
                │  │          │    │    │
                │  │  Relay   │    │    │
                │  │          │    │    │
                │  └─┬────┬───┘    │    │
                │ VCC│IN  │GND     │    │
                │  │ │    │        │    │
                └──┼─┼────┼────────┘    │
                   │ │    │             │
              To 5V│ │    To GND        │
                   │ │                  │
              From D1 (GPIO5)           │
                     │                  │
                     │                  │
    ┌────────────────┼──────────────────┼──────────┐
    │  Relay Contacts│                  │          │
    │                │                  │          │
    │        NO ●────┘             COM ●┘          │
    │                                              │
    │        NC ●  (not used)                      │
    │                                              │
    └───────────┬──────────────────────────────────┘
                │
                │
        ┌───────▼────────┐
        │  12V Solenoid  │
        │     Valve      │
        │                │
        │  (+)      (-)  │
        │   │        │   │
        └───┼────────┼───┘
            │        │
        From 12V PSU│
                    │
                To 12V PSU GND

Optional Status LED:
        ┌─────────┐
        │   LED   │
        │  (Red)  │
        │  │   │  │
        └──┼───┼──┘
           │   │
      From D2  │
           │   └─── 220Ω Resistor ─── To GND
           │
```

**Component Details:**

**5V Relay Module Specifications:**
- Type: 1-channel 5V relay module
- Trigger: Active LOW (triggers when GPIO goes LOW)
- Isolation: Opto-isolated (recommended)
- Contact rating: 10A @ 250VAC / 10A @ 30VDC
- **Buy**: Search "1 channel 5V relay module opto-isolated" (~$1-2)

**Wiring Table:**

| Component | D1 Mini Pin | NodeMCU Pin | Notes |
|-----------|-------------|-------------|-------|
| **Relay Module** | | | |
| VCC | 5V | VIN | Power |
| IN (Signal) | D1 (GPIO5) | D1 (GPIO5) | Control |
| GND | GND | GND | Ground |
| **Status LED** | | | |
| Anode (+) | D2 (GPIO4) | D2 (GPIO4) | Via 220Ω |
| Cathode (-) | GND | GND | |
| **12V Solenoid Valve** | | | |
| (+) Terminal | Relay NO | Relay NO | Switched +12V |
| (-) Terminal | 12V PSU GND | 12V PSU GND | Direct |
| **Power** | | | |
| ESP8266 | 5V USB | 5V USB | Micro-USB |
| Relay | From ESP 5V | From ESP VIN | Shared |
| Valve | 12V 2A PSU | 12V 2A PSU | Separate |

**Circuit Diagram (ASCII Art):**

```
12V Power Supply
      (+)  (-)
       │    │
       │    └────────────────┬──────────── GND Rail
       │                     │
       │                 (Valve -)
       │                     
       └─── Relay COM
               │
               │ (Relay switching)
               │
            Relay NO ──────── (Valve +)

ESP8266 (5V USB) ─── Relay VCC (5V)
ESP8266 GPIO5 ────── Relay IN (Signal)
ESP8266 GND ───┬──── Relay GND
               └──── LED Cathode (via resistor)
```

**Safety Notes:**
1. ⚠️ **Isolate high-voltage circuits** - use opto-isolated relays
2. **Separate power supplies** for ESP8266 (5V) and valves (12V)
3. **Share common ground** between ESP and relay module
4. **Add flyback diode** across solenoid if relay doesn't have one
5. **Waterproof enclosures** (IP65+) for outdoor installations

---

### 4.3 Complete System Wiring Overview

```
                    ┌──────────────────┐
                    │   WiFi Router    │
                    │  192.168.1.1     │
                    └────────┬─────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
    ┌─────▼─────┐      ┌────▼─────┐      ┌────▼─────┐
    │  ESP32    │      │ ESP8266  │      │ ESP8266  │
    │  Sensor   │      │  Valve   │      │  Valve   │
    │  Hub      │      │  Zone 1  │      │  Zone 2  │
    │ .100      │      │ .111     │      │ .112     │
    └─────┬─────┘      └────┬─────┘      └────┬─────┘
          │                 │                  │
    ┌─────┴─────┐     ┌────┴────┐       ┌────┴────┐
    │ Sensor 1  │     │ Valve 1 │       │ Valve 2 │
    │ Sensor 2  │     │ (12V)   │       │ (12V)   │
    │ Sensor 3  │     └─────────┘       └─────────┘
    │ (BME280)  │
    └───────────┘           
                      ┌────▼─────┐
                      │ ESP8266  │       ┌──────────────┐
                      │  Valve   │       │ Docker Host  │
                      │  Zone 3  │       │  Server PC   │
                      │ .113     │       │  192.168.1.X │
                      └────┬─────┘       │              │
                           │             │ • Mosquitto  │
                     ┌────┴────┐        │ • Node-RED   │
                     │ Valve 3 │        │ • InfluxDB   │
                     │ (12V)   │        │ • Grafana    │
                     └─────────┘        │ • Zigbee2M   │
                                        └──────────────┘
```

---

## Part 5: Detailed Node-RED Flow Explanation

### Architecture of the Flow

The Node-RED flow has **5 main sections**:

1. **MQTT Input** - Receives sensor data
2. **Data Processing** - Parses and stores data
3. **Weather Integration** - Fetches forecast
4. **Decision Logic** - Determines watering needs
5. **MQTT Output** - Controls valves

Let me explain each node type in detail:

---

### Node Type 1: MQTT Input Nodes

**Purpose**: Subscribe to sensor topics and receive data

```javascript
// Example: mqtt_in_zone1_moisture
{
  "topic": "garden/sensors/sensor/garden_sensor_hub_zone_1_moisture/state",
  "qos": 1,  // Quality of Service level
  "datatype": "auto",  // Automatically parse JSON/string
  "broker": "mqtt_broker"  // Reference to MQTT config
}
```

**What happens:**
1. ESPHome publishes: `garden/sensors/sensor/.../state` → `"45.3"`
2. Node-RED receives message object:
   ```javascript
   msg = {
     topic: "garden/sensors/sensor/.../state",
     payload: "45.3",  // String from MQTT
     qos: 1,
     retain: false
   }
   ```

**Why QoS 1?**
- QoS 0: Fire and forget (may lose messages)
- **QoS 1**: At least once delivery ✅ (guaranteed)
- QoS 2: Exactly once (overkill for sensors)

---

### Node Type 2: Function Nodes (Data Processing)

**Example: `parse_zone1_moisture`**

```javascript
// This function runs every time a message arrives
const moisture = parseFloat(msg.payload);  // Convert "45.3" → 45.3

// Store in flow context (persists across messages)
flow.set('zone1_moisture', moisture);

// Prepare for InfluxDB (structured data)
msg.payload = {
    measurement: 'soil_moisture',  // InfluxDB "table"
    fields: { 
        value: moisture  // Numeric value
    },
    tags: { 
        zone: 'zone1',      // Indexed field (fast queries)
        location: 'vegetables'
    },
    timestamp: new Date()  // When this reading occurred
};

return msg;  // Pass to next node
```

**Context Storage Types:**
- `flow.set()` - Available to all nodes in this flow/tab
- `global.set()` - Available to all flows
- `context.set()` - Only this function node

**Why use context?**
```javascript
// Later, in decision logic, retrieve the value:
const moisture = flow.get('zone1_moisture');  // Returns 45.3
const rain = flow.get('rain_probability');     // Returns 30
// Now you can compare them without re-querying
```

---

### Node Type 3: InfluxDB Output Node

**Purpose**: Store time-series data for historical analysis

**Configuration:**
```javascript
{
  "influxdb": "influxdb_config",
  "name": "Write to InfluxDB",
  "measurement": "",  // Taken from msg.payload.measurement
  "precision": "ms",  // Timestamp precision
  "database": "garden"
}
```

**Data Structure Sent:**
```javascript
// From msg.payload
{
  measurement: "soil_moisture",
  tags: {
    zone: "zone1",
    location: "vegetables"
  },
  fields: {
    value: 45.3
  },
  timestamp: 1698765432000
}
```

**Stored in InfluxDB as:**
```sql
-- InfluxDB line protocol (internal format)
soil_moisture,zone=zone1,location=vegetables value=45.3 1698765432000

-- Query it later:
SELECT value FROM soil_moisture WHERE zone='zone1' AND time > now() - 24h
```

**Why InfluxDB for sensors?**
- Optimized for time-series data
- Automatic downsampling (e.g., keep hourly averages after 30 days)
- Fast queries over time ranges
- Small storage footprint

---

### Node Type 4: Dashboard UI Nodes

**Gauge Example: `ui_zone1_moisture`**

```javascript
{
  "type": "ui_gauge",
  "name": "Zone 1 Moisture",
  "group": "ui_group_zone1",  // Which dashboard panel
  "format": "{{value}}",  // Display template
  "min": 0,
  "max": 100,
  "colors": [
    "#ca3838",  // Red (0-30%)
    "#e6e600",  // Yellow (30-60%)
    "#00b500"   // Green (60-100%)
  ],
  "seg1": 30,  // First threshold
  "seg2": 60   // Second threshold
}
```

**Visual Output:**
```
┌─────────────────────────┐
│   Zone 1 - Vegetables   │
│                         │
│      Soil Moisture      │
│         ┌───┐           │
│        ╱  45 ╲          │  ← Gauge shows yellow
│       │   %   │         │    (30-60% range)
│        ╲     ╱          │
│         └───┘           │
└─────────────────────────┘
```

**Switch Example: `ui_zone1_manual`**

```javascript
{
  "type": "ui_switch",
  "label": "Manual Override",
  "onvalue": true,
  "offvalue": false,
  "passthru": true  // Send current state on deploy
}
```

**When user toggles:**
```javascript
// Switch ON → sends message:
msg = {
  payload: true,
  topic: "zone1_manual"
}
// Flows to → zone1_manual_handler → Stores in context
```

---

### Node Type 5: Decision Logic (Core Intelligence)

**Full Annotated Example: `zone1_logic`**

```javascript
// ============================================================================
// ZONE 1 WATERING DECISION LOGIC
// ============================================================================

// STEP 1: Retrieve current state from context storage
const moisture = flow.get('zone1_moisture') || 50;  // Default if not set
const rainProb = flow.get('rain_probability') || 0;
const manualOverride = flow.get('zone1_manual_override') || false;
const autoMode = flow.get('zone1_auto_mode') !== false;  // Default true
const lastWatered = flow.get('zone1_last_watered') || 0;  // Timestamp

// STEP 2: Define thresholds (configurable per zone)
const MOISTURE_THRESHOLD = 35;  // % - Water if below this
const RAIN_THRESHOLD = 40;      // % - Skip if rain above this
const MIN_INTERVAL = 4 * 60 * 60 * 1000;  // ms - Min time between waterings (4 hours)
const WATERING_DURATION = 15 * 60 * 1000;  // ms - How long to water (15 minutes)

// STEP 3: Initialize decision variables
let shouldWater = false;
let reason = '';

// STEP 4: Calculate time-based conditions
const now = Date.now();  // Current timestamp (ms since epoch)
const timeSinceWatering = now - lastWatered;
const currentlyWatering = flow.get('zone1_watering') || false;

// STEP 5: Decision tree (evaluated top to bottom)

// Check 1: Are we currently watering?
if (currentlyWatering) {
    // Check if duration exceeded
    if (timeSinceWatering > WATERING_DURATION) {
        shouldWater = false;
        reason = 'Watering duration complete';
        flow.set('zone1_watering', false);  // Reset flag
    } else {
        shouldWater = true;
        reason = 'Currently watering';
    }
}
// Check 2: Manual override active?
else if (manualOverride) {
    shouldWater = true;
    reason = 'Manual override active';
}
// Check 3: Auto mode disabled?
else if (!autoMode) {
    shouldWater = false;
    reason = 'Auto mode disabled';
}
// Check 4: Rain expected?
else if (rainProb > RAIN_THRESHOLD) {
    shouldWater = false;
    reason = `Rain expected (${rainProb}%)`;
}
// Check 5: Too soon since last watering?
else if (timeSinceWatering < MIN_INTERVAL) {
    shouldWater = false;
    const hoursLeft = Math.round((MIN_INTERVAL - timeSinceWatering) / (60 * 60 * 1000));
    reason = `Too soon (${hoursLeft}h left)`;
}
// Check 6: Soil moisture below threshold?
else if (moisture < MOISTURE_THRESHOLD) {
    shouldWater = true;
    reason = `Low moisture (${moisture}%)`;
    // Record watering start
    flow.set('zone1_last_watered', now);
    flow.set('zone1_watering', true);
}
// Default: Moisture sufficient
else {
    shouldWater = false;
    reason = `Sufficient moisture (${moisture}%)`;
}

// STEP 6: Store status for UI display
flow.set('zone1_status', reason);

// STEP 7: Prepare MQTT command message
msg.payload = shouldWater ? 'ON' : 'OFF';
msg.topic = 'garden/valves/zone1/switch/garden_valve_zone1_valve/command';

// STEP 8: Visual feedback in Node-RED editor
node.status({
    fill: shouldWater ? 'green' : 'grey',
    shape: 'dot',
    text: reason
});

// STEP 9: Log decision (appears in Node-RED debug panel)
node.log(`Zone 1 decision: ${shouldWater ? 'WATER' : 'SKIP'} - ${reason}`);

// STEP 10: Return message to next node (MQTT output)
return msg;
```

**Decision Tree Visualization:**

```
┌─────────────────────────────────┐
│   Zone 1 Moisture Reading       │
│         (45%)                    │
└──────────────┬──────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  Is valve currently watering?    │
│           NO                     │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  Manual override active?         │
│           NO                     │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  Auto mode enabled?              │
│           YES                    │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  Rain probability > 40%?         │
│      (30% < 40%)                 │
│           NO                     │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  Last watered < 4 hours ago?     │
│      (6 hours ago)               │
│           NO                     │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  Moisture < 35%?                 │
│      (45% > 35%)                 │
│           NO                     │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  DECISION: SKIP WATERING         │
│  Reason: "Sufficient moisture"   │
│  Command: Send "OFF" to valve    │
└──────────────────────────────────┘
```

---

### Node Type 6: Weather API Integration

**HTTP Request Node: `weather_api_call`**

```javascript
{
  "method": "GET",
  "url": "https://api.openweathermap.org/data/2.5/forecast?lat=51.5074&lon=-0.1278&appid=YOUR_API_KEY&units=metric",
  "ret": "obj"  // Return as JSON object
}
```

**API Response Structure:**
```json
{
  "list": [
    {
      "dt": 1698768000,
      "main": {
        "temp": 18.5,
        "humidity": 65
      },
      "pop": 0.3,  // Probability of precipitation (30%)
      "rain": {
        "3h": 1.2  // Rain volume in last 3h (mm)
      }
    },
    // ... 39 more forecasts (5 days, 3-hour intervals)
  ]
}
```

**Parser Function: `weather_parse`**

```javascript
// Extract rain probability for next 24 hours
const forecasts = msg.payload.list;
let maxRainProb = 0;
let totalRain = 0;

// Check next 8 forecasts (24 hours with 3-hour intervals)
for (let i = 0; i < Math.min(8, forecasts.length); i++) {
    const forecast = forecasts[i];
    
    // Rain probability (0.0 to 1.0 → 0% to 100%)
    if (forecast.pop) {
        maxRainProb = Math.max(maxRainProb, forecast.pop * 100);
    }
    
    // Rain amount (mm)
    if (forecast.rain && forecast.rain['3h']) {
        totalRain += forecast.rain['3h'];
    }
}

// Store in flow context for decision logic
flow.set('rain_probability', Math.round(maxRainProb));
flow.set('rain_amount_24h', totalRain.toFixed(1));
flow.set('current_temp', forecasts[0].main.temp);

// Prepare for InfluxDB storage
msg.payload = {
    measurement: 'weather',
    fields: {
        rain_probability: Math.round(maxRainProb),
        rain_amount: totalRain,
        temperature: forecasts[0].main.temp,
        humidity: forecasts[0].main.humidity
    },
    tags: {
        source: 'openweathermap'
    },
    timestamp: new Date()
};

return msg;
```

**Example Flow:**
```
Every 2 hours → HTTP Request → Parse Weather
                                    ↓
                    ┌───────────────┴────────────────┐
                    │                                 │
                    ▼                                 ▼
            Store in Context                  Store in InfluxDB
         (for immediate use)                  (for historical)
```

---

### Node Type 7: Emergency Stop

**Purpose**: Safety override to shut down all zones

```javascript
// Button pressed → emergency_handler function

// STEP 1: Disable all automation
flow.set('zone1_auto_mode', false);
flow.set('zone2_auto_mode', false);
flow.set('zone3_auto_mode', false);

// STEP 2: Clear all overrides
flow.set('zone1_manual_override', false);
flow.set('zone2_manual_override', false);
flow.set('zone3_manual_override', false);

// STEP 3: Reset watering states
flow.set('zone1_watering', false);
flow.set('zone2_watering', false);
flow.set('zone3_watering', false);

// STEP 4: Send OFF commands to all valves
return [
    { 
        topic: 'garden/valves/zone1/switch/garden_valve_zone1_valve/command', 
        payload: 'OFF' 
    },
    { 
        topic: 'garden/valves/zone2/switch/garden_valve_zone2_valve/command', 
        payload: 'OFF' 
    },
    { 
        topic: 'garden/valves/zone3/switch/garden_valve_zone3_valve/command', 
        payload: 'OFF' 
    }
];
// Returns array of 3 messages (sent to different outputs)
```

**Multi-output Function Node:**
```
[emergency_handler] → Output 1 → [mqtt_out_zone1_valve]
                   → Output 2 → [mqtt_out_zone2_valve]
                   → Output 3 → [mqtt_out_zone3_valve]
```

---

### Complete Message Flow Example

**Scenario**: Zone 1 moisture drops to 28%

```
1. ESP32 publishes MQTT:
   Topic: garden/sensors/sensor/.../zone_1_moisture/state
   Payload: "28.0"

2. Node-RED [mqtt_in_zone1_moisture] receives:
   msg.payload = "28.0"

3. [parse_zone1_moisture] function processes:
   - Converts to float: 28.0
   - Stores: flow.set('zone1_moisture', 28.0)
   - Creates InfluxDB structure
   
4. [influx_write] stores to database:
   - soil_moisture,zone=zone1 value=28.0

5. [ui_zone1_moisture] gauge updates:
   - Shows red (< 30%)
   
6. [zone1_logic] evaluates:
   - moisture (28) < THRESHOLD (35) → TRUE
   - rain_prob (15) < RAIN_THRESHOLD (40) → TRUE
   - timeSinceWatering (8h) > MIN_INTERVAL (4h) → TRUE
   - Decision: shouldWater = TRUE
   
7. [mqtt_out_zone1_valve] publishes:
   Topic: garden/valves/zone1/switch/.../command
   Payload: "ON"
   
8. ESP8266 receives command:
   - Activates relay
   - Valve opens
   - LED turns on
   - Publishes state back

9. After 15 minutes:
   - [zone1_logic] checks duration
   - Sends "OFF" command
   - Valve closes
```

---

## Part 6: Enhanced Grafana Dashboard

I'll provide additional panels beyond the basic dashboard:

### Panel 1: Real-Time Status Table

```json
{
  "type": "table",
  "title": "Current System Status",
  "targets": [
    {
      "query": "SELECT last(\"value\") FROM \"soil_moisture\" WHERE $timeFilter GROUP BY \"zone\"",
      "rawQuery": true
    }
  ],
  "transformations": [
    {
      "id": "organize",
      "options": {
        "excludeByName": {},
        "indexByName": {},
        "renameByName": {
          "zone": "Zone",
          "last": "Moisture %"
        }
      }
    }
  ],
  "overrides": [
    {
      "matcher": { "id": "byName", "options": "Moisture %" },
      "properties": [
        {
          "id": "custom.cellOptions",
          "value": {
            "type": "color-background",
            "mode": "gradient"
          }
        },
        {
          "id": "thresholds",
          "value": {
            "steps": [
              { "color": "red", "value": 0 },
              { "color": "yellow", "value": 30 },
              { "color": "green", "value": 60 }
            ]
          }
        }
      ]
    }
  ]
}
```

**Visual Output:**
```
┌───────────────────────────────────────┐
│   Current System Status               │
├──────────┬─────────────┬──────────────┤
│ Zone     │ Moisture %  │ Last Update  │
├──────────┼─────────────┼──────────────┤
│ Zone 1   │ 45.2 ████   │ 2 min ago    │ ← Yellow background
│ Zone 2   │ 28.1 ██     │ 3 min ago    │ ← Red background
│ Zone 3   │ 67.8 ██████ │ 1 min ago    │ ← Green background
└──────────┴─────────────┴──────────────┘
```

---

### Panel 2: Watering Duration Histogram

```json
{
  "type": "barchart",
  "title": "Watering Duration by Zone (Last 7 Days)",
  "targets": [
    {
      "query": "SELECT sum(\"duration\") FROM \"valve_events\" WHERE $timeFilter AND time > now() - 7d GROUP BY time(1d), \"zone\" fill(0)",
      "rawQuery": true
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "dthms",
      "custom": {
        "stacking": {
          "mode": "normal"
        }
      }
    }
  }
}
```

**Visual Output:**
```
┌────────────────────────────────────────────────┐
│ Watering Duration (minutes)                    │
│                                                 │
│ 120 ┤                                          │
│     │     ███████                              │
│ 100 ┤     ███████                              │
│     │     ███████  ███████                     │
│  80 ┤     ███████  ███████          ███████   │
│     │ ███████████  ███████  ███████ ███████   │
│  60 ┤ ███████████  ███████  ███████ ███████   │
│     │ ███████████  ███████  ███████ ███████   │
│  40 ┤ ███████████  ███████  ███████ ███████   │
│     └─────┬────────┬────────┬────────┬────────│
│          Mon      Tue      Wed      Thu       │
│                                                 │
│ █ Zone 1  █ Zone 2  █ Zone 3                  │
└────────────────────────────────────────────────┘
```

---

### Panel 3: Correlation Heatmap (Moisture vs Temperature)

```json
{
  "type": "heatmap",
  "title": "Soil Moisture vs Temperature",
  "targets": [
    {
      "query": "SELECT mean(\"value\") as moisture, mean(\"temperature\") as temp FROM \"soil_moisture\", \"weather\" WHERE $timeFilter GROUP BY time(1h) fill(linear)",
      "rawQuery": true
    }
  ],
  "options": {
    "calculate": false,
    "cellGap": 2,
    "cellRadius": 0,
    "color": {
      "exponent": 0.5,
      "scheme": "Spectral",
      "steps": 128
    },
    "yAxis": {
      "axisPlacement": "left",
      "reverse": false,
      "unit": "celsius"
    }
  }
}
```

---

### Panel 4: Predictive Water Consumption

```json
{
  "type": "timeseries",
  "title": "Water Usage Forecast",
  "targets": [
    {
      "query": "SELECT cumulative_sum(mean(\"duration\")) * 10 as \"liters\" FROM \"valve_events\" WHERE $timeFilter GROUP BY time(1d), \"zone\"",
      "rawQuery": true,
      "alias": "$tag_zone"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "litre",
      "color": {
        "mode": "palette-classic"
      },
      "custom": {
        "fillOpacity": 30,
        "lineInterpolation": "smooth"
      }
    }
  }
}
```

**Visual Output:**
```
┌────────────────────────────────────────────────┐
│ Cumulative Water Usage (liters)                │
│                                                 │
│ 500 ┤                                    ╱╱╱   │
│     │                               ╱╱╱╱╱      │
│ 400 ┤                          ╱╱╱╱╱           │
│     │                     ╱╱╱╱╱                │
│ 300 ┤                ╱╱╱╱╱                     │
│     │           ╱╱╱╱╱                          │
│ 200 ┤      ╱╱╱╱╱                               │
│     │ ╱╱╱╱╱                                    │
│ 100 ╱╱                                         │
│     └────┬────┬────┬────┬────┬────┬───────────│
│        Mon  Tue  Wed  Thu  Fri  Sat  Sun      │
│                                                 │
│ ─── Zone 1   ─── Zone 2   ─── Zone 3         │
└────────────────────────────────────────────────┘
```

---

### Panel 5: Alert Status Panel

```json
{
  "type": "stat",
  "title": "System Alerts",
  "targets": [
    {
      "query": "SELECT count(\"value\") FROM \"soil_moisture\" WHERE \"value\" < 20 AND time > now() - 1h",
      "rawQuery": true,
      "alias": "Critical Low Moisture"
    },
    {
      "query": "SELECT count(\"state\") FROM \"valve_events\" WHERE \"duration\" > 1800 AND time > now() - 24h",
      "rawQuery": true,
      "alias": "Long Watering Sessions"
    }
  ],
  "options": {
    "reduceOptions": {
      "calcs": ["lastNotNull"]
    },
    "textMode": "value_and_name",
    "colorMode": "background"
  },
  "fieldConfig": {
    "defaults": {
      "thresholds": {
        "steps": [
          { "color": "green", "value": 0 },
          { "color": "yellow", "value": 1 },
          { "color": "red", "value": 3 }
        ]
      }
    }
  }
}
```

**Visual Output:**
```
┌──────────────────────────┐  ┌──────────────────────────┐
│ Critical Low Moisture    │  │ Long Watering Sessions   │
│                          │  │                          │
│          2               │  │          0               │
│    ████████████          │  │    ████████████          │
│    ████████████          │  │    ████████████          │
└──────────────────────────┘  └──────────────────────────┘
   (Yellow - Warning)            (Green - OK)
```

---

### Panel 6: Efficiency Metrics

```json
{
  "type": "gauge",
  "title": "System Efficiency",
  "targets": [
    {
      "query": "SELECT (sum(\"moisture_increase\") / sum(\"water_used\")) * 100 as \"efficiency\" FROM \"watering_events\" WHERE $timeFilter",
      "rawQuery": true
    }
  ],
  "options": {
    "showThresholdLabels": true,
    "showThresholdMarkers": true
  },
  "fieldConfig": {
    "defaults": {
      "unit": "percent",
      "min": 0,
      "max": 100,
      "thresholds": {
        "steps": [
          { "color": "red", "value": 0 },
          { "color": "yellow", "value": 50 },
          { "color": "green", "value": 75 }
        ]
      }
    }
  }
}
```

---

### Complete Dashboard Layout

```
┌─────────────────────────────────────────────────────────────────┐
│                    Garden Irrigation System                     │
├──────────────┬──────────────┬──────────────┬───────────────────┤
│   Zone 1     │   Zone 2     │   Zone 3     │   Weather         │
│   Moisture   │   Moisture   │   Moisture   │   ┌─────────────┐ │
│   ┌─────┐    │   ┌─────┐    │   ┌─────┐    │   │Rain: 30%    │ │
│   │ 45% │    │   │ 28% │    │   │ 67% │    │   │Temp: 18°C   │ │
│   └─────┘    │   └─────┘    │   └─────┘    │   └─────────────┘ │
├──────────────┴──────────────┴──────────────┴───────────────────┤
│   Soil Moisture Trends (24 Hours)                              │
│   ┌───────────────────────────────────────────────────────────┐ │
│   │100%│   ╱╲                    ╱─────╲              ╱╲    │ │
│   │    │  ╱  ╲                  ╱       ╲            ╱  ╲   │ │
│   │ 50%│ ╱    ╲────────────────╱         ╲──────────╱    ╲ │ │
│   │    │╱                                                   │ │
│   │  0%└─────┬─────┬─────┬─────┬─────┬─────┬─────┬────────│ │
│   │         0h    4h    8h   12h   16h   20h   24h         │ │
│   └───────────────────────────────────────────────────────────┘ │
├────────────────────────────┬────────────────────────────────────┤
│  Watering Events (7 Days)  │   Current System Status           │
│  ┌─────────────────────┐   │   ┌──────────────────────────────┐│
│  │ █████████ Zone 1    │   │   │ Zone 1 │ 45% │ Valve: OFF  ││
│  │ ████████  Zone 2    │   │   │ Zone 2 │ 28% │ Valve: ON   ││
│  │ █████     Zone 3    │   │   │ Zone 3 │ 67% │ Valve: OFF  ││
│  └─────────────────────┘   │   └──────────────────────────────┘│
├────────────────────────────┴────────────────────────────────────┤
│  System Alerts             │  Water Usage (Cumulative)         │
│  ┌─────────────────────┐   │  ┌──────────────────────────────┐│
│  │ ⚠ Low Moisture: 1   │   │  │ Total: 450 L                 ││
│  │ ✓ Valves: OK        │   │  │ Zone 1: 180 L (40%)          ││
│  │ ✓ WiFi: Strong      │   │  │ Zone 2: 150 L (33%)          ││
│  └─────────────────────┘   │  │ Zone 3: 120 L (27%)          ││
│                            │  └──────────────────────────────┘│
└────────────────────────────┴────────────────────────────────────┘
```

---

This completes the comprehensive implementation guide! You now have:

1. ✅ **Device recommendations** with detailed reasoning
2. ✅ **Complete ESPHome configs** for 1 sensor hub + 3 valve controllers
3. ✅ **Detailed wiring diagrams** with component specifications
4. ✅ **In-depth Node-RED explanations** with annotated code
5. ✅ **Enhanced Grafana dashboards** with 6+ panel types

Would you like me to:
- Add **OTA update procedures** for ESP devices?
- Create **backup/restore scripts** for configurations?
- Provide **troubleshooting flowcharts** for common issues?
- Design **3D-printable enclosures** for the hardware?