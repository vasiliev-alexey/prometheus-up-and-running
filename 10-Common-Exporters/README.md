# Известные экспортеры данных (Common Exporters)

## Consul

Создаем [DockerCompose](consul/docker-compose.yaml)

Важные метрики:

* consul_up - состояние сервера Consul
* consul_catalog_service_node_healthy - предоставляет метрики о состоянии сервисов Consul
* consul_serf_lan_members - предоставляет метрики о числе агентов Consul
* consul_exporter_build_info - информация о Consul Exporter
* process_и go_ - информация о среде выполнения GO и ее процессах

Скрейп конфиг

~~~ yaml
global:
  scrape_interval: 10s
scrape_configs:
 - job_name: consul
   static_configs:
    - targets:
      - localhost:9107
~~~

## Grok Exporter

Grok - парсит неструктурированные логи и превращает их в метрики на основании правил.

~~~ yaml
global:
  config_version: 2 # Версия конфига
input:
  type: file # тип слежения
  path: /opt/example/examples.log # директория для логов
  readall: true  # считывать ли весь файл или tail
grok:
  additional_patterns: # шаблоны парсинга логов
   - 'METHOD [A-Z]+'
   - 'PATH [^ ]+'
   - 'NUMBER [0-9.]+'
metrics:  # получаемые метрики
 - type: counter
   name: log_http_requests_total
   help: HTTP requests
   match: '%{METHOD} %{PATH:path} %{NUMBER:latency}'
   labels:
     path: '{{.path}}'
 - type: histogram
   name: log_http_request_latency_seconds_total
   help: HTTP request latency
   match: '%{METHOD} %{PATH:path} %{NUMBER:latency}'
   value: '{{.latency}}'
server:
   port: 9144 #  порт отдачи метрик
~~~

## Blackbox

В случае если средства  формирования и отдачи метрик  встроить не удается, или разместить Exporter рядом - используется механизм мониторинга по типу "черный ящик" (Blackbox).
Blackbox опрашивает веб-точку и формирует на основании  опроса метрики, понятные Prometheus.
Blackbox берет на себя формирование метрик, периодичность опроса , по аналогии с конфигурацией Prometheus.
Blackbox exporter позволяет снимать метрики со следующих протоколов ICMP, TCP, HTTP и DNS.

### ICMP

ICMP - часть протокола IP. В контексте Blackbox - работает на основе echo  запроса и echo   ответа - аналог команда ping.

