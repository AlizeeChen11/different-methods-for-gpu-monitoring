# Configure GPU Monitoring with Azure Monitor On Azure GPU VM
This document walkthrough the steps on how to configure the Telegraf agent to collect metrics on Azure N series VM with Azure Monitor.

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

## For Linux




**Reference**
- https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/collect-custom-metrics-linux-telegraf?tabs=ubuntu
- https://github.com/influxdata/telegraf/blob/4b2e2c5263bb8bd030d2ae101438810c1af61945/plugins/outputs/azure_monitor/README.md
