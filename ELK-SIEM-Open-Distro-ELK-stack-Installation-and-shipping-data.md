ELK SIEM Open Distro: ELK stack - данные по установке и настройке.

Оглавление всех постов.

- Введение. Развертывание инфраструктуры и технологий для SOC как Service (SOCasS)

- ELK stack: данные по установке и доставке
- Open Distro для Elasticsearch
- Визуализация Dashboards и ELK SIEM
- Интеграция с WAZUH
- Оповещение (Alerting)
- Отчетность
- Case Management



**1- Установка и настройка ELK STACK**

**1.1- Введение в ELK**

**A- Что такое ELK?**

https://www.elastic.co/fr/what-is/elk-stack

**B- Разница между ELK Basic и ELK Oss?**

https://www.elastic.co/fr/what-is/open-x-pack

![](https://habrastorage.org/webt/3o/fa/qm/3ofaqmz1rrakt1giy49wclz8jve.png)

**1.2- Установка ELK**

В нашем проекте мы приступили к настройке ELK Stack Basic (7.6.1) и сослались на официальное руководство, предоставленное elastic.co:

https://www.elastic.co/guide/en/elastic-stack/current/installing-elastic-stack.html

![](https://habrastorage.org/webt/rs/px/gt/rspxgtz96y7fkvixoybye_ewhii.png)

![](https://habrastorage.org/webt/qt/l0/vu/qtl0vuu--5zwlyppddlx6iwk3p8.png)



**1.3- Конфигурация ELK**

В этом разделе мы предоставим вам конфигурацию, которую мы сделали для стека ELK.

**A- Конфигурация Elasticsearch**

Все настройки были внесены в файл elasticsearch.yml, расположенный в **/etc/elasticsearch/elasticsearch.yml**

Чтобы открыть его, используйте следующую команду: sudo nano **/etc/elasticsearch/elasticsearch.yml**

Это пути по умолчанию для данных и журналов для elasticsearch.

![](https://habrastorage.org/webt/1p/mr/nf/1pmrnfkb9mmnecfagdglgspkmom.png)

Затем перейдите в сетевой раздел файла. Сетевым разделом пользоваться очень просто, вам даже не нужно упоминать порт, если вы используете порт по умолчанию. Но вы должны изменить http.port и отменить комментарий. если вы собираетесь использовать другой порт.

**network.bind_host: 0.0.0.0** включит удаленный доступ к серверу Elasticsearch, который поможет нам позже подключить биты к стеку ELK.

![](https://habrastorage.org/webt/lg/lr/t3/lglrt3nh7tezqeja9lkt6uwhufy.png)

Как только это будет установлено, нам нужно перезапустить службу ElasticSearch с помощью этой команды:

**sudo systemctl restart elasticsearch**

**Предупреждение**: *установка network.bind_host to 0.0.0.0 не рекомендуется из-за проблем с безопасностью и не должна использоваться на производственном уровне. Мы пока находимся на этапе создания прототипа.*

**Конфигурация B-Kibana:**

Все настройки были внесены в файл kibana.yml, расположенный в **/etc/kibana/kibana.yml**. Чтобы открыть его, используйте следующую команду:

**sudo nano /etc/kibana/kibana.yml**

![](https://habrastorage.org/webt/f6/ht/ry/f6htrykkoofo9bhm9nwmjzryixw.png)

![](https://habrastorage.org/webt/wa/a2/vt/waa2vtfszgb2szbl_fkjdslvjpu.png)

Чтобы сделать Kibana доступным удаленно, мы должны установить **server.host: "0.0.0.0"**. Нет никаких ограничений, когда дело доходит до порта, который должен быть включен. Итак, оставим настройку по умолчанию, равную 5601. Теперь перезапустите Kibana: **sudo systemctl restart kibana**

Теперь у вас должна быть возможность получить доступ к своей Kibana из браузера. Http://your_Server_IP: 5601

![](https://habrastorage.org/webt/zj/hh/kb/zjhhkbirlxdt0gt9dcxco1th2ze.png)

*Пока вы занимаетесь этим, наслаждайтесь потрясающим пользовательским интерфейсом, который он предоставляет, и пробуйте некоторые образцы панелей мониторинга и предоставляемые данные.*

**Предупреждение**: *установка server.host на 0.0.0.0 не рекомендуется из-за серьезных проблем с безопасностью и не должна использоваться на производственном уровне. Мы пока находимся на этапе создания прототипа.*

**Конфигурация C-Logstash:**

Теперь займемся настройкой logstash:

**sudo cat /etc/logstash/logstash-sample.conf**

Этот файл содержит необходимую конфигурацию для Logstash. Итак, нам нужно скопировать его в каталог **/etc/logstash/conf.d/** и изменить его имя на logstash.conf

![](https://habrastorage.org/webt/fy/k0/i6/fyk0i6vmglcsr5mmqscoewh-wg8.png)

*Не забудьте перезапустить службу: **sudo systemctl restart logstash***

D-Проверка сервисов:**

После правильной настройки файлов конфигурации logstash, kibana и elasticsearch. вы можете запустить свои сервисы и проверить их:

![](https://habrastorage.org/webt/jk/e2/nv/jke2nvhswffhzibcz-8pzqei6co.png)

![](https://habrastorage.org/webt/63/cr/yn/63cryncis9pmc3jpkyzqptzhjne.png)

Вы можете проверить, прослушивают ли эти службы свои порты. Неважно, используете ли вы tcp6 вместо tcp.

*Kibana: 5601*

*Elasticsearch: 9200*

*Logstash: 5044*

![](https://habrastorage.org/webt/p6/15/pe/p615peat-jngmmdrjnmyi8ats4e.png)

**Конфигурация 2-Beats и доставка данных:**

**A- Загрузка и установка Winlogbeat:**

Скачать URL:

https://www.elastic.co/fr/downloads/beats/winlogbeat

Установка:

https://www.elastic.co/guide/en/beats/winlogbeat/current/winlogbeat-installation.html

**B- Конфигурация Winlogbeat:**

В нашем проекте мы использовали такую конфигурацию winlogbeat.yml:

![](https://habrastorage.org/webt/dy/kt/bu/dyktbu_nwbilbdv31vk1n2yoc7q.png)

**Разбираемся в winlogbeat.event_logs:**

Раздел winlogbeat в winlogbeat.yml определяет все параметры, специфичные для Winlogbeat. Самое главное, он содержит список журналов событий, которые нужно отслеживать. Мы видим, что модуль Sysmon включен по умолчанию.

Чтобы добавить больше модулей, вы можете изучить информацию по ссылке:

https://www.elastic.co/guide/en/beats/winlogbeat/current/configuration-winlogbeat-options.html

**Разбираемся в нужном количестве шардов и количестве реплик:**

**\- index.number_of_shards:**

Индекс потенциально может хранить большой объем данных, который может превышать аппаратные ограничения одного узла. Чтобы решить эту проблему, Elasticsearch предоставляет возможность разбить ваш индекс на несколько частей, называемых осколками, каждая из которых сохраняется на разных машинах.

**- index.number_of_replicas:**

Эта команда задает количество копий, которое Elasticsearch будет хранить. Эта функция полезна, если у вас есть сеть компьютеров, на которых работает Elasticsearch. Если одна машина выходит из строя, данные не теряются.

Выходы:

![](https://habrastorage.org/webt/r2/bo/i-/r2boi-f5jgt6zy4dfwg-xv-vsgi.png)

![](https://habrastorage.org/webt/2p/-g/e5/2p-ge5s55-ijz-vunqkgarvzj-8.png)

![](https://habrastorage.org/webt/vu/5e/iv/vu5eivrztoyeg_2ovorf1-d7ogk.png)


Для вывода Elasticsearch и вывода Logstash **только один из них должен быть включен** при запуске службы или проверке конфигурации.

**Настройки процессоров и журналирования:**

![](https://habrastorage.org/webt/sy/ls/ar/sylsarjvl9mnkcteesf24l8nz-0.png)

Этот раздел содержит процессоры по умолчанию, используемые winlogbeat, и пример настроек журналирования:

**Управление жизненным циклом индекса (ILM):**

Наконец, нам пришлось отключить ILM. ILM или Index Lifecycle Manager - это бесплатная функция x-pack, интегрированная с базовым стеком ELK, но не с версией ELK oss. Вы можете использовать ILM для автоматического управления индексами в соответствии с вашими требованиями к производительности, отказоустойчивости и срокам хранения. Например: создавайте новый индекс каждый день, неделю или месяц и архивируйте предыдущие, запускайте новый индекс, когда индекс достигает определенного размера, или удаляет устаревшие индексы, чтобы обеспечить соблюдение стандартов хранения данных.

Функция ILM включена по умолчанию для базовой версии стека ELK, но требует дополнительной настройки, особенно если ваши биты не подключены напрямую к Elasticsearch. Функция ILM выходит за рамки этой статьи, поэтому мы отключим ее.

![](https://habrastorage.org/webt/ha/gz/if/hagzifgxr1w2c-zjyp76udpc9se.png)

**Конфигурация и интеграция Sysmon с MITER ATT & CK:**

Мы настроим и сделаем новую конфигурацию Sysmon перед загрузкой шаблонов индекса, чтобы убедиться, что новые поля и конфигурация sysmon правильно загружены в стек ELK.

**Системный монитор (Sysmon)** - это системная служба Windows и драйвер устройства, который, будучи установленным в системе, остается резидентным при перезагрузке системы для отслеживания и регистрации активности системы в журнале событий Windows. Он предоставляет подробную информацию о создании процессов, сетевых подключениях и изменениях времени создания файлов. Собирая события, которые он генерирует с помощью Windows Event Collection или агентов SIEM, а затем анализируя их, вы можете определить вредоносную или аномальную активность и понять, как злоумышленники и вредоносные программы действуют в вашей сети.

**MITER ATT & CK** - это глобально доступная база знаний о тактике и методах противодействия, основанная на реальных наблюдениях. База знаний ATT & CK используется в качестве основы для разработки конкретных моделей угроз и методологий в частном секторе, в правительстве, а также в сообществе продуктов и услуг кибербезопасности.

**I.** Загрузите Sysmon:

https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon

**II.** Загрузите xml-конфигурацию для sysmon, содержащую ссылки MITER ATT и CK: https://raw.githubusercontent.com/ion-storm/sysmon-config/master/sysmonconfig-export.xml

**III.** Установите Sysmon с соответствующим файлом конфигурации: **sysmon64 -accepteula -i sysmonconfig-export.xml**

IV. Проверьте текущую конфигурацию: **sysmon64 –c**



**Настройте шаблон индекса, информационные панели и шаблон индекса:**

**I. Шаблон индекса загрузки:**

Шаблоны индексов позволяют определять шаблоны, которые будут автоматически применяться при создании новых индексов. Шаблоны включают как настройки, так и сопоставления, а также простой шаблон шаблона, который определяет, следует ли применять шаблон к новому индексу.

Для загрузки шаблона индекса требуется подключение к Elasticsearch. Если вывод не является Elasticsearch, вы должны загрузить шаблон вручную. В нашем случае winlogbeat не будет напрямую подключен к Elasticsearch, поэтому нам придется вручную настроить шаблон индекса перед запуском службы.

Требуется отключить вывод Logstash и временно включить вывод Elasticsearch.

**II. Загрузка информационных панелей и шаблонов индекса:**

https://www.elastic.co/guide/en/beats/winlogbeat/current/load-kibana-dashboards.html

**Изменение вывода:**

После правильной загрузки шаблона индекса, шаблона индекса и информационных панелей вы можете проверить их в интерфейсе Kibana.

Шаблон индекса и панели мониторинга загружены успешно:

![](https://habrastorage.org/webt/jf/8n/34/jf8n34d_lc6uduboipwidgpsrb0.png)

![](https://habrastorage.org/webt/ky/5g/wa/ky5gwa_rtgwr4grp3srjb8ysbey.png)

![](https://habrastorage.org/webt/4o/9c/wr/4o9cwravl_8mpasbvyzhnv-n9vs.png)

Теперь, что мы делаем после открытия файла конфигурации. Мы отключим вывод Elasticsearch, комментируя его, затем включим вывод Logstash, удалив комментарий.

**Доставляем данные в стек ELK:**

Теперь мы можем запустить службы winlogbeat и sysmon из PowerShell или с помощью интерфейса services.msc, а также проверить данные в интерфейсе Kibana.

При запуске winlogbeat. ELK STACK будет использовать конфигурацию в Logstash для создания индекса, который позволит хранить данные.

![](https://habrastorage.org/webt/qm/tc/4o/qmtc4ocuwokaupthglatnolmdp8.png)

Это панель управления по умолчанию для winlogbeat:

![](https://habrastorage.org/webt/v6/fa/fu/v6fafug7pewjyhhxbn3vwjd7j-u.png)

В разделе Discover мы можем проверить журналы sysmon с новой конфигурацией (ссылки MITER):

![](https://habrastorage.org/webt/9a/yo/jr/9ayojrpeillui2r4fivk7g-rwju.png)

![](https://habrastorage.org/webt/3h/ww/tt/3hwwttzkfbhoaoqthat36ehkg5s.png)


В остальном биты не сильно отличаются от winlogbeat ни по настройке, ни по установке.

Биты, которые мы использовали:

Winlogbeat

Filebeat

Packetbeat

Metricbeat

Мы хотим упомянуть, что некоторые биты, такие как metricbeat или filebeat, имеют несколько модулей, которые вы можете использовать.

Например, мы использовали модульную систему в filebeat для мониторинга аутентификации ssh, команды sudo на машине ubuntu и мы использовали модуль Suricata для сбора журналов из Suricata IDS.

**Включение модуля Suricata:**

Мы использовали эту команду для включения модуля Suricata в filebeat:

**sudo filebeat modules enable Suricata**

чтобы увидеть модули, доступные в filebeat, вы можете проверить каталог **/etc/filebeat/modules.d/**

Чтобы увидеть активные модули, используйте эту команду:

**filebeat modules list** 

Это ссылка, которую мы использовали для установки Suricata на наше устройство:

https://www.alibabacloud.com/blog/594941

Вы должны получить панель инструментов, примерно похожую на эту. Не беспокойтесь, если вы не получите именно такой результат, мы будем работать с дашбордом в следующих статьях.

![](https://habrastorage.org/webt/3k/20/uy/3k20uy4wziycvo1ufw9t1-bae3i.png)



Также можно интегрировать интерфейс Suricata в стек ELK, для чего вы можете проверить эту ссылку ниже:

https://www.howtoforge.com/tutorial/suricata-with-elk-and-web-front-ends-on-ubuntu-bionic-beaver-1804-lts/



Телеграм чат по Elasticsearch: https://t.me/elasticsearch_ru