Сделал dashboard Postgresql overview для [postgres_exporter](https://github.com/wrouesnel/postgres_exporter/blob/master/queries.yaml).

Чем отличается от других дашбородов postgres_exporter?

Я объединил все другие дашборды postgres_exporter в один.

Пока что это дашбор показывает общую информацию по кластеру.

cut

Почему бы не использовать [pgwatch2]() c [influxdb](https://github.com/influxdata/influxdb)?

Ответ под спойлером:

# Про InfluxDB

Вот краткий и неполный список проблем на момент версии 1.7 (применимо и к более младшим и скорее всего к старшим, CORE team не поменялась):

- Стабильность. Периодически падает и теряет данные или ломает данные на диске. В последнем случае не может подняться или не может сделать компакшен, от чего количество открытых файлов улетает в космос. Лечится полной остановкой DB и выполнением [команд](https://docs.influxdata.com/influxdb/v1.8/tools/influx_inspect/) в надежде, что хоть одна поможет.
- Скорость. Заявленное в маркетинговых бумажках касается не постоянного рейта, а спайков.
- Не работают внутренние лимиты на запросы вида `SHOW TAG KEYS FROM ALL` или `SHOW EXACT SERIES CARDINALITY` и на средней базе может положить все.
- Потребление ресурсов. Сожрать 256ГБ RAM, закусить 320GB свопа и все равно упасть по OOMу -- легко (в момент 6-ти часового запуска, который обусловлен тем, что при старте он читает с диска все индексы в память(InMem)).
- Платная кластеризация (была представлена как часть OSS в версии [0.9 (December 8, 2014)](https://www.influxdata.com/blog/clustering-tags-and-enhancements-to-come-in-0-9-0/) и исчезла в [1.0 (September 26, 2014)](https://www.influxdata.com/blog/one-year-of-influxdb-and-the-road-to-1-0/), став привилегией Enterprise версии).
- Частые breaking changes. За 3 года сменили 5+ движков (закончили это делать на версии [0.9 (December 8, 2014)](https://www.influxdata.com/blog/clustering-tags-and-enhancements-to-come-in-0-9-0/)). Следующий Breaking Changes - это Influx 2.0, где они ушли от База Данных\Ретеншн полиси в сторону [Buckets](https://v2.docs.influxdata.com/v2.0/reference/key-concepts/data-elements/#bucket), поменяли язык запросов на [Flux](https://v2.docs.influxdata.com/v2.0/reference/flux/).
- Периодически выкатывают фичи непонятно зачем сделанные, например сделали [ifql](https://www.influxdata.com/blog/announcing-ifql-v0-0-3/) (Flux) или [Continuous Queries](https://docs.influxdata.com/influxdb/v1.8/query_language/continuous_queries/) (последние выпилили в пользу [task](https://v2.docs.influxdata.com/v2.0/process-data/common-tasks/downsample-data/), по факту те же яйца только с Flux-ом) или Chronograf(буква C в TICK), при живой то графане.
- Безалаберность при подготовке релизов.
- Не самосогласованные утилиты экспорта и импорта из базы - если вы что-то экспортировали через cli, то импортировать обратно файлик не прокатит. restore из backup полностью заменяет всю метаинформацию о базах. Селективности и merge не завезли.
- Телеграф как часть платформы TICK(буква T), например, они ломали поддержку Прометея в телеграфе 1.3.2 ([замена символов](https://github.com/influxdata/telegraf/issues/2937) не попадающих под `[a-z]`). Или, например, невозможность оверрайдить Retention Policy в (input,output).kafka, т.е. организовать полноценную связку `metrics -> telegraf -> kafka -> telegraf -> influx` у вас не получится.
- Капаситор(бука K в TICK), очень неадекватно себя ведет, подстать InfluxDB. Выжирает RAM как не в себя, может говорить, что всё "ок", когда данных нет. Требует нежного обращения и ухода.

### PostgreSQL

```
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
yum install -y postgresql96 postgresql96-server postgresql96-contrib
```

Инициализируем PostgreSQL.

```
/usr/pgsql-9.6/bin/postgresql96-setup initdb
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

В файле prometheus.yml для работы с postgres_exporter в scrape_configs добавьте следующую секцию:

```
scrape_configs:
  - job_name: postgresql
    static_configs:
      - targets: ['ip-адрес-prometheus:9187']
        labels:
          alias: postgres
```



Запускаем prometheus2 и postgres_exporter

```
systemctl start prometheus
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


![](https://habrastorage.org/webt/in/de/eu/indeeudff0os9fzdqdfhrpdboby.jpeg)

![](https://habrastorage.org/webt/tr/81/ug/tr81ugngobhru0miibe7bgt4rp8.jpeg)

![](https://habrastorage.org/webt/b0/yo/q-/b0yoq-rwfingvptkzqdrwsdzjvi.jpeg)

![](https://habrastorage.org/webt/km/sb/yc/kmsbycvyy5zpgphtlq8enzefob4.jpeg)

![](https://habrastorage.org/webt/xm/sd/4j/xmsd4jhin8-_a2ywmehg7ohir_8.jpeg)

![](https://habrastorage.org/webt/zg/me/xz/zgmexzb97zio0xcr4gvbnvyeoe0.jpeg)

![](https://habrastorage.org/webt/3f/sa/oo/3fsaoof3obeka8o0cjeiv2famcw.jpeg)

![](https://habrastorage.org/webt/dy/zm/nz/dyzmnzfmgnbs09kaawhjjfcrqrw.jpeg)

![](https://habrastorage.org/webt/_j/nr/qr/_jnrqr7jzo5jmixsyudgblw9zte.jpeg)

![](https://habrastorage.org/webt/e6/w-/p3/e6w-p3if6zw4kug1qi327uj8nqc.jpeg)



P.S. В этом дашборде мне не хватает знаний в promql и postgresql. Поэтому я надеюсь на то что вы мне поможете советом как улучшить дашборд или сделаете pull request.



P.S. Как руки дойдут, планирую сделать дашборд для информации по конретной БД внутри PostgreSQL.

P.S. Хочется получить вот такую схему

![](https://habrastorage.org/webt/n_/06/at/n_06atgmuesvmbsbukc6zegedmw.png)