# Node Exporter
Node Exporter -  открывает для обозрения метрики из операционной системы,  связанных с оборудованием, сетью , памяатью  и другие.

Для windows - необходимо использовать отдельный экспортер WMI_expoter.
Node Exporter  - предназначен для мониторинга всей машины, а не отдельных сервисов или процессов.

Node Exporter спроектирован для запуска из не системного пользователя, и должне запускаться на хосте как сервис.

С помощью ключей комманднос строки, можно сконфигурировать Node Exporter для сбора метрик отдедбной функциональности, например --collector.wifi - включает наблюдение  за метрикам wifi, а --no-collector.wifi отключает.

## CPU Collector
Основной метрикой сбора является node_cpu_seconds_total, которая показывает сколько времени каждый ЦПУ был в том или ином  состоянии.
~~~ sh
# HELP node_cpu_seconds_total Seconds the cpus spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 0
node_cpu_seconds_total{cpu="0",mode="iowait"} 0
node_cpu_seconds_total{cpu="0",mode="irq"} 0
node_cpu_seconds_total{cpu="0",mode="nice"} 0
node_cpu_seconds_total{cpu="0",mode="softirq"} 0
node_cpu_seconds_total{cpu="0",mode="steal"} 0
node_cpu_seconds_total{cpu="0",mode="system"} 0
node_cpu_seconds_total{cpu="0",mode="user"} 0
~~~

Время простоя ЦПУ за минуту
~~~ sh
avg without(cpu, mode)(rate(node_cpu_seconds_total{mode="idle"}[1m]))
~~~


## Filesystem Collector
Сборщик метрик файловых систем собирает метрики для всех смонтированных файловых систем - подобно команде 
~~~ sh
df
~~~
флаги --collector.filesystem.ignored-mount-points  и --collector.filesystem.ignored-fs-types позволяют отключить сбор  для определенных файловых систем.

Если сбор метрик запущен не из под системного пользователя. необходяим убелится что у пользотвалея есть  доступ для системных вызовов statfs.

Все метрики коллектора имеют префикс *node_filesystem_* и имеют метки *device* , *fstype*, *mountpoint*
~~~ sh
# HELP node_filesystem_size_bytes Filesystem size in bytes.
# TYPE node_filesystem_size_bytes gauge
node_filesystem_size_bytes{device="/dev/sdb1",fstype="ext3",mountpoint="/home"}0
~~~

Формула для вычисления использованного пространства

~~~ sh
node_filesystem_avail_bytes/  node_filesystem_size_bytes
~~~

## Diskstats Collector
Сборщик предназначн для сбора метрик I/O  из  */proc/diskstats*

Все метрики имеют метку device
* node_disk_io_now  
    Количество выполняемых операций ввода-вывода.
* node_disk_io_time_seconds_total
    Увеличивается при выполнении ввода-вывода.
* node_disk_read_bytes_total  
   Байты считыванные с  ввода/вывода
* node_disk_read_time_seconds_total  
    Время, затраченное на чтение ввода-вывода.
* node_disk_reads_completed_total  
    Количество полных операций ввода-вывода.
*  node_disk_written_bytes_total  
    Байтов, записанных с помощью операций ввода-вывода
* node_disk_write_time_seconds_total
    Время, затраченное на запись ввода-вывода
* node_disk_writes_completed_total
    Количество полных операций ввода-вывода записи

Утилизация диска
~~~
rate(node_disk_io_time_seconds_total[1m])
~~~
Среднее время ввода вывода
~~~
rate(node_disk_read_time_seconds_total[1m]) /  rate(node_disk_reads_completed_total[1m])
~~~

## Netdev Collector
Коллектор предоставлет метрики о сетевом оборудовании, имеет префикс *node_network_* и метку *device*
~~~ sh
# HELP node_network_receive_bytes_total Network device statistic receive_bytes.
# TYPE node_network_receive_bytes_total counter
node_network_receive_bytes_total{device="lo"}  0 
node_network_receive_bytes_total{device="wlanX"} 0
~~~

node_network_receive_bytes_total и node_network_transmit_bytes_total - основные метрики для расчета пропускной способности сети.
~~~ sh
rate(node_network_receive_bytes_total[1m])
~~~

## Meminfo Collector
Коллектор   записывает стандартные метрики памяти и имеет префикс *node_memory_*. Данные берутся их */proc/meminfo*
* node_memory_MemTotal_bytes  
    Общая физическая память хоста
* node_memory_MemFree_bytes  
    Неиспользуемая память на хосте
* node_memory_MemAvailable   
    Сколько памяти действительно доступно  на хосте

## Hwmon Collector
коллектор предоставляет данные о температуре на оборудовании и имееет префикс *node_hwmon_*

## Stat Collector
предоставляет метрики из */proc/stat* 
* *node_boot_time_seconds* 
    время запуска ядра, по которому вы можете вычислить, как долго ядро работало
    ~~~
    time() - node_boot_time_seconds
    ~~~
* *node_intr_total* - число прерывания устройства
* *node_forks_total* - число системных вызовов fork
* *node_context_switches_total* - число переключений контекстов
* *node_procs_blocked* *node_procs_running  indicate* - число заблокированных и работающих процессов

## Uname Collector
Предоставляеn  метрику node_uname_info
~~~ sh
# HELP node_uname_info Labeled system information as provided by the uname    system call.
# TYPE node_uname_info gauge
node_uname_info{domainname="(none)",machine="x86_64",nodename="kozo",    release="4.4.0-101-generic",sysname="Linux",    version="#124-Ubuntu SMP Fri Nov 10 18:29:59 UTC 2017"} 1
~~~

## Loadavg Collector
Коллектор обеспечивает предоставление метрик load   averages в диапазоне 1-,   5-,   and   15-минут


## Textfile Collector
Для предоставления метрик. у котрых нет доступа для Node Exporter используется механизм отслеживания измнения в текстовых файлах, содержащих необходимоу информацию. Через мехнизм переодических заданий файлы перегенериуются.
Не слежует использовать этот метод, для того чтобы превратить в pull модель Prometheus
### Использвоание Textfile Collector
Необходимо включить флаг *--collector.textfile.directory*  в коммандной строке для указания диретории с файлами.
Файлы должны иметь расширение .prom

~~~ sh
mkdir text_exp
echo sample_metric 1 > text_exp/sample.prom
./node_exporter --collector.textfile.directory=$PWD/text_exp
~~~

### Timestamps
Используетй временную метрику для опеределения давности сняти метерие с  Textfile Collector
~~~
node_textfile_mtime_seconds{file="sample.prom"} 1.516205651e+09
~~~
