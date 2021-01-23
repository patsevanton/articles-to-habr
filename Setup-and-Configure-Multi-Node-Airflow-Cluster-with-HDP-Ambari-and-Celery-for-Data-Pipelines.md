https://medium.com/@khatri_chetan/setup-and-configure-multi-node-airflow-cluster-with-hdp-ambari-and-celery-for-data-pipelines-dc1e96f3d773

Настройка мультинодовый кластер Airflow с HDP Ambari и Celery для конвейеров данных

Airflow - идеальный выбор для конвейеров данных, то есть для оркестрации и планирования ETL. Он широко применяется и популярен для создания конвейеров передачи данных будущего. Он обеспечивает обратное заполнение, управление версиями и происхождение с помощью функциональной абстракции.

![](https://habrastorage.org/webt/jh/yc/5k/jhyc5kh12pbs-vrszqqywvtxnni.jpeg)

*Функциональное программирование - это будущее.*

Оператор определяет единицу в рабочем процессе, DAG - это набор Задач. Операторы обычно работают независимо, на самом деле они могут работать на совершенно двух разных машинах. Если вы инженер данных и работали с Apache Spark или Apache Drill, вы, вероятно, знаете, что такое DAG! Такая же концепция и у Airflow.

Создавайте конвейеры данных, которые:

1. Идемпотентны. 

2. Детерминированы.

3. Не имеют побочных эффектов.

4. Используют неизменные источники и направления.

5. Не обновляются, не добавляются.

Модульность и масштабируемость - основная цель функциональных конвейеров данных.

Если вы работали с функциональным программированием с помощью Haskell, Scala, Erlang или Kotlin, вы удивитесь, что это то, что мы делаем в функциональном программировании, и вышеперечисленные пункты относятся к функциональному программированию, да! Вы правы. Функциональное программирование - это мощный инструмент будущего.

Если у вас есть ETL / Data Lake / Streaming Infrastructure как часть платформы разработки данных, у вас должен быть кластер Hadoop / Spark с некоторым дистрибутивом, таким как Hortonworks, MapR, Cloudera и т. д. Поэтому я собираюсь рассказать о том, как вы можете использовать ту же инфраструктуру, где у вас есть Apache Hadoop / Apache Spark Cluster, и вы используете его для создания Airflow Cluster и его масштабирования.

Если у вас много заданий ETL и вы хотите организовать и составить расписание с помощью некоторых инструментов планирования, у вас есть несколько вариантов, например Oozie, Luigi и Airflow. Oozie основан на XML, и мы взяли его 2019 году! :), Luigi чуть не выбросили после того, как Airflow родился на Airbnb.

Почему не используем **Luigi с** Airflow?

1. У Airflow есть собственный планировщик, в котором Luigi требует синхронизировать задачи в задании cron.

2. С Luigi навигация по пользовательскому интерфейсу становится сложной задачей.

3. В Luigi сложно создавать задачи.

4. Luigi не масштабируется из-за тесной связи с заданиями Cron.

5. Повторный запуск процесса в Luigi невозможен.

6. Luigi не поддерживает распределенное выполнение, так как оно плохо масштабируется.

До появления Airflow я использовал Luigi для поддержки рабочего процесса моделей машинного обучения с помощью Scikit-learn, Numpy, Pandas, Theano и т. д.

В последнем сообщении блога мы обсудили, как настроить мультинодовый кластер Airflow с Celery и RabbitMQ без использования Ambari.

Ага, переходим к главному.

Как настроить мультинодовый кластер Airflow в Hadoop Spark Cluster, чтобы Airflow мог запускать задания Spark/Hive/Hadoop Map Reduce, а также выполнять координацию и планирование.

Давайте сделаем это!

Вы должны использовать airflow-ambari-mpack (пакет управления Apache Airflow для Apache Ambari), я использовал реализацию с открытым исходным кодом от FOSS Contributor https://github.com/miho120/ambari-airflow-mpack, Спасибо за ваш вклад.

Следующие шаги:

Из предыдущего сообщения в блоге вы должны выполнить шаги с 1 по 4, чтобы установить RabbitMQ и другие пакеты.

Ref. Как настроить мультинодовый кластер Airflow с помощью Celery и RabbitMQ

1. **Устанавливаем Apache MPack для Airflow**

   ```
   a. git clone https://github.com/miho120/ambari-mpack.git
   b. stop ambari server
   c. install the apache mpack for airflow on ambari server
   d. start ambari server
   ```

   

2. **Добавляем Airflow Service в Ambari**

После успешного выполнения вышеуказанных шагов вы можете открыть интерфейс Ambari

```
http://<HOST_NAME>:8080
```

Откройте пользовательский интерфейс Ambari (Ambari UI), нажмите Действия -> Добавить службу. (Actions -> Add Service)

