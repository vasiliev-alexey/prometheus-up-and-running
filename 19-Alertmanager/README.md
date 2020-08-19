# Менеджер оповещений (Alertmanager)

Задача  менеджера оповещений собирать предупреждения от всех Prometheus серверов  и проводить оповещения из единого центра.  

## Конвейер нотификации

Конвейер нотификации  группирует оповещения, чтобы вы получили одно уведомление вместо пачки, также он активно работает с метками предупреждений - это ключ к его эффективной работе.

* подавление - останавливает рассылку менее значительный уведомлений, если произошло более крупное (например авария)
* глушение - позволяет игнорировать отдельные оповещения
* маршрутизация - построение цепочки доставки оповещений для разных команд и сред
* группировка - позволяет схлопывать оповещения в группы и подучать одно  
* пропуск и повторение - уведомления могут быть пропущены, если сработало  родительское, а также будут повторены. 
* уведомление -формирование оповещений по шаблонам

## Конфигурационные файлы

Alertmanager  как и остальные утилиты конфигурируется через YAML  файл, часто именуемый alertmanager.yml.  
конфигурация перезагружается при получении сигнала SIGHUP или пост запроса на /-/reload.  
Для проверки конфигурации есть утилита

~~~
amtool check-config
~~~

~~~ yaml
global:
  smtp_smarthost: 'localhost:25'
  smtp_from: 'youraddress@example.org'
  
  route:
    receiver:
     example-email

  receivers:
   - name: example-email
     email_configs:
     - to: 'youraddress@example.org'
~~~

В конфиге должен быть хотябы 1 маршрут и 1 получатель.

### Дерево маршрутов
Группа настроек маршрутов  может быть настроена в секциях

* на верхнем уровне
* в секции fallback
* в секции default
  
~~~ yaml
route: # маршрут
  receiver: fallback-pager  # получатели в случае отсутствия правила назначения
  routes: # Назначение маршрутов для групп получателей
    - match:
        severity: page
      receiver: team-pager
    - match:
        severity: ticket
      receiver: team-ticket # получатель при срабатывании правила
~~~

Когда срабатывает предупреждении в секции маршрутов  ищется первый маршрут, подошедший под критерии правил.
Имеется правило match_re, которое обрабатывает по шаблону регулярного выражения.

~~~ yaml
route: # маршрут
  receiver: fallback-pager # получатели в случае отсутствия правила назначения
  routes: # Назначение маршрутов для групп получателей
    - match:
        severity: page
      receiver: team-pager # получатель при срабатывании правила
    - match_re: # сравнение по регулярному выражению
        severity: (ticket|issue|email)
      receiver: team-ticket
~~~

Для правильной маршрутизации между командами, рекомендуется использовать внешние метки,  специфичные для каждой команды.

Параметр *continue*  используется для поиска  иных маршрутов, кроме первого  сработавшего.

~~~ yaml
route:
  receiver: fallback-pager
  routes:
  # Log all alerts.
  - receiver: log-alerts
    continue: true
  # Frontend team.
  - match:
      team: frontend
  receiver: frontend-pager
~~~

### Группировка

Alertmanager  по умолчанию группирует все уведомления в группу, чтобы получатели получили 1 большое сообшение, а не кучу мелких.
Поле group_by позволяет задать ключевые метки по которым необходимо производить схлопывание.

~~~ yaml
route:
  receiver: fallback-pager
  group_by: [team]
  routes:

    - match:
        team: frontend
      group_by: [region, env]   # Группируем по тегам региона и среды
      receiver: frontend-pager
      routes:
        - match:
            severity: page
          receiver: frontend-pager
    - match:
        severity: ticket
      group_by: [region, env, alertname]  # Группируем по тегам региона и среды и предупреждения
      receiver: frontend-ticket
~~~

### пропуск и повторение

Для пропуска сообщений, если предыдущий набор сработал или для не рассылки устаревших сообщений  существует
настройка пропуска - котрая выражается в 2 секциях *group_wait* и  *group_interval*

* group_wait- интервал ожидания после срабатывания  правила, который необходим для группировки связанных уведомлений  - чтобы связать в цепочку события.
group_interval - это интервал с частотой которого надо отсылать уведомления, после срабатывания правила.

group_wait обнуляется при фиксации проблемы, вызвавшей срабатывание правила.
Пропуск для каждой группы правил независим.

Для повторного напоминания о проблеме, если она не была устранена применяется настройка *repeat_interval*.
По умолчанию через 4 часа будут отправлены повторные оповещений

~~~ yaml
route:
  receiver: fallback-pager
  group_by: [team]
  routes:
    # Frontend team.
    - match:
        team: frontend
      group_by: [region, env]
      group_interval: 10m
      receiver: frontend-pager
      routes:
        - match:
            severity: page
          receiver: frontend-pager
          group_wait: 1m
        - match:
            severity: ticket
          receiver: frontend-ticket
          group_by: [region, env, alertname]
          group_interval: 1d
          repeat_interval: 1d
