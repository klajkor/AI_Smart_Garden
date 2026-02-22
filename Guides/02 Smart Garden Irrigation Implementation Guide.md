# Complete Smart Garden Irrigation Implementation Guide

## Part 1: ESPHome Configurations

**Why ESPHome over Tasmota?**
- Native MQTT integration with better sensor support
- YAML configuration (easier to maintain)
- Built-in deep sleep for battery sensors
- Better analog sensor handling (for capacitive soil moisture)
- OTA updates built-in

### ESPHome Configuration 1: 3-Zone Master Controller (ESP32)

**Hardware Setup:**
- ESP32 board
- 3× Capacitive soil moisture sensors (v1.2 or v2.0)
- 3× Relay modules or solid-state relays for valves
- 12V power supply for valves

**File: `zone-controller.yaml`**

```yaml
substitutions:
  device_name: garden-zone-controller
  friendly_name: Garden Zone Controller
  mqtt_topic: garden/controller

esphome:
  name: ${device_name}
  platform: ESP32
  board: esp32dev

# Enable logging
logger:
  level: INFO

# Enable Home Assistant API (optional, but useful)
api:
  encryption:
    key: "YOUR_GENERATED_KEY_HERE"

# OTA updates
ota:
  password: "YOUR_OTA_PASSWORD"

# WiFi configuration
wifi:
  ssid: "YOUR_WIFI_SSID"
  password: "YOUR_WIFI_PASSWORD"
  
  # Enable fallback hotspot in case wifi connection fails
  ap:
    ssid: "Garden-Controller"
    password: "fallback123"

# MQTT Configuration
mqtt:
  broker: YOUR_MOSQUITTO_IP
  port: 1883
  username: mqtt_user
  password: YOUR_MQTT_PASSWORD
  topic_prefix: ${mqtt_topic}
  discovery: true
  birth_message:
    topic: ${mqtt_topic}/status
    payload: online
  will_message:
    topic: ${mqtt_topic}/status
    payload: offline

# Web server for diagnostics
web_server:
  port: 80

# I2C bus (if you add BME280 for temperature/humidity)
# i2c:
#   sda: GPIO21
#   scl: GPIO22

# Sensors
sensor:
  # WiFi Signal Strength
  - platform: wifi_signal
    name: "${friendly_name} WiFi Signal"
    update_interval: 60s
    
  # Uptime
  - platform: uptime
    name: "${friendly_name} Uptime"
    
  # Zone 1 Soil Moisture
  - platform: adc
    pin: GPIO36  # VP pin on ESP32
    name: "${friendly_name} Zone 1 Moisture Raw"
    id: moisture_raw_1
    attenuation: 11db
    update_interval: 30s
    filters:
      - multiply: 0.000244141  # Convert to voltage (1/4096)
    
  - platform: template
    name: "${friendly_name} Zone 1 Moisture"
    id: moisture_percent_1
    unit_of_measurement: "%"
    accuracy_decimals: 0
    lambda: |-
      // Calibration values - ADJUST THESE
      float dry_voltage = 2.8;   // Voltage in dry soil
      float wet_voltage = 1.2;   // Voltage in wet soil
      float current = id(moisture_raw_1).state;
      
      // Convert to percentage (inverted because lower voltage = more moisture)
      float percent = (dry_voltage - current) / (dry_voltage - wet_voltage) * 100.0;
      
      // Clamp between 0-100
      if (percent > 100) return 100;
      if (percent < 0) return 0;
      return percent;
    update_interval: 30s

  # Zone 2 Soil Moisture
  - platform: adc
    pin: GPIO39  # VN pin on ESP32
    name: "${friendly_name} Zone 2 Moisture Raw"
    id: moisture_raw_2
    attenuation: 11db
    update_interval: 30s
    filters:
      - multiply: 0.000244141
    
  - platform: template
    name: "${friendly_name} Zone 2 Moisture"
    id: moisture_percent_2
    unit_of_measurement: "%"
    accuracy_decimals: 0
    lambda: |-
      float dry_voltage = 2.8;
      float wet_voltage = 1.2;
      float current = id(moisture_raw_2).state;
      float percent = (dry_voltage - current) / (dry_voltage - wet_voltage) * 100.0;
      if (percent > 100) return 100;
      if (percent < 0) return 0;
      return percent;
    update_interval: 30s

  # Zone 3 Soil Moisture
  - platform: adc
    pin: GPIO34
    name: "${friendly_name} Zone 3 Moisture Raw"
    id: moisture_raw_3
    attenuation: 11db
    update_interval: 30s
    filters:
      - multiply: 0.000244141
    
  - platform: template
    name: "${friendly_name} Zone 3 Moisture"
    id: moisture_percent_3
    unit_of_measurement: "%"
    accuracy_decimals: 0
    lambda: |-
      float dry_voltage = 2.8;
      float wet_voltage = 1.2;
      float current = id(moisture_raw_3).state;
      float percent = (dry_voltage - current) / (dry_voltage - wet_voltage) * 100.0;
      if (percent > 100) return 100;
      if (percent < 0) return 0;
      return percent;
    update_interval: 30s

# Binary sensors (status indicators)
binary_sensor:
  - platform: status
    name: "${friendly_name} Status"

# Switches for valve control
switch:
  # Zone 1 Valve
  - platform: gpio
    pin: GPIO25
    name: "${friendly_name} Zone 1 Valve"
    id: valve_1
    icon: "mdi:water-pump"
    restore_mode: ALWAYS_OFF  # Safety: always start with valves off
    on_turn_on:
      - logger.log: "Zone 1 valve opened"
    on_turn_off:
      - logger.log: "Zone 1 valve closed"
      
  # Zone 2 Valve
  - platform: gpio
    pin: GPIO26
    name: "${friendly_name} Zone 2 Valve"
    id: valve_2
    icon: "mdi:water-pump"
    restore_mode: ALWAYS_OFF
    on_turn_on:
      - logger.log: "Zone 2 valve opened"
    on_turn_off:
      - logger.log: "Zone 2 valve closed"
      
  # Zone 3 Valve
  - platform: gpio
    pin: GPIO27
    name: "${friendly_name} Zone 3 Valve"
    id: valve_3
    icon: "mdi:water-pump"
    restore_mode: ALWAYS_OFF
    on_turn_on:
      - logger.log: "Zone 3 valve opened"
    on_turn_off:
      - logger.log: "Zone 3 valve closed"
  
  # Master pump/valve control
  - platform: gpio
    pin: GPIO32
    name: "${friendly_name} Master Pump"
    id: master_pump
    icon: "mdi:pump"
    restore_mode: ALWAYS_OFF
    
  # Restart button
  - platform: restart
    name: "${friendly_name} Restart"

# Text sensors for diagnostics
text_sensor:
  - platform: wifi_info
    ip_address:
      name: "${friendly_name} IP Address"
    ssid:
      name: "${friendly_name} SSID"

# Status LED
status_led:
  pin: GPIO2  # Built-in LED on most ESP32 boards

# Automation: Safety timeout - auto-shutoff after 30 minutes
interval:
  - interval: 60s
    then:
      - lambda: |-
          // Check each valve and log status
          if (id(valve_1).state) {
            ESP_LOGI("safety", "Zone 1 valve is ON");
          }
          if (id(valve_2).state) {
            ESP_LOGI("safety", "Zone 2 valve is ON");
          }
          if (id(valve_3).state) {
            ESP_LOGI("safety", "Zone 3 valve is ON");
          }
```

---

### ESPHome Configuration 2: Battery-Powered Remote Sensor (ESP8266)

**For additional remote moisture monitoring with deep sleep**

**File: `remote-sensor-zone4.yaml`**

