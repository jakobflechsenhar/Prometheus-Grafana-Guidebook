# ------------------------------------------------------ #
# ReadMe: Prometheus + Grafana Guidebook
# ------------------------------------------------------ #

----------------------------------------------------------
## Overview: ##
- This lab sets up a complete monitoring stack using Docker containers.
- Prerequisites - Docker installed and running (or have Orbstack running in background).
----------------------------------------------------------

----------------------------------------------------------
## The Stack: ##
- Node Exporter:
    - An "exporter" that collects raw system metrics from your host (e.g. CPU, memory, disk, network)
    - Exposes these metrics in Prometheus format at the :9100/metrics endpoint
    - It's not part of Prometheus - it's a separate data collector
    - Node Exporter metrics: http://localhost:9100/metrics
- Prometheus:
    - The actual time-series database that stores metrics, queryable via PromQL
    - Actively "scrapes" (pulls) metrics from exporters every 15 seconds
    - Prometheus UI: http://localhost:9090
- Grafana:
    - The visualization layer - it doesn't store any data
    - Queries Prometheus to get the data it needs in order to create visual dashboards
    - Can connect to many data sources (not just Prometheus)
    - Grafana dashboards: http://localhost:3000
----------------------------------------------------------

----------------------------------------------------------
## The Rundown: ##
#### List all remaining resources: ####
```bash
docker ps -a
Docker network ls
```

##### Create a new network for your containers to communicate in: #####
```bash
docker network create monitoring-network
```

#### Create two volumes for persistent data storage: ####
```bash
docker volume create prometheus-volume
docker volume create grafana-volume
```

#### Create and run the node-exporter container: ####
```bash
# delete all in-line comments before running this command (leaving no spaces at end of lines)
docker run -d  \ # create and start a new container
  --name node-exporter-container \ # name the container node-exporter
  --network monitoring-network \ # network in which the container is placed
  -p 9100:9100 \ # maps port 9100 from container to port 9100 on your computer/localhost
  prom/node-exporter # base image
```

#### Check if its running: ####
```bash
docker ps
```

#### Verify that the node-exporter container is actually serving metrics: ####
```bash
curl http://node-exporter-container:9100/metrics
curl http://localhost:9100/metrics
```

#### Create the prometheus config file: ####
```bash
cat > prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
scrape_configs:
- job_name: 'node'
  static_configs:
  - targets: ['node-exporter-container:9100']
EOF
```

#### Create and run the prometheus container: ####
```bash
docker run -d \
--name prometheus-container \
--network monitoring-network \
-p 9090:9090 \
-v $(pwd):/etc/prometheus/ \
-v prometheus-data:/prometheus \
prom/prometheus
# so this command takes the base image (“prom/prometheus”), modifies it using your own configuration file which is mounted as a volume from your local computer (“$(pwd)” is the source on your Mac, while “/etc/prometheus” is the destination inside the container), and it also creates a persistent storage for metrics data (“prometheus-data” is the source, a Docker-managed volume, while “/prometheus” is the destination inside the container), and then  runs it inside a new container called prometheus-container
```

#### Verify: ####
Open your browser at http://localhost:9090 and enter the “up” query in Prometheus to verify the Node Exporter is being scraped

#### Create and run the grafana container: ####
```bash
docker run -d \
  --name grafana-container \
  --network monitoring-network \
  -p 3000:3000 \
  -v grafana-volume:/var/lib/grafana \
  grafana/grafana-oss
  ```

#### Access the Grafana UI: #### 
- Open your browser and navigate to http://localhost:3000
- Default Credentials: Username and password are both “admin”
- Change Password

#### Add Prometheus as a Data Source: ####
- In the Grafana UI, click the Connections and select Data Sources
- Click Add data source and select Prometheus
- URL: http://prometheus-container:9090 (since Grafana is on the same network as Prometheus, you can reference it by name instead of just localhost)
- Click Save & Test to ensure Grafana successfully communicates with Prometheus

#### Build some shit: #### 
- Create Your First Dashboard
- Click the "+" icon in the left sidebar
- Select "Dashboard"
- Click "Add visualization"
- Choose Prometheus as the data source
- Choose the “code” toggle if using queries, not the “builder toggle”
- In the query editor: rate(node_cpu_seconds_total[5m])
- Panel type: Time series (should be default)
- Title: "CPU Usage by Mode"
- Click “Run Queries“ to save the panel

#### Cleanup: ####
```bash
docker stop node-exporter prometheus grafana node-exporter-container prometheus-container grafana-container
docker rm node-exporter prometheus grafana node-exporter-container prometheus-container grafana-container
docker network rm monitoring monitoring-network
docker volume rm prometheus-data grafana-storage prometheus-volume grafana-volume
```
----------------------------------------------------------