~~~

## Получатели (Receivers)

Получатели - это целевая группа технических средств, которые получают уведомления (email, HipChat, PagerDuty, Pushover, Slack, OpsGenie, VictorOps, WeChat, webhook).
Получатели должны иметь  уникальное имя, и могут участвовать во множестве событий.

~~~ yaml
receivers:
 - name: fallback-pager
   XXX_configs:
    - REQ_PARAM: XXXXXXXX
~~~

Конфигурация для вебхука - например Telegram

~~~ yaml
route:
  receiver: fallback-pager
    routes:
    - receiver: log-alerts
      continue: true
    receivers:
    - name: log-alerts
      webhook_configs:
      - url: http://localhost:1234/log
~~~

### Шаблоны нотификации

Разметка нотификации выполняется с помощью шаблонов на языке Go.

Пример разметки для Slack:

~~~ yaml
receivers:
 - name: frontend-pager
   slack_configs:
    - api_url: https://hooks.slack.com/services/XXXXXXXX
      channel: '#pages'
      title: 'Alerts in {{ .GroupLabels.region }} {{ .GroupLabels.env }}!' # шаблон
~~~

Доступные переменные:

* GroupLabels - включаются переменные, определенные в group_by
* CommonLabels - общие метки для всех уведомлений, включают в себя GroupLabels
* CommonAnnotations - подобно GroupLabels ,  но для аннотаций
* ExternalURL - URL для доступа к Alertmanger создавшего оповещение
* Status - статус Не решен\Решен
* Receiver - Имя получателя в конфигурационном файле
* GroupKey - строка с идентификатором группы
* Alerts - суть оповещения

Алерт содержит несколько своих полей:

* Labels - метки оповещения
* Annotations - анотации
* Status - статус Не решен\Решен
* StartsAt - время срабатывания предупреждения
* EndsAt - время устранения проблемы
* GeneratorURL - сылка на правило предупреждений в prometheus

Пример шаблона сообщения

~~~ yaml
receivers:
  - name: frontend-pager
    slack_configs:
      - api_url: https://hooks.slack.com/services/XXXXXXXX
        channel: '#pages'
        title: 'Alerts in {{ .GroupLabels.region }} {{ .GroupLabels.env }}!'
        text: >
            {{ .Alerts | len }} alerts:
            {{ range .Alerts }}
            {{ range .Labels.SortedPairs }}{{ .Name }}={{ .Value }} {{ end }}
            {{ if eq .Annotations.wiki "" -}}
            Wiki: http://wiki.mycompany/{{ .Labels.alertname }}
            {{- else -}}
            Wiki: http://wiki.mycompany/{{ .Annotations.wiki }}
            {{- end }}
            {{ if ne .Annotations.dashboard "" -}}
            Dashboard: {{ .Annotations.dashboard }}&region={{ .Labels.region }}
            {{- end }}
            {{ end }}
~~~

Шаблоны также можно использовать  для создания маршрутов рассылки уведомлений

~~~ yaml
groups:
  - name: example
    rules:
    - record: latency_too_high_threshold
      expr: 0.5
      labels:
        email_to: foo@example.com
        owner: foo
    - record: latency_too_high_threshold
      expr: 0.7
      labels:
        email_to: bar@example.com
        owner: bar
      - alert: LatencyTooHigh
        expr: |
          # Alert based on per-owner thresholds.
          owner:latency:mean5m
          > on (owner) group_left(email_to)
          latency_too_high_threshold
~~~

Далее созданные метки используются для рассылки по группам

~~~ yaml
global:
  smtp_smarthost: 'localhost:25'
  smtp_from: 'youraddress@example.org'
route:
  group_by: [email_to, alertname]
  receiver: customer_email
receivers:
  - name: customer_email
  email_configs:
    - to: '{{ .GroupLabels.email_to }}'
      headers:
        subject: 'Alert: {{ .GroupLabels.alertname }}'
~~~

### Уведомления о разрешении ситуации

Каждое предупреждение имет атрибут send_resolved. Если оно установлено в TRUE, это означает что проблема разрешена.
Если в Alertmanager авктивирована настойка send_resolved, то уведомления о разрешении также будут рассылатся получатлям.


## Запреты  (Inhibitions)

Запреты позволяют выстривать предупрждения в цепочку, и не рассылать уведомлния о срабатывании, если сработало проавило из цепочки.

~~~ yaml
inhibit_rules:
  - source_match:
      severity: 'page-regionfail'
    target_match:
      severity: 'page'
    equal: ['region']
~~~

Предупрждение page-regionfail подавляет все иные предупреждения, с уровнем page и тем же самым регионом.