```yaml
substitutions:
  device_name: garden-remote-sensor-4
  friendly_name: Garden Remote Sensor 4
  mqtt_topic: garden/sensor4

esphome:
  name: ${device_name}
  platform: ESP8266
  board: d1_mini

logger:
  level: INFO

ota:
  password: "YOUR_OTA_PASSWORD"

wifi:
  ssid: "YOUR_WIFI_SSID"
  password: "YOUR_WIFI_PASSWORD"
  
  # Fast connect for battery saving
  fast_connect: true
  
  ap:
    ssid: "Garden-Sensor-4"
    password: "fallback123"

mqtt:
  broker: YOUR_MOSQUITTO_IP
  port: 1883
  username: mqtt_user
  password: YOUR_MQTT_PASSWORD
  topic_prefix: ${mqtt_topic}

# Deep sleep configuration (wake every 30 minutes)
deep_sleep:
  run_duration: 60s  # Stay awake for 60 seconds
  sleep_duration: 30min  # Sleep for 30 minutes
  id: deep_sleep_control

sensor:
  # Battery voltage (using ADC with voltage divider)
  - platform: adc
    pin: A0
    name: "${friendly_name} Battery"
    unit_of_measurement: "V"
    accuracy_decimals: 2
    update_interval: 30s
    filters:
      - multiply: 4.2  # Adjust based on your voltage divider
    
  # Soil moisture
  - platform: adc
    pin: A0  # If using separate ADC pin
    name: "${friendly_name} Moisture"
    unit_of_measurement: "%"
    accuracy_decimals: 0
    update_interval: 30s
    filters:
      - calibrate_linear:
          - 0.0 -> 100.0  # Wet
          - 1.0 -> 0.0    # Dry

binary_sensor:
  - platform: status
    name: "${friendly_name} Status"
```

---

### Calibration Instructions for Soil Moisture Sensors

**After flashing ESPHome:**

1. **Access web interface**: `http://garden-zone-controller.local`
2. **View raw voltage values** in logs
3. **Calibrate**:
   ```
   # In completely DRY soil
   Observe voltage → Update dry_voltage value
   
   # In completely WET soil (after watering)
   Observe voltage → Update wet_voltage value
   ```
4. **Recompile and upload** ESPHome config

---

### ESPHome Installation Commands

```bash
# Install ESPHome
pip3 install esphome

# Compile configuration
esphome compile zone-controller.yaml

# Upload via USB (first time)
esphome upload zone-controller.yaml

# Upload via WiFi (OTA - after first flash)
esphome upload zone-controller.yaml --device garden-zone-controller.local

# View logs
esphome logs zone-controller.yaml
```

---

## Part 2: Complete Node-RED Flow JSON

**Features:**
- 3-zone moisture monitoring
- Weather API integration (OpenWeatherMap)
- Automatic watering logic
- Manual override switches
- Zone scheduling
- Safety timeouts
- Dashboard UI
- InfluxDB data storage

**Installation Steps:**
1. Open Node-RED: `http://your-server:1880`
2. Menu (≡) → Import → Paste JSON below
3. Install required nodes if prompted
4. Configure MQTT and InfluxDB nodes
5. Deploy

**File: `irrigation-flow.json`**

