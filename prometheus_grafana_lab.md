# 📊 Prometheus & Grafana Lab

> Monitor Docker containers and a Python app using Prometheus and Grafana on AWS EC2.

---

## 🖥️ EC2 Instances Required

| Machine | Ports to Open |
|---|---|
| Docker Machine | 22, 9323 |
| Prometheus Machine | 22, 9090 |
| Grafana Machine | 22, 3000 |
| Python App Machine | 22, 8000 |

---

## 🐳 STEP 1: Setup Docker Machine

```bash
sudo su
apt update -y
apt install docker.io -y
systemctl start docker
systemctl enable docker

docker run -dit --name c01 ubuntu
docker run -dit --name c02 ubuntu
```

---

## 📊 STEP 2: Enable Docker Metrics

```bash
vi /etc/docker/daemon.json
```

Add this content:

```json
{
  "metrics-addr": "0.0.0.0:9323",
  "experimental": true
}
```

```bash
sudo systemctl daemon-reexec
sudo systemctl restart docker
```
🔍 VERIFY (VERY IMPORTANT)
✔ Local test (inside Docker EC2):
```
curl http://localhost:9323/metrics
```

> ✅ Verify in browser: `http://<docker-public-ip>:9323/metrics`

---

## 📡 STEP 3: Install Prometheus

```bash
sudo su
wget https://github.com/prometheus/prometheus/releases/download/v2.53.2/prometheus-2.53.2.linux-amd64.tar.gz
tar -zxvf prometheus-2.53.2.linux-amd64.tar.gz
cd prometheus-2.53.2.linux-amd64
```

---

## 📝 STEP 4: Configure prometheus.yml

```bash
nano prometheus.yml
```

Add this config (replace IPs with your private IPs):

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["<prometheus-private-ip>:9090"]

  - job_name: "docker"
    static_configs:
      - targets: ["<docker-private-ip>:9323"]
    metrics_path: /metrics
    scheme: http

  - job_name: "python-app"
    static_configs:
      - targets: ["<app-private-ip>:8000"]
```


## ▶️ STEP 5  : Run Prometheus

```bash
./prometheus
```

> ✅ Open in browser: `http://<prometheus-public-ip>:9090` → Status → Targets

---

## 🐍 STEP 6: Setup Python App Machine
# DO NOT use sudo su (stay as ubuntu user)

```
sudo apt update
sudo apt install python3-venv python3-pip -y
python3 -m venv myenv
source myenv/bin/activate
pip install --upgrade pip
pip install prometheus_client
```
---

## 📝 STEP 7: Create app.py

```bash
nano app.py
```

Paste this code:

```python
from prometheus_client import start_http_server, Counter, Gauge
import random
import time

REQUESTS = Counter('http_requests_total', 'Total HTTP Requests')
TEMPERATURE = Gauge('room_temperature', 'Simulated Room Temperature')

if __name__ == '__main__':
    start_http_server(8000)
    while True:
        REQUESTS.inc()
        TEMPERATURE.set(random.uniform(20, 30))
        time.sleep(2)
```

> Save: `CTRL + X` → `Y` → `Enter`

---

## ▶️ STEP 8: Run Python App

```bash
python app.py
```

> ✅ Verify in browser: `http://<app-public-ip>:8000/metrics`

> TEST NETWORK CONNECTIVITY

👉 From Prometheus EC2:
```
curl http://<DOCKER_PRIVATE_IP>:9323/metrics
curl http://<PYTHON_PRIVATE_IP>:8000/metrics
```
---

## 🔁 STEP 9: Verify Targets in Prometheus

> Open: `http://<prometheus-public-ip>:9090/targets`

All three should show **UP**:
- `docker` → UP
- `prometheus` → UP
- `python-app` → UP

---

## 📊 STEP 10: Install Grafana

```bash
sudo su
wget https://dl.grafana.com/enterprise/release/grafana-enterprise-11.1.4.linux-amd64.tar.gz
tar -zxvf grafana-enterprise-11.1.4.linux-amd64.tar.gz
cd grafana-v11.1.4
./bin/grafana-server
```

> ✅ Open: `http://<grafana-public-ip>:3000` — Login: `admin / admin`

---

## 🔗 STEP 11: Connect Prometheus to Grafana

1. Go to: **Connections → Data Sources → Add Prometheus**
2. Set URL:

```
http://<prometheus-private-ip>:9090
```

3. Click **Save & Test** ✔

---

## 📈 STEP 12: Create Dashboard Panels

Go to: **Dashboards → New → Add Visualization → Select Prometheus → Code mode**

**Panel 1 — Request Rate:**

```
rate(http_requests_total[1m])
```

**Panel 2 — Temperature:**

```
room_temperature
```

> Click **Apply** on each panel → Save dashboard as `Python Monitoring`

---

## 🎉 Final Output

| Panel | Metric |
|---|---|
| 📈 Request Rate | `rate(http_requests_total[1m])` |
| 🌡️ Temperature | `room_temperature` |
