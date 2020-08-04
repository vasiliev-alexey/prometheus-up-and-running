# Инструменты, библиотеки сбора и передачи метрик Prometheus

Самое большое преимущество от использования Prometheus можно получить если инструментировать приложения или использовать в его коде библиотеки.

Официальные библиотеки Prometheus представлены на языках:
* Go
* Python
* Java
* Ruby

## Counter
Counter - наиболее часто встречающаяся метрика. Она показывает число или размер событий. По большей части ее использование - это учет, сколько раз тот или иной код был запущен

Официальные библиотеки Prometheus потоко-безопасны и автоматически регистрируют метрики в своем [реестре](3-1-example.py).


    REQUESTS = Counter('hello_worlds_total',
            'Hello Worlds requested.')

    class MyHandler(http.server.BaseHTTPRequestHandler):
        def do_GET(self):
            REQUESTS.inc()

### Ошибки Counting
Инструментация Prometheus позволяет автоматически [подсчитывать исключения ](3-4-example.py)

    EXCEPTIONS = Counter('hello_world_exceptions_total',
            'Exceptions serving Hello World.')

    class MyHandler(http.server.BaseHTTPRequestHandler):
        def do_GET(self):
            REQUESTS.inc()
            with EXCEPTIONS.count_exceptions():
            if random.random() < 0.2:
                raise Exception

Можно также использовать функции декораторы 

    EXCEPTIONS = Counter('hello_world_exceptions_total',
    'Exceptions serving Hello World.')

    @EXCEPTIONS.count_exceptions()


### Counting Size
Prometheus  в качестве счетчиков использует 64 битные беззнаковые  числа двойной точности, поэтому можно использовать их [для многих целей](3-5-example.py)

    SALES = Counter('hello_world_sales_euro_total', 'Euros made serving Hello World.')

    euros = random.random()
    SALES.inc(euros)

Счетчики могут только увеличиваться, и никогда не уменьшатся.
При рестарет приложения они всегда сбрасываются в 0 и их не надо сохранять при рестарте.


## Gauge
Gauge  -  показывает текущее состояние метрики (снапшот). Метрика  может как увеличиваться,  так и уменьшатся.

Примеры:
* длина очереди
* использование памяти для кеша
* число активных тредов
* последнее время записи
* среднее значение за последнюю минуту

Gauge имеет несколько [методов](3-6-example.py)
* inc
* dec
* set

~~~ python
INPROGRESS = Gauge('hello_worlds_inprogress',
        'Number of Hello Worlds in progress.')
LAST = Gauge('hello_world_last_time_seconds',
        'The last time a Hello World was served.')
...
        INPROGRESS.inc()
...
        LAST.set(time.time())
        INPROGRESS.dec()
~~~

В консоли Prometheus  можно  просматривать время последнего выполнения, через функцию
~~~
time() - hello_world_last_time_seconds
~~~
## Summary
В целом Summary можно применять для расчета latency используя [функцию установки  time.time()](3-9-example.py)

~~~
LATENCY.observe(time.time() - start)
~~~

В консоли Prometheus  это покажет интересный график задержек
~~~
rate(hello_world_latency_seconds_sum[1m])/rate(hello_world_latency_seconds_count[1m])
~~~


Summary  также использует квантили


## Histogram
Histogram предназначены для отображения квантилей например 0.95 0.99 итп
Инструментация Histogram [такая-же](3-11-example.py)    как и у Summary

~~~ python
LATENCY = Histogram('hello_world_latency_seconds',
        'Time for a request Hello World.')

...
    @LATENCY.time()
...
~~~

Это создаст набор метрик вида hello_world_latency_seconds_bucket, которые будут состоять из набора  такого как 1ms ,10ms, 25ms - и количества событий попавших в эту метрику 

в консоли prometheus можно вывести метрику
~~~
histogram_quantile(0.95, rate(hello_world_latency_seconds_bucket[1m]))
~~~