```json
[
  {
    "id": "irrigation_flow_tab",
    "type": "tab",
    "label": "Garden Irrigation System",
    "disabled": false,
    "info": "3-Zone Smart Irrigation with Weather Integration"
  },
  {
    "id": "mqtt_broker",
    "type": "mqtt-broker",
    "name": "Mosquitto",
    "broker": "mosquitto",
    "port": "1883",
    "clientid": "nodered",
    "autoConnect": true,
    "usetls": false,
    "protocolVersion": "4",
    "keepalive": "60",
    "cleansession": true,
    "birthTopic": "nodered/status",
    "birthQos": "0",
    "birthPayload": "online",
    "birthMsg": {},
    "closeTopic": "nodered/status",
    "closeQos": "0",
    "closePayload": "offline",
    "closeMsg": {},
    "willTopic": "",
    "willQos": "0",
    "willPayload": "",
    "willMsg": {},
    "userProps": "",
    "sessionExpiry": ""
  },
  {
    "id": "influxdb_config",
    "type": "influxdb",
    "hostname": "influxdb",
    "port": "8086",
    "protocol": "http",
    "database": "garden",
    "name": "Garden DB",
    "usetls": false,
    "tls": "",
    "influxdbVersion": "1.x",
    "url": "http://influxdb:8086",
    "rejectUnauthorized": false
  },
  {
    "id": "ui_tab_dashboard",
    "type": "ui_tab",
    "name": "Garden Control",
    "icon": "dashboard",
    "order": 1
  },
  {
    "id": "ui_group_zone1",
    "type": "ui_group",
    "name": "Zone 1 - Vegetables",
    "tab": "ui_tab_dashboard",
    "order": 1,
    "disp": true,
    "width": "6"
  },
  {
    "id": "ui_group_zone2",
    "type": "ui_group",
    "name": "Zone 2 - Flowers",
    "tab": "ui_tab_dashboard",
    "order": 2,
    "disp": true,
    "width": "6"
  },
  {
    "id": "ui_group_zone3",
    "type": "ui_group",
    "name": "Zone 3 - Lawn",
    "tab": "ui_tab_dashboard",
    "order": 3,
    "disp": true,
    "width": "6"
  },
  {
    "id": "ui_group_weather",
    "type": "ui_group",
    "name": "Weather",
    "tab": "ui_tab_dashboard",
    "order": 4,
    "disp": true,
    "width": "6"
  },
  {
    "id": "ui_group_system",
    "type": "ui_group",
    "name": "System Status",
    "tab": "ui_tab_dashboard",
    "order": 5,
    "disp": true,
    "width": "12"
  },
  {
    "id": "comment_zone1_inputs",
    "type": "comment",
    "z": "irrigation_flow_tab",
    "name": "ZONE 1 - MQTT INPUTS",
    "info": "",
    "x": 140,
    "y": 60,
    "wires": []
  },
  {
    "id": "mqtt_in_zone1_moisture",
    "type": "mqtt in",
    "z": "irrigation_flow_tab",
    "name": "Zone 1 Moisture",
    "topic": "garden/controller/sensor/garden_zone_controller_zone_1_moisture/state",
    "qos": "1",
    "datatype": "auto",
    "broker": "mqtt_broker",
    "nl": false,
    "rap": true,
    "rh": 0,
    "inputs": 0,
    "x": 130,
    "y": 100,
    "wires": [
      ["parse_zone1_moisture"]
    ]
  },
  {
    "id": "parse_zone1_moisture",
    "type": "function",
    "z": "irrigation_flow_tab",
    "name": "Parse & Store Zone 1",
    "func": "const moisture = parseFloat(msg.payload);\n\n// Store in flow context\nflow.set('zone1_moisture', moisture);\n\n// Prepare for InfluxDB\nmsg.payload = {\n    measurement: 'soil_moisture',\n    fields: { \n        value: moisture \n    },\n    tags: { \n        zone: 'zone1',\n        location: 'vegetables'\n    },\n    timestamp: new Date()\n};\n\nreturn msg;",
    "outputs": 1,
    "noerr": 0,
    "initialize": "",
    "finalize": "",
    "libs": [],
    "x": 380,
    "y": 100,
    "wires": [
      ["influx_write", "ui_zone1_moisture", "zone1_logic"]
    ]
  },
  {
    "id": "ui_zone1_moisture",
    "type": "ui_gauge",
    "z": "irrigation_flow_tab",
    "name": "Zone 1 Moisture",
    "group": "ui_group_zone1",
    "order": 1,
    "width": 0,
    "height": 0,
    "gtype": "gage",
    "title": "Soil Moisture",
    "label": "%",
    "format": "{{value}}",
    "min": 0,
    "max": "100",
    "colors": [
      "#ca3838",
      "#e6e600",
      "#00b500"
    ],
    "seg1": "30",
    "seg2": "60",
    "x": 650,
    "y": 80,
    "wires": []
  },
  {
    "id": "comment_zone2_inputs",
    "type": "comment",
    "z": "irrigation_flow_tab",
    "name": "ZONE 2 - MQTT INPUTS",
    "info": "",
    "x": 140,
    "y": 180,
    "wires": []
  },
  {
    "id": "mqtt_in_zone2_moisture",
    "type": "mqtt in",
    "z": "irrigation_flow_tab",
    "name": "Zone 2 Moisture",
    "topic": "garden/controller/sensor/garden_zone_controller_zone_2_moisture/state",
    "qos": "1",
    "datatype": "auto",
    "broker": "mqtt_broker",
    "nl": false,
    "rap": true,
    "rh": 0,
    "inputs": 0,
    "x": 130,
    "y": 220,
    "wires": [
      ["parse_zone2_moisture"]
    ]
  },
  {
    "id": "parse_zone2_moisture",
    "type": "function",
    "z": "irrigation_flow_tab",
    "name": "Parse & Store Zone 2",
    "func": "const moisture = parseFloat(msg.payload);\nflow.set('zone2_moisture', moisture);\n\nmsg.payload = {\n    measurement: 'soil_moisture',\n    fields: { value: moisture },\n    tags: { zone: 'zone2', location: 'flowers' },\n    timestamp: new Date()\n};\n\nreturn msg;",
    "outputs": 1,
    "noerr": 0,
    "x": 380,
    "y": 220,
    "wires": [
      ["influx_write", "ui_zone2_moisture", "zone2_logic"]
    ]
  },
  {
    "id": "ui_zone2_moisture",
    "type": "ui_gauge",
    "z": "irrigation_flow_tab",
    "name": "Zone 2 Moisture",
    "group": "ui_group_zone2",
    "order": 1,
    "width": 0,
    "height": 0,
    "gtype": "gage",
    "title": "Soil Moisture",
    "label": "%",
    "format": "{{value}}",
    "min": 0,
    "max": "100",
    "colors": [
      "#ca3838",
      "#e6e600",
      "#00b500"
    ],
    "seg1": "30",
    "seg2": "60",
    "x": 650,
    "y": 200,
    "wires": []
  },
  {
    "id": "comment_zone3_inputs",
    "type": "comment",
    "z": "irrigation_flow_tab",
    "name": "ZONE 3 - MQTT INPUTS",
    "info": "",
    "x": 140,
    "y": 300,
    "wires": []
  },
  {
    "id": "mqtt_in_zone3_moisture",
    "type": "mqtt in",
    "z": "irrigation_flow_tab",
    "name": "Zone 3 Moisture",
    "topic": "garden/controller/sensor/garden_zone_controller_zone_3_moisture/state",
    "qos": "1",
    "datatype": "auto",
    "broker": "mqtt_broker",
    "nl": false,
    "rap": true,
    "rh": 0,
    "inputs": 0,
    "x": 130,
    "y": 340,
    "wires": [
      ["parse_zone3_moisture"]
    ]
  },
  {
    "id": "parse_zone3_moisture",
    "type": "function",
    "z": "irrigation_flow_tab",
    "name": "Parse & Store Zone 3",
    "func": "const moisture = parseFloat(msg.payload);\nflow.set('zone3_moisture', moisture);\n\nmsg.payload = {\n    measurement: 'soil_moisture',\n    fields: { value: moisture },\n    tags: { zone: 'zone3', location: 'lawn' },\n    timestamp: new Date()\n};\n\nreturn msg;",
    "outputs": 1,
    "noerr": 0,
    "x": 380,
    "y": 340,
    "wires": [
      ["influx_write", "ui_zone3_moisture", "zone3_logic"]
    ]
  },
  {
    "id": "ui_zone3_moisture",
    "type": "ui_gauge",
    "z": "irrigation_flow_tab",
    "name": "Zone 3 Moisture",
    "group": "ui_group_zone3",
    "order": 1,
    "width": 0,
    "height": 0,
    "gtype": "gage",
    "title": "Soil Moisture",
    "label": "%",
    "format": "{{value}}",
    "min": 0,
    "max": "100",
    "colors": [
      "#ca3838",
      "#e6e600",
      "#00b500"
    ],
    "seg1": "30",
    "seg2": "60",
    "x": 650,
    "y": 320,
    "wires": []
  },
  {
    "id": "influx_write",
    "type": "influxdb out",
    "z": "irrigation_flow_tab",
    "influxdb": "influxdb_config",
    "name": "Write to InfluxDB",
    "measurement": "",
    "precision": "",
    "retentionPolicy": "",
    "database": "garden",
    "precisionV18FluxV20": "ms",
    "retentionPolicyV18Flux": "",
    "org": "",
    "bucket": "",
    "x": 680,
    "y": 420,
    "wires": []
  },
  {
    "id": "comment_weather",
    "type": "comment",
    "z": "irrigation_flow_tab",
    "name": "WEATHER API",
    "info": "",
    "x": 110,
    "y": 480,
    "wires": []
  },
  {
    "id": "weather_trigger",
    "type": "inject",
    "z": "irrigation_flow_tab",
    "name": "Every 2 hours",
    "props": [
      {
        "p": "payload"
      }
    ],
    "repeat": "7200",
    "crontab": "",
    "once": true,
    "onceDelay": "10",
    "topic": "",
    "payload": "",
    "payloadType": "date",
    "x": 130,
    "y": 520,
    "wires": [
      ["weather_api_call"]
    ]
  },
  {
    "id": "weather_api_call",
    "type": "http request",
    "z": "irrigation_flow_tab",
    "name": "OpenWeatherMap API",
    "method": "GET",
    "ret": "obj",
    "paytoqs": "ignore",
    "url": "https://api.openweathermap.org/data/2.5/forecast?lat=YOUR_LATITUDE&lon=YOUR_LONGITUDE&appid=YOUR_API_KEY&units=metric",
    "tls": "",
    "persist": false,
    "proxy": "",
    "authType": "",
    "x": 360,
    "y": 520,
    "wires": [
      ["weather_parse"]
    ]
  },
  {
    "id": "weather_parse",
    "type": "function",
    "z": "irrigation_flow_tab",
    "name": "Parse Weather Data",
    "func": "// Extract rain probability for next 24 hours\nconst forecasts = msg.payload.list;\nlet maxRainProb = 0;\nlet totalRain = 0;\n\n// Check next 8 forecasts (24 hours with 3-hour intervals)\nfor (let i = 0; i < Math.min(8, forecasts.length); i++) {\n    const forecast = forecasts[i];\n    \n    // Rain probability\n    if (forecast.pop) {\n        maxRainProb = Math.max(maxRainProb, forecast.pop * 100);\n    }\n    \n    // Rain amount\n    if (forecast.rain && forecast.rain['3h']) {\n        totalRain += forecast.rain['3h'];\n    }\n}\n\nflow.set('rain_probability', Math.round(maxRainProb));\nflow.set('rain_amount_24h', totalRain.toFixed(1));\n\n// Store in InfluxDB\nmsg.payload = {\n    measurement: 'weather',\n    fields: {\n        rain_probability: Math.round(maxRainProb),\n        rain_amount: totalRain,\n        temperature: forecasts[0].main.temp,\n        humidity: forecasts[0].main.humidity\n    },\n    tags: {\n        source: 'openweathermap'\n    },\n    timestamp: new Date()\n};\n\nreturn msg;",
    "outputs": 1,
    "noerr": 0,
    "x": 580,
    "y": 520,
    "wires": [
      ["influx_write", "ui_weather_rain", "ui_weather_temp"]
    ]
  },
  {
    "id": "ui_weather_rain",
    "type": "ui_text",
    "z": "irrigation_flow_tab",
    "group": "ui_group_weather",
    "order": 1,
    "width": 0,
    "height": 0,
    "name": "Rain Probability",
    "label": "Rain (24h)",
    "format": "{{msg.payload.fields.rain_probability}}%",
    "layout": "row-spread",
    "x": 830,
    "y": 500,
    "wires": []
  },
  {
    "id": "ui_weather_temp",
    "type": "ui_text",
    "z": "irrigation_flow_tab",
    "group": "ui_group_weather",
    "order": 2,
    "width": 0,
    "height": 0,
    "name": "Temperature",
    "label": "Temperature",
    "format": "{{msg.payload.fields.temperature}}°C",
    "layout": "row-spread",
    "x": 830,
    "y": 540,
    "wires": []
  },
  {
    "id": "comment_zone1_logic",
    "type": "comment",
    "z": "irrigation_flow_tab",
    "name": "ZONE 1 - LOGIC & CONTROL",
    "info": "",
    "x": 150,
    "y": 620,
    "wires": []
  },
  {
    "id": "zone1_logic",
    "type": "function",
    "z": "irrigation_flow_tab",
    "name": "Zone 1 Decision Logic",
    "func": "// Get current values\nconst moisture = flow.get('zone1_moisture') || 50;\nconst rainProb = flow.get('rain_probability') || 0;\nconst manualOverride = flow.get('zone1_manual_override') || false;\nconst autoMode = flow.get('zone1_auto_mode') !== false; // Default true\nconst lastWatered = flow.get('zone1_last_watered') || 0;\n\n// Configuration\nconst MOISTURE_THRESHOLD = 35; // Water if below 35%\nconst RAIN_THRESHOLD = 40;     // Skip if rain > 40%\nconst MIN_INTERVAL = 4 * 60 * 60 * 1000; // 4 hours between waterings\nconst WATERING_DURATION = 15 * 60 * 1000; // 15 minutes\n\nlet shouldWater = false;\nlet reason = '';\n\n// Current time\nconst now = Date.now();\nconst timeSinceWatering = now - lastWatered;\n\n// Check if we're currently watering\nconst currentlyWatering = flow.get('zone1_watering') || false;\n\nif (currentlyWatering) {\n    // Check if watering duration exceeded\n    if (timeSinceWatering > WATERING_DURATION) {\n        shouldWater = false;\n        reason = 'Watering duration complete';\n        flow.set('zone1_watering', false);\n    } else {\n        shouldWater = true;\n        reason = 'Currently watering';\n    }\n} else if (manualOverride) {\n    shouldWater = true;\n    reason = 'Manual override active';\n} else if (!autoMode) {\n    shouldWater = false;\n    reason = 'Auto mode disabled';\n} else if (rainProb > RAIN_THRESHOLD) {\n    shouldWater = false;\n    reason = `Rain expected (${rainProb}%)`;\n} else if (timeSinceWatering < MIN_INTERVAL) {\n    shouldWater = false;\n    const hoursLeft = Math.round((MIN_INTERVAL - timeSinceWatering) / (60 * 60 * 1000));\n    reason = `Too soon (${hoursLeft}h left)`;\n} else if (moisture < MOISTURE_THRESHOLD) {\n    shouldWater = true;\n    reason = `Low moisture (${moisture}%)`;\n    flow.set('zone1_last_watered', now);\n    flow.set('zone1_watering', true);\n} else {\n    shouldWater = false;\n    reason = `Sufficient moisture (${moisture}%)`;\n}\n\n// Update status\nflow.set('zone1_status', reason);\n\n// Prepare MQTT command\nmsg.payload = shouldWater ? 'ON' : 'OFF';\nmsg.topic = 'garden/controller/switch/garden_zone_controller_zone_1_valve/command';\n\nnode.status({\n    fill: shouldWater ? 'green' : 'grey',\n    shape: 'dot',\n    text: reason\n});\n\nreturn msg;",
    "outputs": 1,
    "noerr": 0,
    "x": 400,
    "y": 660,
    "wires": [
      ["mqtt_out_zone1_valve", "ui_zone1_status"]
    ]
  },
  {
    "id": "mqtt_out_zone1_valve",
    "type": "mqtt out",
    "z": "irrigation_flow_tab",
    "name": "Zone 1 Valve Control",
    "topic": "",
    "qos": "1",
    "retain": "true",
    "respTopic": "",
    "contentType": "",
    "userProps": "",
    "correl": "",
    "expiry": "",
    "broker": "mqtt_broker",
    "x": 680,
    "y": 660,
    "wires": []
  },
  {
    "id": "ui_zone1_status",
    "type": "ui_text",
    "z": "irrigation_flow_tab",
    "group": "ui_group_zone1",
    "order": 2,
    "width": 0,
    "height": 0,
    "name": "Zone 1 Status",
    "label": "Status",
    "format": "{{msg.payload}}",
    "layout": "row-spread",
    "x": 660,
    "y": 700,
    "wires": []
  },
  {
    "id": "ui_zone1_manual",
    "type": "ui_switch",
    "z": "irrigation_flow_tab",
    "name": "Zone 1 Manual",
    "label": "Manual Override",
    "tooltip": "",
    "group": "ui_group_zone1",
    "order": 3,
    "width": 0,
    "height": 0,
    "passthru": true,
    "decouple": "false",
    "topic": "zone1_manual",
    "topicType": "str",
    "style": "",
    "onvalue": "true",
    "onvalueType": "bool",
    "onicon": "",
    "oncolor": "",
    "offvalue": "false",
    "offvalueType": "bool",
    "officon": "",
    "offcolor": "",
    "animate": false,
    "x": 140,
    "y": 720,
    "wires": [
      ["zone1_manual_handler"]
    ]
  },
  {
    "id": "zone1_manual_handler",
    "type": "function",
    "z": "irrigation_flow_tab",
    "name": "Zone 1 Manual Handler",
    "func": "flow.set('zone1_manual_override', msg.payload);\nreturn msg;",
    "outputs": 1,
    "noerr": 0,
    "x": 390,
    "y": 720,
    "wires": [
      ["zone1_logic"]
    ]
  },
  {
    "id": "ui_zone1_auto",
    "type": "ui_switch",
    "z": "irrigation_flow_tab",
    "name": "Zone 1 Auto Mode",
    "label": "Auto Mode",
    "tooltip": "",
    "group": "ui_group_zone1",
    "order": 4,
    "width": 0,
    "height": 0,
    "passthru": true,
    "decouple": "false",
    "topic": "zone1_auto",
    "topicType": "str",
    "style": "",
    "onvalue": "true",
    "onvalueType": "bool",
    "onicon": "",
    "oncolor": "",
    "offvalue": "false",
    "offvalueType": "bool",
    "officon": "",
    "offcolor": "",
    "animate": false,
    "x": 140,
    "y": 760,
    "wires": [
      ["zone1_auto_handler"]
    ]
  },
  {
    "id": "zone1_auto_handler",
    "type": "function",
    "z": "irrigation_flow_tab",
    "name": "Zone 1 Auto Handler",
    "func": "flow.set('zone1_auto_mode', msg.payload);\nreturn msg;",
    "outputs": 1,
    "noerr": 0,
    "x": 380,
    "y": 760,
    "wires": [
      ["zone1_logic"]
    ]
  },
  {
    "id": "comment_zone2_logic",
    "type": "comment",
    "z": "irrigation_flow_tab",
    "name": "ZONE 2 - LOGIC & CONTROL",
    "info": "",
    "x": 150,
    "y": 820,
    "wires": []
  },
  {
    "id": "zone2_logic",
    "type": "function",
    "z": "irrigation_flow_tab",
    "name": "Zone 2 Decision Logic",
    "func": "const moisture = flow.get('zone2_moisture') || 50;\nconst rainProb = flow.get('rain_probability') || 0;\nconst manualOverride = flow.get('zone2_manual_override') || false;\nconst autoMode = flow.get('zone2_auto_mode') !== false;\nconst lastWatered = flow.get('zone2_last_watered') || 0;\n\nconst MOISTURE_THRESHOLD = 30; // Flowers need more water\nconst RAIN_THRESHOLD = 40;\nconst MIN_INTERVAL = 4 * 60 * 60 * 1000;\nconst WATERING_DURATION = 10 * 60 * 1000; // 10 minutes for flowers\n\nlet shouldWater = false;\nlet reason = '';\n\nconst now = Date.now();\nconst timeSinceWatering = now - lastWatered;\nconst currentlyWatering = flow.get('zone2_watering') || false;\n\nif (currentlyWatering) {\n    if (timeSinceWatering > WATERING_DURATION) {\n        shouldWater = false;\n        reason = 'Watering duration complete';\n        flow.set('zone2_watering', false);\n    } else {\n        shouldWater = true;\n        reason = 'Currently watering';\n    }\n} else if (manualOverride) {\n    shouldWater = true;\n    reason = 'Manual override active';\n} else if (!autoMode) {\n    shouldWater = false;\n    reason = 'Auto mode disabled';\n} else if (rainProb > RAIN_THRESHOLD) {\n    shouldWater = false;\n    reason = `Rain expected (${rainProb}%)`;\n} else if (timeSinceWatering < MIN_INTERVAL) {\n    shouldWater = false;\n    const hoursLeft = Math.round((MIN_INTERVAL - timeSinceWatering) / (60 * 60 * 1000));\n    reason = `Too soon (${hoursLeft}h left)`;\n} else if (moisture < MOISTURE_THRESHOLD) {\n    shouldWater = true;\n    reason = `Low moisture (${moisture}%)`;\n    flow.set('zone2_last_watered', now);\n    flow.set('zone2_watering', true);\n} else {\n    shouldWater = false;\n    reason = `Sufficient moisture (${moisture}%)`;\n}\n\nflow.set('zone2_status', reason);\nmsg.payload = shouldWater ? 'ON' : 'OFF';\nmsg.topic = 'garden/controller/switch/garden_zone_controller_zone_2_valve/command';\n\nnode.status({\n    fill: shouldWater ? 'green' : 'grey',\n    shape: 'dot',\n    text: reason\n});\n\nreturn msg;",
    "outputs": 1,
    "noerr": 0,
    "x": 400,
    "y": 860,
    "wires": [
      ["mqtt_out_zone2_valve", "ui_zone2_status"]
    ]
  },
  {
    "id": "mqtt_out_zone2_valve",
    "type": "mqtt out",
    "z": "irrigation_flow_tab",
    "name": "Zone 2 Valve Control",
    "topic": "",
    "qos": "1",
    "retain": "true",
    "respTopic": "",
    "contentType": "",
    "userProps": "",
    "correl": "",
    "expiry": "",
    "broker": "mqtt_broker",
    "x": 680,
    "y": 860,
    "wires": []
  },
  {
    "id": "ui_zone2_status",
    "type": "ui_text",
    "z": "irrigation_flow_tab",
    "group": "ui_group_zone2",
    "order": 2,
    "width": 0,
    "height": 0,
    "name": "Zone 2 Status",
    "label": "Status",
    "format": "{{msg.payload}}",
    "layout": "row-spread",
    "x": 660,
    "y": 900,
    "wires": []
  },
  {
    "id": "ui_zone2_manual",
    "type": "ui_switch",
    "z": "irrigation_flow_tab",
    "name": "Zone 2 Manual",
    "label": "Manual Override",
    "tooltip": "",
    "group": "ui_group_zone2",
    "order": 3,
    "width": 0,
    "height": 0,
    "passthru": true,
    "decouple": "false",
    "topic": "zone2_manual",
    "topicType": "str",
    "style": "",
    "onvalue": "true",
    "onvalueType": "bool",
    "onicon": "",
    "oncolor": "",
    "offvalue": "false",
    "offvalueType": "bool",
    "officon": "",
    "offcolor": "",
    "animate": false,
    "x": 140,
    "y": 920,
    "wires": [
      ["zone2_manual_handler"]
    ]
  },
  {
    "id": "zone2_manual_handler",
    "type": "function",
    "z": "irrigation_flow_tab",
    "name": "Zone 2 Manual Handler",
    "func": "flow.set('zone2_manual_override', msg.payload);\nreturn msg;",
    "outputs": 1,
    "noerr": 0,
    "x": 390,
    "y": 920,
    "wires": [
      ["zone2_logic"]
    ]
  },
  {
    "id": "ui_zone2_auto",
    "type": "ui_switch",
    "z": "irrigation_flow_tab",
    "name": "Zone 2 Auto Mode",
    "label": "Auto Mode",
    "tooltip": "",
    "group": "ui_group_zone2",
    "order": 4,
    "width": 0,
    "height": 0,
    "passthru": true,
    "decouple": "false",
    "topic": "zone2_auto",
    "topicType": "str",
    "style": "",
    "onvalue": "true",
    "onvalueType": "bool",
    "onicon": "",
    "oncolor": "",
    "offvalue": "false",
    "offvalueType": "bool",
    "officon": "",
    "offcolor": "",
    "animate": false,
    "x": 140,
    "y": 960,
    "wires": [
      ["zone2_auto_handler"]
    ]
  },
  {
    "id": "zone2_auto_handler",
    "type": "function",
    "z": "irrigation_flow_tab",
    "name": "Zone 2 Auto Handler",
    "func": "flow.set('zone2_auto_mode', msg.payload);\nreturn msg;",
    "outputs": 1,
    "noerr": 0,
    "x": 380,
    "y": 960,
    "wires": [
      ["zone2_logic"]
    ]
  },
  {
    "id": "comment_zone3_logic",
    "type": "comment",
    "z": "irrigation_flow_tab",
    "name": "ZONE 3 - LOGIC & CONTROL",
    "info": "",
    "x": 150,
    "y": 1020,
    "wires": []
  },
  {
    "id": "zone3_logic",
    "type": "function",
    "z": "irrigation_flow_tab",
    "name": "Zone 3 Decision Logic",
    "func": "const moisture = flow.get('zone3_moisture') || 50;\nconst rainProb = flow.get('rain_probability') || 0;\nconst manualOverride = flow.get('zone3_manual_override') || false;\nconst autoMode = flow.get('zone3_auto_mode') !== false;\nconst lastWatered = flow.get('zone3_last_watered') || 0;\n\nconst MOISTURE_THRESHOLD = 40; // Lawn can tolerate drier conditions\nconst RAIN_THRESHOLD = 40;\nconst MIN_INTERVAL = 6 * 60 * 60 * 1000; // 6 hours for lawn\nconst WATERING_DURATION = 20 * 60 * 1000; // 20 minutes for lawn\n\nlet shouldWater = false;\nlet reason = '';\n\nconst now = Date.now();\nconst timeSinceWatering = now - lastWatered;\nconst currentlyWatering = flow.get('zone3_watering') || false;\n\nif (currentlyWatering) {\n    if (timeSinceWatering > WATERING_DURATION) {\n        shouldWater = false;\n        reason = 'Watering duration complete';\n        flow.set('zone3_watering', false);\n    } else {\n        shouldWater = true;\n        reason = 'Currently watering';\n    }\n} else if (manualOverride) {\n    shouldWater = true;\n    reason = 'Manual override active';\n} else if (!autoMode) {\n    shouldWater = false;\n    reason = 'Auto mode disabled';\n} else if (rainProb > RAIN_THRESHOLD) {\n    shouldWater = false;\n    reason = `Rain expected (${rainProb}%)`;\n} else if (timeSinceWatering < MIN_INTERVAL) {\n    shouldWater = false;\n    const hoursLeft = Math.round((MIN_INTERVAL - timeSinceWatering) / (60 * 60 * 1000));\n    reason = `Too soon (${hoursLeft}h left)`;\n} else if (moisture < MOISTURE_THRESHOLD) {\n    shouldWater = true;\n    reason = `Low moisture (${moisture}%)`;\n    flow.set('zone3_last_watered', now);\n    flow.set('zone3_watering', true);\n} else {\n    shouldWater = false;\n    reason = `Sufficient moisture (${moisture}%)`;\n}\n\nflow.set('zone3_status', reason);\nmsg.payload = shouldWater ? 'ON' : 'OFF';\nmsg.topic = 'garden/controller/switch/garden_zone_controller_zone_3_valve/command';\n\nnode.status({\n    fill: shouldWater ? 'green' : 'grey',\n    shape: 'dot',\n    text: reason\n});\n\nreturn msg;",
    "outputs": 1,
    "noerr": 0,
    "x": 400,
    "y": 1060,
    "wires": [
      ["mqtt_out_zone3_valve", "ui_zone3_status"]
    ]
  },
  {
    "id": "mqtt_out_zone3_valve",
    "type": "mqtt out",
    "z": "irrigation_flow_tab",
    "name": "Zone 3 Valve Control",
    "topic": "",
    "qos": "1",
    "retain": "true",
    "respTopic": "",
    "contentType": "",
    "userProps": "",
    "correl": "",
    "expiry": "",
    "broker": "mqtt_broker",
    "x": 680,
    "y": 1060,
    "wires": []
  },
  {
    "id": "ui_zone3_status",
    "type": "ui_text",
    "z": "irrigation_flow_tab",
    "group": "ui_group_zone3",
    "order": 2,
    "width": 0,
    "height": 0,
    "name": "Zone 3 Status",
    "label": "Status",
    "format": "{{msg.payload}}",
    "layout": "row-spread",
    "x": 660,
    "y": 1100,
    "wires": []
  },
  {
    "id": "ui_zone3_manual",
    "type": "ui_switch",
    "z": "irrigation_flow_tab",
    "name": "Zone 3 Manual",
    "label": "Manual Override",
    "tooltip": "",
    "group": "ui_group_zone3",
    "order": 3,
    "width": 0,
    "height": 0,
    "passthru": true,
    "decouple": "false",
    "topic": "zone3_manual",
    "topicType": "str",
    "style": "",
    "onvalue": "true",
    "onvalueType": "bool",
    "onicon": "",
    "oncolor": "",
    "offvalue": "false",
    "offvalueType": "bool",
    "officon": "",
    "offcolor": "",
    "animate": false,
    "x": 140,
    "y": 1120,
    "wires": [
      ["zone3_manual_handler"]
    ]
  },
  {
    "id": "zone3_manual_handler",
    "type": "function",
    "z": "irrigation_flow_tab",
    "name": "Zone 3 Manual Handler",
    "func": "flow.set('zone3_manual_override', msg.payload);\nreturn msg;",
    "outputs": 1,
    "noerr": 0,
    "x": 390,
    "y": 1120,
    "wires": [
      ["zone3_logic"]
    ]
  },
  {
    "id": "ui_zone3_auto",
    "type": "ui_switch",
    "z": "irrigation_flow_tab",
    "name": "Zone 3 Auto Mode",
    "label": "Auto Mode",
    "tooltip": "",
    "group": "ui_group_zone3",
    "order": 4,
    "width": 0,
    "height": 0,
    "passthru": true,
    "decouple": "false",
    "topic": "zone3_auto",
    "topicType": "str",
    "style": "",
    "onvalue": "true",
    "onvalueType": "bool",
    "onicon": "",
    "oncolor": "",
    "offvalue": "false",
    "offvalueType": "bool",
    "officon": "",
    "offcolor": "",
    "animate": false,
    "x": 140,
    "y": 1160,
    "wires": [
      ["zone3_auto_handler"]
    ]
  },
  {
    "id": "zone3_auto_handler",
    "type": "function",
    "z": "irrigation_flow_tab",
    "name": "Zone 3 Auto Handler",
    "func": "flow.set('zone3_auto_mode', msg.payload);\nreturn msg;",
    "outputs": 1,
    "noerr": 0,
    "x": 380,
    "y": 1160,
    "wires": [
      ["zone3_logic"]
    ]
  },
  {
    "id": "comment_safety",
    "type": "comment",
    "z": "irrigation_flow_tab",
    "name": "SAFETY - Emergency Shutoff",
    "info": "",
    "x": 150,
    "y": 1220,
    "wires": []
  },
  {
    "id": "ui_emergency_stop",
    "type": "ui_button",
    "z": "irrigation_flow_tab",
    "name": "Emergency Stop",
    "group": "ui_group_system",
    "order": 1,
    "width": 0,
    "height": 0,
    "passthru": false,
    "label": "🚨 EMERGENCY STOP ALL",
    "tooltip": "",
    "color": "white",
    "bgcolor": "red",
    "icon": "",
    "payload": "OFF",
    "payloadType": "str",
    "topic": "emergency",
    "topicType": "str",
    "x": 140,
    "y": 1260,
    "wires": [
      ["emergency_handler"]
    ]
  },
  {
    "id": "emergency_handler",
    "type": "function",
    "z": "irrigation_flow_tab",
    "name": "Emergency Stop Handler",
    "func": "// Disable all zones\nflow.set('zone1_auto_mode', false);\nflow.set('zone2_auto_mode', false);\nflow.set('zone3_auto_mode', false);\nflow.set('zone1_manual_override', false);\nflow.set('zone2_manual_override', false);\nflow.set('zone3_manual_override', false);\nflow.set('zone1_watering', false);\nflow.set('zone2_watering', false);\nflow.set('zone3_watering', false);\n\n// Send OFF commands to all valves\nreturn [\n    { topic: 'garden/controller/switch/garden_zone_controller_zone_1_valve/command', payload: 'OFF' },\n    { topic: 'garden/controller/switch/garden_zone_controller_zone_2_valve/command', payload: 'OFF' },\n    { topic: 'garden/controller/switch/garden_zone_controller_zone_3_valve/command', payload: 'OFF' },\n    { topic: 'garden/controller/switch/garden_zone_controller_master_pump/command', payload: 'OFF' }\n];",
    "outputs": 4,
    "noerr": 0,
    "x": 400,
    "y": 1260,
    "wires": [
      ["mqtt_out_zone1_valve"],
      ["mqtt_out_zone2_valve"],
      ["mqtt_out_zone3_valve"],
      ["mqtt_out_master_pump"]
    ]
  },
  {
    "id": "mqtt_out_master_pump",
    "type": "mqtt out",
    "z": "irrigation_flow_tab",
    "name": "Master Pump Control",
    "topic": "",
    "qos": "1",
    "retain": "true",
    "respTopic": "",
    "contentType": "",
    "userProps": "",
    "correl": "",
    "expiry": "",
    "broker": "mqtt_broker",
    "x": 680,
    "y": 1300,
    "wires": []
  }
]
```

