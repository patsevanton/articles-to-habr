Сделал dashboard Postgresql overview для [postgres_exporter](https://github.com/wrouesnel/postgres_exporter/blob/master/queries.yaml).

Чем отличается от других дашбородов postgres_exporter?

Я объединил все другие дашборды postgres_exporter в один.

Пока что это дашбор показывает общую информацию по кластеру.

cut

Краткая инструкция по использованию.

### PostgreSQL

```
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
yum install -y postgresql96 postgresql96-server postgresql96-contrib
```

Инициализируем PostgreSQL.

```
/usr/pgsql-9.6/bin/postgresql96-setup initdb
Initializing database ... OK
```

Стартуем PostgreSQL

```
systemctl start postgresql-9.6
```

### Postgres_exporter и Prometheus

Postgres_exporter и Prometheus для Redhat систем устанавливаем из этого репозитория: https://github.com/lest/prometheus-rpm

Создаем файл `/etc/yum.repos.d/prometheus.repo` со следующим содержимым:

```
[prometheus]
name=prometheus
baseurl=https://packagecloud.io/prometheus-rpm/release/el/$releasever/$basearch
repo_gpgcheck=1
enabled=1
gpgkey=https://packagecloud.io/prometheus-rpm/release/gpgkey
       https://raw.githubusercontent.com/lest/prometheus-rpm/master/RPM-GPG-KEY-prometheus-rpm
gpgcheck=1
metadata_expire=300
```

Устанавливаем prometheus2 и postgres_exporter

```
yum install -y prometheus2 postgres_exporter
```

Запускаем prometheus2 и postgres_exporter

```
systemctl start prometheus2 
systemctl start postgres_exporter
```

### Grafana

Создаем файл `/etc/yum.repos.d/grafana.repo` со следующим содержимым:

```
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
```

Устанавливаем grafana

```
yum -y install grafana initscripts urw-fonts wget
```

Запускаем grafana

```
systemctl start grafana-server
```



Берем dashboard здесь

https://github.com/patsevanton/postgresql_overview_postgres_exporter



P.S. В этом дашборде мне не хватает знаний в promql и postgresql. Поэтому я надеюсь на то что вы мне поможете советом как улучшить дашборд или сделаете pull request.



P.S. Как руки дойдут, планирую сделать дашборд для информации по конретной БД внутри PostgreSQL.