SonarQube - это открытая платформа для обеспечения непрерывного контроля качества исходного кода, поддерживающая большое количество языков программирования и позволяющая получать отчеты по таким метрикам, как дублирование кода, соответствие стандартам кодирования, покрытие тестами, сложность кода, потенциальные ошибки и т.д. SonarQube удобно визуализирует результаты анализа и позволяет отслеживать динамику развития проекта во времени.

Задача: Показывать разработчикам статус контроля качества исходного кода в SonarQube.

Есть два способа решения:

- Запускать скрипт проверки статуса контроля качества исходного кода в SonarQube. Если контроль качества исходного кода в SonarQube не проходит, то фейлить сборку.
- Показывать на главной странице проекта статус контроля качества исходного кода.


### Установка SonarQube

Для установки sonarqube из rpm пакетов воспользуемся репозиторием https://harbottle.gitlab.io/harbottle-main.

Установим пакет с репозиторием для CentOS 7.

```bash
yum install -y https://harbottle.gitlab.io/harbottle-main/7/x86_64/harbottle-main-release.rpm
```

Устанавливаем сам sonarqube.

```bash
yum install -y sonarqube
```

При установке установятся большинство плагинов, но нужно доустановить findbugs и pmd

```
yum install -y sonarqube-findbugs sonarqube-pmd
```

Запустим и включим в автомзагрузку

```bash
systemctl start sonarqube
systemctl enable sonarqube
```

Если долго загружается, то добавьте в конец опций sonar.web.javaOpts генератор случайных чисел /dev/./urandom

```bash
sonar.web.javaOpts=другие параметры -Djava.security.egd=file:/dev/./urandom
```

### Запуск скрипта проверки статуса контроля качества исходного кода в SonarQube.

К сожалению, плагин sonar-break-maven-plugin давно не обновлялся. Поэтому напишем свой скрипт.

Для тестирования будем использовать репозиторий https://github.com/uweplonus/spotbugs-examples.

Импортируем в Gitlab. Добавляем файл .gitlab-ci.yml:

```yaml
variables:
  MAVEN_OPTS: "-Dhttps.protocols=TLSv1.2 -Dmaven.repo.local=~/.m2/repository -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=WARN -Dorg.slf4j.simpleLogger.showDateTime=true -Djava.awt.headless=true"
  MAVEN_CLI_OPTS: "--batch-mode --errors --fail-at-end --show-version -DinstallAtEnd=true -DdeployAtEnd=true"
  SONAR_HOST_URL: "http://172.26.9.226:9000"
  LOGIN: "admin" # логин sonarqube
  PASSWORD: "admin" # пароль sonarqube

cache:
  paths:
    - .m2/repository

build:
  image: maven:3.3.9-jdk-8
  stage: build
  script:
    - wget http://ftp.ru.debian.org/debian/pool/main/j/jq/jq_1.4-2.1+deb8u1_amd64.deb
    - dpkg -i jq_1.4-2.1+deb8u1_amd64.deb && rm -f jq_1.4-2.1+deb8u1_amd64.deb || true
    - mvn $MAVEN_CLI_OPTS -Dmaven.test.failure.ignore=true org.jacoco:jacoco-maven-plugin:0.8.5:prepare-agent clean verify org.jacoco:jacoco-maven-plugin:0.8.5:report
    - mvn $MAVEN_CLI_OPTS -Dmaven.test.skip=true verify sonar:sonar -Dsonar.host.url=$SONAR_HOST_URL -Dsonar.login=$LOGIN -Dsonar.password=$PASSWORD -Dsonar.gitlab.project_id=$CI_PROJECT_PATH -Dsonar.gitlab.commit_sha=$CI_COMMIT_SHA -Dsonar.gitlab.ref_name=$CI_COMMIT_REF_NAME
    - export URL=$(cat target/sonar/report-task.txt | grep ceTaskUrl | cut -c11- ) #URL where report gets stored
    - echo $URL
    - |
      while : ;do
          curl -k -u "$LOGIN":"$PASSWORD" "$URL" -o analysis.txt
          export status=$(cat analysis.txt | jq -r '.task.status') #Status as SUCCESS, CANCELED, IN_PROGRESS or FAILED
          echo $status
          if [ ${status} == "SUCCESS" ];then
            echo "SONAR ANALYSIS SUCCESS";
            break
          fi
          sleep 5
      done
    - curl -k -u "$LOGIN":"$PASSWORD" "$URL" -o analysis.txt
    - export status=$(cat analysis.txt | jq -r '.task.status') #Status as SUCCESS, CANCELED or FAILED
    - export analysisId=$(cat analysis.txt | jq -r '.task.analysisId') #Get the analysis Id
    - |
      if [ "$status" == "SUCCESS" ]; then
        echo -e "SONAR ANALYSIS SUCCESSFUL...ANALYSING RESULTS";
        curl -k -u "$LOGIN":"$PASSWORD" "$SONAR_HOST_URL/api/qualitygates/project_status?analysisId=$analysisId" -o result.txt; #Analysis result like critical, major and minor issues
        export result=$(cat result.txt | jq -r '.projectStatus.status');

        if [ "$result" == "ERROR" ];then
          echo -e "91mSONAR RESULTS FAILED";
          echo "$(cat result.txt | jq -r '.projectStatus.conditions')"; #prints the critical, major and minor violations
          exit 1 #breaks the build for violations
        else
          echo -e "SONAR RESULTS SUCCESSFUL";
          echo "$(cat result.txt | jq -r '.projectStatus.conditions')";
          exit 0
        fi
      else
          echo -e "\e[91mSONAR ANALYSIS FAILED\e[0m";
          exit 1 #breaks the build for failure in Step2
      fi
  tags:
    - docker
```