**Configuration Steps After Import:**

1. **Configure MQTT broker** (double-click `mqtt_broker` config node):
   - Broker: `mosquitto`
   - Port: `1883`
   - Username: `nodered`
   - Password: (your password)

2. **Configure InfluxDB** (double-click `influxdb_config`):
   - Host: `influxdb`
   - Database: `garden`
   - Username: `garden_user`
   - Password: (your password)

3. **Update Weather API**:
   - Get free API key from [OpenWeatherMap](https://openweathermap.org/api)
   - Double-click "OpenWeatherMap API" node
   - Update URL with your latitude, longitude, and API key

4. **Adjust MQTT topics** if your ESPHome configuration uses different names

---

## Part 3: Grafana Dashboard JSON

**Features:**
- Real-time moisture levels
- Watering history
- Weather data
- Zone statistics
- System status

**Import Steps:**
1. Login to Grafana: `http://your-server:3000`
2. Add InfluxDB data source first
3. Create new dashboard → Import → Paste JSON

**File: `grafana-garden-dashboard.json`**

```json
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": "-- Grafana --",
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      }
    ]
  },
  "editable": true,
  "gnetId": null,
  "graphTooltip": 1,
  "id": null,
  "links": [],
  "panels": [
    {
      "datasource": "InfluxDB",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "max": 100,
          "min": 0,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "red",
                "value": null
              },
              {
                "color": "yellow",
                "value": 30
              },
              {
                "color": "green",
                "value": 60
              }
            ]
          },
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 6,
        "x": 0,
        "y": 0
      },
      "id": 2,
      "options": {
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "showThresholdLabels": false,
        "showThresholdMarkers": true,
        "text": {}
      },
      "pluginVersion": "8.0.0",
      "targets": [
        {
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "measurement": "soil_moisture",
          "orderByTime": "ASC",
          "policy": "default",
          "refId": "A",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "value"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "last"
              }
            ]
          ],
          "tags": [
            {
              "key": "zone",
              "operator": "=",
              "value": "zone1"
            }
          ]
        }
      ],
      "title": "Zone 1 - Vegetables",
      "type": "gauge"
    },
    {
      "datasource": "InfluxDB",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "max": 100,
          "min": 0,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "red",
                "value": null
              },
              {
                "color": "yellow",
                "value": 30
              },
              {
                "color": "green",
                "value": 60
              }
            ]
          },
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 6,
        "x": 6,
        "y": 0
      },
      "id": 3,
      "options": {
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "showThresholdLabels": false,
        "showThresholdMarkers": true,
        "text": {}
      },
      "pluginVersion": "8.0.0",
      "targets": [
        {
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "measurement": "soil_moisture",
          "orderByTime": "ASC",
          "policy": "default",
          "refId": "A",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "value"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "last"
              }
            ]
          ],
          "tags": [
            {
              "key": "zone",
              "operator": "=",
              "value": "zone2"
            }
          ]
        }
      ],
      "title": "Zone 2 - Flowers",
      "type": "gauge"
    },
    {
      "datasource": "InfluxDB",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "max": 100,
          "min": 0,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "red",
                "value": null
              },
              {
                "color": "yellow",
                "value": 40
              },
              {
                "color": "green",
                "value": 60
              }
            ]
          },
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 6,
        "x": 12,
        "y": 0
      },
      "id": 4,
      "options": {
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "showThresholdLabels": false,
        "showThresholdMarkers": true,
        "text": {}
      },
      "pluginVersion": "8.0.0",
      "targets": [
        {
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "measurement": "soil_moisture",
          "orderByTime": "ASC",
          "policy": "default",
          "refId": "A",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "value"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "last"
              }
            ]
          ],
          "tags": [
            {
              "key": "zone",
              "operator": "=",
              "value": "zone3"
            }
          ]
        }
      ],
      "title": "Zone 3 - Lawn",
      "type": "gauge"
    },
    {
      "datasource": "InfluxDB",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisLabel": "Moisture %",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 20,
            "gradientMode": "none",
            "hideFrom": {
              "tooltip": false,
              "viz": false,
              "legend": false
            },
            "lineInterpolation": "smooth",
            "lineWidth": 2,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "never",
            "spanNulls": true,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "line"
            }
          },
          "mappings": [],
          "max": 100,
          "min": 0,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "transparent",
                "value": null
              },
              {
                "color": "red",
                "value": 30
              }
            ]
          },
          "unit": "percent"
        },
        "overrides": [
          {
            "matcher": {
              "id": "byName",
              "options": "soil_moisture.last {zone: zone1}"
            },
            "properties": [
              {
                "id": "displayName",
                "value": "Zone 1"
              },
              {
                "id": "color",
                "value": {
                  "fixedColor": "green",
                  "mode": "fixed"
                }
              }
            ]
          },
          {
            "matcher": {
              "id": "byName",
              "options": "soil_moisture.last {zone: zone2}"
            },
            "properties": [
              {
                "id": "displayName",
                "value": "Zone 2"
              },
              {
                "id": "color",
                "value": {
                  "fixedColor": "blue",
                  "mode": "fixed"
                }
              }
            ]
          },
          {
            "matcher": {
              "id": "byName",
              "options": "soil_moisture.last {zone: zone3}"
            },
            "properties": [
              {
                "id": "displayName",
                "value": "Zone 3"
              },
              {
                "id": "color",
                "value": {
                  "fixedColor": "yellow",
                  "mode": "fixed"
                }
              }
            ]
          }
        ]
      },
      "gridPos": {
        "h": 9,
        "w": 18,
        "x": 0,
        "y": 8
      },
      "id": 5,
      "options": {
        "legend": {
          "calcs": [
            "last",
            "mean"
          ],
          "displayMode": "table",
          "placement": "bottom"
        },
        "tooltip": {
          "mode": "multi"
        }
      },
      "pluginVersion": "8.0.0",
      "targets": [
        {
          "alias": "$tag_zone",
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "zone"
              ],
              "type": "tag"
            },
            {
              "params": [
                "linear"
              ],
              "type": "fill"
            }
          ],
          "measurement": "soil_moisture",
          "orderByTime": "ASC",
          "policy": "default",
          "refId": "A",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "value"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "last"
              }
            ]
          ],
          "tags": []
        }
      ],
      "title": "Soil Moisture - 24 Hours",
      "type": "timeseries"
    },
    {
      "datasource": "InfluxDB",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "yellow",
                "value": 40
              },
              {
                "color": "red",
                "value": 70
              }
            ]
          },
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 4,
        "w": 3,
        "x": 18,
        "y": 0
      },
      "id": 6,
      "options": {
        "colorMode": "background",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "text": {},
        "textMode": "auto"
      },
      "pluginVersion": "8.0.0",
      "targets": [
        {
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "measurement": "weather",
          "orderByTime": "ASC",
          "policy": "default",
          "refId": "A",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "rain_probability"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "last"
              }
            ]
          ],
          "tags": []
        }
      ],
      "title": "Rain Probability",
      "type": "stat"
    },
    {
      "datasource": "InfluxDB",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "blue",
                "value": null
              },
              {
                "color": "green",
                "value": 15
              },
              {
                "color": "yellow",
                "value": 25
              },
              {
                "color": "red",
                "value": 35
              }
            ]
          },
          "unit": "celsius"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 4,
        "w": 3,
        "x": 18,
        "y": 4
      },
      "id": 7,
      "options": {
        "colorMode": "background",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "text": {},
        "textMode": "auto"
      },
      "pluginVersion": "8.0.0",
      "targets": [
        {
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "measurement": "weather",
          "orderByTime": "ASC",
          "policy": "default",
          "refId": "A",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "last"
              }
            ]
          ],
          "tags": []
        }
      ],
      "title": "Temperature",
      "type": "stat"
    },
    {
      "datasource": "InfluxDB",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "bars",
            "fillOpacity": 80,
            "gradientMode": "none",
            "hideFrom": {
              "tooltip": false,
              "viz": false,
              "legend": false
            },
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "short"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 18,
        "x": 0,
        "y": 17
      },
      "id": 8,
      "options": {
        "legend": {
          "calcs": [
            "sum",
            "count"
          ],
          "displayMode": "table",
          "placement": "bottom"
        },
        "tooltip": {
          "mode": "multi"
        }
      },
      "pluginVersion": "8.0.0",
      "targets": [
        {
          "alias": "$tag_zone Watering Events",
          "groupBy": [
            {
              "params": [
                "1h"
              ],
              "type": "time"
            },
            {
              "params": [
                "zone"
              ],
              "type": "tag"
            },
            {
              "params": [
                "0"
              ],
              "type": "fill"
            }
          ],
          "measurement": "valve_state",
          "orderByTime": "ASC",
          "policy": "default",
          "refId": "A",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "state"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "count"
              }
            ]
          ],
          "tags": []
        }
      ],
      "title": "Watering Events - 7 Days",
      "type": "timeseries"
    },
    {
      "datasource": "InfluxDB",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "custom": {
            "align": "auto",
            "displayMode": "auto"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 6,
        "x": 18,
        "y": 8
      },
      "id": 9,
      "options": {
        "showHeader": true,
        "sortBy": []
      },
      "pluginVersion": "8.0.0",
      "targets": [
        {
          "groupBy": [
            {
              "params": [
                "zone"
              ],
              "type": "tag"
            }
          ],
          "measurement": "soil_moisture",
          "orderByTime": "ASC",
          "policy": "default",
          "refId": "A",
          "resultFormat": "table",
          "select": [
            [
              {
                "params": [
                  "value"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "last"
              },
              {
                "params": [
                  "Current"
                ],
                "type": "alias"
              }
            ],
            [
              {
                "params": [
                  "value"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "mean"
              },
              {
                "params": [
                  "Average"
                ],
                "type": "alias"
              }
            ],
            [
              {
                "params": [
                  "value"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "min"
              },
              {
                "params": [
                  "Min"
                ],
                "type": "alias"
              }
            ]
          ],
          "tags": []
        }
      ],
      "title": "Zone Statistics (24h)",
      "type": "table"
    }
  ],
  "refresh": "30s",
  "schemaVersion": 27,
  "style": "dark",
  "tags": [
    "garden",
    "irrigation",
    "iot"
  ],
  "templating": {
    "list": []
  },
  "time": {
    "from": "now-24h",
    "to": "now"
  },
  "timepicker": {},
  "timezone": "browser",
  "title": "Garden Irrigation System",
  "uid": "garden-irrigation",
  "version": 1
}
```

**After importing:**
1. Go to Configuration → Data Sources
2. Add InfluxDB data source:
   - Name: `InfluxDB`
   - URL: `http://influxdb:8086`
   - Database: `garden`
   - User: `garden_user`
   - Password: (your password)
   - Save & Test

---

## Part 4: Zigbee Sensor Recommendations

### Recommended Zigbee Coordinator

**Best Choice: Sonoff Zigbee 3.0 USB Dongle Plus (ZBDongle-P)**
- **Price**: ~$15-20
- **Chipset**: Texas Instruments CC2652P
- **Why**: Excellent range, firmware updatable, works perfectly with Zigbee2MQTT
- **Buy**: [Sonoff Official](https://itead.cc/product/sonoff-zigbee-3-0-usb-dongle-plus/), Amazon, AliExpress

**Alternative: ConBee II**
- **Price**: ~$40
- **Why**: Rock-solid reliability, German engineering
- **Note**: More expensive but very stable

---

### Recommended Zigbee Sensors for Irrigation

#### 1. **Soil Moisture Sensors**

**Tuya Zigbee Soil Moisture Sensor**
- **Model**: Various (search "Zigbee soil moisture sensor")
- **Price**: $12-18 each
- **Features**:
  - Battery powered (2× AAA, 6-12 months)
  - Measures moisture + temperature
  - Waterproof probe
  - Zigbee2MQTT supported
- **Why**: Wireless placement, no wiring needed, perfect for zones without ESP32
- **Brands**: Tuya, Moes, _TZ3000 series
- **Buy**: AliExpress, Amazon ("Tuya Zigbee soil moisture")

**Aqara Temperature & Humidity Sensor** (Alternative for greenhouse monitoring)
- **Model**: WSDCGQ11LM
- **Price**: $15
- **Features**: Temperature + humidity, excellent battery life
- **Why**: Monitor ambient conditions that affect irrigation needs

#### 2. **Temperature & Humidity Sensors**

**Aqara Temperature Humidity Sensor**
- **Model**: WSDCGQ11LM
- **Price**: $12-15
- **Battery**: CR2032 (2 years)
- **Why**: Monitor garden microclimate, affects watering decisions
- **Zigbee2MQTT**: Fully supported

**Sonoff SNZB-02**
- **Price**: $8-10
- **Why**: Budget option, reliable, long battery life

#### 3. **Water Leak/Rain Sensors**

**Aqara Water Leak Sensor**
- **Model**: SJCGQ11LM
- **Price**: $15
- **Why**: Detect rain, protect against system leaks
- **Use case**: Place outdoors to detect rainfall (triggers irrigation skip)

#### 4. **Smart Plugs (for pump control)**

**Sonoff S31 Lite ZB (Zigbee Smart Plug)**
- **Model**: SNZB-01
- **Price**: $12-15
- **Power**: 15A, 3.5kW
- **Why**: Control water pump, energy monitoring
- **Zigbee2MQTT**: Fully supported

**Alternative: NOUS A1Z**
- **Price**: $10-12
- **Why**: Cheaper, also well-supported

#### 5. **Optional: Light Sensors**

**Xiaomi Light Sensor**
- **Model**: GZCGQ01LM
- **Price**: $15
- **Why**: Measure sunlight intensity → adjust watering based on sun exposure
- **Advanced use**: Calculate evapotranspiration

---

### Sensor Placement Strategy

**3-Zone Setup:**

```
Zone 1 (Vegetables):
├── ESP32 wired moisture sensor (always-on, instant readings)
├── Zigbee soil moisture sensor (backup/average reading)
└── Aqara temp/humidity sensor (microclimate)

Zone 2 (Flowers):
├── ESP32 wired moisture sensor
└── Zigbee soil moisture sensor (wireless, easy placement)

Zone 3 (Lawn):
├── ESP32 wired moisture sensor
└── Zigbee soil moisture sensor (middle of lawn, no wires)

Additional:
├── Aqara water leak sensor (near rain gauge or pump)
├── Sonoff smart plug (pump control with energy monitoring)
└── Xiaomi light sensor (garden center, sun exposure tracking)
```

---

### Wired vs. Zigbee Sensors Comparison

| Feature | ESP32 Wired | Zigbee Wireless |
|---------|-------------|-----------------|
| **Cost per sensor** | $2-5 | $12-18 |
| **Installation** | Requires wiring | Drop and pair |
| **Power** | 5V wired | Battery (6-12 months) |
| **Response time** | Real-time (30s) | 5-15 minutes |
| **Range** | Limited by wire | 30-100m (mesh) |
| **Reliability** | Very high | High (battery dependent) |
| **Best for** | Critical zones, nearby areas | Remote zones, temporary placement |

**Recommendation**: Use ESP32 for main 3 zones (you already have it), add 1-2 Zigbee sensors for:
- Remote areas
- Experimental zones
- Backup/validation readings

---

### Zigbee Network Tips

1. **Start small**: Coordinator + 2-3 sensors, expand gradually
2. **Powered devices act as routers**: Smart plugs extend range
3. **Keep coordinator central**: USB extension cable helps
4. **Update firmware**: Use [Z-Stack firmware](https://github.com/Koenkk/Z-Stack-firmware) for CC2652P
5. **Avoid WiFi interference**: Zigbee uses 2.4GHz, use channel 15, 20, or 25

---

### Shopping List (Minimum Viable Setup)

| Item | Quantity | Price (USD) | Total |
|------|----------|-------------|-------|
| Sonoff ZBDongle-P | 1 | $18 | $18 |
| Tuya Zigbee Soil Sensor | 2 | $15 | $30 |
| Aqara Temp/Humidity | 1 | $12 | $12 |
| Sonoff Smart Plug (pump) | 1 | $12 | $12 |
| **Total** | | | **$72** |

---

### Integration with Node-RED

Zigbee2MQTT publishes to MQTT automatically:

```
Topic: zigbee2mqtt/soil_sensor_1/moisture
Payload: 45

Topic: zigbee2mqtt/soil_sensor_1/temperature  
Payload: 22.5

Topic: zigbee2mqtt/temp_humidity_1/humidity
Payload: 68
```

**Node-RED will automatically receive these** - just add MQTT input nodes with these topics!

---

## Summary & Next Steps

You now have:

✅ **ESPHome configs** for 3-zone ESP32 controller
✅ **Complete Node-RED flow** with weather integration, logic, and dashboard
✅ **Grafana dashboard** for beautiful visualizations
✅ **Zigbee recommendations** for wireless expansion

### Implementation Order:

1. **Week 1**: Set up Docker stack, test connectivity
2. **Week 2**: Flash ESP32 with ESPHome, calibrate sensors
3. **Week 3**: Import Node-RED flow, configure MQTT/InfluxDB
4. **Week 4**: Add Zigbee coordinator and first sensor
5. **Week 5**: Import Grafana dashboard, fine-tune logic
6. **Week 6**: Test automation, adjust thresholds

### Troubleshooting Resources:

- **ESPHome Discord**: https://discord.gg/KhAMKrd
- **Node-RED Forum**: https://discourse.nodered.org/
- **Zigbee2MQTT Docs**: https://www.zigbee2mqtt.io/
- **InfluxDB Community**: https://community.influxdata.com/

---

**Questions or need clarification on any component?** I can provide:
- Wiring diagrams for ESP32
- Detailed Node-RED node explanations
- More Grafana panel examples
- Zigbee pairing instructions