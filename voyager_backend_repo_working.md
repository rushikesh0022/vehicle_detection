# Vehicle Zone Intelligence - Complete Setup Guide

## Overview

This system provides real-time vehicle detection, tracking, and zone analytics using:

- **Axelera Metis AIPU** for edge AI inference (YOLOv8s-COCO)
- **OC-SORT** for multi-object tracking
- **FastAPI + Redis + TimescaleDB** for cloud backend
- **mediamtx** for local RTSP streaming (demo purposes)

## Quick Start (If Already Installed)

### Terminal 1: Start RTSP Server

```bash
mediamtx /tmp/mediamtx.yml &
sleep 3
```

### Terminal 2: Start Application

```bash
source /home/admin1/voyager-sdk/venv/bin/activate
cd /home/admin1/Desktop/vehicle/Vehicle_Detection

export SDK_ROOT=/home/admin1/voyager-sdk
export VOYAGER_PYTHON="$SDK_ROOT/venv/bin/python"
export VOYAGER_ACTIVATE="$SDK_ROOT/venv/bin/activate"
export VOYAGER_NETWORK=yolov8s-coco

./start.sh
```

### Terminal 3: Add Camera + Zone

```bash
cd /home/admin1/Desktop/vehicle/Vehicle_Detection

curl -X DELETE http://localhost:8002/api/cameras/demo_cam 2>/dev/null

curl -X POST http://localhost:8002/api/cameras \
 -H "Content-Type: application/json" \
 -d '{
"camera_id": "demo_cam",
"name": "Traffic RTSP",
"source_url": "rtsp://localhost:8554/stream",
"assigned_edge": "vehicle-edge-01"
}'

curl -X POST http://localhost:8002/api/zones \
 -H "Content-Type: application/json" \
 -d '{
"zone_id": "demo_zone",
"name": "Full Frame",
"camera_id": "demo_cam",
"zone_poly": [[0,0],[800,0],[800,500],[0,500]],
"max_vehicles": 20,
"max_dwell_time_s": 900
}'
```

### Open Dashboard

