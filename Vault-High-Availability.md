**Режим высокой доступности Vault (HA)**

Vault может работать в режиме высокой доступности (HA) для защиты от сбоев за счет запуска нескольких серверов Vault. Vault обычно ограничивается пределами операций ввода-вывода серверной части Vault, а не требованиями к вычислениям. Некоторые серверные модули хранения, например Consul, предоставляют дополнительные функции координации, которые позволяют Vault работать в конфигурации высокой доступности, в то время как другие обеспечивают более надежный процесс резервного копирования и восстановления. 

При работе в режиме высокой доступности серверы Vault имеют два дополнительных состояния: **резервное** и **активное**. В кластере Vault активным будет только один экземпляр, который будет обрабатывать все запросы (чтение и запись), а все резервные узлы перенаправляют запросы на активный узел.

![](https://habrastorage.org/webt/tp/pu/vj/tppuvjc9gewbzpbc21magajkesc.png)

ПРИМЕЧАНИЕ. Начиная с версии **0.11**, эти резервные узлы могут обрабатывать большинство запросов только для чтения и вести себя как узлы реплики для чтения. Эта функция **Performance Standby Nodes** включена в Vault Enterprise Premium, а также доступна для Vault Enterprise Pro за дополнительную плату. Это особенно полезно для обработки больших объемов запросов. Прочтите документацию по рабочим узлам и руководство для получения более подробной информации.

 

Это руководство проведет вас через простую реализацию кластера Vault Highly Available (HA). Хотя это не исчерпывающее или предписывающее руководство, которое можно использовать в качестве примера прямого подключения, оно охватывает основы, достаточные для информирования вашей собственной производственной установки.

 

**Примерное время на выполнение** 

25 минут 

 

**Предпосылки**

Это промежуточное руководство по работе с Vault предполагает, что у вас есть практические знания о Vault и Consul.

 

**Шаги**

Наша цель в этом руководстве, - настроить Vault HA, состоящий из: 

·    2 серверов Vault:  1 активный и 1 резервный 

·    Кластер из 3-х серверов Consul  

 

**Справочная диаграмма** 

На этой диаграмме для справки показаны простые детали архитектуры:

 ![](https://habrastorage.org/webt/3r/td/8-/3rtd8-zmebletxt7dnrh_zdox7e.png)

Мы будем выполнять эти шаги: 

- Шаг 1. Настройка кластера серверов Consul 

- Шаг 2. Запуск и проверка состояния кластера Consul 

- Шаг 3. Настройка клиентских агентов Consul на узлах Vault 

- Шаг 4. Настройка серверов Vault 

- Шаг 5. Запуск Vault и его проверка 


 

Для реализации целей этого руководства мы будем использовать программное обеспечение с открытым исходным кодом Vault и Consul; настройка будет аналогична для версии Enterprise.

 

**Шаг 1. Настройка кластера серверов Consul** 

Наши серверы Consul в этом руководстве будут определяться только IP-адресом, но также обозначены меткой:

- **`consul_s1: 10.1.42.101`**
- **`consul_s2: 10.1.42.102`**
- **`consul_s3: 10.1.42.103`**

Двоичный файл Consul находится в **`/usr/local/bin/consul`**, но если в вашем случае не так, вы можете соответствующим образом изменить ссылки на пути.

Имея это в виду, вот начальная точка базовой конфигурации сервера Consul:

```
{
  "server": true,
  "node_name": "$NODE_NAME",
  "datacenter": "dc1",
  "data_dir": "$CONSUL_DATA_PATH",
  "bind_addr": "0.0.0.0",
  "client_addr": "0.0.0.0",
  "advertise_addr": "$ADVERTISE_ADDR",
  "bootstrap_expect": 3,
  "retry_join": ["$JOIN1", "$JOIN2", "$JOIN3"],
  "ui": true,
  "log_level": "DEBUG",
  "enable_syslog": true,
  "acl_enforce_version_8": false
}
```

Обратите внимание, что некоторые значения содержат заполнители переменных, а остальные имеют разумные значения по умолчанию. Вы должны заменить следующие значения в вашей конфигурации сервера Consul на основе примера:

- **$NODE_NAME** - это уникальная метка для узла; в нашем случае это будут `consul_s1`, `consul_s2` и `consul_s3` соответственно. 
- **$CONSUL_DATA_PATH**: абсолютный путь к каталогу данных Consul; убедитесь, что этот каталог доступен для записи пользователю процесса Consul. 
- **$ADVERTISE_ADDR**: задает адрес, чтобы серверы Consul объявляли другим серверам в кластере. Не должно быть установлено в `0.0.0.0`; для этого руководства он должен быть установлен на IP-адрес сервера Consul в каждом экземпляре файла конфигурации или `10.1.42.101`,`10.1.42.102` и `10.1.42.103` соответственно. 
- **$JOIN1**, **$JOIN2**, **$JOIN3**: в этом примере используется метод `retry_join` для присоединения агентов сервера к формированию кластера; Таким образом, значения для этого руководства будут `10.1.42.101`, `10.1.42.102` и `10.1.42.103` соответственно.

 

Обратите внимание, что веб-интерфейс пользователя включен (`"ui": true`), и Consul будет вести журнал на уровне DEBUG в системный журнал (`"log_level": "DEBUG"`). Для целей этого руководства `acl_enforce_version_8` установлено в `false`, поэтому нам не нужно беспокоиться о списках ACL в этом руководстве. Тем не менее, вы могли бы включить списки ACL в производственной среде и следовать [руководству Consul ACL](https://www.consul.io/docs/guides/acl.html#acl-agent-master-token). 

Создайте файл конфигурации для каждого сервера Vault и сохраните его как `/usr/local/etc/consul/client_agent.json`. 

 

Пример `consul_s1.json` 

```
{
  "server": true,
  "node_name": "consul_s1",
  "datacenter": "dc1",
  "data_dir": "/var/consul/data",
  "bind_addr": "0.0.0.0",
  "client_addr": "0.0.0.0",
  "advertise_addr": "10.1.42.101",
  "bootstrap_expect": 3,
  "retry_join": ["10.1.42.101", "10.1.42.102", "10.1.42.103"],
  "ui": true,
  "log_level": "DEBUG",
  "enable_syslog": true,
  "acl_enforce_version_8": false
}
```

Пример `consul_s2.json`

```
{
  "server": true,
  "node_name": "consul_s2",
  "datacenter": "dc1",
  "data_dir": "/var/consul/data",
  "bind_addr": "0.0.0.0",
  "client_addr": "0.0.0.0",
  "advertise_addr": "10.1.42.102",
  "bootstrap_expect": 3,
  "retry_join": ["10.1.42.101", "10.1.42.102", "10.1.42.103"],
  "ui": true,
  "log_level": "DEBUG",
  "enable_syslog": true,
  "acl_enforce_version_8": false
}
```

Пример `consul_s3.json`

```
{
  "server": true,
  "node_name": "consul_s3",
  "datacenter": "dc1",
  "data_dir": "/var/consul/data",
  "bind_addr": "0.0.0.0",
  "client_addr": "0.0.0.0",
  "advertise_addr": "10.1.42.103",
  "bootstrap_expect": 3,
  "retry_join": ["10.1.42.101", "10.1.42.102", "10.1.42.103"],
  "ui": true,
  "log_level": "DEBUG",
  "enable_syslog": true,
  "acl_enforce_version_8": false
}
```

Консул Сервер `systemd` 

У вас есть двоичные файлы Consul и базовая конфигурация, и теперь вам просто нужно запустить Consul на каждом экземпляре сервера; `systemd` популярен в большинстве современных дистрибутивов Linux, поэтому, имея это в виду, вот пример файла systemd unit:

```
### BEGIN INIT INFO
# Provides:          consul
# Required-Start:    $local_fs $remote_fs
# Required-Stop:     $local_fs $remote_fs
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Consul agent
# Description:       Consul service discovery framework
### END INIT INFO

[Unit]
Description=Consul server agent
Requires=network-online.target
After=network-online.target

[Service]
User=consul
Group=consul
PIDFile=/var/run/consul/consul.pid
PermissionsStartOnly=true
ExecStartPre=-/bin/mkdir -p /var/run/consul
ExecStartPre=/bin/chown -R consul:consul /var/run/consul
ExecStart=/usr/local/bin/consul agent \
    -config-file=/usr/local/etc/consul/client_agent.json \
    -pid-file=/var/run/consul/consul.pid
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
KillSignal=SIGTERM
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```

Обратите внимание, что вас может заинтересовать изменение следующих значений в зависимости от стиля, стандартного уровня соблюдения иерархии файловой системы и т. Д. –

- **config-file** 
- **pid-file**

 

После того как файл модуля определен и сохранен (например, `/etc/systemd/system/consul.service`), обязательно выполните перезагрузку демона `systemctl daemon-reload`, а затем вы можете запустить службу Consul на каждом сервере.

 

**Шаг 2. Запуск и проверка состояния кластера Consul**

Убедитесь, что права собственности и разрешения верны в каталоге, который вы указали для значения `data_dir`, а затем запустите службу Consul в каждой системе и проверьте статус:

```
$ sudo systemctl start consul
$ sudo systemctl status consul
● consul.service - Consul server agent
   Loaded: loaded (/etc/systemd/system/consul.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2018-03-19 17:33:14 UTC; 24h ago
 Main PID: 2068 (consul)
    Tasks: 13
   Memory: 13.6M
      CPU: 0m 52.784s
   CGroup: /system.slice/consul.service
           └─2068 /usr/local/bin/consul agent -config-file=/usr/local/etc/consul/client_agent.json -pid-file=/var/run/consul/consul.pid
```

После запуска всех агентов сервера Consul, давайте проверим статус кластера Consul:

```
$consul members
Node       Address           Status  Type    Build  Protocol  DC    Segment
consul_s1  10.1.42.101:8301  alive   server  1.0.6  2         dc1   <all>
consul_s2  10.1.42.102:8301  alive   server  1.0.6  2         dc1   <all>
consul_s3  10.1.42.103:8301  alive   server  1.0.6  2         dc1   <all>
```

Кластер выглядит хорошо, показаны все 3 сервера; Прежде чем продолжить, убедитесь, что у нас есть лидер:

```
$consul operator raft list-peers
Node                   ID                                    Address           State     Voter  RaftProtocol
consul_s2              536b721f-645d-544a-c10d-85c2ca24e4e4  10.1.42.102:8300  follower  true   3
consul_s1              e10ba554-a4f9-6a8c-f662-81c8bb2a04f5  10.1.42.101:8300  follower  true   3
consul_s3              56370ec8-da25-e7dc-dfc6-bf5f27978a7a  10.1.42.103:8300  leader    true   3
```

Приведенный выше вывод показывает, что `consul_s3` является текущим лидером кластера в этом примере. Теперь вы можете перейти к настройке сервера Vault.

 

**Шаг 3. Настройка клиентских агентов Consul на узлах Vault**

Узлам сервера Vault требуются двоичные файлы Consul и Vault на каждом узле. Consul будет настроен как агент клиента, а Vault будет настроен как сервер.

 ![](https://habrastorage.org/webt/ix/xr/uw/ixxruwdww2kyrawybzeyddz2l8q.png)

**Конфигурация агента клиента Consul**

Поскольку Consul используется для обеспечения серверной части высокодоступного хранилища, вам необходимо настроить локальные клиентские агенты Consul на серверах Vault, которые будут взаимодействовать с кластером серверов Consul для регистрации проверок работоспособности, обнаружения служб и координации переключения HA кластера (лидерство кластера).

Обратите внимание, что [не рекомендуется подключать серверы Vault напрямую к серверам Consul.](https://fix-tag--vault-www.netlify.app/docs/configuration/storage/consul#address)

Агенты клиента Consul будут использовать тот же адрес, что и серверы Vault, для сетевой связи с кластером серверов Consul, но они будут связывать `client_address` только с интерфейсом обратной связи, так что Vault может подключиться к нему через интерфейс обратной связи. 

Вот пример конфигурации для агента клиента Consul:

```
{
  "server": false,
  "datacenter": "dc1",
  "node_name": "$NODE_NAME",
  "data_dir": "$CONSUL_DATA_PATH",
  "bind_addr": "$BIND_ADDR",
  "client_addr": "127.0.0.1",
  "retry_join": ["$JOIN1", "$JOIN2", "$JOIN3"],
  "log_level": "DEBUG",
  "enable_syslog": true,
  "acl_enforce_version_8": false
}
```

Аналогично тому, что вы сделали на **шаге 1**, замените следующие значения в своей конфигурации агента клиента Consul соответственно: 

- **$NODE_NAME** - это уникальная метка для узла; в нашем случае это будут `consul_c1` и `consul_c2` соответственно. 
- **$CONSUL_DATA_PATH**: абсолютный путь к каталогу данных Consul; убедитесь, что этот каталог доступен для записи пользователю процесса Consul. 
- **$BIND_ADDR**: это должно быть установлено на адрес, который вы предпочитаете, чтобы серверы Consul объявляли другим серверам в кластере, и не должно быть установлено на `0.0.0.0`; в этом руководстве он должен быть установлен на IP-адрес сервера Vault в каждом экземпляре файла конфигурации или `10.1.42.201` и `10.1.42.202` соответственно. 
- **$JOIN1**, **$JOIN2**, **$JOIN3**: в этом примере используется метод `retry_join` для присоединения агентов сервера для формирования кластера; Таким образом, значения для этого руководства будут `10.1.42.101`, `10.1.42.102` и `10.1.42.103` соответственно.

 

Создайте файл конфигурации для каждого сервера Vault и сохраните его как `/usr/local/etc/consul/client_agent.json`.

 

Пример consul_c1.json

```
{
  "server": false,
  "datacenter": "dc1",
  "node_name": "consul_c1",
  "data_dir": "/var/consul/data",
  "bind_addr": "10.1.42.201",
  "client_addr": "127.0.0.1",
  "retry_join": ["10.1.42.101", "10.1.42.102", "10.1.42.103"],
  "log_level": "DEBUG",
  "enable_syslog": true,
  "acl_enforce_version_8": false
}
```

Пример consul_c2.json

```
{
  "server": false,
  "datacenter": "dc1",
  "node_name": "consul_c2",
  "data_dir": "/var/consul/data",
  "bind_addr": "10.1.42.202",
  "client_addr": "127.0.0.1",
  "retry_join": ["10.1.42.101", "10.1.42.102", "10.1.42.103"],
  "log_level": "DEBUG",
  "enable_syslog": true,
  "acl_enforce_version_8": false
}
```

#### systemd юнит файл Consul сервера 

 

У вас есть двоичные файлы Consul и базовая конфигурация агента клиента, и теперь вам просто нужно запустить Consul на каждом из экземпляров сервера Vault. Вот пример файла модуля `systemd`:

```
### BEGIN INIT INFO
# Provides:          consul
# Required-Start:    $local_fs $remote_fs
# Required-Stop:     $local_fs $remote_fs
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Consul agent
# Description:       Consul service discovery framework
### END INIT INFO

[Unit]
Description=Consul client agent
Requires=network-online.target
After=network-online.target

[Service]
User=consul
Group=consul
PIDFile=/var/run/consul/consul.pid
PermissionsStartOnly=true
ExecStartPre=-/bin/mkdir -p /var/run/consul
ExecStartPre=/bin/chown -R consul:consul /var/run/consul
ExecStart=/usr/local/bin/consul agent \
    -config-file=/usr/local/etc/consul/client_agent.json \
    -pid-file=/var/run/consul/consul.pid
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
KillSignal=SIGTERM
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```

При необходимости измените следующие значения: 

- **-config-file**
- **-pid-file**

 

После того как файл модуля определен и сохранен (например, `/etc/systemd/system/consul.service`), обязательно выполните перезагрузку демона `systemctl daemon-reload`, а затем вы можете запустить службу Consul на каждом сервере Vault. 

Запустите Consul и проверьте его состояние кластера, чтобы убедиться, что права собственности и разрешения верны в каталоге, который вы указали для значения `data_dir`, а затем запустите службу Consul в каждой системе и проверьте статус:

```
$ sudo systemctl start consul
$ sudo systemctl status consul
● consul.service - Consul client agent
   Loaded: loaded (/etc/systemd/system/consul.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2018-03-20 19:36:49 UTC; 6s ago
 Main PID: 23758 (consul)
    Tasks: 11
   Memory: 9.8M
      CPU: 571ms
   CGroup: /system.slice/consul.service
           └─23758 /usr/local/bin/consul agent -config-file=/usr/local/etc/consul/client_agent.json -pid-file=/var/run/consul/consul.pid
```

После запуска всех клиентских агентов Consul проверьте статус кластера Consul:

```
$consul members
Node        Address           Status  Type    Build  Protocol  DC    Segment
consul_s1   10.1.42.101:8301  alive   server  1.0.6  2         dc1   <all>
consul_s2   10.1.42.102:8301  alive   server  1.0.6  2         dc1   <all>
consul_s3   10.1.42.103:8301  alive   server  1.0.6  2         dc1   <all>
consul_c1   10.1.42.201:8301  alive   client  1.0.6  2         arus  <default>
consul_c2   10.1.42.202:8301  alive   client  1.0.6  2         arus  <default>
```

Приведенные выше выходные данные показывают 3 агента сервера Consul и 2 агента клиента Consul в кластере. Теперь вы можете настроить серверы Vault.

 

**Шаг 4. Настройка серверов Vault**

Теперь, когда у нас есть кластер Consul, состоящий из 3-х серверов и 2-х клиентских агентов для наших серверов Vault, давайте вместе получим конфигурацию для Vault и сценарий запуска, чтобы мы могли запустить настройку Vault HA.

Наши серверы Vault в этом руководстве определяются только IP-адресом, но также обозначаются меткой:

- **vault_s1:     10.1.42.201**
- **vault_s2:     10.1.42.202**

В нашем файле конфигурации мы настроим следующее:

- [**tcp**](https://fix-tag--vault-www.netlify.app/docs/configuration/listener/tcp) listener
- [**consul**](https://fix-tag--vault-www.netlify.app/docs/configuration/storage/consul) storage backend
- [Параметры высокой доступности](https://fix-tag--vault-www.netlify.app/docs/configuration/#high-availability-parameters)

В этом разделе предполагается, что двоичный файл Vault находится в **`/usr/local/bin/vault`**.

 

**Конфигурация Vault**

```
listener "tcp" {
  address          = "0.0.0.0:8200"
  cluster_address  = "0.0.0.0:8201"
  tls_disable      = "true"
}

storage "consul" {
  address = "127.0.0.1:8500"
  path    = "vault/"
}

api_addr =  "$API_ADDR"
cluster_addr = "$CLUSTER_ADDR"
```

Мы устанавливаем следующие параметры для нашего `tcp`-слушателя:

- [`address`](https://fix-tag--vault-www.netlify.app/guides/operations/vault-ha-consul.html#address) ("127.0.0.1:8200") - указывает адрес, к которому нужно привязаться для прослушивания. 
- [`cluster_address`](https://fix-tag--vault-www.netlify.app/guides/operations/vault-ha-consul.html#cluster_address) ("127.0.0.1:8201") - указывает адрес для привязки для запросов кластера сервер-сервер. По умолчанию это значение на один порт больше, чем значение адреса. Обычно это не требуется настраивать, но может быть полезно в случае, если серверы Vault изолированы друг от друга таким образом, что им нужно переключаться через балансировщик нагрузки TCP или какую-либо другую схему для разговора.

Эта конфигурация позволяет прослушивать все интерфейсы (например, так, чтобы команда Vault для адреса обратной связи была успешной).

Мы также устанавливаем [параметры высокой доступности](https://fix-tag--vault-www.netlify.app/docs/configuration/#high-availability-parameters) Vault (`api_addr` и `cluster_addr`). Чаще всего нет необходимости настраивать эти два параметра при использовании Consul в качестве серверной части хранилища Vault, поскольку Consul будет пытаться автоматически обнаруживать и объявлять адрес активного узла Vault. Однако для некоторых конфигураций кластера может потребоваться их явная установка (например, доступ к Vault через балансировщик нагрузки). 

Для простоты предположим, что клиенты в нашем сценарии подключаются непосредственно к узлам Vault (а не через балансировщик нагрузки). Просмотрите документацию [Client Redirection](https://fix-tag--vault-www.netlify.app/docs/concepts/ha#client-redirection), чтобы получить дополнительную информацию о шаблонах доступа клиентов и их последствиях. 

Обратите внимание, что некоторые значения содержат заполнители переменных, а остальные имеют разумные значения по умолчанию. Вы должны заменить следующие значения в конфигурации вашего собственного сервера Vault на основе примера:

- **$API_ADDR**: указывает адрес (полный URL) для объявления другим серверам Vault в кластере для перенаправления клиентов. Это также можно сделать с помощью переменной среды `VAULT_API_ADDR`. Как правило, это должен быть полный URL-адрес, указывающий на значение адреса слушателя. В нашем сценарии это будут http://10.1.42.201:8200 и http://10.1.42.202:8200 соответственно. 
- **$CLUSTER_ADDR**: указывает адрес для объявления другим серверам Vault в кластере для пересылки запросов. Это также можно сделать с помощью переменной среды `VAULT_CLUSTER_ADDR`. Это полный URL, например `api_addr`. В нашем сценарии это будут https://10.1.42.201:8201 и https://10.1.42.202:8201 соответственно.

 

Обратите внимание, что схема здесь (https) игнорируется; все члены кластера всегда будут использовать TLS с закрытым ключом / сертификатом.

Пример vault_s1.hcl 

```
listener "tcp" {
  address          = "0.0.0.0:8200"
  cluster_address  = "10.1.42.201:8201"
  tls_disable      = "true"
}

storage "consul" {
  address = "127.0.0.1:8500"
  path    = "vault/"
}

api_addr = "http://10.1.42.201:8200"
cluster_addr = "https://10.1.42.201:8201"
```

Пример vault_s2.hcl 

```
listener "tcp" {
  address          = "0.0.0.0:8200"
  cluster_address  = "10.1.42.202:8201"
  tls_disable      = "true"
}

storage "consul" {
  address = "127.0.0.1:8500"
  path    = "vault/"
}

api_addr = "http://10.1.42.202:8200"
cluster_addr = "https://10.1.42.202:8201"
```

**systemd юнит файл Vault сервера** 

У вас есть двоичные файлы Vault и базовая конфигурация вместе с настроенными локальными агентами клиента. Теперь вам просто нужно запустить Vault на каждом экземпляре сервера. Вот пример файла модуля `systemd`:

```
### BEGIN INIT INFO
# Provides:          vault
# Required-Start:    $local_fs $remote_fs
# Required-Stop:     $local_fs $remote_fs
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Vault server
# Description:       Vault secret management tool
### END INIT INFO

[Unit]
Description=Vault secret management tool
Requires=network-online.target
After=network-online.target

[Service]
User=vault
Group=vault
PIDFile=/var/run/vault/vault.pid
ExecStart=/usr/local/bin/vault server -config=/etc/vault/vault_server.hcl -log-level=debug
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
KillSignal=SIGTERM
Restart=on-failure
RestartSec=42s
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target
```

Обратите внимание, что вас может заинтересовать изменение следующих значений в зависимости от стиля, стандартного уровня соблюдения иерархии файловой системы и т. д.

- **-config**
- **-log-level**

Как только файл модуля определен и сохранен, например, `/etc/systemd/system/vault.service`, обязательно выполните перезагрузку демона `systemctl daemon-reload`, а затем вы можете запустить службу Vault на каждом сервере.

 

**Шаг 5. Запуск Vault и его проверка**

Запустите службу Vault в каждой системе и проверьте состояние:

```
$ sudo systemctl start vault
$ sudo systemctl status vault
● vault.service - Vault secret management tool
   Loaded: loaded (/etc/systemd/system/vault.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2018-03-20 20:42:10 UTC; 42s ago
 Main PID: 2080 (vault)
    Tasks: 12
   Memory: 71.7M
      CPU: 50s
   CGroup: /system.slice/vault.service
           └─2080 /usr/local/bin/vault server -config=/home/ubuntu/vault_nano/config/vault_server.hcl -log-level=debu
```

Теперь вам нужно перейти к [инициализации и распечатыванию](https://fix-tag--vault-www.netlify.app/intro/getting-started/deploy#initializing-the-vault) каждого экземпляра Vault. 

Как только это будет сделано, проверьте статус Vault на каждом из серверов. 

Активный сервер Vault:

```
$ vault status
Key             Value
---             -----
Seal Type       shamir
Sealed          false
Total Shares    5
Threshold       3
Version         0.9.5
Cluster Name    vault
Cluster ID      0ee91bd1-55ec-c84f-3c1d-dcc7f4f644a8
HA Enabled      true
HA Cluster      https://10.1.42.201:8201
HA Mode         active
```

Резервный сервер Vault:

```
vault status
Key                     Value
---                     -----
Seal Type               shamir
Sealed                  false
Total Shares            5
Threshold               3
Version                 0.9.5
Cluster Name            vaultron
Cluster ID              0ee91bd1-55ec-c84f-3c1d-dcc7f4f644a8
HA Enabled              true
HA Cluster              https://10.1.42.201:8201
HA Mode                 standby
Active Node Address:    http://10.1.42.201:8200
```

Теперь серверы Vault  работают в режиме высокой доступности (HA), и вы можете записать секрет из активного или резервного экземпляра Vault и увидеть его успешное выполнение в качестве теста пересылки запросов. Кроме того, вы можете закрыть активный экземпляр (`sudo systemctl stop vault`), чтобы смоделировать сбой системы и увидеть, как резервный экземпляр принимает на себя лидерство.


**Следующие шаги**

Прочтите "[Повышение безопасности](https://fix-tag--vault-www.netlify.app/guides/operations/production)", чтобы узнать о передовых методах развертывания Vault для повышения уровня защиты в production среде.

