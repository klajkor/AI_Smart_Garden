# Complete Smart Garden Irrigation Stack with Zigbee Support

## Updated Component List

### All Components (with Zigbee2MQTT added)

1. **Eclipse Mosquitto** - MQTT Broker
2. **Zigbee2MQTT** - Zigbee device integration
3. **Node-RED** - Automation engine
4. **InfluxDB 1.8** - Time-series database
5. **Grafana** - Visualization dashboards

**Total RAM estimate**: ~570-670MB (comfortable for 2GB system)

---

## Complete Docker Compose Configuration

```yaml
version: '3.8'

services:
  mosquitto:
    image: eclipse-mosquitto:2.0
    container_name: mosquitto
    restart: unless-stopped
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log
    networks:
      - iot-network

  zigbee2mqtt:
    image: koenkk/zigbee2mqtt:latest
    container_name: zigbee2mqtt
    restart: unless-stopped
    ports:
      - "8080:8080"  # Web interface
    volumes:
      - ./zigbee2mqtt/data:/app/data
      - /run/udev:/run/udev:ro
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0  # Adjust to your Zigbee coordinator
    environment:
      - TZ=Europe/London  # Set your timezone
    depends_on:
      - mosquitto
    networks:
      - iot-network

  influxdb:
    image: influxdb:1.8
    container_name: influxdb
    restart: unless-stopped
    ports:
      - "8086:8086"
    volumes:
      - ./influxdb/data:/var/lib/influxdb
    environment:
      - INFLUXDB_DB=garden
      - INFLUXDB_ADMIN_USER=admin
      - INFLUXDB_ADMIN_PASSWORD=secure_admin_pass_CHANGEME
      - INFLUXDB_USER=garden_user
      - INFLUXDB_USER_PASSWORD=secure_user_pass_CHANGEME
      - INFLUXDB_HTTP_AUTH_ENABLED=true
    networks:
      - iot-network

  nodered:
    image: nodered/node-red:latest
    container_name: nodered
    restart: unless-stopped
    ports:
      - "1880:1880"
    volumes:
      - ./nodered/data:/data
    environment:
      - TZ=Europe/London  # Set your timezone
    depends_on:
      - mosquitto
      - influxdb
    networks:
      - iot-network
    user: "1000:1000"  # Match your user ID (run 'id -u' to check)

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - ./grafana/data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=secure_grafana_pass_CHANGEME
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=http://localhost:3000
      - GF_AUTH_ANONYMOUS_ENABLED=false
    depends_on:
      - influxdb
    networks:
      - iot-network
    user: "1000:1000"  # Match your user ID

networks:
  iot-network:
    driver: bridge
```

---

## Initial Setup Instructions

### 1. Create Directory Structure

```bash
mkdir -p ~/garden-automation/{mosquitto/{config,data,log},zigbee2mqtt/data,nodered/data,influxdb/data,grafana/data}
cd ~/garden-automation
```

### 2. Mosquitto Configuration

Create `mosquitto/config/mosquitto.conf`:

```conf
# mosquitto.conf
listener 1883
allow_anonymous false
password_file /mosquitto/config/password.txt

# Persistence
persistence true
persistence_location /mosquitto/data/

# Logging
log_dest file /mosquitto/log/mosquitto.log
log_dest stdout
log_type all
log_timestamp true

# WebSocket support (optional)
listener 9001
protocol websockets
```

**Create password file** (after first run):

```bash
# Start mosquitto first
docker-compose up -d mosquitto

# Create user for your devices
docker exec -it mosquitto mosquitto_passwd -c /mosquitto/config/password.txt mqtt_user

# Add additional users
docker exec -it mosquitto mosquitto_passwd /mosquitto/config/password.txt nodered
docker exec -it mosquitto mosquitto_passwd /mosquitto/config/password.txt zigbee2mqtt

# Restart mosquitto
docker-compose restart mosquitto
```

### 3. Zigbee2MQTT Configuration

**Hardware Requirements**: You need a Zigbee coordinator USB stick:
- **Recommended**: Sonoff Zigbee 3.0 USB Dongle Plus (ZBDongle-P)
- **Alternative**: ConBee II, CC2531, CC2652

