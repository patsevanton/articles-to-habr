Составление отчетов

Чтобы защитить вашу корпоративную сеть от угроз и атак, вы всегда должны выполнять тест на уязвимости в своей системе. Для того, чтобы их исправить. Итак, как вы понимаете, работа с отчетами очень важна для любого SOC, потому что она дает обзор уязвимостей, которые могут быть в вашей системе.

В этой статье мы расскажем вам об инструменте, который мы использовали для создания отчетов и сканирования уязвимостей.


Эта статья разделена на следующие разделы:

- Введение
- Установка Nessus Essentials
- Установка VulnWhisperer

### 1. Введение:

Инструменты, которые мы будем использовать:

- **VulnWhisperer**: VulnWhisperer - это инструмент управления уязвимостями и агрегатор отчетов. VulnWhisperer извлечет все отчеты из различных сканеров уязвимостей и создаст файл с уникальным именем для каждого из них.

URL проекта: https://github.com/HASecuritySolutions/VulnWhisperer

- **Nessus Essentials** (ранее Nessus Home) - это бесплатная версия сканера уязвимостей Nessus.

### 2. Установка необходимых компонентов Nessus:

#### 2.1- Скачать с официального сайта (www.tenable.com), в нашем проекте мы использовали эту версию:

![](https://habrastorage.org/webt/no/5l/yq/no5lyqf80rmp81espaew-znhy3g.png)

#### 2.2 - Установка Nessus:

```
dpkg -i Nessus-8.10.0-ubuntu910_amd64.deb
/etc/init.d/nessusd start
service nessusd start
```

Перейдите по адресу https: // YourServerIp: 8834 и выберите Nessus Essentials.

![](https://habrastorage.org/webt/k5/rv/xf/k5rvxfskpu4simpwz2__dlh-9tw.jpeg)

#### 2.3- Начните с Nessus

![](https://habrastorage.org/webt/s4/fe/bx/s4febxdpyi4hselpgm7uz39vs80.png)


Скопируйте код активации, создайте учетную запись и подождите, пока Nessus подготовит файлы.

2.4-Запустите первое сканирование:

Перейдите в New Scan и выберите Basic Network Scan.

![](https://habrastorage.org/webt/-7/8d/e-/-78de-21kw1_rsxga2fiejlm2ye.png)


Выберите цель, сохраните и запустите:

![](https://habrastorage.org/webt/qx/nx/ha/qxnxhax2pf13oop9jwasmqhy97i.png)

![](https://habrastorage.org/webt/ug/-n/1l/ug-n1lvaflmc40rmq0-tdhwenpi.png)

### 3.Установка VulnWhisperer:

#### 3.1- Используйте Python2.7:

**ПРИМЕЧАНИЕ: VulnWhisperer требует Python2.7, поэтому мы изменим нашу версию Python по умолчанию.**

![](https://habrastorage.org/webt/pw/rh/rl/pwrhrlm-6jew8_p5hygx2uic2vc.png)

#### 3.2- Настроить VulnWhisperer:

```
cd /etc/
git clone https://github.com/HASecuritySolutions/VulnWhisperer
cd VulnWhisperer/
sudo apt-get install zlib1g-dev libxml2-dev libxslt1-dev
pip install -r requirements.txt
python setup.py install
nano configs/ frameworks_example.ini
```

Выберите модули, которые вы хотите включить (в нашем случае мы просто включим Nessus), и напишите данные своей учетной записи Nessus:

![](https://habrastorage.org/webt/ym/dz/bi/ymdzbim8rkskwm22oopdql44wxu.png)

#### 3.3 - Проверьте соединение Nessus и загрузите отчет:

```
*vuln_whisperer -F -c configs/frameworks_example.ini -s nessus*

*Reports will be saved with csv extension.Check them under: /opt/VulnWhisperer/data/nessus/My\ Scans/*
```

![](https://habrastorage.org/webt/em/sh/4e/emsh4epqsok9r4_v1onlngfdbbq.png)

Если нового отчета нет, вы увидите

![](https://habrastorage.org/webt/62/4o/di/624odiwzwprrlk1zmcsd5vlrtzw.png)

#### 3.4- Cronjob с Vulnwhisperer:

Чтобы Vulnwhisperer периодически проверял базу данных Nessus и загружал отчеты, мы добавим задание cron. Таким образом, нам больше не нужно будет выполнять эту команду вручную. Новые отчеты будут автоматически добавляться в Kibana.

```
crontab –e
```

Добавь это:

```
SHELL=/bin/bash

* * * * * /usr/local/bin/vuln_whisperer -c /etc/VulnWhisperer/configs/frameworks_example.ini >/dev/null 2>&1
```

![](https://habrastorage.org/webt/dt/ja/it/dtjaitup1nnnuqw1wugclwb4isk.png)

#### 3.5-Импорт шаблонов Elasticsearch:

Зайдите в kibana Dev Tools и добавьте шаблон:


Ссылка на файл:

https://github.com/HASecuritySolutions/VulnWhisperer/blob/master/resources/elk6/logstash-vulnwhisperer-template_elk7.json

![](https://habrastorage.org/webt/_x/ul/16/_xul16blvcw6wlwdqz2qlseycpa.png)


Теперь у вас будет шаблон индекса

![](https://habrastorage.org/webt/xd/lg/ph/xdlgphmsyrh2xy00dwgtwfwybly.png)

#### 3.6- Импорт визуализаций Кибаны

Перейдите в Kibana → Management → saved object → Import

Импортируйте конфигурацию kibana.json:

*Ссылка на файл:*

[*https://github.com/HASecuritySolutions/VulnWhisperer/blob/master/resources/elk6/kibana.json*](https://github.com/HASecuritySolutions/VulnWhisperer/blob/master/resources/elk6/kibana.json)

(Непонятно почему здесь elk6 - прим. переводчика)

![](https://habrastorage.org/webt/on/6y/ma/on6ymajoi_wzntq43qkt7p88mao.png)

Теперь под панелями мониторинга у вас есть:

![](https://habrastorage.org/webt/t0/of/3m/t0of3mvgca_sm2vh76e6t2vfzo8.png)

#### 3.7 -Добавление файла конфигурации Nessus Logstash

Скопируйте файл журнала Nessys в /etc/logstash/conf.d/:

```
cd /etc/VulnWhisperer/resources/elk6/pipeline/
cp 1000_nessus_process_file.conf /etc/logstash/conf.d/
cd /etc/logstash/conf.d/
nano 1000_nessus_process_file.conf
```

Изменить вывод

![](https://habrastorage.org/webt/ck/p5/x2/ckp5x2gpz5x2dmveb3lkzvarjyo.png)

#### 3.8- Перезапустите свои службы и проверьте отчеты:

```
systemctl restart logstash elasticsearch
```

![](https://habrastorage.org/webt/p9/86/bx/p986bxkqejnus1tu-dm7vzxpj8u.png)

Теперь у вас должен быть создан новый индекс для Vulnwhisperer.


Перейдите к шаблону индекса и проверьте количество ваших полей:

**ПРИМЕЧАНИЕ: обновите шаблон индекса, чтобы распознать все поля.**

![](https://habrastorage.org/webt/li/cq/31/licq318q-gog0oq6hsl-wqddzxe.png)


Наконец, перейдите на панели мониторинга и проверьте свои отчеты

У вас не должно быть ошибок в визуализации.

![](https://habrastorage.org/webt/vn/gr/ik/vngriklcnoelq1qbadqsxsbmk_s.png)

![](https://habrastorage.org/webt/xp/3m/7x/xp3m7xq9m8ckbhxtwdcw7tbrici.png)


Теперь все отчеты, созданные nessus с расширениями csv, будут автоматически отправляться в ваш стек ELK, чтобы вы могли визуализировать их на панелях мониторинга kibana.
