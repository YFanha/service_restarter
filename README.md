# Service Restarter

This repository contains a Flask-based webhook server that listens for Prometheus Alertmanager alerts and restarts specified services on remote servers based on the alert data. This script was tested on _Ubuntu 22.04_.

## Features
- **Automated Service Restart:** Automatically restart services on remote servers when alerts are triggered by Prometheus Alertmanager.
- **Configuration:** Alerts that trigger restarts are configurable via a YAML file.

## Prerequisites
- Python 3.x
- Flask
- PyYAML
- Prometheus and Alertmanager
- SSH access to remote servers, with SSH key pair.

## Installation

### 1. **Download the script**
```bash
sudo curl https://raw.githubusercontent.com/YFanha/service_restarter/main/service_restarter -O /usr/bin/service_restarter
sudo chown restarter:restarter /usr/bin/service_restarter
sudo chmod +x /usr/bin/service_restarter
```

### 2. **Install dependencies:**
```bash
sudo apt update
sudo apt install python3 -y
pip install flask pyyaml
```

### 3. **Set up logging directory:**
Ensure the log directory exists and is writable:
```bash
sudo touch /var/log/service_restarter.log
sudo chmod 666 /var/log/service_restarter.log
```

### 4. **Create the users**
#### 4.1 On server
```bash
sudo useradd -m restarter
sudo -u restarter ssh-keygen -q -t ed25519 -N '' -f /home/restarter/.ssh/id_ed5519
sudo cat /home/restarter/.ssh/id_ed25519.pub
```
_Copy the public key_
#### 4.2 On managed server (client)
```bash
sudo useradd -m restarter 
sudo usermod -aG sudo restarter 
echo "restarter ALL=(ALL) NOPASSWD: ALL" | sudo tee -a /etc/sudoers
echo "<SSH_PUB_KEY>" | sudo tee -a /home/restarter/.ssh/authorized_keys
```

### 4. **Create alerts list :**
Create the `alertnames.yml` configuration file at `/etc/service_restarter/alertnames.yml` with the alerts that should trigger a service restart :
_An example can be found [here](https://raw.githubusercontent.com/YFanha/service_restarter/main/alertnames.yml.example)_
```bash
sudo mkdir -p /etc/service_restarter
sudo nano /etc/service_restarter/alertnames.yml
```
```yaml
alertnames:
    - YourAlertName1
    - YourAlertName2
```

### 5. **Create the service_restart.service file**
```bash
cat <<EOM | sudo tee -a /etc/systemd/system/service_restarter.service
[Unit]
Description=Service Restarter for Prometheus Alerts
After=network.target

[Service] 
User=restarter
ExecStart=/usr/bin/service_restarter
Restart=always

[Install]
WantedBy=multi-user.target
EOM
```

**Start and enable the service**
```bash
sudo systemctl enable service_restarter.service
sudo systemctl start service_restarter.service
sudo systemctl status service_restarter.service
```

### 6. **Configure the webhook in alertmanager.yml**
Configure Prometheus Alertmanager to send alerts to the service endpoint:
```yaml
route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 1m
  repeat_interval: 1m
  receiver: 'service-restarter'

receivers:
  - name: 'service-restarter'
    webhook_configs:
      - url: 'http://localhost:5001/'  # Calling our own script
        send_resolved: true
```

### *Example of alert rules and alertnames.yml matching*
---
> The label **"alert"** has to match with a name in the _alertnames.yml_ file to send the command to restart.
The label **"service"** has to match with the exact name of the service running on the remote host (monitored service).
> - **alert = MySQLDown**
> - **service = mysql**
```bash
groups:
  - name: mysql_alerts
    rules:
      - alert: MySQLDown # Alertname
        expr: mysql_up == 0
        for: 1m
        labels:
          severity: critical
          service: mysql # Service to be restarted on remote server
        annotations:
          summary: "MySQL service is down"
          description: "The MySQL service on instance {{ $labels.instance }}
```
_rules.alerts.yml_

```bash
alertnames:
  - MySQLDown # Alertname 
```
_/etc/service_restarter/alertnames.yml_


