# Обнаружение сервисов (Service Discovery)

Добавление большого числа наблюдаемых узлов в динамической инфраструктур затруднительно - для предотвращения этого сделан механизм  автоматического определения наблюдаемых сервисов.

Из коробки Prometheus может находить множество сервисов таких как Consul, Amazon’s EC2,  и Kubernetes.
Можно также динамически изменять конфигурацию, на основе систем конфигурации таких как Ansible или Chef.

## Механизм обнаружения сервисов

Обнаружение сервисов - это не только составление списка машин. но и составление топологии машин и сервисов для из организации в жизненном цикле эксплуатации.
Хороший механизм обнаружения основан на метаданных сервисов. Метаданные позволяют наилучшим образом сконфигурировать узлы мониторинга.

### Статичное обнаружение

Основано на указании узлов в конфигурации

~~~ yaml
scrape_configs:
  - job_name: 'prometheus'
    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
~~~

В системах управления конфигурация, узлы могут формироваться по шаблону

~~~ yaml
scrape_configs:
  - job_name: 'node'
    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s
    static_configs:
      - targets:
{% for host in groups["all"] %}
        - {{ host }}:9100
{% endfor %}
~~~

### File

Обнаружение через файл работает при отсутствии сети. Файлы конфигурации находится в файловой системе.
Файлы должны быть формата JSON или YAML. Иметь стандартное расширение *.json* *.yaml* *.yml*.

~~~ json
[
    {
        "targets": [
            "host1:9100",
            "host2:9100"
        ],
        "labels": {
            "team": "infra",
            "job": "node"
        }
    },
    {
        "targets": [
            "host1:9090"
        ],
        "labels": {
            "team": "monitoring",
            "job": "prometheus"
        }
    }
]
~~~

В конфигурации Prometheus указывам директорию для сканирования:

~~~ yaml
scrape_configs:
 - job_name: file
   file_sd_configs:
    - files:
      - '*.json'
~~~

## Consul

## Перемаркировка (Relabelling)

Большинство метаданных сервисов поставляемых Service Discovery  содержит множество сырых данных. Чтобы сделать метки более пригодными существует отдельный процесс  -0 перемаркировка.

### Выборка метаданных для перемаркировки

Для модификации метаданных существуют специальные действия (action).

~~~ yaml
scrape_configs:
 - job_name: file
   file_sd_configs:
    - files:
       - '*.json'
   relabel_configs:
    - source_labels: [team]
      regex: infra
      action: keep
~~~

В данном примере - для  всех меток у которых источником является метки  перечисленные в массиве  [source_labels] (*team*), если они соотвтсвуют регулярному выражению (*infra*) - будут сохранены, если не соответствуют - отброшены.
Можно использовать множество правил перемаркировки

~~~ yaml
   relabel_configs:
    - source_labels: [team]
      regex: infra
      action: keep
    - source_labels: [team]
      regex: monitoring
      action: keep
~~~

В данном примере - все метки будут отброшены. поскольку они не соответствуют одновременно 2 правилам
Исправим это применив регулярное выражение по  *ИЛИ*

~~~ yaml
   relabel_configs:
    - source_labels: [team]
      regex: infra|monitoring
      action: keep
~~~

В качестве источников можно использовать несколько меток, разделя запятой, а регулярное выражение точка-запятой.

~~~ yaml
   relabel_configs:
    - source_labels: [job, team]
      regex: prometheus;monitoring
      action: drop
~~~

Prometheus использует механизм RE2 языка Go, что накладывает свои ограничения.

| Шаблон   | Описание                        |
|----------|:--------------------------------|
| a        | символ a                        |
| .        | любой символ                    |
| \\.      | одна точка                      |
| .*       | любое количество символов       |
| .+       | Как минимум 1 символ            |
| a+       | одна или более букв             |
| 0-9      | любое число из 1 цифры          |
| \d       | любое число из 1 цифры          |
| \d*      | любое  количество цифр          |
| [^0-9]   | 1 символ кроме цифр             |
| ab       | последовательность символов  ab |
| a(b\|c)* | ab или ac                       |


