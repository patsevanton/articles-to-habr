Важная часть управления уязвимостями состоит в том, чтобы хорошо понимать и обеспечить безопасность цепочки поставок тех компонентов ПО, из которых строятся современные системы. Команды, практикующие гибкие методики и DevOps, широко применяют библиотеки и каркасы с открытым исходным кодом, чтобы сократить время и стоимость разработки. Но эта медаль имеет и обратную сторону: возможность получить в наследство чужие ошибки и уязвимости.

Очевидно, команда должна обязательно знать, какие компоненты с открытым исходным кодом включены в ее приложения, следить за тем, чтобы заведомо надежные версии скачивались из заведомо надежных источников, и загружать обновленные версии компонентов после исправления вновь обнаруженных уязвимостей.

В этом посте рассмотрим использование OWASP Dependency Check и SonarQube для прерывания сборки в случае обнаружения серьезных проблем с вашим кодом.



В книге "Безопасность разработки в Agile-проектах" описан так. OWASP Dependency Check – это бесплатный сканер, который каталогизирует все используемые в приложении компоненты с открытым исходным кодом и показывает имеющиеся в них уязвимости. Имеются версии для Java, .NET, Ruby (gemspec), PHP (composer), Node.js и Python, а также для некоторых проектов на C/C++. Dependency Check интегрируется с распространенными средствами сборки, в т. ч. Ant, Maven и Gradle и серверами непрерывной интеграции типа Jenkins.

Dependency Check сообщает обо всех компонентах с известными уязвимостями из Национальной базы данных уязвимос­тей (NVD) NIST и обновляется на основании данных из новостных каналов NVD.

По счастью, все это можно делать автоматически с помощью таких инструментов, как проект OWASP Dependency Check, или коммерческих программ типа Black Duck (https://www.blackducksoftware.com/solutions/application-security), JFrog Xray (https://jfrog.com/xray/), Snyk (https://snyk.io/), Nexus Lifecycle (https://www.sonatype.com/nexus-lifecycle) компании Sonatype или SourceClear (https://srcclr.com/).
Эти инструменты можно включить в сборочные конвейеры, чтобы автоматически составлять опись зависимостей с открытым исходным кодом, выявлять устаревшие версии библиотек и библиотеки, содержащие известные уязвимости, и прерывать сборку в случае обнаружения серьезных проблем.

SonarQube - это открытая платформа для обеспечения непрерывного контроля качества исходного кода, поддерживающая большое количество языков программирования и позволяющая получать отчеты по таким метрикам, как дублирование кода, соответствие стандартам кодирования, покрытие тестами, сложность кода, потенциальные ошибки и т.д. SonarQube удобно визуализирует результаты анализа и позволяет отслеживать динамику развития проекта во времени.

### OWASP Dependency Check

Для тестирования и демонстрации работы Dependency Check используем этот репозиторий:

https://github.com/patsevanton/dependency-check-example 

Для просмотра HTML отчета нужно настроить web-севрер nginx на вашем gitlab-runner:

```nginx
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



Самая главная строка в файле gitlab-ci.yaml:

```bash
mvn $MAVEN_CLI_OPTS test org.owasp:dependency-check-maven:check -DfailBuildOnCVSS=7
```

Параметром failBuildOnCVSS вы можете регулировать уровень CVE уязвимостей, на которые нужно реагировать.