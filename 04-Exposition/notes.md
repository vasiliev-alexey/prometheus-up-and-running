# Размещение (обнаружение) метрик

Обычно Prometheus  выставляет метрики для считывания  по протоколу http и корневому URL /metrics
Для удобства - метрики предоставляются в человеко-читаемом виде.
Размещение как правило выполняется в точке входа  испольняемого кода приложения  (обычно часть конфигурации)
Обычно метрики размещаются в реестре default, но некоторые библиотеки могут иметь свои реестры. 

## Размещение для Python
~~~ python
from prometheus_client import start_http_server
if __name__ == '__main__':
start_http_server(8000)
// Your code goes here.
~~~
start_http_server - открывает URL  для обнаружения метрик

## Размещение для Go

Используются библиотеки

~~~ go
package main
import (
"log"
"net/http"
"github.com/prometheus/client_golang/prometheus"
"github.com/prometheus/client_golang/prometheus/promauto"
"github.com/prometheus/client_golang/prometheus/promhttp"
)
var (
requests = promauto.NewCounter(
prometheus.CounterOpts{
Name: "hello_worlds_total",
Help: "Hello Worlds requested.",
})
)
func handler(w http.ResponseWriter, r *http.Request) {
requests.Inc()
w.Write([]byte("Hello World"))
}

func main() {
http.HandleFunc("/", handler)
http.Handle("/metrics", promhttp.Handler())
log.Fatal(http.ListenAndServe(":8000", nil))
}

~~~

## Размещение для  Java

### Используется класс из библиотеки HTTPServer

~~~ JAVA
import io.prometheus.client.Counter;
import io.prometheus.client.hotspot.DefaultExports;
import io.prometheus.client.exporter.HTTPServer;
public class Example {
private static final Counter myCounter = Counter.build()
.name("my_counter_total")
.help("An example counter.").register();
public static void main(String[] args) throws Exception {
DefaultExports.initialize();
HTTPServer server = new HTTPServer(8000);
~~~ 

Зависимости мавена
~~~ xml
<dependencies>
<dependency>
<groupId>io.prometheus</groupId>
<artifactId>simpleclient</artifactId>
<version>0.3.0</version>
</dependency>
<dependency>
<groupId>io.prometheus</groupId>
<artifactId>simpleclient_hotspot</artifactId>
<version>0.3.0</version>
</dependency>
<dependency>
<groupId>io.prometheus</groupId>
<artifactId>simpleclient_httpserver</artifactId>
<version>0.3.0</version>
</dependency>
</dependencies>
~~~ 


### Servlet
Для Jetty -  необходимо использовать Servlet.

~~~ java
import io.prometheus.client.Counter;
import io.prometheus.client.exporter.MetricsServlet;
import io.prometheus.client.hotspot.DefaultExports;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.ServletException;
import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.servlet.ServletContextHandler;
import org.eclipse.jetty.servlet.ServletHolder;
import java.io.IOException;
public class Example {
static class ExampleServlet extends HttpServlet {
private static final Counter requests = Counter.build()
.name("hello_worlds_total")
.help("Hello Worlds requested.").register();
@Override
protected void doGet(final HttpServletRequest req,
final HttpServletResponse resp)
throws ServletException, IOException {
requests.inc();
resp.getWriter().println("Hello World");
}
}
public static void main(String[] args) throws Exception {
DefaultExports.initialize();
Server server = new Server(8000);
ServletContextHandler context = new ServletContextHandler();
context.setContextPath("/");
server.setHandler(context);
context.addServlet(new ServletHolder(new ExampleServlet()), "/");
context.addServlet(new ServletHolder(new MetricsServlet()), "/metrics");
server.start();
server.join();
}
}
~~~

Мавен зависимости
~~~ XML
<dependencies>
<dependency>
<groupId>io.prometheus</groupId>
<artifactId>simpleclient</artifactId>
<version>0.3.0</version>
</dependency>
<dependency>
<groupId>io.prometheus</groupId>
<artifactId>simpleclient_hotspot</artifactId>
<version>0.3.0</version>
</dependency>
<dependency>
<groupId>io.prometheus</groupId>
<artifactId>simpleclient_servlet</artifactId>
<version>0.3.0</version>
</dependency>
<dependency>
<groupId>org.eclipse.jetty</groupId>
<artifactId>jetty-servlet</artifactId>
<version>8.2.0.v20160908</version>
</dependency>
</dependencies>


~~~

## Pushgateway

В тех случаях, когла невозможно собрать метрики с сервиса напрямую (например процессы работающие по графику), применяется сбор метрик через промежуточный приемник. Такой приемник именуется Pushgateway

Запускаем Prometheus c  [конфигом](prometheus.yml) для прослушки Pushgateway

~~~ yaml
global:
  scrape_interval: 10s

scrape_configs:
 - job_name: pushgateway
   honor_labels: true
   static_configs:
    - targets:
      - localhost:9091
~~~


~~~ bash
 ./prometheus  --config.file="$HOME/lab/prometheus-up-and-running/04-Exposition/prometheus.yml" --web.enable-admin-api
~~~

У Pushgateway есть 3 метода  работы с метриками
* push
* pushadd
* delete

## Мосты (Bridges)
Предоставление  метрик не ограничен их выводом  в формате Prometheus. Можно использовать библиотеки и переформатировать метрики в форматы иных мониторинговых систем, например Graphite.
~~~ python
import time
from prometheus_client.bridge.graphite import GraphiteBridge

gb = GraphiteBridge(['graphite.your.org', 2003])
gb.start(10)
while True:
    time.sleep(1)
~~~

тут делается снапшот метрики отдается в формате, пригодном Graphite

## Парсеры (Parsers) 
Клиентские библиотеки позволяют парсить метрики из реестра Prometheus и выводить их в другом формате, для тулов или иных систем мониторинга (DataDog, InfluxDB, Sensu, and Metricbeat), потреблющих метрики в текстовом формате

~~~ python
from prometheus_client.parser import text_string_to_metric_families

for family in text_string_to_metric_families(u"counter_total 1.0\n"):
  for sample in family.samples:
    print("Name: {0} Labels: {1} Value: {2}".format(*sample))
~~~


## Форматы вывода (Exposition Format)
Метрики передаются в текстовом формате
~~~
Content-Type: text/plain; version=0.0.4; charset=utf-8
~~~
Используются 64 разрядные числа, строки разделяются символом \n


## Типы метрик (Metric Types)

Типы метрик:
* counter
* gauge
* summary
* histogram
* untyped. 

untyped применяется, если не  было произведено явное типизированние.


## Метки (Labels)
Множество меток должно быть разделено запятой. Порядок меток не важен, но предпочтительно его соблюдать между снятием метрик.


## Экранирование (Escaping)
переносы строк  обозначаются символом 
~~~ 
\n 
~~~

\ и " ' экранируютс
~~~ 
\ 
~~~

## Врменные значения (timestamp)
Временные значения представлены в виде Unix

## проверка метрик
Для проверки метрик есть отдельный инструмент promtool
~~~ bash
curl http://localhost:8000/metrics | promtool check-metrics
~~~

Общие ошибки
* включать перенос строки в последнюю метрику
* неверный формат имени или меток (не могут включать дефисы или начинаться с цифры)