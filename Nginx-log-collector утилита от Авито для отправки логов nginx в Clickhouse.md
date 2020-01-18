В этой статье будет рассматриваться проект [nginx-log-collector](https://github.com/avito-tech/nginx-log-collector), который будет читать логи nginx, отправлять их в кластер clickhouse. Для просмотра аналитики по логам создадим дашборд для Grafana.  
<cut />
Устанавливаем nginx, grafana стандартным способом.

Устанавливаем кластер clickhouse с помощью ansible-playbook от [Дениса Проскурина](https://daybydayz.ru/2018/02/05/ансибл-роль-для-установки-clickhouse-и-zookeper/).

### Создание бд и таблиц в clickhouse

В этом [файле](https://github.com/avito-tech/nginx-log-collector/blob/master/etc/examples/clickhouse/table_schema.sql) описаны SQL запросы для создания бд и таблиц для nginx-log-collector в clickhouse.

Каждый запрос делаем поочередно на каждом сервере кластера Clickhouse.

Важное замечание. В этой строке logs_cluster нужно заменить на ваше название кластера из файла  clickhouse_remote_servers.xml между "remote_servers" and "shard".

```bash
ENGINE = Distributed('logs_cluster', 'nginx', 'access_log_shard', rand())
```

### Устанавливка и настройка nginx-log-collector-rpm

Nginx-log-collector не имеет rpm. Здесь https://github.com/patsevanton/nginx-log-collector-rpm создаем ему rpm. Собираться rpm будет с помощью [Fedora Copr](https://copr.fedorainfracloud.org/coprs/antonpatsev/nginx-log-collector-rpm/) 

Устанавливаем rpm пакет nginx-log-collector-rpm

```bash
yum -y install yum-plugin-copr
yum copr enable antonpatsev/nginx-log-collector-rpm
yum -y install nginx-log-collector
systemctl start nginx-log-collector
```

Правим конфиг /etc/nginx-log-collector/config.yaml:

```
  .......
  upload:
    table: nginx.access_log
    dsn: http://ip-адрес-кластера-clickhouse:8123/

- tag: "nginx_error:"
  format: error  # access | error
  buffer_size: 1048576
  upload:
    table: nginx.error_log
    dsn: http://ip-адрес-кластера-clickhouse:8123/
```

### Настройка nginx

Общий конфиг nginx:

```nginx
user  nginx;
worker_processes  auto;

#error_log  /var/log/nginx/error.log warn;
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

    log_format avito_json escape=json
                     '{'
                     '"event_datetime": "$time_iso8601", '
                     '"server_name": "$server_name", '
                     '"remote_addr": "$remote_addr", '
                     '"remote_user": "$remote_user", '
                     '"http_x_real_ip": "$http_x_real_ip", '
                     '"status": "$status", '
                     '"scheme": "$scheme", '
                     '"request_method": "$request_method", '
                     '"request_uri": "$request_uri", '
                     '"server_protocol": "$server_protocol", '
                     '"body_bytes_sent": $body_bytes_sent, '
                     '"http_referer": "$http_referer", '
                     '"http_user_agent": "$http_user_agent", '
                     '"request_bytes": "$request_length", '
                     '"request_time": "$request_time", '
                     '"upstream_addr": "$upstream_addr", '
                     '"upstream_response_time": "$upstream_response_time", '
                     '"hostname": "$hostname", '
                     '"host": "$host"'
                     '}';

    access_log     syslog:server=unix:/var/run/nginx_log.sock,nohostname,tag=nginx avito_json; #ClickHouse
    error_log      syslog:server=unix:/var/run/nginx_log.sock,nohostname,tag=nginx_error; #ClickHouse

    #access_log  /var/log/nginx/access.log  main;
    
    proxy_ignore_client_abort on;
    sendfile        on;
    keepalive_timeout  65;
    include /etc/nginx/conf.d/*.conf;
}

```

Виртуальный хост один:

vhost1.conf:

```
upstream backend {
    server ip-адрес-сервера-с-stub_http_server:8080;
    server ip-адрес-сервера-с-stub_http_server:8080;
    server ip-адрес-сервера-с-stub_http_server:8080;
    server ip-адрес-сервера-с-stub_http_server:8080;
    server ip-адрес-сервера-с-stub_http_server:8080;
}

server {
    listen   80;
    server_name vhost1;
    location / {
        proxy_pass http://backend;
    }
}
```

Добавляем в файл /etc/hosts виртуальные хосты:

```bash
ip-адрес-сервера-с-nginx vhost1
```

### Эмулятор HTTP сервера

В качестве эмулятора HTTP сервера будем использовать [nodejs-stub-server](https://github.com/maxiko/nodejs-stub-server) от [Maxim Ignatenko](https://habr.com/users/Anthrax_Beta/)

Nodejs-stub-server не имеет rpm. Здесь https://github.com/patsevanton/nodejs-stub-server создаем ему rpm. Собираться rpm будет с помощью [Fedora Copr](https://copr.fedorainfracloud.org/coprs/antonpatsev/nodejs-stub-server/) 

Устанавливаем на upstream nginx rpm пакет nodejs-stub-server

```bash
yum -y install yum-plugin-copr
yum copr enable antonpatsev/nodejs-stub-server
yum -y install stub_http_server
systemctl start stub_http_server
```

### Нагрузочное тестирование

Тестирование проводим с помощью Apache benchmark.

Устанавливаем его:

```bash
yum install -y httpd-tools
```

Запускаем тестирование с помощью Apache benchmark c 5 разных серверов:

```
while true; do ab -H "User-Agent: 1server" -c 10 -n 10 -t 10 http://vhost1/; sleep 1; done
while true; do ab -H "User-Agent: 2server" -c 10 -n 10 -t 10 http://vhost1/; sleep 1; done
while true; do ab -H "User-Agent: 3server" -c 10 -n 10 -t 10 http://vhost1/; sleep 1; done
while true; do ab -H "User-Agent: 4server" -c 10 -n 10 -t 10 http://vhost1/; sleep 1; done
while true; do ab -H "User-Agent: 5server" -c 10 -n 10 -t 10 http://vhost1/; sleep 1; done
```

### Настройка Grafana

На официальном сайте Grafana вы не найдете дашборд. 

Поэтому будем делать его вручую.

Мой сохраненный дашборд вы можете найти [тут](https://gist.github.com/patsevanton/e6fb0dfda1dfcf97efb6d762088bda18).

Так же вам нужно создать переменную table с содержимым `nginx.access_log`.
![](https://habrastorage.org/webt/hs/ta/jd/hstajdx3bnaqy07jxy96lj5wpjw.png)

Singlestat Total Requests:

```sql
SELECT
 1 as t,
 count(*) as c
 FROM $table
 WHERE $timeFilter GROUP BY t
```

![](https://habrastorage.org/webt/og/sg/ku/ogsgkulqfaxoiivh3xtmi6nvxd4.png)

Singlestat Failed Requests:

```sql
SELECT
 1 as t,
 count(*) as c
 FROM $table
 WHERE $timeFilter AND status NOT IN (200, 201, 401) GROUP BY t
```

![](https://habrastorage.org/webt/am/sr/fs/amsrfs0u3nyvtx3gohufkzdltfa.png)

Singlestat Failing Percent:

```SQL
SELECT
 1 as t, (sum(status = 500 or status = 499)/sum(status = 200 or status = 201 or status = 401))*100 FROM $table
 WHERE $timeFilter GROUP BY t
```

![](https://habrastorage.org/webt/az/xd/w7/azxdw7c3kkguncr5a8ujbtgvdb4.png)

Singlestat Avg Response Time:

```SQL
SELECT
 1, avg(request_time) FROM $table
 WHERE $timeFilter GROUP BY 1
```

![](https://habrastorage.org/webt/5k/gu/l9/5kgul9tbdow2cnzjwvh5yrhm-rm.png)

Singlestat Max Response Time:

```SQL
SELECT
 1 as t, max(request_time) as c
 FROM $table
 WHERE $timeFilter GROUP BY t
```

![](https://habrastorage.org/webt/wl/2b/qd/wl2bqdydk9osyhxodtlnooet80o.png)

Count Status:

```sql
$columns(status, count(*) as c) from $table
```

![](https://habrastorage.org/webt/rg/qg/s_/rgqgs_21jdozcvpcvfv6-znexve.png)

Для вывода данных как пирог, нужно установить плагин и перезагрузить grafana.

```bash
grafana-cli plugins install grafana-piechart-panel
service grafana-server restart
```

Pie TOP 5 Status:

```sql
SELECT
    1, /* fake timestamp value */
    status,
    sum(status) AS Reqs
FROM $table
WHERE $timeFilter
GROUP BY status
ORDER BY Reqs desc
LIMIT 5
```

![](https://habrastorage.org/webt/qo/3y/v1/qo3yv1ll-drlzbhgfr93m1tzwfg.png)

Дальше буду приводить запросы без скриншотов:

Count http_user_agent:

```sql
$columns(http_user_agent, count(*) c) FROM $table
```

GoodRate/BadRate:

```sql
$rate(countIf(status = 200) AS good, countIf(status != 200) AS bad) FROM $table
```

Response Timing:

```sql
$rate(avg(request_time) as request_time) FROM $table
```

Upstream response time (время ответа 1-го  upstream):

```sql
$rate(avg(arrayElement(upstream_response_time,1)) as upstream_response_time) FROM $table
```

Table Count Status for all vhost:

```sql
$columns(status, count(*) as c) from $table
```

Вывод:

Надеюсь, разработчики сделают официальный Grafana дашборд.

Надеюсь, сообщество подключится к разработке/тестированию и использованию nginx-log-collector.

Telegram каналы:
[Clickhouse](https://t.me/clickhouse_ru) 
[Nginx](https://t.me/nginx_ru)
[Церковь метрик](https://t.me/metrics_ru)

Вопрос к читателям: Если вы храните метрики в clickhouse, то какими self-hosted утилитами/проектами вы получаете метрики по статус кодам и по виртуальным хостам с Nginx?

