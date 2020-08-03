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

************************************
PS
Скрипт нагрузочник
    
    docker run --rm -v /etc/hosts:/etc/hosts   skandyla/wrk -t3 -c3 -d60  --timeout 10s   http://192.168.1.50:8001