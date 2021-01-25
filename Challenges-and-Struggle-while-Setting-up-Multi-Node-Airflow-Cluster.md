Проблемы и трудности при настройке Multi-Node Airflow Cluster

Последней статьей, которую я написал об Apache Airflow, была «**Установка и настройка Multi-Node Airflow Cluster с HDP Ambari и Celery для конвейеров данных**». Вопрос был в том, прошло ли все гладко или возникли проблемы. Да, были некоторые проблемы.

В этой статье я расскажу о тех проблемах, с которыми я столкнулся в своем путешествии по настройке Multi-Node Airflow Cluster.

**Проблема 1. После изменения LocalExecutor на CeleryExecutor группа доступности базы данных находилась в режиме выполнения, но на самом деле ни одна из задач не запускалась.**

Worker не смог связаться с Scheduler с помощью Celery.

Ошибка:

```
AttributeError: ‘DisabledBackend’ object has no attribute ‘_get_task_meta_for’
Apr 10 21:03:52 charlie-prod airflow_control.sh: [2019–04–10 21:03:51,962] {celery_executor.py:112} ERROR — Error syncing the celery executor, ignoring it:
Apr 10 21:03:52 charlie-prod airflow_control.sh: [2019–04–10 21:03:51,962] {celery_executor.py:113} ERROR — ‘DisabledBackend’ object has no attribute ‘_get_task_meta_for’
```

Я начал изучать код Airflow в той же строке, где возникла ошибка, но не понимал, что происходит. Но причина была ясна. Celery не могла публиковать сообщения или подписываться на них и не имела возможности участвовать в канале связи.

Решение:

Установленная версия Celery была 3.3.5 (которая слишком старая и несовместима с Airflow 1.10 (текущая установленная версия).

```
pip install --upgrade celery
3.3.5 => 4.3
```

**Проблема 2: после запуска DAG на CeleryExecutor DAG завершился с какой-то странной ошибкой, по крайней мере, для меня.**

```
Apr 11 14:13:13 charlie-prod airflow_control.sh: return load(BytesIO(s))
Apr 11 14:13:13 charlie-prod airflow_control.sh: TypeError: Required argument ‘object’ (pos 1) not found
Apr 11 14:13:13 charlie-prod airflow_control.sh: [2019–04–11 14:13:13,847: ERROR/ForkPoolWorker-6285] Pool process <celery.concurrency.asynpool.Worker object at 0x7f3a88b7b250> error: TypeError(“Required argument ‘object’ (pos 1) not found”,)
Apr 11 14:13:13 charlie-prod airflow_control.sh: Traceback (most recent call last):
Apr 11 14:13:13 charlie-prod airflow_control.sh: File “/usr/lib64/python2.7/site-packages/billiard/pool.py”, line 289, in __call__
Apr 11 14:13:13 charlie-prod airflow_control.sh: sys.exit(self.workloop(pid=pid))
Apr 11 14:13:13 charlie-prod airflow_control.sh: File “/usr/lib64/python2.7/site-packages/billiard/pool.py”, line 347, in workloop
Apr 11 14:13:13 charlie-prod airflow_control.sh: req = wait_for_job()
Apr 11 14:13:13 charlie-prod airflow_control.sh: File “/usr/lib64/python2.7/site-packages/billiard/pool.py”, line 447, in receive
Apr 11 14:13:13 charlie-prod airflow_control.sh: ready, req = _receive(1.0)
Apr 11 14:13:13 charlie-prod airflow_control.sh: File “/usr/lib64/python2.7/site-packages/billiard/pool.py”, line 419, in _recv
Apr 11 14:13:13 charlie-prod airflow_control.sh: return True, loads(get_payload())
Apr 11 14:13:13 charlie-prod airflow_control.sh: File “/usr/lib64/python2.7/site-packages/billiard/common.py”, line 101, in pickle_loads
Apr 11 14:13:13 charlie-prod airflow_control.sh: return load(BytesIO(s))
Apr 11 14:13:13 charlie-prod airflow_control.sh: TypeError: Required argument ‘object’ (pos 1) not found
```

Решение:

Я не смог ничего понять об этой ошибке.

Я изучил блог о airflow на китайском языке: https://blog.csdn.net/u013492463/article/details/80881260

Я ничего не понял, но, по крайней мере, получил небольшое представление о том, что может быть причиной этой ошибки.

Более ранняя настройка:

```
broker_url= amqp://guest:guest@{RabbitMQ-HOSTNAME}:5672/
```

Я понимаю, что pyamqp был бы лучшим выбором, так как многие люди использовали его, и более ранняя статья в некоторой степени давала такое же разрешение.

`amqp://` - это псевдоним, который использует `librabbitmq`, если он доступен, или `py-amqp`, если его нет.

Вы должны использовать `pyamqp://` или `librabbitmq://`, если хотите точно указать, какой протокол передачи данных использовать. Протокол передачи данных pyamqp: // использует библиотеку amqp (http://github.com/celery/py-amqp)

Позднее произвел установку с разрешением:

```
broker_url= pyamqp://guest:guest@{RabbitMQ-HOSTNAME}:5672/
```

изменение amqp на pyamqp устранило указанную выше ошибку.

Установка:

```
pip install pyamqp
```

**Проблема 3: сбой подключения SQL Alchemy**

Более ранняя конфигурация:

SQL alchemy connection

```
sql_alchemy_conn = postgresql://airflow:airflow@{HOST_NAME}:5432/airflow
```

Решение:

Более поздняя конфигурация:

```
sql_alchemy_conn = postgresql+psycopg2://airflow:airflow@{HOSTNAME}/airflow
```

Для `psycopg2` вам необходимо установить `pip` wheel.

Установите адаптер PostGreSQL: `psycopg2`

`Psycopg` - это адаптер PostgreSQL для языка программирования Python.

**Проблема 4: HDP версия 2.6.2 с Ambari, установка Worker Installation при сбое нескольких хостов.**

После успешной установки веб-сервера и планировщика на главном узле, то есть на узле имени, целью было установить Celery worker на всех узлах данных, чтобы группы DAG могли работать параллельно и масштабироваться по горизонтали и вертикали.

Но Амбари дал дурацкие выражения :) с ошибкой ниже.

