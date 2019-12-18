[Cucumber](https://cucumber.io/) – это инфраструктура тестирования, позволяющая преодолеть разрыв между разработчиками ПО и бизнес-менеджерами. Тесты пишутся на простом языке управляемой поведением разработки (BDD) в стиле Given, When, Then (условия, операция, результат), которой понятен любому пользователю. Затем контрольные тесты записываются в файлы функций, охватывающие один или несколько сценариев тестирования. Cucumber интерпретирует тесты на указанном языке программирования и использует Selenium для управления тестами в браузере.

[Selenoid](https://github.com/aerokube/selenoid) представляет собой альтернативное решение Selenium Server, хотя суть та же — организация работы драйверов.

В этом посте будет запуск простого браузерного теста с помощью Cucumber и Selenoid в Gitlab CI c Allure отчетом в конце.

### Подготовка

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

Если у вас нет прямого доступа в инет и docker образы вы скачиваете через registry.

```bash
aerokube-cm selenoid start --force --browsers "firefox:70.0;firefox:71.0;chrome:78.0;chrome:79.0" --registry ваш-docker-registry
```

Если у вас есть прямой доступ в инет.

```bash
aerokube-cm selenoid start --force --browsers "firefox:70.0;firefox:71.0;chrome:78.0;chrome:79.0"
```

Ключ `--force` перезаписывает файл browsers.json.


Итоговый файл browsers.json для тестирования Chrome и Firefox. Заметьте что path у Chrome и Firefox разные.

```json
{
    "chrome": {
        "default": "79.0",
        "versions": {
            "78.0": {
                "image": "docker-registry/selenoid/chrome:78.0",
                "port": "4444",
                "path": "/"
            },
            "79.0": {
                "image": "docker-registry/selenoid/chrome:79.0",
                "port": "4444",
                "path": "/"
            }
        }
    },
    "firefox": {
        "default": "71.0",
        "versions": {
            "70.0": {
                "image": "docker-registry/selenoid/firefox:70.0",
                "port": "4444",
                "path": "/wd/hub"
            },
            "71.0": {
                "image": "docker-registry/selenoid/firefox:71.0",
                "port": "4444",
                "path": "/wd/hub"
            }
        }
    }
}
```

### Изменение browsers.json

При изменении версий браузеров можно изменить файл browsers.json и перезагрузить selenoid.

Но если вы используете ключ `--force`, то browsers.json перезапишится с новыми версиями браузеров.


Проверяем что docker контейнер запустился и образы скачались.

```bash
docker ps
docker images
```

![](https://habrastorage.org/webt/ti/bz/nh/tibznhc23bodj4jxrx1to95znbo.png)

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

![](https://habrastorage.org/webt/mv/c9/g7/mvc9g78xgxhyy45cf3xxahdsy3i.png)

Заходим в selenoid-ui по адресу ip-где-вы-запускали-selenoid-и-selenoid-ui:8080

У вас должно быть гореть зеленым 2 слова CONNECTED и написано android и chrome.

![](https://habrastorage.org/webt/-k/fi/dj/-kfidjzsisil5mdy8migutx-xro.png)

В capabilities видим доступные браузеры.

![](https://habrastorage.org/webt/ed/re/bp/edrebpyonygfiokgctqgy0catak.png)



DEMO TEST

Скачиваем https://github.com/andewBr/cucumber_selenium_test



![](https://habrastorage.org/webt/i-/6v/3_/i-6v3_qo-z7gaxzdx5t_nsdxyrm.png)

или на другой адрес, там где вы запустили selenoid.

В файле WebdriverBeanConfig.java добавляем Capability для запуска chrome. Если вы используете прокси сервер, то добавьте строки про прокси как на стриншоте.

![](https://habrastorage.org/webt/zw/vs/em/zwvsemk7jsrrxobacuqzg5hzvfu.png)

В каждом файле java вы можете включить или выключить запись видео, удаленный просмотр или управление через VNC и запись логов в файл. Чтобы выключить опцию нужно добавить 2 слеша в начало строки.

![](https://habrastorage.org/webt/ji/wx/mh/jiwxmhj3ezvbcf8eh-vlha3gpg4.png)



```bash

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

Общее время запуск 1 теста занимает меньше 1 минуты.



### Запуск тестов в Gitlab CI

Чтобы каждый раз не запускать Selenoid и Selenoid-UI при запуске тестов, можно запустить Selenoid и Selenoid-UI один раз при запуске Gitlab Runner с помощью Ansible, Puppet, Chef или других инструментов.

Итоговый .gitlab-ci.yml

```yaml
---
variables:
  MAVEN_OPTS: "-Dhttps.protocols=TLSv1.2 -Dmaven.repo.local=/home/gitlab-runner/.m2/repository -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=WARN -Dorg.slf4j.simpleLogger.showDateTime=true -Djava.awt.headless=true"
  MAVEN_CLI_OPTS: "--batch-mode --errors --fail-at-end --show-version -DinstallAtEnd=true -DdeployAtEnd=true"

before_script:
  - export PATH="/opt/rh/rh-maven33/root/usr/bin:$PATH"

build:
  stage: build
  script:
    - aerokube-cm selenoid start --force --browsers "firefox:70;firefox:71;chrome:78;chrome:79" --registry docker-registry
    - aerokube-cm selenoid-ui start --registry docker-registry
    - cat ~/.aerokube/selenoid/browsers.json
    - set +e
    - mvn -s maven_settings.xml clean install -Dmaven.test.skip=true
    - mvn -s maven_settings.xml clean test || EXIT_CODE=$?
    - mvn -s maven_settings.xml allure:aggregate;
    - export PATH_WITHOUT_HOME=$(pwd | sed -e "s/\/home\/gitlab-runner\/builds//g")
    - echo '***********************************************************************'
    - echo "http://$HOSTNAME:9090$PATH_WITHOUT_HOME/target/site/allure-maven-plugin/"
    - echo '***********************************************************************'
    - set -e
    - exit ${EXIT_CODE}

```

После строки `echo "http://$HOSTNAME:9090$PATH_WITHOUT_HOME/target/site/allure-maven-plugin/"`

Появится URL, по которому можно просмотреть Allure отчет.

Для отображения Allure отчета нужно чтобы на gitlab runner был установлен nginx с такой конфигурацией:
```bash
server {
    listen       9090;
    listen       [::]:9090;
    server_name  _;
    root         /home/gitlab-runner/builds;

    location / {
        autoindex on;
    }

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
```

Скриншот Allure отчета

![](https://habrastorage.org/webt/1a/nk/-v/1ank-vhxpxd1fc2mkoykry6f8ja.png)
