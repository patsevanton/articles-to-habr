В этом посте рассмотрим хранилище rpm артефактов c помощью простого скрипта с inotify + createrepo. Заливка артефактов осуществляется через webdav используя apache httpd. Почему apache httpd будет написано ближе к концу поста.

### Итак, решение должно отвечать cледующим требованиям для организации только RPM хранилища:

- Бесплатное
- Доступность пакета в репозитории через несколько секунд после загрузки в хранилище артефактов.
- Простое в установке и обслуживании
- Возможность сделать высокую доступность (high availability)

### Почему не [SonaType Nexus](https://habr.com/ru/post/473358/) или [Pulp](https://pulpproject.org/):

- Хранение в [SonaType Nexus](https://habr.com/ru/post/473358/) или [Pulp](https://pulpproject.org/) многих типов артефактов приводит к тому что [SonaType Nexus](https://habr.com/ru/post/473358/) или [Pulp](https://pulpproject.org/) становятся единой точкой отказа.
- Высокая доступность (high availability) в [SonaType Nexus](https://habr.com/ru/post/473358/) является платной.
- [Pulp](https://pulpproject.org/) мне кажетя переусложенным решением.
- Артефакты в [SonaType Nexus](https://habr.com/ru/post/473358/) хранятся в blob. При внезапном выключении электричества вы не сможете восстановить blob, если у в вас нет бекапа. У нас была такая ошибка: ``` ERROR [ForkJoinPool.commonPool-worker-2] *SYSTEM [com.orientechnologies.orient.core.storage](http://com.orientechnologies.orient.core.storage/).fs.OFileClassic - $ANSI{green {db=security}} Error during data read for file 'privilege_5.pcl' 1-th attempt [java.io](http://java.io/).IOException: Bad address```. Blob так и не восстановили.

### Исходный код

Исходный код находится тут https://github.com/patsevanton/inotify-createrepo

Основной скрипт выглядит так:

```bash
#!/bin/bash

source /etc/inotify-createrepo.conf
LOGFILE=/var/log/inotify-createrepo.log

function monitoring() {
    inotifywait -e create,delete -msrq --exclude ".repodata|.olddata|repodata" "${REPO}" | while read events 
    do
      echo $events >> $LOGFILE
      touch /tmp/need_create
    done
}

function run_createrepo() {
  while true; do
    if [ -f /tmp/need_create ];
    then
      rm -f /tmp/need_create
      echo "start createrepo $(date --rfc-3339=seconds)"
      /usr/bin/createrepo --update "${REPO}"
      echo "finish createrepo $(date --rfc-3339=seconds)"
    fi
    sleep 1
  done
}


echo "Start filesystem monitoring: Directory is $REPO, monitor logfile is $LOGFILE"
monitoring >> $LOGFILE &
run_createrepo >> $LOGFILE &
```

### Установка

Inotify-createrepo работает только на CentOS 7 или выше. На CentOS 6 не удалось его заставить работать.

```bash
yum -y install yum-plugin-copr
yum copr enable antonpatsev/inotify-createrepo
yum -y install inotify-createrepo
systemctl start inotify-createrepo
```

### Конфигурирование

По умолчанию inotify-createrepo мониторит директорию `/var/www/repos/rpm-repo/`.

Изменить эту директорию можно в файле `/etc/inotify-createrepo.conf.`

### Использование

При добавлении любого файла в директорию `/var/www/repos/rpm-repo/` inotifywait создаст файл `/tmp/need_create`.  Функция run_createrepo запускается в бесконечном цикле и мониторит файл `/tmp/need_create`. Если файл существует, то запускается `createrepo --update`.

В файле появится запись:

```
/var/www/repos/rpm-repo/ CREATE nginx-1.16.1-1.el7.ngx.x86_64.rpm
start createrepo 2020-03-02 09:46:21+03:00
Spawning worker 0 with 1 pkgs
Spawning worker 1 with 0 pkgs
Spawning worker 2 with 0 pkgs
Spawning worker 3 with 0 pkgs
Workers Finished
Saving Primary metadata
Saving file lists metadata
Saving other metadata
Generating sqlite DBs
Sqlite DBs complete
finish createrepo 2020-03-02 09:46:22+03:00
```



### Возможность сделать высокую доступность (high availability)

Чтобы сделать высокую доступность (high availability) из существующего рещения, думаю можно использовать 2 сервера, Keepalived для HA и Lsyncd для синхронизации артефактов. [Lsyncd](http://code.google.com/p/lsyncd) — демон, который следит за изменениями в локальной директории, агрегирует их, и по прошествии определенного времени стартует rsync для их синхронизации. Подробности и настройка описана в посте "[Cкоростная синхронизация миллиарда файлов](https://habr.com/ru/post/132098/)".

### WebDav

Загружать файлы можно несколькими путями: SSH, NFS, WebDav. WebDav кажется современным и простым вариантом.

Для WebDav будем использовать Apache httpd. Почему Apache httpd в 2020 году, а не nginx?

Хочется использовать автоматизированные средства для сборки Nginx + модули (например, Webdav).

Есть проект по сборке Nginx + модули - [Nginx-builder](https://github.com/TinkoffCreditSystems/Nginx-builder). Если использовать nginx + wevdav для загрузки файлов, то нужен модуль [nginx-dav-ext-module](https://github.com/arut/nginx-dav-ext-module). При попытке собрать и использовать Nginx с [nginx-dav-ext-module](https://github.com/arut/nginx-dav-ext-module) при помощи [Nginx-builder](https://github.com/TinkoffCreditSystems/Nginx-builder) мы получим ошибку [Used by http_dav_module instead of nginx-dav-ext-module](https://github.com/TinkoffCreditSystems/Nginx-builder/issues/27). Такая же ошибка была закрыта летом [nginx: [emerg] unknown directive dav_methods](https://github.com/TinkoffCreditSystems/Nginx-builder/issues/12). 

Я делал Pull request [Add check git_url for embedded, refactored --with-{}_module](https://github.com/TinkoffCreditSystems/Nginx-builder/pull/18) и [if module == "http_dav_module" append --with](https://github.com/TinkoffCreditSystems/Nginx-builder/pull/14). Но их не приняли.

#### Конфиг webdav.conf

```
DavLockDB /var/www/html/DavLock
<VirtualHost localhost:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html
    ErrorLog /var/log/httpd/error.log
    CustomLog /var/log/httpd/access.log combined

    Alias /rpm /var/www/repos/rpm-repo
    <Directory /var/www/repos/rpm-repo>
        DAV On
        Options Indexes FollowSymlinks SymLinksifOwnerMatch IncludesNOEXEC
        IndexOptions NameWidth=* DescriptionWidth=*
        AllowOverride none
        Require all granted
    </Directory>
</VirtualHost>
```

Остальную настройку Apache httpd я думаю вы сделаете сами.

### Nginx перед Apache httpd

В отличие от Apache, Nginx использует событийную модель обработки запросов, благодаря чему на любое количество клиентов требуется всего один процесс HTTP-сервера. Вы можете использовать nginx и снизить нагрузку на сервер.

Конфиг nginx-front.conf. Остальную настройку nginx я думаю вы сделаете сами.

```nginx
upstream nginx_front {
    server localhost:80;
}

server {
    listen 443 ssl;
    server_name ваш-виртуальных-хост;
    access_log /var/log/nginx/nginx-front-access.log main;
    error_log /var/log/nginx/nginx-front.conf-error.log warn;

    location / {
        proxy_pass http://nginx_front;
    }
}
```

### Загрузка файлов через WebDav

Загрузка rpm очень простая.

```bash
curl -T ./nginx-1.16.1-1.el7.ngx.x86_64.rpm https://ваш-виртуальный-хост/rpm/
```
