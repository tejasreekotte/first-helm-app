---

# 🐍 Flask + PostgreSQL CRUD App (Helm Deployment)

This guide redeploys everything we built previously — from **Flask code to a running Kubernetes app** using **Helm** and **PostgreSQL**.

---

## 🧩 Step 1 — Recreate Your Project Structure

Inside your Kubernetes control plane or local Docker environment:

```bash
mkdir flask-crud-demo && cd flask-crud-demo
mkdir flaskapp
```

**Folder layout:**

```
flask-crud-demo/
 ├── flaskapp/
 │    ├── app.py
 │    ├── requirements.txt
 │    └── Dockerfile
```

---

## 🧱 Step 2 — Create Flask CRUD App Files

### 🐍 `flaskapp/app.py`

```python
import os
import psycopg2
from flask import Flask, jsonify, request

app = Flask(__name__)

# Database configuration
DB_CONFIG = {
    "host": os.environ.get("DB_HOST"),
    "database": os.environ.get("DB_NAME"),
    "user": os.environ.get("DB_USER"),
    "password": os.environ.get("DB_PASSWORD"),
}

# Utility: Connect to DB
def get_db_connection():
    conn = psycopg2.connect(**DB_CONFIG)
    return conn

# Initialize table
def init_db():
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS notes (
            id SERIAL PRIMARY KEY,
            title TEXT NOT NULL,
            content TEXT
        )
    """)
    conn.commit()
    cur.close()
    conn.close()

init_db()

# Routes
@app.route('/')
def home():
    return jsonify({"message": "Welcome to Flask Notes API", "routes": ["/notes", "/notes/<id>"]})

@app.route('/notes', methods=['GET'])
def get_notes():
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("SELECT id, title, content FROM notes ORDER BY id;")
    notes = [{"id": r[0], "title": r[1], "content": r[2]} for r in cur.fetchall()]
    cur.close()
    conn.close()
    return jsonify(notes)

@app.route('/notes/<int:note_id>', methods=['GET'])
def get_note(note_id):
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("SELECT id, title, content FROM notes WHERE id = %s;", (note_id,))
    note = cur.fetchone()
    cur.close()
    conn.close()
    if note:
        return jsonify({"id": note[0], "title": note[1], "content": note[2]})
    return jsonify({"error": "Note not found"}), 404

@app.route('/notes', methods=['POST'])
def create_note():
    data = request.get_json()
    title = data.get('title')
    content = data.get('content', '')
    if not title:
        return jsonify({"error": "Title is required"}), 400
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("INSERT INTO notes (title, content) VALUES (%s, %s) RETURNING id;", (title, content))
    note_id = cur.fetchone()[0]
    conn.commit()
    cur.close()
    conn.close()
    return jsonify({"id": note_id, "title": title, "content": content}), 201

@app.route('/notes/<int:note_id>', methods=['DELETE'])
def delete_note(note_id):
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("DELETE FROM notes WHERE id = %s RETURNING id;", (note_id,))
    deleted = cur.fetchone()
    conn.commit()
    cur.close()
    conn.close()
    if deleted:
        return jsonify({"message": f"Note {note_id} deleted"}), 200
    return jsonify({"error": "Note not found"}), 404

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

### 📦 `flaskapp/requirements.txt`

```
flask
psycopg2-binary
```

---

### 🐳 `flaskapp/Dockerfile`

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

## 🧱 Step 3 — Build and Push Docker Image

```bash
# Login to DockerHub
docker login -u <your-dockerhub-username>

# Build and push
cd flaskapp
docker build -t <username>/helmapp:crud .
docker push <username>/helmapp:crud

# Confirm
docker images | grep helmapp
```

---

## ⚓ Step 4 — Create Helm Chart

```bash
cd ..
helm create flask-pg-chart
cd flask-pg-chart
```

Remove unnecessary files:

```bash
rm -f templates/hpa.yaml templates/ingress.yaml templates/serviceaccount.yaml templates/tests/test-connection.yaml
```

---

## 🧩 Step 5 — Add PostgreSQL Dependency

**Chart.yaml**

```yaml
apiVersion: v2
name: flask-pg
description: Flask CRUD app with PostgreSQL
version: 0.1.0
appVersion: "1.0"

dependencies:
  - name: postgresql
    version: 15.5.16
    repository: https://charts.bitnami.com/bitnami
```

Update and confirm dependencies:

```bash
helm dependency update
ls charts/
# You should see postgresql-15.5.16.tgz
```

---

## 🧰 Step 6 — Configure `values.yaml`

```yaml
replicaCount: 1

image:
  repository: <username>/helmapp
  tag: "crud"
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

## 🧾 Step 7 — Deployment Template

**templates/deployment.yaml**

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

## 🌐 Step 8 — Service Template

**templates/service.yaml**

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

## 🚀 Step 9 — Deploy Everything

```bash
helm install flaskpg .
kubectl get pods
kubectl get svc
```

**Expected output:**

```
flaskpg-xxxxxx             1/1   Running   0   1m
flaskpg-postgresql-0       1/1   Running   0   1m
```

---

## 🌍 Step 10 — Access the App

```bash
# Get Node IP
kubectl get nodes -o wide

# Get NodePort
kubectl get svc
# Look for flaskpg → 80:30080/TCP
```

**Test the API:**

```bash
# Get all notes
curl http://<NODE-IP>:30080/notes

# Create a note
curl -X POST http://<NODE-IP>:30080/notes \
     -H "Content-Type: application/json" \
     -d '{"title":"Helm Demo","content":"Flask + PostgreSQL"}'

# Get a note by ID
curl http://<NODE-IP>:30080/notes/1

# Delete a note
curl -X DELETE http://<NODE-IP>:30080/notes/1
```

✅ You’ll receive JSON responses directly from your Flask app inside Kubernetes.

---

## 🧹 Step 11 — Cleanup (Optional)

```bash
helm uninstall flaskpg
kubectl delete pvc -l app.kubernetes.io/instance=flaskpg
```

---

## 🎯 Summary

You now have a **fully working Flask + PostgreSQL CRUD app** deployed via **Helm**:

* Build & push your Docker image anytime
* Deploy effortlessly with Helm
* Flask ↔ PostgreSQL connection auto-configured

---
