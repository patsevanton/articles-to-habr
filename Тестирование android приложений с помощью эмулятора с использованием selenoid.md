Предисловие из [поста](https://habr.com/ru/post/463525/):

Selenoid — это программа, которая позволяет управлять браузерами и Android-эмуляторами с помощью специальных драйверов. Умеет запускать каждый из них изолированно в Docker-контейнере.

Selenoid представляет собой альтернативное решение Selenium Server, хотя суть та же — организация работы драйверов.

Основная идея Selenoid состоит в том, чтобы запускать новый контейнер для каждой сессии (запуска нового браузера или эмулятора) и останавливать его сразу же после закрытия сессии.

Selenoid позволяет поддерживать высокую нагрузку без дополнительных ресурсозатрат.

В этом посте будет запуск простых тестов в Android-эмуляторе.

### Подготовка

Предварительно проверьте что ваша система может запускать виртуальные машины.

Аппаратная виртуализация должна поддерживаться вашим процессором. Это означает, что требуют­ся расширения процессора Intel­VT или AMD­V. Чтобы убедиться, под держивает ли процессор одно из них, выполните команду:

```bash
egrep '(vmx|svm)' /proc/cpuinfo
```

### Docker

На вашей операционной системе обязательно должен быть установлен и запущен Docker.

### Установка Selenoid

Если у вас Redhat-based операционная система, вы можете использовать мой репозиторий для установки Selenoid Configuration manager.

```bash
yum -y install yum-plugin-copr
yum copr enable antonpatsev/aerokube-cm-rpm
yum -y install aerokube-cm
```

Если у вас не Redhat-based операционная система, то вы можете скачать и использовать бинарник Selenoid Configuration manager.

### Запуск Selenoid Configuration manager и формирование browsers.json

```bash
Если у вас нет прямого доступа в инет и docker образы вы скачиваете через registry:
aerokube-cm selenoid start --force --browsers "android:6.0;chrome:78" --registry ваш-docker-registry
```

Если у вас есть прямой доступ в инет.

```bash
aerokube-cm selenoid start --force --browsers "android:6.0;chrome:78"
```

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