```
by ‘SSLError(SSLError(“bad handshake: SysCallError(104, ‘ECONNRESET’)”,),)’: /simple/apache-airflow/
 Retrying (Retry(total=2, connect=None, read=None, redirect=None, status=None)) after connection broken by ‘SSLError(SSLError(“bad handshake: SysCallError(104, ‘ECONNRESET’)”,),)’: /simple/apache-airflow/
 Retrying (Retry(total=1, connect=None, read=None, redirect=None, status=None)) after connection broken by ‘SSLError(SSLError(“bad handshake: SysCallError(104, ‘ECONNRESET’)”,),)’: /simple/apache-airflow/
 Retrying (Retry(total=0, connect=None, read=None, redirect=None, status=None)) after connection broken by ‘SSLError(SSLError(“bad handshake: SysCallError(104, ‘ECONNRESET’)”,),)’: /simple/apache-airflow/
 Could not fetch URL https://pypi.org/simple/apache-airflow/: There was a problem confirming the ssl certificate: HTTPSConnectionPool(host=’pypi.org’, port=443): Max retries exceeded with url: /simple/apache-airflow/ (Caused by SSLError(SSLError(“bad handshake: SysCallError(104, ‘ECONNRESET’)”,),)) — skipping
```

Решение:

Это означает, что pip не может загрузить и установить wheel на машину, когда я пытался установить worker на узле с помощью пользовательского интерфейса Ambari. Однако с помощью команд терминала я смог запустить те же команды для установки wheel’s of pip.

Обычное решение для этой ошибки было запущено с аргументом доверенного пользователя или с изменением репозитория, из которого pypi загружает wheel.

```
pip install --trusted-host pypi.python.org --trusted-host pypi.org --trusted-host files.pythonhosted.org --upgrade  --ignore-installed apache-airflow[celery]==1.10.0' returned 1. Collecting apache-airflow[celery]==1.10.0
```

Пробовал сделать, что указал выше, но снова не удалось. Появился стек других ошибок, но с аналогичным значением.

```
resource_management.core.exceptions.ExecutionFailed: Execution of ‘export SLUGIFY_USES_TEXT_UNIDECODE=yes && pip install — trusted-host pypi.python.org — trusted-host pypi.org — trusted-host files.pythonhosted.org — upgrade — ignore-installed apache-airflow[celery]==1.10.0’ returned 1. Collecting apache-airflow[celery]==1.10.0
 Retrying (Retry(total=4, connect=None, read=None, redirect=None)) after connection broken by ‘ProtocolError(‘Connection aborted.’, error(104, ‘Connection reset by peer’))’: /simple/apache-airflow/
 Retrying (Retry(total=3, connect=None, read=None, redirect=None)) after connection broken by ‘ProtocolError(‘Connection aborted.’, error(104, ‘Connection reset by peer’))’: /simple/apache-airflow/
 Retrying (Retry(total=2, connect=None, read=None, redirect=None)) after connection broken by ‘ProtocolError(‘Connection aborted.’, error(104, ‘Connection reset by peer’))’: /simple/apache-airflow/
 Retrying (Retry(total=1, connect=None, read=None, redirect=None)) after connection broken by ‘ProtocolError(‘Connection aborted.’, error(104, ‘Connection reset by peer’))’: /simple/apache-airflow/
 Retrying (Retry(total=0, connect=None, read=None, redirect=None)) after connection broken by ‘ProtocolError(‘Connection aborted.’, error(104, ‘Connection reset by peer’))’: /simple/apache-airflow/
 Could not find a version that satisfies the requirement apache-airflow[celery]==1.10.0 (from versions: )
No matching distribution found for apache-airflow[celery]==1.10.0
You are using pip version 8.1.2, however version 19.0.3 is available.
You should consider upgrading via the ‘pip install — upgrade pip’ command.
```

Я обновил pip, но безуспешно.

Наконец я понял, Hack не может быть разрешением, которое сработало, - это команды, которые я установил в celery и все необходимые wheel pip, которые перечислены [здесь](https://medium.com/@khatri_chetan/how-to-setup-airflow-multi-node-cluster-with-celery-rabbitmq-cfde7756bb6a).

Но все же он выдал ту же ошибку. Но когда эти колеса были установлены, это не игнорировалось. Согласно коду https://github.com/miho120/ambari-airflow-mpack/blob/e1c9ca004adaa3320e35ab7baa7fdb9b9695b635/airflow-service-mpack/common-services/AIRFLOW/1.10.0/package/scripts/airflow_worker_control.py

В кластере я вручную закомментировал эти строки временно (**позже отменил изменения после успешной установки воркера**) и добавил воркера из Ambari, который работал как шарм :), и этот хак сделал мой день.

После установки worker на другом узле вам может потребоваться перезапуск службы airflowиз Ambari. Вы можете узнать больше из моей предыдущей статьи в блоге; [Настройка Multi-Node Airflow Cluster с HDP Ambari и Celery для конвейеров данных](https://medium.com/@khatri_chetan/setup-and-configure-multi-node-airflow-cluster-with-hdp-ambari-and-celery-for-data-pipelines-dc1e96f3d773)
