# Configure GPU Monitoring for Azure VM
This document walkthrough the steps on how to configure the Telegraf agent to collect metrics on Azure N series VM to send metrics to Azure Monitor.
InfluxData Telegraf is an open source agent and not officially supported by Azure Monitor. For issues with the Telegraf connector, refer to the Telegraf GitHub page in References part.

## Prerequisites
A Azure subscription with GPU VM quota
Proper GPU driver installed

## For Windows VM
### Install Telegraf
Download and install telegraf for windows:
```
wget https://dl.influxdata.com/telegraf/releases/telegraf-1.34.0_windows_amd64.zip -UseBasicParsing -OutFile telegraf-1.34.0_windows_amd64.zip
Expand-Archive .\telegraf-1.34.0_windows_amd64.zip -DestinationPath 'C:\Program Files\InfluxData\telegraf'
```

Generate a configuration file for Telegraf with CPU and Memory input and Azure Monitor output
```
cd C:\Program Files\InfluxData\telegraf\telegraf-1.34.0_windows_amd64
.\telegraf.exe --input-filter cpu:mem --output-filter azure_monitor config > azm-telegraf.conf

mkdir "C:\Program Files\Telegraf"
copy ./* "C:\Program Files\Telegraf"
cd "C:\Program Files\Telegraf"
```

Configure Telegraf to run nvidia-smi:
Add the following section at the end of file "C:\Program Files\Telegraf\azm-telegraf.conf":
```
[[inputs.nvidia_smi]]
## Optional: path to nvidia-smi binary, defaults "/usr/bin/nvidia-smi"
bin_path = "C:\\Windows\\System32\\nvidia-smi.exe"

## Optional: timeout for GPU polling
# timeout = "5s"
'@
```
Save the file.

Test the configuration:
```
telegraf.exe --config "C:\Program Files\Telegraf\telegraf.conf" --test --quiet
```
Install Telegraf as a Windows service:
```
telegraf.exe --config "C:\Program Files\Telegraf\telegraf.conf" --service install
net start telegraf
```
Note: telegraf.conf must be in path "C:\Program Files\Telegraf", otherwise the service will fail to start if it's in a different folder.

Once the telegraf service is in running status, we can proceed to configure in Azure portal.

### Configuration in Azure portal
1. Enable managed identities for Azure VM. Telegraf will automatically authenticate using this method when running on Azure VMs.
2. Make sure susbcription has registered for Microsoft insights resource provider.

In "Monitoring | Metrics", you will see telegraf listed in drop down in Metric Namespace:
![image](https://github.com/user-attachments/assets/dac75326-cfbd-4f20-a87f-62c8a4ab17ac)

You can choose the counter you would like to monitor:
![image](https://github.com/user-attachments/assets/9a6aef7b-89c5-479b-b7f0-47f9cf17f5b7)

## For Linux VM
### Install and configure Telegraf
Installation:
```
# For Ubuntu, Debian
curl -s https://repos.influxdata.com/influxdb.key | sudo apt-key add -
source /etc/lsb-release
sudo echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
sudo curl -fsSL https://repos.influxdata.com/influxdata-archive_compat.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg add

sudo apt-get update
sudo apt-get install telegraf

# For RHEL, Oracle Linux
cat <<EOF | sudo tee /etc/yum.repos.d/influxdb.repo
[influxdb]
name = InfluxDB Repository - RHEL $releasever
baseurl = https://repos.influxdata.com/rhel/$releasever/$basearch/stable
enabled = 1
gpgcheck = 1
gpgkey = https://repos.influxdata.com/influxdata-archive_compat.key
EOF

sudo yum -y install telegraf
```
Configuration:
Create a custom configuration file and have the agent use it by running the following commands:
```
# generate the new Telegraf config file in the current directory
telegraf --input-filter cpu:mem --output-filter azure_monitor config > azm-telegraf.conf

# replace the example config with the new generated config
sudo cp azm-telegraf.conf /etc/telegraf/telegraf.conf
```
Add below to the end of the configuration file /etc/telegraf/telegraf.conf:
```
# Pulls statistics from nvidia GPUs attached to the host
[[inputs.nvidia_smi]]
  ## Optional: path to nvidia-smi binary, defaults "/usr/bin/nvidia-smi"
  ## We will first try to locate the nvidia-smi binary with the explicitly specified value (or default value),
  ## if it is not found, we will try to locate it on PATH(exec.LookPath), if it is still not found, an error will be returned
   bin_path = "/usr/bin/nvidia-smi"

  ## Optional: timeout for GPU polling
   timeout = "5s
```
Start and enable the telegraf agent on the VM to ensure it picks up the latest configuration:
```
sudo systemctl enable --now telegraf
systemctl status telegraf
```
### Configuration in Azure portal
Enable managed identities for Azure VM. Telegraf will automatically authenticate using this method when running on Azure VMs.

In "Monitoring | Metrics", you will see telegraf listed in drop down in Metric Namespace:
![image](https://github.com/user-attachments/assets/f40104e8-c1e9-47d0-9a7e-81cccf4a0dd4)

### Basic GPU test
Running basic GPU test to monitor GPU utilization:
```
git clone https://github.com/vinilvadakkepurakkal/basic-gpu-test.git
cd basic-gpu-test
python3 gputest.py
```
Check Azure portal, GPU metrics:
![image](https://github.com/user-attachments/assets/6c7cdb14-7c20-4630-8016-b965049fd483)


**References**
- https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/collect-custom-metrics-linux-telegraf?tabs=ubuntu
- https://github.com/influxdata/telegraf/blob/4b2e2c5263bb8bd030d2ae101438810c1af61945/plugins/outputs/azure_monitor/README.md
- https://github.com/influxdata/telegraf/blob/master/plugins/inputs/nvidia_smi/README.md
