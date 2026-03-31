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
apt-get update -y
apt install docker.io -y
service docker start
docker run -dit --name c01 ubuntu
docker run -dit --name c02 ubuntu
docker ps -a
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
service docker restart
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
nano /opt/prometheus/prometheus.yml
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

```bash
sudo systemctl restart prometheus
```

---

## ▶️ STEP 5: Run Prometheus

```bash
cd /opt/prometheus
./prometheus
```

> ✅ Open in browser: `http://<prometheus-public-ip>:9090` → Status → Targets

---

## 🐍 STEP 6: Setup Python App Machine

```bash
sudo su
apt update
apt install python3-pip -y
python3 -m venv myenv
source myenv/bin/activate
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
python3 app.py
```

> ✅ Verify in browser: `http://<app-public-ip>:8000/metrics`

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
cd grafana-11.1.4
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


=========================================================================================================================================================

# ☸️ Kubernetes (EKS) Voting App Project

> Deploy a full-stack voting app on AWS EKS with MongoDB, API, and Frontend services.

---

## 🖥️ STEP 1: Install kubectl

```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.24.11/2023-03-17/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo cp ./kubectl /usr/local/bin
export PATH=/usr/local/bin:$PATH
```

---

## ☁️ STEP 2: Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

---

## 🔧 STEP 3: Install Git

```bash
yum install git -y
```

---

## 🔐 STEP 4: Create IAM Roles (AWS Console)

Go to: **AWS → IAM → Roles → Create Role**

| Role Name | Use Case | Policy |
|---|---|---|
| `eks-cluster-role` | EKS → Cluster | Default EKS policy |
| `eks-node-role` | EC2 → Node | Node policies |
| `eks-management-role` | EC2 → Management | Management policies |

---

## ☸️ STEP 5: Create EKS Cluster (AWS Console)

Go to: **EKS → Create Cluster → Custom Configuration**

```
Name                   : cluster-1
Cluster IAM Role       : eks-cluster-role
Auth Mode              : API and CONFIG
Cluster Endpoint Access: Public
Security Group         : default
Add-on                 : EBS CSI Driver  ✅ enable this
```

> ⏳ Wait 5–15 minutes until status shows **Active**

---

## 🖧 STEP 6: Add Node Group (AWS Console)

Go to: **EKS → cluster-1 → Compute → Add Node Group**

```
Node IAM Role : eks-node-role
Instance Type : m7i-flex.large
```

> Click Next → Next → Create. Wait until status shows **Active**.  
> Two new EC2 instances will appear — rename them to `work-node`.

---

## 🔑 STEP 7: Add IAM Access Entry (AWS Console)

Go to: **EKS → cluster-1 → Access → IAM Access Entries → Create**

```
Select : eks-management-role
Click  : Next → Select Policy → Add Policy → confirm
Click  : Next → Next → Create
```

---

## 🔌 STEP 8: Connect kubectl to EKS Cluster

```bash
aws eks update-kubeconfig --name cluster-1 --region ap-south-1
kubectl get nodes
```

---

## 📥 STEP 9: Clone the Voting App

```bash
git clone https://github.com/N4si/K8s-voting-app.git
```

---

## 📁 STEP 10: Create Namespace & Navigate

```bash
kubectl create ns cloudchamp
kubectl config set-context --current --namespace cloudchamp
ls
cd K8s-voting-app
cd manifests
ls
```

---

## 🍃 STEP 11: Deploy MongoDB

```bash
kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo-statefulset.yaml
kubectl apply -f mongo-service.yaml
kubectl get pods
```

> ⏳ Wait until all mongo pods show **Running** before continuing.

---

## 🔁 STEP 12: Initialize MongoDB Replica Set

```bash
cat << EOF | kubectl exec -it mongo-0 -- mongo
rs.initiate();
sleep(2000);
rs.add("mongo-1.mongo:27017");
sleep(2000);
rs.add("mongo-2.mongo:27017");
sleep(2000);
cfg = rs.conf();
cfg.members[0].host = "mongo-0.mongo:27017";
rs.reconfig(cfg, {force: true});
sleep(5000);
EOF
```

> You will see `bye` at the end ✅

---

## 🌱 STEP 13: Seed the Database