Create `zigbee2mqtt/data/configuration.yaml`:

```yaml
# Zigbee2MQTT configuration
homeassistant: false

# MQTT settings
mqtt:
  base_topic: zigbee2mqtt
  server: mqtt://mosquitto:1883
  user: zigbee2mqtt
  password: your_zigbee2mqtt_mqtt_password

# Serial settings (adjust based on your coordinator)
serial:
  port: /dev/ttyUSB0
  adapter: auto  # or 'zstack' for CC2652, 'deconz' for ConBee

# Web interface
frontend:
  port: 8080
  auth_token: secure_web_token_CHANGEME

# Advanced settings
advanced:
  log_level: info
  pan_id: GENERATE
  network_key: GENERATE
  channel: 11

# Device options
device_options:
  retain: true

# Availability
availability: true
```

**Find your Zigbee coordinator device**:

```bash
# List USB devices
ls -l /dev/ttyUSB* /dev/ttyACM*

# Or use:
dmesg | grep -i usb
```

Update the `devices:` section in docker-compose.yml accordingly (might be `/dev/ttyACM0` instead).

### 4. Node-RED Authentication

After first start, enable authentication:

```bash
# Start Node-RED
docker-compose up -d nodered

# Generate password hash
docker exec -it nodered npx node-red-admin hash-pw

# Copy the hash, then edit settings file
nano nodered/data/settings.js
```

Find and uncomment the `adminAuth` section:

```javascript
adminAuth: {
    type: "credentials",
    users: [{
        username: "admin",
        password: "$2b$08$YOUR_GENERATED_HASH_HERE",
        permissions: "*"
    }]
},
```

Restart Node-RED:
```bash
docker-compose restart nodered
```

### 5. Install Node-RED Nodes

Access Node-RED at `http://your-server-ip:1880`, login, and install these nodes:

**Menu (≡) → Manage palette → Install tab**:

```
node-red-contrib-influxdb       # InfluxDB integration
node-red-dashboard              # Built-in UI dashboard
node-red-contrib-openweathermap # Weather API
node-red-contrib-schedex        # Advanced scheduling (optional)
node-red-contrib-sun-position   # Sunrise/sunset calculations
```

---

## Weather API Integration

### Recommended: OpenWeatherMap

**Why?**
- Free tier: 1,000 calls/day (more than enough for irrigation)
- Current weather + 5-day forecast
- Global coverage
- Easy Node-RED integration

**Setup**:

