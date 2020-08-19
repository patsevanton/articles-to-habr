**ELK SIEM Open Distro: Прогулка по open Distro**

Оглавление всех постов.

- Введение. Развертывание инфраструктуры и технологий для SOC как Service (SOCasS)

- ELK stack: данные по установке и доставке
- Open Distro для Elasticsearch
- Визуализация Dashboards и ELK SIEM
- Интеграция с WAZUH
- Оповещение (Alerting)
- Отчетность
- Case Management



В open Distro доступны следующие плагины являются :

- Безопасность (Security )
- Оповещение (Alerting )
- SQL
- Управление информационной безопасностью (ISM)
- Анализатор производительности (Performance Analyzer )

В нашем проекте мы установили только плагины безопасности и оповещений.

### 1-Функция Оповещения :

Open Distro for Elasticsearch позволяет отслеживать Ваши данные и автоматически отправлять оповещения заинтересованным сторонам. Он прост в настройке и управлении, а также использует интерфейс Kibana с мощным API.

Функция оповещения позволяет настроить правила таким образом, чтобы вы могли получать уведомления, когда что-то интересное изменяется в ваших данных. Все, что вы можете запросить, вы можете построить предупреждение на этом. Функция оповещения уведомляет вас, когда данные из одного или нескольких индексов Elasticsearch удовлетворяют определенным условиям.

Этот URL-адрес предоставляет историю версий открытого дистрибутива (в нашем случае мы будем использовать 1.6.0 ) :

https://opendistro.github.io/for-elasticsearch-docs/version-history/

Чтобы добавить функцию оповещения открытого дистрибутива, вы должны управлять плагинами для elasticsearch и kibana : управление плагинами находится в следующих путях

```
/usr/share/elasticsearch : для Elasticsearch
/usr/share/kibana : для Kibana
```

#### 1.1- Установка плагина Alerting для elasticsearch:

```
cd /usr/share/elasticsearch
sudo bin/elasticsearch-plugin install https://d3g5vo6xdbdb9a.cloudfront.net/downloads/elasticsearch-plugins/opendistro-sql/opendistro_sql-1.6.0.0.zip
```

#### 1.2- Установка плагина Alerting для kibana :

```
cd /usr/share/kibana
sudo bin/kibana-plugin install — allow-root https://d3g5vo6xdbdb9a.cloudfront.net/downloads/kibana-plugins/opendistro-alerting/opendistro-alerting-1.6.0.0.zip
```

#### *1.3-Чтобы отобразить ваши плагины или удалить их , вы можете использовать следующие команды :*

*- Для Kibana :*

```
sudo bin/kibana-plugin list
sudo bin/kibana-plugin remove <plugin-name>
```

*- Для elasticsearch :*

```
sudo bin/elasticsearch-plugin list
sudo bin/elasticsearch-plugin remove <plugin-name>
```

#### *1.4-Теперь вы должны перезапустить kibana и elasticsearch :*

```
systemctl restart kibana elasticsearch
```


 *Примечание: после установки , удаления или обновления плагинов для elasticsearch или kibana требуется подождать несколько минут для правильной перезагрузки , в то время как ваш сервер kibana будет отвечать ( kibana is not ready yet ). Вы можете проверить процессы kibana и elasticsearch в режиме реального времени с помощью команды top.*

#### 1.5-Теперь вы можете проверить свои плагины в интерфейсе kibana :

