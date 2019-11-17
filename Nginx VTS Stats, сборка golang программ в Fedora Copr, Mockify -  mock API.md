В этом посте я хотел сделать демо Nginx VTS + Prometheus + Grafana. Для демо необходимо было чтобы upstream могли выдавать разные http коды. Это могли сделать следующие проекты: Mockify, написанный на Golang, и WireMock, написанный на Java.
 - установка и настройка Nginx VTS + Prometheus + Grafana
 - сборка golang программ в Fedora Copr
 - Mockify - легкое, конфигурируемое эмулирование API, написанное на Golang
 - Сравнение использования CPU для Mockify, написанный на Golang, и WireMock, написанный на Java.

Тестовый стенд виртуальная машина:

```bash
inxi
CPU: 8x Single Core Intel Xeon E312xx (Sandy Bridge) (-SMP-) speed: 2594 MHz Kernel: 3.10.0-957.1.3.el7.x86_64 x86_64 Up: 58m 
Mem: 474.9/32011.6 MiB (1.5%) Storage: 80.00 GiB (2.7% used) Procs: 149 Shell: bash 4.2.46 inxi: 3.0.35 
```

Конфиг prometheus:

```
global:
  scrape_interval:     5s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 5s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']
  - job_name: ‘nginx_vts_exporter’
    static_configs:
    - targets: ['localhost:9913']
```

Конфиг Grafana стандартный. ID дашборда 2949.

Для Nginx VTS нужно скомпилировать nginx с модулем nginx-module-vts. Делаем это с помощью [Nginx-builder](https://habr.com/ru/company/tinkoff/blog/452336/). Его конфиг:

```
nginx_version: 1.16.1
output_package: rpm
modules:
  - module:
      name: nginx-module-vts
      git_url: https://github.com/vozlt/nginx-module-vts.git
      git_tag: v0.1.18
```

Устанавливаем собранный nginx. Вот его главный конфиг (не забываем указать vhost_traffic_status_zone;):

```
user  nginx;
worker_processes  auto;
worker_rlimit_nofile 40960;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    use epoll;
    worker_connections 1024;
    multi_accept on;
}

http {
    vhost_traffic_status_zone;
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log /var/log/nginx/access.log main;
    access_log off;
    sendfile on;
    tcp_nodelay on;
    tcp_nopush on;
    keepalive_timeout  65;
    include /etc/nginx/conf.d/*.conf;
    open_file_cache max=200000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
}
```

Создаем конфиги виртуальных хостов:

```
touch vhost1.conf  vhost2.conf  vhost3.conf  vhost4.conf  vhost5.conf
```

Их содержимое:

vhost1.conf:

```
server {
        listen   80;
        server_name vhost1;
        location / {
                proxy_pass http://127.0.0.1:8001/102;
        }
}
```
vhost2.conf:
```
server {
        listen   80;
        server_name vhost2;
        location / {
                proxy_pass http://127.0.0.1:8001/204;
        }
}
```
vhost3.conf:
```
server {
        listen   80;
        server_name vhost3;
        location / {
                proxy_pass http://localhost:8001/304;
        }
}
```
vhost4.conf:
```
server {
        listen   80;
        server_name vhost4;
        location / {
                proxy_pass http://localhost:8001/403;
        }
}
```
vhost5.conf:
```
server {
        listen   80;
        server_name vhost5;
        location / {
                proxy_pass http://localhost:8001/503;
        }
}
```


Устанавливаем nginx-vts-exporter:

```
wget https://github.com/hnlq715/nginx-vts-exporter/releases/download/v0.10.3/nginx-vts-exporter-0.10.3.linux-amd64.tar.gz
tar zxvf nginx-vts-exporter-0.10.3.linux-amd64.tar.gz
cp nginx-vts-exporter-0.10.3.linux-amd64/nginx-vts-exporter /usr/local/bin/nginx-vts-exporter
```

Создаем файл /etc/systemd/system/nginx_vts_exporter.service:

```
[Unit]
Description=Nginx vts exporter  
Wants=network-online.target
After=network-online.target
[Service]
ExecStart=/usr/local/bin/nginx-vts-exporter -nginx.scrape_uri=http://localhost:7070/status/format/json
Restart=always
RestartSec=3
[Install]
WantedBy=default.target
```

Создаем файл конфигурации VTS /etc/nginx/conf.d/vts-exporter.conf

```
server {
 listen 7070;
 listen [::]:7070;

 location / {
 }

 location /status {
   vhost_traffic_status_display;
   vhost_traffic_status_display_format html; 
 }

}
```

Добавляем DNS виртуальных хостов в /etc/hosts:

```
127.0.0.1 vhost1
127.0.0.1 vhost2
127.0.0.1 vhost3
127.0.0.1 vhost4
127.0.0.1 vhost5
```

