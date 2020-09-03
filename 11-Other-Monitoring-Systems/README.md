# Интеграция с другими системами мониторинга

В идеальном мире для Prometheus метрики поставляются или экспортерами  или приложениями. В некоторых случаях метрики приходится получать из других систем мониторинга.

Метрики можно получать из:
* InfluxDB через influxdb_exporter
* collectd через collectd_exporter
* Graphite через graphite_exporter
* SNMP через snmp_exporter
* CloudWatch  exporter
* NewRelicexporter
* Pingdom exporter

## InfluxDB
InfluxDB  экспортер может принимать метрики начиная с версии InfluxDB 0.9.0. Протокол поддерживает Http  и TCP 


## StatsD
InfluxDB  экспортер может принимать метрики начиная с версии InfluxDB 0.9.0. Протокол поддерживает Http  и TCP 