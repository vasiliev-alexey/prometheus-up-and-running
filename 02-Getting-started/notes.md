# Начало работы с Prometheus

Установка - [скачиваем](https://prometheus.io/download/) архив по соответствующей платформе 

Меняем конфигурацию на свою prometheus.yml

    global:
    scrape_interval: 10s
    scrape_configs:
    - job_name: prometheus
    static_configs:
        - targets:
        - localhost:9090


 По умолчанию Prometheus открыт на порту [http://localhost:9090/](http://localhost:9090/)       

 Метрики самого Prometheus открыты по урлу [http://localhost:9090/metrics](http://localhost:9090/metrics)


 ## Использование Expression Browser

 Expression Browser используется для запуска перенастроенных запросов формата PromQL и просмотра данных из хранилища

 Метрики 
 gauge - показывают текущее значение
 counter - показывает сколько событий случилось, они всегда работают на повышение

 Функция rate применяемая к counter показывает как менялась метрика по отношению к временному ряду, например

    rate(prometheus_tsdb_head_samples_appended_total[1m])
 

## Запускаем Node Exporter
Node Exporter - открывает для prometheus метрики операционной системы  Unix и Linux

    global:
    scrape_interval: 10s
    scrape_configs:
    - job_name: prometheus
        static_configs:
        - targets:
        - localhost:9090
    - job_name: node
        static_configs:
        - targets:
        - localhost:9100     

## Alerting
Предупреждение состоит из 2 частей:
* правила выявления
* нотификации


Правила описываются в специальных файлах вида [rules.yml](02-Getting-started/rules.yml)

    groups:
    - name: example
    rules:
        - alert: InstanceDown
        expr: up == 0
        for: 1m

Нотификации и их способы настраиваются  в файлах вида [alertmanager.yml](02-Getting-started/alertmanager.yml)

    global:
    smtp_smarthost: 'localhost:25'
    smtp_from: 'youraddress@example.org'
    route:
    receiver: example-email
    receivers:
    - name: example-email
        email_configs:
        - to: 'youraddress@example.org'