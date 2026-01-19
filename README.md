# Prometheus-Grafana-Monitor

Architecture

Browser
   |
Grafana (Port 3000)
   |
Prometheus (Port 9090)
   |
Sample App (/metrics endpoint)


All services run using Docker Compose.

🪜 STEP 1: Create AWS EC2 Ubuntu Server
1.1 Launch EC2

Go to AWS Console → EC2 → Launch Instance

Name: monitoring-server

AMI: Ubuntu Server 22.04 LTS

Instance type: t2.micro (free tier)

Key pair: Create or use existing (download .pem)

Security Group → Allow ports:

SSH → 22

Grafana → 3000

Prometheus → 9090


1.2 Connect to EC2

From your local machine:

ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>

🪜 STEP 2: Install Docker & Docker Compose
2.1 Update system
sudo apt update && sudo apt upgrade -y

2.2 Install Docker
sudo apt install docker.io -y


Start Docker:

sudo systemctl start docker
sudo systemctl enable docker

2.3 Allow Docker without sudo
sudo usermod -aG docker ubuntu

exit
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>

2.4 Install Docker Compose
sudo apt install docker-compose -y


Verify:

docker --version
docker-compose --version

🪜 STEP 3: Create Project Structure
mkdir monitoring-stack
cd monitoring-stack


Create folders:

mkdir app prometheus

🪜 STEP 4: Create Sample App with /metrics

We’ll use Python + Prometheus client (very simple).

4.1 Create app code
cd app
nano app.py


Paste this:

from flask import Flask
from prometheus_client import Counter, generate_latest

app = Flask(__name__)

REQUEST_COUNT = Counter('app_requests_total', 'Total number of requests')

@app.route("/")
def home():
    REQUEST_COUNT.inc()
    return "Hello from Monitoring App!"

@app.route("/metrics")
def metrics():
    return generate_latest()

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)

4.2 Create requirements file
nano requirements.txt

flask
prometheus_client

4.3 Create Dockerfile
nano Dockerfile

FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py .

CMD ["python", "app.py"]

🪜 STEP 5: Configure Prometheus
5.1 Create Prometheus config
cd ../prometheus
nano prometheus.yml

global:
  scrape_interval: 5s

scrape_configs:
  - job_name: "sample-app"
    static_configs:
      - targets: ["app:5000"]

🪜 STEP 6: Create Docker Compose File

Go back to main folder:

cd ..
nano docker-compose.yml


Paste:

version: "3.8"

services:
  app:
    build: ./app
    container_name: sample_app
    ports:
      - "5000:5000"

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3000:3000"

🪜 STEP 7: Start the Monitoring Stack
docker-compose up -d


Check running containers:

docker ps

🪜 STEP 8: Verify Everything Works
8.1 Sample App

Open browser:

http://<EC2_PUBLIC_IP>:5000


Metrics:

http://<EC2_PUBLIC_IP>:5000/metrics

8.2 Prometheus
http://<EC2_PUBLIC_IP>:9090


Go to Status → Targets

You should see sample-app → UP

8.3 Grafana
http://<EC2_PUBLIC_IP>:3000


Login:

Username: admin

Password: admin
(Change password)

🪜 STEP 9: Add Prometheus to Grafana

Grafana → Settings ⚙ → Data Sources

Add data source → Prometheus

URL:

http://prometheus:9090


Click Save & Test

🪜 STEP 10: Create Grafana Dashboard

Click + → Dashboard

Add New Panel

Query:

app_requests_total


Visualization: Time series

Click Apply

 You now see live metrics!

STEP 11: Generate Traffic

Open browser multiple times:

http://<EC2_PUBLIC_IP>:5000
