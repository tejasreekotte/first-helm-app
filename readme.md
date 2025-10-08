# 🚀 Deploy a Python Flask App with PostgreSQL using Helm

This project demonstrates how to deploy a **Python Flask web application** with a **PostgreSQL database** on **Kubernetes**, using **Helm**.

You’ll learn how to:

* Containerize a Flask app with Docker
* Deploy it with Helm
* Use Bitnami’s PostgreSQL chart as a dependency
* Connect Flask and PostgreSQL inside Kubernetes
* Expose your app externally with NodePort

---

## 🌿 Architecture

```
┌────────────────────────────────────────┐
│              Kubernetes Cluster        │
│                                        │
│  ┌──────────────┐      ┌────────────┐  │
│  │ Flask App    │ ---> │ PostgreSQL │  │
│  │ (Docker Img) │      │ (Bitnami)  │  │
│  └──────────────┘      └────────────┘  │
│          │ Environment Variables       │
│          └──> DB_HOST, DB_USER, etc.   │
└────────────────────────────────────────┘
```

---

## ⚙️ Prerequisites

* Kubernetes cluster (Minikube, Kind, or Cloud)
* Docker (logged into Docker Hub)
* Helm v3+
* kubectl configured for your cluster

---

## 🐍 Flask Application

### `app.py`

```python
import os
import psycopg2
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    try:
        conn = psycopg2.connect(
            host=os.environ.get("DB_HOST"),
            database=os.environ.get("DB_NAME"),
            user=os.environ.get("DB_USER"),
            password=os.environ.get("DB_PASSWORD")
        )
        cur = conn.cursor()
        cur.execute("SELECT version();")
        db_version = cur.fetchone()
        cur.close()
        conn.close()
        return f"✅ Connected to PostgreSQL successfully!<br>Version: {db_version}"
    except Exception as e:
        return f"❌ Database connection failed: {str(e)}"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### `requirements.txt`

```
flask
psycopg2-binary
```

---

## 🐳 Dockerfile

```Dockerfile
FROM python:3.10-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000
CMD ["python", "app.py"]
```

---

## 🧱 Build and Push Image

```bash
docker build -t tejukotte/helmapp:pg .
docker push tejukotte/helmapp:pg
```

---

## ⚓ Create Helm Chart

```bash
helm create flask-pg-chart
cd flask-pg-chart
rm -f templates/hpa.yaml templates/ingress.yaml templates/serviceaccount.yaml templates/tests/test-connection.yaml
```

---

## 🧩 Add PostgreSQL Dependency

`Chart.yaml`

```yaml
apiVersion: v2
name: flask-pg
description: Flask app with PostgreSQL dependency
version: 0.1.0
appVersion: "1.0"

dependencies:
  - name: postgresql
    version: 15.5.16
    repository: https://charts.bitnami.com/bitnami
```

Update dependencies:

```bash
helm dependency update
```

---

## 🧰 Configure Values

`values.yaml`

```yaml
replicaCount: 1

image:
  repository: tejukotte/helmapp
  tag: "pg"
  pullPolicy: IfNotPresent

service:
  type: NodePort
  port: 80

postgresql:
  auth:
    username: flaskuser
    password: flaskpass
    database: flaskdb
  image:
    tag: latest
```

---

## 🧾 Deployment Template

`templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 5000
          env:
            - name: DB_HOST
              value: "{{ .Release.Name }}-postgresql"
            - name: DB_NAME
              value: "{{ .Values.postgresql.auth.database }}"
            - name: DB_USER
              value: "{{ .Values.postgresql.auth.username }}"
            - name: DB_PASSWORD
              value: "{{ .Values.postgresql.auth.password }}"
```

---

## 🌐 Service Template

`templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Release.Name }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Release.Name }}
  ports:
    - protocol: TCP
      port: {{ .Values.service.port }}
      targetPort: 5000
      nodePort: 30080
```

---

## 🚀 Deploy to Kubernetes

```bash
helm install flaskpg .
```

Check:

```bash
kubectl get pods
kubectl get svc
```

---

## 🌍 Access the App

Find your node IP:

```bash
kubectl get nodes -o wide
```

Access using NodePort:

```bash
curl http://<NODE-IP>:30080
```

✅ Example output:

```
✅ Connected to PostgreSQL successfully!
Version: ('PostgreSQL 18.0 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 12.2.0, 64-bit',)
```

---

## 🧹 Cleanup

```bash
helm uninstall flaskpg
kubectl delete pvc -l app.kubernetes.io/instance=flaskpg
```

---

## 💡 Key Takeaways

* Helm can deploy **multi-component apps** easily
* Bitnami’s PostgreSQL chart handles DB creation and credentials
* Flask reads DB credentials from environment variables
* NodePort exposes your app externally
* All configuration is reusable via `values.yaml`

---

## 👤 Author

**Tejasree Kotte**
📧 `tkotte@onemindservices.com`
💼 Onemind Services LLC
