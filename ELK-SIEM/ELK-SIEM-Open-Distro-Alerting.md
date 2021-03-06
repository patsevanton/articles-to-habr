Оповещения(алерты)

![](https://habrastorage.org/webt/2x/jg/pi/2xjgpi6fnh_lamm9bhdhnfutfzw.png)

Здравствуйте и добро пожаловать в нашу новую статью, в которой будет рассказано об оповещениях(алертах) в нашем решении SOCaaS. Как вы все знаете, предупреждения в любом SOC играют жизненно важную роль при уведомлении группы реагирования.

Они могут прервать цепочку кибер-атак или отслеживать эту атаку, в зависимости от политики предприятия и команды. Вы, наверное, задаетесь вопросом, зачем нам нужно включать больше предупреждений. Разве модулей предупреждений Open Distro недостаточно? Это потому, что ему не хватает количества выходов и его интегрируемости с остальной частью нашего решения, например Thehive. Мы познакомим вас с другой альтернативой.



Я призываю вас всех изучить предыдущую статью, чтобы лучше понять, что мы собираемся здесь обсудить.

Статья разделена на следующие разделы:

\* Установка и настройка ElastAlert, ElastAlert-Server и Praeco

\* Создание правил



### 1-Установка и настройка ElastAlert, ElastAlert-Server и Praeco:

#### 1.1 Введение:

**A- Определения**

\- **Praeco**: позволяет создавать оповещения с опциями уведомлений, включая Slack, электронную почту, Telegram, Jira.

Оповещения в Praeco могут быть собраны либо путем выбора полей, о которых нужно оповещать, и соответствующих операторов с помощью конструктора запросов, либо вручную с помощью языка запросов Kibana (KQL).

\- **ElastAlert** - это простая структура для оповещения об аномалиях, всплесках или других интересных паттернах на основе данных в Elasticsearch. Он работает путем объединения Elasticsearch с двумя типами компонентов: типами правил и предупреждениями. Elasticsearch периодически опрашивается, и данные передаются в тип правила, который определяет, когда будет найдено совпадение. Когда происходит совпадение, передается одно или несколько предупреждений, которые принимают меры в зависимости от совпадения.

Это настраивается набором правил, каждое из которых определяет запрос, тип правила и набор предупреждений.

\- Правила **Sigma**: Sigma - это общий и открытый формат подписи, который позволяет вам прямо описывать соответствующие события журнала. Формат правила очень гибкий, его легко написать и он применим к любому типу файла журнала. Основная цель этого проекта - предоставить структурированную форму, в которой исследователи или аналитики могут описать свои однажды разработанные методы обнаружения и сделать их доступными для других.



**B- Клонирование проектов:**

```
cd /etc
git clone https://github.com/Yelp/elastalert.git
git clone https://github.com/ServerCentral/elastalert-server.git
git clone https://github.com/ServerCentral/praeco.git
```

Вы можете найти дополнительную информацию по этому URL-адресу: https://github.com/ServerCentral/praeco

#### 1.2-Настройка Elastalert:

```
cd /etc/elastalert
mkdir rules rule_templates
cp config.yaml.example config.yaml
nano config.yaml
```

Настройте elastalert config.yaml с помощью:

Ваш `es_host: localhost`

Уникальный `writeback_index: elastalert_status`

Измените папку `rules_folder` на `rules`

![](https://habrastorage.org/webt/zx/hh/au/zxhhaunuyafh1d-6mqkpjywwcq0.png)

ПРИМЕЧАНИЕ. Если вы используете python 2.7, вам необходимо изменить его на 3.6.

![](https://habrastorage.org/webt/7n/lc/db/7nlcdb4kiwje4laencvky17a1_c.png)

**A- Установка python3.6 в Ubuntu:**

```
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.6
```

**B-Обновление конфигурации Python:**

```
sudo update-alternatives — install /usr/bin/python python /usr/bin/python2.7
sudo update-alternatives — install /usr/bin/python python /usr/bin/python3.6
```

**C-Изменение Python по умолчанию:**

```
update-alternatives — config python
```

И выберите python3.6

![](https://habrastorage.org/webt/5t/nc/nr/5tncnr6yy6qnqglkhlxri5bc9ba.png)


Теперь по умолчанию у вас должен быть python3.6

**D-Install pip3:**

```
sudo apt install python3-pip
```

**E- Вам также необходимо установить PyYAML (пример 5.1):**

```
pip install PyYAML==5.1
```

**F- Требования к установке и elastalert**

```
cd /etc/elastalert
pip3 install “setuptools>=11.3”
python setup.py install
```

**G- Создание индекса:**

```
cd /usr/local/bin/
./elastalert-create-index
```

Параметры

```
ES Host : localhost
ES Port : 9200
Use ssl : t
Verify ssl :f
```

Мы будем использовать по умолчанию ES_username: admin и ES_password: admin.

Также оставьте ответ по умолчанию для остальных вопросов.

![](https://habrastorage.org/webt/en/_p/b4/en_pb4ocez9tcuaa_fz4tditdiy.png)

#### 1.3- Настройка сервера API:

Настройте сервер API /etc/elastalert-server/config/config.json с помощью:

- Абсолютный путь к вашей папке elastalert для elastalertPath: `/etc/elastalert`
- Адрес вашего экземпляра elasticsearch для es_host: elasticsearch
- Тот же writeback_index из config.yaml: elastalert_status

![](https://habrastorage.org/webt/y_/yc/bq/y_ycbqmmlndlbuz0x7rcl-ddjv4.png)

**A- Журналы предупреждений о неисправностях (`No Data`):**

Обработчик метаданных, относящийся к индексу обратной записи журнала предупреждений, ищет документы с _type elastalert. Начиная с 7.x, это не возвращает результатов, потому что все документы имеют _type  _doc.

Таким образом, в вашем журнале предупреждений (в интерфейсе Praeco позже) вы увидите `No Data`.

Итак, вам необходимо:

```
cd /etc/elastalert-server/src/handlers/metadata/
nano get.js
```

удалите строку, содержащую тип: «elastalert»,

Теперь вы сможете просматривать журналы предупреждений в интерфейсе praeco.

![](https://habrastorage.org/webt/7l/uy/zd/7luyzdmwdks9yyvzsqf9vn34hcm.png)

![](https://habrastorage.org/webt/c8/--/yq/c8--yqsrxz5gfdfgonpz8bay6k0.png)

**B-Установите Elastalert-Server:**

```
sudo npm install
sudo npm run start
```

Вы должны увидеть эту строку, если она запустилась успешно. Это просто предупреждение из-за небезопасного соединения (`SSL_verify = False`).

![](https://habrastorage.org/webt/7a/ru/nu/7arunuw1hqxtlznkgymlegvjyi0.png)

#### 1.4- Настройка Praeco:

**A- Изменить файлы конфигурации:**

```
cd /etc/praeco/config
nano api.config.json
```

![](https://habrastorage.org/webt/qz/dz/zz/qzdzzz1w1x6lquooikli6xjlofu.png)

```
nano elastalert.yml
```

![](https://habrastorage.org/webt/l0/yy/gx/l0yygx9ut1iigttnsr---zvuqtu.png)

![](https://habrastorage.org/webt/d3/wp/_d/d3wp_deppqjgnrlt1osokp3j9le.png)

**B- Установите Praeco:**

```
sudo npm install
export PRAECO_ELASTICSEARCH=localhost
```

**C- Скопируйте BaseRule.cfg:**

Перед запуском сервисов необходимо:

```
cp /etc/praeco/rules/BaseRule.config /etc/elastalert/rules/
```

Этот файл содержит настройки для Slack, SMTP и Telegram.

Здесь мы добавим URL-адрес Slack Webhook, используемый в Разделе 2.

```
cd /etc/elastalert/rules/
nano BaseRule.config
```

добавьте URL-адрес Webhook

![](https://habrastorage.org/webt/wr/fo/v4/wrfov4qshlg7jdxg4fje0iwhezc.png)

**D- Запустите Praeco:**

```
npm run serve
```

Теперь вы должны увидеть, что пользовательский интерфейс работает по адресу http://yourServerIP: 8080.

Вот ваш интерфейс Praeco

![](https://habrastorage.org/webt/q_/oq/wk/q_oqwklwrdcwuu08letzfxzqy8u.png)

### 2- Создание правил:

#### 2.1.- Создание правил с помощью интерфейса Praeco и отправка их в Slack webhook:

Перейдите в Правила(Rules) -> Добавить правило(Add Rule):

![](https://habrastorage.org/webt/rp/hh/t_/rphht_m7u75zf4jhahktxdhvjgk.png)


Теперь вы можете видеть, что создание правила очень похоже на Open Distro Alerting Tool, мы отфильтруем предупреждение и укажем место назначения.

Щелкните UNFILTERED и укажите фильтр вручную или с помощью инструмента предварительной сборки.

Затем нажмите Close

![](https://habrastorage.org/webt/wi/qi/-i/wiqi-icmkw3-gwyi-s4hsur4h_k.png)


Мы будем использовать уведомления Slack с тем же URL-адресом веб-перехватчика, который используется в разделе 2, и тем же каналом (#test).

![](https://habrastorage.org/webt/fx/md/m0/fxmdm0nihunuzgqzgv6pja-myb0.png)

![](https://habrastorage.org/webt/sx/ig/aq/sxigaqlzkhwyeedwqe_a1zdg2qe.png)

![](https://habrastorage.org/webt/ba/kh/oi/bakhoih9xaoj3yut7-arxdhfu08.png)


Нажмите Save

Ваше оповещение включено по умолчанию

![](https://habrastorage.org/webt/iz/ra/yv/izrayvov4fksc4gggy-f_o7ah_o.png)


Мы можем проверить, что оповещения были успешно отправлены в Slack.

![](https://habrastorage.org/webt/6m/cc/oi/6mccoiozv14i5nnmwnoteqk_6rs.png)

В нашем слабом канале:

![](https://habrastorage.org/webt/14/_x/4r/14_x4ruufajly_2xonucd_ndtbi.png)

#### 2.2- Отправка предупреждений из ElastAlert в TheHive

К сожалению, Praeco не обеспечивает вывод предупреждений в TheHive, поэтому мы будем редактировать наши правила вручную и отправлять их с помощью elastalert-server.

Благодаря этому обходному пути правила будут хорошо работать в фоновом режиме благодаря Elastalert-server, они также будут присутствовать в интерфейсе Praeco, но, к сожалению, мы не сможем редактировать или настраивать их с помощью интерфейса Praeco.

**A- Создание правила: «User_creation»:**

Во-первых, мы создадим наше правило из интерфейса Praceo, как и раньше, мы укажем любой URL-адрес в нашем выводе HTTP, потому что мы удалим его позже.

Когда закончите, нажмите Save.

![](https://habrastorage.org/webt/hk/hz/n6/hkhzn6zrlvq6kmfeznklprjsdlw.png)

**B- Уведомления о доставке в Thehive:**

Добавьте настройку TheHive, сохраните и перезапустите Elastalert-Server

Перейдите в `/etc/elastalert/rules`

```
nano User_creation.yml
```

![](https://habrastorage.org/webt/ke/rl/_t/kerl_tob0reks-psccrltndxn_k.png)

**C. Проверка предупреждений в Praeco**

![](https://habrastorage.org/webt/zf/al/x7/zfalx79fqn9e2lsb5qtx62ptf8k.png)

Наше оповещение было успешно отправлено в Hive

К сожалению, как мы упоминали ранее, правило не будет редактироваться с помощью интерфейса Praeco, вам придется отредактировать его вручную в `/etc/elastalert/rules`:

![](https://habrastorage.org/webt/n2/yj/81/n2yj81gshbzjcpd-hhiurkop67q.png)

**D- Проверка предупреждений в интерфейсе TheHive:**

![](https://habrastorage.org/webt/gu/xu/nn/guxunnudcosoyw8b1pp3a-zfupk.png)

**2.3- Получите помощь с инструментом Sigma для создания правил:**

Как упоминалось ранее, инструмент Sigma помог нам преобразовать правила сигмы в несколько форматов, включая Elastalert.

URL проекта с подробными инструкциями: https://github.com/Neo23x0/sigma.git

![](https://habrastorage.org/webt/_x/av/lc/_xavlc01dkngngbyw6smm2-ohas.png)

**A- Загрузить инструмент Sigma**

```
cd ~
git clone https://github.com/Neo23x0/sigma.git
```

**B- Создайте свое оповещение с помощью инструмента сигма**

```
cd ~/sigma/tools
pip3 install -r requirements.txt
```

Выполнить (пример правила) (это одна команда):

```
./sigmac -t elastalert -c winlogbeat ../rules/windows/builtin/win_user_creation.yml
```

![](https://habrastorage.org/webt/bp/9d/vi/bp9dviw0_bwguwbnhsozrzwvk_y.png)

![](https://habrastorage.org/webt/vd/zs/zg/vdzszg2jkdnudglo_vg1108yxqe.png)


К сожалению, это правило не может напрямую использоваться (Praeco / Elastalert-Server) из-за отсутствия нескольких полей.

Таким образом, вы можете выбрать важную информацию из этого правила (строка запроса) и создать собственное правило в интерфейсе Praeco с этой информацией.

Этот инструмент очень важен, поскольку помогает собирать информацию о многих правилах и их строках запроса.

Примечание: Иногда вам нужно проверить свои журналы (Kibana → Discover Interface) и их поля, чтобы убедиться, что поля имен в ваших журналах совпадают с полями имен в правилах сигмы. Если в ваших полях отображается желтая ошибка, перейдите к шаблону индекса, выберите соответствующий индекс и нажмите кнопку «Обновить поля».

**2.4- Отправка предупреждений Wazuh в theHive:**

Мы будем использовать тот же обходной путь, упомянутый ранее, для предупреждений wazuh, сначала мы создадим предупреждения wazuh с помощью интерфейса praeco, затем мы вручную отредактируем файл правил, чтобы добавить вывод Hive:

**A. Создайте правило wazuh и сохраните его:**

![](https://habrastorage.org/webt/gn/tl/7e/gntl7enjah9t4b0nqhseq3mlct8.png)

Мы использовали для фильтрации правила с помощью rule.id (вы можете выбрать любое другое поле).

Вы можете получить идентификатор правил в wazuh → Overview → Security events.

![](https://habrastorage.org/webt/7c/qd/a4/7cqda4y_4ad377qzzecwjkx8hjg.png)

**B-Отредактируйте правило и перезапустите elastalert-server:**

```
nano /etc/elastalert/wazuh-alert-TEST.yaml
```

![](https://habrastorage.org/webt/mx/-d/kj/mx-dkjj84om4vvyfbrnsnllktim.png)



**C- Проверка оповещенний:**

![](https://habrastorage.org/webt/lz/pr/k-/lzprk-pbnz2geirr3ql1n1vcm0i.png)

![](https://habrastorage.org/webt/hp/d8/_b/hpd8_buzaizd_-l-z5ausvl4cyq.png)

Спасибо за уделенное время. Мы надеемся, что вам понравилась эта статья и вы лучше поняли систему оповещений, используемую в нашем проекте. Увидимся в следующей статье.
