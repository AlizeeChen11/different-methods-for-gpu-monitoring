# Different methods for configuring GPU monitoring system
The document walks through different metheods to configure GPU monitoring system. 

### Azure monitor + Telegraf + Log analytics workspace
One is leveraging Telegraf to collect GPU counters and send the data to Azure log analytics workspace, and use Azure monitoring to vitualize the metrics. 
InfluxData Telegraf is an open source agent and not officially supported by Azure Monitor. For issues with the Telegraf connector, refer to the Telegraf GitHub page in References part.

For details, please refer:[Use Azure Monitor](https://github.com/AlizeeChen11/different-methods-for-gpu-monitoring/blob/main/AzureMonitor/README.md)

### Grafana + Prometheus + DCGM exporter
The other method is leveraging DCGM exporter to collect GPU counters and use Prometheus DB as date source for metrics vitualization in Grafana.

For details, please refer:[Use Grafana and Prometheus](https://github.com/AlizeeChen11/different-methods-for-gpu-monitoring/blob/main/Grafana/README.md)


### References
- [Linux telegraf collect custom metrics](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/collect-custom-metrics-linux-telegraf?tabs=ubuntu)
- [Use telegraf with Azure_Monitor](https://github.com/influxdata/telegraf/blob/4b2e2c5263bb8bd030d2ae101438810c1af61945/plugins/outputs/azure_monitor/README.md)
- [Collect nvidia_smi Counters using Telegraf ](https://github.com/influxdata/telegraf/blob/master/plugins/inputs/nvidia_smi/README.md)
