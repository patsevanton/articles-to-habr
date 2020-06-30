# Первый перевод: Как использовать grok exporter для создания метрик prometheus из неструктурированных журналов

Поговорим о grok exporter. В этой статье я объясню, как можно использовать [grok exporter](https://github.com/fstab/grok_exporter) для создания метрик prometheus из неструктурированных журналов.

![](https://www.easyaslinux.com/wp-content/uploads/2019/05/use-grok-exporter-to-create-prometheus-metrics-from-unstructured-logs-1024x576.png)

Grok популярен для обработки журналов в массовом ELK стеке (ElasticSearch, Logstash, Kibana) и благодаря [Fabian Stäber](https://github.com/fstab) для разработки **grok exporter**.

Вот официальная документация grok exporter => https://github.com/fstab/grok_exporter

## Шаг 1: Установите Grok exporter

Давайте получим готовый zip файл grok exporter от https://github.com/fstab/grok_exporter/releases.

1. ​	Перейдите в раздел релизы (releases) и нажмите на последнюю версию (теперь это v0.2.7).
2. ​	Затем загрузите zip-файл, соответствующий вашей операционной системе. Моя операционная система - это 64-битный Linux. Такова будет команда.

```
wget https://github.com/fstab/grok_exporter/releases/download/v0.2.7/grok_exporter-0.2.7.linux-amd64.zip
```

1. ​	Распакуйте файл и перейдите в извлеченный каталог.
2. ​	Затем выполните команду ниже, чтобы запустить grok exporter.

```
[root@localhost grok_exporter-0.2.7.linux-amd64]# ./grok_exporter -config ./config.yml
Starting server on http://localhost.localdomain:9144/metrics
```

Теперь вы можете посмотреть примеры метрик по адресу http://localhost.localdomain:9144/metrics.

## **Шаг 2: Давайте обработаем некоторые пользовательские журналы**

Давайте исследуем некоторые примеры журналов с Grok exporter. Вот несколько случайных журналов, сделанных мной.

```
30.07.2016 04:33:03 10.3.4.1 user=Nijil message="logged in"
30.07.2016 06:47:03 10.3.4.2 user=Alex message="logged failed"
30.07.2016 06:55:03 10.3.4.2 user=Alex message="logged in"
30.07.2016 07:03:03 10.3.4.3 user=Alan message="logged in"
30.07.2016 07:37:03 10.3.4.1 user=Nijil message="logged out"
30.07.2016 08:47:03 10.3.4.2 user=Alex message="logged out"
30.07.2016 14:34:03 10.3.4.3 user=Alan message="logged out"
```

Как вы уже могли догадаться, журнал показывает активность входа пользователя в поле. Теперь мне нужно создать метрику Prometheus из этого.

На Шаге 1 вы, возможно, заметили путь config.xml, который упоминается в стартовой команде grok exporter. Откройте конфигурационный файл и замените содержимое нижеприведенными данными.

```
global:
    config_version: 2
input:
    type: file
    path: ./example/nijil.log  # Specify the location of the your log
    readall: true              # This should be True if you want to read whole log and False if you want to read only new lines.
grok:
    patterns_dir: ./patterns    
metrics:
    - type: counter
      name: user_activity
      help: Counter metric example with labels.
      match: "%{DATE} %{TIME} %{HOSTNAME:instance} user=%{USER:user} message=\"%{GREEDYDATA:data}\""
      labels:
          user    : '{{.user}}'

server:
    port: 9144
```

Схема вышеприведенной конфигурации сделана внизу.

```
global:
    # Config version
input:
    # How to read log lines (file or stdin).
grok:
    # Available Grok patterns.
metrics:
    # How to map Grok fields to Prometheus metrics.
server:
    # How to expose the metrics via HTTP(S).
```

## **Шаг 3: Настройка сердца Grok exporter**

В приведенной выше конфигурации наиболее интересным разделом являются метрики, где мы указываем, как строки журнала должны быть сопоставлены с метриками Prometheus.

```
metrics:
    - type: counter
      name: user_activity
      help: Counter metric example with labels.
      match: "%{DATE} %{TIME} %{HOSTNAME:instance} user=%{USER:user} message=\"%{GREEDYDATA:data}\""
      labels:
          user    : '{{.user}}'
```

Синтаксис шаблона grok – это `%{SYNTAX:SEMANTIC}`, где `SYNTAX` - это имя шаблона, который будет соответствовать журналу, а `SEMANTIC` – это имя поля для присвоения значения сопоставленному журналу. Возьмем в качестве примера `%{HOSTNAME:instance}`, `HOSTNAME` – это шаблон grok, который только удаляет IP-часть из журнала, и эта `IP-часть` сохраняется в экземпляре (здесь вы можете дать любое имя), который вы можете использовать позже. Вы должны отметить, что каждый `SYNTAX` имеет свою собственную цель, это означает, что вы не можете отказаться от IP-адреса в журнале с синтаксисом даты. И как следует из названия, `DATE`, `TIME`, `HOSTNAME`, `USER` и `GREEDYDATA` обрывки даты, времени, имени хоста и "любого сообщения" соответственно.

Вы можете использовать метки, чтобы решить, какая метрика на основе параметров должна быть сгенерирована. Из приведенной выше конфигурации вы можете понять, что метрика основана на имени пользователя. Обратите внимание, что вам нужно использовать семантику синтаксиса (SEMANTIC of the SYNTAX) для метки. Для метрики должно быть по крайней мере два параметра. Здесь мы имеем имя пользователя в одной оси и количество вхождений пользователя в другую ось. Второй параметр определяется на основе счетчика метрического типа. Как и Счетчик (Counter), у grok exporter есть и другие метрические типы, узнайте о них из официальных документов.

А теперь займитесь grok exporter `./grok_exporter -config ./config.yml` и откройте метрики в браузере. Вы увидите метрики, созданные с именем `user_activity`, как мы указали в конфигурационном файле.

```
# TYPE user_activity counter
user_activity{user="Alan"} 2
user_activity{user="Alex"} 3
user_activity{user="Nijil"} 2
```

Построение этой метрики в виде графика в Prometheus выходит за рамки данной статьи. Но это очень просто, вам просто нужно указать конечную точку метрики в конфигурации prometheus, чтобы Prometheus мог удалить данные Метрики и построить график для вас.

Второй перевод по теме grok exporter

# Второй перевод: Получение метрик из журналов Apache с помощью grok exporter

Не все приложения создают полезные метрики, но некоторые из них создают журналы.

Существует класс приложений, которые практически не имеют полезных метрик, однако информацию, которую вы хотели бы иметь, можно найти в журналах. Одним из вариантов решения этой проблемы является [экспортер grok](https://github.com/fstab/grok_exporter). Допустим, вы работали с Apache, вот несколько примеров журналов из типичного *access.log*:

```
x.x.x.x - - [20/Jan/2020:06:25:24 +0000] "GET / HTTP/1.1" 200 62316 "http://178.62.121.216" "Go-http-client/1.1"
x.x.x.x - - [20/Jan/2020:06:25:25 +0000] "GET / HTTP/1.1" 200 16061 "-" "Go-http-client/1.1"
x.x.x.x - - [20/Jan/2020:06:25:25 +0000] "GET / HTTP/1.1" 200 16064 "-" "Go-http-client/1.1"
x.x.x.x - - [20/Jan/2020:06:25:25 +0000] "GET /blog/rss HTTP/1.1" 301 3478 "-" "Tiny Tiny RSS/19.2 (adc2a51) (http://tt-rss.org/)"
x.x.x.x - - [20/Jan/2020:06:25:26 +0000] "GET / HTTP/1.1" 200 16065 "-" "Go-http-client/1.1"
x.x.x.x - - [20/Jan/2020:06:25:26 +0000] "GET /blog/feed HTTP/1.1" 200 3413 "-" "Tiny Tiny RSS/19.2 (adc2a51) (http://tt-rss.org/)"
x.x.x.x - - [20/Jan/2020:06:25:27 +0000] "GET /feed HTTP/1.1" 200 6496 "-" "Emacs Elfeed 3.2.0"
x.x.x.x - - [20/Jan/2020:06:25:27 +0000] "GET / HTTP/1.1" 200 62316 "http://178.62.121.216" "Go-http-client/1.1"
```

Для начала загрузите:

```
wget https://github.com/fstab/grok_exporter/releases/download/v1.0.0.RC2/grok_exporter-1.0.0.RC2.linux-amd64.zip
```

распакуйте

```
unzip grok_exporter-*.zip
cd grok_exporter*amd64
```

настройке:

```
cat << 'EOF' > config.yml
global:
    config_version: 2
input:
    type: file
    path: access.log
    readall: true
grok:
    patterns_dir: ./patterns
metrics:
    - type: counter
      name: apache_http_response_codes_total
      help: HTTP requests to Apache
      match: '%{COMBINEDAPACHELOG}'
      labels:
          method: '{{.verb}}'
          path: '{{.request}}'
          code: '{{.response}}'
server:
    port: 9144
EOF
```

и запустите grok exporter:

```
./grok_exporter -config config.yml
```

Если вы посетите нас http://localhost:9144/metrics вы увидите метрику:

```
# HELP apache_http_response_codes_total HTTP requests to Apache
# TYPE apache_http_response_codes_total counter
apache_http_response_codes_total{code="200",method="GET",path="/"} 5
apache_http_response_codes_total{code="200",method="GET",path="/blog/feed"} 1
apache_http_response_codes_total{code="200",method="GET",path="/feed"} 1
apache_http_response_codes_total{code="301",method="GET",path="/blog/rss"} 1
```

Для демонстрации было включено **readall**, которое позволит экспортеру читать файлы журнала с самого начала, а не обычное желаемое поведение обработки только недавно добавленных строк журнала.

Чтобы сделать небольшую резервную копию, Grok – это способ, которым вы можете анализировать [строки журнала в Logstash](https://www.elastic.co/guide/en/logstash/7.5/plugins-filters-grok.html) (Logstash - это L в стеке ELK). Соответственно, паттерны были написаны для многих распространенных приложений, включая Apache. Экспортер Grok позволяет вам строить на существующем корпусе шаблонов, а также добавлять свои собственные, чтобы извлекать метрики из журналов. `COMMMONAPACHELOG` наверху, например, определяется как

```
COMMONAPACHELOG %{IPORHOST:clientip} %{HTTPDUSER:ident} %{USER:auth} \[%{HTTPDATE:timestamp}\] "(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})" %{NUMBER:response} (?:%{NUMBER:bytes}|-)
```

что само по себе опирается на множество других паттернов. Так что вы должны быть рады, что вам не придется писать это самому с нуля. Поля меток предоставляют вам шаблон Go (как показано в шаблонах Prometheus alerting и notification) для доступа к этим значениям.

Помимо подсчета строк журнала, экспортер Grok может позволить вам наблюдать размеры событий, такие как количество байтов в ответах:

```
    - type: summary
      name: apache_http_response_bytes
      help: Size of HTTP responses
      match: '%{COMMONAPACHELOG}'
      value: '{{.bytes}}'
```

Или датчики, например, когда был последний запрос:

```
    - type: gauge 
      name: apache_http_last_request_seconds
      help: Timestamp of the last HTTP request
      match: '%{COMMONAPACHELOG}'
      value: '{{timestamp "02/Jan/2006:15:04:05 -0700" .timestamp}}'
```

Это происходит с помощью функции метки времени (**timestamp**) grok exporter, которая в основном является [time.Parse](https://golang.org/pkg/time/#Parse) из Golang. Разбор. Есть также функция деления (**divide**), если вам нужно преобразовать миллисекунды в секунды.

Это небольшая проба того, что вы можете сделать с экспортером Grok. Как вы можете видеть, синтаксический анализ журналов требует некоторой конфигурации, поэтому вы должны использовать такой экспортер только в том случае, если у вас нет возможности заставить само приложение создавать хорошие показатели. 

Телеграм чат по метрикам https://t.me/metrics_ru