![](https://habrastorage.org/webt/lx/xb/77/lxxb77wuburch8avv1eqqk4dkku.png)

#### 1.6-Работа с плагином оповещения :

##### а) Создание URL-адреса веб-перехватчика Slack:

Slack - это инструмент коммуникации на рабочем месте, «единое место для обмена сообщениями, инструментами и файлами». То есть Slack - это система обмена мгновенными сообщениями с множеством надстроек для других инструментов рабочего места.

Входящие Webhooks - это простой способ отправлять сообщения из приложений в Slack. Создание входящего веб-перехватчика дает вам уникальный URL-адрес, на который вы отправляете полезные данные [JSON](https://en.wikipedia.org/wiki/JSON) с текстом сообщения и некоторыми параметрами. Вы можете использовать все обычные блоки [форматирования](https://api.slack.com/reference/surfaces/formatting) и [макета](https://api.slack.com/messaging/composing/layouts) с помощью Incoming Webhooks, чтобы сообщения выделялись.

- Сначала создайте учетную запись (slack.com)

  ![](https://habrastorage.org/webt/si/ao/qg/siaoqgipkqgj-saxok5yycfbisa.png)

- Выберите второй пункт, если вы новичок в Slack.

  ![](https://habrastorage.org/webt/6m/h5/h_/6mh5h_atdxsowzleyj-65gjk9g8.png)

- Получите код, который вы только что получили в своем почтовом ящике, и создайте новое рабочее пространство

  ![](https://habrastorage.org/webt/xm/-n/wp/xm-nwpnalrlwcbl0v8h6eu5oyfc.png)

- Попробуйте следовать инструкциям, пока не дойдете до своей домашней страницы, перейдите в приложение и найдите Incoming Webhook, нажмите добавить:

  ![](https://habrastorage.org/webt/37/2f/ip/372fipa1eoqvhukzkv9gyieezmg.png)

- Нажмите добавить в Slack

  ![](https://habrastorage.org/webt/b4/dl/sd/b4dlsdnjtnctc0pl2cezxusgvqe.png)

- Выберите канал для получения сообщений (например тестовый) и нажмите «Добавить интеграцию».

  ![](https://habrastorage.org/webt/4t/vo/3i/4tvo3icknqz3vy0x28maepn4aha.png)

- Теперь прокрутите вниз, пока не найдете URL-адрес веб-перехватчика (сохраните его, потому что мы будем использовать его позже)

  ![](https://habrastorage.org/webt/q3/oj/rs/q3ojrsnoaye5kqoq6gesnsr2itk.png)

- Перейдите в Kibana → Alerting → Destination и нажмите кнопку add destination:

  ![](https://habrastorage.org/webt/xi/ew/zr/xiewzrhqy5cysh-geavdivvbyno.png)

- Выберите имя назначения, выберите место назначения Slack, скопируйте URl вашего веб-перехватчика и нажмите создать.

![](https://habrastorage.org/webt/u3/n9/gx/u3n9gxclz8ses_16hlskgif146c.png)

##### 1.6.2- Создание оповещений и отправка их в Slack:

- Теперь перейдите в Мониторинг и нажмите Создать монитор:

  ![](https://habrastorage.org/webt/0-/3d/wv/0-3dwvk6_0gfcogbmkennacz04m.png)

- Настройте свои параметры: вы можете использовать графическую настройку или настройку запроса извлечения

![](https://habrastorage.org/webt/t2/nu/1n/t2nu1n-ejuiigqfj-fo-l_zhx7w.png)

![](https://habrastorage.org/webt/vo/qt/l0/voqtl0ojhkywlic37avp9q6du7q.png)

Вот пример графической настройки (идентификатор события: 4624 означает, что учетная запись успешно вошла в систему)

![](https://habrastorage.org/webt/y6/oa/7i/y6oa7iptpgips5nc5a2pfgqsryq.png)

- Проверьте Monitor Schedule и нажмите кнопку Создать

![](https://habrastorage.org/webt/ry/lc/w3/rylcw3ukrzogd-8tsxvnaqr9jlk.png)

Теперь вы должны создать триггер, например:

![](https://habrastorage.org/webt/su/cy/xy/sucyxya1o9phs0p4brz0svxlssa.png)

Теперь перейдите к уведомлению и выберите место назначения, которое вы создали, затем нажмите создать:

![](https://habrastorage.org/webt/bi/74/78/bi7478jk8iwwgt7giibuhbg_fso.png)

Теперь вы можете следить за своими оповещениями в интерфейсе оповещений Kibana, и каждое оповещение будет отправляться на ваш канал Slack:

![](https://habrastorage.org/webt/0g/wb/v8/0gwbv82drgwpac4hd5jyap3ml50.png)

Перейдите на свой Slack-канал (#test канал в этом руководстве) и дождитесь ваших предупреждений:

![](https://habrastorage.org/webt/h5/0b/x8/h50bx8qxuwm_0cxuszalew9_byu.png)

### 2- Функция безопасности:

Этот плагин предоставляет пользовательский интерфейс для управления пользователями, ролями, сопоставлениями, группами действий и арендаторами.

#### 2.1- Установка плагина безопасности:

Выбор этого подключения был сделан потому, что Kibana не имеет панели аутентификации в базовой версии. Таким образом, чтобы обеспечить безопасность наших интерфейсов, мы будем использовать бесплатную панель аутентификации, предлагаемую открытым дистрибутивом.

Вы можете выполнить те же шаги (от 1 до 4), что и делали для  установки плагина предупреждений, чтобы установить плагины безопасности. Вам просто нужно изменить плагин установки URL:

Для Kibana:

```
sudo bin/kibana-plugin install — allow-root https://d3g5vo6xdbdb9a.cloudfront.net/downloads/kibana-plugins/opendistro-security/opendistro_security_kibana_plugin-1.6.0.0.zip
```

Для Elasticsearch:

```
sudo bin/elasticsearch-plugin install https://d3g5vo6xdbdb9a.cloudfront.net/downloads/elasticsearch-plugins/opendistro-security/opendistro_security-1.6.0.0.zip
```

Будет отображено предупреждающее сообщение: type y

![](https://habrastorage.org/webt/qq/uc/gx/qqucgxl5hrrhv8syjyqmcfmttlm.png)

После установки плагина безопасности вы можете запустить:

```
sudo sh /usr/share/elasticsearch/plugins/opendistro_security/tools/install_demo_configuration.sh
```

Чтобы быстро начать работу с демонстрационными сертификатами.

В противном случае вы должны настроить его вручную и запустить securityadmin.sh.

Воспользуемся первым вариантом:

```
cd /usr/share/elasticsearch/plugins/opendistro_security/tools/
chmod +x install_demo_configuration.sh
./install_demo_configuration.sh
```

Для всех вопросов типа y и учетных данных по умолчанию (Имя пользователя: admin / Пароль: admin)

![](https://habrastorage.org/webt/ah/ox/h7/ahoxh7ay91a9-zozq9hybbtlgn4.png)

Другая конфигурация будет добавлена безопасностью открытого дистрибутива в ваш /etc/elasticsearch/elasticsearch.yml

![](https://habrastorage.org/webt/_o/ue/sl/_ouesl61hbmcikztyygo9g7pai0.png)

#### 2.2- Изменение конфигурации для elasticsearch logstash и kibana:

В нашем случае мы создадим имя пользователя, пароль и сертификат SSL для elasticsearch. Мы хотим упомянуть, что проверка сертификата в этом разделе и во всех других следующих разделах выходит за рамки этих статей.

##### 2.2.1- Для Elasticsearch:

Отключение  **x-pack security** для elasticsearch: при перезапуске elasticsearch вы, вероятно, получите ошибку из-за функции безопасности xpack, включенной по умолчанию для базовой версии ELK Stack, поэтому вам придется отключить ее в /etc/elasticsearch/elasticsearch.yml перед перезапуском вашей службы.

##### 2.2.2- Для Кибаны:

Отключение  **x-pack security** для Kibana: Также для Кибаны мы должны отключить функцию xpack.security и игнорировать проверку ssl в /etc/kibana/kibana.yml

**ПРИМЕЧАНИЕ**. Убедитесь, что ваш протокол - https, а не http.

![](https://habrastorage.org/webt/iz/8s/pj/iz8spjbymumnjtcql8ft0fgfews.png)

![](https://habrastorage.org/webt/fu/or/v7/fuorv7cq4vbfljgzgdiejhxomq8.png)

![](https://habrastorage.org/webt/lj/-q/rw/lj-qrwthrmeg93y_9ovskr5za5q.png)

##### 2.2.3- Для Logstash:

Поскольку наши биты не связаны напрямую с elasticsearch, они вместо этого подключены к logstash, поэтому нам не нужно управлять своими битами или перенастраивать их, нам нужно только настроить файл конфигурации logstash.

Убедитесь, что ваш протокол https, а не http.

```
sudo nano /etc/logstash/conf.d/logstash.conf
```

![](https://habrastorage.org/webt/rf/dw/kx/rfdwkxwrkmnqlzghstaihcurxuc.png)

**ПРИМЕЧАНИЕ**: если вы перенастраиваете биты или настраиваете другой бит, в то время как в вашем elasticsearch установлен плагин безопасности с именем пользователя, паролем и сертификатом SSL, вы можете добавить эту конфигурацию в свой бит, чтобы сделать его доступным. Убедитесь, что ваш протокол - https, а не http.

![](https://habrastorage.org/webt/r-/f2/5m/r-f25mxa2a0eisq1jw5zodbryyy.png)

#### 2.3- Перезапустите все свои службы:

```
systemctl restart elasticsearch
systemctl restart logtash
systemctl restart kibana
```

Как упоминалось ранее, для правильного перезапуска может потребоваться несколько минут. Вы  можете проверять это с помощью команды top в режиме реального времени. тем временем ваш сервер Кибаны будет отвечать (kibana is not ready yet).

Теперь ваш стек ELK правильно подключен с новыми учетными данными.

Вы можете проверить это, используя URL-адрес Elasticsearch (http не будет работать, вместо этого вам нужно использовать https)

![](https://habrastorage.org/webt/e-/ek/ov/e-ekov2dd3yvwrew8hy8s8mqmdo.png)

![](https://habrastorage.org/webt/se/_0/kq/se_0kqo9ueo9dwktkfdt9ampauu.png)

Вы также можете проверить это в Кибане:

![](https://habrastorage.org/webt/86/ij/ms/86ijms8r5es77xujsakxlugrjjy.png)

![](https://habrastorage.org/webt/rd/an/wb/rdanwb6bkmahx5v7sdclgzhllng.png)

Здесь вы можете создавать пользователей, назначать роли и разрешения:

![](https://habrastorage.org/webt/7r/uq/i6/7ruqi63mwzpy4vwwwqz5yeicj_s.png)

Это поможет вам организовать группы SOC на основе ролей, действий и привилегий.

Вот роли и база данных внутренних пользователей, определенные по умолчанию:

![](https://habrastorage.org/webt/7q/s9/5u/7qs95ulnaomeu2pkvtlec8hlmae.png)

![](https://habrastorage.org/webt/ak/ls/hc/aklshcud37b7nfx8k9mnys2xyug.png)



Телеграм чат по Elasticsearch: https://t.me/elasticsearch_ru