## Buckets
Buckets  обычно покрывают диапазон ожиданий (range of latencies)  между 1ms и 10s
Это типичное время ожидания у WEB приложений, и оно достаточно для отслеживания  SLA
~~~ python
LATENCY = Histogram('hello_world_latency_seconds',
'Time for a request Hello World.',
buckets=[0.0001, 0.0002, 0.0005, 0.001, 0.01, 0.1])
~~~

Если используются линейные или экспоненциальные бакеты, то можно использовать функции
~~~ python
buckets=[0.1 * x for x in range(1, 10)] # Linear
buckets=[0.1 * 2**x for x in range(1, 10)] # Exponential
~~~ 

## Юнит тестирование инструментария.
Можно использовать в юнит тестах, замеряя значения метрик из реестра библиотек
~~~ python
import unittest
from prometheus_client import Counter, REGISTRY
FOOS = Counter('foos_total', 'The number of foo calls.')
def foo():
    FOOS.inc()

class TestFoo(unittest.TestCase):
def test_counter_inc(self):
before = REGISTRY.get_sample_value('foos_total')
foo()
after = REGISTRY.get_sample_value('foos_total')
self.assertEqual(1, after - before)
    
~~~


## Преимущества инструментирования

Инструментируют 2 вещи:
* сервисы
* библиотеки

### *инструментация сервисов*
Выделяют 3 вида сервисов:
* Онлайн сервисы - сервисы в которых пользовать ждет ответ синхронно (web , базы данных). Метрики таких систем должны быть как на серверно так  и на клиентской стороне. Обычно тут используется подход RED
* Оффлайн сервисы - сервисы, от которых не ждут прямого ответа. Метрики таких систем, должны содержать множество. позволяющее определить состояние - число в очереди, сколько уже в прогрессе, как быстро идет обработка, количество ошибок. Обычно тут используется подход USE.
* Сервисы пакетных заданий - похожи на офлайновые сервисы, но они имеют  календарь запуска. Как правило метрики таких систем  имеют метрики вида - сколько задач выполнено, когда запускались в последний раз, сколько успешных и так далее. Часто они содержат средства оповещения о проблемах по тем или иным критериям.

### *инструментация библиотек*
библиотеки подобны минисервисам, и для них может быть применен аналогичный подход.
Хорошей идеей является  подсчет ошибок в работе методов. используя метрики - это уменьшит число просмотра логов, и приведет к более точной отладке.
Пулы потоков  и обработчиков, могут рассматриваться как офлайн сервисы
Фоновые обработчики - как сервисы пакетных заданий.

## сколько инструментировать
можно экстримально много, но лучше отталкиваться от выбранной стратегии.
Не стоит сильно заморачиваться над вопросом, единичная метрика займет примерно 0,01% от ресурсов приложений.
В конце концов вы остановитесь на десятке ключевых метрик, характеризующих поведение системы.


## Как именовать метрики
Именовать метрики - наука. Вот несколько лучщих правил:
* Символы  (Characters) - используйте метрики по регулярному выражению [a-zA-Z_:][a-zA-Z0-9_:]*. Разделяйте  подчеркиванием (snake_case)
* Суффиксы метрик (Metric suffixes). Суффиксы _total,   _count, _sum, и  _bucket 
 используются в  counter, summary,   histogram  метриках. Избегайте  их в окончании своих метрик, чтобы не обманутся.
* Единицы измерения (Units) - избегайте префиксов seconds, bytes, и  ratios  - в своих метриках. особенно в таких  как  time
* Имена (Name). Имя метрики должно отвечать знаниям о системе. Например  имя requests плохо, 
http_requests - хорошо, http_requests_authenticated  еще лучше. Используйте имена для понимания, не включайте лейблы в имена метрик.
* Библиотеки (Library) - избегайте коллизии в метриках в глобальном контексте. приходящих из зависимостей через библиотеки. Используйте имя библиотеки как часть имени метрики - чтобы не пересечься в глобальном контексте 



************************************
 