Visit: [http://localhost:8002/dashboard](http://localhost:8002/dashboard)

## Full Setup From Scratch

### Step 1: Install System Dependencies

**Terminal 1:**

```bash
sudo apt update
sudo apt install -y postgresql postgresql-contrib lsof curl mosquitto
sudo systemctl enable --now mosquitto postgresql
systemctl status mosquitto postgresql
```

### Step 2: Install TimescaleDB

```bash
echo "deb https://packagecloud.io/timescale/timescaledb/ubuntu/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/timescaledb.list
curl -fsSL https://packagecloud.io/timescale/timescaledb/gpgkey | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/timescaledb.gpg
sudo apt update
sudo apt install -y timescaledb-2-postgresql-16
sudo timescaledb-tune --quiet --yes
sudo systemctl restart postgresql
sudo -u postgres psql -c "SELECT extname, extversion FROM pg_extension WHERE extname = 'timescaledb';"
```

### Step 3: Create Database User

```bash
sudo -u postgres psql -c "CREATE ROLE apexedge LOGIN PASSWORD 'apexedge' CREATEDB SUPERUSER;"
sudo -u postgres psql -c "\du apexedge"
```

### Step 4: Install mediamtx

```bash
cd /tmp
wget -q https://github.com/bluenviron/mediamtx/releases/download/v1.9.3/mediamtx_v1.9.3_linux_amd64.tar.gz
tar xzf mediamtx_v1.9.3_linux_amd64.tar.gz
sudo mv mediamtx /usr/local/bin/
mediamtx --version
```

### Step 5: Create mediamtx Configuration

```bash
cat > /tmp/mediamtx.yml << 'EOF'
paths:
  stream:
    runOnDemand: ffmpeg -re -stream_loop -1 -i /home/admin1/Desktop/traffic_vehicle.mp4 -c copy -f rtsp rtsp://localhost:8554/stream
    runOnDemandStartTimeout: 10s
    runOnDemandCloseAfter: 10s
EOF

cat /tmp/mediamtx.yml
```

### Step 6: Start mediamtx

```bash
mediamtx /tmp/mediamtx.yml &
sleep 3
ps aux | grep mediamtx | grep -v grep
ffprobe -v error -print_format json -show_streams rtsp://localhost:8554/stream 2>&1 | head -15
```

**Keep Terminal 1 running.**

### Step 7: Prepare Python Environment

**Terminal 2:**

```bash
source /home/admin1/voyager-sdk/venv/bin/activate
cd /home/admin1/Desktop/vehicle/Vehicle_Detection
python -m pip install -r requirements-edge.txt
python -m pip install -r requirements-cloud.txt
axdevice
```

### Step 8: Start Application

```bash
export SDK_ROOT=/home/admin1/voyager-sdk
export VOYAGER_PYTHON="$SDK_ROOT/venv/bin/python"
export VOYAGER_ACTIVATE="$SDK_ROOT/venv/bin/activate"
export VOYAGER_NETWORK=yolov8s-coco
chmod +x start.sh
./start.sh
```

**Keep Terminal 2 running.**

### Step 9: Add Camera and Zone

**Terminal 3:**

```bash
cd /home/admin1/Desktop/vehicle/Vehicle_Detection

curl -X DELETE http://localhost:8002/api/cameras/demo_cam 2>/dev/null

curl -X POST http://localhost:8002/api/cameras \
 -H "Content-Type: application/json" \
 -d '{
"camera_id": "demo_cam",
"name": "Traffic RTSP",
"source_url": "rtsp://localhost:8554/stream",
"assigned_edge": "vehicle-edge-01"
}'

curl -X POST http://localhost:8002/api/zones \
 -H "Content-Type: application/json" \
 -d '{
"zone_id": "demo_zone",
"name": "Full Frame",
"camera_id": "demo_cam",
"zone_poly": [[0,0],[800,0],[800,500],[0,500]],
"max_vehicles": 20,
"max_dwell_time_s": 900
}'
```

### Step 10: Verify Pipeline

```bash
sleep 15
tail -30 /home/admin1/Desktop/vehicle/Vehicle_Detection/.logs/edge.log | grep -E "Voyager|iteration|pipeline|demo_cam|ready"
```

**Expected output:**

```
[Voyager] demo_cam stream started → rtsp://127.0.0.1:8554/stream
[Voyager] demo_cam pipeline ready (pipeline=0, network=yolov8s-coco)
```

### Step 11: Check Live Metrics

```bash
curl -s http://localhost:8002/api/zones/demo_zone/live | python3 -m json.tool
```

### Step 12: Open Dashboard

Visit: [http://localhost:8002/dashboard](http://localhost:8002/dashboard)

## Troubleshooting

### Annotated frames return 503

**Check mediamtx:**

```bash
ps aux | grep mediamtx | grep -v grep
```

**If not running:**

```bash
mediamtx /tmp/mediamtx.yml &
```

**Test RTSP stream:**

```bash
ffprobe -v error rtsp://localhost:8554/stream
```

**Check edge logs:**

```bash
tail -50 /home/admin1/Desktop/vehicle/Vehicle_Detection/.logs/edge.log | grep -E "error|exited|Voyager"
```

**Restart edge agent if needed:**

```bash
pkill -f "edge_agent.main"
sleep 2
source /home/admin1/voyager-sdk/venv/bin/activate
cd /home/admin1/Desktop/vehicle/Vehicle_Detection
export SDK_ROOT=/home/admin1/voyager-sdk
export VOYAGER_PYTHON="$SDK_ROOT/venv/bin/python"
export VOYAGER_ACTIVATE="$SDK_ROOT/venv/bin/activate"
export VOYAGER_NETWORK=yolov8s-coco
PYTHONPATH="$PWD:$SDK_ROOT" AXELERA_FRAMEWORK="$SDK_ROOT" VOYAGER_NETWORK="$VOYAGER_NETWORK" \
 "$VOYAGER_PYTHON" -m edge_agent.main > .logs/edge.log 2>&1 &
sleep 15
```

### No bounding boxes on video

**Check if Voyager iteration loop is running:**

```bash
tail -20 /home/admin1/Desktop/vehicle/Vehicle_Detection/.logs/edge.log | grep "iteration"
```

If "iteration loop exited", restart edge agent (see above).

**Check inference stats:**

```bash
curl -s http://localhost:8002/api/zones/demo_zone/live | grep -E "inf_fps|inf_ms"
```

### Redis connection error

```bash
redis-cli ping
# Should return: PONG
```

### Database connection error

```bash
systemctl status postgresql
sudo -u postgres psql -c "\du apexedge"
psql -U apexedge -h localhost -d vehicle_zone -c "SELECT 1;"
```

## Stop Everything

**Terminal 1:**

```bash
pkill mediamtx
```

**Terminal 2:**

```bash
pkill -f "uvicorn cloud.main"
pkill -f "edge_agent.main"
```

**Kill ports:**

```bash
lsof -ti:8002 | xargs kill -9 2>/dev/null
lsof -ti:8003 | xargs kill -9 2>/dev/null
lsof -ti:8554 | xargs kill -9 2>/dev/null
```

## Architecture

```
RTSP Camera/Video
    ↓
GStreamer (Intel iGPU decode)
    ↓
Axelera Metis AIPU (YOLOv8s inference)
    ↓
OC-SORT Tracking
    ↓
VehicleZoneAnalytics (1 Hz metrics)
    ↓
MQTT
    ↓
Cloud (FastAPI + Redis + TimescaleDB)
    ↓
WebSocket → Dashboard
```

## Ports

| Service       | Port |
| ------------- | ---- |
| Cloud API     | 8002 |
| Edge HTTP     | 8003 |
| MQTT Broker   | 1883 |
| PostgreSQL    | 5432 |
| Redis         | 6379 |
| mediamtx RTSP | 8554 |

## File Locations

| Path                                                    | Purpose             |
| ------------------------------------------------------- | ------------------- |
| `/home/admin1/Desktop/vehicle/Vehicle_Detection/`       | Project root        |
| `/home/admin1/voyager-sdk/`                             | Axelera Voyager SDK |
| `/home/admin1/Desktop/traffic_vehicle.mp4`              | Demo video          |
| `/tmp/mediamtx.yml`                                     | RTSP server config  |
| `/home/admin1/Desktop/vehicle/Vehicle_Detection/.logs/` | Logs                |

## API Endpoints

| Method | Path                          | Description      |
| ------ | ----------------------------- | ---------------- |
| GET    | `/health`                     | Health check     |
| GET    | `/api/cameras`                | List cameras     |
| POST   | `/api/cameras`                | Create camera    |
| DELETE | `/api/cameras/{id}`           | Delete camera    |
| GET    | `/api/cameras/{id}/annotated` | Annotated frame  |
| GET    | `/api/zones`                  | List zones       |
| POST   | `/api/zones`                  | Create zone      |
| GET    | `/api/zones/{id}/live`        | Live metrics     |
| GET    | `/api/zones/{id}/history`     | Time-series      |
| GET    | `/api/alerts`                 | Alert history    |
| GET    | `/api/overview`               | Dashboard data   |
| WS     | `/ws/dashboard`               | Real-time stream |

## Environment Variables

### Edge

- `MQTT_HOST`: localhost
- `CLOUD_API_URL`: http://localhost:8002
- `TARGET_FPS`: 8
- `INF_CONF`: 0.18
- `VOYAGER_NETWORK`: yolov8s-coco

### Cloud

- `VZI_DATABASE_URL_RAW`: postgresql://apexedge:apexedge@localhost/vehicle_zone
- `VZI_REDIS_URL`: redis://localhost:6379/1
- `VZI_MQTT_HOST`: localhost
