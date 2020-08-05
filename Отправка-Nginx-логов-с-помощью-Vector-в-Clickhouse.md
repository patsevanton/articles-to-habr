![](https://habrastorage.org/webt/hs/th/d5/hsthd5wjxdinkrwblonh7cxfvp0.png)

Vector, предназначенной для сбора, преобразования и отправки данных логов, метрик и событий.

Будучи написанной на языке Rust, она отличается высокой производительностью и низким потреблением оперативной памяти по сравнению с аналогами. Кроме того, большое внимание уделено функциям, связанным с корректностью, в частности, возможностям сохранения неотправленных событий в буфер на диске и ротации файлов.

Архитектурно Vector является роутером событий, принимающим сообщения из одного или нескольких *источников*, опционально применяющим над этими сообщениями *преобразования*, и отправляющим их в один или несколько *стоков*.

Vector это замена filebeat и logstash, он может выступать в обоих ролях (получать и отправлять логи), более подробней на их [сайте](https://vector.dev).

Если в Logstash цепочка строится как input -> filter -> output то в Vector это [sources](https://vector.dev/docs/reference/sources/) -> [transforms](https://vector.dev/docs/reference/transforms/) -> [sinks](https://vector.dev/docs/reference/sinks/)  

Примеры можно посмотреть в документации.   

Будем настраивать связку Nginx (Access logs) -> Vector (Client | Filebeat) -> Vector (Server | Logstash) -> Clickhouse. Установим 3 сервера. Хотя можно обойти 2 серверами.

### На сервере (Log server)

#### [Установка Clickhouse](https://clickhouse.tech/docs/ru/getting-started/install/)

ClickHouse используют набор инструкций SSE 4.2, поэтому, если не указано иное, его поддержка в используемом процессоре, становится дополнительным требованием к системе. Вот команда, чтобы проверить, поддерживает ли текущий процессор SSE 4.2:

```
grep -q sse4_2 /proc/cpuinfo && echo "SSE 4.2 supported" || echo "SSE 4.2 not supported"
```

Выключаем Selinux на всех ваших серверах

```
sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
reboot
```

Сначала нужно подключить официальный репозиторий:

```
sudo yum install -y yum-utils mc
sudo rpm --import https://repo.clickhouse.tech/CLICKHOUSE-KEY.GPG
sudo yum-config-manager --add-repo https://repo.clickhouse.tech/rpm/stable/x86_64
```

Для установки пакетов необходимо выполнить следующие команды:

```
sudo yum install -y clickhouse-server clickhouse-client
```

Разрешаем clickhouse-server слушать сетевую карту в файле /etc/clickhouse-server/config.xml

```
<listen_host>0.0.0.0</listen_host>
```

Меняем уровень логирования c trace до debug

<level>debug</level>

Для запуска сервера в качестве демона, выполните:

```
service clickhouse-server start
```

#### Установка GeoLite2-City.mmdb

Если вам нужен GeoIP, то там где у вас будет установлен Vector скачайте базу Maxmind и положите в /opt/geoip/GeoLite2-City.mmdb.

Раньше MaxMind предоставляли часть своих библиотек для прямого скачивания по ссылке. Однако сейчас [функция прямого скачивания без регистрации недоступна](https://blog.maxmind.com/2019/12/18/significant-changes-to-accessing-and-using-geolite2-databases/). Поэтому сначала нужно [зарегистрироваться здесь](https://www.maxmind.com/en/geolite2/signup) и скачать базу:

Пример скачивания старой бд Maxmind 2019 года.

```
mkdir /opt/geoip
cd /opt/geoip
wget https://github.com/DocSpring/geolite2-city-mirror/raw/master/GeoLite2-City.tar.gz
tar zxvf GeoLite2-City.tar.gz
mv GeoLite2-City_20191029/GeoLite2-City.mmdb .
```

#### Установка [Vector](https://vector.dev/docs/setup/installation/)

```text
yum install -y https://packages.timber.io/vector/0.9.X/vector-x86_64.rpm mc
```

Настроим Vector как замену Logstash. Редактируем файл /etc/vector/vector.toml

```text
# /etc/vector/vector.toml

data_dir = "/var/lib/vector"

[sources.nginx_input_vector]
  # General
  type                          = "vector"
  address                       = "0.0.0.0:9876"
  shutdown_timeout_secs         = 30

[transforms.nginx_parse_json]
  inputs                        = [ "nginx_input_vector" ]
  type                          = "json_parser"

[transforms.nginx_parse_geoip]
  inputs                        = [ "nginx_parse_json" ]
  type                          = "geoip"
  database                      = "/opt/geoip/GeoLite2-City.mmdb"
  source                        = "remote_addr"
  target                        = "geoip"

[transforms.nginx_parse_rename_fields]
  inputs                        = ["nginx_parse_geoip"]
  type                          = "rename_fields"

  fields.geoip.city_name        = "remote_geoip_city_name"
  fields.geoip.country_code     = "remote_geoip_country_iso_code"
  fields.geoip.country_name     = "remote_geoip_country_name"
  fields.geoip.continent_code   = "remote_geoip_continent_code"
  fields.geoip.continent_name   = "remote_geoip_continent_name"
  fields.geoip.region_code      = "remote_geoip_region_iso_code"
  fields.geoip.region_name      = "remote_geoip_region_name"
  fields.geoip.latitude         = "remote_geoip_location_latitude"
  fields.geoip.longitude        = "remote_geoip_location_longitude"

[transforms.nginx_parse_add_defaults]
  inputs                        = [ "nginx_parse_rename_fields" ]
  type                          = "lua"
  version                       = "2"

  hooks.process = """
  function (event, emit)

    function split_first(s, delimiter)
      result = {};
      for match in (s..delimiter):gmatch("(.-)"..delimiter) do
          table.insert(result, match);
      end
      return result[1];
    end

    function split_last(s, delimiter)
      result = {};
      for match in (s..delimiter):gmatch("(.-)"..delimiter) do
          table.insert(result, match);
      end
      return result[#result];
    end

    event.log.upstream_addr             = split_first(split_last(event.log.upstream_addr, ', '), ':')
    event.log.upstream_bytes_received   = split_last(event.log.upstream_bytes_received, ', ')
    event.log.upstream_bytes_sent       = split_last(event.log.upstream_bytes_sent, ', ')
    event.log.upstream_connect_time     = split_last(event.log.upstream_connect_time, ', ')
    event.log.upstream_header_time      = split_last(event.log.upstream_header_time, ', ')
    event.log.upstream_response_length  = split_last(event.log.upstream_response_length, ', ')
    event.log.upstream_response_time    = split_last(event.log.upstream_response_time, ', ')
    event.log.upstream_status           = split_last(event.log.upstream_status, ', ')

    if event.log.upstream_addr == "" then
        event.log.upstream_addr = "127.0.0.1"
    end

    if event.log.user_id == "" then
        event.log.user_id = "0"
    end

    if (event.log.upstream_bytes_received == "-" or event.log.upstream_bytes_received == "") then
        event.log.upstream_bytes_received = "0"
    end

    if (event.log.upstream_bytes_sent == "-" or event.log.upstream_bytes_sent == "") then
        event.log.upstream_bytes_sent = "0"
    end

    if event.log.upstream_cache_status == "" then
        event.log.upstream_cache_status = "DISABLED"
    end

    if (event.log.upstream_connect_time == "-" or event.log.upstream_connect_time == "") then
        event.log.upstream_connect_time = "0"
    end

    if (event.log.upstream_header_time == "-" or event.log.upstream_header_time == "") then
        event.log.upstream_header_time = "0"
    end

    if (event.log.upstream_response_length == "-" or event.log.upstream_response_length == "") then
        event.log.upstream_response_length = "0"
    end

    if (event.log.upstream_response_time == "-" or event.log.upstream_response_time == "") then
        event.log.upstream_response_time = "0"
    end

    if (event.log.upstream_status == "-" or event.log.upstream_status == "") then
        event.log.upstream_status = "0"
    end

    if (event.log.fields.geoip.longitude == "") then
        event.log.upstream_status = 0.0
    end

    emit(event)

  end
  """

[transforms.nginx_parse_remove_fields]
    inputs                              = [ "nginx_parse_add_defaults" ]
    type                                = "remove_fields"
    fields                              = ["data", "file", "geoip", "host", "source_type"]

[transforms.nginx_parse_coercer]

    type                                = "coercer"
    inputs                              = ["nginx_parse_remove_fields"]

    types.request_length = "int"
    types.request_time = "float"

    types.response_status = "int"
    types.response_body_bytes_sent = "int"

    types.remote_port = "int"
    types.remote_geoip_location_latitude = "float"
    types.remote_geoip_location_longitude = "float"

    types.upstream_bytes_received = "int"
    types.upstream_bytes_send = "int"
    types.upstream_connect_time = "float"
    types.upstream_header_time = "float"
    types.upstream_response_length = "int"
    types.upstream_response_time = "float"
    types.upstream_status = "int"

    types.timestamp = "timestamp"

[sinks.nginx_output_clickhouse]
    inputs   = ["nginx_parse_coercer"]
    type     = "clickhouse"

    database = "vector"
    healthcheck = true
    host = "http://172.26.10.109:8123" #  Адрес Clickhouse
    table = "logs"

    encoding.timestamp_format = "unix"

    buffer.type = "disk"
    buffer.max_size = 104900000
    buffer.when_full = "block"

    request.in_flight_limit = 20
```

Вы можете откорректировать секцию transforms.nginx_parse_add_defaults.

Так как я использую данные конфиги для небольшого CDN и там в upstream_* может прилетать несколько значений   

Например:
```text
"upstream_addr": "128.66.0.10:443, 128.66.0.11:443, 128.66.0.12:443"
"upstream_bytes_received": "-, -, 123"
"upstream_status": "502, 502, 200"
```

Если это не ваша ситуация то эту секцию можно упростить   

Создадим настройки service для systemd /etc/systemd/system/vector.service
```text
# /etc/systemd/system/vector.service

[Unit]
Description=Vector
After=network-online.target
Requires=network-online.target

[Service]
User=vector
Group=vector
ExecStart=/usr/bin/vector
ExecReload=/bin/kill -HUP $MAINPID
Restart=no
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=vector

[Install]
WantedBy=multi-user.target
```

Теперь перейдем к настройке Clickhouse   

Заходим в Clickhouse

```
clickhouse-client -h 172.26.10.109 -m
```

172.26.10.109 - IP сервера где установилен Clickhouse.

Создадим БД vector

```
CREATE DATABASE vector;
```

Проверим что бд есть.

```
show databases;
```

Создаем таблицу vector.logs.

```sql
/* Это таблица где хранятся логи как есть */

CREATE TABLE vector.logs
(
    `node_name` String,
    `timestamp` DateTime,
    `server_name` String,
    `user_id` String,
    `request_full` String,
    `request_user_agent` String,
    `request_http_host` String,
    `request_uri` String,
    `request_scheme` String,
    `request_method` String,
    `request_length` UInt64,
    `request_time` Float32,
    `request_referrer` String,
    `response_status` UInt16,
    `response_body_bytes_sent` UInt64,
    `response_content_type` String,
    `remote_addr` IPv4,
    `remote_port` UInt32,
    `remote_user` String,
    `remote_geoip_city_name` String,
    `remote_geoip_country_iso_code` String,
    `remote_geoip_country_name` String,
    `remote_geoip_continent_code` String,
    `remote_geoip_continent_name` String,
    `remote_geoip_region_iso_code` String,
    `remote_geoip_region_name` String,
    `remote_geoip_location_latitude` Float32,
    `remote_geoip_location_longitude` Float32,
    `upstream_addr` IPv4,
    `upstream_port` UInt32,
    `upstream_bytes_received` UInt64,
    `upstream_bytes_sent` UInt64,
    `upstream_cache_status` String,
    `upstream_connect_time` Float32,
    `upstream_header_time` Float32,
    `upstream_response_length` UInt64,
    `upstream_response_time` Float32,
    `upstream_status` UInt16,
    `upstream_content_type` String,
    INDEX idx_http_host request_http_host TYPE set(0) GRANULARITY 1
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY timestamp
TTL timestamp + toIntervalMonth(1)
SETTINGS index_granularity = 8192;
```

Создаем таблицу vector.data_domain_traffic.

```
/* Например мы хотим сделать статистику по кол-ву трафика в разрезе домена с шагом в час */
/* Это таблица в которой будет хранится статистика */

CREATE TABLE vector.data_domain_traffic
(
    `timestamp` DateTime,
    `domain` String,
    `cached` UInt64,
    `uncached` UInt64,
    `total` UInt64
)
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (domain, timestamp)
TTL timestamp + toIntervalMonth(3)
SETTINGS index_granularity = 8192;
```

Создаем MATERIALIZED VIEW vector.view_domain_traffic. Запускаем `clickhouse-client -m` и делаем запрос.

```shell script
/* А это Materialized view который считает статистику и отправляет ее в нужную таблицу */
/* В моем случаи так как это проксирование всех запросов то кеш или не кеш я определяю адресом upstream (если он 127.0.0.1 то нода отдала из кэша) */
/* В вашем случаи надо смотреть на значения upstream_cache_status “MISS”, “BYPASS”, “EXPIRED”, “STALE”, “UPDATING”, “REVALIDATED” или “HIT” */
/* Например так */

CREATE MATERIALIZED VIEW vector.view_domain_traffic TO vector.data_domain_traffic
(
    `timestamp` DateTime('Etc/UTC'),
    `domain` String,
    `cached` UInt64,
    `uncached` UInt64,
    `total` UInt64
) AS
SELECT
    toStartOfHour(timestamp) AS timestamp,
    domainWithoutWWW(domain(lowerUTF8(request_http_host))) AS domain,
    sumIf(response_body_bytes_sent, upstream_cache_status == 'HIT') AS cached,
    sumIf(response_body_bytes_sent, upstream_cache_status != 'HIT') AS uncached,
    sum(response_body_bytes_sent) AS total
FROM vector.logs
WHERE (IPv4StringToNum(domain) = 0) AND (domain != '')
GROUP BY (domain, timestamp)
ORDER BY (domain, timestamp) ASC;

/* Остальные примеры смотрите в repo: clickhouse-sql/schemes.txt */
```

Проверяем что создались таблицы. Запускаем `clickhouse-client` и делаем запрос.

Переходим в бд vector.

```
use vector;

Ok.

0 rows in set. Elapsed: 0.001 sec.
```

Смотрим таблицы.

```
show tables;

┌─name────────────────┐
│ data_domain_traffic │
│ logs                │
│ view_domain_traffic │
└─────────────────────┘
```

После создания таблиц и вьюшек можно запускать Vector

```
systemctl enable vector
systemctl start vector
```

Логи vector можно посмотреть так

```
journalctl -f -u vector
```

В логах должна быть такая запись

```
INFO vector::topology::builder: Healthcheck: Passed.
```

### На клиенте (Web server)

На cервере с nginx необходимо выключить ipv6, так как в таблице logs в clickhouse используется поле `upstream_addr` IPv4, так как я не использую ipv6 внутри сети. Если ipv6 не выключить, то будут ошибки:

```
DB::Exception: Invalid IPv4 value.: (while read the value of key upstream_addr)
```

Возможно читатели, добавлять поддержку ipv6.

Создаем файл /etc/sysctl.d/98-disable-ipv6.conf

```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

Применяем настройки

```
sysctl --system
```

#### Установим nginx. 

Добавил файл репозитория nginx /etc/yum.repos.d/nginx.repo

```
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
```

Установим пакет nginx

```
yum install -y nginx mc
```

Для начала нам надо настроить формат логов в Nginx для этого в nginx.conf или в отдельный фаил надо добавить новый формат

```text
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

log_format vector escape=json
    '{'
        '"node_name":"web-us-1",'
        '"timestamp":"$time_iso8601",'
        '"server_name":"$server_name",'
        '"request_full": "$request",'
        '"request_user_agent":"$http_user_agent",'
        '"request_http_host":"$http_host",'
        '"request_uri":"$request_uri",'
        '"request_scheme": "$scheme",'
        '"request_method":"$request_method",'
        '"request_length":"$request_length",'
        '"request_time": "$request_time",'
        '"request_referrer":"$http_referer",'
        '"response_status": "$status",'
        '"response_body_bytes_sent":"$body_bytes_sent",'
        '"response_content_type":"$sent_http_content_type",'
        '"remote_addr": "$remote_addr",'
        '"remote_port": "$remote_port",'
        '"remote_user": "$remote_user",'
        '"upstream_addr": "$upstream_addr",'
        '"upstream_bytes_received": "$upstream_bytes_received",'
        '"upstream_bytes_sent": "$upstream_bytes_sent",'
        '"upstream_cache_status":"$upstream_cache_status",'
        '"upstream_connect_time":"$upstream_connect_time",'
        '"upstream_header_time":"$upstream_header_time",'
        '"upstream_response_length":"$upstream_response_length",'
        '"upstream_response_time":"$upstream_response_time",'
        '"upstream_status": "$upstream_status",'
        '"upstream_content_type":"$upstream_http_content_type"'
    '}';


    # Что бы не поломать вашу текущую конфигурациию, Nginx позволяет иметь несколько директив access_log
    access_log  /var/log/nginx/access.log  main;            # Стандартный лог
    access_log  /var/log/nginx/access.json.log vector;      # Новый лог в формате json

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```

Не забудте добавить правило в logrotate для новых логов (если log фаил не заканчивается на .log)

Удаляем default.conf из /etc/nginx/conf.d/

```
rm /etc/nginx/conf.d/default.conf
```

Добавляем виртуальный хост /etc/nginx/conf.d/vhost1.conf - vhost5.conf

```
upstream backend {
    server 127.0.0.1:8080;
}

server {
    listen 80;
    server_name vhost1; # делаем 5 виртуальных хостов, отличающиеся только server_name
    location / {
        proxy_pass http://backend;
    }
}
```

Добавляем в файл /etc/hosts виртуальные хосты:

```
ip-адрес-сервера-с-nginx vhost1
ip-адрес-сервера-с-nginx vhost2
ip-адрес-сервера-с-nginx vhost3
ip-адрес-сервера-с-nginx vhost4
ip-адрес-сервера-с-nginx vhost5
```

### Эмулятор HTTP сервера

В качестве эмулятора HTTP сервера будем использовать [nodejs-stub-server](https://github.com/maxiko/nodejs-stub-server) от [Maxim Ignatenko](https://habr.com/users/Anthrax_Beta/)

Nodejs-stub-server не имеет rpm. Здесь https://github.com/patsevanton/nodejs-stub-server создаем ему rpm. Собираться rpm будет с помощью [Fedora Copr](https://copr.fedorainfracloud.org/coprs/antonpatsev/nodejs-stub-server/)

Устанавливаем на upstream nginx rpm пакет nodejs-stub-server

```
yum -y install yum-plugin-copr epel-release
yum copr enable antonpatsev/nodejs-stub-server
yum -y install stub_http_server
systemctl start stub_http_server
systemctl enable stub_http_server
```

И если все готово то 

```text
nginx -t 
systemctl restart nginx
```

Теперь установим сам [Vector](https://vector.dev/docs/setup/installation/)
```text
yum install -y https://packages.timber.io/vector/0.9.X/vector-x86_64.rpm
```

Создадим фаил настроек для systemd /etc/systemd/system/vector.service
```text
[Unit]
Description=Vector
After=network-online.target
Requires=network-online.target

[Service]
User=vector
Group=vector
ExecStart=/usr/bin/vector
ExecReload=/bin/kill -HUP $MAINPID
Restart=no
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=vector

[Install]
WantedBy=multi-user.target
```

И настроим замену Filebeat в конфиге /etc/vector/vector.toml где 172.26.10.108 это IP адрес log сервера (Vector-Server)

```text
data_dir = "/var/lib/vector"

[sources.nginx_file]
  type                          = "file"
  include                       = [ "/var/log/nginx/access.json.log" ]
  start_at_beginning            = false
  fingerprinting.strategy       = "device_and_inode"

[sinks.nginx_output_vector]
  type                          = "vector"
  inputs                        = [ "nginx_file" ]

  address                       = "172.26.10.108:9876"
```

Не забудте добавить юзера vector в нужную группу что бы он мог читать log файлы. Например, nginx в centos создает логи с правами группы adm.

```
usermod -a -G adm vector
```

Запустим сервис vector

```text
systemctl enable vector
systemctl start vector
```

### Нагрузочное тестирование

Тестирование проводим с помощью Apache benchmark.

Устанавливаем его:

```
yum install -y httpd-tools mc
```

Запускаем тестирование с помощью Apache benchmark c 5 разных серверов:

```
while true; do ab -H "User-Agent: 1server" -c 10 -n 10 -t 10 http://vhost1/; sleep 1; done
while true; do ab -H "User-Agent: 2server" -c 10 -n 10 -t 10 http://vhost2/; sleep 1; done
while true; do ab -H "User-Agent: 3server" -c 10 -n 10 -t 10 http://vhost3/; sleep 1; done
while true; do ab -H "User-Agent: 4server" -c 10 -n 10 -t 10 http://vhost4/; sleep 1; done
while true; do ab -H "User-Agent: 5server" -c 10 -n 10 -t 10 http://vhost5/; sleep 1; done
```

### Проверим. 

Заходим в Clickhouse

```
clickhouse-client -h 172.26.10.109 -m
```

Делаем SQL запрос

```text
SELECT * FROM vector.data_domain_traffic WHERE domain = 'vhost1' ORDER BY timestamp ASC;

┌───────────timestamp─┬─domain───────┬─cached─┬─uncached─┬─total─┐
│ 2020-07-22 13:00:00 │ example.com  │  34370 │        0 │ 34370 │
└─────────────────────┴──────────────┴────────┴──────────┴───────┘
┌───────────timestamp─┬─domain───────┬─cached─┬─uncached─┬─total─┐
│ 2020-07-22 14:00:00 │ example.com  │      4 │        0 │     4 │
└─────────────────────┴──────────────┴────────┴──────────┴───────┘
┌───────────timestamp─┬─domain───────┬─cached─┬─uncached─┬─total─┐
│ 2020-07-22 14:00:00 │ example.com  │     27 │        0 │    27 │
└─────────────────────┴──────────────┴────────┴──────────┴───────┘
┌───────────timestamp─┬─domain───────┬─cached─┬─uncached─┬─total─┐
│ 2020-07-22 14:00:00 │ example.com  │      8 │        0 │     8 │
│ 2020-07-22 14:00:00 │ example.com  │  16531 │        0 │ 16531 │
└─────────────────────┴──────────────┴────────┴──────────┴───────┘
```

Так как clickhouse еще не сделал Merge для timestamp 
```text
┌───────────timestamp─┬─domain───────┬─cached─┬─uncached─┬─total─┐
│ 2020-07-22 14:00:00 │ example.com  │      8 │        0 │     8 │
│ 2020-07-22 14:00:00 │ example.com  │  16531 │        0 │ 16531 │
└─────────────────────┴──────────────┴────────┴──────────┴───────┘
```

То правильнее будет наверно делать запрос так
```text
SELECT
    timestamp,
    sum(cached) AS cached,
    sum(uncached) AS uncached,
    sum(total) AS total
FROM vector.data_domain_traffic
WHERE domain = 'vhost1'
GROUP BY timestamp
ORDER BY timestamp ASC;

┌───────────timestamp─┬─cached─┬─uncached─┬─total─┐
│ 2020-07-22 13:00:00 │  34370 │        0 │ 34370 │
│ 2020-07-22 14:00:00 │  19787 │        0 │ 19787 │
└─────────────────────┴────────┴──────────┴───────┘
```

