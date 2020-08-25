Составление отчетов

Чтобы защитить вашу корпоративную сеть от угроз и атак, вы всегда должны выполнять тест на уязвимости в своей системе. Для того, чтобы их исправить. Итак, как вы понимаете, работа с отчетами очень важна для любого SOC, потому что она дает обзор уязвимостей, которые могут быть в вашей системе.

В этой статье мы расскажем вам об инструменте, который мы использовали для создания отчетов и сканирования уязвимостей.

Я призываю вас всех проверить предыдущую статью, чтобы лучше понять, что мы собираемся здесь обсудить.

Эта статья разделена на следующие разделы:

- 
- Введение

- 
- Установка Nessus Essentials

- 
- Установка VulnWhisperer

\1. Введение :

Инструменты, которые мы будем использовать:

- 
- VulnWhisperer: VulnWhisperer - это инструмент управления уязвимостями и агрегатор отчетов. VulnWhisperer извлечет все отчеты из различных сканеров уязвимостей и создаст файл с уникальным именем для каждого из них.

URL проекта: https://github.com/HASecuritySolutions/VulnWhisperer

- 
- Основы Nessus: Nessus Essentials (ранее Nessus Home) - это бесплатная версия сканера уязвимостей Nessus.

\2. Установка необходимых компонентов Nessus:

2.1- Скачать с официального сайта (www.tenable.com), в нашем проекте мы использовали эту версию:


 

2.2 - Установка Nessus:

***dpkg -i Nessus-8.10.0-ubuntu910_amd64.deb***

***/etc/init.d/nessusd start***

***service nessusd start***

Перейдите по адресу https: // YourServerIp: 8834 и выберите Nessus Essentials.


 

2.3- Начните с Nessus


 

Скопируйте код активации, создайте учетную запись и подождите, пока Nessus подготовит файлы.

2.4-Запустите первое сканирование:

Перейдите в New Scan и выберите Basic Network Scan.


 

Выберите цель, сохраните и запустите:


 

3.Установка VulnWhisperer:

3.1- Используйте Python2.7:

ПРИМЕЧАНИЕ: VulnWhisperer требует Python2.7, поэтому мы изменим нашу версию Python по умолчанию.


 

3.2- Настроить VulnWhisperer:

***cd /etc/***

***git clone*** [***https://github.com/HASecuritySolutions/VulnWhisperer***](https://github.com/HASecuritySolutions/VulnWhisperer)

***cd VulnWhisperer/***

***sudo apt-get install zlib1g-dev libxml2-dev libxslt1-dev***

***pip install -r requirements.txt***

***python setup.py install***

***nano configs/ frameworks_example.ini***

Выберите модули, которые вы хотите включить (в нашем случае мы просто включим Nessus), и напишите данные своей учетной записи Nessus:


 

3.3 - Проверьте соединение Nessus и загрузите отчет:

*vuln_whisperer -F -c configs/frameworks_example.ini -s nessus*

*Reports will be saved with csv extension.Check them under: /opt/VulnWhisperer/data/nessus/My\ Scans/*


 

Если нового отчета нет, вы увидите


 

3.4- Cronjob с Vulnwhisperer:

Чтобы Vulnwhisperer периодически проверял базу данных Nessus и загружал отчеты, мы добавим задание cron. Таким образом, нам больше не нужно будет выполнять эту команду вручную. Новые отчеты будут автоматически добавляться в Kibana.

***crontab –e***

Добавь это:

***SHELL=/bin/bash***

**** \* \* \* \* /usr/local/bin/vuln_whisperer -c /etc/VulnWhisperer/configs/frameworks_example.ini >/dev/null 2>&1***


 

3.5-Импорт шаблонов Elasticsearch:

Зайдите в kibana Dev Tools и добавьте шаблон:


 

**Ссылка на файл****:**

https://github.com/HASecuritySolutions/VulnWhisperer/blob/master/resources/elk6/logstash-vulnwhisperer-template_elk7.json


 

Теперь у вас будет шаблон индекса


 

3.6- Импорт визуализаций Кибаны

Перейдите в Kibana → Management → saved object → Import

Импортируйте конфигурацию kibana.json:

В VulnWhisperer / resources / elk6 / kibana.json:

*Ссылка на файл:*

[*https://github.com/HASecuritySolutions/VulnWhisperer/blob/master/resources/elk6/kibana.json*](https://github.com/HASecuritySolutions/VulnWhisperer/blob/master/resources/elk6/kibana.json)

Теперь под панелями мониторинга у вас есть:


 

3.7 -Добавление файла конфигурации Nessus Logstash

Скопируйте файл журнала Nessys в /etc/logstash/conf.d/:

***cd /etc/VulnWhisperer/resources/elk6/pipeline/***

***cp 1000_nessus_process_file.conf /etc/logstash/conf.d/***

***cd /etc/logstash/conf.d/***

***nano 1000_nessus_process_file.conf***

Изменить вывод


 

3.8- Перезапустите свои службы и проверьте отчеты:

**systemctl restart logstash elasticsearch**

**Т**еперь у вас должен быть создан новый индекс для Vulnwhisperer.


 

Перейдите к шаблону индекса и проверьте количество ваших полей:

ПРИМЕЧАНИЕ: обновите шаблон индекса, чтобы распознать все поля.


 

Наконец, перейдите на панели мониторинга и проверьте свои отчеты

У вас не должно быть ошибок в визуализации.


 

Теперь все отчеты, созданные nessus с расширениями csv, будут автоматически отправляться в ваш стек ELK, чтобы вы могли визуализировать их на панелях мониторинга kibana.