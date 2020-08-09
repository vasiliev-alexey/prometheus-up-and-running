# Оповещение *(Alerting)*

Сам Prometheus не спроектирован для рассылки оповещений, для этой цели предназначен отдельный инструмент - Alertmanager.
На основании правил Prometheus формирует уведомлений, и посылает его Alertmanager, который может собирать их со множества Prometheus серверов.

## Правила оповещения

Правила оповещений настраиваются  в конфигурационном файле
~~~ yaml
groups: 
    - name: node_rules   
      rules:    
        - record: job:up:avg      
          expr: avg without(instance)(up{job="node"})    
        - alert: ManyInstancesDown      
          expr: job:up:avg{job="node"} < 0.5
~~~

Язык указания  метрик богат и позволяет задавать сложные конструкции, например диапазоны в рамках которых должны срабатывать правила

~~~ yaml
- alert: ManyInstancesDown
  expr: >    
     (   avg without(instance)(up{job="node"}) < 0.5      
            and   on()        
                    hour() > 9 < 17    )
~~~

~~~ yaml
- alert: BatchJobNoRecentSuccess
  expr: >
      time() - my_batch_job_last_success_time_seconds{job="batch"} > 86400*2
~~~


##for

Мониторинг основанный на метриках подвержен ситуации гонок (таймауты, недоставка пакетов, задержка в определении правил)
~~~ yaml
groups:
- name: node_rules
  rules:
    - record: job:up:avg
        expr: avg without(instance)(up{job="node"})
    - alert: ManyInstancesDown
        expr: avg without(instance)(up{job="node"}) < 0.5
        for: 5m
~~~


Prometheus не имеет представлений о верхних  и нижних границах метрик для оповещения, они назначаются в конфигурационном файле

Рекомендуется использовать скользящее среднее в диапазонов от 5 до 20 минут, для того чтобы убрать шумы от ложных и пустых срабатываний. 

Поскольку for может сбросить состояние, рекомендуется использовать функции скользящего среднего через временной диапазон avg_over_time или  max_over_time
~~~ yaml
- alert: FDsNearLimit
  expr:
      (        max_over_time(process_open_fds[5m])      >        max_over_time(process_max_fds[5m]) * 0.9    )
  for: 5m
~~~
метрика UP -  это исключение, и в ней не надо использовать функции скользящего среднего.

## Аварийные метки
Как и в случае с правилами записи. можно указать и метки для оповещения, но это не самая распространенная практика. В этом случае в алерт добавляется метрика
~~~ yaml
groups:
- name: node_rules
  rules:
    - record: job:up:avg
      expr: avg without(instance)(up{job="node"})
    - alert: ManyInstancesDown
      expr: avg without(instance)(up{job="node"}) < 0.5
      for: 2m
      labels: #тут лейбл
        severity: page #тут лейбл
- name: example
  rules:
   - alert: InstanceDown
     expr: up == 0
     for: 1m
     labels: #тут лейбл
       severity: ticket    #тут лейбл  
~~~        
Метка серьезности неи имеет особого смысла - она предназначена для обработки Alertmanager
Важно чтобы метки не мутировали  между срезами, иначе это будет большой спам.

Prometheus не позволяет определять несколько порогов срабатывания, но это можно обойти путем создания правил с разными уровнями серьезности.