Практическое руководство по HashiCorp Consul - Часть 2
https://medium.com/velotio-perspectives/a-practical-guide-to-hashicorp-consul-part-2-3c0ebc0351e8

![](https://habrastorage.org/webt/r-/-d/a0/r--da0gbyafieifqbiwxezr48a4.jpeg)

Это вторая часть из 2 частей Практического руководства по HashiCorp Consul. Предыдущая часть была в первую очередь сосредоточена на понимании проблем, которые решает Consul, и на том, как он их решает. Эта часть посвящена практическому применению Consul на примере реальной жизни. Давайте начнем.

Развивая большую часть теории, рассмотренной в предыдущей части, давайте перейдем к практическому примеру Consul.

 

### Что мы строим?

Мы собираемся создать [веб-приложение Django](https://www.djangoproject.com/), которое хранит свои постоянные данные в [MongoDB](https://www.mongodb.com/). Мы поместим их в [контейнер](https://www.docker.com/resources/what-container) с помощью [Docker](https://www.docker.com/). Скомпилируйте и запустите их с помощью [Docker Compose](https://docs.docker.com/compose/).

Чтобы показать, как наше веб-приложение будет масштабироваться в этом контексте, мы собираемся запустить два экземпляра приложения Django. Кроме того, чтобы сделать это еще более интересным, мы запустим MongoDB как [набор реплик](https://docs.mongodb.com/manual/replication) с одним первичным узлом и двумя вторичными узлами.

Учитывая, что у нас есть два экземпляра приложения Django, нам понадобится способ сбалансировать нагрузку между этими двумя экземплярами, поэтому мы собираемся использовать [Fabio](https://fabiolb.net/), балансировщик нагрузки, поддерживающий Consul, для доступа к экземплярам приложения Django.

Этот пример примерно поможет нам смоделировать реальное практическое приложение.

 ![](https://habrastorage.org/webt/yq/z5/1d/yqz51ddx8ce0iqv_0we5l4rwxb0.png)

*Примеры узлов приложений и развернутых на них служб*

 

Полный исходный код этого приложения является открытым и доступен на GitHub - [pranavcode/consul-demo](pranavcode/consul-demo).

*Примечание. Обсуждаемая здесь архитектура не накладывает особых ограничений на какие-либо технологии, используемые для создания уровней приложений или данных. Этот пример вполне может быть построен с использованием комбинации Ruby on Rails и Postgres, или Node.js и MongoDB, или Laravel и MySQL.*

 

### Какое место занимает Консул?

Мы развертываем приложение и слои данных с контейнерами Docker. Они будут построены как службы и будут общаться друг с другом по HTTP.

Таким образом, мы будем использовать [Consul для Service Discovery](https://velotio.com/blog/2019/3/11/hashicorp-consul-guide-1). Это позволит серверам Django найти [первичный узел MongoDB](https://docs.mongodb.com/manual/core/replica-set-primary). В этом примере мы собираемся использовать Consul для разрешения служб через [интерфейс Consul DNS](https://www.consul.io/docs/agent/dns.html).

Consul также поможет нам с [автоматической настройкой Fabio](https://github.com/fabiolb/fabio/wiki/Configuration) в качестве балансировщика нагрузки для доступа к экземплярам нашего приложения Django.

Мы также используем функцию проверки работоспособности Consul, чтобы отслеживать работоспособность каждого из наших экземпляров во всей инфраструктуре.

Consul предоставляет красивый пользовательский интерфейс, как часть своего веб-интерфейса, чтобы показать все службы на одной панели. Мы будем использовать его, чтобы увидеть, как устроены наши услуги.

Давайте начнем.

 

### Настройка: MongoDB, Django, Consul, Fabio и Dockerization

Мы сделаем это настолько быстро и просто, насколько это возможно, чтобы продемонстрировать вам все возможности.

 

#### MongoDB

Мы нацелены на настройку MongoDB в форме набора реплик MongoDB. Один первичный узел и два вторичных узла.

Первичный узел будет управлять всеми [операциями записи](https://docs.mongodb.com/manual/core/replica-set-write-concern) и [журналом операций](https://docs.mongodb.com/manual/core/replica-set-oplog), чтобы поддерживать последовательность записи и реплицировать данные между [вторичными узлами](https://docs.mongodb.com/manual/core/replica-set-secondary). Мы также настраиваем вторичные серверы для [операций чтения](https://docs.mongodb.com/manual/core/read-preference). Вы можете узнать больше о [MongoDB Replica Set](https://docs.mongodb.com/manual/tutorial/deploy-replica-set) в их официальной документации.

Мы будем называть наш набор репликации `consuldemo`.

Мы запустим MongoDB на [стандартном порту 27017](https://docs.mongodb.com/manual/reference/default-mongodb-port) и укажем имя реплики, установленной в командной строке, с помощью параметра `--replSet`.

Как вы можете прочитать из документации, MongoDB также позволяет настраивать имя набора реплик через [файл конфигурации](https://docs.mongodb.com/manual/reference/configuration-options) с параметром репликации, как показано ниже:

 ```
replication:
    replSetName: "consuldemo"
 ```

В нашем случае конфигурация набора репликации, которую мы будем применять на одном из узлов MongoDB, после того, как все узлы будут запущены, выглядит следующим образом:

 ```
var config = {
    _id: "consuldemo",
    version: 1,
    members: [{
        _id: 0,
        host: "mongo_1:27017",
    }, {
        _id: 1,
        host: "mongo_2:27017",
    }, {
        _id: 2,
        host: "mongo_3:27017",
    }],
    settings: { 
        chainingAllowed: true 
    }
};
rs.initiate(config, { force: true });
rs.slaveOk();
db.getMongo().setReadPref("nearest");
 ```



Эта конфигурация будет применена к одному из предопределенных узлов, и [MongoDB решит, какой узел будет основным и дополнительным](https://docs.mongodb.com/manual/core/replica-set-elections).

*Примечание. Мы не форсируем создание набора с предопределенными обозначениями того, кто станет основным и второстепенным, чтобы обеспечить динамизм в обнаружении услуг. Обычно узлы определяются для конкретной роли.*

Мы разрешаем ведомой реплике читать с ближайшего узла в качестве [предпочтения чтения](https://docs.mongodb.com/manual/core/read-preference).

Мы запустим MongoDB на всех узлах с помощью следующей команды:

 ```
mongod --bind_ip_all --port 27017 --dbpath /data/db --replSet "consuldemo"
 ```



Это дает нам набор реплик MongoDB с одним первичным экземпляром и двумя вторичными экземплярами, которые работают и готовы принимать соединения.

Мы обсудим контейнеризацию службы MongoDB в последней части этой статьи.

 

#### Django

Мы создадим простой [проект Django](https://realpython.com/django-setup/), представляющий приложение Blog, и контейнер его с помощью Docker.

[Создание приложения Django с нуля](https://docs.djangoproject.com/en/2.1/intro/tutorial01/) выходит за рамки этого руководства, мы рекомендуем вам обратиться к [официальной документации Django](https://docs.djangoproject.com/en/2.1/), чтобы начать работу с проектом Django. Но мы все же рассмотрим некоторые важные аспекты.

Поскольку нам нужно, чтобы наше приложение Django взаимодействовало с MongoDB, мы будем использовать коннектор MongoDB для Django ORM, [Djongo](https://nesdis.github.io/djongo/). Мы настроим наши настройки Django для использования Djongo и подключения к нашей MongoDB. Djongo довольно прост в настройке.

Для локальной установки MongoDB потребуется всего две строки кода:

```
...

DATABASES = {
    'default': {
        'ENGINE': 'djongo',
        'NAME': 'db',
    }
}

...
```

В нашем случае, поскольку нам понадобится доступ к MongoDB через другой контейнер, наша конфигурация будет выглядеть так:

 ```
...

DATABASES = {
    'default': {
        'ENGINE': 'djongo',
        'NAME': 'db',
        'HOST': 'mongodb://mongo-primary.service.consul',
        'PORT': 27017,
    }
}

...
@velotiotech
 ```



Детали:

·    ENGINE: Коннектор базы данных для использования в Django ORM.

·    NAME: Имя базы данных.

·    HOST: адрес хоста, на котором работает MongoDB.

·    PORT: какой порт MongoDB прослушивает запросы.

Djongo внутренне общается с [PyMongo](https://api.mongodb.com/python/current/) и использует [MongoClient](http://api.mongodb.com/python/current/api/pymongo/mongo_client.html) для выполнения запросов в Mongo. Мы также можем использовать другие коннекторы MongoDB, доступные для Django, чтобы достичь этого, например, [django-mongodb-engine](https://django-mongodb-engine.readthedocs.io/) или [pymongo](https://github.com/mongodb/mongo-python-driver) напрямую, в зависимости от наших потребностей.

*Примечание. В настоящее время мы читаем и записываем через Django на один хост MongoDB, основной, но мы можем настроить Djongo так, чтобы он также разговаривал со второстепенными хостами для операций только для чтения. Это не входит в рамки нашего обсуждения. Вы можете обратиться к официальной документации Djongo, чтобы добиться именно этого.*

Продолжая процесс создания приложения Django, нам нужно определить наши модели. Поскольку мы создаем приложение, похожее на блог, наши модели будут выглядеть следующим образом:

```
from djongo import models

class Blog(models.Model):
    name = models.CharField(max_length=100)
    tagline = models.TextField()

    class Meta:
        abstract = True

class Entry(models.Model):
    blog = models.EmbeddedModelField(
        model_container=Blog,
    )
    
    headline = models.CharField(max_length=255)
```



Мы можем запустить локальный экземпляр MongoDB и создать миграции для этих моделей. Кроме того, можем зарегистрировать эти модели в вашем администраторе Django, например:

```
from django.contrib import admin
from.models import Entry

admin.site.register(Entry)
```



В этом примере мы можем поиграть с CRUD-операциями модели Entry через Django Admin.

Кроме того, для реализации возможности подключения Django-MongoDB мы создадим настраиваемый вид и шаблон, который отображает информацию о настройке MongoDB и подключенном в данный момент хосте MongoDB.

Наши настройки Django выглядят так:

```
from django.shortcuts import render
from pymongo import MongoClient

def home(request):
    client = MongoClient("mongo-primary.service.consul")
    replica_set = client.admin.command('ismaster')

    return render(request, 'home.html', { 
        'mongo_hosts': replica_set['hosts'],
        'mongo_primary_host': replica_set['primary'],
        'mongo_connected_host': replica_set['me'],
        'mongo_is_primary': replica_set['ismaster'],
        'mongo_is_secondary': replica_set['secondary'],
    })
```



Наша конфигурация URL-адресов или маршрутов для приложения выглядит так:

```
from django.urls import path
from tweetapp import views

urlpatterns = [
    path('', views.home, name='home'),
]
```



А для проекта - URL-адреса приложений включены так:

```
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
    path('web', include('tweetapp.urls')),
] + static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)
```



Наш шаблон Django, `templates/home.html`, выглядит так:

 ```
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
    
    <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css" integrity="sha384-ggOyR0iXCbMQv3Xipma34MD+dH/1fQ784/j6cY/iJTQUOhcWr7x9JvoRxT2MZw1T" crossorigin="anonymous">
    <link href="https://fonts.googleapis.com/css?family=Armata" rel="stylesheet">

    <title>Django-Mongo-Consul</title>
</head>
<body class="bg-dark text-white p-5" style="font-family: Armata">
    <div class="p-4 border">
        <div class="m-2">
            <b>Django Database Connection</b>
        </div>
        <table class="table table-dark">
            <thead>
                <tr>
                    <th scope="col">#</th>
                    <th scope="col">Property</th>
                    <th scope="col">Value</th>
                </tr>
            </thead>
            <tbody>
                <tr>
                    <td>1</td>
                    <td>Mongo Hosts</td>
                    <td>{% for host in mongo_hosts %}{{ host }}<br/>{% endfor %}</td>
                </tr>
                <tr>
                    <td>2</td>
                    <td>Mongo Primary Address</td>
                    <td>{{ mongo_primary_host }}</td>
                </tr>
                <tr>
                    <td>3</td>
                    <td>Mongo Connected Address</td>
                    <td>{{ mongo_connected_host }}</td>
                </tr>
                <tr>
                    <td>4</td>
                    <td>Mongo - Is Primary?</td>
                    <td>{{ mongo_is_primary }}</td>
                </tr>
                <tr>
                    <td>5</td>
                    <td>Mongo - Is Secondary?</td>
                    <td>{{ mongo_is_secondary }}</td>
                </tr>
            </tbody>
        </table>
    </div>
    
    <script src="https://code.jquery.com/jquery-3.3.1.slim.min.js" integrity="sha384-q8i/X+965DzO0rT7abK41JStQIAqVgRVzpbzo5smXKp4YfRvH+8abtTE1Pi6jizo" crossorigin="anonymous"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.14.7/umd/popper.min.js" integrity="sha384-UO2eT0CpHqdSJQ6hJty5KVphtPhzWj9WO1clHTMGa3JDZwrnQq4sF86dIHNDz0W1" crossorigin="anonymous"></script>
    <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/js/bootstrap.min.js" integrity="sha384-JjSmVgyd0p3pXB1rRibZUAYoIIy6OrQ6VrjIEaFf/nJGzIxFDsf4x0xIM+B07jRM" crossorigin="anonymous"></script>
</body>
</html>
 ```



Чтобы запустить приложение, нам нужно сначала перенести базу данных, используя следующую команду:

```
python ./manage.py migrate
```



А также собрать все статические активы в статический каталог:

```
python ./manage.py collectstatic --noinput
```



Теперь запустите приложение Django с Gunicorn, HTTP-сервером WSGI, как показано ниже:

```
gunicorn --bind 0.0.0.0:8000 --access-logfile - tweeter.wsgi:application
```



Это дает нам базовое приложение Django, подобное блогу, которое подключается к бэкэнду MongoDB.

Мы обсудим контейнеризацию этого веб-приложения Django в последней части этой статьи.

 

#### Consul

Мы размещаем [агент Consul](https://www.consul.io/docs/agent/basics.html) для каждой службы в рамках нашей установки Consul.

Агент Consul отвечает за обнаружение службы путем [регистрации службы в кластере Consul](https://velotio.com/blog/2019/3/11/hashicorp-consul-guide-1), а также контролирует работоспособность каждого экземпляра службы.

 

### Consul на узлах, на которых работает MongoDB Replica Set

Сначала мы обсудим настройку Consul в контексте MongoDB Replica Set, поскольку она решает интересную проблему. В любой момент времени один из экземпляров MongoDB может быть первичным или вторичным.

Агент Consul, регистрирующий и отслеживающий наш экземпляр MongoDB в наборе реплик, имеет уникальный механизм - динамическую регистрацию и отмену регистрации службы MongoDB в качестве первичного или вторичного экземпляра в зависимости от того, какой набор реплик назначил его.

Мы достигаем этого динамизма путем написания и запуска сценария оболочки после интервала, который переключает определение службы Consul для MongoDB Primary и MongoDB Secondary на Consul Agent узла экземпляра.

Определения сервисов для сервисов MongoDB хранятся в виде файлов JSON в каталоге конфигурации Consul `/etc/config.d`.

Определение службы для первичного экземпляра MongoDB:

 ```
{
    "service": {
        "name": "mongo-primary",
        "port": 27017,
        "tags": [
            "nosql",
            "database"
        ],
        "check": {
            "id": "mongo_primary_status",
            "name": "Mongo Primary Status",
            "args": ["/etc/consul.d/check_scripts/mongo_primary_check.sh"],
            "interval": "30s",
            "timeout": "20s"
        }
    }
}
 ```



Если вы присмотритесь, определение службы позволяет нам получить запись DNS, специфичную для MongoDB Primary, а не общий экземпляр MongoDB. Это позволяет нам отправлять записи базы данных в конкретный экземпляр MongoDB. В случае набора реплик записи поддерживаются MongoDB Primary.

Таким образом, мы можем достичь как обнаружения сервисов, так и мониторинга работоспособности для первичного экземпляра MongoDB.

Точно так же с небольшим изменением определение службы для вторичного экземпляра MongoDB выглядит следующим образом:

 ```
{
    "service": {
        "name": "mongo-secondary",
        "port": 27017,
        "tags": [
            "nosql",
            "database"
        ],
        "check": {
            "id": "mongo_secondary_status",
            "name": "Mongo Secondary Status",
            "args": ["/etc/consul.d/check_scripts/mongo_secondary_check.sh"],
            "interval": "30s",
            "timeout": "20s"
        }
    }
}
 ```



Учитывая весь этот контекст, можете ли вы представить, как мы можем динамически переключать эти определения сервисов?

Мы можем определить, является ли данный [экземпляр MongoDB основным](https://docs.mongodb.com/manual/reference/command/isMaster) или нет, запустив команду `db.isMaster()` в MongoDB shell.

Можно оформить проверку в виде скрипта оболочки следующим образом:

 ```
#!/bin/bash

mongo_primary=$(mongo --quiet --eval 'JSON.stringify(db.isMaster())' | jq -r .ismaster 2> /dev/null)
if [[ $mongo_primary == false ]]; then
    exit 1
fi

echo "Mongo primary healthy and reachable"
 ```



Точно так же неосновные экземпляры MongoDB также могут быть проверены на соответствие той же команде, путем проверки `вторичного` значения:

 ```
#!/bin/bash

mongo_secondary=$(mongo --quiet --eval 'JSON.stringify(db.isMaster())' | jq -r .secondary 2> /dev/null)
if [[ $mongo_secondary == false ]]; then
    exit 1
fi

echo "Mongo secondary healthy and reachable"
 ```



Примечание. Мы используем `jq` - легкий и гибкий процессор JSON командной строки - для обработки выходных данных команд оболочки MongoDB в кодировке JSON.

Один из способов написания сценария, выполняющего этот динамический переключатель, выглядит так:

 ```
#!/bin/bash

# Wait until Mongo starts
while [[ $(ps aux | grep [m]ongod | wc -l) -ne 1 ]]; do
    sleep 5
done

REGISTER_MASTER=0
REGISTER_SECONDARY=0

mongo_primary=$(mongo --quiet --eval 'JSON.stringify(db.isMaster())' | jq -r .ismaster 2> /dev/null)
mongo_secondary=$(mongo --quiet --eval 'JSON.stringify(db.isMaster())' | jq -r .secondary 2> /dev/null)  

if [[ $mongo_primary == false && $mongo_secondary == true ]]; then

  # Deregister as Mongo Master
  if [[ -a /etc/consul.d/check_scripts/mongo_primary_check.sh && -a /etc/consul.d/mongo_primary.json ]]; then
    rm -f /etc/consul.d/check_scripts/mongo_primary_check.sh
    rm -f /etc/consul.d/mongo_primary.json

    REGISTER_MASTER=1
  fi

  # Register as Mongo Secondary
  if [[ ! -a /etc/consul.d/check_scripts/mongo_secondary_check.sh && ! -a /etc/consul.d/mongo_secondary.json ]]; then
    cp -u /opt/checks/check_scripts/mongo_secondary_check.sh /etc/consul.d/check_scripts/
    cp -u /opt/checks/mongo_secondary.json /etc/consul.d/

    REGISTER_SECONDARY=1
  fi

else

  # Register as Mongo Master
  if [[ ! -a /etc/consul.d/check_scripts/mongo_primary_check.sh && ! -a /etc/consul.d/mongo_primary.json ]]; then
    cp -u /opt/checks/check_scripts/mongo_primary_check.sh /etc/consul.d/check_scripts/
    cp -u /opt/checks/mongo_primary.json /etc/consul.d/

    REGISTER_MASTER=2
  fi

  # Deregister as Mongo Secondary
  if [[ -a /etc/consul.d/check_scripts/mongo_secondary_check.sh && -a /etc/consul.d/mongo_secondary.json ]]; then
    rm -f /etc/consul.d/check_scripts/mongo_secondary_check.sh
    rm -f /etc/consul.d/mongo_secondary.json

    REGISTER_SECONDARY=2
  fi

fi

if [[ $REGISTER_MASTER -ne 0 && $REGISTER_SECONDARY -ne 0 ]]; then
  consul reload
fi
 ```



Примечание. Это пример сценария. Но мы можем проявить более творческий подход и оптимизировать сценарий.

Когда мы закончим с определениями наших сервисов, мы можем запустить агент Consul на каждом узле MongoDB. Для запуска агента мы будем использовать следующую команду:

 ```
consul agent -bind 33.10.0.3 \
    -advertise 33.10.0.3 \
    -join consul_server \
    -node mongo_1 \
    -dns-port 53 \
    -data-dir /data \
    -config-dir /etc/consul.d \
    -enable-local-script-checks
 ```



Здесь `consul_server` представляет хост, на котором работает Consul Server. Точно так же мы можем запускать такие агенты на каждом из других узлов экземпляра MongoDB.

Примечание. Если у нас есть несколько экземпляров MongoDB, работающих на одном хосте, определение службы изменится, чтобы отразить разные порты, используемые каждым экземпляром для уникальной идентификации, обнаружения и мониторинга отдельного экземпляра MongoDB.

 

### Consul на узлах, на которых запущено приложение Django

Для приложения Django настройка Consul будет очень простой. Нам нужно только отслеживать порт приложения Django, на котором Gunicorn прослушивает запросы.

Определение службы Consul будет выглядеть так:

 ```
{
    "service": {
        "name": "web",
        "port": 8000,
        "tags": [
            "web",
            "application",
            "urlprefix-/web"
        ],
        "check": {
            "id": "web_app_status",
            "name": "Web Application Status",
            "tcp": "localhost:8000",
            "interval": "30s",
            "timeout": "20s"
        }
    }
}
 ```

Когда у нас есть определение службы Consul для приложения Django, мы можем запустить агента Consul, сидящего на узле, в котором приложение Django работает как служба. Чтобы запустить агента Consul, мы должны запустить следующую команду:

```
consul agent -bind 33.10.0.10 \
    -advertise 33.10.0.10 \
    -join consul_server \
    -node web_1 \
    -dns-port 53 \
    -data-dir /data \
    -config-dir /etc/consul.d \
    -enable-local-script-checks
```



### Consul Сервер

Мы запускаем кластер Consul с выделенным серверным узлом Consul. Узел сервера Consul может легко размещать, обнаруживать и отслеживать работающие на нем службы точно так же, как мы делали это в приведенных выше разделах для приложений MongoDB и Django.

Чтобы запустить Consul в режиме сервера и разрешить агентам подключаться к нему, мы запустим следующую команду на узле, на котором мы хотим запустить наш сервер Consul:

```
consul agent -server \
    -bind 33.10.0.2 \
    -advertise 33.10.0.2 \
    -node consul_server \
    -client 0.0.0.0 \
    -dns-port 53 \
    -data-dir /data \
    -ui -bootstrap
```

На данный момент на нашем узле сервера Consul нет служб, поэтому нет определений служб, связанных с этой конфигурацией агента Consul.

### Fabio

Мы используем возможности Fabio для [автоконфигурации](https://github.com/fabiolb/fabio) и поддержки Consul.

Это очень упрощает нашу задачу по балансировке нагрузки трафика для наших экземпляров приложения Django.

Чтобы позволить Fabio автоматически определять службы через Consul, одним из способов является добавление [тега или обновление тега в **определении**](https://github.com/fabiolb/fabio/wiki/Service-Configuration) сервиса с помощью префикса и идентификатора службы `urlprefix-/<service>`. Определение службы нашего Consul для приложения Django теперь будет выглядеть так:

```
{
    "service": {
        "name": "web",
        "port": 8000,
        "tags": [
            "web",
            "application",
            "urlprefix-/web"
        ],
        "check": {
            "id": "web_app_status",
            "name": "Web Application Status",
            "tcp": "localhost:8000",
            "interval": "30s",
            "timeout": "20s"
        }
    }
}
```



В нашем случае приложение или служба Django - единственная служба, которая нуждается в балансировке нагрузки, поэтому это изменение определения службы Consul выполняет требования по настройке Fabio.

 

### Докеризация

Все наше приложение будет развернуто как набор контейнеров Docker. Давайте поговорим о том, как мы этого достигаем в контексте Consul.

 

### Докеризация набора реплик MongoDB вместе с агентом Consul

Нам нужно запустить агент Consul, как описано выше, вместе с MongoDB в том же контейнере Docker, поэтому нам нужно будет запустить настраиваемый ENTRYPOINT в контейнере, чтобы разрешить запуск двух процессов.

Примечание. Этого также можно добиться с помощью проверок уровня контейнера Docker в Consul. Таким образом, вы сможете запустить агент Consul на хосте и проверить сервис, работающий в контейнере Docker. Что, по сути, будет запускаться в контейнер для мониторинга службы.

Для этого мы будем использовать инструмент, похожий на [Foreman](https://www.theforeman.org/). Это инструмент управления жизненным циклом физических и виртуальных серверов, включая выделение ресурсов, мониторинг и настройку.

Чтобы быть точным, мы будем использовать Golang, заимствованный у Foreman, [Goreman](https://github.com/mattn/goreman). Он принимает конфигурацию в форме [Procfile Heroku](https://devcenter.heroku.com/articles/procfile), чтобы понимать, какие процессы должны оставаться активными на хосте.

В нашем случае Procfile выглядит так:

```
# Mongo
mongo: /opt/mongo.sh

# Consul Client Agent
consul: /opt/consul.sh

# Consul Client Health Checks
consul_check: while true; do /opt/checks_toggler.sh && sleep 10; done
```



`consul_check` в конце профиля поддерживает динамизм между проверками как первичного, так и вторичного узла MongoDB в зависимости от того, за какую роль в наборе реплик MongoDB проголосовали.

Сценарии оболочки, которые выполняются соответствующими клавишами в файле Procfile, определены ранее в этом обсуждении.

Наш [Dockerfile](https://docs.docker.com/engine/reference/builder/) с некоторыми дополнительными инструментами для отладки и диагностики будет выглядеть так:

```
FROM ubuntu:18.04

RUN apt-get update && \
    apt-get install -y \
    bash curl nano net-tools zip unzip \
    jq dnsutils iputils-ping

# Install MongoDB
RUN apt-get install -y mongodb

RUN mkdir -p /data/db
VOLUME data:/data/db

# Setup Consul and Goreman
ADD https://releases.hashicorp.com/consul/1.4.4/consul_1.4.4_linux_amd64.zip /tmp/consul.zip
RUN cd /bin && unzip /tmp/consul.zip && chmod +x /bin/consul && rm /tmp/consul.zip

ADD https://github.com/mattn/goreman/releases/download/v0.0.10/goreman_linux_amd64.zip /tmp/goreman.zip
RUN cd /bin && unzip /tmp/goreman.zip && chmod +x /bin/goreman && rm /tmp/goreman.zip

RUN mkdir -p /etc/consul.d/check_scripts
ADD ./config/mongod /etc

RUN mkdir -p /etc/checks
ADD ./config/checks /opt/checks

ADD checks_toggler.sh /opt
ADD mongo.sh /opt
ADD consul.sh /opt

ADD Procfile /root/Procfile

EXPOSE 27017

# Launch both MongoDB server and Consul
ENTRYPOINT [ "goreman" ]
CMD [ "-f", "/root/Procfile", "start" ]
```



*Примечание. Мы использовали здесь чистый образ Ubuntu 18.04 для наших целей, но вы можете использовать [официальный образ MongoDB](https://hub.docker.com/_/mongo) и адаптировать его для запуска Consul вместе с MongoDB или даже выполнять проверки Consul на уровне контейнера Docker, как указано в официальной документации.*

 

### Докеризация веб-приложения Django вместе с Consul Agent

Нам также необходимо запустить агент Consul вместе с нашим приложением Django в том же контейнере Docker, что и у нас с контейнером MongoDB.

```
# Django
django: /web/tweeter.sh

# Consul Client Agent
consul: /opt/consul.sh
```

Точно так же у нас будет Dockerfile для веб-приложения Django, как и для наших контейнеров MongoDB.

```
FROM python:3.7

RUN apt-get update && \
    apt-get install -y \
    bash curl nano net-tools zip unzip \
    jq dnsutils iputils-ping

# Python Environment Setup
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Setup Consul and Goreman
RUN mkdir -p /data/db /etc/consul.d

ADD https://releases.hashicorp.com/consul/1.4.4/consul_1.4.4_linux_amd64.zip /tmp/consul.zip
RUN cd /bin && unzip /tmp/consul.zip && chmod +x /bin/consul && rm /tmp/consul.zip

ADD https://github.com/mattn/goreman/releases/download/v0.0.10/goreman_linux_amd64.zip /tmp/goreman.zip
RUN cd /bin && unzip /tmp/goreman.zip && chmod +x /bin/goreman && rm /tmp/goreman.zip

ADD ./consul /etc/consul.d
ADD Procfile /root/Procfile

# Install pipenv
RUN pip3 install --upgrade pip
RUN pip3 install pipenv

# Setting workdir
ADD consul.sh /opt
ADD . /web
WORKDIR /web/tweeter

# Exposing appropriate ports
EXPOSE 8000/tcp

# Install dependencies
RUN pipenv install --system --deploy --ignore-pipfile

# Migrates the database, uploads staticfiles, run API server and background tasks
ENTRYPOINT [ "goreman" ]
CMD [ "-f", "/root/Procfile", "start" ]
```



#### Докеризация Consul Server

Мы поддерживаем тот же поток с серверным узлом Consul, чтобы запустить его с пользовательским ENTRYPOINT. Это не является обязательным требованием, но мы поддерживаем единообразное представление различных файлов запуска Consul.

Кроме того, для демонстрации мы используем образ Ubuntu 18.04. Для этого вы можете использовать [официальный образ Consul](https://hub.docker.com/_/consul), которое принимает все пользовательские параметры, указанные здесь.

```
FROM ubuntu:18.04

RUN apt-get update && \
    apt-get install -y \
    bash curl nano net-tools zip unzip \
    jq dnsutils iputils-ping

ADD https://releases.hashicorp.com/consul/1.4.4/consul_1.4.4_linux_amd64.zip /tmp/consul.zip
RUN cd /bin && unzip /tmp/consul.zip && chmod +x /bin/consul && rm /tmp/consul.zip

# Consul ports
EXPOSE 8300 8301 8302 8400 8500

ADD consul_server.sh /opt
RUN mkdir -p /data
VOLUME /data

CMD ["/opt/consul_server.sh"]
```



### Docker Compose

Мы используем [Compose](https://github.com/docker/compose) для запуска всех наших контейнеров Docker в желаемой, повторяемой форме.

Наш [файл Compose](https://docs.docker.com/compose/compose-file/) написан для обозначения всех аспектов, о которых мы упомянули выше, и использует возможности инструмента Docker Compose для их беспрепятственного выполнения.

Файл Docker Compose будет выглядеть так, как показано ниже:

```
version: "3.6"

services:

  consul_server:
    build:
      context: consul_server
      dockerfile: Dockerfile
    image: consul_server
    ports:
      - 8300:8300
      - 8301:8301
      - 8302:8302
      - 8400:8400
      - 8500:8500
    environment:
      - NODE=consul_server
      - PRIVATE_IP_ADDRESS=33.10.0.2
    networks:
      consul_network:
        ipv4_address: 33.10.0.2

  load_balancer:
    image: fabiolb/fabio
    ports:
      - 9998:9998
      - 9999:9999
    command: -registry.consul.addr="33.10.0.2:8500"
    networks:
      consul_network:
        ipv4_address: 33.10.0.100

  mongo_1:
    build:
      context: mongo
      dockerfile: Dockerfile
    image: mongo_consul
    dns:
      - 127.0.0.1
      - 8.8.8.8
      - 8.8.4.4
    environment:
      - NODE=mongo_1
      - MONGO_PORT=27017
      - PRIMARY_MONGO=33.10.0.3
      - PRIVATE_IP_ADDRESS=33.10.0.3
    restart: always
    ports:
      - 27017:27017
      - 28017:28017
    depends_on:
      - consul_server
      - mongo_2
      - mongo_3
    networks:
      consul_network:
        ipv4_address: 33.10.0.3

  mongo_2:
    build:
      context: mongo
      dockerfile: Dockerfile
    image: mongo_consul
    dns:
      - 127.0.0.1
      - 8.8.8.8
      - 8.8.4.4
    environment:
      - NODE=mongo_2
      - MONGO_PORT=27017
      - PRIMARY_MONGO=33.10.0.3
      - PRIVATE_IP_ADDRESS=33.10.0.4
    restart: always
    ports:
      - 27018:27017
      - 28018:28017
    depends_on:
      - consul_server
    networks:
      consul_network:
        ipv4_address: 33.10.0.4

  mongo_3:
    build:
      context: mongo
      dockerfile: Dockerfile
    image: mongo_consul
    dns:
      - 127.0.0.1
      - 8.8.8.8
      - 8.8.4.4
    environment:
      - NODE=mongo_3
      - MONGO_PORT=27017
      - PRIMARY_MONGO=33.10.0.3
      - PRIVATE_IP_ADDRESS=33.10.0.5
    restart: always
    ports:
      - 27019:27017
      - 28019:28017
    depends_on:
      - consul_server
    networks:
      consul_network:
        ipv4_address: 33.10.0.5

  web_1:
    build:
      context: django
      dockerfile: Dockerfile
    image: web_consul
    ports:
      - 8080:8000
    environment:
      - NODE=web_1
      - PRIMARY=1
      - LOAD_BALANCER=33.10.0.100
      - PRIVATE_IP_ADDRESS=33.10.0.10
    dns:
      - 127.0.0.1
      - 8.8.8.8
      - 8.8.4.4
    depends_on:
      - consul_server
      - mongo_1
    volumes:
      - ./django:/web
    cap_add:
      - NET_ADMIN
    networks:
      consul_network:
        ipv4_address: 33.10.0.10

  web_2:
    build:
      context: django
      dockerfile: Dockerfile
    image: web_consul
    ports:
      - 8081:8000
    environment:
      - NODE=web_2
      - LOAD_BALANCER=33.10.0.100
      - PRIVATE_IP_ADDRESS=33.10.0.11
    dns:
      - 127.0.0.1
      - 8.8.8.8
      - 8.8.4.4
    depends_on:
      - consul_server
      - mongo_1
    volumes:
      - ./django:/web
    cap_add:
      - NET_ADMIN
    networks:
      consul_network:
        ipv4_address: 33.10.0.11

networks:
  consul_network:
    driver: bridge
    ipam:
     config:
       - subnet: 33.10.0.0/16
```



На этом мы подошли к концу настройки всей среды. Теперь мы можем запустить Docker Compose для сборки и запуска контейнеров.

 

### Обнаружение сервисов с использованием Consul

Когда все службы запущены и работают, пользовательский интерфейс Consul Web дает нам хороший обзор нашей общей настройки.

![](https://habrastorage.org/webt/72/vj/_k/72vj_k_n7thle30ldz6pn44rrck.png)

*Сервис MongoDB доступен для приложения Django для обнаружения с помощью интерфейса Consul DNS.*

```
root@82857c424b15:/web/tweeter# dig @127.0.0.1 mongo-primary.service.consul

; <<>> DiG 9.10.3-P4-Debian <<>> @127.0.0.1 mongo-primary.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 8369
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;mongo-primary.service.consul.	IN	A

;; ANSWER SECTION:
mongo-primary.service.consul. 0	IN	A	33.10.0.3

;; ADDITIONAL SECTION:
mongo-primary.service.consul. 0	IN	TXT	"consul-network-segment="

;; Query time: 139 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Mon Apr 01 11:50:45 UTC 2019
;; MSG SIZE  rcvd: 109
```



Приложение Django теперь может подключить первичный экземпляр MongoDB и начать записывать в него данные.

Мы можем использовать [балансировщик](https://velotio.com/blog/2017/7/5/http-load-balancing-in-kubernetes-with-ingress) нагрузки Fabio для подключения к экземпляру приложения Django, автоматически обнаруживая его через реестр Consul с использованием специализированных сервисных тегов и отображая страницу со всей информацией о подключении к базе данных, о которой мы говорим.

Наш балансировщик нагрузки находится на `33.10.0.100`, а `/web` настроен для перенаправления на один из наших экземпляров приложения Django, работающий за балансировщиком нагрузки.

 ![](https://habrastorage.org/webt/tt/59/oi/tt59oi-jm7kmm40vdqvaravdzpo.png)

*Fabio автоматически определяет конечные точки веб-приложения Django*

Как вы можете видеть из автоматического определения и настройки балансировщика нагрузки Fabio из его пользовательского интерфейса выше, он одинаково взвешивал конечные точки веб-приложения Django. Это поможет сбалансировать нагрузку запроса или трафика на экземпляры приложения Django.

Когда мы посещаем наш URL-адрес Fabio `33.10.0.100:9999` и используем исходный маршрут как `/web`, мы перенаправляемся на один из экземпляров Django. Итак, посещение `33.10.0.100:9999/web` дает нам следующий результат.

![](https://habrastorage.org/webt/dj/gn/o8/djgno8tjtf-5asfkbdzgf6zvglk.png)

*Веб-приложение Django отображает статус подключения MongoDB на домашней странице*

Мы можем ограничить Fabio только экземплярами приложения Django с балансировкой нагрузки, добавив только необходимые теги в определения служб приложений Django в Consul.

Это обнаружение первичного экземпляра MongoDB помогает приложению Django выполнять миграцию базы данных и развертывание приложения.

Можно изучить веб-интерфейс Consul, чтобы увидеть все экземпляры служб веб-приложений Django.

![](https://habrastorage.org/webt/kt/zs/lw/ktzslw35zj7uq56j-ksal3si8w4.png)

*Сервисы веб-приложений Django в веб-интерфейсе Consul*

Точно так же посмотрите, как устроены экземпляры набора реплик MongoDB.

![](https://habrastorage.org/webt/xg/5j/uo/xg5juovp_7ahtrynrpqvpkljtfq.png)

*MongoDB Replica Set Primary service видимый в веб-интерфейсе Consul*

![](https://habrastorage.org/webt/z2/xb/ut/z2xbut_xm9mkhpzv_ne9emanu5m.png)

*MongoDB Replica Set Secondary service видимый в веб-интерфейсе Consul*

Давайте посмотрим, как Consul помогает с услугами проверки работоспособности и обнаружением только действующих служб.

Мы остановим текущий контейнер MongoDB Replica Set Primary (`mongo_2`), чтобы посмотреть, что произойдет.

 ![](https://habrastorage.org/webt/77/bz/2z/77bz2z7cr-3vbq_orajymfujn7m.png)

*Первичная служба MongoDB заменяется одним из вторичных экземпляров MongoDB*

![](https://habrastorage.org/webt/cd/5g/i6/cd5gi6bnn7gnunoien3gym6zrfw.png)

*Во вторичном экземпляре MongoDB теперь остался только один экземпляр службы*

Consul перестал проходить проверку работоспособности для предыдущей основной службы MongoDB. Набор реплик MongoDB также обнаружил, что узел не работает и необходимо переизбрать основной узел. Таким образом, мы автоматически получаем новый MongoDB Primary (`mongo_3`).

Наш переключатель проверок сработал и заменил проверку `mongo_3` со вторичной проверки MongoDB на первичную проверку MongoDB.

Когда мы смотрим на представление из приложения Django, мы видим, что теперь оно подключено к новой первичной службе MongoDB (`mongo_3`).

 ![](https://habrastorage.org/webt/hx/ge/oz/hxgeozzkdi2ewdktqbigjqjmwmq.png)

*Отключение MongoDB Primary также отражается в веб-приложении Django.*

Давайте посмотрим, как это разовьется, когда мы вернем остановленный экземпляр MongoDB.

 ![](https://habrastorage.org/webt/_q/km/nl/_qkmnlrjyqua8oi_9emofpdx2aw.png)

*Неисправный экземпляр первичной службы MongoDB теперь удален из экземпляров службы, так как теперь он является исправным экземпляром вторичной службы MongoDB.*

![](https://habrastorage.org/webt/hy/le/if/hyleifzakuglm745rz_pybppwwc.png)

*Ранее отказавший экземпляр первичной службы MongoDB теперь повторно принят в качестве вторичного экземпляра службы MongoDB, поскольку он снова стал работоспособным*

Точно так же, если мы остановим экземпляры службы приложения Django, Fabio теперь сможет обнаруживать только работоспособный экземпляр и направлять трафик только на этот экземпляр.

 ![](https://habrastorage.org/webt/uz/ya/s_/uzyas_24bjt97ypblawnstk02ys.png)

*Fabio может автоматически настраиваться, используя реестр служб Consul и обнаруживая действующие экземпляры служб.*

Вот как можно использовать возможность обнаружения служб Consul для обнаружения, мониторинга и проверки состояния служб.

 

### Настройка сервиса с помощью Consul

В настоящее время мы настраиваем экземпляры приложения Django напрямую либо из [переменных среды, установленных в контейнерах с помощью Docker Compose](https://docs.djangoproject.com/en/2.1/topics/settings/), и потребляем их в настройках проекта Django, либо [путем жесткого кодирования параметров конфигурации](https://docs.djangoproject.com/en/2.1/ref/settings/) напрямую.

Мы можем использовать хранилище [ключей / значений Consul](https://www.consul.io/api/kv.html) для обмена конфигурацией между обоими экземплярами приложения Django.

Мы можем использовать [HTTP-интерфейс Consul](https://www.consul.io/api/index.html) для хранения пары ключ / значение и получения их в приложении, используя клиент Python с открытым исходным кодом для Consul, называемый [python-consul](https://python-consul.readthedocs.io/en/latest/). Вы также можете использовать любую другую библиотеку Python, которая может взаимодействовать с Consul KV, если хотите.

Давайте начнем с того, что посмотрим, как мы можем установить пару ключ / значение в Consul, используя его HTTP-интерфейс.

 ```bash
# Flag to run Django app in debug mode
curl -X PUT -d 'True' consul_server:8500/v1/kv/web/debug

# Dynamic entries into Django app configuration 
# to denote allowed set of hosts
curl -X PUT -d 'localhost, 33.10.0.100' consul_server:8500/v1/kv/web/allowed_hosts

# Dynamic entries into Django app configuration
# to denote installed apps
curl -X PUT -d 'tweetapp' consul_server:8500/v1/kv/web/installed_apps
 ```

После того, как мы установили хранилище KV, мы можем использовать его в экземплярах приложения Django, чтобы настроить его с этими значениями.

Давайте установим [python-consul](https://pypi.org/project/python-consul/) и добавим его как зависимость проекта.

 ```bash
$ pipenv shell
Launching subshell in virtual environment…
 . /home/pranav/.local/share/virtualenvs/tweeter-PYSn2zRU/bin/activate

$  . /home/pranav/.local/share/virtualenvs/tweeter-PYSn2zRU/bin/activate

(tweeter) $ pipenv install python-consul
Installing python-consul…
Adding python-consul to Pipfile's [packages]…
✔ Installation Succeeded 
Locking [dev-packages] dependencies…
Locking [packages] dependencies…
✔ Success! 
Updated Pipfile.lock (9590cc)!
Installing dependencies from Pipfile.lock (9590cc)…
  🐍   ▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉ 14/14 — 00:00:20
 ```

Нам нужно будет подключить наше приложение к Consul с помощью python-consul.

 ```bash
import consul

consul_client = consul.Consul(
    host='consul_server',
    port=8500,
)
 ```

Мы можем захватить и настроить наше приложение Django соответствующим образом с помощью библиотеки `python-consul`.

 ```bash
# Set DEBUG flag using Consul KV store
index, data = consul_client.kv.get('web/debug')
DEBUG = data.get('Value', True)

# Set ALLOWED_HOSTS dynamically using Consul KV store
ALLOWED_HOSTS = []

index, hosts = consul_client.kv.get('web/allowed_hosts')
ALLOWED_HOSTS.append(hosts.get('Value'))

# Set INSTALLED_APPS dynamically using Consul KV store
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]

index, apps = consul_client.kv.get('web/installed_apps')
INSTALLED_APPS += (bytes(apps.get('Value')).decode('utf-8'),)
 ```

Эти пары ключ / значение из магазина Consul KV также можно просматривать и обновлять через его веб-интерфейс.

 ![](https://habrastorage.org/webt/83/oy/38/83oy38ag475voobbycc5mos7jhk.png)

*Consul KV store в интерфейсе Consul Web с параметрами конфигурации приложения Django*
 

Код, используемый как часть этого руководства для раздела конфигурации службы Consul, доступен в ветви «service-configuration» проекта pranavcode / consul-demo.

Вот как можно использовать Consul KV и легко настраивать отдельные службы в их архитектуре.

Сегментация услуг с использованием Consul

В рамках сегментации услуг Consul мы рассмотрим намерения Consul Connect и распределение центров обработки данных.

Connect обеспечивает авторизацию и шифрование соединений между сервисами с использованием взаимного TLS.

Чтобы использовать Consul, вам необходимо включить его в конфигурации сервера. Для правильного функционирования кластера необходимо включить подключение в кластере Consul.

В нашем контексте мы можем определить, что связь должна быть идентифицирована и защищена TLS, мы определим восходящую сопутствующую службу с прокси в приложении Django для ее связи с первичным экземпляром MongoDB.
 

Наряду с настройкой Connect для бокового прокси нам также потребуется запустить прокси-сервер Connect для приложения Django. Этого можно добиться, выполнив следующую команду.

Мы можем добавить Consul Connect Intentions для создания графа сервисов для всех сервисов и определения шаблонов трафика. Мы можем создавать намерения, как показано ниже:


Намерениями для графа услуг также можно управлять из веб-интерфейса Consul.
 

Это определяет ограничения на подключение к службе, чтобы разрешить или запретить им общаться через Connect.

Мы также добавили возможность для агентов Consul указывать, к каким центрам обработки данных они принадлежат, и быть доступными через один или несколько серверов Consul в данном центре обработки данных.

Код, используемый как часть этого руководства для раздела сегментации услуг Consul, доступен в ветви «service-segmentation» проекта velotiotech / consul-demo.

Вот как можно использовать функцию сегментации услуг Consul и настроить контроль доступа к подключению на уровне обслуживания.


Вывод

Возможность беспрепятственно контролировать сервисную сеть, которую предоставляет Consul, очень упрощает жизнь оператора. Мы надеемся, что вы узнали, как Consul можно использовать для обнаружения, настройки и сегментации сервисов с его практической реализацией.

Как обычно, мы надеемся, что изучение Консула было познавательным. Это была последняя часть этой серии из двух частей. В этой части делается попытка охватить большинство аспектов архитектуры Consul и то, как она вписывается в ваш текущий проект. Если вы пропустили первую часть, вы сможете найти ее здесь.

Мы продолжим изучать различные технологии и предоставим вам наиболее ценную информацию. Сообщите нам, что вы хотели бы услышать от нас в следующий раз, или, если у вас есть какие-либо вопросы по этой теме, мы будем очень рады на них ответить.


![](https://habrastorage.org/webt/cm/hm/ax/cmhmaxjf6psd0zbbe7kmajcvcwm.png)

