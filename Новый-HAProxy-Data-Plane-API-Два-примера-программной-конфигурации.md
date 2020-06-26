**Новый HAProxy Data Plane API: два примера программной конфигурации**

Используйте HAProxy Data Plane API для динамического управления конфигурацией балансировщика нагрузки с помощью HTTP-команд.

Проектирование для высокой доступности почти всегда означает наличие высокого уровня прокси / балансировщика нагрузки. Прокси-сервер предоставляет основные услуги, такие как:

- обнаружение и удаление неисправных серверов
- очередь соединений
- разгрузка шифрования TLS
- компрессия
- кэширование

Задача состоит в том, чтобы поддерживать ваши конфигурации в актуальном состоянии, что особенно сложно, поскольку сервисы перемещаются в контейнеры, и эти контейнеры становятся эфемерными.  Доступный начиная с [HAProxy 2.0](https://www.haproxy.com/blog/haproxy-2-0-and-beyond/) вы можете использовать новый [HAProxy Data Plane API](https://www.haproxy.com/documentation/hapee/1-9r1/configuration/dataplaneapi/), который является современным REST API.

HAProxy Data Plane API дополняет [гибкий язык конфигурации](https://www.haproxy.com/blog/the-four-essential-sections-of-an-haproxy-configuration/) HAProxy, который предоставляет строительные блоки для определения простых и сложных правил маршрутизации. Это также идеальное дополнение к существующему [Runtime API](https://www.haproxy.com/blog/dynamic-configuration-haproxy-runtime-api/), которое позволяет запускать, останавливать и пропускать трафик с серверов, изменять вес сервера и управлять проверками работоспособности.

Новый Data Plane API дает вам возможность динамически добавлять и настраивать внешние интерфейсы, внутренние интерфейсы и логику маршрутизации трафика. Вы также можете использовать его для обновления конечных точек ведения журнала и создания фильтров SPOE. По сути, почти весь балансировщик нагрузки может быть настроен с помощью HTTP-команд. В этой статье вы узнаете, как лучше его использовать.

**Управление конфигурацией**

Обычно при настройке HAProxy вам нужно вручную отредактировать файл конфигурации `/etc/haproxy/haproxy.cfg`. Этот файл используется для управления всеми функциями балансировщика нагрузки. Он в основном разделен на `frontend` разделы, определяющие общедоступные IP-адреса, к которым подключаются клиенты, и `backend` разделы, содержащие вышестоящие сервера, на которые направляется трафик. Но вы можете сделать гораздо больше, в том числе установить глобальные параметры, влияющие на работающий процесс, установить значения по умолчанию, добавить анализ поведения трафика с помощью таблиц флеш-памяти, прочитать файлы карт, определить правила фильтрации с помощью ACL и многое другое.

Хотя редактировать файл вручную довольно просто, это не всегда целесообразно, особенно при работе с десятками или даже сотнями сервисов и прокси. Это позволяет всему трафику проходить от прокси к прокси, по существу абстрагируя сеть и ее непостоянство от смежных сервисов приложений. Прокси на этом уровне могут добавить к вашим услугам  логику повторных попыток, авторизацию и шифрование TLS. Тем не менее, количество прокси будет быстро расти, так как для каждой услуги есть свой прокси.

В этом случае крайне важно иметь возможность вызвать HTTP API для динамического обновления определений прокси. В сервисной сетке программное обеспечение Data Plane контролирует прокси и динамически вызывает API конфигурации. HAProxy Data Plane API позволяет интегрировать HAProxy с этими платформами. Более того, API использует API времени выполнения для внесения изменений, которые по возможности не требуют перезагрузки.

Data Plane API использует Go пакеты [config-parser](https://github.com/haproxytech/config-parser) и [client-native](https://github.com/haproxytech/client-native) для парсинга конфигурации HAProxy и вызова команд Runtime API соответственно. Вы можете использовать их в своих собственных проектах для интеграции с HAProxy.

**Динамическое конфигурирование HAProxy**

С помощью Data Plane API вы можете многое сделать. В этом разделе вы узнаете, как создать `backend` с серверами и `frontend`, который направляет трафик на него. Но сначала следуйте [официальной документации](https://www.haproxy.com/documentation/hapee/1-9r1/configuration/dataplaneapi/) по установке и настройке API.

После того как он будет  установлен и запущен, вызовите GET в конечной точке **/v1/services/haproxy/configuration/backends**, чтобы увидеть уже определенные разделы `backend`, например:

```
$ curl --get --user admin:mypassword \
    http://localhost:5555/v1/services/haproxy/configuration/backends
```

Если вы хотите добавить новый `backend`, вызовите тот же endpoint с помощью POST. Есть два способа внести изменения в это состояние - индивидуальные вызовы или пакетные команды внутри транзакции. Поскольку мы хотим внести несколько связанных изменений, давайте начнем с создания транзакции.

Вызовите endpoint **/v1/services/haproxy/transactions** для создания новой транзакции. Для этого потребуется параметр версии в URL, но командам внутри транзакции он не нужен. Всякий раз, когда вызывается команда POST, PUT или DELETE, должна быть включена версия, которая затем записывается в файл конфигурации HAProxy. Это гарантирует, что если несколько клиентов будут использовать API, они избежат конфликтов. Если версия, которую вы передаете, не соответствует версии, указанной в файле конфигурации, вы получите сообщение об ошибке. При использовании транзакции эта версия указывается заранее при ее создании.

```
$ curl -X POST --user admin:mypassword \
       -H "Content-Type: application/json" \
       http://localhost:5555/v1/services/haproxy/transactions?version=1
```

Вы найдете идентификатор транзакции в возвращенном документе JSON:

```
{"_version":5,"id":"9663c384-5052-4776-a968-abcef032aeef","status":"in_progress"}
```

Затем используйте endpoint **/v1/services/haproxy/configuration/backends**, чтобы создать новый бэкэнд, отправив идентификатор транзакции в качестве параметра URL:

```
$ curl -X POST --user admin:mypassword \
       -H "Content-Type: application/json" \
       -d '{"name": "test_backend", "mode":"http", "balance": {"algorithm":"roundrobin"}, "httpchk": {"method": "HEAD", "uri": "/", "version": "HTTP/1.1"}}' \     
       http://localhost:5555/v1/services/haproxy/configuration/backends?transaction_id=9663c384-5052-4776-a968-abcef032aeef
```

Затем вызовите endpoint **/v1/services/haproxy/configuration/servers** для добавления серверов в `backend`:

```
$ curl -X POST --user admin:mypassword \
       -H "Content-Type: application/json" \
       -d '{"name": "server1", "address": "127.0.0.1", "port": 8080, "check": "enabled", "maxconn": 30, "weight": 100}' \
       "http://localhost:5555/v1/services/haproxy/configuration/servers?backend=test_backend&transaction_id=9663c384-5052-4776-a968-abcef032aeef"

```

Затем добавьте `frontend` с помощью endpoint **/v1/services/haproxy/configuration/frontends** :

```
$ curl -X POST --user admin:mypassword \
       -H "Content-Type: application/json" \
       -d '{"name": "test_frontend", "mode": "http", "default_backend": "test_backend", "maxconn": 2000}' \
       http://localhost:5555/v1/services/haproxy/configuration/frontends?transaction_id=9663c384-5052-4776-a968-abcef032aeef
```

У этого `frontend` еще нет никаких `bind` операторов. Добавьте одно, используя endpoint **/v1/services/haproxy/configuration/binds**, как на примере:

```
$ curl -X POST --user admin:mypassword \
       -H "Content-Type: application/json" \
       -d '{"name": "http", "address": "*", "port": 80}' \
       "http://localhost:5555/v1/services/haproxy/configuration/binds?frontend=test_frontend&transaction_id=9663c384-5052-4776-a968-abcef032aeef"
```

Затем, чтобы зафиксировать транзакцию и применить все изменения, вызовите endpoint **/v1/services/haproxy/transactions/[transaction ID]** с помощью PUT, например:

```
$ curl -X PUT --user admin:mypassword \
       -H "Content-Type: application/json" \
       http://localhost:5555/v1/services/haproxy/transactions/9663c384-5052-4776-a968-abcef032aeef
```

Вот как выглядит конфигурация сейчас:

```
frontend test_frontend
    mode http
    maxconn 2000
    bind *:80 name http
    default_backend test_backend

backend test_backend
    mode http
    balance roundrobin
    option httpchk HEAD / HTTP/1.1
    server server1 127.0.0.1:8080 check maxconn 30 weight 100
```

Этот балансировщик нагрузки готов принимать трафик и перенаправлять его на вышестоящий сервер.

Поскольку в спецификации Data Plane API используется OpenAPI, его можно использовать для [генерации клиентского кода](https://github.com/OpenAPITools/openapi-generator) на многих поддерживаемых языках программирования.

Во время этого упражнения мы упаковали все команды внутри транзакции. Вы также можете вызывать их одну за другой. В этом случае вместо включения параметра URL с именем `transaction_id` вы должны включить одну `версию`, которая будет увеличиваться с каждым вызовом.

**Другой пример**

Вы увидели простоту и мощь HAProxy Data Plane API. С помощью нескольких HTTP-команд вы можете динамически изменять конфигурацию. Давайте рассмотрим другой пример. Мы создадим ACL, который будет проверять имеет ли заголовок *Host* значение **example.com**. Если это так, то строка **use_backend** будет направлена на другой сервер с именем **example_servers**. Мы также добавим правило **http-request deny**, которое будет отклонять любые запросы для пути URL **/admin.php**, если исходный IP-адрес клиента не находится в сети **192.168.50.20/24**.

Сначала используйте endpoint **/v1/services/haproxy/transactions** для создания новой транзакции и получения ее идентификатора:

```
$ curl -X POST --user admin:mypassword \
       -H "Content-Type: application/json" \ 
       http://localhost:5555/v1/services/haproxy/transactions?version=2

{"_version":2,"id":"7d0d6737-655e-4489-92eb-6d29cdd69827","status":"in_progress"}
```

Затем используйте endpoint **/v1/services/haproxy/configuration/backends** вместе с идентификатором транзакции, чтобы создать новый `backend` с именем *example_servers*:

```
$ curl -X POST --user admin:mypassword \
       -H "Content-Type: application/json" \
       -d '{"name": "example_servers", "mode":"http", "balance": {"algorithm":"roundrobin"}}' \
       http://localhost:5555/v1/services/haproxy/configuration/backends?transaction_id=7d0d6737-655e-4489-92eb-6d29cdd69827
```

Используйте endpoint **/v1/services/haproxy/configuration/servers** для добавления **server** в **backend**:

```
$ curl -X POST --user admin:mypassword \
       -H "Content-Type: application/json" \
       -d '{"name": "server1", "address": "127.0.0.1", "port": 8081, "check": "enabled", "maxconn": 30, "weight": 100}' \
       "http://localhost:5555/v1/services/haproxy/configuration/servers?backend=example_servers&transaction_id=7d0d6737-655e-4489-92eb-6d29cdd69827"
```

Используйте endpoint **/v1/services/haproxy/configuration/acls**, чтобы определить ACL с именем *is_example*, который проверяет, имеет ли заголовок узла значение **example.com**:

```
$ curl -X POST --user admin:mypassword \
       -H "Content-Type: application/json" \
       -d '{"id": 0, "acl_name": "is_example", "criterion": "req.hdr(Host)", "value": "example.com"}' \
       "http://localhost:5555/v1/services/haproxy/configuration/acls?parent_type=frontend&parent_name=test_frontend&transaction_id=7d0d6737-655e-4489-92eb-6d29cdd69827"
```

Используйте **/v1/services/haproxy/configuration/backend_switching_rules**, чтобы добавить строку **use_backend**, которая оценивает ACL *is_example*:

```
$ curl -X POST --user admin:mypassword \
       -H "Content-Type: application/json" \
       -d '{"id": 0, "cond": "if", "cond_test": "is_example", "name": "example_servers"}' \
       "http://localhost:5555/v1/services/haproxy/configuration/backend_switching_rules?frontend=test_frontend&transaction_id=7d0d6737-655e-4489-92eb-6d29cdd69827"
```

Используйте endpoint **/v1/services/haproxy/configuration/http_request_rules**, чтобы добавить правило **http-request deny**, которое проверяет, является ли путь **/admin.php**, а исходный IP-адрес клиента не находится в сети **192.168.50.20/24**:

```
$ curl -X POST --user admin:mypassword \
       -H "Content-Type: application/json" \
       -d '{"id": 0, "cond": "if", "cond_test": "{ path /admin.php } !{ src 192.168.50.20/24 }", "type": "deny"}' \
       "http://localhost:5555/v1/services/haproxy/configuration/http_request_rules?parent_type=frontend&parent_name=test_frontend&transaction_id=7d0d6737-655e-4489-92eb-6d29cdd69827"
```

Затем подтвердите транзакцию, чтобы изменения вступили в силу:

```
$ curl -X PUT --user admin:mypassword \
       -H "Content-Type: application/json" \
       http://localhost:5555/v1/services/haproxy/transactions/7d0d6737-655e-4489-92eb-6d29cdd69827
```

Ваша конфигурация HAProxy теперь выглядит вот так:

```
frontend test_frontend
    mode http
    maxconn 2000
    bind *:80 name http
    acl is_example req.hdr(Host) example.com
    http-request deny deny_status 0 if { path /admin.php } !{ src 192.168.50.20/24 }
    use_backend example_servers if is_example
    default_backend test_backend

backend example_servers
    mode http
    balance roundrobin
    server server1 127.0.0.1:8081 check maxconn 30 weight 100

backend test_backend
    mode http
    balance roundrobin
    option httpchk HEAD / HTTP/1.1
    server server1 127.0.0.1:8080 check maxconn 30 weight 100
```

**Заключение**

В этой статье вы ознакомились с HAProxy Data Plane API, который позволяет полностью настроить HAProxy с использованием современного REST API. Более подробную информацию можно найти в [официальной документации](https://www.haproxy.com/documentation/hapee/1-9r1/configuration/dataplaneapi/) (Перевод: https://habr.com/ru/post/508132/). У него имеются три мощных функции, которые включают гибкий язык конфигурации HAProxy и API времени выполнения. Data Plane API открывает двери для многостороннего использования, в частности, с использованием HAProxy в качестве уровня прокси в сетке сервиса.

Мы рады будущему использованию нового API для создания новых партнерских отношений и разнообразных функций. HAProxy продолжает обеспечивать высокую производительность и устойчивость в любой среде и в любом масштабе.

Если вам понравилась эта статья и вы хотите быть в курсе похожих тем, подпишитесь на этот блог. Вы также можете подписаться на нас в [Твиттере](https://twitter.com/haproxy) и присоединиться к разговору на [Slack](https://slack.haproxy.com/). HAProxy Enterprise позволяет легко приступить к работе с Data Plane API, поскольку его можно установить в виде удобного системного пакета. Он также включает в себя надежную и передовую кодовую базу, корпоративный набор дополнений, экспертную поддержку и профессиональные услуги. Хотите узнать больше? Свяжитесь с нами сегодня и скачайте бесплатную пробную версию.

P.S. Телеграм чат HAproxy https://t.me/haproxy_ru



