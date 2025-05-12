# GPU monitor using Grafana


## Detailed steps
### Deploy an Azure VM to host Grafana and Prometheus:
```
az vm create -n MyVm -g MyResourceGroup --image Canonical:0001-com-ubuntu-server-jammy:22_04-lts-gen2:latest -l LOCATION --size Standard_D8_v5 --generate-ssh-keys --public-ip-sku standard --security-type Standard
```

### Install Grafana
Run below commands to install Grafana:

```
 sudo apt update
 sudo apt-get install -y software-properties-common
 sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
 sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 963FA27710458545
 sudo apt-get update
 sudo apt update
 sudo apt install grafana -y
 sudo systemctl start grafana-server
 sudo systemctl enable grafana-server
```
- Add inbound rule for port 3000.
- Then you will be able to open the Grafana web portal from http://publicip:3000
- Initial admin account: admin
- pw:admin
- Donload and Import dashboard: https://grafana.com/api/dashboards/12239/revisions/2/download

### Install Prometheus:
Run below commands to install Prometheus
```
sudo apt update
sudo apt upgrade -y
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
wget https://github.com/prometheus/prometheus/releases/download/v3.3.1/prometheus-3.3.1.linux-amd64.tar.gz
tar -xvzf prometheus-3.3.1.linux-amd64.tar.gz
cd prometheus-3.3.1.linux-amd64
sudo mv prometheus /usr/local/bin/
sudo mv promtool /usr/local/bin/
sudo mv prometheus.yml /etc/prometheus
sudo chown -R prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
sudo chown -R prometheus:prometheus /etc/prometheus
```
Prometheus uses configuration file (prometheus.yml) to define scrape configurations, alerting, and other settings. Add below configuration:
```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'gpu-dcgm-exporter'
    static_configs:
      - targets: ['10.0.0.5:9400']
```
Add private ip for the target GPU nodes for gpu-dcgm-exporter.

Create a Systemd Service for Prometheus
```
sudo cat << EOF > /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
After=network.target

[Service]
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/var/lib/prometheus/ --web.listen-address=0.0.0.0:9090

[Install]
WantedBy=multi-user.target
EOF
```
Enabel and start Prometheus service:
```
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
sudo systemctl status prometheus
```
Add inbound rule for port 9090. Then you will be able to open the Prometheus Web UI from http://pubip:9090

### Run DCGM exporter on GPU nodes
Prometheus only collects data from DCGM exporter. Therefore, each GPU node needs to have DCGM exporter deployed.
```
sudo docker run -d --gpus all --cap-add SYS_ADMIN --rm -p 9400:9400 nvcr.io/nvidia/k8s/dcgm-exporter:4.2.3-4.1.1-ubuntu22.04
curl localhost:9400/metrics
```

Run GPU workload:
```
import torch
import time

# Check GPU
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print(f"Running on device: {device}")

# Create large tensors (adjust size as needed for higher/lower GPU usage)
size = 8192
a = torch.randn(size, size, device=device)
b = torch.randn(size, size, device=device)

# Warm up
torch.matmul(a, b)
torch.cuda.synchronize()

# Run workload for ~10 minutes
duration_sec = 10 * 60  # 10 minutes
start_time = time.time()
iterations = 0

print(f"Starting workload for {duration_sec} seconds...")

while time.time() - start_time < duration_sec:
    c = torch.matmul(a, b)
    torch.cuda.synchronize()  # Ensure the GPU finishes before next iteration
    iterations += 1
    if iterations % 10 == 0:
        print(f"{iterations} iterations completed...")

print(f"Workload finished. Total iterations: {iterations}")
```
Then you will be able to see the metrics
![image](https://github.com/user-attachments/assets/73876d7a-409c-43c9-acb9-8ad55da1f24e)







