https://medium.com/@khatri_chetan/setup-and-configure-multi-node-airflow-cluster-with-hdp-ambari-and-celery-for-data-pipelines-dc1e96f3d773

Настройка мультинодовый кластер Airflow с HDP Ambari и Celery для конвейеров данных

Airflow - идеальный выбор для конвейеров данных, то есть для оркестрации и планирования ETL. Он широко применяется и популярен для создания конвейеров передачи данных будущего. Он обеспечивает обратное заполнение, управление версиями и происхождение с помощью функциональной абстракции.

Оператор определяет единицу в рабочем процессе, DAG - это набор Задач. Операторы обычно работают независимо, на самом деле они могут работать на совершенно двух разных машинах. Если вы инженер данных и работали с Apache Spark или Apache Drill, вы, вероятно, знаете, что такое DAG! Такая же концепция и у Airflow.

Создавайте конвейеры данных, которые:

1. Идемпотентны. 

2. Детерминированы.

3. Не имеют побочных эффектов.

4. Используют неизменные источники и направления.

5. Не обновляются, не добавляются.

Модульность и масштабируемость - основная цель функциональных конвейеров данных.

Если вы работали с функциональным программированием с помощью Haskell, Scala, Erlang или Now Kotlin, вы удивитесь, что это то, что мы делаем в функциональном программировании, и вышеперечисленные пункты относятся к функциональному программированию, да! Вы правы. Функциональное программирование - это мощный инструмент будущего.

Если у вас есть ETL / Data Lake / Streaming Infrastructure как часть платформы разработки данных, у вас должен быть кластер Hadoop / Spark с некоторым дистрибутивом, таким как Hortonworks, MapR, Cloudera и т. д. Поэтому я собираюсь рассказать о том, как вы можете использовать ту же инфраструктуру, где у вас есть Apache Hadoop / Apache Spark Cluster, и вы используете его для создания Airflow Cluster и его масштабирования.

Если у вас много заданий ETL и вы хотите организовать и составить расписание с помощью некоторых инструментов планирования, у вас есть несколько вариантов, например Oozie, Luigi и Airflow. Oozie основан на XML, и мы взяли его 2019 году! :), Луиджи чуть не выбросили после того, как Airflow родился на Airbnb.

Почему не используем **Luigi с** Airflow?

1. У Airflow есть собственный планировщик, в котором Луиджи требует синхронизировать задачи в задании cron.

2. С Луиджи навигация по пользовательскому интерфейсу становится сложной задачей.

3. В Луиджи сложно создавать задачи.

4. Luigi не масштабируется из-за тесной связи с заданиями Cron.

5. Повторный запуск процесса в Луиджи невозможен.

6. Луиджи не поддерживает распределенное выполнение, так как оно плохо масштабируется.

До появления Airflow я использовал Луиджи для поддержки рабочего процесса моделей машинного обучения с помощью Scikit-learn, Numpy, Pandas, Theano и т. д.

В последнем сообщении блога мы обсудили, как настроить мультинодовый кластер Airflow с Celery и RabbitMQ без использования Ambari.

Ага, переходим к главному.

Как настроить мультинодовый кластер Airflow в Hadoop Spark Cluster, чтобы Airflow мог запускать задания Spark/Hive/Hadoop Map Reduce, а также выполнять координацию и планирование.

Давайте сделаем это!

Вы должны использовать airflow-ambari-mpack (пакет управления Apache Airflow для Apache Ambari), я использовал реализацию с открытым исходным кодом от FOSS Contributor https://github.com/miho120/ambari-airflow-mpack, Спасибо за ваш вклад.

Следующие шаги:

Из предыдущего сообщения в блоге вы должны выполнить шаги с 1 по 4, чтобы установить RabbitMQ и другие пакеты.

Ref. Как настроить мультинодовый кластер Airflow с помощью Celery и RabbitMQ

1.   Устанавливаем Apache MPack для Airflow

2.   Добавляем Airflow Service в Ambari

После успешного выполнения вышеуказанных шагов вы можете открыть интерфейс Ambari

http: // <HOST_NAME>: 8080

Откройте пользовательский интерфейс Ambari (Ambari UI), нажмите Действия -> Добавить службу. (Actions -> Add Service)

Если шаг 1 выполнен успешно, вы сможете увидеть Airflow как часть службы Ambari.

Вы должны выбрать, на каком узле вы хотите установить веб-сервер, планировщик и воркер. Я бы порекомендовал установить Airflow на веб-сервер, планировщик на главном узле, то есть на узле имени, и на Install Worker на узлах данных.

Как вы можете видеть на изображении выше, веб-сервер Airflow и планировщик Airflow установлены на узле имени кластера Hadoop / Spark.

Как вы можете видеть на скриншоте выше, служба Airflow Worker установлена на узлах данных кластера.

В итоге, у меня есть 3 рабочих узла на трех моих узлах данных.

Вы можете добавлять рабочие узлы столько, сколько хотите, вы можете добавлять / удалять воркеров, когда захотите. Эта стратегия может масштабироваться по горизонтали + вертикали.

Конфигурация Airflow в Ambari:

Нажмите на Airflow Service, а затем на Config в пользовательском интерфейсе Ambari.

1. Смените исполнителя

В разделе Advanced airflow-core-site укажите Executor как CeleryExecutor.

2. Свяжите с SQL Alchemy

Измените соединение SQL Alchemy на соединение postgresql, пример приведен выше.

3. URL-адрес брокера

4. Прочие

После того, как все эти изменения будут внесены в конфигурацию Ambari Airflow, Ambari попросит вас перезапустить все затронутые службы, перезапустите службы и нажмите **Service Actions -> InitDB**.

а затем запустите службу airflow. Теперь у вас должно получиться хорошо с мультинодовый кластер Airflow.

Кое-что из Чек-листа для проверки служб для мультинодовый кластер Airflow:

1. Очереди RabbitMQ должны быть запущены:

2. Подключения RabbitMQ должны быть активными:

3. Каналы RabbitMQ должны быть запущены:

4. Отображение *Celery Flower*

Celery Flower - это веб-инструмент для мониторинга и управления кластерами Celery. Номер порта по умолчанию - 5555.

Вы также можете видеть здесь, что 3 рабочих находятся в сети, и вы можете отслеживать одну единицу «задачи» Celery здесь.

Подробнее о Celery Flower: https://flower.readthedocs.io/en/latest/

Обратите внимание, что вы также можете запустить «Celery Flower», веб-интерфейс, созданный поверх Celery, для наблюдения за своими рабочими. Вы можете использовать команду быстрого доступа airflow flower, чтобы запустить веб-сервер Flower.

Мы закончили установку и настройку мультинодовый кластер Airflow на \ Ambari HDP Hadoop / Spark Cluster.

Я столкнулся с некоторыми проблемами, о которых я расскажу в следующей статье в блоге.