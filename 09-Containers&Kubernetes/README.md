# Контейнеры и Kubernetes
Все компоненты Prometheus пригодны для эксплуатации в контейнерной среде. (Для Node Exporter - это спорное заявление)

## cAdvisor
По аналогии с Node Exporter для  контейнерной среды существует контейнер со сбором метрик с Linux cgroups.

~~~ yaml
  cadvisor:
    image: gcr.io/google-containers/cadvisor:latest
    container_name: cadvisor
    ports:
    - 8080:8080
    volumes:
    - /:/rootfs:ro
    - /var/run:/var/run:rw
    - /sys:/sys:ro
    - /var/lib/docker/:/var/lib/docker:ro
~~~

### ЦПУ (CPU)
С ЦПУ связано 3 метрики:

* *container_cpu_usage_seconds_total* - с метками по ЦПУ.
* *container_cpu_system_seconds_total* - время когда ЦПУ было занято процессами system space.
* *ontainer_cpu_user_seconds_total* - время когда ЦПУ было занято процессами user space.

### ОЗУ (Memory)

* *container_memory_cache* - страничный кеш памяти, занятый контейнерами
* *container_memory_rss* - резидентный набор в байтах
* *container_memory_usage_bytes* - резидентный набор  и страничный кеш памяти 
* *container_spec_memory_limit_bytes* - предел использования памяти контейнером

### Метки (Labels)
cgrous образуют иерархию.  В дополнение к *id* cAdvisor добавляет в метрикам метки *image* и *name*.
Также включатся метрика *container_label_ prefix* для идентификации.


## Kubernetes


Отложим до отдельной лекции





 