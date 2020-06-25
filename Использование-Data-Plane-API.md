# Использование HAProxy в качестве Data Plane Service Mesh в архитектуре микросервисов

HAProxy Data Plane API – это программа, которая работает вместе с HAProxy, чтобы вы могли полностью настроить балансировщик нагрузки HAProxy во время выполнения.

Эта функция позволяет использовать ряд вариантов использования, подходящих для динамически создаваемой конфигурации.

Лучшим примером является использование HAProxy в качестве Data Plane Service Mesh в архитектуре микросервисов, где центральная плоскость управления управляет конфигурацией набора прокси-серверов sidecar.

В этом разделе показано, как установить и настроить Data Plane API и начать работу с несколькими примерами.

См. также [документацию спецификации API](https://www.haproxy.com/documentation/dataplaneapi/latest/) для более подробной информации.

## Требования

- HAProxy Community 1.9 или новее (2.0 при использовании Process Manager)

- HAProxy Enterprise 1.9 или новее

- Рабочие знания HTTP и использование REST API


## Установка Data Plane API

### Установите Data Plane API вместе с HAProxy Enterprise

Data Plane API поставляется в виде системного пакета, который вы можете установить с помощью apt или yum.

Ниже приведен пример установки на сервере Ubuntu 18.04.

1. Экспортируйте свой корпоративный ключ HAProxy в переменной с именем haproxy_key и добавьте ключ репозитория HAProxy apt:

   ```
   $ haproxy_key=[YOUR KEY]
   $ curl -fsSL https://www.haproxy.com/download/hapee/key/$haproxy_key-common/HAPEE-key-1.9r1.asc | sudo apt-key add -
   ```

2. Выполните команду add-apt-repository, чтобы включить следующие репозитории пакетов:

   ```
   $ sudo add-apt-repository "deb [arch=amd64] https://www.haproxy.com/download/hapee/key/${haproxy_key}-common/1.9r1/ubuntu-18.04/amd64/ bionic main"
   $ sudo add-apt-repository "deb [arch=amd64] https://www.haproxy.com/download/hapee/key/${haproxy_key}-plus/1.9r1/ubuntu-18.04/amd64/ bionic main"
   $ sudo add-apt-repository "deb [arch=amd64] https://www.haproxy.com/download/hapee/key/${haproxy_key}-plus/extras/ubuntu-18.04/amd64/ bionic main"
   ```

3. Установите пакет Data Plane API следующим образом:

   ```
   $ sudo apt install hapee-extras-dataplane-api
   ```

Это добавляет программу по пути: /opt/hapee-extras/sbin/hapee-dataplane-api.

### Установите Data Plane API с HAProxy Community

Data Plane API поставляется в виде программы, которую вы можете скачать с Github.

После загрузки установите его разрешения на запуск в качестве исполняемого файла и скопируйте его в папку на вашем пути:

```
$ chmod +x dataplaneapi
$ sudo cp dataplaneapi /usr/local/bin/
```

### Настройка доступа к HAProxy

Data Plane API требует, чтобы вы включили базовую аутентификацию, что означает, что любой пользователь, вызывающий его методы, должен предоставить действительные учетные данные.

Имена пользователей и пароли хранятся в конфигурационном файле HAProxy внутри раздела userlist. Список пользователей – это раздел верхнего уровня, такой как frontend и backend.

Добавьте следующее в конфигурационный файл HAProxy `/etc/hapee-1.9/hapee-lb.cfg`:

```
userlist hapee-dataplaneapi
    user dataplaneapi insecure-password mypassword
```

В этом примере мы добавили одного пользователя с именем `dataplaneapi` и паролем `mypassword`.

Вы также можете зашифровать пароль с помощью команды `mkpasswd` из пакета `whois`.

```
$ sudo apt install -y whois
$ mkpasswd -m sha-256 mypassword
$5$aVnIFECJ$2QYP64eTTXZ1grSjwwdoQxK/AP8kcOflEO1Q5fc.5aA
```

Скопируйте и вставьте зашифрованный пароль в файл конфигурации:

Вам не нужно перезагружать HAProxy, чтобы `userlist` вступил в силу.

Однако если Data Plane API запущен, остановите его, а затем перезапустите.

Data Plane API работает в своем собственном процессе, что позволяет ему записывать и перезагружать конфигурацию HAProxy по мере необходимости.

### Настройка доступа к сокету HAProxy

Data Plane API нуждается в доступе для чтения и записи в сокет HAProxy.

Для этого обновите конфигурационный файл HAProxy так, чтобы он имел строку `stats socket` в разделе `global`:

1. Задайте параметры `user` и `group` системы, которые будут использовать Data Plane API. Например, с HAProxy Community, можно задать как для `haproxy`. Они должны быть установлены в `hapee-lb` и `hapee` соответственно для HAProxy Enterprise, как показано на ниже:

   ```
   global
       stats socket /run/haproxy.sock user hapee-lb group hapee mode 660 level admin
   ```

2. Если вы протестируете API, запустив его самостоятельно, вы можете добавить свое собственное имя пользователя в ту же группу:

   ```
   $ sudo usermod -a -G hapee $USER
   ```

   ```
   ПРИМЕЧАНИЕ
   
   Чтобы разрешения вступили в силу, выйдите из системы и снова входите в нее.
   ```

## Запуск Data Plane API

Основное использование API заключается в следующем:

```
hapee-dataplane-api [OPTIONS]
```

- Ниже перечислены параметры редактирования и управления экземплярами HAProxy.

- Вы можете вызвать hapee-dataplane-api с флагом `--help`, чтобы просмотреть список доступных опций.


### Параметры Application

Эти параметры управляют тем, как размещается Data Plane API и как он должен управлять завершением работы.

| Параметр             | Описание                                                     |
| -------------------- | ------------------------------------------------------------ |
| --scheme=            | Подключить listeners. Можно повторить и по умолчанию использовать схемы в спецификации swagger |
| --cleanup-timeout=   | Льготный период, в течение которого следует подождать, прежде чем уничтожать бездействующие соединения (default: 10s) |
| --graceful-timeout=  | Льготный период ожидания перед выключением сервера (default: 15s) |
| --max-header-size=   | Управляет максимальным числом байтов, которые сервер будет считывать при анализе ключей и значений заголовка запроса, включая строку запроса. Он не ограничивает размер тела запроса. (default:1MiB) |
| --socket-path=       | Сокет Unix для прослушивания (default: /var/run/data-plane.sock)) |
| --host=              | IP-адрес для прослушивания (default: localhost) [$HOST]      |
| --port=              | Порт для прослушивания небезопасных соединений; по умолчанию используется случайное значение [$PORT] |
| --listen-limit=      | Ограничивает количество невыполненных запросов               |
| --keep-alive=        | Устанавливает тайм-ауты TCP keep-alive для принятых подключений. Он обрезает мертвые TCP-соединения (например, закрытие ноутбука в середине загрузки) (default: 3m) |
| --read-timeout=      | Максимальная продолжительность ожидания для чтения запроса перед тайм-аутом (default: 30s) |
| --write-timeout=     | Максимальная продолжительность ожидания для записи ответа перед тайм-аутом (default: 60s) |
| --tls-host=          | IP-адрес для прослушивания TLS; если он не указан, то это то же самое, что и --host [$TLS_HOST] |
| --tls-port=          | Порт для прослушивания защищенных соединений; по умолчанию используется случайное значение [$TLS_PORT] |
| --tls-certificate=   | Сертификат, используемый для безопасных соединений [$TLS_CERTIFICATE] |
| --tls-key=           | Закрытый ключ, используемый для безопасных соединений [$TLS_PRIVATE_KEY] |
| --tls-ca=            | Файл центра сертификации для использования с mutual tls auth [$TLS_CA_CERTIFICATE] |
| --tls-listen-limit=  | Ограничивает количество невыполненных запросов               |
| --tls-keep-alive=    | Устанавливает тайм-ауты TCP keep-alive для принятых подключений. Он обрезает мертвые TCP-соединения (например, закрытие ноутбука в середине загрузки) |
| --tls-read-timeout=  | Максимальная продолжительность ожидания для чтения запроса перед тайм-аутом |
| --tls-write-timeout= | Максимальная продолжительность ожидания для записи ответа перед тайм-аутом |