Для старта проверки можно использовать web  доступ
[/probe?module=icmp&target=prometheus](http://192.168.1.50:9115/probe?module=icmp&target=prometheus)

котрый будет генерировать метрики для указного  в параметре хоста

~~~ yaml
# HELP probe_success Displays whether or not the probe was a success
# TYPE probe_success gauge
probe_success 1
~~~

probe_success = 1  говорит об успешности проверки, 0 о провале.

для принудительного переключения на протокол IP4V - необходимо в конфиге выставить значение

~~~ yaml
icmp_ipv4:
  prober: icmp
  icmp:
    preferred_ip_protocol: ip4
~~~

### TCP

TCP проверет по различным портам и протоколам. таких как HTTP, SMTP, Telnet, SSH, IRC и выполняет различне пробы на основании текстовых данных.

~~~ yaml
# пробы через TCP
  tcp_connect:
    prober: tcp
~~~

Проверка через SSH, проверяется что приветствие от протокола будет начинаться на  SSH-2.0-

~~~ yaml
# пробы через SSH
  ssh_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^SSH-2.0-"
~~~

Можно также осуществлять проверки через TLS

~~~ yaml
    tcp:
      query_response:
      - expect: "^+OK"
      tls: true
      tls_config:
        insecure_skip_verify: false
~~~

### HTTP

HTTP - проверяет доступность сервисов, работающих по одноименному протоколу

~~~ http
http://localhost:9115/probe?module=http_2xx&target=https://www.robustperception.io
~~~

probe_success - означает что сервис доступен, также можно увидеть иные атрибуты, версию HTTP,
тайминги и прочее

~~~ yaml
# HELP probe_dns_lookup_time_seconds Returns the time taken for probe dns lookup in seconds
# TYPE probe_dns_lookup_time_seconds gauge
probe_dns_lookup_time_seconds 0.004275723
# HELP probe_duration_seconds Returns how long the probe took to complete in seconds
# TYPE probe_duration_seconds gauge
probe_duration_seconds 0.519700631
# HELP probe_failed_due_to_regex Indicates if probe failed due to regex
# TYPE probe_failed_due_to_regex gauge
probe_failed_due_to_regex 0
# HELP probe_http_content_length Length of http content response
# TYPE probe_http_content_length gauge
probe_http_content_length -1
# HELP probe_http_duration_seconds Duration of http request by phase, summed over all redirects
# TYPE probe_http_duration_seconds gauge
probe_http_duration_seconds{phase="connect"} 0.056799502
probe_http_duration_seconds{phase="processing"} 0.223834359
probe_http_duration_seconds{phase="resolve"} 0.004275723
probe_http_duration_seconds{phase="tls"} 0.174330132
probe_http_duration_seconds{phase="transfer"} 0.116654322
# HELP probe_http_redirects The number of redirects
# TYPE probe_http_redirects gauge
probe_http_redirects 0
# HELP probe_http_ssl Indicates if SSL was used for the final redirect
# TYPE probe_http_ssl gauge
probe_http_ssl 1
# HELP probe_http_status_code Response HTTP status code
# TYPE probe_http_status_code gauge
probe_http_status_code 200
# HELP probe_http_uncompressed_body_length Length of uncompressed response body
# TYPE probe_http_uncompressed_body_length gauge
probe_http_uncompressed_body_length 58106
# HELP probe_http_version Returns the version of HTTP of the probe response
# TYPE probe_http_version gauge
probe_http_version 1.1
# HELP probe_ip_protocol Specifies whether probe ip protocol is IP4 or IP6
# TYPE probe_ip_protocol gauge
probe_ip_protocol 4
# HELP probe_ssl_earliest_cert_expiry Returns earliest SSL cert expiry in unixtime
# TYPE probe_ssl_earliest_cert_expiry gauge
probe_ssl_earliest_cert_expiry 1.605988154e+09
# HELP probe_success Displays whether or not the probe was a success
# TYPE probe_success gauge
probe_success 1
# HELP probe_tls_version_info Contains the TLS version used
# TYPE probe_tls_version_info gauge
probe_tls_version_info{version="TLS 1.2"} 1
~~~

Можно указывать  загловки аутнетификации, TLS  и прочее.

~~~ yaml
  http_200_ssl_prometheus:
    prober: http
    http:
      valid_status_codes: [200]
      fail_if_not_ssl: true
      fail_if_not_matches_regexp:
        - Prometheus
~~~

HTTP  - самый настраиваемы протокол

### DNS

DNS probber предназначен для снятия метрик доступности DNS серверов

~~~ yaml
dns_tcp:
  prober: dns
  dns:
    transport_protocol: "tcp"
    query_name: "www.prometheus.io
~~~

проверяем доступность 8.8.8.8 (аналог dig -tcp @8.8.8.8 www.prometheus.io)

~~~ http
http://localhost:9115/probe?module=dns_tcp&target=8.8.8.8
~~~

Провека доступности записи MX

~~~ yaml
dns_mx_present_rp_io:
  prober: dns
  dns:
    query_name: "robustperception.io"
    query_type: "MX"
    validate_answer_rrs:
      fail_if_not_matches_regexp:
        - ".+"

~~~

### конфигурация  Prometheus

Blackbox exporter отдает метрики по пути /probe.
Для того чтобы автоматически оперделять существует Service discovery, а используя *__param_<name>* можно давать метрикам человекопонятные имена.

~~~ yaml
scrape_configs:
 - job_name: blackbox
   metrics_path: /probe
   params:
     module: [http_2xx]
   static_configs:
    - targets:
       - http://www.prometheus.io
       - http://www.robustperception.io
       - http://demo.robustperception.io
   relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: 127.0.0.1:9115
~~~
