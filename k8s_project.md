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
