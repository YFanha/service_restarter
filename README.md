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

### 1. **Clone the repository:**
```bash
git clone https://github.com/yourusername/service-restarter.git
cd service-restarter
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
sudo mkdir -p /var/log/service_restarter
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

### 4. **Create alert configuration file:**
Create the `alertnames.yml` configuration file at `/etc/service_restarter/alertnames.yml` with the alerts that should trigger a serice restart :
```yaml
alertnames:
    - YourAlertName1
    - YourAlertName2
```

5. **Create the service_restart.service file**
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

### 6. **Configure the webhook in alertmanager.yml**
Configure Prometheus Alertmanager to send alerts to the service endpoint:
```yaml
route:
    receiver: 'service-restarter'

receivers:
- name: 'service-restarter'
    webhook_configs:
    - url: 'http://your-server-ip:5001/'
```
