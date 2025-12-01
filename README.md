# 📈 Homelab Monitoring Stack: Prometheus, Grafana, and Node Exporter

This document outlines the setup and configuration of the resource monitoring stack deployed on the Ubuntu server using Docker Compose. The stack uses Prometheus for time-series data collection, Node Exporter for gathering host system metrics, and Grafana for visualization.

## 🚀 Setup & Deployment

All configuration files (`.env`, `docker-compose.yml`, `prometheus.yml`) are located in the `/docker/monitoring_stack/` directory.

### Step 1: Directory Structure

Ensure the following structure exists:
```text
/docker/monitoring_stack/
  ├── .env # Environment variables and secrets
  ├── docker-compose.yml # Defines services, ports, and volumes
  └── prometheus.yml # Prometheus scraping configuration
```
---

### Step 2: Configure the .env File

The `.env` file centralizes configuration and secrets. **The passwords MUST be changed.**

```text
# .env

# --- Grafana Configuration ---
# CHANGE THIS TO A STRONG, SECURE PASSWORD
GF_ADMIN_PASSWORD=your_secure_grafana_password

# --- Port Configuration ---
# The ports exposed to the host system
PROMETHEUS_PORT=9090
GRAFANA_PORT=3000
NODE_EXPORTER_PORT=9100
```
|Variable          |Service      |Purpose                                                       |
|------------------|-------------|--------------------------------------------------------------|
|GF_ADMIN_PASSWORD |Grafana      |Sets the initial password for the default admin user.         |
|PROMETHEUS_PORT   |Prometheus   |Host port for accessing the Prometheus UI (Default: 9090).    |
|GRAFANA_PORT      |Grafana      |Host port for accessing the Grafana UI (Default: 3000).       |
|NODE_EXPORTER_PORT|Node Exporter|Host port for accessing Node Exporter metrics (Default: 9100).|

### Step 3: Define Services in docker-compose.yml

The `docker-compose.yml` file uses the variables from `.env` and specifies the necessary host permissions for the Node Exporter.

Key Configurations:
  - Networking: All services share the `monitoring` bridge network.
  - Node Exporter: Uses `pid: host` and mounts host directories (`/proc`, `/sys`, `/`) to read system metrics.
  - Prometheus: Mounts the local `prometheus.yml` configuration and uses a persistent volume for data.
  - Grafana: Uses the `GF_ADMIN_PASSWORD` from the `.env` file.

Important Fix: We removed the deprecated flags (`--web.console.html` and `--web.console.libraries`) from the Prometheus command that caused container restarts.

```yaml
# docker-compose.yml

volumes:
  prometheus_data: {}
  grafana_data: {}

networks:
  monitoring:
    driver: bridge

services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/host/rootfs'
    ports:
      - "${NODE_EXPORTER_PORT}:9100" 
    networks:
      - monitoring
    restart: unless-stopped
    pid: host 

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    ports:
      - "${PROMETHEUS_PORT}:9090"
    networks:
      - monitoring
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "${GRAFANA_PORT}:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin 
      - GF_SECURITY_ADMIN_PASSWORD=${GF_ADMIN_PASSWORD} 
    networks:
      - monitoring
    restart: unless-stopped
    depends_on:
      - prometheus
```
### Step 4: Configure Prometheus Targets (`prometheus.yml`)

This file instructs Prometheus to scrape metrics from itself and the Node Exporter service running on the internal Docker network.

```yaml
# prometheus.yml

global:
  scrape_interval: 15s 

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100'] # Uses the internal service name
```

### Step 5: Deployment and Access
1. Deploy the stack:
    ```bash
    cd /docker/monitoring_stack/
    docker compose up -d
    ```
2. Verify container status:
    ```bash
    docker compose ps
    ```
  (Ensure all containers are in the Up state.)
3. Access UIs (using your server's IP):
    - Prometheus: `http://<Your-Server-IP>:<PROMETHEUS_PORT>`
    - Grafana: `http://<Your-Server-IP>:<GRAFANA_PORT>`

## 📊 Post-Deployment Grafana Setup

To visualize your server metrics, you must configure Prometheus as a data source and import a dashboard in Grafana.

1. Log in to Grafana (`admin` / `your_secure_grafana_password`)
2. Add Data Source:
    - Go to Configuration → Data Sources.
    - Select Prometheus.
    - Set the URL to: `http://prometheus:9090` (using the internal service name).
    - Click Save & Test.
3. Import Node Exporter Dashboard:
    - Go to Dashboards → Import.
    - Use the ID `1860` (Node Exporter Full dashboard).
    - Select your newly created Prometheus data source.