```bash
cat << EOF | kubectl exec -it mongo-0 -- mongo
use langdb;
db.languages.insert({"name":"csharp","codedetail":{"usecase":"system, web, server-side","rank":5,"compiled":false,"homepage":"https://dotnet.microsoft.com/learn/csharp","download":"https://dotnet.microsoft.com/download/","votes":0}});
db.languages.insert({"name":"python","codedetail":{"usecase":"system, web, server-side","rank":3,"script":false,"homepage":"https://www.python.org/","download":"https://www.python.org/downloads/","votes":0}});
db.languages.insert({"name":"javascript","codedetail":{"usecase":"web, client-side","rank":7,"script":false,"homepage":"https://en.wikipedia.org/wiki/JavaScript","download":"n/a","votes":0}});
db.languages.insert({"name":"go","codedetail":{"usecase":"system, web, server-side","rank":12,"compiled":true,"homepage":"https://golang.org","download":"https://golang.org/dl/","votes":0}});
db.languages.insert({"name":"java","codedetail":{"usecase":"system, web, server-side","rank":1,"compiled":true,"homepage":"https://www.java.com/en/","download":"https://www.java.com/en/download/","votes":0}});
db.languages.insert({"name":"nodejs","codedetail":{"usecase":"system, web, server-side","rank":20,"script":false,"homepage":"https://nodejs.org/en/","download":"https://nodejs.org/en/download/","votes":0}});
db.languages.find().pretty();
EOF
```

> You will see `bye` at the end ✅

---

## 🚀 STEP 14: Deploy API

```bash
kubectl apply -f mongo-secret.yaml
kubectl apply -f api-service.yaml
kubectl apply -f api-deployment.yaml
```

---

## ✏️ STEP 15: Edit api-service.yaml

```bash
nano api-service.yaml
```

> Inside the file change `selector: app:` → `selector: role:`

```bash
kubectl apply -f api-service.yaml
```

---

## ✏️ STEP 16: Edit frontend-service.yaml

```bash
nano frontend-service.yaml
```

> Inside the file change `selector: app:` → `selector: role:`

```bash
kubectl apply -f frontend-service.yaml
kubectl get svc
```

> 📋 Copy the **EXTERNAL-IP** of the `api` service — you need it in the next step.

---

## 🌐 STEP 17: Configure & Deploy Frontend

```bash
nano frontend-deployment.yaml
```

> Paste the API external IP you copied into the correct field, then:

```bash
kubectl apply -f frontend-deployment.yaml
```

---

## 🔄 STEP 18: Restart Frontend & Verify

```bash
kubectl rollout restart deployment frontend -n cloudchamp
kubectl get svc
```

> Copy the **EXTERNAL-IP** of the `frontend` service and open in browser:

```
http://<frontend-external-ip>
```

---

## 🎉 Final Output

The voting app is live! You should see a page to vote for programming languages:

| Language | Rank |
|---|---|
| Java | 1 |
| Python | 3 |
| C# | 5 |
| JavaScript | 7 |
| Go | 12 |
| Node.js | 20 |




==============================================================================================================================

1. Install tools
sudo apt update -y
sudo apt install -y curl unzip  

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

curl -LO https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/ 

curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
________________________________________
✅ 2. Configure AWS
aws configure
•	Region: ap-south-1 
________________________________________
✅ 3. Create EKS Cluster
eksctl create cluster \
--name game-cluster \
--region ap-south-1 \
--node-type t3.medium \
--nodes 3
________________________________________
✅ 4. Verify Nodes
kubectl get nodes
________________________________________
✅ 5. Create Deployment
nano deployment.yaml 

apiVersion: apps/v1
kind: Deployment
metadata:
  name: game-2048
spec:
  replicas: 2
  selector:
    matchLabels:
      app: game
  template:
    metadata:
      labels:
        app: game
    spec:
      containers:
      - name: game
        image: public.ecr.aws/l6m2t8p7/docker-2048:latest
        ports:
        - containerPort: 80
________________________________________
✅ 6. Create Service
nano service.yaml

apiVersion: v1
kind: Service
metadata:
  name: game-service
spec:
  type: LoadBalancer
  selector:
    app: game
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
________________________________________
✅ 7. Deploy Application
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
________________________________________
✅ 8. Check Pods
kubectl get pods
________________________________________
✅ 9. Get External URL
kubectl get svc
________________________________________
✅ 10. Open in Browser
http://<EXTERNAL-IP>
________________________________________
🎮 OUTPUT
👉 2048 game will open ✅

