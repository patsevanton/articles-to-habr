Важная часть управления уязвимостями состоит в том, чтобы хорошо понимать и обеспечить безопасность цепочки поставок тех компонентов ПО, из которых строятся современные системы. Команды, практикующие гибкие методики и DevOps, широко применяют библиотеки и каркасы с открытым исходным кодом, чтобы сократить время и стоимость разработки. Но эта медаль имеет и обратную сторону: возможность получить в наследство чужие ошибки и уязвимости.

Очевидно, команда должна обязательно знать, какие компоненты с открытым исходным кодом включены в ее приложения, следить за тем, чтобы заведомо надежные версии скачивались из заведомо надежных источников, и загружать обновленные версии компонентов после исправления вновь обнаруженных уязвимостей.

В этом посте рассмотрим использование OWASP Dependency Check и SonarQube для прерывания сборки в случае обнаружения серьезных проблем с вашим кодом.



В книге "Безопасность разработки в Agile-проектах" описан так. OWASP Dependency Check – это бесплатный сканер, который каталогизирует все используемые в приложении компоненты с открытым исходным кодом и показывает имеющиеся в них уязвимости. Имеются версии для Java, .NET, Ruby (gemspec), PHP (composer), Node.js и Python, а также для некоторых проектов на C/C++. Dependency Check интегрируется с распространенными средствами сборки, в т. ч. Ant, Maven и Gradle и серверами непрерывной интеграции типа Jenkins.

Dependency Check сообщает обо всех компонентах с известными уязвимостями из Национальной базы данных уязвимос­тей (NVD) NIST и обновляется на основании данных из новостных каналов NVD.

По счастью, все это можно делать автоматически с помощью таких инструментов, как проект OWASP Dependency Check, или коммерческих программ типа Black Duck (https://www.blackducksoftware.com/solutions/application-security), JFrog Xray (https://jfrog.com/xray/), Snyk (https://snyk.io/), Nexus Lifecycle (https://www.sonatype.com/nexus-lifecycle) компании Sonatype или SourceClear (https://srcclr.com/).
Эти инструменты можно включить в сборочные конвейеры, чтобы автоматически составлять опись зависимостей с открытым исходным кодом, выявлять устаревшие версии библиотек и библиотеки, содержащие известные уязвимости, и прерывать сборку в случае обнаружения серьезных проблем.

### OWASP Dependency Check

Для тестирования и демонстрации работы Dependency Check используем этот репозиторий [dependency-check-example](https://github.com/patsevanton/dependency-check-example).

Для просмотра HTML отчета нужно настроить web-севрер nginx на вашем gitlab-runner.

Пример минимального конфига nginx:

```nginx
server {
    listen       9999;
    listen       [::]:9999;
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
}
```

В конце сборки вы можете видеть вот таку картину:

![](https://habrastorage.org/webt/dt/e8/ib/dte8ibohz7sbhhbyxsq-ynjup2y.png)

Переходим по ссылке и видим отчет Dependency Check.

Первый скриншот - верхняя часть отчета с кратким содержанием.

![](https://habrastorage.org/webt/ir/3y/av/ir3yavm9sinwnl0dfubbhs3hdtu.png)

Второй скриншот подробности CVE-2017-5638. Здесь мы видим уроверь CVE и ссылки на эксплойты.

![](https://habrastorage.org/webt/hb/cu/np/hbcunpffqa0yana-2mrk7egizcq.png)

Третий скриншот - подробности log4j-api-2.7.jar. Видим что уровни CVE 7.5 и 9.8.

![](https://habrastorage.org/webt/um/o7/as/umo7ashepjrwzig0acuuz8vcxbq.png)

Четвертый скриншот - подробности commons-fileupload-1.3.2.jar. Видим что уровни CVE 7.5 и 9.8.

![](https://habrastorage.org/webt/4r/6l/vr/4r6lvrgkpzr-i-z-30qryxzsx5i.png)

Если вы хотите использовать gitlab pages, то не получится - упавшая таска не создаст артефакт.

Пример здесь https://gitlab.com/anton_patsev/dependency-check-example-gitlab-pages.

Вывод сборки: артефактов нет, html отчета не вижу.

https://gitlab.com/anton_patsev/dependency-check-example-gitlab-pages/-/jobs/400004246

![](https://habrastorage.org/webt/ja/az/u_/jaazu_byj-i6iwgv9zy8tmnzgrw.png)



##### Регулирование уровня CVE уязвимостей

Самая главная строка в файле gitlab-ci.yaml:

```bash
mvn $MAVEN_CLI_OPTS test org.owasp:dependency-check-maven:check -DfailBuildOnCVSS=7
```

Параметром failBuildOnCVSS вы можете регулировать уровень CVE уязвимостей, на которые нужно реагировать.

##### Скачивание с интернета базы данных уязвимос­тей (NVD) NIST

Вы заметили что постоянно скачивает базы данных уязвимос­тей (NVD) NIST с интернета:

![](https://habrastorage.org/webt/7s/hy/8b/7shy8blkqreiypfplbfcpgtg50s.png)

Для скачивания можно использовать утилиту [nist_data_mirror_golang](https://github.com/patsevanton/nist_data_mirror_golang)

Установим и запустим ее.

```
yum -y install yum-plugin-copr
yum copr enable antonpatsev/nist_data_mirror_golang
yum -y install nist-data-mirror
systemctl start nist-data-mirror
```

Nist-data-mirror закачивает CVE JSON NIST в /var/www/repos/nist-data-mirror/ при запуске и обновляет данные каждые 24 часа.

Для скачивания CVE JSON NIST нужно настроить web-севрер nginx (например на вашем gitlab-runner).

Пример минимального конфига nginx:

```nginx
server {
    listen       12345;
    listen       [::]:12345;
    server_name  _;
    root         /var/www/repos/nist-data-mirror/;

    location / {
        autoindex on;
    }

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }

}
```

Чтобы не делать длинную строку где запускается mvn вынесем параметры в отдельную переменную  DEPENDENCY_OPTS.

Итоговый минимальный конфиг .gitlab-ci.yml получится вот так: 

```yaml
variables:
  MAVEN_OPTS: "-Dhttps.protocols=TLSv1.2 -Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=WARN -Dorg.slf4j.simpleLogger.showDateTime=true -Djava.awt.headless=true"
  MAVEN_CLI_OPTS: "--batch-mode --errors --fail-at-end --show-version -DinstallAtEnd=true -DdeployAtEnd=true"
  DEPENDENCY_OPTS: "-DfailBuildOnCVSS=7 -DcveUrlModified=http://localhost:12345/nvdcve-1.1-modified.json.gz -DcveUrlBase=http://localhost:12345/nvdcve-1.1-%d.json.gz"

cache:
  paths:
    - .m2/repository

verify:
  stage: test
  script:
    - set +e
    - mvn $MAVEN_CLI_OPTS test org.owasp:dependency-check-maven:check $DEPENDENCY_OPTS || EXIT_CODE=$?
    - export PATH_WITHOUT_HOME=$(pwd | sed -e "s/\/home\/gitlab-runner\/builds//g")
    - echo "************************* URL Dependency-check-report.html *************************"
    - echo "http://$HOSTNAME:9999$PATH_WITHOUT_HOME/target/dependency-check-report.html"
    - set -e
    - exit ${EXIT_CODE}
  tags:
    - shell

```