![](https://habrastorage.org/webt/fw/bt/_0/fwbt_0o5stlnjqgsljybnpsaokw.png)

*HDP Ambari Dashboard*

Если шаг 1 выполнен успешно, вы сможете увидеть Airflow как часть службы Ambari.

![](https://habrastorage.org/webt/ae/f2/jf/aef2jfzi268anz0g5uxbxxklv70.png)

*Сервис Airflow в Ambari*

Вы должны выбрать, на каком узле вы хотите установить веб-сервер, планировщик и воркер. Я бы порекомендовал установить Airflow на веб-сервер, планировщик на master ноде, то есть на узле имени, и на Install Worker на data-нодах.

![](https://habrastorage.org/webt/wv/6k/wc/wv6kwcv1ldh_vzqpjwoszwetlbw.png)

*Ambari Master нода / Name нода для Airflow*

Как вы можете видеть на изображении выше, веб-сервер Airflow и планировщик Airflow установлены на Name ноде кластера Hadoop / Spark.

![](https://habrastorage.org/webt/c3/es/68/c3es68bqrg0906g65tjhrw2dbga.png)

Как вы можете видеть на скриншоте выше, служба Airflow Worker установлена на Data ноде кластера.

В итоге, у меня есть 3 воркер (worker) ноды на трех data нодах.

![](https://habrastorage.org/webt/rf/yl/jl/rfyljlmdoolqm6dlcxyzy6nliy8.png)

*Сервис Airflow в Ambari*

![](https://habrastorage.org/webt/az/uk/y0/azuky0q-iuz-21cuarb3oflzo1g.png)

*Ambari UI: 3 воркера в Airflow*

Вы можете добавлять рабочие нода столько, сколько хотите, вы можете добавлять / удалять воркеров, когда захотите. Эта стратегия может масштабироваться по горизонтали + вертикали.

**Конфигурация Airflow в Ambari:**

Нажмите на Airflow Service, а затем на Config в пользовательском интерфейсе Ambari.

![](https://habrastorage.org/webt/hm/zi/oj/hmziojzjyzowkgz2akherlqry4e.png)

*Конфигурация Airflow в Ambari*

1. **Смените Executor**

```
executor = CeleryExecutor
```

![](https://habrastorage.org/webt/ca/sm/8l/casm8lavu4d61y09gscavgblaiw.png)

В разделе Advanced airflow-core-site укажите Executor как **CeleryExecutor**.

2. **SQL Alchemy Connection**

```
sql_alchemy_conn = postgresql+psycopg2://airflow:airflow@{HOSTNAME}/airflow
```

![](https://habrastorage.org/webt/l2/vs/gq/l2vsgqyjixsakeygrb8grmzcjsu.png)

*SQL Alchemy Connection*

Измените соединение SQL Alchemy на соединение postgresql, пример приведен выше.

3. **URL-адрес брокера**

   ```
   broker_url= pyamqp://guest:guest@{RabbitMQ-HOSTNAME}:5672/
   celery_result_backend = db+postgresql://airflow:airflow@{HOSTNAME}/airflow
   ```

   ![](https://habrastorage.org/webt/nb/id/n9/nbidn9dsbkd262yn3wyzaqwmesw.png)

   *URL-адрес брокера и Celery result backend для Airflow*

4. **Прочие**

```
dags_are_paused_at_creation = True
load_examples = False
```

![](https://habrastorage.org/webt/s1/68/2w/s1682wn8a2gf_efa9mbsdini-ba.png)

*Конфигурация Airflow-core-site.*

После того, как все эти изменения будут внесены в конфигурацию Ambari Airflow, Ambari попросит вас перезапустить все затронутые службы, перезапустите службы и нажмите **Service Actions -> InitDB**.

![](https://habrastorage.org/webt/km/bc/ej/kmbcejrnboap3n4ggzandk5767q.png)

*Airflow Initdb из Ambari*

А затем запустите службу airflow. Теперь у вас должен получиться мультинодовый кластер Airflow.

**Кое-что из Чек-листа для проверки служб для мультинодового кластера Airflow:**

1. Очереди RabbitMQ должны быть запущены:

   ![](https://habrastorage.org/webt/kt/n8/hj/ktn8hjha9gbd6nehpwegscq_k5m.png)

2. Подключения RabbitMQ должны быть активными:

   ![](https://habrastorage.org/webt/ke/m9/yx/kem9yxmco0j9_qw_k6y6rdjcwro.png)

3. Каналы RabbitMQ должны быть запущены:

   ![](https://habrastorage.org/webt/fd/5d/az/fd5dazk_mn28xzpzgabhay0l82i.png)

4. Отображение *Celery Flower*

Celery Flower - это веб-инструмент для мониторинга и управления кластерами Celery. Номер порта по умолчанию - 5555.

![](https://habrastorage.org/webt/qr/1b/wo/qr1bwoby1zjc8koojffbvzpokee.png)

Вы также можете видеть здесь, что 3 рабочих находятся в сети, и вы можете отслеживать одну единицу «задачи» Celery здесь.

Подробнее о Celery Flower: https://flower.readthedocs.io/en/latest/

Обратите внимание, что вы также можете запустить «Celery Flower», веб-интерфейс, созданный поверх Celery, для наблюдения за своими рабочими. Вы можете использовать команду быстрого доступа `airflow flower`, чтобы запустить веб-сервер Flower.

```
nohup airflow flower >> /var/local/airflow/logs/flower.logs &
```

Мы закончили установку и настройку мультинодовый кластер Airflow на Ambari HDP Hadoop / Spark Cluster.

Я столкнулся с некоторыми проблемами, о которых я расскажу в следующей статье в блоге.
