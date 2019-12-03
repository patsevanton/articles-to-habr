Предисловие из [поста](https://habr.com/ru/post/463525/):

[Selenoid](https://github.com/aerokube/selenoid) — это программа, которая позволяет управлять браузерами и [Android-эмуляторами](https://hub.docker.com/r/selenoid/android) с помощью специальных драйверов. Умеет запускать каждый из них изолированно в Docker-контейнере.

[Selenoid](https://github.com/aerokube/selenoid) представляет собой альтернативное решение Selenium Server, хотя суть та же — организация работы драйверов.

Основная идея Selenoid состоит в том, чтобы запускать новый контейнер для каждой сессии (запуска нового браузера или эмулятора) и останавливать его сразу же после закрытия сессии.

[Selenoid](https://github.com/aerokube/selenoid) позволяет поддерживать высокую нагрузку без дополнительных ресурсозатрат.

В этом посте будет запуск простых тестов в [Android-эмуляторе](https://hub.docker.com/r/selenoid/android).

### Подготовка

Предварительно проверьте что ваша система может запускать виртуальные машины.

Аппаратная виртуализация должна поддерживаться вашим процессором. Это означает, что требуют­ся расширения процессора Intel­VT или AMD­V. Чтобы убедиться, под держивает ли процессор одно из них, выполните команду:

```bash
egrep '(vmx|svm)' /proc/cpuinfo
```

### Docker

На вашей операционной системе обязательно должен быть установлен и запущен Docker.

### Установка Selenoid

Если у вас Redhat-based операционная система, вы можете использовать мой репозиторий для установки [Configuration manager](https://github.com/aerokube/cm).

```bash
yum -y install yum-plugin-copr
yum copr enable antonpatsev/aerokube-cm-rpm
yum -y install aerokube-cm
```

Если у вас не Redhat-based операционная система, то вы можете [скачать](https://github.com/aerokube/cm/releases) и использовать бинарник [Configuration manager](https://github.com/aerokube/cm).

### Запуск Selenoid используя Configuration manager и формирование browsers.json

Если у вас нет прямого доступа в инет и docker образы вы скачиваете через registry:

```bash
aerokube-cm selenoid start --force --browsers "android:6.0;chrome:78" --args "-session-attempt-timeout 2m -service-startup-timeout 2m" --registry ваш-docker-registry
```

Если у вас есть прямой доступ в инет.

```bash
aerokube-cm selenoid start --force --browsers "android:6.0;chrome:78" --args "-session-attempt-timeout 2m -service-startup-timeout 2m"
```

Ключ `--args "-session-attempt-timeout 2m -service-startup-timeout 2m"` нужен если у вас apk большого размера долго устанавливается.

Ключ `--force` перезаписывает файл browsers.json

Так как Selenoid Configuration manager пока что не умеет формировать browsers.json для мобильного Chrome, то его нужно поправить самостоятельно.

По умолчанию browsers.json формируется в директории ~/.aerokube/selenoid.

Итоговый файл browsers.json для тестирования Android приложений и Chrome внутри Android эмулятора.

```json
{
    "android": {
        "default": "6.0",
        "versions": {
            "6.0": {
                "image": "docker-registry:443/selenoid/android:6.0",
                "port": "4444",
                "path": "/wd/hub"
            }
        }
    },
    "chrome": {
        "default": "mobile-75.0",
        "versions": {
            "mobile-75.0": {
                "image": "docker-registry:443/selenoid/chrome-mobile:75.0",
                "port": "4444",
                "path": "/wd/hub"
            }
        }
    }
}
```

Пока что версия мобильного хрома отстает от версии обычного хрома.
Скачиваем образ мобильного хрома
```bash
docker pull selenoid/chrome-mobile:75.0
```
Проверяем что docker контейнер запустился и образы скачались.

```bash
docker ps
docker images
```

![](https://habrastorage.org/webt/6u/xe/4s/6uxe4sj8mr1whgda1ayt6pceoos.png)

### Запуск Selenoid UI используя Configuration manager

```bash
aerokube-cm selenoid-ui start --registry https://docker-registry
```

Или

```bash
aerokube-cm selenoid-ui start
```

Проверяем что docker контейнер запустился и образы скачались.

```bash
docker ps
docker images
```

![](https://habrastorage.org/webt/-x/pd/cw/-xpdcwppkyiael9ly2agw0opgv4.png)

Заходим в selenoid-ui по адресу ip-где-вы-запускали-selenoid-и-selenoid-ui:8080

У вас должно быть гореть зеленым 2 слова CONNECTED и написано android и chrome.

![](https://habrastorage.org/webt/no/ic/l8/noicl8rlt7_9tjmaihi2w3l1wia.png)

### DEMO TEST

Скачиваем https://github.com/aerokube/demo-tests

Во всех трех java файлах меняем путь в RemoteWebDriver на localhost

![](https://habrastorage.org/webt/i-/6v/3_/i-6v3_qo-z7gaxzdx5t_nsdxyrm.png)

или на другой адрес, там где вы запустили selenoid.

В файле AndroidRemoteApkTest.java меняем путь где можно скачать вашу APK.

```bash
device.setCapability("app", "http://ci.example.com/game2048.apk");
```

на

device.setCapability("app", "http://ссылка-до-вашей-apk");

или 

```bash
device.setCapability("app", "http://hostname-или-FQDN-сервера-где-лежит-apk:8000/game2048.apk");
```

Если вы будете ссылаться на localhost из docker, то у вас будет вот такая ошибка, так как вы из сети docker пытаетесь обраться к localhost основного сервера:

```bash
Tests in error: 
  browserTest(com.aerokube.selenoid.AndroidRemoteApkTest): An unknown server-side error occurred while processing the command. Original error: Problem downloading app from url http://localhost:8000/apk/game2048.apk: connect ECONNREFUSED 127.0.0.1:8000
```

Как сделать доступной для скачивания ваши локальные файлы будет ниже.

В файле DemoTest.java добавляем setCapability для запуска chrome на Android чтобы получилось примерно так.

![](https://habrastorage.org/webt/nb/at/4d/nbat4dgrfnrsjz-u-inkwdo_r8q.png)

В каждом файле java вы можете включить или выключить запись видео, удаленный просмотр или управление через VNC и запись логов в файл. Чтобы выключить опцию нужно добавить 2 слеша в начало строки.

![](https://habrastorage.org/webt/bl/dy/jd/bldyjdaxstceqigh7szqsb5qvlo.png)

Чтобы сделать доступной для скачивания файлы из текущей директории, нужно запустить в отдельной консоле в этой директории команду:

```bash
ruby -rwebrick -e'WEBrick::HTTPServer.new(:Port => 8000, :DocumentRoot => Dir.pwd).start'
```

### Запуск тестов

В директории demo-tests запускаем тесты:

Если вам нужно указать настройки и у вас используется maven-прокси (Nexus, Artifactory)

```bash
mvn -s settings.xml clean test
```

Если запускаем с прямым доступом в инет и без каких-либо настроек

```bash
mvn clean test
```

### Скорость

Общее время разворачивания android эмулятора и запуск 1 теста занимает меньше 1 минуты.

### Изменение browsers.json

При изменении файла browsers.json нужно перезагрузить selenoid

```bash
aerokube-cm selenoid stop
```

```bash
aerokube-cm selenoid start
```
Reloading configuration
Можно сделать Reloading configuration. Подробности по ссылке:
https://aerokube.com/selenoid/latest/#_reloading_configuration


### Извесные баги

<https://github.com/aerokube/demo-tests/issues/5>