### Параметры HAProxy

Эти параметры настраивают доступ и управление с помощью API работающего экземпляра HAProxy.

| Параметр               | Описание                                                     |
| ---------------------- | ------------------------------------------------------------ |
| -c, --config-file=     | Путь к HAProxy файл конфигурации (default: файл /etc/haproxy/haproxy.cfg) |
| -u, --userlist=        | Список пользователей в конфигурации HAProxy для использования для базовой аутентификации API (default: controller) |
| -b, --haproxy-bin=     | Путь к двоичному файлу HAProxy (default: haproxy)            |
| -d, --reload-delay=    | Минимальная задержка между двумя перезагрузками (в секундах) (default: 5) |
| -r, --reload-cmd=      | Команда Reload                                               |
| -s, --restart-cmd=     | Команда перезапуска                                          |
| --reload-retention=    | Сохранение перезагрузки в днях, каждый более старый идентификатор перезагрузки будет удален (default: 1) |
| -t, --transaction-dir= | Путь к каталогу транзакций (default: /tmp/haproxy)           |
| -n, --backups-number=  | Количество резервных конфигурационных файлов, которые вы хотите сохранить, хранящихся в config dir с суффиксом номера версии (default: 0) |
| -m, --master-runtime=  | Путь к основному сокету Runtime API                          |
| -i, --show-system-info | Показать системную информацию на конечной точке info         |