1. **Get API key**: Sign up at [OpenWeatherMap](https://openweathermap.org/api)
2. **Free tier includes**:
   - Current weather
   - 5-day forecast
   - Hourly forecast (48 hours)

**Node-RED Flow Example**:

Install `node-red-contrib-openweathermap`, then use this logic:

```
[Inject: Daily at 6 AM]
    ↓
[OpenWeatherMap: Get forecast]
    ↓
[Function: Check rain probability]
    ↓
[Decision: Skip watering if rain > 50%]
```

**Alternative**: [Open-Meteo](https://open-meteo.com/)
- Completely free, no API key needed
- Use HTTP request node in Node-RED
- Example: `https://api.open-meteo.com/v1/forecast?latitude=52.52&longitude=13.41&current_weather=true&hourly=precipitation_probability`

---

## Node-RED Flow Structure (Example)

### Basic 3-Zone Irrigation Flow

```
┌─────────────────────────────────────────────────┐
│ MQTT Input: Soil Moisture Sensors              │
│  - zigbee2mqtt/zone1/soil_moisture              │
│  - zigbee2mqtt/zone2/soil_moisture              │
│  - zigbee2mqtt/zone3/soil_moisture              │
└─────────────────┬───────────────────────────────┘
                  ↓
┌─────────────────────────────────────────────────┐
│ Store to InfluxDB                               │
└─────────────────┬───────────────────────────────┘
                  ↓
┌─────────────────────────────────────────────────┐
│ Decision Logic:                                 │
│  - Check moisture threshold (< 30%)             │
│  - Check weather forecast (rain probability)    │
│  - Check schedule (allowed watering times)      │
│  - Check manual override switch                 │
└─────────────────┬───────────────────────────────┘
                  ↓
┌─────────────────────────────────────────────────┐
│ MQTT Output: Control Valves/Pumps              │
│  - esp32/zone1/valve (ON/OFF)                   │
│  - esp32/zone2/valve (ON/OFF)                   │
│  - esp32/zone3/valve (ON/OFF)                   │
└─────────────────────────────────────────────────┘
```

### Example Function Node (Watering Decision Logic)

```javascript
// Watering decision for Zone 1
const moisture = msg.payload.soil_moisture;
const rainProbability = flow.get('rain_probability') || 0;
const manualOverride = flow.get('zone1_manual_override') || false;

// Thresholds
const MOISTURE_THRESHOLD = 30; // Water if below 30%
const RAIN_THRESHOLD = 40;     // Skip if rain > 40%

// Decision logic
if (manualOverride) {
    msg.payload = "OFF";
    node.status({fill:"red",shape:"dot",text:"Manual override"});
} else if (rainProbability > RAIN_THRESHOLD) {
    msg.payload = "OFF";
    node.status({fill:"blue",shape:"dot",text:"Rain expected"});
} else if (moisture < MOISTURE_THRESHOLD) {
    msg.payload = "ON";
    node.status({fill:"green",shape:"dot",text:"Watering needed"});
} else {
    msg.payload = "OFF";
    node.status({fill:"grey",shape:"dot",text:"Sufficient moisture"});
}

return msg;
```

---

## Access Points & Credentials Summary

| Service | URL | Default Login |
|---------|-----|---------------|
| **Node-RED** | http://server-ip:1880 | admin / (set during setup) |
| **Zigbee2MQTT** | http://server-ip:8080 | Token-based (see config) |
| **Grafana** | http://server-ip:3000 | admin / secure_grafana_pass_CHANGEME |
| **InfluxDB** | http://server-ip:8086 | admin / secure_admin_pass_CHANGEME |

---

## Startup & Management Commands

```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f

# View specific service
docker-compose logs -f zigbee2mqtt

# Stop all services
docker-compose down

# Restart specific service
docker-compose restart nodered

# Check resource usage
docker stats

# Update all containers
docker-compose pull
docker-compose up -d
```

---

## Memory Optimization for 2GB System

Add to `docker-compose.yml` under each service:

```yaml
services:
  influxdb:
    # ... existing config
    mem_limit: 300m
    memswap_limit: 300m
  
  nodered:
    mem_limit: 200m
    memswap_limit: 200m
    
  grafana:
    mem_limit: 200m
    memswap_limit: 200m
    
  zigbee2mqtt:
    mem_limit: 150m
    memswap_limit: 150m
```

---

## Next Steps

1. **Set up hardware**:
   - Flash ESP8266/ESP32 with MQTT firmware (ESPHome or Tasmota recommended)
   - Plug in Zigbee coordinator
   - Pair Zigbee sensors via Zigbee2MQTT web interface

2. **Create Node-RED flows**:
   - Start with simple: read sensor → display on dashboard
   - Add: sensor → store in InfluxDB
   - Build: decision logic → control actuators

3. **Configure Grafana dashboards**:
   - Add InfluxDB as data source
   - Create panels for moisture levels, watering history, weather data

4. **Test each zone individually** before automating

---

## Useful References

- **Zigbee2MQTT**: [Supported devices list](https://www.zigbee2mqtt.io/supported-devices/)
- **Node-RED Flows**: [Community flows](https://flows.nodered.org/) - search "irrigation"
- **ESPHome**: [ESP32/ESP8266 firmware](https://esphome.io/) - easier than Arduino IDE
- **Tasmota**: [Alternative firmware](https://tasmota.github.io/docs/)

---

Would you like me to provide:
1. **Sample ESP32/ESP8266 code** for soil moisture sensors and valve control?
2. **Complete Node-RED flow JSON** for 3-zone irrigation system?
3. **Grafana dashboard JSON** for monitoring?
4. **Detailed Zigbee sensor recommendations** for your use case?