### Целевые метки (Target Labels)
Целевые метки - это метки добавлены в процессе снятия. В идеальном случае, они не должны мутировать исходные метки, чтобы не было разрыва временного ряда.
Должна быть сформирована топология меток - Регион - ЦОД  - зона

#### Замена (Replace)

~~~ yaml
relabel_configs:
- source_labels: [team]
    regex: monitoring
    replacement: monitor
    target_label: team
    action: replace
~~~
Производится замена метки  team="monitoring” на метку team="monitor”

~~~ yaml
relabel_configs:
- source_labels: [team]
    regex: '(.*)ing'
    replacement: '${1}'
    target_label: team
    action: replace
~~~
Производится замена метки  team="XXXing” на метку team="XXX - но уже по шаблону

#### метки: job, instance, and __address__

* Если целевая метка не имееет метки экземпляра  то она получает метку *\_\_address\_\_*
* instance и job - добавляется автоматически из конфигурации job
* \_\_address\_\_ - это имя и порт, откуда Prometheus снял состояние метрик

#### Labelmap
Labelmap - это действие, которое применяется к именам меток. Это необходимо если сервисы возвращают данные в формате ключ-значение, и необходимо их преобразовать в формат меток.

#### Словари (Lists)
Если сервисы не возвращают значения в виде ключ-значение, а например в виде конкатенированного списка значений
например ,dublin,prod, или ,prod,dublin

~~~  yaml 
relabel_configs:
- source_labels: [__meta_consul_tags]
  regex:  '.*,prod,.*'
  action: keep
~~~

Сохраняются только метрики с меткой prod

~~~  yaml 
relabel_configs:
- source_labels: [__meta_consul_tags]
  regex:  '.*,(prod|staging|dev),.*'
  target_label: env
~~~

Меняются метрики на env


## Как снимать метрики

~~~ yaml
scrape_configs:
 - job_name: example
   consul_sd_configs:
    - server: 'localhost:8500'
   scrape_timeout: 5s
   metrics_path: /admin/metrics
   params:
     foo: [bar]
   scheme: https
   tls_config:
     insecure_skip_verify: true
   basic_auth:
     username: brian
     password: hunter2
~~~

* metrics_path - путь с точке сбора метрик
* params  - переметры, передаваемые запросы
* scheme - может быть *https* или *http*. https - требует указания  tls конфигурации. *insecure_skip_verify* - отключает проверку валидности сертификата.
* basic_auth - параметры  базовой аутентификации.
* scrape_timeout - время для сбора метрик


### metric_relabel_configs
Есть 2 случая использования перемаркировки метрик

* отказ от дорогих показателй
* исправление плохих метрик

Изменение метрик - происходит до момента записи в хранилище, поэтому можно предотвратить ее запись через действие *drop* для метки *\_\_name\_\_*

~~~ yaml
scrape_configs:
 - job_name: prometheus
   static_configs:
    - targets:
       - localhost:9090
   metric_relabel_configs:
    - source_labels: [__name__]
      regex: http_request_size_bytes
      action: drop
~~~
Удаляются метрики с именем http_request_size_bytes

Можно отбросить отдельные бакеты для гистограмм


~~~ yaml
scrape_configs:
 - job_name: prometheus
   static_configs:
    - targets:
       - localhost:9090
   metric_relabel_configs:
    - source_labels: [__name__, le]
      regex: 'prometheus_tsdb_compaction_duration_seconds_bucket;(4|32|256)'
      action: drop
~~~
Удаляются  бакеты prometheus_tsdb_compaction_duration_seconds_bucket для 4 32 256 распределения.


### labeldrop и labelkeep
Это два действия для управления метками labeldrop и labelkeep.

Удаление меток по условию

~~~ yaml
metric_relabel_configs:
- regex: 'node_.*'
  action: labeldrop
~~~

## Коллизии меток и   honor_labels
При коллизии меток  (они совпали целевые и инструментальные - побеждают целевые). Чтобы в коллизии побеждали инструментальные  - необходимо выставить в конфигурации honor_labels = true.

#
# curr 148 151
  