### Параметры ведения журнала

Эти параметры настраивают уровень ведения журнала для запросов к API.

| Параметр                                         | Описание                                                     |
| ------------------------------------------------ | ------------------------------------------------------------ |
| --log-to=[stdout\|file]                          | Цель журнала, может быть stdout или file (default: stdout)   |
| --log-file=                                      | Расположение файла журнала (default: /var/log/dataplaneapi/dataplaneapi.log) |
| --log-level=[trace\|debug\|info\|warning\|error] | Уровень ведения журнала (default: warning)                   |
| --log-format=[text\|JSON]                        | Формат ведения журнала (default: text)                       |

### Другие параметры

Эти параметры предоставляют дополнительную информацию об API.

| Параметр      | Описание                                    |
| ------------- | ------------------------------------------- |
| -v, --version | Показать информацию о версии и сборке       |
| -h, --help    | Показать эту справку (справочное сообщение) |

### Запустить Data Plane API с HAProxy Enterprise

Запуск Data Plane API с HAProxy Enterprise из консоли

```
$ sudo /opt/hapee-extras/sbin/hapee-dataplane-api \
--host 0.0.0.0 \
--port 5555 \
--haproxy-bin /opt/hapee-1.9/sbin/hapee-lb \
--config-file /etc/hapee-1.9/hapee-lb.cfg \
--reload-cmd "systemctl reload hapee-1.9-lb" \
--reload-delay 5 \
--userlist hapee-dataplaneapi \
--restart-cmd "systemctl restart hapee-1.9-lb"
```

### Запустить Data Plane API как сервис

При запуске в качестве службы параметры Data Plane API такие же, как и при использовании CLI. Вы можете установить эти параметры в файле `/etc/default/hapee-extras-dataplane-api`.

1. Заменить 1.9 в по умолчанию сервиса файл с вашей версией HAProxy Enterprise:

   ```
   $ sudo sed -i 's/1.9/1.9/g' /etc/default/hapee-extras-dataplane-api
   ```

2. Включите сервис:

   ```
   $ sudo systemctl enable hapee-extras-dataplane-api
   ```

3. Запустить службу:

   ```
   $ sudo systemctl enable hapee-extras-dataplane-api
   ```

### Отображение Data Plane API в браузере

При запущенном API откройте браузер и получите доступ к спецификации OpenAPI на порту 5555, например http://localhost:5555/v1/specification.

- По ссылке возвратиться документ JSON, который определяет все доступные методы API.

- В Firefox вы получаете удобочитаемую версию с разборными разделами. Узел пути содержит URL-адреса, которые сопоставляются с каждым методом.


