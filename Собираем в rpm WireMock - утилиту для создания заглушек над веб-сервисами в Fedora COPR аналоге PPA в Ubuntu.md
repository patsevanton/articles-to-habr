[WireMock](https://github.com/tomakehurst/wiremock) - утилита, библиотека на java для создания заглушек и насмешек над веб-сервисами. Он создает HTTP-сервер, к которому мы могли бы подключиться, как к реальному веб-сервису.

[Fedora COPR](http://copr.fedorainfracloud.org) - это бесплатный хостинг для размещения пользовательских репозиториев (аналог AUR в Arch Linux или PPA в Ubuntu). Из особенностей встроенная возможность собирать rpm пакеты указав имя PIP и RubyGems.

В этом посте напишу как собирать rpm из вашего репозитория по коммиту в [Fedora COPR](http://copr.fedorainfracloud.org).

Для примера возьмем репозиторий **[wiremock-rpm](https://github.com/patsevanton/wiremock-rpm)**, в котором я создал файлы spec и systemd unit. При создании вашего репозитория, вы можете взять **[wiremock-rpm](https://github.com/patsevanton/wiremock-rpm)** за основу и поменять то что вам нужно. Написание файлов spec и systemd unit я опускаю. Думаю что вы знаете.

Создание проекта в [Fedora COPR](http://copr.fedorainfracloud.org)

Заходим в [Fedora COPR](http://copr.fedorainfracloud.org).

![](https://habrastorage.org/webt/4w/lt/dz/4wltdzsujvnqe4veuclvm69_5y0.png)

**Project Name**: Указываем название пакета. Обычно совпадает с названием вашего git репозитория.

**Description**: Краткое описание проекта.

**Instructions**: Как установить ваш пакет. Указывайте следующее:

```
yum -y install yum-plugin-copr

yum copr enable ваша-учетка-в-fedora-copr/название-проекта

yum -y install название-пакета

systemctl start название-пакета
```

**Homepage**: Указываем домашнюю страницу той программы, которую вы хотите собрать или ваш репозиторий.

![](https://habrastorage.org/webt/zp/yz/sy/zpyzsylltgccvcevdclqrcyu1mo.png)

**Build options**: В Chroots указывем под какую оперционную систему вы хотите собрать пакет.

![](https://habrastorage.org/webt/nd/zt/o6/ndzto6phhvt-ff6wm9xlg53a8g4.png)

Получается как на скриншоте.

![](https://habrastorage.org/webt/bv/1u/xe/bv1uxe-xlqq9m5sguv45qnhqll8.png)

**Other options**: Если вам нужен интернет, то в Booleans поставьте галочку *Enable internet access during builds* 

![](https://habrastorage.org/webt/us/wz/5t/uswz5tg0asbs-g2lviqwvykfade.png)

После создания проекта переходим в **Packages**.

![](https://habrastorage.org/webt/da/vi/jp/davijpuooqd2feuchcnip3ws10i.png)

**Package name:** Указываем имя пакета.

**Clone url:** Указываем git репозиторий.

**Subdirectory:** Этим пунктом лучше не пользоваться и держать исходники и spec файл в корне проекта. Если у вас будут исходники в какой-то директории, а spec файл в другой директории, возможны проблемы при сборке.

**Spec File:** Путь до spec файла.

![](https://habrastorage.org/webt/h9/tx/bg/h9txbg-vbfze1rqec4sreunzi2u.png)

**Generic package setup**: Обязательно поставьте галочку *Auto-rebuild*

![](https://habrastorage.org/webt/xk/h2/f2/xkh2f24aegiiazzr1ym73i3tswk.png)

После того как создали пакет идем в *Settings*, далее в Integrations. Ниже на странице копируем webhook той системы, где у вас расположен ваш git репозиторий.

![](https://habrastorage.org/webt/f0/0g/es/f00gest0bm1kk-6gttm42aqsphm.png)

Идем в Settings где у вас расположен ваш git репозиторий. Далее для github идем в webhook. Добавляем webhook. Вставляем Payload URL, выбираем Content type *application/json*

**Теперь про WireMock.**

Установим wiremock согласно инструкции. Рабочая директория wiremock в моем проекте  - `/usr/lib/wiremock`/. В этой директории лежит wiremock.jar и директория mappings. В директории mappings находятся json файлы с запросами, которые вы отправляете к wiremock и подготовленными ответами.

Пример из http://wiremock.org/docs/running-standalone/:

```
{
    "request": {
        "method": "GET",
        "url": "/api/mytest"
    },
    "response": {
        "status": 200,
        "body": "More content\n"
    }
}
```

Отправляем запрос к /api/mytest и получаем:

```
curl http://localhost:8080/api/mytest
More content
```

Пример из моих подготовленных json:

```
{
    "request": {
        "method": "GET",
        "url": "/503"
    },
    "response": {
        "status": 503,
        "body": "503 Service Unavailable\n"
    }
}
```

Сделаем запрос к /503

```
curl -i -v 172.26.9.123:8080/503
* About to connect() to 172.26.9.123 port 8080 (#0)
*   Trying 172.26.9.123...
* Connected to 172.26.9.123 (172.26.9.123) port 8080 (#0)
> GET /503 HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 172.26.9.123:8080
> Accept: */*
> 
< HTTP/1.1 503 Service Unavailable
HTTP/1.1 503 Service Unavailable
< Matched-Stub-Id: d8b419e1-7e33-4f04-889e-2428f849dc7d
Matched-Stub-Id: d8b419e1-7e33-4f04-889e-2428f849dc7d
< Transfer-Encoding: chunked
Transfer-Encoding: chunked
< Server: Jetty(9.2.z-SNAPSHOT)
Server: Jetty(9.2.z-SNAPSHOT)

< 
503 Service Unavailable
```