Файл .gitlab-ci.yml неидеальный. Тестировался если задачи проверки в sonarqube завершались статусом: "SUCCESS". Пока что другие статусов не было. Как будут другие статусы, поправлю .gitlab-ci.yml  в этом посте.

### Отображение на главной странице проекта статус контроля качества исходного кода.

Устанавливаем плагин для SonarQube

```bash
yum install -y sonarqube-qualinsight-badges
```

Заходим в SonarQube по адресу http://172.26.9.115:9000/
Создаем обычного пользователя, например "badges".
Заходим под этим пользователем в SonarQube.

![](https://habrastorage.org/webt/sh/py/jh/shpyjhemnnulcw9kbrxakwza1lu.png)

Заходим в "My account", создаем новый токер, например с названием "read_all_repository" и нажимаем "Genereate".

![](https://habrastorage.org/webt/a4/mm/qi/a4mmqitdr2mlbbspaxvlosm_0ty.png)

Видим что появился токен. Он появлется только 1 раз.

Заходим под администратором.

Идем в Configuration -> SVG Badges

![](https://habrastorage.org/webt/_1/la/oz/_1laozdzrcqevucvlg6fkvzws_s.png)

Копируем этот токен в поле "Activity badge token" и нажимаем кнопку save.

![](https://habrastorage.org/webt/--/4z/rd/--4zrd1ngsyoiek-ecv6wbh_qjw.png)

Заходим в Administration -> Security -> Permission Templates -> Default template (и другие шаблоны, которые у вас будут).

У пользователя badges необходимо установить галку "Browse".

Тестирование.

Для примера возьмем проект <https://github.com/jitpack/maven-simple>.

Импортируем этот проект. 

Добавляяем файл .gitlab-ci.yml в корень проекта со следующим содержимым.

```yaml
variables:
  MAVEN_OPTS: "-Dhttps.protocols=TLSv1.2 -Dmaven.repo.local=~/.m2/repository -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=WARN -Dorg.slf4j.simpleLogger.showDateTime=true -Djava.awt.headless=true"
  MAVEN_CLI_OPTS: "--batch-mode --errors --fail-at-end --show-version -DinstallAtEnd=true -DdeployAtEnd=true"
  SONAR_HOST_URL: "http://172.26.9.115:9000"
  LOGIN: "admin" # логин sonarqube
  PASSWORD: "admin" # пароль sonarqube

cache:
  paths:
    - .m2/repository

build:
  image: maven:3.3.9-jdk-8
  stage: build
  script:
    - mvn $MAVEN_CLI_OPTS -Dmaven.test.failure.ignore=true org.jacoco:jacoco-maven-plugin:0.8.5:prepare-agent clean verify org.jacoco:jacoco-maven-plugin:0.8.5:report
    - mvn $MAVEN_CLI_OPTS -Dmaven.test.skip=true verify sonar:sonar -Dsonar.host.url=$SONAR_HOST_URL -Dsonar.login=$LOGIN -Dsonar.password=$PASSWORD -Dsonar.gitlab.project_id=$CI_PROJECT_PATH -Dsonar.gitlab.commit_sha=$CI_COMMIT_SHA -Dsonar.gitlab.ref_name=$CI_COMMIT_REF_NAME
  tags:
    - docker
```



Опции для получение метрик можно посмотреть тут:

https://github.com/QualInsight/qualinsight-plugins-sonarqube-badges/wiki/Measure-badges



[![Quality Gate](http://172.26.9.223:9000/api/badges/gate?key=com.github.jitpack:maven-simple)](http://172.26.9.223:9000/dashboard?id=com.github.jitpack%3Amaven-simple)
[![Vulnerabilities](http://172.26.9.223:9000/api/badges/measure?key=com.github.jitpack:maven-simple&metric=vulnerabilities)](http://172.26.9.223:9000/dashboard?id=com.github.jitpack%3Amaven-simple)
[![Code Smells](http://172.26.9.223:9000/api/badges/measure?key=com.github.jitpack:maven-simple&metric=code_smells)](http://172.26.9.223:9000/dashboard?id=com.github.jitpack%3Amaven-simple)
[![Coverage](http://172.26.9.223:9000/api/badges/measure?key=com.github.jitpack:maven-simple&metric=coverage)](http://172.26.9.223:9000/dashboard?id=com.github.jitpack%3Amaven-simple)
[![Duplications](http://172.26.9.223:9000/api/badges/measure?key=com.github.jitpack:maven-simple&metric=duplicated_lines_density)](http://172.26.9.223:9000/dashboard?id=com.github.jitpack%3Amaven-simple)


https://github.com/QualInsight/qualinsight-plugins-sonarqube-badges/wiki/Measure-badges

http://172.26.9.223:9000/api/badges/measure?key=com.github.jitpack:maven-simple&metric=coverage

http://172.26.9.223:9000/api/badges/measure?key=com.github.jitpack:maven-simple&metric=bugs

http://172.26.9.223:9000/api/badges/measure?key=com.github.jitpack:maven-simple&metric=vulnerabilities

http://172.26.9.223:9000/api/badges/measure?key=com.github.jitpack:maven-simple&metric=code_smells

http://172.26.9.223:9000/api/badges/gate?key=com.github.jitpack:maven-simple