![](https://cdn.haproxy.com/documentation/hapee/1-9r1/_assets/images/dataplaneapi_firefox.png)

### Тест методов (Test methods)

Вы можете использовать команду `curl` для тестирования вызова различных методов.

Вызовите метод `info`, который возвращает информацию о процессе:

```
$ curl -X GET --user dataplaneapi:mypassword http://localhost:5555/v1/services/haproxy/info

{"haproxy":{"processes":1,"release_date":"2019-04-02","time":"2019-04-08T22:07:39.291Z","uptime":1006,"version":"1.9.1-1.0.0-200.372"}}
```

Если вы получите эту ошибку:

```
{"code":500,"message":"dial unix /run/haproxy.sock: connect: permission denied"}
```

Это означает, что пользователь, который запускает API, не имеет доступа к сокету HAProxy. Убедитесь, что вы добавили его в группу HAProxy, и снова войдите в систему.

```
ПРИМЕЧАНИЕ

Задача Data Plane API состоит в том, чтобы полностью управлять конфигурационным файлом HAProxy. Поэтому каждый раз, когда вы редактируете конфигурацию вручную, убедитесь, что вы перезапустили API или отправили ему сигнал SIGUSR2, чтобы он мог снова прочитать файл. Ниже приведены примеры использования этих команд.
```

### Использование HAProxy Process Manager

При использовании HAProxy 2.0 или более поздней версии вы можете использовать HAProxy Process Manager для запуска Data Plane API. Process Manager добавляет новый раздел под названием `program` в конфигурацию HAProxy для запуска внешних процессов при запуске HAProxy.

### Настройка HAProxy для старта Data Plane API

Следующая конфигурация запускает Data Plane API при запуске процесса HAProxy. Строка `no option start-on-reload` позволяет избежать перезагрузки Data Plane API каждый раз, когда к HAProxy перезагружается.

1. Обновите конфигурацию HAProxy:

   ```
   program api
       command dataplaneapi --host 0.0.0.0 --port 5555 --haproxy-bin /usr/sbin/haproxy --config-file /etc/haproxy/haproxy.cfg --reload-cmd "systemctl reload haproxy" --reload-delay 5 --userlist hapee-dataplaneapi
       no option start-on-reload
   ```

2. Чтобы это сработало, вы должны запустить HAProxy в режиме `master-worker`, добавив директиву `master-worker` в раздел `global` (или добавив аргумент -W командной строки). Затем, когда вы просматриваете статус HAProxy, вы увидите новую программу, работающую вместе с рабочими процессами HAProxy.

   ```
   $ sudo systemctl restart haproxy
   $ sudo systemctl status haproxy
   
   Main PID: 1274 (haproxy)
       Tasks: 6
   Memory: 5.5M
       CPU: 2.838s
   CGroup: /system.slice/haproxy.service
       1274 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -sf 2768 -x /run/haproxy/admin.sock
       2768 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -sf 2662 -x /run/haproxy/admin.sock
       2830 /opt/hapee-extras/sbin/hapee-dataplane-api --host 0.0.0.0 --port 5555 -b /usr/local/sbin/haproxy -c /etc/haproxy/haproxy.cfg -d 5 -r systemctl reload haproxy -u controller
       2831 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -sf 2768 -x /run/haproxy/admin.sock
   ```

3. Если HAProxy запускается внутри контейнера Docker в режиме master-worker (который используется по умолчанию), вы можете использовать команду `kill-SIGUSR2 [PID]` для аргумента `--reload-cmd`, где PID всегда равен 1, чтобы перезагрузить только рабочие процессы HAProxy без завершения главного процесса контейнера.

   ```
   program api
       command /etc/haproxy/dataplaneapi --host 0.0.0.0 --port 5555 --haproxy-bin /usr/sbin/haproxy --config-file /etc/haproxy/haproxy.cfg --reload-cmd "kill -SIGUSR2 1" --reload-delay 5 --userlist hapee-dataplaneapi
       no option start-on-reload
   ```

### Использование Data Plane API

В этом разделе мы покажем вам команды, которые вы можете использовать с Data Plane API.

#### GET запросы 

При извлечении данных с помощью GET-запросов вам не нужны никакие дополнительные параметры URL-адреса. Например, чтобы получить список интерфейсов в вашей конфигурации, используйте:

```
$ curl -X GET --user admin:mypassword http://localhost:5555/v1/services/haproxy/configuration/frontends

{"_version":1,"data":[{"default_backend":"be_web","name":"fe_main"}]}
```

Это возвращает документ JSON с двумя ключами верхнего уровня: `_version` и `data`.

- Версия отображается в верхней части конфигурационного файла HAProxy и увеличивается всякий раз, когда вы записываете данные с помощью запроса POST, PUT или DELETE. Это гарантирует, что изменения в файле не конфликтуют.

- Ключ `data` содержит результат выполнения команды, который в данном случае представляет собой массив объектов, описывающих каждый интерфейс.


#### POST запросы 

При записи данных, например с помощью POST-запроса, необходимо включить версию в URL-адрес. Если он не соответствует текущей версии, вы получаете ошибку о несоответствии версии.

Ниже приведен пример запроса POST на добавление нового интерфейса:

```
СОВЕТ

Это хорошая идея, чтобы заключить URL-адрес в кавычки, так что оболочка не интерпретирует любой параметр URL с префиксом амперсанда как запрос на выполнение команды в фоновом режиме.
```

```
$ curl -X POST --user admin:mypassword \
  -H "Content-Type: application/json" \
  -d '{"name": "test_frontend", "default_backend": "be_web", "mode": "http", "maxconn": 2000}' \
  "http://localhost:5555/v1/services/haproxy/configuration/frontends?version=1"
```

#### Использование транзакций

При выполнении приведенного выше запроса возникают следующие проблемы:

- Если вы не определили `be_web`, то получите сообщение об ошибке, указывающее, что API не может найти серверную часть.

- API создает `frontend` без `bind` линии.


Чтобы устранить эти проблемы, вы можете выполнить следующие команды только один раз в следующей последовательности:

1. Создать бэкенд `be_web`

2. Добавить сервера к бэкенду

3. Создание внешнего интерфейса

4. Добавьте строку `bind` к `frontend`

Каждая из этих операций требует отдельной команды; и пока вы не выполнили их все, вы продолжаете перезагружать конфигурацию HAProxy с допустимыми, но неполными данными. Решение состоит в том, чтобы использовать транзакцию, которая хранит команды, а затем применяет их все сразу.

1. Инициализация транзакции: 

   ```
   $ curl -X POST --user admin:mypassword \
   -H "Content-Type: application/json" \
   http://localhost:5555/v1/services/haproxy/transactions?version=5
   ```

   Вернет:

   ```
   {"_version":5,"id":"b9a0ecd7-b8ae-4ef8-8865-0cd5e38396cd","status":"in_progress"}
   ```

Затем каждая команда, выполняемая в рамках этой транзакции, включает параметр `transaction_id` в URL-адрес. Для просмотра всех транзакций используйте следующую команду:

```
$ curl -X GET --user admin:mypassword \
-H "Content-Type: application/json" \
"http://localhost:5555/v1/services/haproxy/transactions"
```

2. Добавить бэкэнд:

   ```
   $ curl -X POST --user admin:mypassword \
   -H "Content-Type: application/json" \
   -d '{"name": "test_backend", "mode":"http", "balance": {"algorithm":"roundrobin"}, "httpchk": {"method": "HEAD", "uri": "/check", "version": "HTTP/1.1"}}' \
   "http://localhost:5555/v1/services/haproxy/configuration/backends?transaction_id=b9a0ecd7-b8ae-4ef8-8865-0cd5e38396cd"
   ```

3. Добавить сервер в бэкэнд:

   ```
   $ curl -X POST --user admin:mypassword \
   -H "Content-Type: application/json" \
   -d '{"name": "server1", "address": "127.0.0.1", "port": 8080, "check": "enabled", "maxconn": 30, "weight": 100}' \
   "http://localhost:5555/v1/services/haproxy/configuration/servers?backend=test_backend&transaction_id=b9a0ecd7-b8ae-4ef8-8865-0cd5e38396cd"
   ```

4. Добавьте `frontend`:

   ```
   $ curl -X POST --user admin:mypassword \
   -H "Content-Type: application/json" \
   -d '{"name": "test_frontend", "mode": "http", "default_backend": "test_backend", "maxconn": 2000}' \
   "http://localhost:5555/v1/services/haproxy/configuration/frontends?transaction_id=b9a0ecd7-b8ae-4ef8-8865-0cd5e38396cd"
   ```

5. Добавьте строку `bind` к `frontend` :

   ```
   $ curl -X POST --user admin:mypassword \
   -H "Content-Type: application/json" \
   -d '{"name": "http", "address": "*", "port": 80}' \
   "http://localhost:5555/v1/services/haproxy/configuration/binds?frontend=test_frontend&transaction_id=b9a0ecd7-b8ae-4ef8-8865-0cd5e38396cd"
   ```

6. После завершения транзакции примените изменения:

   ```
   $ curl -X PUT --user admin:mypassword \
   -H "Content-Type: application/json" \
   "http://localhost:5555/v1/services/haproxy/transactions/b9a0ecd7-b8ae-4ef8-8865-0cd5e38396cd"
   ```

#### Monitor Configuration Reloads

Data Plane API с `--reload-delay` или `-d` для установки временного интервала между перезагрузками. Например, если вы установили задержку в пять секунд, то каждые пять секунд программа проверяет, не было ли каких-либо изменений, требующих перезагрузки HAProxy. Если ничего не найдено, API ждет еще пять секунд и проверяет снова. Все изменения, которые происходят в промежутке между проверками, ждут следующей проверки, прежде чем они будут применены. Другими словами, команда не вызывает перезагрузку. Перезагрузка происходит автоматически через равные промежутки времени по мере необходимости.

Используйте конечную точку `/services/haproxy/reloads`, чтобы получить список выполняемых и неудачных перезагрузок. Это может быть использовано для проверки того, что конфигурация была успешно обновлена.

Можно перезагрузить HAProxy сразу же, минуя параметр force_reload=true URL при вызове команды.

### Пример конфигурации

Следующий пример добавляет ACL с именем `is_api` к интерфейсу с именем `test_frontend`.

1. Добавьте ACL с именем `is_api`:

   ```
   $ curl -X POST --user admin:mypassword \
   -H "Content-Type: application/json" \
   -d '{"id": 0, "acl_name": "is_api", "criterion": "path_beg", "value": "/api"}' \
   "http://localhost:5555/v1/services/haproxy/configuration/acls?parent_type=frontend&parent_name=test_frontend&version=4"
   ```

   Результат:

   ```
   frontend test_frontend
       mode http
       maxconn 2000
       acl is_api path_beg /api
       default_backend test_backend
   ```

2. Добавьте строку `use_backend`, которая ссылается на ACL `is_api`:

   ```
   $ curl -X POST --user admin:mypassword \
   -H "Content-Type: application/json" \
   -d '{"id": 0, "cond": "if", "cond_test": "is_api", "name": "test_backend"}' \
   "http://localhost:5555/v1/services/haproxy/configuration/backend_switching_rules?frontend=test_frontend&version=5"
   ```

   Результат:

   ```
   frontend test_frontend
       mode http
       maxconn 2000
       acl is_api path_beg /api
       use_backend test_backend if is_api
       default_backend test_backend
   ```

3. Удалите строку `use_backend` (обратите внимание, что мы передаем `id` 0 в URL-адрес):

   ```
   $ curl -X DELETE --user admin:mypassword \
   -H "Content-Type: application/json" \
   "http://localhost:5555/v1/services/haproxy/configuration/backend_switching_rules/0?frontend=test_frontend&version=6"
   ```

4. Добавьте встроенный ACL, который отклоняет все запросы, кроме запросов от localhost:

   ```
   $ curl -X POST --user admin:mypassword \
   -H "Content-Type: application/json" \
   -d '{"id": 0, "cond": "unless", "cond_test": "{ src 127.0.0.1 }", "type": "deny"}' \
   "http://localhost:5555/v1/services/haproxy/configuration/http_request_rules?parent_type=frontend&parent_name=test_frontend&version=7"
   ```

   Конфигурация HAProxy теперь выглядит следующим образом:

   ```
   frontend test_frontend
       mode http
       maxconn 2000
       acl is_api path_beg /api
       http-request deny unless { src 127.0.0.1 }
       default_backend test_backend
   ```

### Summary

В данном разделе представлена HAProxy Data Plane API, который позволяет настроить HAProxy HTTP-запросы к процессу, который запускается независимо от HAProxy.

Существуют методы для создания прокси, бэкендов, серверов, ACLs (списков управления доступом), правил таблиц и многого другого.

Смотрите дополнительные сведения о доступных командах в [документации по спецификации API](https://www.haproxy.com/documentation/dataplaneapi/latest/). 

P.S. Телеграм чат HAproxy https://t.me/haproxy_ru