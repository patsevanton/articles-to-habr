[Cucumber](https://cucumber.io/) – это инфраструктура тестирования, позволяющая преодолеть разрыв между разработчиками ПО и бизнес-менеджерами. Тесты пишутся на простом языке управляемой поведением разработки (BDD) в стиле Given, When, Then (условия, операция, результат), которой понятен любому пользователю. Затем контрольные тесты записываются в файлы функций, охватывающие один или несколько сценариев тестирования. Cucumber интерпретирует тесты на указанном языке программирования и использует Selenium для управления тестами в браузере.

[Selenoid](https://github.com/aerokube/selenoid) представляет собой альтернативное решение Selenium Server, хотя суть та же — организация работы драйверов.

В этом посте будет запуск простых браузерных тестов с помощью Cucumber и Selenoid в Gitlab CI.md.

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

Если у вас нет прямого доступа в инет и docker образы вы скачиваете через registry:

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

![](https://habrastorage.org/webt/-k/fi/dj/-kfidjzsisil5mdy8migutx-xro.png)

В capabilities видим доступные браузеры.

![](https://habrastorage.org/webt/ed/re/bp/edrebpyonygfiokgctqgy0catak.png)



### DEMO TEST

Скачиваем https://github.com/aerokube/demo-tests



![](https://habrastorage.org/webt/i-/6v/3_/i-6v3_qo-z7gaxzdx5t_nsdxyrm.png)

или на другой адрес, там где вы запустили selenoid.

В файле DemoTest.java добавляем setCapability для запуска chrome на Android чтобы получилось примерно так. Если вы используете прокси сервер, то добавьте строки про прокси как на стриншоте.

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




