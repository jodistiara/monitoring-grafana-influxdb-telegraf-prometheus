# Server-Monitoring

This service contains easily deployable monitoring solution which uses:
 - Prometheus (monitoring solution pulling metrics from exporter)
 - Node Exporter for Prometheus (metrics exporter-exposer for Prometheus)
 - Telegraf (collect log from Prometheus and store it to InfluxDB)
 - alertmanager (alerting)


## Deployment: 

1. Pull code
2. Fill in Env File
   - .env
   - .env.influxdb2-admin-username
   - .env.influxdb2-admin-password
   - .env.influxdb2-admin-token
   
3. Up docker
   ```
   sudo chown -R 1000:1000 ./prometheus/data/
   docker compose up -d
   ```

4. Test
   1. Prometheus: http://localhost:9090
   2. AlertManager: http://localhost:9993


## Agent Install


### Agent install on each VM (Ubuntu)
1. Create compose file
   ```
   echo "
   version: '3'
   services:
       pve-agent:
           image: quay.io/prometheus/node-exporter:latest
           container_name: server-monitor-agent
           ports:
               - 9221:9100
           restart: always
    " > /home/${USER}/docker-compose-agent.yml
   ```
2. Run Docker
   ```
   docker compose -f docker-compose-agent.yml up -d --build
   ```
3. Run service if docker is stopped
   ```
   docker start server-monitor-agent
   ```

### Agent install on each Proxmox Node

#### 1. User creation for each Proxmox Cluster
1. Create a Proxmox VE API User
   First, log into your Proxmox shell and create a user that Prometheus will use to access the API.
   Create the API user and set up a dummy password. We'll be creating a token instead
   ```
   pveum user add pve-exporter@pve -password "password"
   ```

2. Assign API permissions:
   This command gives the user read-only permissions, which is sufficient for exporting metrics.
   ```
   pveum acl modify / -user pve-exporter@pve -role PVEAuditor
   ```

3. Create a Linux User for Exporter
   Next, we'll create a local Linux user without login privileges, specifically for running the exporter service. This step creates a local Linux user called pve-exporter to run the exporter service on your Proxmox VE machine.
   ```
   useradd -s /bin/false pve-exporter
   ```
   The -s /bin/false option ensures that this user cannot log in interactively.

4. Create API token
   Create the token through GUI or Shell
   ```
   pveum user token add pve-exporter@pve exporter --comment "Token for monitoring" --privsep 0
   ```
   
5. Store token value somewhere safe

#### 2. Create Configuration Files for each Proxmox Node(s)
1. Log in to selected nodes
2. Create prometheus config file
   ```
   mkdir -p /etc/pve_exporter
   echo "
   default:
       user: pve-exporter@pve
       token_name: exporter   # Token name created in the first step
       token_value: xxx # Secret Token created in the first step
       verify_ssl: false 
   " | tee /etc/pve_exporter/config.yaml >> /dev/null
   ```

#### 3. Install and Run Agent for each Proxmox Node(s)
1. Install pip
   ```
   apt update 
   apt install -y python3-venv
   ```
2. Create virtual env and install Proxmox VE Exporter using pip
   ```
   python3 -m venv /opt/prometheus-pve-exporter
   source /opt/prometheus-pve-exporter/bin/activate
   pip install prometheus-pve-exporter
   deactivate
   ```

3. Register background process to Systemd
   ```

   echo "
   [Unit]
   Description=PVE Exporter
   Wants=network-online.target
   After=network-online.target

   [Service]
   Type=simple
   User=pve-exporter
   EnvironmentFile=/etc/default/pve_exporter
   ExecStart=/opt/prometheus-pve-exporter/bin/pve_exporter --config.file=/etc/pve_exporter/config.yaml --web.listen-address=<LOCAL IP ADDRESS>:9221

   [Install]
   WantedBy=multi-user.target
   " | tee /etc/systemd/system/pve_exporter.service >> /dev/null

   systemctl daemon-reload
   systemctl enable --now pve_exporter.service
   ```
   Replace `<LOCAL IP ADDRESS>` with node IP address
   Run `systemctl status pve_exporter.service` to check status

### Check Agent installation

Access Prometheus Agent endpoint from your browser: `http://<SERVER LOCAL IP>:9221`
* VM: `http://<SERVER LOCAL IP>:9221/metrics`
* Node: `http://<SERVER LOCAL IP>:9221/pve`
